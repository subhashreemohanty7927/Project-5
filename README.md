# TellCo user analytics

Analysis of one month of xDR data for a mobile operator (TellCo): user overview, engagement,
experience and satisfaction, with a Streamlit dashboard.

## Easiest way (Windows)
See `START_HERE.txt`: put `telcom_data.xlsx` in `data/raw/`, then double-click `1_SETUP.bat`, `2_RUN_ANALYSIS.bat`, `3_OPEN_DASHBOARD.bat`, `4_EXPORT_TO_MYSQL.bat`.
SQL queries are in `MySQL_commands.sql`.

## Quick start
```bash
pip install -e ".[dev]"
telco-pipeline --raw data/raw/telcom_data.xlsx --out artifacts    # ~3 min, writes tables/figures/mlruns
streamlit run dashboard/app.py                                     # http://localhost:8501
pytest --cov=telco_analytics                                       # unit tests
```

## Notebook (VS Code)
Open `TellCo_Analysis.ipynb` in VS Code (needs the Python and Jupyter extensions), pick the kernel where you ran
`pip install -e ".[dev]"`, and click **Run All**. It walks through Tasks 1-4 using the package, one step per section.

## Layout
| Path | Purpose |
|---|---|
| `src/telco_analytics/data.py`, `cleaning.py` | load the Excel extract, drop totals row, replace missing values / IQR outliers with the mean |
| `overview.py` | Task 1: handsets, per-user aggregates, EDA, deciles, correlation, PCA |
| `engagement.py` | Task 2: engagement metrics, k-means (k=3), elbow, top users / apps |
| `experience.py` | Task 3: per-user TCP / RTT / throughput / handset, k-means (k=3) |
| `satisfaction.py` | Task 4: engagement & experience scores, satisfaction, regression, k-means (k=2) |
| `feature_store.py` | file-based, versioned feature store (parquet + JSON registry) |
| `tracking.py` | MLflow tracking (params, metrics, loss curve, artifacts, model) |
| `db.py` | export scores to MySQL and run the `SELECT` preview |
| `pipeline.py` | runs everything end to end |
| `dashboard/app.py` | Streamlit dashboard reading `artifacts/` |

## Task 4.6 - MySQL export
```bash
export MYSQL_URL="mysql+pymysql://user:password@localhost:3306/telecom"   # database must exist
telco-pipeline --raw data/raw/telcom_data.xlsx --out artifacts --mysql-url "$MYSQL_URL"
```
Then screenshot the result of `SELECT * FROM user_satisfaction LIMIT 10;` in MySQL Workbench / CLI.
(`artifacts/tables/mysql_select_preview.csv` holds the same rows.)

## Task 4.7 - model tracking
Each pipeline run logs four MLflow runs (engagement k-means, experience k-means, satisfaction
regression, satisfaction k-means) with parameters, metrics, the boosting loss per iteration, CSV/PNG
artifacts and the model. Start time, end time and git commit are recorded automatically.
```bash
mlflow ui --backend-store-uri sqlite:///artifacts/mlruns/mlflow.db     # screenshot for the report
```
`artifacts/tables/mlflow_runs.csv` is a CSV export of all runs. Run the pipeline from inside a git
repository so the code version (commit hash) is captured.


## Method choices worth knowing
- **Cleaning:** missing values and values outside 1.5xIQR are replaced with the mean of the remaining
  values, as specified. `telco-pipeline --outlier-k 3` loosens the outlier rule. See `treatment_report.csv`.
- **Decile classes:** users are split into 10 duration deciles; the top five (6-10) are reported.
- **Engagement score:** distance to the least-engaged centroid in min-max space. **Experience score:**
  distance to the worst-experience centroid, also min-max. Satisfaction is their average.
- **Handsets:** "undefined" is treated as missing. Users with no known handset are given the modal
  handset (per the brief) but flagged and excluded from handset-level analysis.
- **Top/bottom/most-frequent values (Task 3.2)** use raw values, not imputed ones; otherwise the most
  frequent value would simply be the imputed mean.

## Limitations
- One month of data, 73% of users with a single session; no revenue, churn or growth data.
- Per-application volumes exceed Total DL in every session, so the application mix is unreliable.
- TCP retransmission is missing in ~57% of sessions and RTT in ~19%; mean replacement hides this.
- 15,291 rows share a Bearer Id with another row; the IDs lost precision in Excel, so they are not used.
- The satisfaction regression predicts a score computed from its own inputs (R2 ~ 0.997); it shows the
  formula is recoverable, not that satisfaction is predictable.
- The satisfaction score is driven mostly by experience (its spread is ~3x that of engagement).
