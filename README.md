# Cyclone Sensor Analysis — `read.md`

Project purpose: time-series analysis of 3 years of cyclone sensor data (5-minute resolution) to (1) prepare & explore data; (2) detect shutdown/idle periods; (3) segment machine states via clustering; (4) perform contextual anomaly detection and root-cause hypotheses; (5) short-horizon forecasting (1 hour / 12 steps); and (6) produce actionable insights and monitoring recommendations.



 Quick start

1. Put your dataset at:
   `/content/data.xlsx` (or change the path in the script/notebook).
   The dataset must contain a `time` column (parseable as datetime) and these numeric columns:

    `Cyclone_Inlet_Gas_Temp`
    `Cyclone_Gas_Outlet_Temp`
    `Cyclone_Outlet_Gas_draft`
    `Cyclone_cone_draft`
    `Cyclone_Inlet_Draft`
    `Cyclone_Material_Temp`

2. Install dependencies (recommended virtualenv/conda):

```bash
pip install pandas numpy matplotlib seaborn scikit-learn statsmodels openpyxl
# Optional / recommended:
pip install hdbscan   # if you want HDBSCAN
```

3. Run the provided Python script or the notebook that contains your pasted code. The script assumes a Jupyter/Colab environment but can be run as a `.py` file after minor changes.



 Repository / artifact layout (what the code produces)

 `plots/` — PNG plots (correlation heatmap, time-slices, shutdown visualization, cluster week view, anomaly plots, forecast comparison)

   `correlation_matrix.png`
   `timeseries_week.png`, `timeseries_year.png`
   `shutdowns_year.png`
   `clustered_week.png`
   `anomaly_event_.png`
   `forecast_comparison.png`
 `outputs/` — tabular results (CSV)

   `summary_stats.csv` — descriptive stats
   `data_quality_report.csv` — missingness, min/max/mean
   `shutdown_periods.csv` — detected shutdown events (start, end, duration_minutes)
   `clusters_summary.csv` — cluster-level summary stats
   `clusters_duration_summary.csv` — frequency & duration descriptors per cluster
   `anomalous_periods.csv` — per-timestamp anomalies
   `anomalous_events_summary.csv` — consolidated anomaly events (start/end/duration/clusters)
   `forecasts.csv` — forecast outputs (true + baseline + arima + randomforest)



 What the included code does (mapping to Tasks)

 Task 1 — Data preparation & EDA

 Loads `data.xlsx`, parses `time` as index and sorts.
 Forces columns to numeric (`pd.to_numeric(..., errors=coerce)`).
 Reindexes to a strict 5-minute grid (`pd.date_range(..., freq=5T)`) to ensure uniform frequency.
 Missing values:

   Linear interpolation for short gaps (`limit=12`, i.e. up to 1 hour).
   Back/forward fill for remaining gaps.
 Outliers: z-score truncation (`|z| > 5`) masked then interpolated.
 Saves descriptive stats and a correlation heatmap.
 Saves slices (1 week, 1 year) as plots.

 Task 2 — Shutdown / Idle detection

 Example rule used (can & should be tuned to domain):

```py
is_shutdown = (Cyclone_Inlet_Gas_Temp < 100) & (Cyclone_Inlet_Draft < 5)
```

 Segments shutdown episodes (start/end/duration) and computes total downtime and event count.
 Produces a 1-year plot with shutdown spans highlighted.

> Tuning note: thresholds (`100°C`, `5` draft) are placeholders — confirm with operators/domain experts and tune accordingly.

 Task 3 — Machine state segmentation (clustering)

 Excludes shutdown rows.
 Feature engineering: raw features + rolling mean & std (30 min window; `rolling_window=6`) + deltas.
 Normalizes features and runs `KMeans(n_clusters=4)` by default.
 Produces:

   `clusters_summary.csv` (mean inlet/outlet temps, etc.)
   `clusters_duration_summary.csv` (episode durations / descriptive stats).
 Produces a sample week plot colored by cluster.

> Tuning note: `k=4` is an initial choice. Use elbow/silhouette or try DBSCAN/HDBSCAN for density-based clusters.

 Task 4 — Contextual anomaly detection & root cause analysis

 For each cluster, trains an `IsolationForest(contamination=0.01)` if cluster size ≥ 50.
 Flags anomalies relative to cluster distribution (cluster-conditional).
 Consolidates timestamp anomalies into multi-minute events (`>15min` gap to split events).
 Saves events and plots (top 3 longest events plotted with ±1 hour context).
 The code invites picking 3–6 interesting events and hypothesizing root causes by examining timing and variable co-movement.

 Task 5 — Short-horizon forecasting (1 hour / 12 steps)

 Target: `Cyclone_Inlet_Gas_Temp`.
 Train/test split: last ~90 days used as test (the script attempts to pick `split_date = target.index[-(122490)]` — ensure dataset length > 90 days).
 Models compared:

   persistence baseline (last value carried forward)
   ARIMA(2,1,2) on the series
   RandomForestRegressor on lag features (12 lags = 1 hour)
 Evaluation: RMSE and MAE saved/printed and forecasts saved to `outputs/forecasts.csv`.
 Produces a forecast comparison plot.

