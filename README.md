# Indian Electricity Demand Forecasting

Forecast the next **24 hourly electricity-demand values** from the previous **168 hours (7 days)** using a PyTorch multilayer perceptron (MLP). Compare the neural network against persistence and seasonal-naive forecasting baselines.

## Project overview

This project processes Indian electricity-demand measurements from Excel workbooks, converts five-minute SCADA readings into hourly demand, creates chronological forecasting windows, trains a PyTorch MLP, and evaluates its forecasts against simple baselines.

### Forecasting setup

| Item | Configuration |
|---|---|
| Target measurement | `NLDC_DEMAND|P` |
| Input lookback | Previous 168 hourly values |
| Forecast horizon | Next 24 hourly values |
| Input source | Monthly `.xlsx` workbooks |
| Worksheet | `Sheet1` |
| Source sampling frequency | 5 minutes |
| Model | PyTorch MLP |
| Train / validation / test split | 70% / 15% / 15% by usable timestamps |
| Evaluation | MAE, RMSE, WAPE |

## Repository structure

```text
india-electricity-demand-forecasting/
├── india_electricity_demand_forecasting.ipynb
├── README.md
├── requirements.txt
├── .gitignore
├── data/
│   └── raw/       # Place monthly source .xlsx files here
└── outputs/       # Generated plots, metrics and model checkpoint
```

## Dataset and workbook format

The notebook expects monthly Excel workbooks (`.xlsx`) in `data/raw/`.

The inspected workbook layout is:

- Worksheet: `Sheet1`
- Row 1: SCADA tag names
- Row 2: measurement labels
- Row 3 onward: observations
- Timestamp column is identified by the tag name `Time`
- Demand target is selected by the measurement label `NLDC_DEMAND|P`

The source observations are recorded every five minutes. The notebook averages repeated timestamps and resamples the demand readings to hourly means.

**Dataset files are not included.** Add only data that you are permitted to use and redistribute. The `.gitignore` excludes Excel files under `data/raw/` by default so that large or restricted source data are not accidentally committed.

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/india-electricity-demand-forecasting.git
cd india-electricity-demand-forecasting
```

### 2. Create a virtual environment

Windows PowerShell:

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

macOS / Linux:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

For GPU training, install a PyTorch build compatible with your installed NVIDIA driver and CUDA runtime. See the official [PyTorch installation selector](https://pytorch.org/get-started/locally/).

### 4. Add the data

Download the relevant monthly workbooks and place them in:

```text
data/raw/
```

For meaningful multi-year forecasting, use a sufficiently complete set of monthly files. The notebook will report missing hours and long gaps.

### 5. Run the notebook

```bash
jupyter notebook
```

Open `india_electricity_demand_forecasting.ipynb` and run all cells from top to bottom.

The notebook automatically selects CUDA when available; otherwise, it uses CPU.

## Methodology

1. **Load workbooks:** read `Sheet1`, identify the `Time` column and select the `NLDC_DEMAND|P` measurement by its label.
2. **Clean observations:** parse timestamps and numeric demand values, remove invalid rows, sort observations and average duplicate timestamps.
3. **Hourly aggregation:** resample five-minute readings to hourly means. Only short internal gaps of up to two hours are interpolated; longer gaps remain missing.
4. **Build contiguous segments:** split at long missing-hour gaps so training examples never span discontinuities.
5. **Create forecasting windows:** use 168 hourly observations as input and the following 24 hours as the target.
6. **Prevent leakage:** use chronological 70% / 15% / 15% partitions and calculate normalization statistics using training-period values only.
7. **Train the MLP:** use Adam, MSE loss, dropout, weight decay and early stopping based on validation loss.
8. **Evaluate:** compare predictions in the original demand scale against persistence, daily seasonal-naive and weekly seasonal-naive baselines.

## Model architecture

The feed-forward MLP maps a 168-dimensional input window to 24 forecast values:

```text
168 hourly inputs
       |
Linear(168, 256) + ReLU + Dropout(0.1)
       |
Linear(256, 128) + ReLU + Dropout(0.1)
       |
Linear(128, 24)
       |
Next 24 hourly demand values
```

Training configuration in the notebook:

- Optimizer: Adam
- Learning rate: `1e-3`
- Weight decay: `1e-4`
- Batch size: `64`
- Maximum epochs: `100`
- Early-stopping patience: `10`
- Loss: Mean Squared Error (MSE)
- Random seed: `42`

## Baseline models

- **Persistence:** repeats the most recent observed demand for all 24 forecast hours.
- **Daily seasonal naive:** uses the most recent 24 hours as the next day's forecast.
- **Weekly seasonal naive:** uses the corresponding 24-hour period from 168 hours earlier.

These baselines help determine whether the neural network adds predictive value beyond simple temporal patterns.

## Evaluation metrics

- **MAE (Mean Absolute Error):** average absolute difference between actual and predicted demand.
- **RMSE (Root Mean Squared Error):** gives larger errors more weight.
- **WAPE (Weighted Absolute Percentage Error):** total absolute error divided by total absolute actual demand, expressed as a percentage.

The notebook also calculates MAE by forecast lead hour (hour 1 through hour 24). Results are exported in the original demand units after reversing the training-only scaling.

## Generated outputs

The notebook saves files to `outputs/`:

- `demand_mlp.pt` — trained model checkpoint and normalization parameters
- `mlp_training_curve.png` — training and validation loss
- `hourly_demand_overview.png` — hourly demand overview, if generated by the run
- `average_demand_by_hour.png` — average demand by hour, if generated by the run
- `example_24_hour_forecast.png` — example 24-hour forecast
- `test_model_comparison.csv` — test-set model comparison
- `test_metrics_by_forecast_hour.csv` — MAE by forecast lead hour

Generated outputs are ignored by Git by default; regenerate them by running the notebook.

## Data coverage and limitations

The notebook's saved sample run showed a calendar span from **2021-09-01 to 2023-10-31**, with **4,396 usable hourly values out of 18,984 hourly slots** and two very long gaps of approximately 304 days. This is sparse coverage, not a continuous two-year time series.

Important implications:

- Forecast windows are not created across long missing-hour gaps.
- A split may contain too few valid windows if the downloaded files do not provide enough continuous hourly data.
- A few isolated monthly workbooks are not sufficient to support broad claims about continuous multi-year demand forecasting.
- Download all available monthly files and rerun the notebook before reporting final performance.
- Confirm the source units and measurement definitions from the dataset documentation before interpreting forecasts.
- The model forecasts the selected aggregate measurement only; it does not identify causal drivers of demand.

## Reproducibility

Keep the source workbook versions, Python/PyTorch versions, split configuration and random seed consistent when comparing experiments. Report the actual test metrics produced by your run; do not pre-state that the MLP outperforms the baselines.

## License

No license is specified. Add a license only after deciding how the notebook and any redistributable material may be reused.
