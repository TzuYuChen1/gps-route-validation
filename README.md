# GPS Route Validation — School Transportation Data Quality & Route Execution Analysis

**An end-to-end GPS data validation and route execution scoring pipeline for school transportation**

📍 University of Minnesota · Carlson School of Management · MSBA Program · MSBA 6411 Exploratory Data Analytics · Live Case

---

## My Individual Contribution

While this is a collaborative team project with shared ownership across the pipeline, my primary contributions span data processing and stop-level scoring.

* **Stop Locations Cleaning:** Cleaned and standardized the stop locations dataset — handling missing coordinates, normalizing intersection addresses, mapping logical duplicates to a single `parent_stop_id`, engineering geofence radius fields, and extracting city/state/zip from raw address strings.
* **Route Stop–Location Join:** Built the join between cleaned route stops and stop locations on `stop_location_id`, producing the enriched route-stop table consumed by all downstream scoring notebooks.
* **Stop Service Reliability Score (SSRS):** Developed the SSRS for Type A stops (logged but not completed). Filtered to completed trips, computed per-stop completion rates, and produced `stop_level_diagnostic.csv` with per-stop reliability signals including stop type flags and average timing deviations.
* **Stop Score Integration:** Merged SSRS, timing outlier scores, and trip-level GPS and execution scores into a unified stop-level diagnostic table for downstream confidence scoring.
* **Presentation:** Contributed problem framing and the Stop Missing Score section of the final client presentation.

---

## Executive Summary

This project evaluates school transportation GPS and stop-event data on behalf of a transportation technology client to assess whether vehicles followed planned routes, whether scheduled stops were serviced, and how reliable the underlying GPS data is for route validation.

The analysis combines trip metadata, GPS ping streams, route stop definitions, scheduled stop times, and stop completion events into trip-level and stop-level quality scores. The final output is a confidence score for each trip that blends GPS data quality with route execution reliability — giving the client a systematic view of where data quality limitations end and genuine operational misses begin.

**Key deliverables:**

- End-to-end data processing pipeline across five core entities (trips, GPS positions, route stops, stop locations, stop times)
- Stop Service Reliability Score (SSRS) measuring per-stop completion rates
- Stop timing anomaly detection using rule-based scoring and Local Outlier Factor (LOF)
- Route execution scoring measuring planned stop completion rates
- GPS health scoring across five dimensions (coverage, continuity, freshness, validity, trigger quality)
- Final trip confidence score combining GPS data quality and route execution reliability

---

## Key Findings

- The trip table contains 14,859 trips; 4,059 (27.3%) were flagged for logical issues, mostly pending trips that never started
- GPS position validation covered ~2.35 million pings; 21 invalid or zero-coordinate records were found
- Frozen GPS pings were a material issue: 229,801 pings (9.8%) repeated identical coordinates across timestamp updates
- Signal continuity gaps longer than 300 seconds totaled 40,664 instances
- Stop timing anomaly detection flagged 3,054 of 103,103 stop events (~3.0%) using the combined LOF and rule-based approach
- Route execution scoring covered 9,972 trips; average execution score was 83.2, median 94.0

---

## Repository Structure

```text
.
├── notebooks/
│   ├── 01_trips_cleaning.ipynb
│   ├── 02_stop_times_cleaning.ipynb
│   ├── 03_gps_positions_health_check.ipynb
│   ├── 04_route_stops_cleaning.ipynb
│   ├── 05_stop_locations_cleaning.ipynb
│   ├── 06_join_trip_stop_events.ipynb
│   ├── 07_join_route_stop_locations.ipynb
│   ├── 08_join_scheduled_route_stops.ipynb
│   ├── 09_trip_master_join.ipynb
│   ├── 10_trip_score_gps_execution.ipynb
│   ├── 11_trip_stop_outlier_detection.ipynb
│   ├── 12_stop_service_reliability_type_a.ipynb
│   ├── 13_stop_trip_score_integration.ipynb
│   ├── 14_route_execution_scoring.ipynb
│   ├── 15_trip_gps_health_scoring.ipynb
│   └── 16_final_confidence_score.ipynb
│
├── src/
│   └── utils/                                      # Shared Python utilities
│
├── tests/
├── requirements.txt
└── .gitignore
```

