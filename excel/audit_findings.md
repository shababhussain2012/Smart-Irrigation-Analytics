# Excel Audit Findings — Raw Sensor Data

Output of the Power Query audit pass in `excel/data_audit.xlsx`. This is a count of
data-quality issues in the raw file nothing here has been cleaned or corrected yet.
See the notebook in `notebooks/` for how each issue was actually resolved.

## Duplicate entries

Reading_ID should be unique per row (one log entry, one ID). It isn't.

| Metric | Count |
|---|---|
| Total rows | 2,277 |
| Distinct Reading_ID values | 2,232 |
| Reading_IDs appearing exactly once ("Unique") | 2,187 |
| Reading_IDs appearing more than once | 45 |
| **Duplicate rows (Total − Distinct)** | **45** |

**Method:** Power Query's Distinct and Unique counts aren't the same measurement
Distinct collapses repeats to one, Unique counts only values with zero repeats. The
gap between them is how many Reading_IDs repeat. Verified: every duplicate found is a
full exact-row repeat (same Timestamp, Sensor_ID, and every reading value), not a
partial match.

## Missing values by column

| Column | Missing Count | Missing % |
|---|---|---|
| Timestamp | 69 | 3 |
| Soil_Moisture | 184 | 8 |
| Soil_Temp | 164 | 7 |
| Air_Temp | 166 | 7 |
| Humidity | 166 | 7 |
| Light_Lux | 188 | 8 |
| Battery_Level | 40 | 2 |
| Rainfall_mm | 160 | 7 |
| Irrigation_Status | 91 | 4 |

Note: 8 rows are partial "heartbeat" pings Timestamp and Sensor_ID transmitted, but
every environmental payload field arrived blank.

## Missing + outlier counts by sensor

Counted using a temporary grouping key (casing/spacing normalized for counting purposes
only the real Sensor_ID column is untouched, actual cleaning happens in Python).

| Sensor_ID | Missing (any column) | Outliers (any column) |
|---|---|---|
| SN-A01 | 140| 12|
| SN-A02 | 127| 11|
| SN-A03 | 63| 10|
| SN-B01 | 103| 14|
| SN-B02 | 86| 5|
| SN-B03 | 91| 13|
| SN-C01 | 78| 10|
| SN-C02 | 96| 12|
| SN-C03 | 93| 17|
| SN-D01 | 86| 368|
| SN-D02 | 126| 358|
| SN-D03 | 139| 7|

## Missing + outlier counts by zone

Same approach temporary grouping key for Field_Zone, real column untouched.

| Field_Zone | Missing (any column) | Outliers (any column) |
|---|---|---|
| Zone A | 330| 33|
| Zone B | 280| 32|
| Zone C | 267| 39|
| Zone D | 351| 733|

## Observations worth investigating further

- Battery_Level: 49 rows read the text "LOW" instead of a number not counted as
  Missing or as Outlier above (tracked separately), but a real data-quality issue on
  its own worth a mention.
- Two sensors (SN-D01 & SN-D02) show outlier counts far above the rest of the network worth checking
  during the Python stage whether this traces back to the temperature-unit mismatch
  noted in the main README's known issues.
- Convert timestamp to iso 8601 format in the python stage.
