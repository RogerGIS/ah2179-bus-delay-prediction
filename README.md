# Bus Delay Prediction with GTFS-RT and OpenStreetMap

## Project Overview

This repository contains the code for the final project in **AH2179 Applied Artificial Intelligence in Transportation** at KTH Royal Institute of Technology.

The project investigates whether segment-level road-infrastructure features derived from OpenStreetMap (OSM) improve short-term bus delay prediction for **Stockholm Bus Line 4**.

Historical GTFS-Realtime observations from January 2024 are combined with static GTFS information and OSM-derived road-infrastructure features. Three model families are evaluated:

- Ridge Regression
- LightGBM
- Long Short-Term Memory (LSTM)

The main experiment compares a temporal and operational feature set with the same feature set augmented by five spatial infrastructure variables.

---

## Research Questions

1. To what extent do segment-level road-infrastructure features derived from OSM improve next-stop bus delay prediction beyond temporal and schedule-related information?

2. Does the predictive benefit differ across linear, tree-based, and recurrent sequence models?

3. How does model performance change between routine operating conditions and rare extreme disruption events?

---

## Repository Structure

- `01_line4_spatial_pipeline.ipynb`
- `02_line4_eda.ipynb`
- `03_model_comparison.ipynb`
- `04_lstm_model.ipynb`
- `README.md`
- `requirements.txt`
- `.gitignore`

---

## Notebooks

### 01_line4_spatial_pipeline.ipynb

Processes the January 2024 Line 4 data, extracts official GTFS route geometry, derives segment-level OSM infrastructure features, and constructs the final modelling table.

Main tasks include:

- filtering and quality-checking Line 4 observations;
- extracting representative GTFS route shapes for both directions;
- constructing directed stop-to-stop segments;
- deriving OSM-based road-infrastructure features;
- constructing lagged and temporal predictors; and
- producing the final master feature table.

### 02_line4_eda.ipynb

Performs exploratory data analysis of the final modelling dataset.

The analysis includes:

- spatiotemporal delay patterns by hour and stop;
- example trip-level delay trajectories;
- delay persistence across consecutive stops; and
- the distribution of arrival delays and extreme observations.

### 03_model_comparison.ipynb

Implements the tabular prediction experiments and diagnostics.

The notebook includes:

- persistence baseline;
- Ridge Regression;
- LightGBM;
- temporal versus spatial feature ablation;
- chronological train-validation-test evaluation;
- routine-operation diagnostics;
- prediction error by delay severity; and
- extreme-disruption diagnostics.

### 04_lstm_model.ipynb

Implements the recurrent sequence-model experiments using LSTM.

It compares:

- temporal and operational features only; and
- temporal, operational, and spatial features.

The LSTM notebook was developed by **Zhaotong Tao**.

---

## Data

The analysis combines three main data sources:

- historical Stockholm public transport GTFS-Realtime observations from January 2024;
- static GTFS data;
- OpenStreetMap road-network data.

The historical operational data provide observed bus arrival and departure information. Static GTFS provides scheduled route, trip, stop, and timetable information. OpenStreetMap provides road-network and infrastructure context for each directed stop-to-stop segment.

The final modelling dataset contains:

- **195,524 stop-level observations**
- **48 directed stop-to-stop segments**

Large raw and processed datasets are not included in this repository because of file-size constraints.

---

## Prediction Task

Each observation represents one bus trip arriving at one stop.

The prediction target is `observed_arrival_delay`, measured in seconds.

The task is interpreted as a rolling one-stop-ahead prediction problem: after the bus state at the previous stop is observed, the model predicts the arrival delay at the following stop.

---

## Feature Sets

Two feature configurations are compared.

### Temporal + Schedule

The baseline feature set contains 12 variables:

- `scheduled_arrival_time_sfm`
- `delay_lag_1`
- `delay_lag_2`
- `delay_diff_prev`
- `hour_sin`
- `hour_cos`
- `day_of_week`
- `is_weekend`
- `is_morning_peak`
- `is_evening_peak`
- `stop_sequence`
- `direction_id`

Recent-delay features are constructed within the same service date, trip, and travel direction.

### Spatial Features

Five OSM-derived segment-level variables are added:

- `network_length_m`
- `num_traffic_signals`
- `num_intersections`
- `bus_lane_ratio`
- `has_bus_priority`

`network_length_m` represents the length of the stop-to-stop route segment along the official route geometry.

`num_traffic_signals` represents clustered signalized locations intersecting the route segment corridor.

`num_intersections` is used as a segment-level proxy for intersection exposure derived from intersecting OSM road edges.

`bus_lane_ratio` represents the estimated proportion of the route segment associated with bus-lane infrastructure.

`has_bus_priority` is a derived binary indicator based on bus-lane coverage and should not be interpreted as actual traffic-signal priority.

The spatial-enhanced feature set therefore contains **17 variables** in total.

---

## Temporal Train-Validation-Test Split

To reduce temporal leakage, the January 2024 observations are divided chronologically:

- **Training:** January 1-18 — 115,164 observations
- **Validation:** January 19-23 — 27,666 observations
- **Test:** January 24-31 — 52,694 observations

