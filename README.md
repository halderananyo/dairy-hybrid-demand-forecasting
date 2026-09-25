# Hybrid Time Series & ML Approach for Multi-Echelon Supply Chain Demand Forecasting

**Case study: a 3-echelon dairy supply chain (Farm → Distribution → Retail)**

This repository holds the two-stage hybrid forecasting pipeline built for this thesis work:
ARIMA is fit across every product×location series in the dataset, and a Random Forest is then
trained on the **ARIMA residuals** to capture the nonlinear structure ARIMA misses — combining
the two into a hybrid forecast.

## Pipeline

1. **Stage 1 — ARIMA (all series)** — `notebooks/01_ARIMA_AllSeries.ipynb`
   Fits ARIMA independently on every valid product×location series across all three echelons
   (a length gate excludes series too short to fit reliably), walk-forward forecasts the test
   period, and saves per-series metrics plus month-by-month residuals (`Actual − ARIMA_Predicted`,
   in original target units) to `arima_residuals_for_rf.xlsx` — the bridge file for Stage 2.

2. **Stage 2 — Random Forest on residuals** — `notebooks/02_RF_Hybrid.ipynb`
   Loads the residual file, builds calendar/festival/lag/rolling features, and trains a Random
   Forest to predict the leftover ARIMA residual per echelon. Two hybrid variants are compared:
   - **Hybrid-resid**: `ARIMA_Pred + RF_predicted_residual`
   - **Hybrid-augmented**: RF also receives `ARIMA_Pred` as an input feature
   All four approaches (ARIMA alone, RF-only, Hybrid-resid, Hybrid-augmented) are evaluated
   against each other per echelon.

## Echelons

| Echelon | Target | Grouping |
|---|---|---|
| E1 — Farm | `Quantity_Produced` | Location × Product |
| E2 — Distribution | `Quantity_Sold` | Location × Product × Sales Channel |
| E3 — Retail | (retail-level demand) | Location × Product |

## Repository structure

```
├── notebooks/
│   ├── 01_ARIMA_AllSeries.ipynb   # Stage 1: per-series ARIMA, walk-forward forecast, residuals
│   └── 02_RF_Hybrid.ipynb         # Stage 2: RF on residuals, hybrid comparison, feature importance
├── reports/
│   ├── 01_ARIMA_AllSeries.html    # Full rendered notebook — EDA plots, ARIMA diagnostics
│   └── 02_RF_Hybrid.html          # Full rendered notebook — results tables, comparison plots
├── data/
│   ├── dairy_thesis_dataset_3echelons.xlsx   # Raw 3-echelon dairy demand dataset
│   └── arima_residuals_for_rf.xlsx           # Bridge file: ARIMA residuals + features (output of notebook 01, input to notebook 02)
├── requirements.txt
├── LICENSE
└── .gitignore
```

The `reports/` HTML files are the executed notebooks with all outputs (EDA visuals, ARIMA
diagnostics, results tables, feature importance) preserved — open these directly in a browser
to see the full analysis without re-running anything. The `notebooks/` versions have outputs
stripped to keep the repo lightweight; re-run them in order (01 → 02) to regenerate everything.

## Running the pipeline

```bash
pip install -r requirements.txt
jupyter notebook notebooks/02_RF_Hybrid.ipynb
```

`data/arima_residuals_for_rf.xlsx` is already included, so you can run `02_RF_Hybrid.ipynb`
directly without re-running Stage 1. To reproduce everything from the raw data instead, run
`01_ARIMA_AllSeries.ipynb` first (it regenerates `arima_residuals_for_rf.xlsx` plus per-series
metrics into an `ARIMA_Results/` folder), then run `02_RF_Hybrid.ipynb`. Both notebooks expect
their input file(s) to be in the same folder they're run from (or update `BASE_DIR` at the top
of each notebook).

## Key results (MAE by echelon, pooled — lower is better)

| Echelon | ARIMA | Hybrid-resid | Hybrid-augmented | Best |
|---|---|---|---|---|
| E1 — Farm | 411.17 | 409.02 | **362.94** | Hybrid-augmented |
| E2 — Distribution | 241.81 | 236.56 | **204.11** | Hybrid-augmented |
| E3 — Retail | 253.91 | 260.90 | **210.24** | Hybrid-augmented |

| Echelon | ARIMA (RMSE) | Hybrid-resid (RMSE) | Hybrid-augmented (RMSE) | Best |
|---|---|---|---|---|
| E1 — Farm | 565.69 | 561.83 | **472.88** | Hybrid-augmented |
| E2 — Distribution | 333.04 | 319.79 | **257.20** | Hybrid-augmented |
| E3 — Retail | 336.62 | 354.37 | **271.82** | Hybrid-augmented |

The feature-augmented hybrid (RF sees `ARIMA_Pred` as an input feature, not just a residual
target) wins on both MAE and RMSE across all three echelons. Full results tables, per-series
breakdowns, and feature-importance plots are in `reports/02_RF_Hybrid.html`.

## License

MIT — see [LICENSE](LICENSE).
