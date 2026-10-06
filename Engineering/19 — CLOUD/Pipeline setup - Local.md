---
tags: [cloud, pipeline, setup, local, docker, self-hosted]
status: not-started
---

# Pipeline setup - Local

**Your laptop is the server.** The complete pipeline, no cloud account, no bill.

Overview: [[Pipeline setup - overview]] · Section: [[19 — CLOUD]]

---

## Why start here

- **Free.** No account, no card, no surprise bill.
- **Fast.** Restart a service in 2 seconds instead of 4 minutes.
- **Offline.** Works on a train.
- **It's your CI environment.** The same compose file runs in GitHub Actions for real integration tests.
- **Every concept transfers.** MinIO/SeaweedFS *is* S3. Delta OSS *is* Databricks Delta. MLflow is MLflow.

> **The only things you can't learn locally:** true scale (skew, shuffle spill), cloud identity models, and managed-service behaviour. Everything else is identical.

## What you'll have at the end

| Service | URL | Role |
|---|---|---|
| SeaweedFS | http://localhost:9333 | Object storage (≈ S3) |
| PostgreSQL | localhost:5432 | Warehouse |
| Kafka | localhost:9092 | Streaming |
| Mosquitto | localhost:1883 | MQTT ingest |
| Jupyter | http://localhost:8888 | Notebooks |
| MLflow | http://localhost:5000 | Tracking + registry |
| FastAPI | http://localhost:8000/docs | Serving |
| Prometheus | http://localhost:9090 | Metrics |
| Grafana | http://localhost:3000 | Dashboards |

## Prerequisites

```bash
docker --version          # 24+
docker compose version    # v2
python --version          # 3.11+
```

**~12 GB RAM** if you run everything. See the trimming note at the end — you rarely need it all.

---

## Step 1 — Foundation

```bash
mkdir -p ~/pipeline/{infra,pipelines,api,models,notebooks,dags}
cd ~/pipeline
git init
```

`.gitignore`:
```
.env
data/
mlruns/
__pycache__/
*.onnx
*.pth
```

`.env`:
```bash
ENV=local
LAKE_ROOT=s3a://bronze
DATABASE_URL=postgresql+psycopg://pipeline:devonly@localhost:5432/pipeline
S3_ENDPOINT=http://localhost:8333
S3_ACCESS_KEY=pipelineadmin
S3_SECRET_KEY=pipelineadmin
KAFKA_BOOTSTRAP=localhost:9092
MLFLOW_TRACKING_URI=http://localhost:5000
```

> **Every later guide uses the same variable names.** That's what makes the same code run on all four stacks.

---

## Step 2 — Storage: object store + database

`infra/docker-compose.yml`:

```yaml
services:

  # ---------- OBJECT STORAGE (= S3 / Blob / GCS) ----------
  seaweedfs:
    image: chrislusf/seaweedfs:latest
    command: >
      server -dir=/data -s3 -s3.port=8333 -filer
      -master.volumeSizeLimitMB=1024 -volume.max=0
    ports:
      - "9333:9333"   # master UI
      - "8888:8888"   # filer
      - "8333:8333"   # S3 API  <- the one you use
    volumes: [seaweed_data:/data]
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:9333/cluster/status"]
      interval: 10s
      retries: 10

  seaweed-init:
    image: amazon/aws-cli:latest
    depends_on:
      seaweedfs: {condition: service_healthy}
    environment:
      AWS_ACCESS_KEY_ID: pipelineadmin
      AWS_SECRET_ACCESS_KEY: pipelineadmin
    entrypoint: >
      /bin/sh -c "
      for b in bronze silver gold mlflow; do
        aws --endpoint-url http://seaweedfs:8333 s3 mb s3://$$b || true;
      done; echo buckets ready"

  # ---------- WAREHOUSE (= RDS / Azure PG / Cloud SQL) ----------
  postgres:
    image: postgres:16
    ports: ["5432:5432"]
    environment:
      POSTGRES_USER: pipeline
      POSTGRES_PASSWORD: devonly
      POSTGRES_DB: pipeline
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U pipeline"]
      interval: 5s
      retries: 10

volumes:
  seaweed_data:
  pgdata:
```

