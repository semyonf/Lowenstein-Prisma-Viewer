# Lowenstein Prisma CPAP Viewer — Implementation Plan

A fullstack web application for uploading, parsing, storing, and visualizing Lowenstein Prisma CPAP therapy data. Built on the T3 stack (Next.js 15, tRPC 11, Drizzle ORM, Better Auth, Tailwind CSS 4, PostgreSQL).

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│  Browser                                                 │
│  ┌────────────┐  ┌────────────┐  ┌─────────────────────┐ │
│  │ Upload     │  │ Dashboard  │  │ Night Detail View   │ │
│  │ .pcfg/.pdat│  │ AHI chart  │  │ Event timeline      │ │
│  │ drag+drop  │  │ Usage chart│  │ Signal channels     │ │
│  │            │  │ Leak chart │  │ Session breakdown   │ │
│  └─────┬──────┘  └──────┬─────┘  └──────────┬──────────┘ │
│        │       tRPC (React Query)           │            │
├────────┴────────────────┴───────────────────┴────────────┤
│  Next.js API                                             │
│  ┌──────────┐  ┌────────────┐  ┌────────────────────────┐│
│  │ Auth     │  │ Upload &   │  │ tRPC Routers           ││
│  │ (Better  │  │ Parse      │  │ device, night, stats,  ││
│  │  Auth)   │  │ Pipeline   │  │ config, signals, logs  ││
│  └──────────┘  └──────┬─────┘  └───────────┬────────────┘│
│                       │  Drizzle ORM       │             │
├───────────────────────┴────────────────────┴─────────────┤
│  PostgreSQL                                              │
│  devices, nights, sessions, resp_events, signal_metadata,│
│  statistics, config_params, logs                         │
└──────────────────────────────────────────────────────────┘
```

### Data Flow

1. User uploads `.pcfg` + `.pdat` ZIP archives
2. Server extracts ZIPs in memory, parses XML/INI/EDF headers
3. Parsed data is stored relationally in PostgreSQL via Drizzle
4. tRPC queries serve data to React components
5. Charts rendered client-side with a charting library

---

## Phase 1 — Database Schema & Data Models

**Goal:** Design and implement the Drizzle schema to hold all parsed CPAP data.

### Tables

#### `devices`
Stores device identity from `device.xml`.

| Column | Type | Notes |
|--------|------|-------|
| id | serial PK | |
| userId | text FK → user | Owner |
| serialNumber | text | DeviceSerialNumber |
| mainboardSerial | text | MainboardSerialNumber |
| deviceType | int | DeviceType (10 = Prisma) |
| fwVersion | text | e.g. "5.07" |
| fwRevision | text | e.g. "0002" |
| fwBuild | text | e.g. "2023-0310-1825-Eyra" |
| fwGitHash | text | |
| pmVersion | text | Parameter Management version |
| hwVersion | text | Mainboard HW version |
| displayHwVersion | text | |
| deviceVariant | int | |
| deviceBranding | int | |
| createdAt | timestamp | |
| updatedAt | timestamp | |

Unique constraint on `(userId, serialNumber)` — a user can't upload the same device twice, but merges data on re-upload.

#### `uploads`
Tracks each archive upload for audit/re-processing.

| Column | Type | Notes |
|--------|------|-------|
| id | serial PK | |
| userId | text FK → user | |
| deviceId | int FK → devices | |
| filename | text | Original filename |
| fileType | text | "pcfg" or "pdat" |
| fileSize | int | Bytes |
| status | text | "processing", "complete", "error" |
| error | text | nullable error message |
| createdAt | timestamp | |

#### `nights`
One row per therapy date (YYYYMMDD directory).

| Column | Type | Notes |
|--------|------|-------|
| id | serial PK | |
| deviceId | int FK → devices | |
| date | date | Therapy date |
| sessionCount | int | Number of event files this night |
| createdAt | timestamp | |

Unique constraint on `(deviceId, date)`.

#### `sessions`
One row per event/signal file pair.

| Column | Type | Notes |
|--------|------|-------|
| id | serial PK | |
| nightId | int FK → nights | |
| deviceId | int FK → devices | |
| sequenceNum | int | Event file number (e.g. 48 from event_000048.xml) |
| startTime | timestamp | Computed from signal EDF header |
| durationSec | int | From RespEventID=231 Duration |
| therapyMode | int | From DeviceEvent ParameterID=1003 |
| ahiX10 | int | From RespEventID=1230 Strength |
| aiX10 | int | 1231 |
| hiX10 | int | 1232 |
| leak95th | int | 1233 |
| pressureMedian | int | 1237 (Pa) |
| pressure95th | int | 1238 (Pa) |
| createdAt | timestamp | |

#### `respEvents`
Individual respiratory/therapy events from event XML files.

| Column | Type | Notes |
|--------|------|-------|
| id | serial PK | |
| sessionId | int FK → sessions | |
| eventTypeId | int | RespEventID |
| endTimeSec | int | Seconds from session start |
| durationSec | int | |
| pressure | int | Pa |
| strength | int | Severity (0-10) |
| visible | boolean | Default true |

Index on `(sessionId, eventTypeId)` for filtered queries.

#### `deviceEvents`
Configuration snapshots and runtime state changes from event XMLs.

| Column | Type | Notes |
|--------|------|-------|
| id | serial PK | |
| sessionId | int FK → sessions | |
| deviceEventId | int | 0=config, 1=runtime |
| timeSec | int | Seconds from session start |
| parameterId | int | Parameter ID |
| newValue | text | Parameter value |

#### `signalMetadata`
EDF header data for each signal file (not the waveform data itself).

| Column | Type | Notes |
|--------|------|-------|
| id | serial PK | |
| sessionId | int FK → sessions | |
| startDate | text | DD.MM.YY |
| startTime | text | HH:MM:SS |
| numSignals | int | Channel count (typically 18) |
| numRecords | int | -1 for streaming |
| recordDuration | int | Seconds per record |
| channels | jsonb | Array of channel labels |
| fileSize | int | Bytes |

#### `nightStats`
Denormalized nightly statistics from `statistics_year.bin`.

| Column | Type | Notes |
|--------|------|-------|
| id | serial PK | |
| nightId | int FK → nights | |
| deviceId | int FK → devices | |
| therapyMode | int | `m` attribute |
| therapyTimeSec | int | Stat 113 |
| usageTimeSec | int | Stat 111 |
| ahiX10 | int | Stat 106 |
| aiX10 | int | Stat 107 |
| hiX10 | int | Stat 108 |
| caiX10 | int | Stat 109 |
| oaCount | int | Stat 100 |
| caCount | int | Stat 101 |
| maCount | int | Stat 102 |
| hypopneaCount | int | Stat 104 |
| reraCount | int | Stat 116 |
| leak95th | int | Stat 119 |
| leakMedian | int | Stat 120 |
| pressureMedianPa10 | int | Stat 208 |
| pressure95thPa10 | int | Stat 210 |
| snoringDurationSec | int | Stat 204 |
| flowLimitDurationSec | int | Stat 206 |
| spo2Mean | int | Stat 300 (nullable) |
| spo2Min | int | Stat 301 (nullable) |
| hrMean | int | Stat 306 (nullable) |
| hrMax | int | Stat 308 (nullable) |
| respRateMean | int | Stat 400 |
| tidalVolume | int | Stat 402 |
| minuteVentilation | int | Stat 405 |
| inspTimeMsec | int | Stat 406 |
| expTimeMsec | int | Stat 407 |
| ieRatioX10 | int | Stat 408 |
| maskOffCount | int | Stat 418 |
| timestamps | text | Raw t= attribute (on/off pairs) |
| rawStats | jsonb | Full stat dump for fields not in columns |

#### `configParams`
Device configuration (from configuration.xml, one row per parameter).

| Column | Type | Notes |
|--------|------|-------|
| id | serial PK | |
| deviceId | int FK → devices | |
| section | text | "OBL" or "OPT" |
| parameterId | int | Numeric ID |
| value | text | Raw value |
| uploadId | int FK → uploads | Which upload set this |

#### `deviceLogs`
Stores log file contents.

| Column | Type | Notes |
|--------|------|-------|
| id | serial PK | |
| deviceId | int FK → devices | |
| uploadId | int FK → uploads | |
| logType | text | "therapy_sw", "kernel", etc. |
| content | text | Full log text (truncated to last N lines) |

### Files to create/modify

- `src/server/db/schema.ts` — Add all tables, relations, indexes
- Run `pnpm db:generate` + `pnpm db:push`

### Verification

- `pnpm db:studio` — Inspect empty tables in Drizzle Studio
- `pnpm typecheck` — Ensure schema types compile

---

## Phase 2 — Upload & Parse Pipeline

**Goal:** Accept `.pcfg`/`.pdat` file uploads, extract ZIPs, parse all formats, and store in the database.

### Upload API

Create a Next.js API route at `/api/upload` (not tRPC — file uploads are easier with raw routes):

- Accept `multipart/form-data` with one or more `.pcfg`/`.pdat` files
- Require authentication (check Better Auth session)
- Size limit: 50MB per file
- Process in a background-friendly way (can start with synchronous for MVP)

### Parse Pipeline

Create a `src/lib/parsers/` directory with pure functions:

| File | Input | Output |
|------|-------|--------|
| `extract-zip.ts` | Buffer | Map<string, Buffer> of extracted files |
| `parse-device-xml.ts` | XML string | DeviceInfo object |
| `parse-configuration-xml.ts` | XML string | ConfigParam[] |
| `parse-event-xml.ts` | XML string | { deviceEvents, respEvents } |
| `parse-signal-header.ts` | Buffer (first 5KB) | SignalMetadata |
| `parse-statistics-xml.ts` | XML string | NightStat[] |
| `parse-ini.ts` | INI string | Record<string, Record<string, string>> |
| `parse-trend-header.ts` | Buffer | TrendHeader |
| `index.ts` | Full ZIP buffer | Complete parsed archive |

Dependencies needed:
- `jszip` — ZIP extraction in Node.js (no native unzip in browser-safe Node)
- `fast-xml-parser` — Fast XML to JSON (lighter than xml2js)

### Ingest Orchestrator

`src/lib/parsers/ingest.ts` — Coordinates the full upload:

1. Extract ZIP
2. Route files to appropriate parsers based on path patterns
3. Upsert device (match on serialNumber + userId)
4. Upsert nights (match on deviceId + date)
5. Insert sessions, events, stats, config, logs
6. Update upload record status

### tRPC Router

`src/server/api/routers/upload.ts`:
- `upload.list` — List user's uploads with status
- `upload.delete` — Remove an upload and cascade-delete its data

### Files to create

- `src/lib/parsers/*.ts` — 8 parser files + index + ingest
- `src/app/api/upload/route.ts` — Upload endpoint
- `src/server/api/routers/upload.ts` — Upload management router

### Dependencies to add

```bash
pnpm add jszip fast-xml-parser
```

### Verification

- Unit test: parse each sample file from `/tmp/cpap/extracted/`
- Integration test: upload both archives via curl, check DB in Drizzle Studio
- `pnpm typecheck`

---

## Phase 3 — UI Foundation & Component Library

**Goal:** Set up shadcn/ui, layout shell, navigation, and auth pages.

### Install shadcn/ui

```bash
pnpm dlx shadcn@latest init
```

Components to add:
- `button`, `card`, `table`, `badge`, `tabs`, `dialog`, `dropdown-menu`
- `skeleton` (loading states), `toast` (notifications)
- `input`, `label`, `form` (upload form)
- `sheet` (mobile nav), `separator`, `scroll-area`

### App Layout

`src/app/layout.tsx` — Update with:
- Dark mode support (class-based via `next-themes`)
- Sidebar navigation (collapsible)
- Top bar with user avatar + sign out

### Navigation Structure

```
/                   → Dashboard (redirect to /dashboard if authed)
/sign-in            → Sign in page
/sign-up            → Sign up page
/dashboard          → Overview cards + charts
/upload             → Upload page (drag & drop)
/nights             → Night list
/nights/[id]        → Night detail (sessions, events, timeline)
/statistics         → Statistics table + charts
/signals            → Signal metadata browser
/device             → Device info + config
/logs               → Log viewer
```

### Auth Pages

- `src/app/sign-in/page.tsx` — Email/password + GitHub OAuth button
- `src/app/sign-up/page.tsx` — Registration form

### Shared Components

| Component | Purpose |
|-----------|---------|
| `src/app/_components/sidebar.tsx` | App sidebar with nav links |
| `src/app/_components/topbar.tsx` | User menu, theme toggle |
| `src/app/_components/pressure-display.tsx` | Renders Pa as cmH2O |
| `src/app/_components/duration-display.tsx` | Renders seconds as Xh Ym |
| `src/app/_components/ahi-badge.tsx` | Color-coded AHI badge (green/orange/red) |
| `src/app/_components/empty-state.tsx` | "No data yet" placeholder |

### Dependencies to add

```bash
pnpm add next-themes
pnpm dlx shadcn@latest init
pnpm dlx shadcn@latest add button card table badge tabs dialog ...
```

### Files to create/modify

- `src/app/layout.tsx` — Restructure with sidebar layout
- `src/app/(auth)/sign-in/page.tsx`
- `src/app/(auth)/sign-up/page.tsx`
- `src/app/(app)/layout.tsx` — Authenticated layout with sidebar
- `src/app/(app)/dashboard/page.tsx` — Placeholder
- `src/app/_components/*.tsx` — Shared components

### Verification

- Navigate all routes, verify layout renders
- Sign in / sign out flow works
- Responsive on mobile (sidebar collapses)
- `pnpm typecheck && pnpm build`

---

## Phase 4 — Upload UI & Data Import Flow

**Goal:** Build the upload page with drag-and-drop, progress indication, and import feedback.

### Upload Page

`src/app/(app)/upload/page.tsx`:
- Drag-and-drop zone accepting `.pcfg` and `.pdat` files
- File type validation (must be ZIP with correct internal structure)
- Upload progress bar
- Processing status (extracting → parsing → storing → done)
- Error display with details
- History of past uploads with status badges

### Upload Flow (Client)

1. User drops files onto zone
2. Client validates file extensions
3. `fetch('/api/upload', { method: 'POST', body: formData })`
4. Stream progress via response or poll upload status via tRPC
5. On success, redirect to dashboard or show success toast

### Upload Flow (Server)

1. Validate auth session
2. Read multipart files into buffers
3. Create `uploads` record (status: "processing")
4. Run ingest pipeline (Phase 2)
5. Update upload status to "complete" or "error"
6. Return result

### Components

| Component | Purpose |
|-----------|---------|
| `upload-dropzone.tsx` | Drag-and-drop file input |
| `upload-progress.tsx` | Progress/status display |
| `upload-history.tsx` | Table of past uploads |

### Verification

- Upload both `.pcfg` and `.pdat` from `/tmp/cpap/`
- Check Drizzle Studio for populated tables
- Re-upload same device → data merges (upsert, not duplicate)
- Upload invalid file → clear error message

---

## Phase 5 — Dashboard & Charts

**Goal:** Build the main dashboard with summary cards and time-series charts.

### Charting Library

Install **Recharts** (React-native, good Tailwind integration, well-maintained):

```bash
pnpm add recharts
```

### tRPC Routers

`src/server/api/routers/dashboard.ts`:
- `dashboard.summary` — Returns device info, night count, avg AHI, total therapy hours
- `dashboard.ahiTrend` — AHI per night for chart
- `dashboard.usageTrend` — Therapy duration per night
- `dashboard.leakTrend` — Leak 95th per night
- `dashboard.pressureTrend` — Pressure median + 95th per night

All queries scoped to the authenticated user's devices.

### Dashboard Page

`src/app/(app)/dashboard/page.tsx`:

**Summary Cards Row:**
- Device name + serial
- Nights recorded (date range)
- Average AHI (color-coded)
- Total therapy time
- Pressure range (min–max from config)
- Average leak

**Charts (Recharts `ResponsiveContainer`):**
1. **AHI Trend** — Bar chart, color-coded by severity threshold (green <5, orange 5-15, red >15)
2. **Therapy Duration** — Bar chart with hours per night
3. **Pressure Trend** — Line chart with median + 95th percentile bands
4. **Leak Trend** — Line chart with 95th percentile

Each chart is a client component (`"use client"`) fetching data via tRPC + React Query.

### Components

| Component | Purpose |
|-----------|---------|
| `stat-card.tsx` | Reusable summary card |
| `ahi-chart.tsx` | AHI bar chart |
| `usage-chart.tsx` | Duration bar chart |
| `pressure-chart.tsx` | Pressure line chart |
| `leak-chart.tsx` | Leak line chart |

### Verification

- Dashboard loads with real data after upload
- Charts render correctly with 6 nights of data
- Responsive layout (cards stack on mobile, charts resize)
- Loading skeletons show while data fetches

---

## Phase 6 — Night Detail View

**Goal:** Build the per-night detail page with session breakdown, event timeline, and statistics.

### tRPC Routers

`src/server/api/routers/night.ts`:
- `night.list` — All nights for user's device(s), sorted by date
- `night.getById` — Night + stats + sessions with event counts
- `night.getSessionEvents` — All resp events for a session (paginated)

### Night List Page

`src/app/(app)/nights/page.tsx`:
- Table/card list of all nights
- Columns: Date, AHI, Therapy Time, Leak, Pressure, Event Counts
- Click to navigate to detail

### Night Detail Page

`src/app/(app)/nights/[id]/page.tsx`:

**Stats Summary Bar:**
- AHI, AI, HI (color-coded badges)
- Therapy time, usage time
- Pressure median / 95th
- Leak 95th / median
- Snoring duration
- OA / CA / Hypopnea counts

**Session Cards:**
- Expandable cards for each session file
- Header: filename, duration badge, event count badges
- Expanded: event timeline

**Event Timeline:**
- Vertical timeline component
- Color-coded dots by event type (red=apnea, orange=leak, purple=flow limitation, yellow=snoring)
- Time offset from session start
- Duration and severity for each event

**Session Statistics (from summary events 1230-1238):**
- Mini stat row per session

### Therapy Segment Visualization

A horizontal bar showing the night's therapy on/off periods parsed from the statistics `t=` attribute (timestamp pairs). Shows when therapy was active vs. mask was off.

### Components

| Component | Purpose |
|-----------|---------|
| `night-stats-bar.tsx` | Summary stat grid |
| `session-card.tsx` | Expandable session details |
| `event-timeline.tsx` | Vertical event timeline |
| `therapy-segments.tsx` | Horizontal on/off bar |

### Verification

- Navigate from night list → night detail
- All 6 nights display correctly
- Events render in chronological order
- Large sessions (65+ events) perform well

---

## Phase 7 — Statistics, Config & Logs Pages

**Goal:** Build the remaining data browsing pages.

### Statistics Page

`src/app/(app)/statistics/page.tsx`:

- Sortable data table with all nightly statistics
- Columns from `nightStats` table (AHI, therapy time, leak, pressure, event counts, ventilation metrics)
- Export to CSV button
- Optional: sparkline mini-charts in table cells

tRPC: `stats.getAll` — All night stats for user's devices

### Device & Config Page

`src/app/(app)/device/page.tsx`:

- Device info card (from `devices` table)
- Two-section config table: OBL (prescribed) and OPT (user)
- Each row: Parameter name, ID, raw value, interpreted value
- Interpretation logic reuses the parameter name/unit mappings from Phase 2 parsers

tRPC:
- `device.get` — Device info
- `device.getConfig` — All config params with sections

### Signal Metadata Page

`src/app/(app)/signals/page.tsx`:

- Grouped by night
- Table: filename, start time, channel count, channel list, file size
- Info callout: "Full waveform visualization requires OSCAR"

tRPC: `signal.listByNight` — Signal metadata grouped by night

### Log Viewer Page

`src/app/(app)/logs/page.tsx`:

- Tab bar for each log type (therapy-sw, kernel, connman, develop, service)
- Monospace scrollable log display
- Optional: basic search/filter within log text (client-side)

tRPC: `logs.get(logType)` — Returns log content

### Verification

- All pages render with real data
- Statistics table sorts correctly
- Config shows interpreted pressure values (cmH2O)
- Logs display with proper monospace formatting
- `pnpm build` succeeds (no SSR errors)

---

## Phase 8 — Polish & Production Readiness

**Goal:** Final polish, error handling, loading states, and deployment prep.

### Error Handling

- tRPC error boundaries on each page
- Upload validation errors with user-friendly messages
- Empty states for users with no uploads yet
- 404 page for invalid night/session IDs

### Loading States

- Skeleton components for all data pages
- Suspense boundaries around data-fetching components
- Upload progress with percentage

### Responsive Design

- Test all pages at mobile, tablet, desktop breakpoints
- Sidebar → bottom nav or hamburger on mobile
- Tables → card views on narrow screens

### Performance

- React Query caching (staleTime, gcTime tuning)
- tRPC query prefetching on navigation (RSC where possible)
- Index optimization on frequently-queried columns

### Multi-Device Support

- Device selector dropdown if user has uploaded data from multiple machines
- All queries filter by selected device

### Security

- All tRPC routes use `protectedProcedure`
- Upload endpoint checks auth
- File type validation (reject non-ZIP files)
- Max upload size enforcement
- SQL injection prevention (Drizzle parameterized queries — automatic)
- No sensitive data in client bundles

### Deployment

- Dockerfile or Vercel config
- Database migration strategy (Drizzle push or migrate)
- Environment variable documentation
- Build verification: `pnpm build && pnpm start`

---

## Dependency Summary

| Package | Purpose | Phase |
|---------|---------|-------|
| `jszip` | ZIP extraction | 2 |
| `fast-xml-parser` | XML parsing | 2 |
| `next-themes` | Dark mode | 3 |
| `shadcn/ui` | Component library | 3 |
| `recharts` | Charts | 5 |

---

## File Structure (Final)

```
src/
├── app/
│   ├── (auth)/
│   │   ├── sign-in/page.tsx
│   │   └── sign-up/page.tsx
│   ├── (app)/
│   │   ├── layout.tsx              # Authed layout w/ sidebar
│   │   ├── dashboard/page.tsx
│   │   ├── upload/page.tsx
│   │   ├── nights/
│   │   │   ├── page.tsx            # Night list
│   │   │   └── [id]/page.tsx       # Night detail
│   │   ├── statistics/page.tsx
│   │   ├── signals/page.tsx
│   │   ├── device/page.tsx
│   │   └── logs/page.tsx
│   ├── api/
│   │   ├── auth/[...all]/route.ts
│   │   ├── trpc/[trpc]/route.ts
│   │   └── upload/route.ts
│   ├── _components/
│   │   ├── sidebar.tsx
│   │   ├── topbar.tsx
│   │   ├── stat-card.tsx
│   │   ├── ahi-badge.tsx
│   │   ├── pressure-display.tsx
│   │   ├── duration-display.tsx
│   │   ├── event-timeline.tsx
│   │   ├── therapy-segments.tsx
│   │   ├── upload-dropzone.tsx
│   │   ├── charts/
│   │   │   ├── ahi-chart.tsx
│   │   │   ├── usage-chart.tsx
│   │   │   ├── pressure-chart.tsx
│   │   │   └── leak-chart.tsx
│   │   └── empty-state.tsx
│   ├── layout.tsx
│   └── page.tsx                    # Landing → redirect
├── lib/
│   └── parsers/
│       ├── index.ts                # Orchestrator
│       ├── ingest.ts               # DB ingest pipeline
│       ├── extract-zip.ts
│       ├── parse-device-xml.ts
│       ├── parse-configuration-xml.ts
│       ├── parse-event-xml.ts
│       ├── parse-signal-header.ts
│       ├── parse-statistics-xml.ts
│       ├── parse-ini.ts
│       ├── parse-trend-header.ts
│       └── constants.ts            # PARAM_NAMES, RESP_EVENT_NAMES, STAT_NAMES
├── server/
│   ├── api/
│   │   ├── root.ts
│   │   ├── trpc.ts
│   │   └── routers/
│   │       ├── dashboard.ts
│   │       ├── night.ts
│   │       ├── stats.ts
│   │       ├── device.ts
│   │       ├── signal.ts
│   │       ├── logs.ts
│   │       └── upload.ts
│   ├── better-auth/
│   │   ├── config.ts
│   │   ├── index.ts
│   │   ├── server.ts
│   │   └── client.ts
│   └── db/
│       ├── index.ts
│       └── schema.ts
├── trpc/
│   ├── react.tsx
│   ├── server.ts
│   └── query-client.ts
├── styles/
│   └── globals.css
└── env.js
```

---

## Execution Order

| Phase | Dependency | Estimated Scope |
|-------|-----------|-----------------|
| 1 — Schema | None | schema.ts (~300 lines) |
| 2 — Parsers | Phase 1 | ~10 files, ~600 lines |
| 3 — UI Shell | None (parallel with 1-2) | Layout, nav, auth pages |
| 4 — Upload UI | Phases 1, 2, 3 | Upload page + API route |
| 5 — Dashboard | Phases 1, 2, 4 | Dashboard + 4 charts |
| 6 — Night Detail | Phases 1, 2, 5 | Night list + detail pages |
| 7 — Remaining Pages | Phases 1, 2 | Stats, config, signals, logs |
| 8 — Polish | All above | Error handling, responsive, deploy |

Phases 1+2 and 3 can be worked in parallel since they don't overlap in files.
