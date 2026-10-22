# Customer Churn MLOps Case Study

[Leia em Português](README.pt-BR.md)

An end-to-end, reproducible machine-learning workflow for customer churn classification. This case study demonstrates practical data-science engineering: versioning data separately from code, defining an executable pipeline, tracking outputs and metrics, and making experiments auditable with Git and DVC.

## Highlights

- Deterministic data preparation and train/test split with Python.
- Random Forest classification with scikit-learn.
- DVC versioning for data and model artifacts, without committing large files to Git.
- A dependency-aware pipeline in `dvc.yaml` and exact run hashes in `dvc.lock`.
- Reproducible experiments, collaboration, and rollback with Git + DVC.

## Business context

Customer churn is a retention problem: identifying customers likely to leave enables targeted offers, improved service, and proactive support. This project transforms a raw churn CSV into prepared data, a trained model, and a tracked metric through one reproducible workflow.

## Pipeline

```text
data/raw/churn.csv  ──► prepare ──► data/processed/{train,test}.csv
                                │
                                ▼
                           train ──► models/model.pkl
                                 └─► metrics.json
```

| Stage | Script | Inputs | Outputs |
| --- | --- | --- | --- |
| `prepare` | `src/prepare.py` | Raw churn CSV | Train/test CSV files |
| `train` | `src/train.py` | Prepared training data | Serialized model and metric |

The DAG is defined in [`dvc.yaml`](dvc.yaml). [`dvc.lock`](dvc.lock) records the precise hashes of inputs and outputs from the last successful execution.

## Current implementation

`src/prepare.py` loads `data/raw/churn.csv`, fills missing values in numeric columns with the median, then creates an 80/20 split using `random_state=42`. The fixed seed makes the result repeatable.

`src/train.py` trains a `RandomForestClassifier` with `n_estimators=50` and `random_state=42`, writes `models/model.pkl`, and records accuracy in `metrics.json`.

The latest recorded metric is **1.0 training accuracy**. This confirms that the pipeline executes, but it is not a production performance estimate because the metric is calculated on training data. A production-oriented next step is to evaluate `data/processed/test.csv` and report accuracy, precision, recall, F1, ROC-AUC, and a confusion matrix.

## Quick start

### Prerequisites

- Python 3.12+
- Git
- A DVC remote and credentials, if raw data is not already available locally

Create an isolated local environment. Virtual environments are ignored by Git.

**Windows PowerShell**

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

**macOS/Linux**

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

### Obtain the data

Git stores the small DVC pointer [`data/raw/churn.csv.dvc`](data/raw/churn.csv.dvc), not the original CSV. When a DVC remote is configured and access is available, restore data with:

```bash
dvc pull
```

Without a remote, put the authorized source file at `data/raw/churn.csv`, then register it:

```bash
dvc add data/raw/churn.csv
git add data/raw/churn.csv.dvc data/raw/.gitignore
```

### Reproduce the pipeline

```bash
dvc repro
```

DVC resolves the dependency graph, runs `prepare` before `train`, and updates `dvc.lock`. Running it again without changes skips work; changing only `src/train.py` reruns only training.

```bash
dvc status
dvc dag
dvc metrics show
dvc metrics diff HEAD~1
```

## Git + DVC hybrid versioning

| Git tracks | DVC tracks |
| --- | --- |
| Code, `dvc.yaml`, `dvc.lock`, configuration, documentation | Raw/processed data, models, and other large artifacts |
| Lightweight pointers, for example `churn.csv.dvc` | Content hashes and cache/remote objects |
| Reviewable history and collaboration | Artifact lineage and reproduction |

DVC-generated `.gitignore` files prevent the raw CSV, `data/processed/`, `models/model.pkl`, and `metrics.json` from being committed as normal Git files. Git carries the pipeline and metadata; DVC carries artifact contents in its cache and remote storage.

## Team workflow

1. Get code and artifacts: `git pull`, then `dvc pull`.
2. Change code, parameters, or source data.
3. Run `dvc repro` and inspect `dvc metrics show`.
4. Commit reproducibility metadata:

   ```bash
   git add src dvc.yaml dvc.lock requirements.txt README.md README.pt-BR.md
   git commit -m "exp: improve churn model"
   ```

5. Publish artifacts with `dvc push`, then push the Git commit.

To restore a previous experiment:

```bash
git checkout <commit-or-tag>
dvc checkout
```

`git checkout` restores the code and DVC metadata at that revision; `dvc checkout` restores matching local artifacts. Run `dvc pull` first if artifacts must be retrieved from remote storage.

## DVC remote

Choose managed, access-controlled storage such as S3, Azure Blob, Google Drive, SSH, or shared filesystem. Local demonstration only:

```bash
dvc remote add -d storage ../dvc-storage
dvc push
```

Keep credentials outside Git. Never commit secrets or sensitive customer data.

## Repository layout

```text
data/raw/          # DVC pointer and ignored source CSV
data/processed/    # Pipeline output, ignored by Git and managed by DVC
models/            # Model output, ignored by Git and managed by DVC
src/prepare.py     # Cleaning and deterministic split
src/train.py       # Training and metric creation
dvc.yaml           # Pipeline DAG
dvc.lock           # Reproducible hashes
requirements.txt   # Python dependencies
case.txt           # Original case brief
```

## MLOps next steps

- Move model settings to `params.yaml` and use `dvc exp run`.
- Evaluate against the held-out test set with business-relevant metrics.
- Add feature/schema validation and tests for leakage and model behavior.
- Add CI for linting, tests, and controlled pipeline reproduction.
- Add deployment packaging, model registry, monitoring, drift detection, and retraining criteria.
- Document data provenance, consent, retention, access control, and fairness checks before operational use.

## For reviewers

The purpose of this case is to make engineering decisions visible: lightweight metadata in Git, large artifacts in DVC, a declarative DAG, deterministic execution, and a lockfile that enables traceability. Together, these practices make experiments easier to review, reproduce, compare, and roll back safely.