> Caveat: ARIMA may fail if the series is non-stationary or has regime changes (shutdowns); RandomForest on lags is a simple nonparametric baseline. For better performance consider:
>
>  Prophet, SARIMAX with exogenous variables (drafts), or a small Seq2Seq/LSTM model trained on rolling windows.
>  Feature engineering with cluster labels as additional features.

 Task 6 — Insights & recommendations

 The code base saves the outputs you need to derive insights linking shutdowns, clusters, anomalies, and forecast performance. See the Example insights section below to copy into reports.



 Key hyperparameters & where to tune them

 Reindex freq: `freq="5T"` — keep if data is strictly 5-min.
 Interpolation gap limit: `limit=12` (1 hour) — adjust if you expect longer short gaps.
 Outlier cutoff: `z > 5` — more/less aggressive depending on sensor noise.
 Shutdown rule: currently `Inlet_Temp < 100` and `Inlet_Draft < 5`. Replace with domain thresholds.
 Rolling window for features: `rolling_window = 6` (30 minutes). Increase for smoother behavior or decrease to capture short transients.
 Clustering: `KMeans(n_clusters=4)` — try different `k`, elbow/silhouette, or HDBSCAN.
 IsolationForest contamination: `0.01` — lower for fewer anomalies, higher to flag more.
 Anomaly grouping gap: `15 minutes` — determines whether close timestamps aggregate into one event.



 Interpreting outputs (quick guide)

 `summary_stats.csv` & `data_quality_report.csv` — sanity check sensors and missingness.
 `correlation_matrix.png` — see which temperatures/drafts move together (helps root-cause explanation).
 `shutdown_periods.csv` — use to quantify downtime and correlate with anomalies and clusters.
 `clusters_summary.csv` — label clusters semantically (e.g., `Normal`, `High Load`, `Startup/Shutdown`, `Degraded`) by inspecting means and percentiles.
 `anomalous_events_summary.csv` — candidate incidents for root-cause analysis; inspect the associated `anomaly_event_.png`.
 `forecasts.csv` + `forecast_comparison.png` — evaluate forecast skill; RMSE / MAE printed to console.



 Example insights (paste-ready)

1. Cluster X shows higher anomaly rate and precedes shutdowns.
   Many anomalies in cluster X occur 10–30 minutes before shutdown events — inspect whether transient draft reductions drive cascading cooling.

2. Draft anomalies correlate with outlet temp drops.
   Sudden increases in `Cyclone_Inlet_Draft` often coincide with `Cyclone_Gas_Outlet_Temp` drops → possible upstream surge or partial blockage clearing.

3. Forecasting is challenged during regime switches.
   Persistence and ARIMA have acceptable performance in steady clusters but fail during startups/shutdowns; a regime-aware model (include cluster/state as input) improves local short-horizon forecasts.

4. Actionable monitoring rule (example):
   Trigger an operator alert when (cluster == `Normal`) and `Cyclone_Inlet_Gas_Temp` falls > 8°C within 10 minutes and `Cyclone_Inlet_Draft` spikes by > 20% — this combination historically precedes shutdowns.

5. Data collection recommendation:
   Add a high-resolution upstream flow or valve position signal (if available). It will improve root-cause confidence for draft/temperature anomalies.



 Next steps & improvements

 Replace static shutdown rule with a learned classifier (e.g., logistic regression trained on labeled shutdown windows).
 Use HDBSCAN / time-series segmentation methods (e.g., Bayesian online changepoint detection) for variable-length states.
 Build a small Seq2Seq or Transformer/LSTM model for robust 1-hour forecasts that include cluster/state and draft variables as inputs.
 Add domain labels (operator logs, maintenance records) to validate anomaly-to-shutdown relationships and reduce false positives.
 Add an online pipeline for real-time detection (streaming ingestion + rolling model scoring).



 Troubleshooting & notes

 If `split_date` indexing throws an error, your dataset is shorter than 90 days — adjust the indexing logic or choose a different test split.
 ARIMA can raise exceptions on non-stationary / insufficient data — catch and fallback (script already catches).
 Large datasets (≈370k rows) may take memory/time — run in Colab with a GPU is unnecessary, but ensure enough RAM.


 Contact & license

 Author: mojesh/mojeshthaddi246

