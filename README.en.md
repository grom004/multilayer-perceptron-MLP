# 🛡️ NetworkSearchAI — Network Attack Detection with ML

[Русская версия / Russian README](README.md)

This project solves **multi-class classification of network connections** based on
[Zeek/Bro `conn.log`](https://docs.zeek.org/en/master/logs/conn.html) logs. The model
predicts whether a connection belongs to one of six classes:

| Class | Description |
|---|---|
| `Normal` | Regular benign traffic |
| `Spoofing` | IP spoofing (source from an internal RFC1918 range) |
| `Suspicious` | Suspicious volume of data transferred |
| `Flooding` | Port scan (high number of unique destination ports from one IP) |
| `Brute-Force` | Targeted credential-stuffing on a narrow port range |
| `DDoS` | Large number of connections per day from a single IP |

End-to-end in a single notebook: log loading → rule-based labeling → feature engineering →
class balancing → training and comparing **four models** (MLP, RandomForest, XGBoost, LightGBM) →
detailed evaluation (ROC-AUC, PR-AUC, confusion matrix, permutation importance) →
artifact saving → ready-to-use inference function.

---

## 📑 Table of Contents

- [Features](#-features)
- [Demo](#-demo)
- [Project Structure](#-project-structure)
- [Installation](#-installation)
- [Running](#-running)
- [Pipeline Overview](#-pipeline-overview)
- [Artifacts](#-artifacts)
- [Inference](#-inference-predicting-on-a-new-record)
- [Customization](#-customization)
- [Improvements Over the Baseline](#-improvements-over-the-baseline)
- [Next Steps](#-next-steps)
- [License](#-license)

---

## 🚀 Features

- **Runs without original data** — built-in synthetic `conn.log` generator that produces
  realistic patterns for every attack type. If a real `conn.log` exists in the directory,
  it is used; otherwise synthetic data is generated. No manual switches needed.
- **Reproducibility** — fixed seeds for `random`, `numpy`, `tensorflow`; stratified
  train/val/test split.
- **Correct labeling** — explicit label hierarchy: stronger class (DDoS) overrides weaker
  ones (Spoofing). Rules live in config.
- **Rich feature space** (23 numeric + one-hot for categoricals): per-`src_ip` aggregates,
  time features, binary flags (`is_internal`, `is_well_known_port`, `is_ephemeral`),
  destination-port entropy.
- **No data leakage** — `StandardScaler` is fit **on the training set only**.
- **Full-featured MLP** — `BatchNormalization`, `Dropout`, L2 regularization,
  `EarlyStopping`, `ReduceLROnPlateau`, `ModelCheckpoint`.
- **4-model comparison** under identical conditions; the best one is auto-selected by `macro-F1`.
- **Detailed evaluation** — classification report, absolute and normalized confusion matrix,
  ROC curves (OvR), PR curves, permutation importance, top class-pair confusions.
- **Inference function** — accepts a dict (one log record) and returns the predicted class
  plus class probabilities.

---

## 🎨 Demo

### Class distribution after labeling

![Class distribution](screenshots/01_label_distribution.png)

### Top-10 IPs and hourly activity

![Top-10 IPs](screenshots/02_top_ips.png)
![Hourly activity](screenshots/03_hourly_activity.png)

### MLP training

![Accuracy](screenshots/07_mlp_accuracy.png)
![Loss](screenshots/08_mlp_loss.png)

### Confusion matrices

![Confusion matrix](screenshots/04_confusion_matrix.png)
![Normalized confusion matrix](screenshots/05_confusion_matrix_norm.png)

### Feature importance

![Feature importance](screenshots/06_feature_importance.png)

### Model comparison

![Model comparison](screenshots/09_model_comparison.png)

> ⚠️ These screenshots are from **synthetic data** — illustrative only. On a real
> `conn.log` metrics will differ.

---

## 📁 Project Structure

```
NetworkSearchAI/
├── NetworkSearchAI.ipynb     # main notebook — the entire pipeline
├── README.md                 # Russian version
├── README.en.md              # this file
├── requirements.txt          # dependencies
├── conn.log                  # ← (optional) put your real log here
├── screenshots/              # screenshots used in the README
└── artifacts/                # created on first run:
    ├── best_<model>.{keras,pkl}
    ├── model_mlp.keras
    ├── model_<name>.pkl
    ├── scaler.pkl
    ├── label_encoder.pkl
    └── metadata.json
```

---

## 💻 Installation

### Local

```bash
# 1. Clone the repository
git clone <YOUR_GITLAB_URL>/NetworkSearchAI.git
cd NetworkSearchAI

# 2. Create a virtual environment
python3 -m venv .venv
source .venv/bin/activate          # Linux / macOS
# .venv\Scripts\activate           # Windows

# 3. Install dependencies
pip install -r requirements.txt

# 4. Launch Jupyter
jupyter notebook NetworkSearchAI.ipynb
```

**Requirements:** Python 3.10+, ≈4 GB RAM for the synthetic mode, ≥16 GB for a full
`conn.log` from a real environment.

### Google Colab

1. Open [Google Colab](https://colab.research.google.com/) → **File → Upload notebook** →
   upload `NetworkSearchAI.ipynb`.
2. **(Optional)** Upload your `conn.log.zip` to the Colab files panel and unzip:
   ```python
   !unzip /content/conn.log.zip -d /content/
   ```
   Then set `CFG["log_path"] = "/content/conn.log"` in the config cell.
3. Use **Runtime → Change runtime type → GPU** (or TPU for very large logs).
4. **Runtime → Run all**.

Without a real log file, the notebook will simply generate synthetic data.

---

## ▶️ Running

### Quick start (synthetic data)

```bash
jupyter notebook NetworkSearchAI.ipynb
# → Cell → Run All
```

End-to-end runtime: ≈ **2–5 minutes** on CPU without a real log.

### With a real log file

1. Place `conn.log` in the project root (or set `CFG["log_path"]` to a different path).
2. If the file is compressed: `unzip conn.log.zip`.
3. Run the notebook.

Runtime depends on log size: from minutes to hours. Large files benefit from GPU/TPU.

---

## 🔬 Pipeline Overview

```
┌─────────────────┐  ┌──────────────┐  ┌──────────────────┐  ┌──────────────┐
│ 1. Loading      │→ │ 2. Cleaning  │→ │ 3. Rule-based    │→ │ 4. EDA       │
│ (real or syn.)  │  │ + typing     │  │    labeling      │  │              │
└─────────────────┘  └──────────────┘  └──────────────────┘  └──────────────┘
                                                                     │
                                                                     ▼
┌──────────────────┐  ┌──────────────┐  ┌────────────┐  ┌───────────────────┐
│ 8. Inference     │← │ 7. Artifacts │← │ 6. Evalu-  │← │ 5. Feature        │
│    function      │  │    saving    │  │    ation   │  │    engineering    │
└──────────────────┘  └──────────────┘  └────────────┘  │    + balancing    │
                                                        │    + train 4 mdls │
                                                        └───────────────────┘
```

### Labeling rules

Applied **from weakest to strongest**; stronger classes overwrite weaker ones:

| Class | Condition |
|---|---|
| `Spoofing` | `src_ip` is RFC1918 (`10.*`, `192.168.*`, `172.16-31.*`) or IPv6 link-local |
| `Suspicious` | Total bytes for an `src_ip` > **1 GB** |
| `Flooding` | Unique `dst_port` per `src_ip` > **500** |
| `Brute-Force` | Sessions per `src_ip` > **50** **and** unique ports in **5–30** (focused attack, not a scan) |
| `DDoS` | Connections per `src_ip` per day > **1000** |

All thresholds live in `CFG["rules"]` and are easy to tune.

### MLP architecture

```
Input(N)
  → Dense(256, ReLU, L2=1e-4) → BatchNorm → Dropout(0.4)
  → Dense(128, ReLU, L2=1e-4) → BatchNorm → Dropout(0.3)
  → Dense(64,  ReLU, L2=1e-4) → BatchNorm → Dropout(0.2)
  → Dense(32,  ReLU)
  → Dense(N_CLASSES, Softmax)

Optimizer: Adam(1e-3)
Loss:      sparse_categorical_crossentropy
Callbacks: EarlyStopping, ReduceLROnPlateau, ModelCheckpoint
```

### Compared models

| Model | Key hyperparameters |
|---|---|
| **MLP** (TF/Keras) | 4 hidden layers, BN, Dropout, L2, Adam, 40 epochs max |
| **RandomForest** | 300 trees, `class_weight="balanced"` |
| **XGBoost** | 400 trees, depth=6, lr=0.1, hist tree method |
| **LightGBM** | 400 trees, 63 leaves, lr=0.05, `class_weight="balanced"` |

---

## 💾 Artifacts

After running the notebook you'll find in `artifacts/`:

| File | Description |
|---|---|
| `best_<model>.keras` or `.pkl` | Best model by `macro-F1` |
| `model_mlp.keras`, `model_<name>.pkl` | All trained models |
| `scaler.pkl` | Fitted `StandardScaler` |
| `label_encoder.pkl` | `LabelEncoder` for the class names |
| `metadata.json` | Metrics, feature list, config, timestamp |

---

## 🎯 Inference (predicting on a new record)

```python
import joblib, json
from tensorflow.keras.models import load_model

# 1. Load artifacts
scaler = joblib.load("artifacts/scaler.pkl")
le     = joblib.load("artifacts/label_encoder.pkl")
meta   = json.load(open("artifacts/metadata.json"))

# 2. Load the best model
best = meta["best_model"]
if best == "MLP":
    model = load_model("artifacts/best_mlp.keras")
else:
    model = joblib.load(f"artifacts/best_{best.lower()}.pkl")

# 3. Predict (see predict_one() in the notebook)
record = {
    "timestamp": 1_700_001_234,
    "session_id": "Cabc12345",
    "src_ip": "10.0.5.13",
    "src_port": 54321,
    "dst_ip": "8.8.8.8",
    "dst_port": 22,
    "protocol": "tcp",
    "app_protocol": "ssh",
    "duration": 0.4,
    "bytes_sent": 150,
    "bytes_received": 30,
    "packets_sent": 3,
    "packets_received": 1,
}
class_name, probabilities = predict_one(record)
print(class_name, probabilities)
```

> ⚠️ **About aggregate features.** Features such as `src_n_sess`, `src_total_bytes`
> are computed over the entire dataset during training. In production, these values
> must be served from a background store (in-memory store / Redis / Materialize).
> For single-record inference `predict_one` zeroes them out.

---

## ⚙️ Customization

All key parameters live in a single `CFG` dictionary inside the notebook:

```python
CFG = {
    "log_path":          "conn.log",
    "artifacts_dir":     "artifacts",
    "use_synthetic":     "auto",          # "auto" | True | False
    "synthetic_size":    25_000,

    "rules": {
        "ddos_per_day":       1000,
        "brute_force_sess":   50,
        "brute_force_ports":  (5, 30),
        "suspicious_bytes":   1e9,
        "flooding_ports":     500,
    },

    "test_size":         0.2,
    "val_size":          0.2,
    "random_state":      42,
    "epochs":            40,
    "batch_size":        256,
    "patience":          7,
}
```

---

## 📈 Improvements Over the Baseline

| # | Before | After |
|---|---|---|
| 1 | `scaler.fit_transform(X)` on the entire dataset → leakage | `fit_transform` on train only, `transform` on val/test |
| 2 | Labeling rules applied in arbitrary order (Spoofing overrode attacks) | Explicit hierarchy weakest → strongest + config |
| 3 | Brute-Force: `n_ports > 5` (caught scanners too) | `n_ports ∈ (5, 30)` — focused attack only |
| 4 | Raw row fields only | + 16 engineered features (aggregates, ratios, flags, time) |
| 5 | Single train/test split, no validation set | Stratified train / val / test |
| 6 | MLP without regularization | + BatchNorm, Dropout, L2, ReduceLROnPlateau, ModelCheckpoint |
| 7 | MLP only | Compared with RandomForest, XGBoost, LightGBM |
| 8 | Only accuracy and confusion matrix | + ROC-AUC, PR-AUC, permutation importance, error analysis |
| 9 | `model.save('*.h5')` (legacy) | Modern `.keras` + scaler/encoder/feature-names saved |
| 10 | No seed fixing | Full reproducibility |
| 11 | Can't run without `conn.log.zip` | Built-in synthetic generator |
| 12 | No inference function | `predict_one(record)` |

---

## 🔮 Next Steps

- **Real labeled datasets**: [CICIDS2017](https://www.unb.ca/cic/datasets/ids-2017.html),
  [UNSW-NB15](https://research.unsw.edu.au/projects/unsw-nb15-dataset) — replace heuristic
  rules with ground truth.
- **Rolling-window aggregates** by `src_ip` over the last N minutes instead of global.
- **Sequence models** — 1D-CNN or Transformer on a sequence of connections per IP.
- **Stratified K-Fold** instead of a single split for more stable estimates.
- **Probability calibration** — `CalibratedClassifierCV` for interpretable thresholds.
- **Serving** — wrap the best model in FastAPI + Docker for online predictions.

---

## 📜 License

MIT — see [LICENSE](LICENSE) if present, or use freely.
