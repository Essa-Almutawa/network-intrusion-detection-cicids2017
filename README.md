# DIRA: Network Intrusion Detection on CIC-IDS2017

**[Final report (PDF)](Docs/DIRA_Report.pdf) · [Presentation (PDF)](Docs/DIRA_Presentation.pdf)**

This project classifies network flows from the CIC-IDS2017 dataset as **benign** or **malicious** (binary classification). It compares three approaches:

- a class-weighted **Logistic Regression** baseline
- a gradient-boosted **LightGBM** classifier
- an **Autoencoder** trained on benign traffic only and used as an anomaly detector

The project emphasises an evaluation protocol that supports the reported numbers, and states clearly what those numbers do not show.

## Pipeline

| Step | Notebook | Input | Output |
|---|---|---|---|
| 1. EDA and data preparation | `notebooks/01_eda_and_data_preparation.ipynb` | 8 raw CSV files in `data/MachineLearningCVE/` | `data/processed/cic_ids2017_cleaned_v2.parquet` + metadata JSON; `results/eda/` |
| 2. Model development and evaluation | `notebooks/02_model_development.ipynb` | `data/processed/cic_ids2017_cleaned_v2.parquet` | `models/`, `results/model_development/` |

## Repository layout

```
network-intrusion-detection-cicids2017/
├── README.md
├── requirements.txt
├── data/
│   ├── MachineLearningCVE/       raw CIC-IDS2017 CSV files (git-ignored)
│   └── processed/                cleaned checkpoint written by notebook 01 (git-ignored)
├── notebooks/
│   ├── 01_eda_and_data_preparation.ipynb
│   └── 02_model_development.ipynb
├── models/                       trained models (git-ignored) + config.json
├── results/
│   ├── eda/                      notebook 01: tables/, figures/, report_assets.html
│   └── model_development/        notebook 02: tables/, figures/, report_assets.html
└── Docs/
    ├── DIRA_Report.pdf           final project report
    └── DIRA_Presentation.pdf     final presentation slides
```

## Data

The project uses the **MachineLearningCVE** CSV version of CIC-IDS2017: 8 files, one per capture day or scenario, 2,830,743 flows, 78 flow features plus `Label`. Download the files from the Canadian Institute for Cybersecurity and place them in `data/MachineLearningCVE/`. See [`data/README.md`](data/README.md) for details.

## Method

### Notebook 01: EDA and data preparation

1. Merge the 8 day files and keep `source_file` so every row can be traced to its capture day.
2. Strip column names and repair the corrupted byte in the `Web Attack` labels.
3. Convert `Infinity` to `NaN`, keeping count separately of values that were already missing in the raw files.
4. Scan every numeric column for negative values:
   - Impossible negatives (durations, inter-arrival times, rates, header and segment sizes) become `NaN`.
   - The `-1` sentinel in `Init_Win_bytes_*` ("no TCP window observed") is kept.
5. Report full-row and feature-level duplicates and label-conflicting patterns. They are recorded here, not removed.
6. Flag constant and exactly duplicated columns. They are flagged here, not removed.
7. Explore class imbalance, feature distributions, correlations and mutual information (all descriptive).
8. Document capture structure that can act as a shortcut:
   - Every attack family comes from one capture day.
   - Most families target a single destination port.
   - `Init_Win_bytes_forward = -1` appears almost only in benign traffic.
9. Save a versioned Parquet checkpoint with its SHA-256 and a metadata JSON.

No rows or columns are removed in notebook 01. Imputation is deliberately left to the modelling pipeline.

### Notebook 02: model development

1. Load and verify the checkpoint (row count and SHA-256), and validate labels. `BENIGN` becomes 0 and every other label becomes 1.
2. Remove the leakage columns `Label` and `source_file`.
3. Keep one copy of each distinct feature vector and drop patterns that appear with both labels. This happens before splitting, so metrics describe this filtered population.
4. Split into stratified 60 / 20 / 20 train / validation / test sets, with an assertion that no test row repeats a training pattern.
5. Select features on the training split only: drop constant features and one of each pair with |r| > 0.95.
6. Preprocess:
   - LightGBM: median imputation only.
   - Logistic Regression and Autoencoder: imputation plus a uniform `QuantileTransformer`.

   Everything is fitted on the training split only.
