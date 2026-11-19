# Earthquake Magnitude Estimation

## Abstract

This B.Tech Semester 5 project uses regression to estimate USGS catalogue magnitude (`mag`) from an event's location, depth, month, and hour. It compares tree-based regressors using chronological validation. The output is a magnitude estimate for supplied event conditions, not an earthquake forecast or warning.

## Dataset

The catalogue covers 2000-2024 and uses earthquakes with magnitude M >= 4.5. It has 173,122 raw rows and 173,041 cleaned rows. The target is `mag`; raw year and post-event measurement fields are not model inputs. The cleaned catalogue is stored in `data/clean.csv`.

## Method

1. Load the local USGS catalogue or download it from the USGS FDSN event service.
2. Clean invalid timestamps, missing core values, invalid coordinates, non-earthquake types, duplicate IDs, and records below M 4.5. Negative depths are retained.
3. Explore magnitude, depth, annual counts, locations, numeric correlations, and magnitude versus depth.
4. Use latitude, longitude, depth, month, hour, absolute latitude, and log depth as features. The shared `make_features()` function is used by training and the app.
5. Sort by time and reserve the later 20% for a final test. The final test has 34,609 rows from May 30, 2020 through December 31, 2024.
6. Fit median imputation within each training fold. Linear Regression also scales features. The current core feature columns have no missing values, so imputation is a safety step.
7. Compare Dummy, Linear Regression, Decision Tree, Random Forest, Extra Trees, Gradient Boosting, and HistGradientBoosting. Use three expanding chronological folds grouped by timestamp so equal-time events cannot straddle fold boundaries. RandomizedSearchCV tunes Random Forest; small GridSearchCV runs compare the other ensembles and a Random Forest / Huber Gradient Boosting blend.
8. Select by training-only cross-validation RMSE. Evaluate finalists on the chronological test set only after selection; test results do not affect model or feature selection.

## Results

Cross-validation selected a Random Forest / Huber Gradient Boosting blend with weights 0.6 / 0.4. Its components use RandomForestRegressor (`n_estimators=40`, `max_depth=8`, `min_samples_leaf=5`) and GradientBoostingRegressor (`n_estimators=160`, `learning_rate=0.05`, `max_depth=2`, `min_samples_leaf=20`, `loss=huber`), with random state 42.

| Model | CV MAE | CV RMSE | CV R2 | Test MAE | Test RMSE | Test R2 |
|---|---:|---:|---:|---:|---:|---:|
| DummyRegressor | 0.281602 | 0.383119 | -0.011409 | 0.277259 | 0.370705 | -0.009343 |
| LinearRegression | 0.280313 | 0.382066 | -0.005976 | 0.278852 | 0.371718 | -0.014864 |
| DecisionTreeRegressor | 0.284881 | 0.385633 | -0.026082 | 0.269046 | 0.362016 | 0.037421 |
| RandomForestRegressor | 0.279975 | 0.376752 | 0.021374 | 0.268024 | 0.359271 | 0.051960 |
| ExtraTreesRegressor | 0.280382 | 0.376970 | 0.020188 | 0.267382 | 0.357988 | 0.058719 |
| GradientBoostingRegressor | 0.265514 | 0.378588 | 0.013102 | 0.260191 | 0.364501 | 0.024160 |
| HistGradientBoostingRegressor | 0.282282 | 0.379875 | 0.004061 | 0.269153 | 0.361126 | 0.042144 |
| RF / Huber GB blend (selected by CV) | 0.272683 | 0.374930 | 0.031523 | 0.264231 | 0.359846 | 0.048926 |

Compared with the saved Random Forest, the blend reduced validation RMSE by 0.001822 (0.48%) and validation MAE by 0.007292 (2.60%). On the final test period, MAE improved by 0.003792 (1.41%), but RMSE increased by 0.000574 (0.16%) and R2 decreased by 0.003034. The test result is mixed: typical absolute error improved, but test RMSE and R2 did not. Extra Trees scored slightly better on test RMSE/R2, but its validation RMSE was higher; the test set was not used to change the selection.

The target is concentrated just above the catalogue cutoff: 76.7% of test events are M 4.5-5, while only 65 test events are M 7 or larger. Location, depth, and calendar inputs explain little of the remaining magnitude variation. This weak signal and changes in catalogue magnitude-type mix over time help explain the low R2. Measurement type is excluded because it is tied to how the target is assessed and is not an input in the app.

The recent-seismicity feature set did not help in the baseline comparison: its best training-only CV RMSE was 0.378124 versus 0.376752 for base features. About one third of training rows had no qualifying nearby event in the prior 30 days; event counts were highly skewed (95th percentile 89, maximum 598). Replacing month/hour with cyclical encodings also failed to improve CV RMSE (best 0.376769), so neither feature change was retained.

## Limitations and leakage

The selected model's test R2 of 0.048926 means it explains little of the variation in test magnitudes. The final holdout comparison shows no RMSE or R2 improvement over the original Random Forest, despite a lower MAE. Some USGS fields, such as station counts and magnitude errors, are recorded after detection or are tied to how magnitude is assessed; including them could leak target information. This project estimates catalogue magnitude only; it does not predict earthquake occurrence, time, location, or issue warnings.

## How to download the data

If `data/earthquakes_raw.csv` is absent, training downloads the catalogue from the USGS FDSN event service using annual windows. For example:

`https://earthquake.usgs.gov/fdsnws/event/1/query?format=csv&starttime=2024-01-01&endtime=2025-01-01&minmagnitude=4.5&orderby=time-asc`

## Run

In PowerShell from the project folder, activate the project's CPython 3.11 environment, then run:

```powershell
.\.venv311\Scripts\Activate.ps1
python -m src.train
streamlit run app.py
python -m pytest -q
```

The notebook can be run with `python -m nbconvert --to notebook --execute --inplace --ExecutePreprocessor.timeout=1200 notebook.ipynb`.

## Team

- Samiksha Rana (Team Leader, 240161)
- Naisha Jaiswal (240149)
- Sangini Garg (240162)
- Swara Mishra (240173)

Mentor: Dr. Vikas Raina
