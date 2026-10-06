---
tags: [databricks, unity-catalog, orchestration]
status: not-started
---

# Unity Catalog & Orchestration

> **What this is:** two things that turn a notebook into a production pipeline — a catalogue so people can *find and be allowed to use* your tables, and a scheduler so the pipeline runs without you.
> **Why you care:** a notebook you run by hand isn't a data platform. This is the difference between a demo and a job.

---

## Part 1 — Unity Catalog

### The problem

Without a catalogue, a table is a path: `abfss://silver@retailstore01.dfs.core.windows.net/sales/`.

That means:
- Nobody can *discover* what exists. You have to already know the path.
- Permissions live on the storage account, in a completely different system, with different vocabulary.
- Nobody knows where a table came from or who owns it.
- Every notebook hardcodes long paths, so moving storage breaks everything.

**Unity Catalog** puts a single naming and permission layer over all of it.

### The three-level namespace

```
catalog . schema . table
```

```
retail.silver.sales
retail.gold.daily_sales
retail.gold.customer_rfm
```

| Level | What it's for | Typical use |
|---|---|---|
| **Metastore** | One per region, sits above catalogs. Set up once by an admin. | — |
| **Catalog** | Top-level grouping | Per environment (`retail_dev`, `retail_prod`) or per domain |
| **Schema** | A group of related tables (also called a database) | `bronze`, `silver`, `gold` |
| **Table** | The actual data | `sales`, `daily_sales` |

```sql
CREATE CATALOG IF NOT EXISTS retail;
CREATE SCHEMA IF NOT EXISTS retail.bronze;
CREATE SCHEMA IF NOT EXISTS retail.silver;
CREATE SCHEMA IF NOT EXISTS retail.gold;

USE CATALOG retail;
USE SCHEMA silver;
SELECT * FROM sales LIMIT 10;      -- short name now works
```

> **Naming convention:** environment goes in the **catalog** (`retail_dev.silver.sales`), medallion layer in the **schema**. That way promoting dev→prod is a catalog swap, and your code can take the catalog name as a parameter.

### Managed vs. external tables

| | Managed | External |
|---|---|---|
| Where the data lives | A location UC controls | A path you specify |
| `DROP TABLE` | **Deletes the data** | Deletes only the metadata; files remain |
| Use for | Everything you create | Data owned by another system, or that must stay put |

```sql
-- managed: UC decides where the files go
CREATE TABLE retail.silver.sales AS SELECT * FROM retail.bronze.online_retail WHERE ...;

-- external: you say where
CREATE TABLE retail.bronze.online_retail
USING DELTA
LOCATION 'abfss://bronze@retailstore01.dfs.core.windows.net/online_retail/';
```

> **`DROP TABLE` on a managed table deletes the data.** Not just the pointer — the files. There's a short undrop window (`UNDROP TABLE`), but don't rely on it. Know which kind you're dropping.

### Storage credentials and external locations

This is the setup that makes paths disappear from your code. Done once:

1. **Storage credential** — wraps an Azure managed identity that can reach your storage account.
2. **External location** — binds a path prefix to that credential.

```sql
CREATE EXTERNAL LOCATION retail_bronze
URL 'abfss://bronze@retailstore01.dfs.core.windows.net/'
WITH (STORAGE CREDENTIAL retail_access_connector);

GRANT READ FILES, WRITE FILES ON EXTERNAL LOCATION retail_bronze TO `data_engineers`;
```

Afterwards, nobody configures `fs.azure.*` properties in notebooks ever again. Access is granted in SQL, and it's the same permission model as tables. This is the option-1 approach mentioned in [[ADLS Gen2]], and it's why it's worth the setup.

### Permissions

Familiar SQL `GRANT`, but with a twist: **you need permission at every level of the path.**

```sql
GRANT USE CATALOG ON CATALOG retail        TO `analysts`;    -- ← required
GRANT USE SCHEMA  ON SCHEMA  retail.gold   TO `analysts`;    -- ← required
GRANT SELECT      ON TABLE retail.gold.daily_sales TO `analysts`;

SHOW GRANTS ON TABLE retail.gold.daily_sales;
```

