# Earthquake Magnitude Estimation

## Abstract

This B.Tech Semester 5 project uses regression to estimate USGS catalogue magnitude (`mag`) from an event's location, depth, month, and hour. It compares five models using a chronological evaluation. The output is a magnitude estimate for supplied event conditions, not an earthquake forecast or warning.

## Dataset

The catalogue covers 2000–2024 and uses earthquakes with magnitude M >= 4.5. The current data has 173,122 raw rows and 173,041 cleaned rows. The target is `mag`; raw year is not a feature. The raw CSV is ignored by Git; `data/clean.csv` is the cleaned catalogue.

## Method

1. Load the local USGS CSV or download it from the USGS FDSN event service.
2. Clean invalid timestamps, missing core values, invalid coordinates, non-earthquake types, duplicate IDs, and records below M 4.5. Negative depths are retained.
3. Explore magnitude, depth, annual counts, locations, numeric correlations, and magnitude versus depth.
4. Use latitude, longitude, depth, month, hour, absolute latitude, and log depth as features. The single `make_features()` function is shared by training, the notebook, and the app.
5. Sort by time and split chronologically: earlier 80% for training and later 20% for the final test. The final test contains 34,609 rows from May 30, 2020 through December 31, 2024.
6. Use median imputation inside a scikit-learn Pipeline; Linear Regression also scales features. This keeps imputation and scaling fit within each training fold.
7. Compare Dummy, Linear Regression, Decision Tree, Random Forest, and Gradient Boosting. Use three-fold TimeSeriesSplit on training rows; a small GridSearchCV tunes Random Forest and Gradient Boosting.
8. Select the model by training-only cross-validation RMSE, then evaluate once on the chronological test set. A separate random-split diagnostic uses training rows only.

## Results

Cross-validation RMSE selected **GradientBoostingRegressor** (0.376934). The final test metrics are:

| Model | CV RMSE | Test MAE | Test RMSE | Test R² |
|---|---:|---:|---:|---:|
| DummyRegressor | 0.383119 | 0.277259 | 0.370705 | -0.009343 |
| LinearRegression | 0.382066 | 0.278852 | 0.371718 | -0.014864 |
| DecisionTreeRegressor | 0.385633 | 0.269046 | 0.362016 | 0.037421 |
| RandomForestRegressor | 0.377045 | 0.268060 | 0.359415 | 0.051199 |
| GradientBoostingRegressor | 0.376934 | 0.268781 | 0.360457 | 0.045692 |

Gradient Boosting was selected using cross-validation, even though Random Forest had a slightly lower RMSE on the final test set. The selected model's test RMSE is 2.764489% lower than the Dummy baseline. Its R² is low. Test MAE rises from 0.206449 for magnitudes 4.5–5 to 2.417181 for magnitudes 7+, where the test set contains only 65 events.

The training-only diagnostic gave RMSE 0.376934 for chronological three-fold CV and 0.378114 for the random 80/20 split. Random splitting is less representative of later-event use because it mixes time periods; its score is diagnostic only and does not replace the chronological evaluation.

## Limitations and leakage

R² of 0.045692 means the selected model explains little of the variation in test magnitudes, and its improvement over the baseline is small. The model uses only catalogue location, depth, month, and hour fields. It does not predict earthquake occurrence, time, or location and is not a warning system. Some USGS fields, such as station counts and magnitude errors, are recorded after detection or are tied to how magnitude is assessed; including them could leak target information, so they are excluded.

## How to download the data

The training command downloads the catalogue if `data/earthquakes_raw.csv` is absent. The request uses the USGS FDSN event API with one annual window at a time, for example:

`https://earthquake.usgs.gov/fdsnws/event/1/query?format=csv&starttime=2024-01-01&endtime=2025-01-01&minmagnitude=4.5&orderby=time-asc`

The raw CSV is not included in the project archive. It is ignored by `.gitignore`; the cleaned `data/clean.csv` is retained.

## Run

From the project folder:

```bash
python -m pip install -r requirements.txt
python -m src.train
streamlit run app.py
```

Run the saved notebook with `python -m nbconvert --to notebook --execute --inplace --ExecutePreprocessor.timeout=1200 notebook.ipynb`. Run the three tests with `python -m pytest -q`.

## Team

- Samiksha Rana (Team Leader, 240161)
- Naisha Jaiswal (240149)
- Sangini Garg (240162)
- Swara Mishra (240173)

Mentor: Dr. Vikas Raina
