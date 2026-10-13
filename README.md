# FuzzyCred-HQCM

**Fuzzy-credibility stacked hybrid quantum-classical learning for ECG-derived myocardial-infarction annotation classification.**

This repository contains an executable Python project with a LightGBM branch, an optional four-qubit variational quantum classifier (VQC), a fuzzy-credibility stacking layer, patient-grouped out-of-fold (OOF) stacking, PTB-XL fold integrity checks, and exportable predictions and metrics.

> **Research and clinical-use notice.** This is research software, not a medical device and not intended for diagnosis or treatment. The endpoint is an ECG-derived PTB-XL SCP-code annotation (positive if AMI, IMI, LMI, or PMI is present), not independently adjudicated clinical MI. Do not interpret benchmark performance as clinical efficacy or deployment readiness.

## What is implemented

- Strict `ecg_id` joins for the PTB-XL metadata, SCP statements, 12SL features, and Uni-G features.
- Target creation using SCP subclass codes `AMI`, `IMI`, `LMI`, and `PMI`.
- Official outer partitions: folds 1–8 train, fold 9 validation, fold 10 locked test.
- Explicit checks for patient overlap across outer partitions.
- Patient-grouped `GroupKFold` OOF predictions to train the fuzzy-credibility meta-learner; inner-fold patient overlap is checked before each fit.
- Training-only imputation, variance filtering, mutual-information selection, robust scaling, standard scaling, and PCA.
- LightGBM classical branch and optional four-qubit PennyLane state-vector VQC with RY/RZ angle encoding and ring CNOT entanglement.
- Fuzzy membership features (low/medium/high), branch confidence, and branch disagreement supplied to a supervised logistic meta-readout.
- Validation and locked-test probability exports, classification metrics, split audits, inner-fold audit, and feature manifest.
- Automated unit tests and GitHub Actions CI.

## Important implementation specification

The archived source bundle supplied for this project did not contain the exact historical FuzzyCred meta-learner equation, fitted weights, or original row-aligned predictions. This repository therefore makes the executable FuzzyCred layer explicit: it derives low/medium/high fuzzy memberships for each branch probability and fits a standardized logistic readout to the fuzzy memberships, raw branch probabilities, branch disagreement, and branch confidence using **only patient-grouped OOF training predictions**. The implementation is internally complete and testable; it does not hard-code historical manuscript results. Manuscript results must be regenerated with this code before they are attributed to this repository.

## Repository layout

```text
src/fuzzycred_hqcm/
  data.py       Input validation, label construction, patient-fold audit
  fuzzycred.py  Fuzzy membership functions and stacked meta-learner
  quantum.py    PennyLane/PyTorch four-qubit VQC
  metrics.py    Binary classification metrics
  pipeline.py   OOF stacking, fitting, evaluation, and artifact export
  cli.py        Command-line entry point
tests/          Unit tests
.github/        Continuous integration workflow
```

## Environment

Python 3.10–3.12 is supported. The VQC requires PennyLane and PyTorch; these are optional dependencies so the classical baseline can be installed independently.

### Install

```bash
python -m venv .venv
```

Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -e ".[quantum,dev]"
```

macOS/Linux:

```bash
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -e '.[quantum,dev]'
```

### Input data

Create a local `data/` directory containing the four CSV files below. They are not distributed in this repository; confirm that you have the right to redistribute any source-derived feature tables before publishing them.

| File | Required columns | Purpose |
|---|---|---|
| `ptbxl_database.csv` | `ecg_id`, `patient_id`, `strat_fold` | ECG identifiers, patient grouping, official fold |
| `ptbxl_statements.csv` | `ecg_id`, `scp_codes` | Target-label construction |
| `12sl_features.csv` | `ecg_id` plus numeric predictors | 12SL-derived features |
| `unig_features.csv` | `ecg_id` plus numeric predictors | Uni-G-derived features |

All feature files must have a unique, non-null `ecg_id`. The pipeline refuses to join by row order. If the feature tables do not contain validated identifiers, obtain the source mapping first; do not infer identifiers from row position.

## Run

Run the complete hybrid model (OOF VQC training can be computationally expensive):

```bash
fuzzycred-hqcm --data-dir ./data --output-dir ./results \
  --seed 2026 --features 128 --inner-folds 5 \
  --vqc-layers 2 --vqc-epochs 200 --vqc-batch-size 256 \
  --vqc-learning-rate 0.03 --device cpu
```

Run only the LightGBM baseline:

```bash
fuzzycred-hqcm --data-dir ./data --output-dir ./results_lgbm --classical-only
```

Or use the module entry point:

```bash
python -m fuzzycred_hqcm.cli --data-dir ./data --output-dir ./results
```

## Output artifacts

| Artifact | Contents |
|---|---|
| `test_metrics.csv` | Accuracy, precision, recall, specificity, F1, ROC-AUC, average precision, Brier score, confusion counts |
| `test_predictions.csv` | Locked-test row identifiers, patient IDs, labels, and branch/stack probabilities |
| `validation_probabilities.npz` | Validation labels, identifiers, and aligned model probabilities; generated for hybrid runs |
| `test_probabilities.npz` | Test labels, identifiers, and aligned model probabilities; generated for hybrid runs |
| `patient_split_audit.json` | Cohort counts and outer partition audit |
| `inner_grouped_cv_audit.csv` | Per-fold record and patient counts for OOF stacking |
| `feature_manifest.csv` | Candidate features and selected-feature indicators |
| `run_config.json` | Seed, target, splits, feature counts, and model settings |
| `vqc_training_info.json` | Quantum circuit details and best validation checkpoint |

## Testing and quality checks

```bash
pytest -q
```

GitHub Actions runs the unit tests on Python 3.10, 3.11, and 3.12. Before submitting results to a journal, run the project from a clean environment and archive the input-data version, package versions, command, seed, output artifacts, and commit hash.

## Reproducibility and reporting checklist

1. Keep the fold-10 test set locked. Use fold 9 for model selection and any calibration or decision-threshold selection.
2. Use the exported aligned validation/test predictions for paired inference and calibration diagnostics. Never substitute another run's predictions.
3. Report patient-clustered uncertainty intervals when records from the same patient are present in the evaluation cohort.
4. Compare the hybrid with the LightGBM baseline on identical records, labels, preprocessing, and thresholds; do not claim superiority from rounded point estimates alone.
5. Treat the broader PTB-XL MI diagnostic superclass and the selected AMI/IMI/LMI/PMI subclass endpoint as different target definitions.
6. Do not claim quantum advantage from the presence of a quantum circuit. A suitable classical control, matched tuning budget, repeated seeds, and statistical comparisons are needed.
7. Do not commit patient-level data or restricted feature tables. Publish data access instructions and checksums where redistribution is not permitted.
8. Update the manuscript's FuzzyCred mathematical description to match `fuzzycred.py`, and report only metrics produced by this exact implementation. Historical values must not be copied into a new results table unless they are independently reproduced.

## Citation

The `CITATION.cff` file is included. Replace or supplement it with the finalized manuscript citation and archived release DOI when those details are available. No DOI or public repository URL is claimed by this project.

## License

MIT for the code in this repository. This license does not change the license or terms governing PTB-XL, 12SL, Uni-G, ECGDeli, or any third-party data and dependencies.