> **The most common UC permission bug:** you grant `SELECT` on the table, the user still can't see it, and the error is unhelpful. They're missing `USE CATALOG` and/or `USE SCHEMA`. Think of it as needing a key to the building, a key to the floor, *and* a key to the room. Check all three.

### Column and row level security

Restrict *parts* of a table rather than all of it:

```sql
-- mask a column for anyone not in the finance group
CREATE FUNCTION retail.gold.mask_email(email STRING)
RETURN CASE WHEN is_account_group_member('finance') THEN email ELSE '***@***.com' END;

ALTER TABLE retail.gold.customers
  ALTER COLUMN email SET MASK retail.gold.mask_email;

-- row filter: UK analysts see only UK rows
CREATE FUNCTION retail.gold.uk_only(country STRING)
RETURN is_account_group_member('global_analysts') OR country = 'United Kingdom';

ALTER TABLE retail.gold.sales SET ROW FILTER retail.gold.uk_only ON (country);
```

Enforced by the engine, so it applies to every query path — notebooks, SQL warehouse, BI tools, JDBC. You can't sidestep it by using a different client.

### Lineage — the free superpower

UC automatically records which tables and columns fed which others. No configuration.

You get: "`gold.daily_sales` was built from `silver.sales` and `silver.products`", down to column level, plus which notebooks and dashboards consume it.

**Why it matters:** when someone asks "if I change this column, what breaks?", you have an actual answer. And when a number looks wrong, you can walk backwards to the source instead of guessing.

---

## Part 2 — Orchestration

### The idea in plain English

Right now your pipeline is "open the notebook, click Run All." A production pipeline needs:

- To run **on a schedule**, at 6am, without you
- To run steps **in the right order** — silver can't start before bronze finishes
- To **retry** if a step fails for a transient reason
- To **tell someone** when it fails for a real reason
- To run for **different dates** without editing code

That's orchestration. In Databricks it's **Workflows**.

### A workflow is a DAG

**DAG** = Directed Acyclic Graph. Grand name, simple idea: **a set of tasks, and arrows showing what must finish before what.** "Acyclic" just means no loops — nothing can depend on itself.

```
        ┌──────────┐
        │  bronze  │
        └────┬─────┘
             ▼
        ┌──────────┐
        │  silver  │
        └────┬─────┘
             ▼
        ┌──────────┐
        │   gold   │
        └────┬─────┘
             ├──────────────┐
             ▼              ▼
      ┌─────────────┐  ┌──────────────┐
      │ load_to_sql │  │ train_models │   ← these two run in parallel
      └─────────────┘  └──────────────┘
```

Tasks with no dependency on each other run **at the same time**. That's most of the speed benefit.

### Building one

In the UI: **Workflows → Create Job**. Per task you set: a name, a type (notebook / Python script / SQL / dbt), the cluster, the parameters, and which tasks it depends on.

