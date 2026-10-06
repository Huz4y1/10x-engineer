---
tags: [awareness, tooling]
status: not-started
---

# Adjacent Tools You'll Meet on the Job

> **What this is:** enough about five tools that none of them are a surprise in an interview or a new codebase.
> **Why you care:** you don't need to learn these now. You do need to be able to say what each one does and how it relates to what you've already built. Go deeper only if a specific job calls for it.

Each section: what it is, the mental model, the minimum syntax to recognise, and how it maps to something you already know.

---

## 1. Airflow — the orchestrator

### What it is

The most widely used open-source workflow scheduler. Does what Databricks Workflows does ([[Unity Catalog and orchestration]]), but for *any* system, not just Databricks.

### The mental model

**A DAG defined in Python code, not in a UI.** That's the core difference and the core appeal — your pipeline is version-controlled, reviewable, and testable like any other code.

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

with DAG(
    "retail_daily",
    schedule="0 6 * * *",
    start_date=datetime(2026, 1, 1),
    catchup=False,                       # ← don't backfill every day since start_date
) as dag:

    bronze = PythonOperator(task_id="bronze", python_callable=load_bronze)
    silver = PythonOperator(task_id="silver", python_callable=clean_silver)
    gold   = PythonOperator(task_id="gold",   python_callable=build_gold)

    bronze >> silver >> gold             # the >> is the dependency arrow
```

`>>` means "then". `[a, b] >> c` means both a and b must finish before c.

### Vocabulary

| Term | Meaning |
|---|---|
| **DAG** | The pipeline |
| **Task** | One step |
| **Operator** | A task *template* — `PythonOperator`, `BashOperator`, `DatabricksSubmitRunOperator` |
| **Sensor** | A task that waits for something (a file to appear, an API to respond) |
| **Hook** | A connection to an external system |
| **XCom** | Passing small values between tasks. Small — **not** for DataFrames. |
| **Backfill** | Running historical dates |
| **Catchup** | Whether it auto-backfills from `start_date`. **Set `catchup=False`** unless you mean it. |

### The trap everyone hits

`execution_date` (now `logical_date`) is the *start of the period being processed*, not when the task runs. A daily DAG for 1 January runs on 2 January. It's confusing, it's deliberate (it makes backfills correct), and it catches everyone once.

### How it maps

| Databricks Workflows | Airflow |
|---|---|
| Job | DAG |
| Task | Task |
| `depends_on` | `>>` |
| Job parameters | `params` / `{{ ds }}` templating |
| Cron schedule | Same cron |

**Everything you learned about DAGs, retries, idempotency and backfills transfers exactly.** Only the syntax is new.

> **When you'd use it:** a pipeline touching several systems — pull from an API, load to S3, trigger Databricks, refresh Power BI, email a report. Databricks Workflows is better *inside* Databricks; Airflow is better across many tools.
>
> Managed versions: Azure Data Factory (Microsoft's own, drag-and-drop), Astronomer, MWAA, Google Cloud Composer. Most companies use a managed one — self-hosting Airflow is a real job.

---

## 2. dbt — SQL-first transformation

### What it is

A tool for writing your transformation layer as **SQL `SELECT` statements**, with dependency management, testing and documentation on top.

### The mental model

**You write `SELECT` statements. dbt turns them into tables, in the right order.**

```sql
-- models/silver/stg_sales.sql
SELECT
    invoice_no,
    product_id,
    customer_id,
    quantity,
    unit_price,
    quantity * unit_price AS revenue
FROM {{ source('bronze', 'online_retail') }}
WHERE quantity > 0
  AND NOT invoice_no LIKE 'C%'
```

```sql
-- models/gold/daily_sales.sql
SELECT
    CAST(invoice_date AS DATE) AS date,
    product_category,
    SUM(revenue)  AS revenue,
    SUM(quantity) AS units_sold
FROM {{ ref('stg_sales') }}          -- ← the magic
GROUP BY 1, 2
```

**`ref()` is the whole product.** By referencing models instead of writing table names, dbt builds the dependency graph automatically. `dbt run` executes everything in the correct order, in parallel where possible. You never write "run silver before gold" — it's inferred.

### Testing, declaratively

```yaml
# models/gold/schema.yml
models:
  - name: daily_sales
    columns:
      - name: date
        tests: [not_null]
      - name: revenue
        tests:
          - not_null
          - dbt_utils.accepted_range: {min_value: 0}
      - name: product_category
        tests:
          - relationships:
              to: ref('dim_product')
              field: category
