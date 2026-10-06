---
tags: [capstone, project, build-guide]
status: not-started
---

# Capstone Build Guide

> Follow in order. Each step names what you're producing and roughly what it takes to get there — the "how" for each individual tool is in its own note earlier in this vault.

Read [[Capstone overview and architecture]] first. Design the schema ([[Data modeling]]) before writing pipeline code.

---

## Part 0 — Setup

- [ ] **💸 Set a budget alert on your Azure subscription. Before anything else.** `Cost Management → Budgets`, set it to £20. See [[Azure fundamentals]].

- [ ] **Repo structure.** Create a project with this shape:
  ```
  retail-platform/
  ├── ingestion/           # scripts to pull raw data into ADLS
  ├── notebooks/           # Databricks notebooks: bronze, silver, gold
  ├── pipelines/           # ← pure transformation functions (imported by notebooks, TESTED)
  ├── api/                 # FastAPI app
  ├── dashboard/           # Streamlit app
  ├── models/              # training scripts, saved model artifacts
  ├── tests/
  ├── infra/               # Azure CLI / Terraform for provisioning
  ├── DECISIONS.md         # ← three lines per judgement call. Write as you go.
  ├── README.md
  └── .github/workflows/ci.yml
  ```

  > `pipelines/` isn't in the original sketch and it matters. Notebooks aren't testable; pure `DataFrame → DataFrame` functions are. The notebook reads, calls `clean_sales(df)`, and writes. The function lives in `pipelines/` and has tests. See [[Testing and CI-CD]].

- [ ] **Git from the first commit.** Not once it works. `.gitignore` must cover `.env`, `data/`, `*.pth`, `*.onnx`, `mlruns/`.

- [ ] **Provision.** Via Azure CLI ([[Dev environment - Git, Docker, CLI]]): a resource group, a storage account **with hierarchical namespace** (ADLS Gen2), an Azure SQL Database (**Serverless tier**), a Databricks workspace. All in one region.

  > HNS cannot be enabled after creation. Get it right the first time ([[ADLS Gen2]]).

- [ ] **Set Databricks cluster auto-terminate to 15 minutes** the moment you create a cluster.

---

## Part 1 — Ingest & transform

### 1.1 Land the raw data

- [ ] Download the Online Retail II dataset, upload it to `raw/` in ADLS via the Python SDK or CLI — **not the portal**.

```python
from azure.identity import DefaultAzureCredential
from azure.storage.filedatalake import DataLakeServiceClient

service = DataLakeServiceClient(
    account_url="https://retailstore01.dfs.core.windows.net",
    credential=DefaultAzureCredential(),
)
fs = service.get_file_system_client("raw")
with open("online_retail_II.csv", "rb") as f:
    fs.get_file_client("online_retail/raw.csv").upload_data(f, overwrite=True)
```

> If you get `403 AuthorizationPermissionMismatch`, you need the **data-plane** role `Storage Blob Data Contributor`. Being Owner is not enough ([[Azure fundamentals]]).

### 1.2 Bronze — land it, don't touch it

- [ ] In a Databricks notebook, read the raw CSV/Excel with PySpark, **cast types explicitly**, write out as a Delta table unchanged except for typing. This is your audit trail — never mutate it later.

```python
from pyspark.sql import functions as F
from pyspark.sql.types import *

schema = StructType([
    StructField("invoice_no",   StringType()),    # STRING — cancellations start with "C"
    StructField("product_id",   StringType()),
    StructField("description",  StringType()),
    StructField("quantity",     IntegerType()),
    StructField("invoice_date", TimestampType()),
    StructField("unit_price",   DoubleType()),
    StructField("customer_id",  StringType()),    # STRING — it's an ID, not a number
    StructField("country",      StringType()),
])

raw = (spark.read.option("header", "true").schema(schema)
       .csv("abfss://raw@retailstore01.dfs.core.windows.net/online_retail/"))

(raw
 .withColumn("_ingested_at", F.current_timestamp())
 .withColumn("_source_file", F.input_file_name())
 .write.format("delta").mode("append")
 .saveAsTable("retail.bronze.online_retail"))
```