```bash
docker compose -f infra/docker-compose.yml up -d
```

**Verify:**
```bash
aws --endpoint-url http://localhost:8333 s3 ls          # 4 buckets
psql "postgresql://pipeline:devonly@localhost:5432/pipeline" -c "SELECT 1;"
```

> Full detail: [[SeaweedFS]] · [[PostgreSQL reference]]

---

## Step 3 — Ingest and streaming

Add to the compose file:

```yaml
  # ---------- MQTT (= IoT Hub / IoT Core) ----------
  mosquitto:
    image: eclipse-mosquitto:2
    ports: ["1883:1883"]
    volumes: ["./mosquitto.conf:/mosquitto/config/mosquitto.conf:ro"]

  # ---------- STREAMING (= Event Hubs / Kinesis / Pub-Sub) ----------
  kafka:
    image: bitnami/kafka:3.7
    ports: ["9092:9092"]
    environment:
      KAFKA_CFG_NODE_ID: 0
      KAFKA_CFG_PROCESS_ROLES: controller,broker
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_CFG_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 0@kafka:9093
      KAFKA_CFG_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
    volumes: [kafka_data:/bitnami]
```

`infra/mosquitto.conf`:
```
listener 1883
allow_anonymous true      # LOCAL ONLY - never in production
```

Add `kafka_data:` to the `volumes:` block, then:

```bash
docker compose -f infra/docker-compose.yml up -d
```

**Verify:**
```bash
mosquitto_sub -h localhost -t 'sensors/#' -v &     # subscribe
mosquitto_pub -h localhost -t sensors/1/temp -m '{"temp":42.1}'
```

> [[MQTT]] · [[Kafka]]

---

## Step 4 — Process: Spark + Delta on SeaweedFS

The one configuration that makes the local lakehouse work:

```python
# pipelines/spark.py
from pyspark.sql import SparkSession
import os

def build_spark(app="pipeline"):
    return (SparkSession.builder.appName(app).master("local[*]")
        .config("spark.jars.packages",
                "io.delta:delta-spark_2.12:3.2.0,"
                "org.apache.hadoop:hadoop-aws:3.3.4,"
                "com.amazonaws:aws-java-sdk-bundle:1.12.262")
        .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
        .config("spark.sql.catalog.spark_catalog",
                "org.apache.spark.sql.delta.catalog.DeltaCatalog")
        # --- point Spark's S3 client at SeaweedFS ---
        .config("spark.hadoop.fs.s3a.endpoint", os.environ["S3_ENDPOINT"])
        .config("spark.hadoop.fs.s3a.access.key", os.environ["S3_ACCESS_KEY"])
        .config("spark.hadoop.fs.s3a.secret.key", os.environ["S3_SECRET_KEY"])
        .config("spark.hadoop.fs.s3a.path.style.access", "true")        # <- REQUIRED
        .config("spark.hadoop.fs.s3a.connection.ssl.enabled", "false")
        .config("spark.hadoop.fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
        .config("spark.sql.shuffle.partitions", "4")                    # 200 is absurd locally
        .getOrCreate())
```

> **`path.style.access=true` is mandatory.** Real S3 addresses buckets as `bucket.host/key`; SeaweedFS and MinIO use `host/bucket/key`. Without it you get DNS errors that look nothing like a config problem.

Bronze → silver → gold:

```python
# pipelines/etl.py
from pyspark.sql import functions as F
from pipelines.spark import build_spark

spark = build_spark()

# BRONZE - land it untouched
bronze = (spark.read.json("s3a://bronze/raw/")
          .withColumn("_ingested_at", F.current_timestamp()))
bronze.write.format("delta").mode("append").save("s3a://bronze/readings/")

# SILVER - clean, dedupe
silver = (spark.read.format("delta").load("s3a://bronze/readings/")
          .filter(F.col("temp").between(-40, 150))
          .filter(F.col("device_id").isNotNull())
          .dropDuplicates(["device_id", "ts"]))
silver.write.format("delta").mode("overwrite").save("s3a://silver/readings/")

# GOLD - aggregate
gold = (silver.groupBy("device_id", F.to_date("ts").alias("date"))
        .agg(F.avg("temp").alias("avg_temp"),
             F.max("temp").alias("max_temp"),
             F.count("*").alias("n")))
gold.write.format("delta").mode("overwrite").save("s3a://gold/daily/")

# serve it from Postgres
(gold.write.format("jdbc")
   .option("url", "jdbc:postgresql://localhost:5432/pipeline")
   .option("dbtable", "daily_readings")
   .option("user", "pipeline").option("password", "devonly")
   .option("batchsize", 10000)          # <- or this takes forever
   .mode("overwrite").save())
```

**Verify Delta really works:**
```python
spark.sql("DESCRIBE HISTORY delta.`s3a://silver/readings/`").show()
spark.sql("RESTORE TABLE delta.`s3a://silver/readings/` TO VERSION AS OF 0")
```

> [[PySpark core]] · [[Databricks and Delta Lake]]

---

## Step 5 — Train + registry

```yaml
  mlflow:
    image: ghcr.io/mlflow/mlflow:latest
    depends_on:
      postgres:  {condition: service_healthy}
      seaweedfs: {condition: service_healthy}
    ports: ["5000:5000"]
    environment:
      MLFLOW_S3_ENDPOINT_URL: http://seaweedfs:8333
      AWS_ACCESS_KEY_ID: pipelineadmin
      AWS_SECRET_ACCESS_KEY: pipelineadmin
    command: >
      /bin/sh -c "pip install psycopg2-binary boto3 &&
      mlflow server --host 0.0.0.0 --port 5000
      --backend-store-uri postgresql://pipeline:devonly@postgres/pipeline
      --artifacts-destination s3://mlflow"
```

```python
import mlflow
mlflow.set_tracking_uri("http://localhost:5000")
mlflow.set_experiment("sensor-model")

with mlflow.start_run():
    mlflow.log_params({"lr": 1e-3, "hidden": 64})
    mlflow.log_metric("baseline_mae", baseline_mae)   # ALWAYS log the baseline
    mlflow.log_metric("test_mae", test_mae)
    mlflow.pytorch.log_model(model, "model", registered_model_name="sensor-model")
```

> [[MLflow experiment tracking]] · [[Tensors, autograd and the training loop]]

---

## Step 6 — Serve

```yaml
  api:
    build: ..
    ports: ["8000:8000"]
    depends_on:
      postgres: {condition: service_healthy}
    env_file: ../.env
    environment:
      DATABASE_URL: postgresql+psycopg://pipeline:devonly@postgres:5432/pipeline
      MLFLOW_TRACKING_URI: http://mlflow:5000
```

> **Note the hostnames change inside compose** — `postgres` and `mlflow`, not `localhost`. `localhost` inside a container means *that container* ([[Docker deep dive]]).

`Dockerfile`:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install --no-cache-dir uv && uv sync --frozen --no-dev
COPY api/ ./api/
COPY models/ ./models/
RUN useradd --create-home appuser && chown -R appuser /app
USER appuser
ENV PYTHONUNBUFFERED=1
EXPOSE 8000
CMD ["uv","run","uvicorn","api.main:app","--host","0.0.0.0","--port","8000"]
```

> `--host 0.0.0.0` or nothing reaches it. `PYTHONUNBUFFERED=1` or you get no logs.

> [[FastAPI data and deployment]]

---

## Step 7 — Observe