```

`dbt test` runs them all. Data quality assertions as configuration rather than code — a genuinely nice idea, and directly comparable to the quality gates in [[Unity Catalog and orchestration]].

### Commands

```bash
dbt run              # build all models
dbt run --select gold.daily_sales+     # this model and everything downstream
dbt test             # run data tests
dbt docs generate && dbt docs serve    # lineage graph + docs site
```

### How it maps

Your bronze → silver → gold PySpark notebooks do exactly what dbt does. The differences:

| | PySpark notebooks | dbt |
|---|---|---|
| Language | Python | SQL only |
| Dependencies | You order the tasks | Inferred from `ref()` |
| Testing | You write it | Declarative in YAML |
| Docs/lineage | Unity Catalog | Generated |
| Can do ML | Yes | No |

> **When you'd use it:** the transformation layer is pure SQL and the team is analysts rather than engineers. dbt's real achievement is bringing software practices — version control, testing, code review, modularity — to people who write SQL. It runs happily on Databricks (`dbt-databricks`), so it's a complement, not a competitor.

---

## 3. Terraform — infrastructure as code

### What it is

Define your cloud resources in files. Terraform makes reality match the files.

### The mental model

**Declarative, not imperative.** You describe the desired end state; Terraform works out the steps.

```hcl
# main.tf
resource "azurerm_resource_group" "retail" {
  name     = "retail-rg"
  location = "uksouth"
}

resource "azurerm_storage_account" "lake" {
  name                     = "retailstore01"
  resource_group_name      = azurerm_resource_group.retail.name
  location                 = azurerm_resource_group.retail.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  is_hns_enabled           = true                    # ADLS Gen2
}

resource "azurerm_storage_container" "layers" {
  for_each             = toset(["bronze", "silver", "gold"])
  name                 = each.value
  storage_account_name = azurerm_storage_account.lake.name
}
```

```bash
terraform init      # download providers
terraform plan      # ← show what WOULD change. Read this every time.
terraform apply     # make it so
terraform destroy   # tear it all down
```

### Why it beats the CLI scripts you wrote

| | `az` CLI script | Terraform |
|---|---|---|
| Re-running it | Errors — "already exists" | Does nothing. Already correct. |
| Knowing current state | You look | `terraform plan` tells you |
| Changing something | Write an update script | Edit the file, apply |
| Deleting everything | Remember what you made | `terraform destroy` |

That first row is the key word: **idempotent**. Terraform tracks what it created in a **state file** and only changes the difference.

> **`terraform plan` before every `apply`.** It prints exactly what will be created, changed, and *destroyed*. The `-/+` symbol means "destroy and recreate" — on a database, that's data loss. Read the plan.

> **State file:** by default it's local, which breaks the moment two people work on the same infra. Real setups store it in Azure Blob Storage with locking. Never commit it — it can contain secrets.

### How it maps

Your `az` commands from [[Dev environment - Git, Docker, CLI]], but version-controlled, reviewable in a PR, and repeatable. Same idea as moving from clicking in the portal to scripting the CLI — one more step up the same ladder.

> **When you'd use it:** any team with more than one environment. "How do we know dev matches prod?" — because they're built from the same files with different variables. Alternatives: Bicep/ARM (Azure-only, Microsoft's own), Pulumi (real programming languages).

---

## 4. Power BI — the Microsoft dashboard

### What it is

Microsoft's BI tool. In Azure-heavy organisations you will meet it, and often it's what the business actually uses rather than your Streamlit app.

### The mental model

**Drag-and-drop dashboards over a semantic model.** Analysts build reports; you provide clean, well-modelled tables.

### The vocabulary

| Term | Meaning |
|---|---|
| **Power Query (M)** | The data prep step — like a lightweight ETL |
| **DAX** | The formula language for measures. Excel-like, and genuinely hard. |
| **Measure** | A calculation evaluated at query time — `Total Revenue = SUM(fact[revenue])` |
| **Calculated column** | Computed once at refresh, stored. Prefer measures. |
| **Import mode** | Data copied into Power BI. Fast. |
| **DirectQuery** | Queries hit the source live. Fresh but slower. |
| **Semantic model** | The tables + relationships + measures — the reusable layer |

```dax
Total Revenue = SUM(fact_transactions[revenue])

Revenue LY =
CALCULATE([Total Revenue], SAMEPERIODLASTYEAR(dim_date[full_date]))

YoY Growth % =
DIVIDE([Total Revenue] - [Revenue LY], [Revenue LY])
```

### Why [[Data modeling]] matters here

**Power BI is built for star schemas.** Its whole engine assumes facts in the middle, dimensions around the outside, single-direction relationships.

Hand it a snowflaked mess or a giant flat table and it's slow and painful. Hand it the star schema you designed and it's fast and the DAX is simple.

And `dim_date` isn't optional — Power BI's entire time-intelligence library (`SAMEPERIODLASTYEAR`, `DATESYTD`, `TOTALMTD`) requires a proper marked date table. **The modelling work you did is what makes Power BI good or bad.**

### How it maps

| Streamlit | Power BI |
|---|---|
| You write Python | Analysts drag and drop |
| Reads your FastAPI | Connects to Azure SQL / Databricks directly |
| You control everything | Fixed visual vocabulary |
| Free | Per-user licence |

> **When you'd meet it:** a Microsoft shop where business users self-serve. Your job becomes providing the semantic layer — clean gold tables and a good star schema — rather than building the dashboards. That's usually the more valuable half.

---

## 5. Kafka / Event Hubs — streaming

### What it is

A **durable, ordered log of events** that many producers write to and many consumers read from independently.

### The mental model

Not a queue (where reading removes the message). **A log.**

```
Topic: "sales-events"
Partition 0:  [e1][e2][e3][e4][e5][e6] →
                    ▲           ▲
              consumer A   consumer B
              (offset 2)   (offset 4)
