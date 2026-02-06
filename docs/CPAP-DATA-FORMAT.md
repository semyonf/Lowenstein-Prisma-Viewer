# Lowenstein Prisma CPAP — Data Archive Reference

This document describes the internal structure and data formats found in Lowenstein (formerly Weinmann) Prisma series CPAP machine data archives. All findings are based on direct inspection of exported `.pcfg` and `.pdat` files from a Prisma device.

---

## Table of Contents

- [Overview](#overview)
- [Archive Structure](#archive-structure)
- [Device Identification — device.xml](#device-identification)
- [Configuration — configuration.xml](#configuration)
  - [Parameter ID Reference](#parameter-id-reference)
  - [Pressure Value Convention](#pressure-value-convention)
- [Therapy Events — event\_\*.xml](#therapy-events)
  - [DeviceEvent Elements](#deviceevent-elements)
  - [RespEvent Elements](#respevent-elements)
  - [RespEventID Reference](#respeventid-reference)
  - [Session Lifecycle](#session-lifecycle)
  - [Computing Clinical Indices](#computing-clinical-indices-from-events)
- [Signal Files — \*.wmedf](#signal-files-wmedf)
  - [EDF Header Layout](#edf-header-layout)
  - [Channel List](#channel-list)
- [Trend Curves — trendCurves.tc](#trend-curves)
- [Annual Statistics — statistics\_year.bin](#annual-statistics)
  - [Stat ID Reference](#stat-id-reference)
- [INI Configuration Files](#ini-configuration-files)
- [Debug Logs](#debug-logs)
- [Tools and Compatibility](#tools-and-compatibility)

---

## Overview

The Lowenstein Prisma series (including Prisma 20A, 25S, SMART, SMART Max, etc.) exports therapy data via SD card or USB as two archive files:

| File | Extension | Contents |
|------|-----------|----------|
| Configuration archive | `.pcfg` | Device identity and prescribed/user settings |
| Therapy data archive | `.pdat` | Therapy sessions, signals, statistics, logs |

Both files are **standard ZIP archives** (PK header, deflate compression) with custom extensions. They can be extracted with any ZIP tool (`unzip`, 7-Zip, etc.).

### Example dataset

| Property | Example Value |
|----------|---------------|
| Device Type | 10 (Prisma series) |
| Serial Number | `XXXXXXXX` |
| Firmware | 5.07 r0002, build "Eyra" (2023-03-10) |
| Branding | 2 (Lowenstein-branded) |
| Data range | 6 consecutive nights |
| Total sessions | ~66 |

---

## Archive Structure

Both archives mirror the device's internal flash filesystem rooted at `mnt/flash/`.

### config.pcfg

```
mnt/flash/
  conf/
    device.xml              # Device hardware/firmware identification
    configuration.xml       # Therapy settings (prescribed + user)
```

### therapy.pdat

```
mnt/flash/
  conf/
    device.xml              # Duplicate of device identity
    config_notifications.ini    # Reminder schedules
    config_operatingtime.ini    # Boot/therapy counters
  data/
    therapy/
      therapy_hash.txt          # Integrity hash
      events/
        YYYYMMDD/
          event_NNNNNN.xml      # Per-session therapy events
      signals/
        YYYYMMDD/
          signal_NNNNNN.wmedf   # Per-session waveform data (EDF)
      trendcurves/
        YYYYMMDD/
          trendCurves.tc        # Nightly trend summary
    statistics/
      statistics_year.bin       # Annual statistics (XML format despite .bin)
    debug/
      therapy-sw.log            # Therapy software log
      therapy-sw1.log           # Previous therapy software log (rotated)
      kern.log                  # Linux kernel log
      connman.log               # Network connection manager log
      develop.log               # Developer/debug log
      service.log               # Firmware update/service log
      dosfsck.txt               # Filesystem check output
    maintenance/
      service.log               # Maintenance service log
      mc_calibr.bin             # Motor controller calibration (4 bytes)
```

Events and signals are organized by **therapy date** using `YYYYMMDD` directories. Each therapy session produces a matching pair: `event_NNNNNN.xml` and `signal_NNNNNN.wmedf` with the same sequence number.

> **Noon cutoff date convention:** The device uses a **noon-to-noon** day boundary. Therapy sessions starting before noon are attributed to the previous calendar day's "night." For example, a session starting at 01:00 on Feb 5th belongs to the Feb 4th night directory. This matches how the device display groups nights.

---

## Device Identification

**File:** `mnt/flash/conf/device.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<?weinmann version="1.1" type="dde-tpg"?>
<DeviceID>
  <DeviceType value="10"/>
  <DeviceSerialNumber value="XXXXXXXX"/>
  <MainboardSerialNumber value="XXXXXXXXX"/>
  <BlowerIndex value="0"/>
  <FWVersion value="5.07"/>
  <FWRevision value="0002"/>
  <FWBuild value="2023-0310-1825-Eyra"/>
  <FWGITHASH value="xxxxxxx"/>
  <KernelGITHASH value="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"/>
  <BuildrootGITHASH value="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"/>
  <PMVersion value="2.18.3"/>
  <StatisticVersion value="2.4.2"/>
  <NotificationVersion value="00.00.01"/>
  <LoggerVersion value="00.01.02"/>
  <MCVersion value="xxxxxx.1.0"/>
  <MainboardHWVersion value="11"/>
  <DisplayHWVersion value="24"/>
  <DeviceVariant value="0"/>
  <DeviceBranding value="2"/>
</DeviceID>
```

Key observations:
- The `<?weinmann ...?>` processing instruction identifies this as a Weinmann/Lowenstein device
- `type="dde-tpg"` indicates the device data export type
- The device runs an embedded Linux system (Buildroot-based) with git-tracked firmware
- `PMVersion` corresponds to the Parameter Management version used in `configuration.xml`

---

## Configuration

**File:** `mnt/flash/conf/configuration.xml`

```xml
<DCM MODUL_VERSION="2.18.3">
  <OBL>
    <P id="1001" val="20180003" />
    <P id="1138" val="550" />
    ...
  </OBL>
  <OPT>
    <P id="1007" val="1" />
    ...
  </OPT>
</DCM>
```

The configuration uses a numeric parameter ID system divided into two sections:

| Section | Tag | Description |
|---------|-----|-------------|
| **OBL** (Obligatory) | `<OBL>` | Prescribed therapy settings — set by clinician |
| **OPT** (Optional) | `<OPT>` | User-adjustable comfort and device settings |

### Pressure Value Convention

**All pressure values are stored in Pascals (Pa).** To convert to the clinical unit cmH2O:

```
cmH2O = Pa / 100
```

For example: `val="550"` = 5.50 cmH2O, `val="1600"` = 16.00 cmH2O.

Pressure support (PS) and ramp values also use Pa.

### Parameter ID Reference

#### Therapy Parameters (OBL)

| ID | Name | Example | Notes |
|----|------|---------|-------|
| 1001 | Device Code | 20180003 | Internal product code |
| 1002 | Device Type | 10 | Matches DeviceType in device.xml |
| 1003 | Therapy Mode | 2 | 0=CPAP, 1=APAP, 2=APAP+, 3=BiLevel, 4=ASV, 5=iVAPS |
| 1005 | Autostart | 1 | 0=off, 1=on |
| 1011 | Humidifier Level | 0 | 0=off, 1-5=level |
| 1012 | Heated Tube Temp | 0 | 0=off, degrees C otherwise |
| 1014 | Ramp Enabled | 1 | 0=off, 1=on |
| 1015 | Ramp Time | 3 | Minutes |
| 1016 | Ramp Start Pressure | 0 | Pa (0 = use min pressure) |
| 1017 | Ramp End Pressure | 0 | Pa (0 = use therapy pressure) |
| 1018 | CPAP Fixed Pressure | 550 | Pa — used in CPAP mode |
| 1084 | Autostart Enabled | 1 | 0=off, 1=on |
| 1123 | EPAP Mode | 3 | Auto-EPAP setting |
| 1124 | IPAP Mode | 0 | Fixed/auto IPAP |
| 1125 | EPAP Pressure | 450 | Pa — fixed EPAP in BiLevel modes |
| 1126 | IPAP Pressure | 450 | Pa — fixed IPAP in BiLevel modes |
| 1127 | PS Min | 20 | Pa — minimum pressure support |
| 1128 | PS Max | 20 | Pa — maximum pressure support |
| 1138 | Min Pressure | 550 | Pa — APAP lower bound |
| 1139 | Max Pressure | 1600 | Pa — APAP upper bound |
| 1140 | Min EPAP | 0 | Pa — BiLevel/ASV lower EPAP bound |
| 1141 | Max EPAP | 0 | Pa — BiLevel/ASV upper EPAP bound |
| 1154 | Trigger Sensitivity | 0 | 0=auto |
| 1156 | Cycle Sensitivity | 0 | 0=auto |
| 1157 | Rise Time | 0 | 0=auto |
| 1158 | I:E Ratio | 0 | 0=auto |
| 1160 | Auto-Titration | 1 | 1=enabled |
| 1162 | Leak Compensation | 1 | 1=enabled |
| 1199 | Max Therapy Pressure | 1600 | Pa — absolute pressure ceiling |
| 1200 | Start Pressure | 550 | Pa — therapy start pressure |
| 1201 | Current Pressure | 550 | Pa — last active pressure |
| 1203 | Exhalation Relief | 0 | 0=off |
| 1205 | Config Checksum | -XXXXXXXXX | Signed int32 CRC |
| 1209 | Therapy Sub-Mode | 2 | Mode variant |
| 1212 | PSV Min | 0 | Pa |
| 1213 | PSV Max | 0 | Pa |
| 1214 | Target Ventilation Min | 500 | Unit TBD |
| 1215 | Target Ventilation Max | 4000 | Unit TBD |
| 1216 | Backup Rate | 0 | Breaths/min (0=off) |
| 1217 | Auto Backup Rate | 1 | 1=enabled |
| 1218 | Auto EPAP | 1 | 1=enabled |
| 1219 | Manual EPAP | 0 | Pa |
| 1220 | Manual IPAP | 0 | Pa |
| 1223 | Comfort Transition Time | 10 | Seconds |
| 1224 | Flow Rounding | 0 | 0=off |

#### User/Device Settings (OPT)

| ID | Name | Example | Notes |
|----|------|---------|-------|
| 1007 | Pressure Unit Display | 1 | 0=Pa, 1=cmH2O, 2=mbar |
| 1023 | Mask Type | 0 | 0=nasal, 1=full face, 2=nasal pillows |
| 1083 | Comfort Level | 3 | 1-3 scale |
| 1085 | Ramp Duration | 1200 | Seconds (20 minutes) |
| 1086 | EPR Level | 30 | Expiratory pressure relief (Pa) |
| 1087 | Pressure Unit | 2 | Internal unit code |
| 1088 | Language | 1 | Language index |
| 1089 | Brightness | 1 | Display brightness |
| 1090 | Contrast | 14 | Display contrast |
| 1091 | Volume | 22 | Alert volume (%) |
| 1092 | Display Orientation | 0 | 0=normal |
| 1093 | Screen Lock | 0 | 0=off |
| 1094 | Data Record Mode | 4 | Recording detail level |
| 1096 | SD Card Write | 1 | 1=enabled |
| 1097 | Bluetooth | 0 | 0=off |
| 1099 | Altitude Setting | 19 | Altitude index |
| 1100 | Altitude Value | — | Meters above sea level |
| 1101 | Display AHI | 0 | 0=off |
| 1102 | Softstart Mode | 3 | Comfort start mode |
| 1103 | Delay Start | 0 | 0=off |
| 1104 | Delay Start Time | 0 | Minutes |
| 1105 | Auto-Off Time | 8 | Minutes after mask removal |
| 1106 | Leak Alarm | 0 | 0=off |
| 1107 | Apnea Alarm Time | 24 | Seconds |
| 1108 | Mask Off Alert | 0 | 0=off |
| 1109 | SpO2 Alarm | 0 | 0=off |
| 1110 | Pulse Alarm | 2 | Threshold setting |
| 1111 | Display Theme | 6 | Color theme index |
| 1150 | Ti Min | 0 | 0=auto |
| 1161 | Tube Resistance | 45 | Resistance compensation value |
| 1202 | SoftPAP Enabled | 1 | 1=enabled |
| 1204 | Mask Fit Check | 0 | 0=off |
| 1206 | BiLevel Mode | 0 | 0=off |
| 1207 | ASV Mode | 0 | 0=off |
| 1208 | iVAPS Mode | 0 | 0=off |
| 1210 | Display Timeout | 7 | Seconds |
| 1211 | Alarm Volume | 1 | Volume level |
| 1225 | Data Export Format | 1 | Export version |

---

## Therapy Events

**Files:** `mnt/flash/data/therapy/events/YYYYMMDD/event_NNNNNN.xml`

Each file represents one continuous therapy segment (mask on to mask off, or a significant event boundary). A single night typically produces multiple event files.

```xml
<?xml version="1.0" encoding="utf-8"?>
<desc>
  <DeviceEvent DeviceEventID="0" Time="0" ParameterID="1003" NewValue="2"/>
  <RespEvent RespEventID="101" EndTime="15440" Duration="124" Pressure="0" Strength="5"/>
  ...
</desc>
```

### DeviceEvent Elements

Record device state changes and configuration snapshots.

| Attribute | Description |
|-----------|-------------|
| `DeviceEventID` | Event category: `0` = config snapshot, `1` = runtime state change |
| `Time` | Seconds from session start |
| `ParameterID` | References the same parameter ID system as configuration.xml |
| `NewValue` | The new parameter value |

**DeviceEventID=0** events at `Time="0"` capture the full therapy configuration active at session start — essentially a snapshot of all relevant parameters from `configuration.xml`.

**DeviceEventID=1** events record runtime changes:
- `ParameterID="257"` — Therapy state (0=starting)
- `ParameterID="271"` — Therapy phase (0=ramp, 2=therapy active, 3=standby)

### RespEvent Elements

Record respiratory and therapy events detected by the device algorithms.

| Attribute | Description |
|-----------|-------------|
| `RespEventID` | Event type identifier (see table below) |
| `EndTime` | **Deciseconds** from session start when the event ended |
| `Duration` | Event duration in **deciseconds** (divide by 10 for seconds) |
| `Pressure` | Therapy pressure at event time (Pa) — often 0 in logged data |
| `Strength` | Event severity/intensity (scale varies by event type) |
| `Visible` | Optional; `"0"` = hidden event not shown on device display |

> **Important: Duration and EndTime are in deciseconds (1/10th of a second).** Divide by 10 to get actual seconds. This was confirmed by cross-referencing epoch-derived percentages (e.g. deep sleep, snore) with device display values, and by checking that individual apnea durations fall in the clinically expected 10-30 second range.

The event start time can be calculated as: `StartTime = (EndTime - Duration) / 10` (seconds)

### RespEventID Reference

Event IDs confirmed via [OSCAR Prisma loader source code](https://gitlab.com/pholy/OSCAR-code/-/blob/master/oscar/SleepLib/loader_plugins/prisma_loader.h) and cross-validated against device display values across 6 nights of data.

#### Epoch Events (2-minute evaluation windows)

These represent the device's rolling 2-minute epoch assessments. Duration is the total time in deciseconds spent in that epoch state.

| ID | Name | Notes |
|----|------|-------|
| 1 | **Epoch: Severe Obstruction** | Percentage = `round(Duration/10 / therapy_seconds * 100)` |
| 2 | **Epoch: Mild Obstruction** | Same percentage calculation |
| 3 | **Epoch: Flow Limitation** | Same percentage calculation |
| 4 | **Epoch: Snore** | Same percentage calculation |
| 5 | **Epoch: Periodic Breathing** | Same percentage calculation |
| 261 | **Epoch: Deep Sleep** | Percentage matches device "Deep Sleep %" display |

#### Individual Respiratory Events

| ID | Name | Duration | Strength |
|----|------|----------|----------|
| 101 | **Obstructive Apnea (OA)** | Event duration (ds, ÷10 for s) | Severity |
| 102 | **Central Apnea (CA)** | Event duration (ds) | Severity |
| 103 | **Apnea (Leakage)** | Event duration (ds) | — |
| 105 | **Apnea (High Pressure)** | Event duration (ds) | — |
| 106 | **Apnea (Movement)** | Event duration (ds) | — |
| 111 | **Obstructive Hypopnea (OH)** | Event duration (ds) | Severity |
| 112 | **Central Hypopnea (CH)** | Event duration (ds) | Severity |
| 113 | **Hypopnea (Leakage)** | Event duration (ds) | — |
| 121 | **RERA** | Event duration (ds) | — |
| 131 | **Snore** | Episode duration (ds) | Intensity |
| 141 | **Artifact** | Duration (ds) | — |
| 151 | **Flow Limitation** | Duration (ds) | — |
| 161 | **Critical Leakage** | Duration (ds) | — |
| 171 | **Periodic Breathing** | Duration (ds) | — |
| 181 | **Cheyne-Stokes Respiration** | Duration (ds) | — |
| 221 | **Timed Breath** | Duration (ds) | — |

#### Session/Structure Markers

| ID | Name | Notes |
|----|------|-------|
| 231 | **Session Duration** | Duration = total session time (ds). Appears at session end |
| 241 | **Session End Marker** | Paired with 231 |
| 262 | **Pressure Change** | Marks pressure titration event |
| 306 | **Mask Off** | `Visible="0"` — internal tracking |
| 307 | **Mask On** | `Visible="0"` — internal tracking |
| 330 | **Large Leak** | Excessive mask leak detected |
| 1262 | **Therapy Pressure Change** | Pressure adjustment marker |

#### Per-Session Summary Flags

These appear at session boundaries. Despite the names suggesting AHI/AI/HI values, the Strength field contains **status flags** (typically 0 or 1), not usable index values. Do not use these for AHI computation.

| ID | Name | Notes |
|----|------|-------|
| 1230 | **Summary: AHI flag** | Status flag, not an AHI value |
| 1231 | **Summary: AI flag** | Status flag |
| 1232 | **Summary: HI flag** | Status flag |
| 1233 | **Summary: Leak95 flag** | Status flag |
| 1234 | **Summary: flag** | Status flag |
| 1237 | **Summary: PrMed flag** | Status flag |
| 1238 | **Summary: PrP95 flag** | Status flag |

### Session Lifecycle

A typical session file follows this pattern:

1. **DeviceEventID=0 block** — Configuration snapshot (all params at Time=0)
2. **DeviceEventID=1, ParameterID=257** — Therapy starts (NewValue=0)
3. **DeviceEventID=1, ParameterID=271** — Phase transitions (0→ramp, 3→standby, 2→active therapy)
4. **RespEventID=1230-1238** — Session summary flags (baseline)
5. **Epoch events (1-5, 261)** — Rolling 2-minute epoch assessments during therapy
6. **Clinical RespEvents** — Apneas (101/102), hypopneas (111/112), RERA (121), snore (131), flow limitations (151), critical leaks (161) during therapy
7. **RespEventID=262/1262** — Pressure change markers
8. **RespEventID=231** — Session total duration (deciseconds)
9. **RespEventID=1230-1238** — Final session summary flags
10. **RespEventID=241** — Session end marker

### Computing Clinical Indices from Events

The following formulas have been **confirmed across all 6 nights** of data against device display values:

```
therapy_hours = Therapy Time (s) / 3600    # from statistics stat 113

AIcent  = round(count_of_event_102 / therapy_hours)    # Central Apnea Index
HIcent  = round(count_of_event_112 / therapy_hours)    # Central Hypopnea Index
RERA/h  = round(count_of_event_121 / therapy_hours)    # RERA Index

Deep Sleep %  = round(epoch_261_duration_ds / 10 / therapy_seconds * 100)
Snore %       = round(epoch_4_duration_ds   / 10 / therapy_seconds * 100)
Flow Lim %    = round(epoch_3_duration_ds   / 10 / therapy_seconds * 100)
```

**Device composite indices (derived from independently rounded components):**
```
AHI incl. cH  = AHI + HIcent
AHI            = AHIobs + AIcent
```

> **Note:** The exact AHIobs formula remains unsolved. Exhaustive brute-force search of all combinations of event counts (OA, CA, OH, CH, RERA, MA — singles, pairs, and triples) divided by therapy hours with both `round()` and `floor()` did not produce a formula matching the device display for all 6 nights. The device likely uses additional internal logic (e.g., time-windowed calculations, minimum duration thresholds, or epoch-based scoring) not captured in the exported event data.

---

## Signal Files (WMEDF)

**Files:** `mnt/flash/data/therapy/signals/YYYYMMDD/signal_NNNNNN.wmedf`

These contain waveform data in **Weinmann Modified EDF** format — a variant of the European Data Format (EDF/EDF+) standard. Each signal file is paired 1:1 with its corresponding event file.

### EDF Header Layout

The main header occupies 256 bytes in standard EDF format:

| Offset | Length | Field | Example |
|--------|--------|-------|---------|
| 0 | 8 | Version | `1       ` (EDF version) |
| 8 | 80 | Patient ID | `Patient Name` (anonymized) |
| 88 | 80 | Recording ID | `Recording start at Sat, DD.MM.YYYY HH:MM:SS` |
| 168 | 8 | Start Date | `DD.MM.YY` (DD.MM.YY) |
| 176 | 8 | Start Time | `HH.MM.SS` (HH.MM.SS) |
| 184 | 8 | Header Size | `4864` (total header bytes = 256 + ns*256) |
| 192 | 44 | Reserved | `#s-1` followed by padding |
| 236 | 8 | Num Data Records | `-1` (continuous/unknown) |
| 244 | 8 | Record Duration | `1` (1 second per record) |
| 252 | 4 | Number of Signals | `18` |

Following the main header, each signal has a 256-byte header block (16 bytes per field * 16 fields) containing label, transducer type, physical dimension, physical min/max, digital min/max, prefiltering, and samples per record.

**Header reserved field `#s-1`**: The `-1` likely indicates streaming mode (continuous recording, total records unknown at start).

### Channel List

The 18 channels recorded per session:

| # | Channel Name | Description |
|---|-------------|-------------|
| 1 | **Pressure** | Delivered mask pressure |
| 2 | **EEPAPsoll** | Target EEPAP (effective EPAP) setpoint |
| 3 | **IPAPsoll** | Target IPAP setpoint |
| 4 | **EPAPsoll** | Target EPAP setpoint |
| 5 | **RespFlow** | Respiratory flow waveform |
| 6 | **rAMV** | Respiratory Alveolar Minute Ventilation |
| 7 | **BreathVolume** | Tidal volume per breath |
| 8 | **BreathFrequency** | Respiratory rate |
| 9 | **LeakFlowBreath** | Unintentional leak flow (per breath) |
| 10 | **ObstructLevel** | Obstruction detection level |
| 11 | **SpO2** | Blood oxygen saturation (requires oximeter) |
| 12 | **HeartFrequency** | Pulse rate (requires oximeter) |
| 13 | **SPRstatus** | SPR (Signal Processing Result) status flags |
| 14 | **InspExpirRel** | Inspiratory/Expiratory ratio |
| 15 | **MV** | Minute Ventilation |
| 16 | **rMVFluctuation** | Minute Ventilation fluctuation (variability) |
| 17 | **TotalLeakage** | Total leak (intentional + unintentional) |
| 18 | **RSBI** | Rapid Shallow Breathing Index |

File sizes range from ~5 KB (very short sessions/mask-off) to ~1.5 MB (full night sessions of several hours).

---

## Trend Curves

**Files:** `mnt/flash/data/therapy/trendcurves/YYYYMMDD/trendCurves.tc`

Each file begins with a JSON header followed by binary trend data:

```json
{"Type":"P3","SN":"00XXXXXXXX","devid":"010","Day":"DD.MM.YYYY","Format":"v1.0","Offset":"0030"}
```

| Field | Description |
|-------|-------------|
| Type | Device type code (`P3` = Prisma) |
| SN | Serial number (zero-padded to 10 digits) |
| devid | Device ID (`010` = device type 10) |
| Day | Recording date (DD.MM.YYYY) |
| Format | Data format version |
| Offset | Byte offset where binary data begins (hex) |

The binary payload following the JSON header contains per-minute or per-epoch trend summaries in a packed binary format. The exact binary structure requires further reverse engineering — the data appears to contain multi-byte integer values for pressure, flow, leak, and event counts sampled at regular intervals throughout the night.

---

## Annual Statistics

**File:** `mnt/flash/data/statistics/statistics_year.bin`

Despite the `.bin` extension, this file is **XML text**:

```xml
<stat u="NNNNNN" n1="NNNNNN" n2="NNNNN" n4="NNNN"
      cd="YYYY-MM-DD_HH:MM:SS" mv="2.4.2" v="3.2.1">
  <day d="YYYY-MM-DD" i="22">
    <rec m="2" v="12" t="46846-69,46933-3,...">
      <s i="100" v="0"/>
      <s i="101" v="4"/>
      ...
    </rec>
  </day>
  ...
</stat>
```

### Root `<stat>` attributes

| Attribute | Description |
|-----------|-------------|
| `u` | Total usage time (seconds) |
| `n1` | Total therapy time (seconds) |
| `n2` | Unknown counter |
| `n4` | Device operating time (minutes) |
| `cd` | Creation/export date |
| `mv` | Statistics module version |
| `v` | Data format version |

### `<day>` elements

| Attribute | Description |
|-----------|-------------|
| `d` | Date (YYYY-MM-DD) |
| `i` | Day index (sequential counter) |

### `<rec>` elements (one per day, or multiple if settings changed)

| Attribute | Description |
|-----------|-------------|
| `m` | Therapy mode (2=APAP+) |
| `v` | Record version |
| `t` | Timestamp pairs: `start-duration,start-duration,...` (seconds from midnight) representing therapy on/off segments |

### `<s>` stat entries

Each `<s>` element within a `<rec>` has:
- `i` — Stat ID (see table below)
- `v` — Value (integer, or comma-separated array for histogram data)

### Stat ID Reference

#### Event Counts (confirmed)

These stat IDs store **raw event counts** matching the corresponding RespEventIDs in event XML files. Verified by exact match across all 6 nights of data.

| Stat ID | Name | Corresponding Event |
|---------|------|---------------------|
| 100 | Central Apnea Count | RespEventID 102 |
| 101 | Obstructive Apnea Count | RespEventID 101 |
| 106 | RERA Count | RespEventID 121 |
| 107 | Central Hypopnea Count | RespEventID 112 |
| 108 | Obstructive Hypopnea Count | RespEventID 111 |

> **Caution:** Note that stat IDs do NOT directly match event IDs. Stat 100 = CA (event 102), stat 101 = OA (event 101), stat 106 = RERA (event 121), stat 107 = CH (event 112), stat 108 = OH (event 111). Earlier documentation (including older versions of this file) incorrectly labeled stats 106/107/108 as AHI/AI/HI indices x10 — this is wrong.

Per-hour indices are computed as: `index = round(count / therapy_hours)` where `therapy_hours = stat_113 / 3600`.

#### Timing

| ID | Name | Unit |
|----|------|------|
| 111 | Usage Time | seconds |
| 113 | Therapy Time | seconds |

#### Pressure

| ID | Name | Unit |
|----|------|------|
| 308 | Max Therapy Pressure | Pa |
| 309 | Min Therapy Pressure | Pa |

#### Other Stat IDs (unconfirmed — meanings inferred or speculative)

The remaining stat IDs appear in the data but their exact meanings have not been confirmed against device display. They are shown as `Stat_NNN` in the viewer. Tentative mappings from OSCAR and other sources:

| ID | Tentative Name | Unit |
|----|---------------|------|
| 102 | Mixed/Movement Apnea Count? | count |
| 104 | Total Hypopnea Count? | count |
| 109-115 | Various index values? | unknown |
| 116-121 | Leak/breathing metrics? | unknown |
| 200-212 | Breathing pattern stats? | unknown |
| 300-309 | Oximetry stats? | unknown |
| 400-418 | Ventilation stats? | unknown |

#### Histogram Data (comma-separated arrays)

These stat IDs contain comma-separated integer arrays representing histogram bin counts. Use bin-edge percentile calculation to extract P50, P95, etc.

| ID | Name | Start Value | Bin Step | Unit |
|----|------|-------------|----------|------|
| 1005 | Pressure Distribution | 4.0 | 0.5 | hPa (cmH2O) |
| 1013 | Breath Frequency Distribution | 11 | 1 | bpm |
| 1016 | Leak Flow Distribution | 0 | 2.5 | L/min |
| 1017 | Tidal Volume Distribution | 175 | 50 | mL |
| 1018 | Minute Ventilation Distribution | 0 | 1.0 | L/min |
| 1023 | Ti/T Ratio Distribution | 15 | 2 | % |

To compute a percentile from histogram data:
```
bins = [int(x) for x in csv_value.split(',')]
total = sum(bins)
target = total * (percentile / 100)
cumulative = 0
for i, count in enumerate(bins):
    cumulative += count
    if cumulative >= target:
        return start_value + i * bin_step
```

---

## INI Configuration Files

### config_operatingtime.ini

```ini
[OperatingTime]
bootCount=N
therapyCount=N
timeValue=N

[OperatingTimeSettingsVersion]
version=00.00.01
```

| Key | Description |
|-----|-------------|
| bootCount | Number of device power-on cycles |
| therapyCount | Number of therapy sessions started |
| timeValue | Total device operating time in **minutes** |

### config_notifications.ini

```ini
[NotificationSettingsVersion]
version=00.00.01

[changeFilter]
reminderExecutionDate=666-06-06
reminderExecutionInterval=6

[maintenance]
reminderExecutionDate=666-06-06
reminderExecutionInterval=2
```

| Section | Description |
|---------|-------------|
| changeFilter | Filter replacement reminder (interval in months) |
| maintenance | Device maintenance reminder (interval in months) |

The date `666-06-06` is a sentinel value indicating no reminder has been triggered yet.

---

## Debug Logs

The device maintains several rotating log files in `mnt/flash/data/debug/`. All logs follow a similar format:

```
YYMMDD HHMMSS.mmm| line# SourceFile.cpp         | thread|level: message
```

| Field | Description |
|-------|-------------|
| Timestamp | `YYMMDD HHMMSS.mmm` — date, time with milliseconds |
| Line | Source code line number |
| Source | C++ source file name |
| Thread | Thread/module ID |
| Level | `I`=Info, `D`=Debug, `W`=Warning, `E`=Error |
| Message | Log message (may contain ANSI color codes) |

### Log files

| File | Contents | Typical Size |
|------|----------|-------------|
| `therapy-sw.log` | Main therapy software — session management, SD card operations, compression, data export | ~600 KB |
| `therapy-sw1.log` | Rotated previous therapy-sw log | ~1.6 MB |
| `kern.log` | Linux kernel messages — USB, MTD, filesystem, hardware init | ~160 KB |
| `connman.log` | ConnMan network manager — WiFi/BT state | ~2 KB |
| `develop.log` | Developer debug — detailed therapy algorithm state | ~380 KB |
| `service.log` | Firmware update process — flashing, post-update cleanup | ~2 KB |
| `dosfsck.txt` | FAT filesystem consistency check output | ~83 bytes |

The `service.log` reveals the update process: firmware updates are applied from `/tmp/update/`, followed by cleanup of backup data. The embedded system uses MTD block devices (`/dev/mtdblock11`) for internal flash and mounts SD cards at `/mnt/SD/`.

---

## Tools and Compatibility

### OSCAR

[OSCAR (Open Source CPAP Analysis Reporter)](https://www.sleepfiles.com/OSCAR/) is the primary open-source tool for analyzing CPAP data. It supports Lowenstein Prisma devices and can read the `.pdat` and `.pcfg` archives directly or the extracted file tree.

### Included Tools

This dataset includes two tools generated during analysis:

| File | Description |
|------|-------------|
| `build_viewer.py` | Python 3 script that parses all extracted data and generates a self-contained HTML viewer |
| `viewer.html` | Generated single-file web application for browsing all device data, events, statistics, and logs |

To regenerate the viewer after re-extraction:

```bash
python3 build_viewer.py
open viewer.html
```

### Extracting the Archives

```bash
mkdir -p extracted/config extracted/therapy
unzip config.pcfg -d extracted/config
unzip therapy.pdat -d extracted/therapy
```