```yaml
  prometheus:
    image: prom/prometheus:latest
    ports: ["9090:9090"]
    volumes: ["./prometheus.yml:/etc/prometheus/prometheus.yml:ro"]

  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes: [grafana_data:/var/lib/grafana]
```

`infra/prometheus.yml`:
```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: api
    static_configs:
      - targets: ["api:8000"]
```

> [[Observability for data and ML pipelines]] · [[Grafana]] · [[Prometheus-Mimir]]

---

## The five verification checks

```bash
# 1 - data arrives
mosquitto_pub -h localhost -t sensors/1/temp -m '{"device_id":"1","ts":"2026-01-01T00:00:00","temp":42.1}'
aws --endpoint-url http://localhost:8333 s3 ls s3://bronze/ --recursive

# 2 - pipeline runs
python -m pipelines.etl

# 3 - database serves
psql "$DATABASE_URL" -c "SELECT * FROM daily_readings LIMIT 5;"

# 4 - API predicts
curl -s localhost:8000/health
curl -s -X POST localhost:8000/predict -H 'Content-Type: application/json' -d '{"device_id":"1"}'

# 5 - monitoring sees it
open http://localhost:9090/targets     # api target = UP
open http://localhost:3000             # Grafana
```

---

## Running it in CI

The real payoff — genuine integration tests, no cloud credentials:

```yaml
# .github/workflows/ci.yml
      - name: Start the stack
        run: docker compose -f infra/docker-compose.yml up -d --wait
      - name: Integration tests
        run: uv run pytest tests/integration/
      - name: Tear down
        if: always()
        run: docker compose -f infra/docker-compose.yml down -v
```

> `--wait` blocks until healthchecks pass. Without it, tests start before Postgres is ready and fail randomly.

---

## Trimming it down

The full stack wants ~12 GB. You rarely need it all:

| Doing | Run only |
|---|---|
| Spark / Delta work | seaweedfs, postgres |
| ML experiments | seaweedfs, postgres, mlflow |
| API development | postgres, api |
| Streaming | kafka, mosquitto |
| Everything | ~12 GB |

## Teardown

```bash
docker compose -f infra/docker-compose.yml down       # stop, KEEP data
docker compose -f infra/docker-compose.yml down -v    # stop, DELETE data
docker system prune -a                                # reclaim disk
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Spark: `ClassNotFoundException: S3AFileSystem` | `hadoop-aws` version mismatch | Match it to Spark's Hadoop version |
| Spark can't reach SeaweedFS | Missing `path.style.access` | Set it `true` |
| Delta commands unrecognised | Extensions not configured | Set `spark.sql.extensions` **and** the catalog |
| Spark jobs crawl on tiny data | 200 shuffle partitions | `spark.sql.shuffle.partitions=4` |
| API can't reach Postgres | Used `localhost` inside a container | Use the service name `postgres` |
| API starts before Postgres is ready | `depends_on` without a healthcheck | `condition: service_healthy` |
| MLflow artifacts fail to save | `mlflow` bucket missing, or S3 env unset | Create it; set `MLFLOW_S3_ENDPOINT_URL` |
| No logs from the API container | Buffered stdout | `ENV PYTHONUNBUFFERED=1` |
| Connection refused from browser | Bound to `127.0.0.1` | `--host 0.0.0.0` |
| Everything OOMs | ~12 GB stack | Comment out unused services |
| Data gone after restart | `down -v` deletes volumes | Use `down` without `-v` |
| Port already in use | Something else on 5432/8000 | Remap: `"5433:5432"` |

More: [[I HAVE A PROBLEM]] · [[Running the whole stack locally]]

## Next

[[Pipeline setup - Azure]] — the same pipeline, managed.

## Related

[[Pipeline setup - overview]] · [[SeaweedFS]] · [[Kafka]] · [[MQTT]] · [[PySpark core]] · [[Docker deep dive]] · [[Cloud comparison dictionary]]
