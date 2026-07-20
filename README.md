# Multimodal Intrusion & Insider Threat Detection

This repository contains a reproducible research workflow for **network intrusion detection and insider-threat risk scoring** across three modality branches:

- Network traffic (CICIDS2017)
- Enterprise user behavior — primary (CERT Insider Threat, r6.2)
- Email communication — proxy (Enron Maildir)
- Multimodal fusion — exploratory (behavior + network + email features)

The project compares network-flow classification, enterprise log-based insider threat detection, email activity proxies, and exploratory multimodal fusion with leakage-safe splits, threshold tuning, and explainable AI.

> **Note:** This repository is intended for research and educational use. It is not a production security monitoring system.

**Repository:** [Tahis-Fzs/cicids2017-intrusion-detection](https://github.com/Tahis-Fzs/cicids2017-intrusion-detection)

---

## Project Overview

The study evaluates intrusion and insider-threat modeling strategies on CICIDS2017, CERT r6.2, and Enron using logistic regression, Random Forest, XGBoost, voting ensembles, Isolation Forest (unsupervised), feature ablation, time-based holdout, group (user-level) splits, and SHAP / permutation importance.

The workflow includes dataset loading and cleaning, per-user / per-day aggregation, multimodal fusion, leakage checks, model training, operating-point selection, cross-model comparison, user risk ranking, and paper-ready visualizations.

### Main model families

| Family | Models |
|---|---|
| Network intrusion (CICIDS2017) | Label exploration, device/day aggregation, flow feature summaries |
| Enterprise insider threat (CERT, primary) | Logistic regression, Random Forest, XGBoost, soft-voting ensemble |
| Email proxy (Enron) | Sent/inbox activity features joined as multimodal context |
| Multimodal fusion (exploratory) | CERT behavior + CICIDS network aggregates + Enron email activity |
| Unsupervised screening | Isolation Forest anomaly scores on fused numeric features |

---

## Completed Workflow

The final project workflow contains the following sections:

1. Environment setup
2. CICIDS2017 CSV discovery and load (~2.8M flows, 80 columns)
3. CICIDS class distribution and exploratory plots
4. CICIDS device / day aggregation (`cicids_device_agg.csv`)
5. CERT r6.2 log discovery (logon, file, http, email, device, LDAP)
6. CERT multimodal daily aggregation (`cert_multimodal.csv`)
7. LDAP / employee metadata join (role, department, team)
8. Enron Maildir scan and active-user filtering
9. Enron sent/inbox activity features
10. Multimodal fusion (CERT + Enron + CICIDS) → `multimodal_fused_*.csv`
11. Label attachment and cleaning (`cert_multimodal_labeled_clean.csv`)
12. Leakage-safe column drops and infinite-value cleanup
13. User-grouped train/test split (GroupShuffleSplit)
14. Time-based holdout (train ≤ 2010-03-31, test ≥ 2010-04-01)
15. Logistic regression baseline + best-F1 threshold tuning
16. Random Forest / XGBoost / ensemble comparison
17. Feature ablation (e.g., remove `http_count`)
18. User-level risk ranking tables
19. Permutation importance and SHAP explainability
20. Artifact export (`results/`, `models/`)
21. Paper-ready discussion, limitations, and reproducibility notes

Canonical notebook: `preprocessing.ipynb` — run cells top to bottom after updating local dataset paths.

---

## Key Results

### Primary evaluation (CERT multimodal, time-based holdout)

Train: **2010-01-02 → 2010-03-31** (~146,752 rows) · Test: **2010-04-01 → 2010-04-06** (~1,740 rows)

| Model | Test ROC-AUC | PR-AUC | Best-F1 threshold | F1@best | Precision@best | Recall@best |
|---|---:|---:|---:|---:|---:|---:|
| RandomForest | **0.9608** | **0.8759** | 0.7471 | **0.7933** | 0.7685 | 0.8196 |
| XGBoost | 0.9554 | 0.8552 | 0.7023 | 0.7765 | 0.6950 | 0.8797 |
| Ensemble | 0.9533 | 0.8288 | 0.5268 | 0.8006 | 0.7157 | 0.9082 |
| LogisticRegression | 0.8063 | 0.4685 | 0.6746 | 0.5045 | 0.4774 | 0.5348 |

Locked primary model for ranking/explainability: **RandomForest** (best ROC-AUC / PR-AUC balance on the temporal test window).

### Secondary evaluation (user-grouped split, earlier fused labels)

| Setting | Metric | Value |
|---|---|---:|
| Logistic regression (group split) | ROC-AUC | 0.9995 |
| Logistic regression (group 5-fold CV) | Mean ROC-AUC | 0.9993 ± 0.0002 |
| Best-F1 operating point | Threshold / F1 | 0.8750 / 0.9302 |
| Ablation without `http_count` | ROC-AUC | 0.9607 |

> High group-split AUCs should be interpreted cautiously; the **time-based holdout** is the stricter primary evaluation.

### Network branch (CICIDS2017)

| Item | Value |
|---|---|
| Raw flows loaded | ~2,830,743 rows × 80 columns |
| Dominant labels | BENIGN, DoS Hulk, PortScan, DDoS, … |
| Device aggregate artifact | `cicids_device_agg.csv` |

### Auxiliary logon detector artifact

From `results/metrics.json` (XGBoost logon/logoff detector):

| Metric | Value |
|---|---:|
| Accuracy | 0.9874 |
| ROC-AUC | 0.9992 |
| Average precision | 0.9994 |

Top features (`results/feature_importance.csv`): `hour`, `secs_since_prev`, `user_events_today`, `is_offhours`.

### Main findings

- Random Forest was the strongest single model on the **temporal** CERT holdout (ROC-AUC 0.9608, PR-AUC 0.8759).
- Ensemble slightly improved best-F1 recall but did not beat RF on ranking metrics.
- Logistic regression remained a strong, interpretable baseline on grouped splits but degraded under strict time holdout.
- Removing `http_count` reduced ROC-AUC (0.9995 → 0.9607 on the ablation path), showing web activity is informative but not the only signal.
- Multimodal fusion joins CERT behavior with CICIDS aggregates and Enron email features; sources are **not same-user linked** across real organizations — fusion is exploratory / protocol-level.
- Threshold tuning materially changes precision–recall trade-offs for rare positive (threat) classes.
- Organizational categorical features (team, role, functional unit) ranked highly in permutation importance for the RF explainability pass.

---

## Repository Structure

Recommended structure:

```text
cicids2017-intrusion-detection/
├── README.md
├── LICENSE
├── .gitignore
├── requirements.txt
├── preprocessing.ipynb          # canonical end-to-end notebook
├── Preprocessing.py             # early CERT logon/file EDA script
├── models/
│   └── xgboost_logon_detector.pkl
├── results/
│   ├── metrics.json
│   ├── feature_importance.csv
│   ├── figs/                    # confusion_matrix, roc_curve, pr_curve
│   └── artifacts/               # train/test indices, model pickles
├── *.csv                        # intermediate fused / labeled datasets (local)
└── data/                        # optional: local paths / symlinks to raw corpora
```

Large raw corpora (CICIDS CSVs, CERT r6.2, Enron Maildir) and multi-GB fused tables should generally **not** be committed to GitHub. Keep them outside the repo or behind `.gitignore`.

---

## Installation

Create and activate a Python environment:

```bash
git clone https://github.com/Tahis-Fzs/cicids2017-intrusion-detection.git
cd cicids2017-intrusion-detection
python3 -m venv .venv
```

Linux/macOS:

```bash
source .venv/bin/activate
```

Windows PowerShell:

```bash
.\.venv\Scripts\Activate.ps1
```

Install dependencies:

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

If `requirements.txt` is not yet present, a minimal install is:

```bash
pip install pandas numpy scikit-learn matplotlib seaborn xgboost shap jupyter
```

---

## Dataset

The project expects independent CICIDS2017, CERT, and Enron sources. Update absolute paths in `preprocessing.ipynb` to match your machine.

Expected external layout (example):

```text
Research & Innovation Project/
├── archive/              # CICIDS2017 *pcap_ISCX.csv files
├── r6.2/                 # CERT Insider Threat release
│   ├── logon.csv
│   ├── file.csv
│   ├── http.csv
│   ├── email.csv
│   ├── device.csv
│   └── LDAP/
└── maildir/              # Enron Maildir tree
```

Project-side derived artifacts (generated by the notebook):

| Artifact | Role |
|---|---|
| `cicids_device_agg.csv` | Network device/day aggregates |
| `cert_multimodal.csv` | CERT user/day behavior table |
| `cert_multimodal_labeled_clean.csv` | CERT table with cleaned binary labels |
| `multimodal_fused_*.csv` | Fused CERT + Enron + CICIDS features |
| `active_enron_users.csv` | Enron users with sent/inbox activity |
| `user_threat_summary.csv` | Per-user threat summary / flags |
| `test_event_risks.csv` | Event-level risk scores on holdout |
| `test_user_risk_summary.csv` | User-level risk ranking on holdout |

Expected label conventions:

| Branch | Positive definition |
|---|---|
| CICIDS2017 | Attack class labels (DoS, DDoS, PortScan, Bot, …) vs BENIGN |
| CERT | Insider-threat / flagged user-day labels after cleaning to {0,1} |
| Enron | Activity proxy features only (not a native threat label source) |

Raw data is excluded from Git by default.

---

## Training and Evaluation

The notebook workflow trains and evaluates:

- Logistic regression (balanced class weight)
- Random Forest and XGBoost
- Soft-voting ensemble
- Isolation Forest (unsupervised screening)
- Feature ablation studies

Evaluation includes:

- Accuracy, precision, recall, F1
- ROC-AUC and PR-AUC
- Confusion matrices
- Best-F1 threshold selection from precision–recall curves
- User-grouped CV and time-based holdout
- Permutation importance and SHAP
- User risk ranking (`max_prob`, `mean_prob`, `threat_events`)

---

## Inference

After training on the time-based split, the pipeline exports:

- Event-level threat probabilities (`test_event_risks.csv`)
- User-level risk summaries (`test_user_risk_summary.csv`)
- Thresholded alert status at the tuned best-F1 operating point

Important wording:

- Without ground-truth labels at inference time, outputs are **risk scores and ranked alerts**, not confirmed incident verdicts.
- With held-out labeled user-days, ROC-AUC, PR-AUC, F1, precision, and recall can be computed.

---

## Paper Assets

Recommended paper-ready assets:

- Multimodal workflow diagram (CICIDS + CERT + Enron)
- Dataset split and leakage-check summary
- Model family comparison table (RF / XGB / Ensemble / LR)
- ROC and PR curves (`results/figs/`)
- Confusion matrix for locked primary model
- Feature ablation bar chart
- SHAP / permutation importance top features
- Top-k user risk ranking table
- Supplementary tables for thresholds and CV scores

---

## Reproducibility

The project records:

- Random seed (`42` in `results/metrics.json`)
- Train/test index artifacts under `results/artifacts/`
- Saved model pickles (`models/`, `results/artifacts/`)
- Thresholds chosen via best-F1 on the evaluation split
- Group vs time-based split configuration in notebook cells

Final environment example:

| Item | Value |
|---|---|
| Python | 3.10+ recommended |
| scikit-learn | ≥1.3 |
| XGBoost | latest compatible |
| Platform | macOS (development) |
| Primary model (temporal) | RandomForest |
| Primary seed | 42 |

---




## Team

Intrusion Detection research project · **Daffodil International University**

| Member | ID |
|--------|-----|
| Md. Shadman Hasin | 0242220005101462 |
| Md. Shadman Tahsin | 0242220005101461 |

## Limitations

- CERT, CICIDS2017, and Enron are **not same-organization / same-user linked**; multimodal fusion is exploratory
- Extreme AUCs on some grouped splits may reflect leakage, label construction, or easy negatives — prefer the temporal holdout
- Short temporal test window (early April 2010) limits external generalization claims
- Class imbalance remains severe; threshold choice strongly affects precision
- Limited hyperparameter search
- No online / streaming evaluation
- No formal comparison against published SOTA on identical CERT splits
- External validation on another CERT release (e.g., r5.2 vs r6.2) is recommended before operational claims

---

## Citation

If this project is used in academic work, please cite the repository and related datasets appropriately.

```bibtex
@misc{cicids2017_multimodal_ids,
  title  = {Multimodal Intrusion and Insider Threat Detection on CICIDS2017, CERT, and Enron},
  author = {Tahsin, Md. Shadman},
  year   = {2026},
  url    = {https://github.com/Tahis-Fzs/cicids2017-intrusion-detection},
  note   = {Network intrusion preprocessing, CERT multimodal risk scoring, and exploratory fusion}
}
```

Please also cite the original dataset providers (Canadian Institute for Cybersecurity CICIDS2017, CERT Insider Threat Dataset, Enron Email Dataset).

---

## License

This project is released under the [MIT License](LICENSE).

---

## Disclaimer

This software is for research purposes only and is not intended for production intrusion detection, employee monitoring, or legal decision-making.
