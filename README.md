# Behavioral authentication: feature extraction, model, calibration, ablation, and adversarial test

Reference implementation for the manuscript "Augmenting Graphical Password
Authentication with Mouse-Dynamics Behavioral Biometrics: System Design,
Formal Analysis, and Proxy Evaluation on Remote-Desktop Data". The script
reproduces every number and figure in Sections 12, 13 and 15 of the
manuscript.

## Dataset

The Balabit Mouse Dynamics Challenge dataset (Fulop, Kovacs, Kurics, and
Windhager-Pokol, 2016), publicly available at
https://github.com/balabit/Mouse-Dynamics-Challenge

```bash
git clone https://github.com/balabit/Mouse-Dynamics-Challenge.git
```

## Installation

```bash
pip install -r requirements.txt
```

Python 3.9 or later.

## Running

```bash
python run_experiments.py --data-root /path/to/Mouse-Dynamics-Challenge \
                           --output-dir ./artifacts
```

The run takes a few minutes. It performs the following steps.

1. Segments each session into click-terminated attempts and extracts the
   eight-dimensional feature vector (Equations 3-7).
2. Fits one softmax classifier per feature arm (full, behavioral only,
   region only) on the training sessions, holding out the most recent
   training session of each account for validation.
3. Splits the 816 publicly labeled test sessions, by whole session and
   stratified by account and genuine/impostor label, into a calibration
   partition and a test partition. The decision threshold is chosen on the
   calibration partition (FAR = FRR point) and all reported error rates are
   measured on the test partition at that threshold.
4. Computes session-level bootstrap intervals and repeats the division 20
   times with different seeds.
5. Runs the Monte Carlo perturbation test and the combined risk at the
   calibrated threshold of each arm, and at tau = 0.75 for comparison.

Outputs in `--output-dir`: `results.json` (all numbers), `training_history.json`,
and the figures `training_curve.pdf`, `pca_projection.pdf`,
`far_frr_curve.pdf`, `entropy_vs_probability.pdf`, `asr_histogram.pdf`
(vector PDF).

## Reproducibility

Parameters are initialized to zero, gradient descent is deterministic, the
training/validation split is defined by session file order, and every random
operation (calibration/test division, bootstrap, subsampling for the PCA
plot, Monte Carlo noise) uses a fixed seed (`SEED = 20260902` in
`run_experiments.py`). Re-running against an unmodified copy of the dataset
reproduces `expected_results.json`, which is included for comparison.

| Quantity | Value |
|---|---|
| Training / validation accuracy (full model) | 21.9% / 23.7% |
| Calibrated threshold (full model) | 0.098 |
| Test FAR / FRR / HTER (full model) | 41.2% / 40.0% / 40.6% |
| Test HTER, behavioral arm / region arm | 46.7% / 40.8% |

## Layout

```
behavioral_auth/
├── requirements.txt
├── expected_results.json
├── run_experiments.py      # end-to-end experiments
└── src/
    ├── data_loader.py      # session loading, click-terminated segmentation
    ├── features.py         # eight-dimensional feature vector
    ├── model.py            # normalization, softmax model, gradient-descent training
    ├── protocol.py         # labeled sessions, calibration/test division
    ├── evaluate.py         # FAR, FRR, EER, HTER, session bootstrap
    ├── adversarial.py      # slack risk, Monte Carlo perturbation test
    └── figures.py          # vector figures
```

## Adapting the region features

`src/features.extract_features` defines the region ratios R1-R4 as screen
quadrants split at the median coordinates observed in the training sessions,
because the dataset comes from unconstrained remote-desktop use rather than a
four-region password interface. For a real four-region interface, replace the
quadrant test with that interface's region boundaries; the rest of the
pipeline is unchanged.
