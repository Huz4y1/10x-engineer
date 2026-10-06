---
tags: [local, docker, self-hosted, seaweedfs, postgres, spark, mlflow]
status: not-started
---

# Running the whole stack locally

> **What this is:** the entire pipeline — lake, warehouse, Spark, catalog, orchestrator, MLflow, API, dashboard, monitoring — on your own machine, with no Azure account and no bill.
> **Why you care:** you can learn and build the whole thing for free, iterate in seconds instead of minutes, work offline, and run the same stack in CI. Every concept transfers to Azure unchanged.

---

## The idea in plain English

Azure isn't magic. Every service in it is a managed version of something open source, and you can run that something in a container on your laptop.

Once you see the mapping, "learning Azure" stops being about Azure and becomes about the underlying tools. The portal is a wrapper.

| What you need | Azure sells you | Runs locally as | Same API? |
|---|---|---|---|
| Object storage (the lake) | **ADLS Gen2** | **SeaweedFS** | S3 API — Spark treats them the same |
| SQL serving store | **Azure SQL** | **PostgreSQL** | SQL, minor dialect differences |
| Spark cluster | **Databricks** | **Spark in Docker** / local PySpark | Identical PySpark API |
| Table format | Delta on Databricks | **Delta Lake OSS** | Identical — same project |
| Catalog & governance | **Unity Catalog** | **Unity Catalog OSS** / Hive metastore | Close |
| Orchestration | **Databricks Workflows** | **Airflow** / Prefect / Dagster | Same DAG concepts |
| Experiment tracking | Managed MLflow | **MLflow server** | Identical — same project |
| Secrets | **Key Vault** | `.env` / **Infisical** / Vault | Different API |
| Container hosting | **Container Apps** | **Docker Compose** / k3s | Concepts map |
| Image registry | **ACR** | **local registry** / GHCR | Docker API |
| Monitoring | **Azure Monitor** | **Prometheus + Grafana + Loki** | See [[Observability]] |
| Streaming | **Event Hubs** | **Redpanda** / Kafka | Kafka API |
| Vector search | **Azure AI Search** | **pgvector** | Different API |
| CI/CD | **Azure DevOps** | **GitHub Actions** / Gitea | Similar YAML |

> **The two that matter most are drop-in identical.** SeaweedFS speaks the S3 API and Delta Lake OSS *is* the same code Databricks runs. So `spark.read.format("delta")` works exactly the same locally and in the cloud — only the path prefix changes (`s3a://` vs `abfss://`).

---

## The `docker-compose.yml`

One file, the whole platform. Put it in `infra/local/`.