> **`mode("append")`, never `overwrite`.** Bronze is immutable. Never declare `inferSchema` — it guesses wrong and doubles the read ([[PySpark core]]).

### 1.3 Silver — clean it

- [ ] Drop cancelled invoices (`invoice_no` starting with "C"), handle nulls in `customer_id`, dedupe, standardize country names. Write as a Delta table.

Put the logic in `pipelines/silver.py` as a pure function:

```python
# pipelines/silver.py
from pyspark.sql import functions as F

def clean_sales(df: DataFrame) -> DataFrame:
    return (df
        .filter(~F.col("invoice_no").startswith("C"))       # cancellations
        .filter(F.col("quantity") > 0)
        .filter(F.col("unit_price") > 0)
        .filter(~F.col("product_id").isin("POST", "MANUAL", "D", "C2", "BANK CHARGES"))
        .withColumn("country", F.trim(F.initcap("country")))
        .withColumn("revenue", F.round(F.col("quantity") * F.col("unit_price"), 2))
        .dropDuplicates(["invoice_no", "product_id", "customer_id"]))
```

- [ ] **Log what you dropped.** `bronze.count() - silver.count()` should be a number you can explain.

```python
before, after = bronze.count(), silver.count()
print(f"bronze {before:,} → silver {after:,} ({before-after:,} dropped, {(before-after)/before:.1%})")
```

- [ ] **Decide what to do with the ~20% missing `customer_id`, and write it in `DECISIONS.md`.** Recommended: keep them for revenue aggregates, exclude them from RFM. Document the reasoning.

### 1.4 Gold — answer questions

- [ ] Design the star schema first (see [[Data modeling]]): a `fact_transactions` table plus `dim_customer`, `dim_product`, `dim_date`.

- [ ] Build two aggregate gold tables:
  - `daily_sales(date, product_category, revenue, units_sold)`
  - `customer_rfm(customer_id, recency_days, frequency, monetary_value, as_of_date)`

```python
from pyspark.sql import functions as F

# daily_sales
daily_sales = (silver
    .join(F.broadcast(dim_product), "product_id")           # small table → broadcast
    .groupBy(F.to_date("invoice_date").alias("date"), "category")
    .agg(F.sum("revenue").alias("revenue"),
         F.sum("quantity").alias("units_sold")))

# customer_rfm — as of a snapshot date, per [[Data modeling]]
as_of = "2011-12-09"
customer_rfm = (silver
    .filter(F.col("customer_id").isNotNull())
    .groupBy("customer_id")
    .agg(
        F.datediff(F.lit(as_of), F.max("invoice_date")).alias("recency_days"),
        F.countDistinct("invoice_no").alias("frequency"),
        F.sum("revenue").alias("monetary_value"),
    )
    .withColumn("as_of_date", F.lit(as_of).cast("date")))
```

> **`product_category` doesn't exist in the raw data** — you derive it from `description` with keyword rules, or by clustering descriptions. Keep it simple: 8–12 keyword-based categories with an "Other" bucket. Note the approach in `DECISIONS.md`.

### 1.5 Land in Azure SQL

- [ ] Write both gold tables to Azure SQL Database — this is what FastAPI will actually query, not Delta directly.

```python
(daily_sales.write.format("jdbc")
    .option("url", "jdbc:sqlserver://retail-sql-01.database.windows.net:1433;database=retaildb")
    .option("dbtable", "daily_sales")
    .option("user", dbutils.secrets.get("retail-scope", "sql-user"))
    .option("password", dbutils.secrets.get("retail-scope", "sql-password"))
    .option("batchsize", 10000)                          # ← or this takes 20 minutes
    .mode("overwrite").save())
```

- [ ] **Create the indexes after the load** — `mode("overwrite")` drops and recreates the table, taking your indexes with it ([[Azure SQL Database]]).

```sql
CREATE INDEX ix_daily_sales_cat_date ON daily_sales (product_category, date);
CREATE INDEX ix_customer_rfm_id      ON customer_rfm (customer_id);
```

> Connection fails? Add the "allow Azure services" firewall rule (`0.0.0.0 → 0.0.0.0`).

