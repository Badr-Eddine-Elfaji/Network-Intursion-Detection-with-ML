# Network Intrusion Detection System
### ML-based classification of network attacks on the CSE-CIC-IDS2018 dataset

---

This project is the practical implementation behind my Master's thesis in Decision Engineering. The goal was to build a realistic, end-to-end intrusion detection pipeline that goes beyond the typical benchmark-and-report approach common in the literature. Three classifiers were trained, tuned, and then stress-tested on a naturally imbalanced real-world holdout, rather than only the clean balanced split.

---

## The Problem

Signature-based IDS tools can only catch attacks they already know. This project takes the anomaly-based route: train ML models on labeled network flow data so they can recognize attack behavior on their own, including across 14 different attack categories simultaneously.

The dataset used is the **CSE-CIC-IDS2018**, generated on Amazon AWS by the Canadian Institute for Cybersecurity. It contains realistic bidirectional network flows extracted with CICFlowMeter and covers a wide spectrum of threats.

---

## Attack Classes Covered

| Category | Examples |
|---|---|
| Volumetric DoS/DDoS | DoS Hulk, DoS Slowloris, SlowHTTPTest, DDoS HOIC, LOIC-HTTP, LOIC-UDP, GoldenEye |
| Brute Force | SSH-BruteForce, FTP-BruteForce, Brute Force-Web, Brute Force-XSS |
| Injection | SQL Injection |
| Botnet | Bot traffic |
| Infiltration | Infiltration attempts |
| Benign | Normal traffic |

---

## Pipeline Overview

```
Raw CSE-CIC-IDS2018 CSVs (Kaggle)
        |
        v
  Merge + Deduplicate --> 4.45 GB unified file
        |
        +--------> 10% holdout (real-world imbalanced, never touched until evaluation)
        |
        v
  90% training pool --> cap at 100,000 rows/class --> balanced training set
        |
        v
  EDA on the 10% subset (univariate, bivariate, multivariate)
        |
        v
  Feature selection (26 flow-level features)
        |
        v
  80/20 train-test split --> SMOTE on train only
        |
        v
  Train RF + XGBoost (RandomizedSearchCV, f1_macro) + DNN (EarlyStopping)
        |
        v
  Evaluate on balanced test split --> save models as .pkl
        |
        v
  Re-evaluate on the 10% real-world holdout
```

---

## Models

### Random Forest
- Baseline: 100 trees, `class_weight='balanced'`
- Tuned with `RandomizedSearchCV` (20 iterations, 3-fold CV, optimized for `f1_macro`)
- Serialized as `RF_model.pkl`

### XGBoost
- Baseline: 100 rounds, `eval_metric='mlogloss'`, `n_jobs=-1`
- Tuned: `n_estimators=300`, `learning_rate=0.1`, `max_depth=9`, `subsample=0.8`, `colsample_bytree=0.7`, `gamma=0.1`
- Optimization took ~146 minutes
- Serialized as `XGB_model.pkl`

### Deep Neural Network
- Dense layers with Batch Normalization and Dropout (rate=0.15)
- Optimizer: Adam | Loss: categorical crossentropy
- Training: batch size 1024, up to 70 epochs, EarlyStopping on val_loss (patience=5)
- Output: Softmax over 15 classes

---

## Results (Balanced Test Split)

| Metric | Random Forest | XGBoost | DNN |
|---|---|---|---|
| Accuracy | 0.94 | 0.94 | 0.93 |
| Macro Precision | 0.80 | 0.83 | **0.84** |
| Macro Recall | **0.85** | **0.85** | 0.80 |
| Macro F1-score | 0.81 | **0.84** | 0.80 |
| Weighted F1-score | **0.94** | **0.94** | 0.93 |

All three models achieve near-perfect scores on high-volume attacks (DDoS HOIC, LOIC-UDP, Bot, SSH-BruteForce). The harder challenge, and the more honest test, is what happens on real data.

---

## Real-World Evaluation (10% Imbalanced Holdout)

This is where the results get interesting. The 10% holdout was set aside before any preprocessing and never touched during training. It preserves the original class distribution, meaning common attacks dominate and rare ones may appear only a handful of times.