```yaml
services:

  # ---------- THE LAKE (≈ ADLS Gen2) ----------
  seaweedfs:
    image: chrislusf/seaweedfs:latest
    command: >
      server -dir=/data -s3 -s3.port=8333 -filer
      -master.volumeSizeLimitMB=1024 -volume.max=0
    ports:
      - "9333:9333"      # master UI
      - "8888:8888"      # filer
      - "8333:8333"      # S3 API   <- the one you use
    volumes: [seaweed_data:/data]
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:9333/cluster/status"]
      interval: 10s
      retries: 10

  # creates the bronze/silver/gold buckets, then exits
  seaweed-init:
    image: amazon/aws-cli:latest
    depends_on:
      seaweedfs: {condition: service_healthy}
    environment:
      AWS_ACCESS_KEY_ID: any
      AWS_SECRET_ACCESS_KEY: any
    entrypoint: >
      /bin/sh -c "
      for b in bronze silver gold mlflow; do
        aws --endpoint-url http://seaweedfs:8333 s3 mb s3://$$b || true;
      done; echo 'buckets ready';
      "

  # ---------- THE WAREHOUSE (≈ Azure SQL) ----------
  postgres:
    image: postgres:16
    ports: ["5432:5432"]
    environment:
      POSTGRES_USER: retail
      POSTGRES_PASSWORD: devonly
      POSTGRES_DB: retail
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U retail"]
      interval: 5s
      retries: 10

  # ---------- SPARK (≈ Databricks) ----------
  spark:
    image: bitnami/spark:3.5
    environment:
      SPARK_MODE: master
    ports:
      - "7077:7077"
      - "8080:8080"      # Spark master UI

  spark-worker:
    image: bitnami/spark:3.5
    depends_on: [spark]
    environment:
      SPARK_MODE: worker
      SPARK_MASTER_URL: spark://spark:7077
      SPARK_WORKER_MEMORY: 4G
      SPARK_WORKER_CORES: 2

  # notebooks with PySpark + Delta already wired up
  jupyter:
    image: jupyter/pyspark-notebook:latest
    ports: ["8888:8888"]
    volumes: ["../../notebooks:/home/jovyan/work"]
    environment:
      JUPYTER_ENABLE_LAB: "yes"
    command: start-notebook.sh --NotebookApp.token=''

  # ---------- EXPERIMENT TRACKING (identical to Databricks MLflow) ----------
  mlflow:
    image: ghcr.io/mlflow/mlflow:latest
    depends_on:
      postgres: {condition: service_healthy}
      seaweedfs: {condition: service_healthy}
    ports: ["5000:5000"]
    environment:
      MLFLOW_S3_ENDPOINT_URL: http://seaweedfs:8333
      AWS_ACCESS_KEY_ID: any
      AWS_SECRET_ACCESS_KEY: any
    command: >
      /bin/sh -c "pip install psycopg2-binary boto3 &&
      mlflow server --host 0.0.0.0 --port 5000
      --backend-store-uri postgresql://retail:devonly@postgres/retail
      --artifacts-destination s3://mlflow"

  # ---------- ORCHESTRATION (≈ Databricks Workflows) ----------
  airflow:
    image: apache/airflow:2.10.3
    depends_on:
      postgres: {condition: service_healthy}
    ports: ["8081:8080"]
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql://retail:devonly@postgres/retail
      AIRFLOW__CORE__LOAD_EXAMPLES: "false"
    volumes: ["../../dags:/opt/airflow/dags"]
    command: >
      bash -c "airflow db migrate &&
      airflow users create --username admin --password admin
      --firstname a --lastname b --role Admin --email a@b.c || true;
      airflow standalone"

  # ---------- YOUR APPS ----------
  api:
    build: ../../
    ports: ["8000:8000"]
    depends_on:
      postgres: {condition: service_healthy}
    environment:
      DATABASE_URL: postgresql+asyncpg://retail:devonly@postgres/retail
      MLFLOW_TRACKING_URI: http://mlflow:5000

  dashboard:
    build: ../../dashboard
    ports: ["8501:8501"]
    depends_on: [api]
    environment:
      API_URL: http://api:8000

volumes:
  seaweed_data:
  pgdata:
```

```bash
docker compose -f infra/local/docker-compose.yml up -d
```

| Service | URL | Login |
|---|---|---|
| SeaweedFS master | http://localhost:9333 | - |
| SeaweedFS filer | http://localhost:8888 | - |
| Spark master UI | http://localhost:8080 | — |
| Jupyter | http://localhost:8888 | no token |
| MLflow | http://localhost:5000 | — |
| Airflow | http://localhost:8081 | admin / admin |
| API docs | http://localhost:8000/docs | — |
| Dashboard | http://localhost:8501 | — |

> **Memory warning:** all of this together wants ~10GB of RAM. Comment out what you're not using — you rarely need Airflow and Spark cluster mode at the same time. For most work, SeaweedFS + Postgres + MLflow + local PySpark is plenty and runs in ~2GB.

---

## Spark + Delta + SeaweedFS

The one bit of configuration worth understanding. This is the local equivalent of the `abfss://` setup in [[ADLS Gen2]]:

```python
from pyspark.sql import SparkSession

spark = (SparkSession.builder
    .appName("local-lakehouse")
    .master("local[*]")
    .config("spark.jars.packages",
            "io.delta:delta-spark_2.12:3.2.0,"
            "org.apache.hadoop:hadoop-aws:3.3.4,"
            "com.amazonaws:aws-java-sdk-bundle:1.12.262")
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
    # --- point Spark's S3 client at SeaweedFS ---
    .config("spark.hadoop.fs.s3a.endpoint", "http://localhost:8333")
    .config("spark.hadoop.fs.s3a.access.key", "engineadmin")
    .config("spark.hadoop.fs.s3a.secret.key", "engineadmin")
    .config("spark.hadoop.fs.s3a.path.style.access", "true")     # ← SeaweedFS needs this
    .config("spark.hadoop.fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
    .config("spark.sql.shuffle.partitions", "4")                 # ← 200 is absurd locally
    .getOrCreate())
```

Then **everything from [[PySpark core]] and [[Databricks and Delta Lake]] works unchanged**:

```python
df.write.format("delta").mode("overwrite").save("s3a://bronze/online_retail/")

spark.sql("DESCRIBE HISTORY delta.`s3a://bronze/online_retail/`").show()
spark.sql("RESTORE TABLE delta.`s3a://bronze/online_retail/` TO VERSION AS OF 2")
```

Time travel, `MERGE`, `OPTIMIZE`, `ZORDER` — all of it. It's the same Delta Lake.

> **`path.style.access=true` is the SeaweedFS-specific line.** Real S3 uses `bucket.host/key`; SeaweedFS uses `host/bucket/key`. Without it you get baffling DNS errors.

### The one line that changes for Azure

```python
LAKE = "s3a://bronze"                                          # local
LAKE = "abfss://bronze@retailstore01.dfs.core.windows.net"      # Azure
```

> **Put the lake root in config, never hardcode a path.** Then the same pipeline code runs locally and in Databricks with one environment variable. This is the whole point of the exercise.

---

## Postgres instead of Azure SQL

95% identical. The differences that matter ([[Azure SQL Database]] has the full table):

| Task | Azure SQL (T-SQL) | Postgres |
|---|---|---|
| Limit | `SELECT TOP 10` | `LIMIT 10` |
| Concat | `'a' + 'b'` | `'a' \|\| 'b'` |
| Now | `GETDATE()` | `NOW()` |
| Auto key | `IDENTITY(1,1)` | `GENERATED ALWAYS AS IDENTITY` |
| Null fallback | `ISNULL()` | `COALESCE()` ← works in both |
| Driver | `pyodbc` / `aioodbc` | `psycopg` / `asyncpg` |

```python
# same SQLAlchemy code, different URL
DATABASE_URL = "postgresql+asyncpg://retail:devonly@localhost/retail"     # local
DATABASE_URL = "mssql+aioodbc://...database.windows.net/retaildb?..."     # Azure
```

> **Stick to `COALESCE`, `CAST`, and standard SQL** and the same queries run on both. Where you can't, keep the dialect-specific SQL in one module rather than scattered through the codebase.

> **Bonus: no ODBC driver headaches.** Postgres drivers are pure Python wheels — the whole apt block from [[Docker deep dive]] disappears.

---

## MLflow

Identical to the Databricks-hosted version. Only the URI changes:

```python
import mlflow
mlflow.set_tracking_uri("http://localhost:5000")     # local
# mlflow.set_tracking_uri("databricks")              # Databricks
mlflow.set_experiment("sales-forecasting")
```

Everything in [[MLflow experiment tracking]] applies unchanged — runs, params, metrics, artifacts, the model registry, aliases.

---

## Airflow instead of Databricks Workflows

Same DAG concepts, Python instead of JSON ([[Adjacent tools you will meet]]):

```python
# dags/retail_daily.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

with DAG("retail_daily", schedule="0 6 * * *",
         start_date=datetime(2026, 1, 1), catchup=False) as dag:

    bronze = PythonOperator(task_id="bronze", python_callable=run_bronze,
                            op_kwargs={"run_date": "{{ ds }}"})
    silver = PythonOperator(task_id="silver", python_callable=run_silver,
                            op_kwargs={"run_date": "{{ ds }}"})
    gold   = PythonOperator(task_id="gold",   python_callable=run_gold,
                            op_kwargs={"run_date": "{{ ds }}"})
    check  = PythonOperator(task_id="quality_check", python_callable=assert_quality)

    bronze >> silver >> gold >> check
```

> **`catchup=False`.** Without it, Airflow immediately backfills every scheduled interval since `start_date` — dozens of runs firing at once the moment you start it. Everyone learns this once.

> Lighter alternatives: **Prefect** and **Dagster** are much easier to run locally than Airflow, and Dagster's asset model maps very naturally onto bronze/silver/gold. If Airflow feels heavy, they're a fair swap.

---

## Monitoring

You already have these notes — [[Prometheus-Mimir]], [[Grafana]], [[Loki]], [[Tempo]], [[OpenTelemetry]]. Add them to the same compose file and you have Azure Monitor's function locally, with more control. Wiring: [[Observability for data and ML pipelines]].

---

## Secrets

```bash
# .env — gitignored
POSTGRES_PASSWORD=devonly
SEAWEED_S3_SECRET_KEY=devonly
```

Read them the same way you would in Azure, so the code doesn't change:

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    model_config = {"env_file": ".env"}
```

Locally the values come from `.env`; in Azure they come from Key Vault via a managed identity ([[Azure fundamentals]]). **The application code is identical** — it just reads environment variables. That's the design goal.

---

## Making one codebase run in both

The whole point. One config object, two environments:

```python
# api/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    env: str = "local"                 # local | azure
    lake_root: str                     # s3a://bronze  |  abfss://bronze@....
    database_url: str
    mlflow_tracking_uri: str
    model_config = {"env_file": ".env"}

settings = Settings()
```