### 1.6 Orchestrate

- [ ] Turn bronze → silver → gold into a Databricks Workflow, parameterized by date, so it can be re-run for a new day of data ([[Unity Catalog and orchestration]]).

- [ ] Add `email_notifications.on_failure`. A pipeline that fails silently is worse than no pipeline.

- [ ] **Make it idempotent.** Use `replaceWhere` or `MERGE`, not blind `append`:

```python
(df.write.format("delta").mode("overwrite")
   .option("replaceWhere", f"date = '{run_date}'")
   .saveAsTable("retail.silver.sales"))
```

- [ ] Add a data-quality task that **fails the job** on bad output:

```python
from pyspark.sql import functions as F

n = spark.table("retail.gold.daily_sales").count()
assert n > 100, f"Only {n} rows in daily_sales — upstream problem"
assert spark.table("retail.gold.daily_sales").filter(F.col("revenue") < 0).count() == 0
```

### ✅ Checkpoint 1

You can query `daily_sales` and `customer_rfm` directly in Azure SQL and the numbers make sense against a manual spot-check in the raw file.

**Also verify:**
- [ ] Running the workflow **twice** produces the same row counts — no duplicates
- [ ] The failure alert email actually arrives (break something on purpose)
- [ ] `bronze → silver` row loss is a number you can explain out loud

---

## Part 2 — Train the models

### 2.1 Sales forecasting

- [ ] Build features from `daily_sales` (lagged revenue, day-of-week, rolling averages).

```python
FEATURE_ORDER = [                     # ← save this. Serving depends on it.
    "revenue_lag_1", "revenue_lag_2", "revenue_lag_4",
    "rolling_mean_4", "rolling_std_4",
    "week_of_year", "is_december",
]
```

- [ ] **Split by time, never randomly.**

```python
train = weekly[weekly.week < "2011-08-01"]
val   = weekly[(weekly.week >= "2011-08-01") & (weekly.week < "2011-10-01")]
test  = weekly[weekly.week >= "2011-10-01"]
assert train.week.max() < val.week.min() < test.week.min()      # make it a real test
```

> A random split puts the future in the training set. Your validation score looks brilliant and the model is worthless. This is the single most common serious ML mistake ([[Tensors, autograd and the training loop]]).

- [ ] **Compute the naive baseline first.**

```python
baseline_mae = (test.revenue - test.revenue_lag_1).abs().mean()
print(f"Naive baseline MAE: £{baseline_mae:,.0f}")     # BEAT THIS or you have nothing
```

- [ ] Fit the scaler on **train only**, and save it.

```python
from sklearn.preprocessing import StandardScaler
import joblib

scaler = StandardScaler().fit(X_train)
joblib.dump(scaler, "models/forecast_scaler.pkl")
```

- [ ] Train a small feed-forward or LSTM PyTorch model to predict next-week revenue per category. Log every run — params, metrics, the model itself — to MLflow.

- [ ] **Overfit one batch to near-zero loss before the real training run.** If it can't, you have a bug — find it in 30 seconds instead of after four hours.

### 2.2 Customer segmentation

- [ ] From `customer_rfm`, train a model that assigns each customer a segment. Log to MLflow alongside the forecasting runs.

Simplest defensible route:

```python
# 1. define segments with business rules (RFM quintiles)
# 2. train a small PyTorch classifier to reproduce them
SEGMENT_FEATURES = ["recency_days", "frequency", "monetary_value"]
SEGMENTS = ["at_risk", "regular", "loyal", "champion"]
```

- [ ] **Log the confusion matrix and F1**, not just accuracy. If 80% of customers are "regular", accuracy is meaningless.

- [ ] **`monetary_value` has extreme outliers.** Log-transform it (`np.log1p`) before scaling, or one customer with £280,000 dominates everything.

### 2.3 Track and choose

- [ ] Log data provenance in every run — this is what makes it reproducible:

```python
import mlflow

mlflow.log_params({
    "delta_version": 12,               # ← the exact table version trained on
    "train_start": "2010-12-01", "train_end": "2011-07-31",
    "feature_order": ",".join(FEATURE_ORDER),
})
mlflow.log_metric("baseline_mae", baseline_mae)
mlflow.log_artifact("models/forecast_scaler.pkl")
```