---

## Data

Data was provided by a transportation technology client under a data confidentiality agreement and is not included in this repository.

| Source File | Description |
|---|---|
| `trips.csv` | Trip metadata — status, vehicle, vendor, rider counts |
| `trip_positions.csv` | GPS ping stream — coordinates, timestamps, trip linkage |
| `routes.csv` | Route definitions |
| `route_stops.csv` | Planned stop sequences per route |
| `stop_locations.csv` | Physical stop location coordinates and address data |
| `stop_times.csv` | Scheduled stop times per route and trip |
| `vehicles.csv` | Vehicle metadata |
| `vendors.csv` | Transportation vendor records |
| `tracking_devices.csv` | GPS device metadata |
| `schools.csv` | School location and schedule data |
| `school_times.csv` | School session timing records |
| `vendor_schedules.csv` | Vendor operating schedules |
| `geofencing_config.json` | Geofence radius configuration |

Notebooks use path placeholders (`<RAW_DATA_DIR>`, `<PROCESSED_DATA_DIR>`, `<CLEANED_DATA_DIR>`) instead of hardcoded local paths. Replace these with your local or shared folder paths before running.

---

## Data Processing

The pipeline begins by cleaning five core entities and building the join-ready tables that all scoring notebooks consume.

**Entity Cleaning**

Each source table is validated for primary keys, missing values, duplicate rows, and logical consistency before being promoted to a cleaned output.

- *Trips* — `trip_id` validated as a unique primary key; 4,059 trips (27.3%) flagged for logical issues, mostly pending trips that never started · 📓 [01](notebooks/01_trips_cleaning.ipynb)
- *GPS Positions* — ~2.35 million pings validated for coordinate validity and timestamp ordering; 21 invalid-coordinate records identified; 229,801 frozen pings (9.8%) and 40,664 continuity gaps > 300 seconds flagged · 📓 [03](notebooks/03_gps_positions_health_check.ipynb)
- *Route Stops* — route and stop-location references validated · 📓 [04](notebooks/04_route_stops_cleaning.ipynb)
- *Stop Locations* — missing coordinates flagged; intersection addresses normalized; 16 logical duplicate stop locations mapped to a single `parent_stop_id`; geofence radius fields (80m or 300m) and city/state/zip extracted · 📓 [05](notebooks/05_stop_locations_cleaning.ipynb)
- *Stop Times* — scheduled stop occurrences exploded across the calendar into join-ready records · 📓 [02](notebooks/02_stop_times_cleaning.ipynb)

**Join Layer**

- Stop locations joined to route stops on `stop_location_id`, producing the enriched route-stop table that carries physical location context into all downstream scoring · 📓 [06](notebooks/06_join_trip_stop_events.ipynb) · [07](notebooks/07_join_route_stop_locations.ipynb)
- Scheduled stop times added to complete the stop-level view of what was planned, scheduled, and available to be completed · 📓 [08](notebooks/08_join_scheduled_route_stops.ipynb)
- Trip-level master table assembled from trip metadata, GPS summaries, and position-level aggregates · 📓 [09](notebooks/09_trip_master_join.ipynb) · [10](notebooks/10_trip_score_gps_execution.ipynb)

---

## Scoring

Stop-level and trip-level scoring run in parallel before being merged into the final confidence score.

### Stop-Level Scoring

**Stop Service Reliability Score**

The SSRS measures how reliably each planned stop is actually completed across all trips that serve it. Analysis covers Type A stops — stops that were logged in the system but marked as not completed.

After filtering to completed trips only, a per-stop completion rate is computed:

```text
SSRS = (completed == True count / total appearances) × 100
```

Output: `stop_level_diagnostic.csv` — per-stop reliability signals with stop type flags (`is_anchor`, `is_pickup`, `is_dropoff`, `is_school`) and average timing deviations. Covers 2,760 stops and 82,282 stop records.

