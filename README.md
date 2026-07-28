# EEG Time-Domain Analysis for Alzheimer's Disease and Frontotemporal Dementia Diagnosis

Official implementation of **"Diagnosis of Alzheimer's Disease and Frontotemporal Dementia From Electroencephalography Signals"**
*IEEE Transactions on Neural Systems and Rehabilitation Engineering*, vol. 33, pp. 2160–2169, 2025.

[![Paper](https://img.shields.io/badge/DOI-10.1109%2FTNSRE.2025.3575840-blue)](https://doi.org/10.1109/TNSRE.2025.3575840)
[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

---

## Overview

This repository provides the full pipeline for classifying Alzheimer's disease (AD), frontotemporal dementia (FTD), and cognitively normal (CN) subjects from resting-state EEG using **Hjorth parameters** (Activity, Mobility, Complexity).

Instead of relying on frequency-domain representations, the method searches over sliding-window configurations (window length × moving length) to locate the temporal scale at which dementia-related signal characteristics are most pronounced. Segment-level predictions are aggregated by **hard voting** to produce a single subject-level decision.

**Key points**

- Time-domain only: 3 Hjorth parameters × 19 channels = 57 features
- Exhaustive search over 90 temporal configurations (window 20–100 s, moving 10–100 s)
- Five classifiers compared: LDA, SVM, CatBoost, LightGBM, XGBoost
- SHAP-based channel contribution analysis
- Random-forest regression of MMSE scores from Hjorth features

---

## Results

### Subject-level performance (hard voting, optimal segmentation)

| Task | Window / Moving | Best model | Accuracy | AUC |
|---|---|---|---|---|
| AD&FTD vs. CN | 80 s / 10 s | LDA | 0.925 | 0.904 |
| AD vs. CN | 80 s / 10 s | LDA | 0.886 | 0.883 |
| FTD vs. CN | 80 s / 80 s | SVM | 0.960 | 0.950 |

### Segment-level performance at optimal segmentation

| Task | Model | Acc | Sens | Spec | PPV | NPV | AUC |
|---|---|---|---|---|---|---|---|
| AD&FTD/CN (w80/m10) | LDA | 0.902 | 0.965 | 0.782 | 0.899 | 0.913 | 0.874 |
| AD/CN (w80/m10) | LDA | 0.886 | 0.926 | 0.835 | 0.891 | 0.879 | 0.881 |
| FTD/CN (w80/m80) | SVM | 0.919 | 0.859 | 0.955 | 0.951 | 0.925 | 0.907 |

### MMSE regression (AD + FTD subjects)

Pearson *r* = **0.70** (*p* = 0.01), RMSE = **3.91 ± 0.02**

---

## Dataset

Public resting-state EEG dataset from OpenNeuro: [**ds004504**](https://openneuro.org/datasets/ds004504)

| | AD | FTD | CN |
|---|---|---|---|
| n | 36 | 23 | 29 |
| Age | 66.4 ± 7.8 | 63.7 ± 8.2 | 67.9 ± 5.4 |
| MMSE | 17.8 ± 4.5 | 22.2 ± 2.6 | 30 ± 0 |
| Recording (min) | 13.5 ± 2.7 | 12.0 ± 2.3 | 13.9 ± 1.1 |

19 scalp electrodes (10–20 system), Nihon Kohden EEG 2100, 500 Hz sampling rate.

> The dataset is **not** redistributed here. Download it from OpenNeuro and place it under `data/raw/`.

---

## Repository structure

```
.
├── data/
│   ├── raw/                  # OpenNeuro ds004504 (not tracked)
│   └── processed/            # preprocessed .npy / .csv outputs
├── preprocessing/
│   ├── asr_ica_pipeline.m    # EEGLAB: ASR + ICA + bandpass + re-referencing
│   └── export_to_numpy.py
├── src/
│   ├── hjorth.py             # Activity / Mobility / Complexity
│   ├── segmentation.py       # sliding-window generation
│   ├── classify.py           # 5 classifiers, stratified 5-fold CV
│   ├── voting.py             # segment-level → subject-level hard voting
│   ├── mmse_regression.py    # random forest regression
│   └── explain_shap.py       # SHAP feature attribution
├── notebooks/
│   ├── 01_topographic_maps.ipynb
│   ├── 02_segmentation_heatmap.ipynb
│   └── 03_shap_analysis.ipynb
├── results/
├── requirements.txt
└── README.md
```

---

## Installation

```bash
git clone https://github.com/<user>/<repo>.git
cd <repo>
conda create -n hjorth-eeg python=3.10
conda activate hjorth-eeg
pip install -r requirements.txt
```

Main dependencies: `numpy`, `scipy`, `mne`, `scikit-learn`, `xgboost`, `lightgbm`, `catboost`, `shap`, `matplotlib`, `seaborn`.

Preprocessing requires **MATLAB with EEGLAB** (ASR via the `clean_rawdata` plugin, plus ICA). Preprocessed arrays can be exported to `.npy` for the Python pipeline.

---

## Usage

### 1. Preprocessing

Raw EEG → ASR → ICA (ocular/masticatory artifact removal) → Butterworth band-pass 0.5–45 Hz → re-reference to averaged mastoids (A1, A2).

```matlab
run('preprocessing/asr_ica_pipeline.m')
```

```bash
python preprocessing/export_to_numpy.py --input data/raw --output data/processed
```

### 2. Feature extraction over the temporal grid

```bash
python src/segmentation.py \
    --window-range 20 100 --window-step 10 \
    --moving-range 10 100 --moving-step 10 \
    --output data/processed/features
```

### 3. Classification

```bash
python src/classify.py --task AD_FTD_vs_CN --window 80 --moving 10 --cv 5
python src/classify.py --task AD_vs_CN     --window 80 --moving 10 --cv 5
python src/classify.py --task FTD_vs_CN    --window 80 --moving 80 --cv 5
```

### 4. Subject-level voting

```bash
python src/voting.py --predictions results/segment_preds.csv --strategy hard
```

### 5. MMSE regression and SHAP

```bash
python src/mmse_regression.py --groups AD FTD
python src/explain_shap.py --task AD_vs_CN
```

---

## Model configurations

| Model | Settings |
|---|---|
| SVM | polynomial kernel, `probability=True` |
| LDA | `solver='lsqr'`, automatic shrinkage |
| XGBoost | 200 estimators, max depth 6, lr 0.01 |
| LightGBM | 100 estimators, unrestricted depth, lr 0.01 |
| CatBoost | 200 iterations, depth 4, lr 0.1 |

Validation: stratified 5-fold cross-validation. Metrics: accuracy, sensitivity, specificity, PPV, NPV, AUC.

---

## Citation

```bibtex
@article{lee2025dementia,
  title   = {Diagnosis of {A}lzheimer's Disease and Frontotemporal Dementia From Electroencephalography Signals},
  author  = {Lee, Dong-Geun and Lee, Seung-Bo},
  journal = {IEEE Transactions on Neural Systems and Rehabilitation Engineering},
  volume  = {33},
  pages   = {2160--2169},
  year    = {2025},
  doi     = {10.1109/TNSRE.2025.3575840}
}
```

Please also cite the source dataset:

```bibtex
@article{miltiadous2023dataset,
  title   = {A Dataset of Scalp {EEG} Recordings of {A}lzheimer's Disease, Frontotemporal Dementia and Healthy Subjects from Routine {EEG}},
  author  = {Miltiadous, Andreas and others},
  journal = {Data},
  volume  = {8},
  number  = {6},
  pages   = {95},
  year    = {2023}
}
```

---

## Funding

Supported by the Bio and Medical Technology Development Program of the National Research Foundation (NRF) of South Korea, Grant RS-2024-00439078.

## License

Code released under the MIT License. The associated article is published under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

## Contact

Dong-Geun Lee — Department of Medical Informatics, Keimyung University School of Medicine, Daegu, Republic of Korea
Corresponding author: Seung-Bo Lee