```

Events stay for a **retention period** (hours to forever). Each consumer tracks its own position (**offset**). Consumer B being ahead doesn't affect A. A new consumer can start from the beginning and replay all of history.

> **That replay ability is the point.** Deploy a new fraud model and replay six months of events through it. Fix a bug and reprocess. With a queue, consumed means gone.

### Vocabulary

| Term | Meaning |
|---|---|
| **Topic** | A named stream — "sales-events" |
| **Partition** | A topic split for parallelism. **Order is guaranteed within a partition, not across.** |
| **Producer** | Writes events |
| **Consumer group** | A set of consumers sharing the work of one topic |
| **Offset** | A consumer's position |
| **Retention** | How long events are kept |

> **Partition key determines ordering.** Key by `customer_id` and all of one customer's events land in one partition, in order. Key randomly and you get parallelism but no per-customer ordering. This is the main design decision.

### Azure Event Hubs

Microsoft's managed equivalent, **Kafka-protocol compatible** — Kafka clients connect to it unchanged. In an Azure shop you'll use Event Hubs while everyone still says "Kafka".

### Reading it with Spark

Structured Streaming means you already mostly know how:

```python
df = (spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "...")
      .option("subscribe", "sales-events")
      .load())

(df.selectExpr("CAST(value AS STRING)")
   .writeStream
   .format("delta")
   .outputMode("append")
   .option("checkpointLocation", "abfss://bronze@.../_checkpoints/sales")   # ← essential
   .trigger(processingTime="1 minute")
   .toTable("retail.bronze.sales_stream"))
```

> **The `checkpointLocation` is how the stream remembers where it got to.** Without it, a restart reprocesses everything. Lose it and you've lost your position. It's the single most important streaming config.

**The mental shift:** your batch code becomes streaming code with almost no changes — same DataFrame API, same Delta sink. Spark handles the incremental bookkeeping. That's why Structured Streaming is called "streaming as an unbounded table."

### When streaming is actually worth it

| Latency needed | Use |
|---|---|
| Daily / hourly | **Batch.** Simpler, cheaper, easier to debug. |
| Minutes | Micro-batch (Structured Streaming, or Auto Loader) |
| Sub-second | True streaming |

> **Batch until you have a stated business reason not to.** Streaming multiplies operational complexity: late-arriving data, watermarks, exactly-once semantics, checkpoint corruption, harder testing, harder backfills. "It'd be cool" is not a reason. Most "real-time" requirements are satisfied by a 5-minute batch.

---

## How much of this to learn now

**None of it, yet.** Finish the roadmap and the capstone first — depth in one full stack beats shallow familiarity with ten tools, in interviews and in the job.

Then, if you want one:

| If your target job... | Learn |
|---|---|
| Is Databricks-heavy | Nothing here — go deeper on Delta and Spark tuning |
| Mentions Airflow / ADF | **Airflow.** A day gets you productive; the DAG concepts transfer. |
| Is analytics-engineering shaped | **dbt.** Fastest to learn, most immediately employable. |
| Is platform / DevOps shaped | **Terraform.** |
| Is Microsoft-shop shaped | **Power BI**, at least the modelling side. |
| Says "real-time" | **Kafka + Structured Streaming.** The biggest jump; leave it last. |

> **In an interview, "I haven't used Airflow, but it's the same DAG model as Databricks Workflows — tasks, dependencies, retries, backfills — and I'd expect to be productive in a day or two" is a strong answer.** It shows you understand the concept rather than having memorised a tool. That's what this note is for.

---

## Practice checklist

- [ ] **Airflow** — DAGs in Python, operators, sensors, XCom, `catchup`, and the `execution_date` trap; same concepts as Databricks Workflows
- [ ] **dbt** — models as `SELECT`s, `ref()` building the DAG, declarative tests; conceptually your silver → gold layer
- [ ] **Terraform** — declarative infrastructure, `plan` before `apply`, state files, idempotency vs. your CLI scripts
- [ ] **Power BI** — semantic models, measures vs. calculated columns, DAX, and **why your star schema and `dim_date` decide whether it's good**
- [ ] **Kafka / Event Hubs** — log not queue, topics, partitions, offsets, replay, checkpointing, and **when batch is the right answer**

## Hands-on (optional — only if a job asks)

- [ ] Write one Airflow DAG with three dependent tasks, locally via Docker
- [ ] Build two dbt models with a `ref()` between them and one test
- [ ] Recreate your capstone resource group in Terraform, then `destroy` it
- [ ] Connect Power BI Desktop to your Azure SQL gold tables and build one measure
- [ ] Stream a file source into a Delta table with Structured Streaming and a checkpoint

## Resources

- [Airflow: core concepts](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/index.html)
- [dbt: getting started](https://docs.getdbt.com/guides)
- [Terraform: AzureRM provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [Power BI: star schema guidance](https://learn.microsoft.com/power-bi/guidance/star-schema)
- [Spark Structured Streaming guide](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)

## Next

[[Certification map]]