**What held up:** Volumetric DoS/DDoS, Bot traffic, and SSH-BruteForce remain strongly detected across all three models. These classes have large real-world support and distinct enough behavioral signatures (bandwidth, port concentration, flag patterns) that the learned boundaries transfer cleanly.

**What degraded:** Precision collapsed for ultra-rare classes like Brute Force-Web (56 real samples), SQL Injection (8), and FTP-BruteForce (5). SMOTE generated synthetic examples to balance these during training, but the resulting decision boundaries were too permissive on real traffic. Recall stayed reasonable; precision did not.

**DNN-specific issue:** Unlike the tree-based models, the DNN also missed detections on SlowHTTPTest and Brute Force-XSS, likely because it was evaluated without hyperparameter tuning.

The main takeaway: a strong macro F1 on a balanced benchmark does not guarantee equivalent performance in production. This project explicitly tests both, which is relatively uncommon in the CICIDS-2018 literature.

---

## Feature Selection

26 flow-level features were selected as the common input for all three models. The selection combined:
- Zero-variance feature removal (`Fwd Byts/b Avg`, `Bwd Blk Rate Avg`, etc. dropped entirely)
- Univariate and bivariate behavioral analysis (protocol, port concentration, flag mechanics, window sizes, bandwidth, timing)
- Feature importance rankings from RF and XGBoost

Key discriminative features include: `Flow Byts/s`, `Flow Pkts/s`, `Flow Duration`, `Dst Port`, `Init Fwd Win Byts`, `ECE Flag Cnt`, `Active Mean`, `Flow IAT Mean`, `RST Flag Cnt`, and TCP flag profiles.

---

## Class Imbalance Handling

SMOTE (`k_neighbors=3`) was applied exclusively to the training set after the 80/20 split. It was never applied to the test set or the real-world holdout. The low `k` value was chosen to safely accommodate the most underrepresented classes without generating noisy synthetic samples.

---

## Tech Stack

- **Python** (pandas, numpy, scikit-learn, imbalanced-learn, XGBoost, TensorFlow/Keras, matplotlib, seaborn)
- **Jupyter Notebooks** for EDA and modeling
- **pickle** for model serialization

---

## Repository Structure

```
.
|-- notebooks/
|   |-- 01_EDA.ipynb                  # Exploratory data analysis on the 10% subset
|   |-- 02_Feature_Selection.ipynb    # Feature importance + final 26-feature selection
|   |-- 03_Modeling_RF.ipynb          # Random Forest training and tuning
|   |-- 04_Modeling_XGB.ipynb         # XGBoost training and tuning
|   |-- 05_Modeling_DNN.ipynb         # Neural network training
|
|-- models/
|   |-- RF model is too large, so it won't be uploaded to github, if you want it use the notebook to train it.
|   |-- XGB_model.pkl
|   |-- XGB_label_encoder.pkl
|   |-- DNN_model.keras                    
|   |-- DNN_scaler.pkl                    
|   |-- DNN_label_encoder.pkl                    
|
|-- requirements.txt
|-- README.md
```

> **Note:** The dataset is not included due to its size (4.45 GB merged). It can be downloaded from [Kaggle](https://www.kaggle.com/).

---

## Limitations

- SMOTE-generated boundaries do not transfer cleanly to ultra-rare real-world attack classes. Alternative balancing strategies (cost-sensitive learning, hybrid resampling, balanced bagging) are worth exploring.
- The real-world holdout, while imbalanced, still comes from the same CICIDS-2018 source. Validation on an independent network environment would be a stronger test of deployability.
- The DNN was evaluated without hyperparameter tuning; its results likely understate what the architecture could achieve with proper optimization.

---

## Academic Context

**Thesis:** Application of Datamining and Machine Learning in Cybersecurity  
**Case Study:** CSE-CIC-IDS2018 Dataset  
**Program:** Master's in Decision Engineering (M2)  
**Institution:** Hassan 1st University, FEG Settat, Morocco  
**Supervisor:** Pr. Abdeljalil El Ouardighi  
**Academic Year:** 2025-2026  
**Author:** Badreddine Elfaji

---

## Contact

Feel free to reach out via [LinkedIn](https://www.linkedin.com/in/badreddine-elfaji) or open an issue on this repo.
