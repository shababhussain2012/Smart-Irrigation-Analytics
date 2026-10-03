# Smart Irrigation Analytics

Cleaning and analyzing messy IoT sensor data from a simulated smart-irrigation deployment, then building an interactive Power BI dashboard to quantify water waste and support sustainable irrigation decisions.

## Overview

Al Bustan Research Farm (a simulated case study) runs four open-field growing zones tomato, cucumber, date palm, and lettuce in an arid climate where water is scarce and expensive. As part of a digital sustainability initiative, the farm replaced fixed-schedule irrigation timers with a 12-node IoT sensor network, logging soil moisture, temperature, humidity, light, and irrigation status every 4 hours.

This project takes one month of raw, messy sensor exports and turns them into a clean dataset, a set of verified operational findings, and a dashboard a farm manager could actually act on.

## Objectives

- Build a clean, analysis-ready dataset from the raw sensor feed, with every transformation documented
- Profile sensor reliability missing/duplicate readings by node, and whether that tracks battery level
- Flag zones and periods of under- and over-irrigation against each crop's optimal soil-moisture band, and estimate the water impact
- Examine how temperature, humidity, and light relate to how fast soil dries out
- Deliver a Power BI dashboard covering sensor uptime, at-risk zones, irrigation adherence, and estimated water savings

## Data

| File | Description |
|---|---|
| `data/raw/sensor_readings_raw.csv` | 2,277 readings from 12 sensors (3 per zone × 4 zones), 1–31 May 2026, every 4 hours |
| `data/raw/sensor_reference.csv` | Sensor-to-zone mapping for all 12 physical sensor nodes |
| `data/raw/field_zone_reference.csv` | Zone master data crop type, planting date, area, irrigation type, optimal soil-moisture band |
| `data/cleaned/sensor_readings_clean.csv` | 2,232 rows deduplicated, unit-corrected, outlier-nulled, categories standardized. Loads directly into Power BI |

Join path: `sensor_readings_clean` (sensor + timestamp grain) → `sensor_reference` (sensor grain) → `field_zone_reference` (zone grain).

*Synthetic dataset generated to mirror a realistic smart-agriculture IoT deployment, built for portfolio purposes not from a real farm.*

**Columns in `sensor_readings_clean.csv`:** Reading _ID, Timestamp (ISO 8601 comes from python step), Timestamp_clean (comes from raw data), Date, Time, Day, Sensor_ID, Field_Zone, Soil_Moisture (%), Soil_Temp, Air_Temp (°C), Humidity (%), Light_Lux, Battery_Level (%), Battery_Level_Status, Rainfall_mm, Irrigation_On (boolean).

**Key Columns in `field_zone_reference.csv`:** Optimal_Soil_Moisture_Min / Max same 0–100% scale as Soil_Moisture, stored as plain numbers since the unit is documented here rather than embedded in each value. Zone C's max (50) was not present in the original export and was inferred from the min–max spread of the other three zones (all exactly 20), then independently confirmed against Zone C's own non-drought readings, which cluster at 44–51%.

**Note on grain:** Soil_Moisture and Battery_Level are genuine, independent per-sensor readings. Soil_Temp varies per sensor but is derived from a shared zone-level air temperature baseline, so the 3 sensors in a zone track closely without being identical. Humidity, Light_Lux, Air_Temp (pre-conversion), Rainfall_mm, and Irrigation_On are zone-level conditions replicated across a zone's 3 sensors expected, not a cleaning artifact.

### Data Quality Issues Identified and Resolved