The validation period is used for model selection and LightGBM early stopping. The final test period remains unseen during model fitting and selection.

---

## Models

Four prediction approaches are evaluated.

### Persistence Baseline

The persistence baseline predicts the next-stop delay using the observed delay at the preceding stop.

### Ridge Regression

Ridge Regression provides a regularized linear benchmark. Continuous predictors are standardized using parameters fitted on the training data.

### LightGBM

LightGBM is used to represent nonlinear relationships and interactions among temporal, operational, and spatial predictors.

Validation data are used for early stopping.

### LSTM

The LSTM processes ordered stop-level sequences from the same service date, trip, and direction.

The temporal-only and spatial-enhanced LSTM models use the same sequence construction and evaluation split.

---

## Main Results

### Full Test Set

| Model | Feature Set | MAE (s) | RMSE (s) | R² |
|---|---|---:|---:|---:|
| Persistence | Previous-stop delay | 38.68 | 60.56 | 0.9389 |
| Ridge | Temporal + Schedule | 37.20 | 57.22 | 0.9454 |
| Ridge | Temporal + Schedule + Spatial | 36.49 | 54.60 | 0.9503 |
| LightGBM | Temporal + Schedule | 25.67 | 99.52 | 0.8349 |
| LightGBM | Temporal + Schedule + Spatial | 23.56 | 98.08 | 0.8397 |
| LSTM | Temporal + Schedule | 29.25 | 83.82 | 0.8829 |
| LSTM | Temporal + Schedule + Spatial | 21.57 | 45.09 | 0.9661 |

Adding the five spatial features reduced MAE by approximately:

- **Ridge:** 1.9%
- **LightGBM:** 8.2%
- **LSTM:** 26.3%

The spatial-enhanced LSTM achieved the best full-test performance.

---

## Routine-Operation Diagnostics

For diagnostic analysis, routine operation is defined as:

`|arrival delay| <= 600 seconds`

This threshold is used only to separate routine and extreme operating regimes and is not an official service-quality standard.

The routine subset contains **51,168 observations**, corresponding to approximately **97.1%** of the test set.

Selected results are:

| Model | MAE (s) | RMSE (s) | R² |
|---|---:|---:|---:|
| Persistence | 38.19 | 56.68 | 0.8333 |
| Ridge + Spatial | 35.94 | 50.31 | 0.8686 |
| LightGBM + Spatial | 20.66 | 30.62 | 0.9513 |
| LSTM + Spatial | 19.92 | 30.18 | 0.9527 |

LightGBM and LSTM therefore perform similarly under routine operating conditions.

---

## Extreme-Disruption Diagnostic

A severe disruption was identified on **25 January 2024**.

One Line 4 trip contained **15 consecutive observations** with arrival delays of approximately **7,100-7,400 seconds**, corresponding to more than two hours.

This event helps explain the large difference between LightGBM's routine-operation and full-test RMSE. LightGBM substantially underestimated the extreme delay trajectory, while its prediction accuracy remained strong for the large majority of routine observations.

The diagnostic illustrates why MAE and RMSE should be interpreted together and why rare operating regimes should be examined separately from average model performance.

---

## Interpretation

The results indicate that segment-level road-infrastructure context provides additional predictive information beyond recent delay history and schedule-related variables.

However, the spatial benefit is architecture-dependent:

- Ridge shows a relatively small improvement;
- LightGBM shows a moderate improvement;
- LSTM shows the largest improvement.

These results demonstrate predictive association rather than causality. The experiments evaluate the spatial feature set collectively and do not establish that an individual road-infrastructure variable directly causes changes in bus delay.

---

## Reproducibility

The notebooks were developed in Google Colab.

To reproduce the workflow:

1. Install the dependencies listed in `requirements.txt`.
2. Prepare the required GTFS-RT and static GTFS datasets.
3. Adjust the Google Drive paths in the notebooks if necessary.
4. Run the notebooks in numerical order:

   - `01_line4_spatial_pipeline.ipynb`
   - `02_line4_eda.ipynb`
   - `03_model_comparison.ipynb`
   - `04_lstm_model.ipynb`

The first notebook produces the processed data and spatial feature tables required by the subsequent notebooks.

Raw and processed datasets are stored externally and are therefore not included in the GitHub repository.

---

## Software Dependencies

Main Python packages used in the project include:

- pandas
- NumPy
- scikit-learn
- LightGBM
- TensorFlow / Keras
- GeoPandas
- OSMnx
- Shapely
- Matplotlib
- Seaborn
- Folium
- PyArrow

See `requirements.txt` for the dependency list.

---

## Limitations

This repository reproduces a case study of Stockholm Bus Line 4 using January 2024 data.

The spatial variables represent relatively static road-infrastructure characteristics and do not include dynamic factors such as:

- real-time congestion;
- weather;
- roadworks;
- incidents;
- temporary signal failures; or
- passenger demand.

The spatial-feature experiment is conducted at the feature-set level. Individual infrastructure variables should therefore not be interpreted causally.

---

## Authors

**Jiayi Wu**  
**Zhaotong Tao**

KTH Royal Institute of Technology  
AH2179 Applied Artificial Intelligence in Transportation
