# Practical Lab 1: Streaming Data for Predictive Maintenance

**Course:** CSCN8010 — Foundations of Machine Learning Frameworks  
**Repository:** [PracticalLab1_PredictiveMaintenance](https://github.com/koushikbalne97/PracticalLab1_PredictiveMaintenance)

## Project overview

This lab extends the Data Stream Visualization Workshop by fitting linear-regression models to robot joint-current telemetry and monitoring the differences between observed and predicted current. Sustained positive deviations are recorded as Alert or Error events.

 The notebook demonstrates both a local CSV analysis and a Neon PostgreSQL workflow. A sample of the training telemetry is seeded into the database, read back using SQL, and used to train and demonstrate regression models for Axes #1–#8. Synthetic test data is generated using training-data statistics, then a test stream is written and queried.

## Dataset

The notebook uses `dataset/RMBR4-2_export_test.csv`, the CSCN8010 course dataset. It contains 39,672 readings with timestamp, trait, and joint-current fields. The models and event detector use Axes #1–#8. The course dataset is not publicly downloadable, so the repository uses the supplied file rather than an external data link.

## Regression and event rules

For each axis, the notebook fits a univariate linear regression with elapsed time in seconds as the feature and current as the target:

```text
predicted current = slope × elapsed seconds + intercept
residual = observed current − predicted current
```

Positive residuals indicate readings above the regression baseline. Thresholds are calibrated from the **training residuals for Axis #1**, then applied as shared absolute limits across all eight axes:

| Setting | Value | Basis |
| --- | ---: | --- |
| `MinC` (Alert) | 3.62 kWh | Axis #1 training-residual 95th percentile: 3.622423 kWh, rounded to two decimals |
| `MaxC` (Error) | 10.64 kWh | Axis #1 training-residual 99th percentile: 10.637570 kWh, rounded to two decimals |
| `T` (minimum duration) | 4 seconds | Requires a deviation to persist beyond an isolated reading; the training data’s median sample gap is 1.891 seconds and its 95th-percentile gap is 2.003 seconds |

An Alert is raised when a residual is at least `MinC` continuously for at least `T` seconds; an Error is raised at `MaxC` for at least `T` seconds. The final detector also splits events across gaps greater than five seconds. These are exploratory thresholds: because one absolute threshold pair is calibrated on Axis #1 and shared across axes with different residual distributions, the notebook compares and visualizes per-axis behavior rather than treating the values as validated universal limits.

## Notebook workflow and results

Run [PracticalLab1.ipynb](./PracticalLab1.ipynb) from the project root, from top to bottom:

1. Load and inspect the CSV, parse timestamps, and examine missing values and current distributions.
2. Fit time-to-current regression models for all eight axes; record slopes and intercepts, plot the fitted lines, and analyze residuals and percentiles.
3. Create repeatable synthetic data by adding small noise relative to each training axis’s standard deviation. Inject known Alert and Error intervals to test the detector.
4. Detect sustained events, split detections at long sampling gaps, save event tables, and plot event counts.
5. Optionally load/query the Neon training data, train the eight database-backed models, generate synthetic test data using those database training statistics, and send/query a small stream batch.

Notebook charts visualize the regression baselines, residuals, and detected-event counts. The notebook writes:

- `data/synthetic_streaming_data.csv` — local synthetic stream input.
- `results/detected_events.csv` — initial detected event output.
- `results/final_detected_events.csv` — final gap-aware events across Axes #1–#8.

The Neon PostgreSQL section demonstrates the required database integration workflow. A valid Neon connection is required to reproduce the database-backed portion of the project.

## Setup

Use Python 3.14 from the project root. In PowerShell:

```powershell
py -3.14 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
jupyter lab
```

In Command Prompt, activate the environment with:

```bat
.venv\Scripts\activate.bat
```

Open `PracticalLab1.ipynb` in Jupyter and run its cells in order. The local analysis does not need database credentials. To run the Neon section, configure the optional database connection first.

## Neon PostgreSQL setup 

Create a project-root `.env` file containing your Neon connection string:

```text
DATABASE_URL=postgresql://<user>:<password>@<host>/<database>?sslmode=require
```

Keep `.env` private and never commit database credentials. The notebook initializes the `cat_dc` schema and seeds the training readings only if the database has no seed data, avoiding duplicate training inserts on repeated runs. It then reads the seeded records back from Neon and trains the database-backed models. A network connection and valid database credentials are required for these cells.