- Timestamps arrived in 5 different formats plus blanks parsed with an explicit per-format function (checked in a specific order, since one ISO format is a substring of another) and standardized to true ISO 8601
- Sensor and zone names were inconsistently cased, spaced, underscored, or abbreviated (` SN-A01` / `sn_a01` / `SNA01`) — normalized via a strip-uppercase-reformat function; verified against an exact count of 12 sensors and 4 zones
- Missing values were coded 5 different ways (blank, NA, N/A, null, and a -999 sentinel) the -999 sentinel is a genuine trap: it parses as a valid number and will silently corrupt an average if not explicitly excluded before aggregating (confirmed from raw data file directly Zone C's May 13 daily mean swings from a naive −114.8% to a corrected 11.5% depending on whether -999 values are filtered out first) 
- A batch of 2 sensors (SN-D01, SN-D02) reported temperature in Fahrenheit while the other 10, including a third sensor in the same zone, reported in Celsius identified via a per-sensor median check, converted, and re-verified (sensor-to-sensor spread dropped from ~55°C to ~1°C after correction)
- Physically impossible outliers (negative percentages, values over 100%, temperatures like −999 or 999) were nulled in place rather than dropping the row e.g. of the 83 rows with an out-of-range Soil_Moisture value, the large majority still carry a perfectly good Battery_Level reading on the same row, which a row-level filter would have discarded along with the one bad value
- Irrigation status was logged 5 different ways (`ON`/`On`/`1`/`Yes`/`TRUE` and their `OFF` equivalents) mapped to a single boolean with zero unmapped values
- 45 duplicate log entries were identified and removed, verified against `Reading_ID` specifically rather than a full-row comparison (a full-column comparison produces false positives when two genuinely different rows happen to share the same missing values in the same fields)
- 8 partial "heartbeat" records (timestamp and sensor ID present, entire environmental payload blank) were identified and accounted for in the missing-value counts
- The zone reference file's key didn't match the main file's zone naming (`Zone_D` vs `Zone D`) reconciled before any join

Full breakdown of every issue's exact count is in `excel/audit_findings.md`.

## Methodology

| Stage | Tool | What was done |
|---|---|---|
| Data audit | Excel (Power Query) | Built Missing/Present and Outlier/Normal flag columns for all 9 raw fields using a disposable grouping key (casing/spacing normalized for counting only the real columns stayed untouched until the Python stage), then aggregated by sensor and by zone into `audit_findings.md` |
| Cleaning & feature engineering | Python (pandas) | A 8-cell notebook: parsed the 5 timestamp formats to ISO 8601, cleaned Sensor_ID/Field_Zone, corrected the Fahrenheit mismatch, standardized 5 different missing-value tokens, nulled impossible values in place (never dropped rows), removed duplicates by Reading_ID, standardized Irrigation_On to boolean, then exported the verified result. Every cell asserts its output against a number already confirmed in the Excel audit before moving to the next |
| Storage & querying | *(not used — see note below)* | |
| Dashboard & reporting | Power BI | Built a relational model (readings → sensor reference → zone reference) via Model view relationships rather than a flat table; two calculated columns (`Irrigation_Status_Flag`, `Over_Irrigation_Cause`) evaluate each reading against its own zone's optimal band and separate genuine equipment-driven over-irrigation from a shared rainfall response; 4 main DAX measures power the KPI cards; interactive Zone and Date-range slicers connect every visual. 1 DAX is not added to visuals leaved by choice Missing_Rate_By_Battery which makes the total measures 8.

**Note on SQL:** the project plan originally included a MySQL loading stage. In practice, the CSV export couldn't represent two SQL-native concepts (a true blank vs. an empty string, and a real boolean vs. the text "True"/"False"), which caused MySQL's Table Data Import Wizard to reject a large share of rows outright with no clear warning why. The underlying fix (an explicit `LOAD DATA INFILE` statement with `NULLIF()` handling) was identified and verified working against a live test server, but building the full analysis on top of a tool fight that had nothing to do with the actual data or analysis wasn't a good use of the time available before applying to roles. The join logic that stage would have handled was moved into Power BI's relationship model instead, which does the equivalent work natively.

## Key Findings

- **Zone C experienced a clear 4-day drought event.** Daily average soil moisture fell from a healthy 44–51% to as low as 4.2% between May 11–14, traced to irrigation staying off through the whole window rather than running on its normal schedule. The event's tail extended one day further than the main window 8 additional Under-Irrigated readings occurred on May 15 as moisture continued recovering. This fully explains Zone C's Under-Irrigated count (53 of 53 farm-wide: 43 readings on May 11–14, 8 on May 15, 2 more comes from the blank timestamp).
- **Every zone showed some over-irrigation, not just one.** 284 readings (12.7% of valid readings) exceeded their zone's optimal moisture ceiling. Splitting by cause (via `Rainfall_mm`) shows 156 are explained by genuine rain events shared across zones, and 128 are not. A handful of single days account for most of the total, and comparing them individually (rather than grouping unequal date ranges together) is the fairer read: May 7 is the single largest day (58 readings, 54 of them rain-driven a single large storm), followed by May 25 (55), May 18 (50), May 8 (37, entirely irrigation-driven with no rain recorded that day), May 19 (34), and May 26 (30). Zone A specifically shows a 4-day window (May 18–21, 28 readings: 15 irrigation-driven, 13 rain event) consistent with irrigation running far more often than its scheduled 2×/day during that period.
- **An estimated 10.5 million liters of excess water were applied during over-irrigation events**, using each affected zone's actual field area and an assumed drip-irrigation flow rate of 3 L/hour/m² (a stated assumption, not a measured value flagged directly on the dashboard). Zone C alone accounts for an estimated 4.2 million liters of that total despite having fewer flagged readings than Zone A, because its field area (5,000 m²) is more than double the others'.
- **Sensor reliability is uneven and tracks battery level closely.** This isn't a fixed trait of certain sensors it's a condition any sensor drifts into as its battery runs down. Pooling every reading across all 12 sensors, the missing-reading rate at the moment a sensor's battery is below 15% runs roughly 30.7%, against about 6.3% when battery is adequate an 8× jump, tied to the sensor's state at that specific reading, not a monthly average (individual sensors' month-long uptime stays between 84.4% and 94.6%, since most of any sensor's month is spent with healthy battery). The rate of physically-impossible readings barely moves with battery level, pointing to two separate problems: missing data (fixed by recharging/replacing the battery) and garbage data (likely a probe calibration issue, unrelated to power). SN-A01 has the lowest overall uptime (84.4%) of the 12 sensors; SN-C01 the highest (94.6%).
- **Temperature, humidity, and light showed only a weak relationship with how fast soil dried out**, checked three separate ways (same-interval, lagged, and zone-specific correlations all stayed under 0.15 in magnitude once a genuine confound light and moisture both moving with the fixed reading schedule rather than with each other was identified and excluded). In this dataset, drying speed is driven far more by the irrigation schedule and equipment reliability than by ambient conditions, which points a farm manager toward fixing valves and monitoring batteries rather than building a weather-responsive irrigation model.

## Dashboard Preview
<img width="4100" height="2350" alt="dashboard_preview" src="https://github.com/user-attachments/assets/8115bfb1-0d1f-4678-8a7c-0b804f918ed9" />



## How to Reproduce

1. Clone this repo
2. Raw data: `data/raw/`
3. `excel/data_audit.xlsx` — Power Query audit (open in Excel; findings summarized in `excel/audit_findings.md`)
4. `notebooks/01_data_cleaning_and_eda.py` — full cleaning pipeline → outputs `data/cleaned/sensor_readings_clean.csv`
5. `powerbi/Smart_Irrigation_Dashboard.pbix` in Power BI Desktop

## Repo Structure

```
smart-irrigation-analytics/
├── data/
│   ├── raw/
│   │   ├── sensor_readings_raw.csv
│   │   ├── sensor_reference.csv
│   │   └── field_zone_reference.csv
│   └── cleaned/
│       └── sensor_readings_clean.csv
├── excel/
│   ├── data_audit.xlsx
│   └── audit_findings.md
├── notebooks/
│   └── 01_data_cleaning_and_eda.py
└── powerbi/
    ├── Smart_Irrigation_Dashboard.pbix
    └── dashboard_preview.png
├── .gitignore
├── LICENSE
├── README.md
```

## About

Built by Syed Shabab Hussain as a portfolio project connecting a background in plant science with data analytics.
[LinkedIn](https://www.linkedin.com/in/syed-shabab-hussain-898596252/?skipRedirect=true) · [Medium](https://medium.com/@shababhussain2012)
