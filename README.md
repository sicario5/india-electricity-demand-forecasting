# Indian Electricity Demand Forecasting (PyTorch)

Forecast the next **24 hourly values** of all-India electricity demand from the previous **168 hours (7 days)** using a PyTorch MLP, and benchmark it against persistence and seasonal-naive baselines.

## Results

Test set: 637 windows covering 2023-10-04 to 2023-10-31 (demand in source units, mean ≈ 162,000).

| Model                 |      MAE |     RMSE | WAPE (%) |
|-----------------------|---------:|---------:|---------:|
| Daily seasonal naive  | **3,845** | **5,342** | **2.04** |
| PyTorch MLP           |    5,993 |    7,794 |     3.17 |
| Weekly seasonal naive |    8,846 |   11,010 |     4.68 |
| Persistence           |   14,366 |   17,871 |     7.60 |

**Takeaways**
- The MLP clearly beats persistence (~58% lower MAE) and weekly seasonal-naive (~32% lower MAE).
- It does **not** beat the daily seasonal-naive baseline in this run. Demand here is dominated by a strong daily cycle, and the MLP sees only about 2,500 training windows.
- Persistence is competitive only at forecast hour 1 (MAE 3,570 vs 4,484 for the MLP) and degrades quickly after that.

Reporting an honest baseline comparison is deliberate: a model is only useful if it beats the simple alternatives.

## Setup

| Item                 | Configuration                                |
|----------------------|----------------------------------------------|
| Target               | `NLDC_DEMAND\|P` (selected by measurement label) |
| Input / horizon      | 168 hourly values → next 24 hourly values    |
| Source               | Monthly `.xlsx` workbooks, worksheet `Sheet1` |
| Preprocessing        | Duplicate timestamps averaged, resampled to hourly mean, gaps ≤ 2 h interpolated |
| Split (chronological)| Train 2,550 / validation 613 / test 637 windows |
| Scaling              | z-score fitted on training period only       |
| Model                | MLP 168 → 256 → 128 → 24, ReLU, dropout 0.1  |
| Training             | Adam (lr 1e-3, weight decay 1e-4), MSE loss, batch 64, early stopping (patience 10; best epoch 13) |
| Metrics              | MAE, RMSE, WAPE in original demand units     |

## Methodology

1. **Load** each workbook, find the `Time` column and the `NLDC_DEMAND|P` column by label.
2. **Clean** timestamps and values, average duplicates, resample to hourly means.
3. **Segment** the series at gaps so no window spans missing data (3 usable segments of ~1,465 hours each).
4. **Window** into 168-hour inputs and 24-hour targets.
5. **Split chronologically** (70/15/15 of usable timestamps) with no shuffling across splits; normalisation statistics come from the training period only to avoid leakage.
6. **Train** the MLP with early stopping on validation loss.
7. **Evaluate** on the held-out test period against three baselines and report MAE by forecast hour.

## Data and limitations

- Six monthly workbooks were used: **September and October of 2021, 2022 and 2023**. This is not a continuous multi-year series: 4,396 usable hourly values out of 18,984 hourly slots, with two gaps of about 304 days.
- Sampling frequency differs between files: 2021 files and September 2022 are 5-minute data, while October 2022 and the 2023 files are about 10-second data. All are averaged to hourly values.
- Training covers Sep–Oct 2021, Sep–Oct 2022 and the first week of Sep 2023. Validation and test are both from autumn 2023, so results reflect **one season** and may not generalise to other times of year.
- Windows overlap heavily (stride of 1 hour), so the 637 test windows are far fewer independent samples than the count suggests.
- Confirm the units and measurement definition of `NLDC_DEMAND|P` in the dataset documentation before interpreting absolute values.
- The model forecasts the aggregate series only; it says nothing about causes of demand.

## Ideas for improvement

- Download all available months for a full-year, multi-year series.
- Add calendar features (hour of day, weekday, holidays) and weather as inputs.
- Predict the residual over the daily seasonal-naive forecast instead of the raw series.
- Try an LSTM or temporal convolution, and walk-forward (rolling-origin) validation.

## Repository structure

```
india-electricity-demand-forecasting/
├── india_electricity_demand_forecasting.ipynb
├── README.md
├── requirements.txt
├── .gitignore
├── data/raw/      # monthly .xlsx files (not included)
└── outputs/       # plots, metrics CSVs, model checkpoint
```

## How to run

```bash
git clone https://github.com/sicario5/india-electricity-demand-forecasting.git
cd india-electricity-demand-forecasting
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

1. Place the monthly `.xlsx` workbooks in `data/raw/`.
2. In the first code cell, set `DATA_DIR = Path("data/raw")` and `OUTPUT_DIR = Path("outputs")`.
3. Run `jupyter notebook` and execute all cells top to bottom. CUDA is used automatically if available, otherwise CPU.

The workbooks are excluded from Git by `.gitignore`. Only use data you are permitted to use and redistribute.

## Outputs

Saved to `outputs/`: `demand_mlp.pt` (checkpoint with scaling parameters), `mlp_training_curve.png`, `example_24_hour_forecast.png`, `test_model_comparison.csv`, `test_metrics_by_forecast_hour.csv`.

## Tech stack

Python, PyTorch, pandas, NumPy, scikit-learn, Matplotlib.