- [ ] Run at least 3 hyperparameter combinations per model.

- [ ] Compare runs in the MLflow UI on your chosen metric (e.g. MAE for forecasting), register the best of each in the Model Registry.

> **Choose on validation, report on test.** Picking the best test score means you tuned on the test set and your reported number is a lie ([[MLflow experiment tracking]]).

### 2.4 Export

- [ ] Export both winning models to ONNX and confirm identical predictions to the original PyTorch models in a clean script.

```python
import torch

model.eval()                              # ← MANDATORY, or dropout is baked in
torch.onnx.export(
    model, dummy_input, "models/forecast_v1.onnx",
    input_names=["features"], output_names=["prediction"],
    dynamic_axes={"features": {0: "batch"}, "prediction": {0: "batch"}},
    opset_version=17,
)
```

- [ ] Verify with a **different batch size** than the dummy — that's what catches missing `dynamic_axes`.

- [ ] Write a metadata file so the serving contract is code, not folklore:

```json
{
  "version": "v1",
  "feature_order": ["revenue_lag_1", "..."],
  "input_dtype": "float32",
  "scaler_path": "forecast_scaler.pkl",
  "mlflow_run_id": "abc123",
  "trained_on_delta_version": 12
}
```

### ✅ Checkpoint 2

Two exported model files, both reproducible from an MLflow run ID, both giving identical predictions in and out of PyTorch.

**Also verify:**
- [ ] The forecast model **beats the naive baseline** — if not, say so honestly and investigate
- [ ] The train/val/test split has no time leakage (assert it)
- [ ] The scaler and feature order are saved alongside each model

---

## Part 3 — Serve & visualize

### 3.1 FastAPI app

- [ ] Build these endpoints at minimum:
  - `GET /sales/forecast?category=X&weeks=4` — reads recent data from Azure SQL, runs the forecasting model, returns predictions
  - `GET /customers/{id}/segment` — looks up RFM values from Azure SQL, returns the customer's segment
  - `GET /health` — trivial liveness check
  - Load both ONNX models once at startup via a `lifespan` event, **not per-request**.

- [ ] Build model inputs from the saved `feature_order`, never by hand:

```python
import numpy as np

features = np.array([[row[f] for f in metadata["feature_order"]]], dtype=np.float32)
features = scaler.transform(features).astype(np.float32)
```

- [ ] Run inference off the event loop with `run_in_threadpool` — ONNX is blocking CPU work.

- [ ] Return `model_version` in every prediction response.

- [ ] One SQLAlchemy engine at module level, with `pool_pre_ping=True` and `pool_recycle=1800` ([[Azure SQL Database]]).

- [ ] Use `:named` parameters. Never f-string user input into SQL.

### 3.2 Tests

- [ ] Cover all three endpoints with `pytest` + `TestClient`, mocking the Azure SQL calls.

Plus the ones that catch the expensive bugs:

- [ ] `test_onnx_matches_pytorch` — numerical equivalence, different batch size
- [ ] `test_feature_order_unchanged` — the metadata matches the constant
- [ ] `test_no_time_leakage` — train max date < val min date
- [ ] `test_clean_sales_drops_cancellations` — the pure Spark transformation, on 3 rows

### 3.3 Streamlit dashboard

- [ ] At minimum: a sales trend chart (actual vs. forecast), a customer segment breakdown (bar or pie), and a filter by date range or category.

- [ ] **Every number on the page comes from the FastAPI endpoints** — the dashboard never queries Azure SQL or the model directly.

- [ ] `@st.cache_data(ttl=300)` on the fetch function, or every slider nudge re-hits the API.

- [ ] Wrap API calls in try/except with `st.error()` + `st.stop()` — a raw traceback on the page is not a dashboard.

### 3.4 Containerize & deploy

- [ ] Dockerize both the FastAPI app and the Streamlit app, push to Azure Container Apps. Confirm the deployed dashboard is pulling live predictions from the deployed API.

Things that will bite you, in order of likelihood:

