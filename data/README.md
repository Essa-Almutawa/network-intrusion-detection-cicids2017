# Data

Neither the raw nor the processed data are committed to Git (see `.gitignore`).

## Raw input: `data/MachineLearningCVE/`

The 8 CIC-IDS2017 *MachineLearningCVE* CSV files, one per capture day or scenario (2,830,743 rows in total):

```
Friday-WorkingHours-Afternoon-DDos.pcap_ISCX.csv
Friday-WorkingHours-Afternoon-PortScan.pcap_ISCX.csv
Friday-WorkingHours-Morning.pcap_ISCX.csv
Monday-WorkingHours.pcap_ISCX.csv
Thursday-WorkingHours-Afternoon-Infilteration.pcap_ISCX.csv
Thursday-WorkingHours-Morning-WebAttacks.pcap_ISCX.csv
Tuesday-WorkingHours.pcap_ISCX.csv
Wednesday-workingHours.pcap_ISCX.csv
```

Notebook 01 checks that exactly these 8 files are present.

## Processed output: `data/processed/`

Written by `notebooks/01_eda_and_data_preparation.ipynb`:

| File | Contents |
|---|---|
| `cic_ids2017_cleaned_v2.parquet` | 2,830,743 rows × 80 columns (78 features, `Label`, `source_file`); infinities and impossible negative values set to `NaN`; labels normalised |
| `cic_ids2017_eda_metadata_v2.json` | Checkpoint SHA-256, library versions, cleaning decisions, duplicate statistics and flagged columns |

`notebooks/02_model_development.ipynb` reads this Parquet file and compares its SHA-256 with the reference value in Section A.1.

## Citation

> Iman Sharafaldin, Arash Habibi Lashkari, and Ali A. Ghorbani, "Toward Generating a New Intrusion Detection Dataset and Intrusion Traffic Characterization", 4th International Conference on Information Systems Security and Privacy (ICISSP), 2018.