📓 Notebook: [12\_stop\_service\_reliability\_type\_a.ipynb](notebooks/12_stop_service_reliability_type_a.ipynb)

**Stop Timing Anomaly Detection**

Timing anomalies are detected at the stop level using two complementary methods:

- *Rule-based score* — penalizes `completed_diff` and `departed_diff` greater than 10 minutes linearly, with deviations above 20 minutes (approximately the 1.5× IQR upper bound) scored as 0
- *LOF score* — Local Outlier Factor applied to `abs_completed_diff` and `abs_departed_diff` with RobustScaler; raw LOF scores converted to a 0–100 reliability scale

3,054 of 103,103 stop events (~3.0%) were flagged using the combined approach.

📓 Notebook: [11\_trip\_stop\_outlier\_detection.ipynb](notebooks/11_trip_stop_outlier_detection.ipynb)

### Trip-Level Scoring

**Route Execution Score**

Scores whether each trip completed its planned stops:

```text
execution_rate = completed_stops / total_planned_stops
```

Trips with execution rates below 80% are flagged as not fully executed as planned. Average execution score: 83.2; median: 94.0; range: 0–100 across 9,972 trips.

📓 Notebook: [14\_route\_execution\_scoring.ipynb](notebooks/14_route_execution_scoring.ipynb)

**GPS Health Score**

Each trip receives a GPS health score from five weighted dimensions:

| Dimension | Weight | Description |
|---|---:|---|
| Coverage | 25% | Whether enough GPS pings were received for the trip duration |
| Continuity | 25% | Whether the trip had large gaps in GPS reporting |
| Freshness | 20% | Whether coordinates froze in suspicious ways; enhanced with a DBSCAN-based stop-aware check so legitimate dwell time near stops is distinguished from technical freezing |
| Validity | 15% | Whether coordinates were physically plausible for the operating area |
| Trigger Quality | 15% | Whether the trip ended through a reliable GPS-related trigger |

📓 Notebook: [15\_trip\_gps\_health\_scoring.ipynb](notebooks/15_trip_gps_health_scoring.ipynb)

---

## Score Integration & Final Confidence

**Stop Score Integration**

SSRS, timing outlier scores, and trip-level GPS and execution scores are merged into a unified stop-level diagnostic table, consolidating all scoring signals before the final confidence computation.

📓 Notebook: [13\_stop\_trip\_score\_integration.ipynb](notebooks/13_stop_trip_score_integration.ipynb)

**Final Confidence Score**

The final trip confidence score blends data quality and execution reliability:

```text
final_confidence_score = 0.4 × trip_data_quality_score + 0.6 × trip_execution_reliability_score
```

Vendor labels are joined and the composite score is computed per trip, producing `trips_final_confidence_scores.csv`.

📓 Notebook: [16\_final\_confidence\_score.ipynb](notebooks/16_final_confidence_score.ipynb)

---

## Setup

```bash
python -m venv env
source env/bin/activate
pip install -r requirements.txt
```

Run notebooks in numeric order (01 → 16). Replace path placeholders with your local data directories before running.

---

## Tools & Technologies

| Layer | Technology |
|---|---|
| Data processing | Python · Pandas · NumPy |
| Anomaly detection | scikit-learn (LOF · DBSCAN) · RobustScaler |
| Geospatial | Geofencing config · coordinate validation |
| Visualization | Matplotlib · Seaborn |

---

## Team

**MSBA 6411 Exploratory Data Analytics · Live Case**

| Member | Contributions |
|---|---|
| Shivanshu Dagur | Data processing · GPS health scoring |
| Amogha Yalgi | Data processing · Route execution scoring |
| Ankit Cao | Data processing · Stop timing anomaly detection |
| Davey Johnson | Data processing · Final confidence framework |
| Tzu-Yu Chen | Data processing · Stop Service Reliability Score · Score integration · Problem framing |

---

## Data Confidentiality Note

This project was conducted under a data confidentiality agreement with the client organization. No data files are included in this repository. The client organization is not named in accordance with the agreement.

---

## Usage and License Note

This repository is shared for academic and portfolio purposes. Please contact the author before reusing or redistributing the code.