- [ ] **ODBC Driver 18 in the API image.** `pyodbc` fails at runtime without it ([[FastAPI data and deployment]]).
- [ ] **`--host 0.0.0.0`** (API) and **`--server.address=0.0.0.0 --server.headless=true`** (Streamlit).
- [ ] **Dependency files copied before code**, so a code change doesn't reinstall PyTorch.
- [ ] **The dashboard reaches the API by its internal name**, not `localhost`.
- [ ] **Version your image tags.** Not `:latest`.

```bash
az containerapp create --name retail-api --resource-group retail-rg \
  --environment retail-env --image $ACR.azurecr.io/retail-api:v1 \
  --target-port 8000 --ingress external --min-replicas 0 --max-replicas 3 \
  --secrets "db-url=<conn>" --env-vars "DATABASE_URL=secretref:db-url"
```

> Set the API's ingress to `internal` and only the dashboard to `external` — both are in the same environment, so they can still talk. Nicer architecture, one flag.

### 3.5 CI

- [ ] GitHub Actions workflow: run tests on every push, and (optional stretch) build and push the Docker images on merge to main.

- [ ] **Turn on branch protection** requiring the check to pass. CI you can merge past is decoration ([[Testing and CI-CD]]).

### ✅ Checkpoint 3

A working URL you can open in a browser, showing real forecasts and segments computed by models you trained, served by an API you wrote, over data that started as a raw CSV in a data lake.

**Also verify:**
- [ ] No credentials anywhere in the dashboard code
- [ ] A failing test blocks a merge
- [ ] Someone else could follow your README and rebuild it

---

## Part 4 — Make it presentable

The build is done. This part is what turns it into something that gets you interviews, and it takes an afternoon.

- [ ] **README with the architecture diagram at the top.** What it does, how to run it, what you'd do differently. The mermaid diagram from [[Capstone overview and architecture]] pastes straight in — GitHub renders it.

- [ ] **A screenshot of the dashboard in the README.** People don't click links; they do look at images.

- [ ] **Finish `DECISIONS.md`.** Every judgement call, three lines each. This is the document that makes an interview go well, because it's evidence you thought rather than followed a tutorial.

- [ ] **Write down what you'd do differently with more time.** Genuinely — "the segmentation model should be K-means, I used a neural net to exercise the serving path" is a *strong* answer, not a weak one. Self-awareness reads as seniority.

- [ ] **Practise explaining the architecture in two minutes.** Out loud. Every arrow, and why it isn't somewhere else. You will be asked.

- [ ] **Know your numbers.** Row counts, how much you dropped and why, your MAE vs. the baseline, your p95 latency. "About a million rows, I dropped 22% — 2% cancellations, 20% missing customer IDs which I kept for revenue but excluded from RFM" is the answer of someone who did the work.

---

## Stretch goals

- [ ] Add a second FastAPI endpoint that retrains and re-registers a model on demand

  > Do this as a **background task that triggers a Databricks job**, not inline training in the request. An HTTP request that trains a model for ten minutes will time out. Explaining *why* you did it that way is worth more than the feature.

- [ ] Rebuild the dashboard's core screen in Django instead of Streamlit, with basic auth, and compare the experience

- [ ] Add data quality checks (e.g. row count / null rate assertions) as a task in the Databricks Workflow, failing the pipeline if they don't pass

- [ ] Implement `dim_customer` as a genuine SCD Type 2 with `MERGE` ([[Data modeling]]) — the most interview-relevant stretch goal here

- [ ] Add basic monitoring: log every prediction with its model version and input features, then plot the input distribution over time to detect drift

- [ ] Replace the CLI provisioning scripts with Terraform ([[Adjacent tools you will meet]])

---

## When you're stuck

Every failure mode in this build is documented in the note for that tool. The lookup table in [[Capstone overview and architecture]] maps symptoms to notes. Each note's own "When something goes wrong" table has the specific fix.

> **And a general one:** when a pipeline "works" but the output is wrong, check the row counts at every stage. Ninety percent of data bugs are a join fanning out, a filter dropping more than you thought, or a re-run duplicating rows. Count before, count after, explain the difference.
