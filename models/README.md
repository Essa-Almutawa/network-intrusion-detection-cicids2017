# Models

Trained artifacts are written here by Section K of `notebooks/02_model_development.ipynb` and are **not committed** (see `.gitignore`); only `config.json` is tracked.

| File | Contents |
|---|---|
| `logistic_regression.pkl` | scikit-learn `LogisticRegression` (joblib) |
| `lightgbm.pkl` | `lightgbm.LGBMClassifier` (joblib) — selected model |
| `autoencoder.keras` | Keras autoencoder trained on benign flows |
| `imputer.pkl` | median `SimpleImputer` fitted on the training split |
| `quantile_transformer.pkl` | `QuantileTransformer` (uniform) fitted on the training split |
| `config.json` | input fingerprint, library versions, features, thresholds, parameters |

Pickles are tied to the library versions in `requirements.txt`; load them only from a trusted source and with the same versions.

## Inference outline

1. Strip whitespace from column names and select `config["features"]` in that order.
2. Replace `inf`/`-inf` with `NaN`.
3. `imputer.transform(...)` → LightGBM input.
4. For Logistic Regression and the Autoencoder, additionally apply `quantile_transformer.transform(...)`.
5. Compare the score with the threshold stored in `config.json` (`thresholds_validation_f1`; the Autoencoder score is the mean squared reconstruction error).