As JSON (which is what you'd commit to Git):

```json
{
  "name": "retail-daily-pipeline",
  "schedule": {
    "quartz_cron_expression": "0 0 6 * * ?",
    "timezone_id": "Europe/London"
  },
  "job_clusters": [{
    "job_cluster_key": "shared",
    "new_cluster": {
      "spark_version": "15.4.x-scala2.12",
      "node_type_id": "Standard_DS3_v2",
      "num_workers": 2
    }
  }],
  "tasks": [
    {
      "task_key": "bronze",
      "job_cluster_key": "shared",
      "notebook_task": {
        "notebook_path": "/Repos/retail/notebooks/01_bronze",
        "base_parameters": {"run_date": "{{job.parameters.run_date}}"}
      },
      "max_retries": 2,
      "min_retry_interval_millis": 60000
    },
    {
      "task_key": "silver",
      "depends_on": [{"task_key": "bronze"}],
      "job_cluster_key": "shared",
      "notebook_task": {"notebook_path": "/Repos/retail/notebooks/02_silver"}
    },
    {
      "task_key": "gold",
      "depends_on": [{"task_key": "silver"}],
      "job_cluster_key": "shared",
      "notebook_task": {"notebook_path": "/Repos/retail/notebooks/03_gold"}
    }
  ],
  "parameters": [{"name": "run_date", "default": "2011-01-01"}],
  "email_notifications": {"on_failure": ["you@example.com"]},
  "max_concurrent_runs": 1
}
```

Points worth noticing:

- **`job_clusters`** — one cluster shared by all tasks, created at the start and destroyed at the end. Cheaper than an all-purpose cluster and much cheaper than a new cluster per task.
- **`max_retries`** — retries only help for *transient* failures (a network blip, a cluster hiccup). A bug fails identically three times and just wastes ten minutes. Set 1–2, not 10.
- **`email_notifications.on_failure`** — **the single most important line.** A pipeline that fails silently is worse than no pipeline: you'll build dashboards on stale data and not know.
- **`max_concurrent_runs: 1`** — stops a slow run overlapping with the next scheduled one and corrupting the table.

### Cron, briefly

`0 0 6 * * ?` = second, minute, hour, day-of-month, month, day-of-week.

| Expression | Meaning |
|---|---|
| `0 0 6 * * ?` | 6:00am daily |
| `0 30 2 * * ?` | 2:30am daily |
| `0 0 * * * ?` | Every hour on the hour |
| `0 0 6 ? * MON` | 6am Mondays |

Don't memorise it — use [crontab.guru](https://crontab.guru/) and paste. Do set the **timezone** explicitly, or a UK job silently shifts by an hour twice a year.

### Parameters — how one notebook serves every date

The whole point of widgets from [[Databricks and Delta Lake]]:

```python
from pyspark.sql import functions as F

dbutils.widgets.text("run_date", "2011-01-01")
run_date = dbutils.widgets.get("run_date")

df = spark.table("retail.bronze.online_retail").filter(F.to_date("invoice_date") == run_date)
```

Databricks supplies useful values automatically:

```
{{job.parameters.run_date}}   custom parameter
{{job.start_time.iso_date}}   today's date
{{job.run_id}}                unique run identifier — good for logging
{{task.name}}
```

> **Why parameterising matters more than it sounds:** it's what makes **backfills** possible. When you discover a bug and need to reprocess three months of history, a parameterised job is a loop over dates. A hardcoded one is three months of manual edits.

### Making tasks idempotent

**Idempotent** = running it twice produces the same result as running it once.

This matters because jobs get retried, backfilled, and accidentally re-run. If a rerun duplicates data, every retry corrupts your table, and you can never safely re-run anything.

**How to get it:**
- Use `MERGE` instead of blind `append` ([[Databricks and Delta Lake]])
- Or `.mode("overwrite")` with `replaceWhere` on the partition being processed:

```python
(df.write.format("delta")
   .mode("overwrite")
   .option("replaceWhere", f"date = '{run_date}'")   # replaces ONLY that day
   .saveAsTable("retail.silver.sales"))
```

That replaces one day's partition and leaves the rest alone. Re-runnable any number of times, safely.

### Data quality gates

Add a task that checks the output and **fails the job** if it's wrong. Better a loud failure than a quiet wrong number on a dashboard.

```python
from pyspark.sql import functions as F

row_count = spark.table("retail.silver.sales").count()
assert row_count > 1000, f"Silver has only {row_count} rows — upstream problem?"

null_customers = spark.table("retail.silver.sales").filter(F.col("customer_id").isNull()).count()
assert null_customers == 0, f"{null_customers} null customer_ids in silver"
```

An exception fails the task, which stops downstream tasks and triggers the alert. That's exactly what you want.

---

## Part 3 — How this compares to Airflow

You'll meet Airflow outside pure-Databricks shops. Same concept, different tool.

| | Databricks Workflows | Airflow |
|---|---|---|
| DAG defined in | UI or JSON | Python code |
| Runs on | Databricks | Anything — it's a general orchestrator |
| Setup | None; it's built in | You run and maintain a server (or pay for a managed one) |
| Good at | Databricks-native pipelines | Co-ordinating many *different* systems |
| Weak at | Non-Databricks tasks | Being simple |

```python
from datetime import datetime
from airflow import DAG
from airflow.providers.databricks.operators.databricks import DatabricksSubmitRunOperator

# Airflow, for shape recognition
with DAG("retail_daily", schedule="0 6 * * *", start_date=datetime(2026, 1, 1)) as dag:
    bronze = DatabricksSubmitRunOperator(task_id="bronze", ...)
    silver = DatabricksSubmitRunOperator(task_id="silver", ...)
    gold   = DatabricksSubmitRunOperator(task_id="gold", ...)

    bronze >> silver >> gold      # the `>>` is the dependency arrow
```

> **The concepts transfer completely.** DAG, task, dependency, schedule, retry, backfill, idempotency — identical in both. Learn them here; you'll pick up Airflow's syntax in an afternoon. See [[Adjacent tools you will meet]].

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| "Table not found" but it exists | Missing `USE CATALOG` / `USE SCHEMA` grant | Grant all three levels, not just `SELECT` |
| `DROP TABLE` deleted the files | It was a managed table | Try `UNDROP TABLE`; use external tables for data you don't own |
| Job succeeded but table is empty | Task ran against the wrong date parameter | Log the parameter at the start of every notebook |
| Rerunning a job duplicated rows | Not idempotent | `MERGE`, or `replaceWhere` overwrite |
| Job runs at the wrong time | Timezone not set, or DST | Set `timezone_id` explicitly |
| Retries burn ten minutes on a real bug | `max_retries` too high | Set 1–2; retries are for transient faults only |
| Two runs overlapped and corrupted a table | `max_concurrent_runs` > 1 | Set it to 1 |
| Notebook works interactively, fails as a job | Cluster libraries, or leftover notebook state | Declare libraries in the job; "Clear state and run all" to reproduce |
| Nobody noticed the pipeline broke for a week | No failure alert | Add `email_notifications.on_failure` today |
| Access to storage works in a notebook but not in a job | Job cluster runs as a different principal | Grant the job's service principal the same access |

---

## Practice checklist

- [ ] Unity Catalog structure: metastore → catalog → schema → table
- [ ] Managed vs. external tables, and what `DROP` does to each
- [ ] External locations — how paths disappear from your notebooks
- [ ] Table and column-level permissions, and **why `SELECT` alone isn't enough**
- [ ] Lineage — and what question it lets you answer
- [ ] What a DAG is, in one sentence
- [ ] Databricks Workflows: multi-task jobs, dependencies, retries, alerts on failure
- [ ] Job clusters vs. all-purpose clusters for scheduled work
- [ ] Parameterizing a notebook so the same job runs for different dates/inputs
- [ ] **Idempotency** — `MERGE` and `replaceWhere`, and why retries demand it
- [ ] Data quality gates that fail the job loudly
- [ ] Awareness-level: how this compares to Airflow (see [[Adjacent tools you will meet]])

## Hands-on

- [ ] Set up Unity Catalog and register the bronze/silver/gold tables from your pipeline
- [ ] Grant a table to a group, deliberately omit `USE SCHEMA`, see the error — then fix it
- [ ] Look at the lineage graph for `gold.daily_sales`
- [ ] Turn your pipeline notebooks into a scheduled, parameterized Workflow with a failure alert
- [ ] Run it twice with the same parameter and confirm no duplicate rows
- [ ] Add a data-quality task that fails on purpose, and confirm the alert email arrives
- [ ] Backfill three different dates by re-running the job with different parameters

## Resources

- [Unity Catalog overview](https://docs.databricks.com/data-governance/unity-catalog/index.html)
- [Databricks Workflows docs](https://docs.databricks.com/workflows/index.html)
- [crontab.guru](https://crontab.guru/) — build and read cron expressions

## Next

[[FastAPI fundamentals]]