```python
# pipelines/spark.py
from pyspark.sql import SparkSession

def build_spark():
    b = SparkSession.builder.config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    if settings.env == "local":
        b = (b.master("local[*]")
              .config("spark.hadoop.fs.s3a.endpoint", "http://localhost:8333")
              .config("spark.hadoop.fs.s3a.path.style.access", "true"))
    return b.getOrCreate()
```

Everything else — the transformations, the models, the API, the dashboard — is **completely unchanged**. Only the edges know where they're running.

> **This is also how you get real CI.** Spin the local stack up in GitHub Actions and run integration tests against actual SeaweedFS, actual Postgres, actual Delta tables — no cloud credentials, no cost, no flakiness from someone else's environment ([[CI-CD pipelines]]).

---

## What you *can't* do locally

Be honest about the gaps:

| Missing | Why it matters |
|---|---|
| **True scale** | 4 cores ≠ a 20-node cluster. Skew and shuffle problems don't show up. |
| **Unity Catalog governance** | UC OSS exists but is less complete than the Databricks version |
| **Azure RBAC / managed identity** | The whole identity model from [[Azure fundamentals]] has no local equivalent |
| **Autoscaling, serverless, spot** | Cost and elasticity behaviour |
| **Managed services** | Document Intelligence, AI Search, etc. |
| **Real network topology** | Private endpoints, VNets, firewall rules |

> **The honest recommendation: develop locally, validate on Azure.** Build and iterate on your laptop for speed and cost, then run the pipeline on a small Databricks cluster before you trust it. The things that only appear at scale — skew, shuffle spill, permission errors — appear there and nowhere else.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| Spark can't reach SeaweedFS | Missing `path.style.access` | Set it to `true` |
| `ClassNotFoundException: S3AFileSystem` | `hadoop-aws` jar missing/mismatched | Match `hadoop-aws` to your Spark's Hadoop version |
| Delta commands not recognised | Delta extensions not configured | Set both `spark.sql.extensions` and the catalog config |
| Spark jobs crawl locally | 200 shuffle partitions on tiny data | `spark.sql.shuffle.partitions=4` |
| MLflow artifacts fail to save | SeaweedFS bucket missing, or S3 env vars unset | Create the `mlflow` bucket; set `MLFLOW_S3_ENDPOINT_URL` |
| Airflow fires 400 runs on startup | `catchup` defaulted to True | `catchup=False` |
| API can't reach Postgres | Used `localhost` from inside a container | Use the service name `postgres` |
| Everything OOMs | Whole stack is ~10GB | Comment out unused services |
| Works locally, fails on Azure | Hardcoded `s3a://` paths, or SQL dialect | Config-driven paths; standard SQL |
| Data gone after `compose down -v` | `-v` deletes volumes | Use `down` without `-v` |
| Port already in use | Something else on 5432/8080 | Remap: `"5433:5432"` |

---

## Practice checklist

- [ ] The Azure → open-source mapping table
- [ ] SeaweedFS as ADLS, and why the S3 API makes it a drop-in
- [ ] Postgres as Azure SQL, and the dialect differences that matter
- [ ] **Delta Lake OSS is the same Delta** — time travel, MERGE, OPTIMIZE all work
- [ ] The Spark + Delta + SeaweedFS configuration, especially `path.style.access`
- [ ] MLflow self-hosted vs. managed — identical API
- [ ] Airflow vs. Databricks Workflows, and `catchup=False`
- [ ] Secrets via `.env` locally, Key Vault in Azure, **identical application code**
- [ ] **One codebase, two environments** — config at the edges only
- [ ] Using the local stack for real integration tests in CI
- [ ] What genuinely can't be reproduced locally

## Hands-on

- [ ] Bring up the compose stack and confirm every UI loads
- [ ] Rebuild the capstone bronze → silver → gold pipeline against SeaweedFS
- [ ] Run `DESCRIBE HISTORY` and a `RESTORE` on a local Delta table
- [ ] Point MLflow at the local server and log a run with artifacts
- [ ] Build the Airflow DAG version of the pipeline
- [ ] Refactor the capstone to be config-driven so it runs locally **and** on Azure with no code change
- [ ] Add the local stack to GitHub Actions and run one integration test against it

## Resources

- [SeaweedFS wiki](https://github.com/seaweedfs/seaweedfs/wiki)
- [Delta Lake: quickstart](https://docs.delta.io/latest/quick-start.html)
- [MLflow: tracking server setup](https://mlflow.org/docs/latest/tracking/server.html)
- [Airflow: running in Docker](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html)

## Next

[[Kubernetes and AKS]]