7. Train the models. LightGBM uses early stopping on validation AP. The Autoencoder (44-64-32-16-32-64-44) is trained on benign training flows only.
8. Choose each model's threshold (F1-optimal) and the final model (by validation average precision) on the **validation** set. The test set is used only for reporting.
9. Run diagnostics:
   - train/test AP gap, single-feature AUC scan and a trivial-baseline comparison
   - per-family recall
   - two exploratory experiments: Autoencoder bottleneck width, and retraining without FTP-Patator

## Results

Each notebook saves its own outputs:

- `results/eda/tables/` (CSV) and `results/eda/figures/` (PNG, 300 dpi) from notebook 01
- `results/model_development/tables/` (CSV) and `results/model_development/figures/` (PNG, 300 dpi) from notebook 02

The last section of each notebook also writes a `report_assets.html` page that collects every table and figure. Open it in a browser and copy any table or figure directly into a report (Word or Google Docs).

Main tables in `results/model_development/tables/`:

| File | Contents |
|---|---|
| `model_comparison.csv`, `operational_counts.csv` | Test-set AP, precision, recall, F1, FPR and confusion counts at validation-selected thresholds |
| `autoencoder_operating_points.csv` | Autoencoder at the validation-F1 threshold and at the label-free benign-P95 threshold |
| `per_attack_recall.csv` | Recall per attack family |
| `exploratory_family_exclusion.csv` | FTP-Patator exclusion experiment |
| `validation_comparison.csv`, `threshold_sweep.csv`, `operating_points.csv`, `overfitting_check.csv`, `exploratory_bottleneck.csv` | Supporting tables |
| `data_filtering_summary.csv`, `split_summary.csv`, `feature_reduction.csv` | Data accounting |

**Headline test-set results** (thresholds chosen on validation; from `results/model_development/tables/model_comparison.csv`):

| Model | Precision | Recall | F1 | AP | ROC-AUC | FPR |
|---|---|---|---|---|---|---|
| Logistic Regression | 0.9256 | 0.9621 | 0.9435 | 0.9308 | 0.9928 | 1.57% |
| LightGBM | 0.9981 | 0.9983 | 0.9982 | 0.9999 | 1.0000 | 0.04% |
| Autoencoder | 0.8291 | 0.7414 | 0.7828 | 0.8511 | 0.9498 | 3.10% |

**How to read them**

- On a random split of this capture, the supervised models, especially LightGBM, are expected to score close to perfectly.
- The FTP-Patator exclusion experiment shows how much of that depends on each attack family being present in training.
- Always compare recall together with false alarms, and ignore per-family results for very rare families (Infiltration, SQL Injection, Heartbleed).

## Limitations

- **Random split within one 2017 capture.** Flows from the same session and day land in all three splits. Each attack family occurs on a single day, so a day-based split cannot evaluate every family. Scores should not be expected to carry over to other networks or periods.
- **Shortcut features.** `Destination Port` and the `Init_Win_bytes_*` sentinel separate much of the traffic by service or protocol rather than by malicious behaviour (notebook 01, Section 18b).
- **Label-dependent filtering before the split.** Duplicate and conflict removal in notebook 02 defines the evaluated population.
- **Single seed.** The Autoencoder and exploratory experiments are sensitive to seed and environment.
- **Dataset quality.** Negative and overflow values, and a few label-conflicting patterns, show that the flow features contain measurement artefacts. Their causes are not verified here.

## Reproducing

```bash
python -m venv .venv && source .venv/bin/activate      # Python 3.12
pip install -r requirements.txt
# place the 8 CIC-IDS2017 MachineLearningCVE CSV files in data/MachineLearningCVE/
jupyter nbconvert --to notebook --execute --inplace notebooks/01_eda_and_data_preparation.ipynb
jupyter nbconvert --to notebook --execute --inplace notebooks/02_model_development.ipynb
```

Allow about 8 GB of RAM. Notebook 02 prints a warning if the checkpoint's SHA-256 differs from the reference value. This can happen with other pandas or pyarrow versions even when the data are identical. Training times depend on the machine.

## Team

Essa Almutawa · Dana Alghamdi · Rana Alziyadi · Raneem Alqahtani

Data Science Bootcamp, Saudi Digital Academy in partnership with WeCloudData, 2026

## Dataset credit

CIC-IDS2017, Canadian Institute for Cybersecurity, University of New Brunswick. I. Sharafaldin, A. H. Lashkari, A. A. Ghorbani, *Toward Generating a New Intrusion Detection Dataset and Intrusion Traffic Characterization*, ICISSP 2018. Follow the dataset's terms of use.

## License
      
Code is released under the [MIT License](LICENSE). The CIC-IDS2017 dataset is not included and remains subject to its own terms.
