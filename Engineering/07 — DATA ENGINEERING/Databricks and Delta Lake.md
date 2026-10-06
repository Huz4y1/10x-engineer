---
tags: [databricks, delta-lake, pyspark]
status: not-started
---

# Databricks & Delta Lake

> **What this is:** Databricks is the managed platform where Spark actually runs at work. Delta Lake is the file format that makes a folder of files behave like a proper database table.
> **Why you care:** Delta is the single most important idea in modern data engineering. It's what turns a data swamp into something you can trust.

---

## Part 1 — Databricks

### The idea in plain English

Spark is free and open source. Running it yourself means provisioning machines, installing Java, configuring networking, handling failures, and patching everything forever.

Databricks is "someone else does all that." You click a button, get a cluster, run a notebook, and click it off.

### What's in a workspace

| Thing | What it is |
|---|---|
| **Notebook** | Cells of code you run interactively. Python, SQL, Scala, R — mixable in one notebook. |
| **Cluster** | The actual machines running Spark. Costs money while alive. |
| **SQL Warehouse** | A cluster tuned specifically for SQL queries and BI tools |
| **Job / Workflow** | A scheduled, automated run. See [[Unity Catalog and orchestration]]. |
| **Unity Catalog** | The governance layer — what tables exist, who can see them |
| **DBFS** | A built-in filesystem. Convenient, but put real data in ADLS ([[ADLS Gen2]]). |

### Clusters — the part that costs money

Two kinds:

- **All-purpose** — interactive, for notebook work. More expensive per hour.
- **Job** — created for one scheduled run, destroyed after. **Roughly half the price.** Use these for anything automated.

**Settings that matter:**

| Setting | What to pick while learning |
|---|---|
| **Terminate after** | **15–30 minutes.** Non-negotiable. |
| Workers | 1–2. Or use **Single Node** mode for small data. |
| Node type | The smallest available (`Standard_DS3_v2` or similar) |
| Autoscaling | Off for learning — it hides what's happening and can scale up expensively |
| Photon | Off while learning (costs more DBUs; it's a faster C++ query engine) |

> **💸 The bill:** you pay **twice** — Azure for the VMs, Databricks for DBUs (Databricks Units) on top. A small cluster is roughly £0.50–2/hour. Left running over a weekend, that's £100+. **Set auto-terminate the moment you create the cluster, before you do anything else.**

### Notebook essentials

```python
# magic commands switch language per cell
%sql
SELECT * FROM silver.sales LIMIT 10
```

```python
%pip install onnxruntime      # installs for this notebook's session
```

```python
dbutils.fs.ls("abfss://bronze@retailstore01.dfs.core.windows.net/")   # list files
dbutils.secrets.get(scope="retail-scope", key="sql-password")         # read a secret
dbutils.widgets.text("run_date", "2011-01-01")                        # a parameter
run_date = dbutils.widgets.get("run_date")                            # read it back
dbutils.notebook.run("./silver", timeout_seconds=3600)                # call another notebook
```

`spark` and `dbutils` already exist in every notebook — don't create a SparkSession.

> **Widgets are how notebooks become jobs.** A notebook with a hardcoded date is a one-off. The same notebook reading `dbutils.widgets.get("run_date")` can be scheduled to run every day with a different value. Build them in from the start.

> **Notebooks are bad software engineering by default** — hidden state, out-of-order execution, no tests. Use them to *explore*. Move logic into `.py` files in the repo (Databricks Repos syncs with Git), import those into the notebook, and test them properly ([[Testing and CI-CD]]).

---

## Part 2 — Delta Lake

### The problem it solves

Plain Parquet in a lake has four serious holes:

1. **No transactions.** A job writing 500 files dies after 300. Now the folder holds half a dataset, and readers see it. There's no "undo".
2. **Readers see partial writes.** Someone reads while you're writing and gets an inconsistent mix.
3. **No updates or deletes.** Parquet files are immutable. Deleting one customer (GDPR request) means rewriting entire partitions by hand.
4. **No history.** Overwrite a table with bad data and yesterday's version is gone forever.

### The solution: a transaction log

Delta Lake is **Parquet files plus a `_delta_log/` folder**.

```
silver/sales/
├── _delta_log/
│   ├── 00000000000000000000.json     ← version 0: "added files A, B, C"
│   ├── 00000000000000000001.json     ← version 1: "removed B, added D"
│   └── 00000000000000000002.json     ← version 2: ...
├── part-0000-A.parquet
├── part-0001-B.parquet
└── part-0002-D.parquet
```

The log is the source of truth about **which files are currently part of the table**. Everything follows from that:

- **Atomic writes.** A write only "counts" once its log entry is committed. A job that dies mid-write leaves orphan Parquet files that no log entry references, so readers never see them. Nothing is half-applied.
- **Snapshot isolation.** A reader reads the log once, gets a file list, and reads those files. A concurrent writer adding a new version doesn't disturb them.
- **Updates and deletes.** Rewrite the affected files, then commit a log entry saying "old file out, new file in."
- **Time travel.** Old log entries still list the old files. Replay the log to any version.

> **The key insight:** Delta didn't invent a new storage format. It's still Parquet. It added a *log*, and the log is what buys you database behaviour on top of cheap object storage. This is why "lakehouse" is a real idea and not just marketing.

### Using it

```python
# write
df.write.format("delta").mode("overwrite").save("abfss://silver@store.dfs.core.windows.net/sales/")

# or as a catalog table (preferred — see [[Unity Catalog and orchestration]])
df.write.format("delta").mode("overwrite").saveAsTable("retail.silver.sales")

# read
df = spark.read.format("delta").load("abfss://silver@store.dfs.core.windows.net/sales/")
df = spark.table("retail.silver.sales")
```

```sql
-- SQL works identically
CREATE TABLE retail.silver.sales USING DELTA AS SELECT * FROM ...;
DELETE FROM retail.silver.sales WHERE customer_id = '17850';   -- impossible in plain Parquet
UPDATE retail.silver.sales SET country = 'UK' WHERE country = 'United Kingdom';
```

On Databricks, Delta is the default format — `df.write.saveAsTable(...)` gives you Delta without asking.

---

## 3. Time travel

Every write creates a version. Every version is still readable.

```sql
SELECT * FROM retail.silver.sales VERSION AS OF 3;
SELECT * FROM retail.silver.sales TIMESTAMP AS OF '2026-09-01 14:00:00';

DESCRIBE HISTORY retail.silver.sales;   -- who changed what, when, with which operation
```

```python
spark.read.format("delta").option("versionAsOf", 3).load(path)
```

**What it's actually for:**

- **Undoing a mistake.** Ran a bad transformation and overwrote a good table? `RESTORE TABLE retail.silver.sales TO VERSION AS OF 4`. Two seconds.
- **Reproducibility.** "This model was trained on version 12 of the sales table." Now you can retrain on exactly that data, months later. This is genuinely valuable for ML and hard to get any other way.
- **Debugging.** "The number changed on Tuesday" — diff the two versions and see what happened.
- **Auditing.** `DESCRIBE HISTORY` shows every operation and who ran it.

> **The limit:** time travel only reaches back as far as the files still exist. `VACUUM` deletes files no longer referenced by recent versions, with a **default 7-day retention**. Run `VACUUM` and you lose the ability to travel past that point. That's a deliberate trade-off between history and storage cost.

```sql
VACUUM retail.silver.sales RETAIN 168 HOURS;   -- delete files unreferenced for 7+ days
```

---

## 4. `MERGE` — the upsert

Daily, a batch of records arrives. Some are new customers. Some are updates to existing ones. Some are deletions.

Without `MERGE` you'd read everything, combine it in Spark, and rewrite the whole table. On a big table that's absurd.

`MERGE` does it in one statement, touching only affected files:

```sql
MERGE INTO retail.silver.customers AS target
USING staging_customers AS source
  ON target.customer_id = source.customer_id

WHEN MATCHED AND source.is_deleted = true THEN
  DELETE

WHEN MATCHED THEN
  UPDATE SET
    target.country     = source.country,
    target.updated_at  = source.updated_at

WHEN NOT MATCHED THEN
  INSERT (customer_id, country, updated_at)
  VALUES (source.customer_id, source.country, source.updated_at);
```

Read it as: **for each source row, find the matching target row. If found, update or delete it. If not found, insert it.**

Python equivalent:

```python
from delta.tables import DeltaTable

target = DeltaTable.forName(spark, "retail.silver.customers")
(target.alias("t")
   .merge(source.alias("s"), "t.customer_id = s.customer_id")
   .whenMatchedUpdateAll()
   .whenNotMatchedInsertAll()
   .execute())
```

**Why this matters beyond convenience:** `MERGE` makes your pipeline **idempotent** — running it twice with the same input gives the same result, instead of duplicating everything. That's the property that lets you safely retry a failed job. It's also the mechanism behind SCD Type 2 ([[Data modeling]]).

> **Gotcha:** if the source has two rows with the same key, `MERGE` throws. Deduplicate the source first — usually with the `ROW_NUMBER() ... WHERE rn = 1` pattern from [[SQL fundamentals]].

---

## 5. Bronze → silver → gold, in practice

The layout is from [[ADLS Gen2]]. Here's what actually happens in each layer for the capstone.

### Bronze — land it, don't touch it

```python
from pyspark.sql import functions as F

raw = (spark.read
       .option("header", "true")
       .schema(bronze_schema)                 # explicit, always
       .csv("abfss://raw@store.dfs.core.windows.net/online_retail_II.csv"))

(raw
 .withColumn("_ingested_at", F.current_timestamp())     # when did we load this
 .withColumn("_source_file", F.input_file_name())       # where did it come from
 .write.format("delta").mode("append")
 .saveAsTable("retail.bronze.online_retail"))
```

**Rules:** cast types, add ingestion metadata, change nothing else. **Append only, never overwrite.** This is your audit trail — the thing that lets you answer "why is this number wrong?" six months from now.

### Silver — clean it

```python
from pyspark.sql import functions as F

bronze = spark.table("retail.bronze.online_retail")

silver = (bronze
    .filter(~F.col("invoice_no").startswith("C"))         # cancellations out
    .filter(F.col("quantity") > 0)                        # returns/errors out
    .filter(F.col("unit_price") > 0)                      # freebies and glitches out
    .filter(F.col("customer_id").isNotNull())             # can't attribute anonymous sales
    .withColumn("country", F.trim(F.initcap("country")))  # standardise
    .withColumn("revenue", F.col("quantity") * F.col("unit_price"))
    .dropDuplicates(["invoice_no", "product_id", "customer_id"]))

silver.write.format("delta").mode("overwrite").saveAsTable("retail.silver.sales")
```

**Rules:** one row per real event. Types correct, nulls handled, duplicates gone, categories standardised. A downstream user should be able to trust silver without checking anything.

> **Log what you drop.** `bronze.count() - silver.count()` should be a number you can explain. If you silently discard 40% of rows, someone will eventually ask where the missing revenue went.

### Gold — answer questions

```python
from pyspark.sql import functions as F

silver = spark.table("retail.silver.sales")

daily_sales = (silver
    .join(F.broadcast(spark.table("retail.silver.products")), "product_id")
    .groupBy(F.to_date("invoice_date").alias("date"), "product_category")
    .agg(F.sum("revenue").alias("revenue"),
         F.sum("quantity").alias("units_sold")))

daily_sales.write.format("delta").mode("overwrite").saveAsTable("retail.gold.daily_sales")
```

**Rules:** aggregated, denormalised, named for the business question. Small enough to copy into Azure SQL for serving ([[Azure SQL Database]]).

---

## 6. Performance

### `OPTIMIZE` — fixing the small file problem

Streaming or frequent small writes leave thousands of tiny files. Every file has fixed read overhead, so this destroys performance.

```sql
OPTIMIZE retail.silver.sales;                              -- compact into ~1GB files
OPTIMIZE retail.silver.sales WHERE date >= '2011-01-01';   -- only recent partitions
```

### `ZORDER` — clustering related rows together

Partitioning works for low-cardinality columns (year, country). For high-cardinality columns you filter on often — `customer_id`, `product_id` — partitioning would make millions of directories.

`ZORDER` instead physically **sorts related values into the same files**, so Spark can skip whole files using the min/max statistics in the Delta log. This is **data skipping**, and it's how a "needle in a haystack" query stays fast.

```sql
OPTIMIZE retail.silver.sales ZORDER BY (customer_id, product_id);
```

> **Rules of thumb:** `PARTITION BY` for columns with tens-to-hundreds of values that you filter on. `ZORDER` for high-cardinality columns you filter on. Z-order on **at most 3–4 columns** — effectiveness dilutes fast beyond that, and it's the *first* column that benefits most.
>
> Also: don't partition a table under ~1TB at all. On small tables partitioning creates more overhead than it saves.

### Liquid Clustering (the modern option)

Newer Databricks runtimes offer `CLUSTER BY`, which replaces both partitioning and Z-ordering and adapts automatically as data changes:

```sql
CREATE TABLE retail.silver.sales CLUSTER BY (invoice_date, customer_id) AS SELECT ...;
```

If your runtime supports it, prefer it — no partition-column decisions to get wrong up front.

### Caching

```sql
CACHE SELECT * FROM retail.gold.daily_sales;
```

Same trade-off as [[PySpark core]]: only for data read repeatedly in one session.

### Reading the Spark UI

On Databricks: cluster → **Spark UI**. Same as local. Check in this order:
1. **SQL tab** — the query plan with real row counts. Start here.
2. **Stages** — count the `Exchange` (shuffle) steps.
3. **Tasks** — max duration vs. median. A big gap means skew.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| Surprise Databricks bill | Cluster left running | Set auto-terminate to 15–30 min. Check now. |
| `ConcurrentAppendException` | Two jobs writing the same Delta table at once | Partition writes so they don't overlap, or serialise the jobs |
| `MERGE` fails: "multiple source rows matched" | Duplicate keys in the source | Deduplicate the source first (`ROW_NUMBER` + `rn = 1`) |
| Time travel: "version not found" | `VACUUM` deleted the underlying files | Increase retention *before* vacuuming; you can't get them back |
| Table reads slowly, lots of small files | Frequent small writes | `OPTIMIZE`; consider fewer, larger writes |
| Filter on `customer_id` still scans everything | Not partitioned or Z-ordered on it | `OPTIMIZE ... ZORDER BY (customer_id)` |
| Schema mismatch on append | New column in the source | `.option("mergeSchema", "true")`, deliberately — then check why it changed |
| Overwrote a good table | — | `RESTORE TABLE x TO VERSION AS OF n`. This is why Delta exists. |
| Notebook works, job fails | Notebook had leftover state from earlier cells | "Clear state and run all" before trusting it |
| `%pip install` package missing in a job | Session-scoped installs don't persist | Declare libraries on the cluster or in the job config |
| Cluster takes 5+ min to start | Normal — VMs are being provisioned | Use a cluster pool, or just wait |

---

## Practice checklist

- [ ] Databricks workspace: clusters vs. SQL warehouses, notebooks, jobs
- [ ] All-purpose vs. job clusters, and why job clusters are cheaper
- [ ] Cluster sizing, autoscaling, and **auto-terminate** — enough to not waste money on your own subscription
- [ ] `dbutils` basics: `fs`, `secrets`, `widgets`
- [ ] Delta Lake: ACID transactions on top of Parquet — **what the transaction log actually is and why it buys you everything**
- [ ] Time travel (`VERSION AS OF`), `DESCRIBE HISTORY`, `RESTORE`, and how `VACUUM` limits it
- [ ] `MERGE` for upserts, and why idempotency matters for retries
- [ ] The bronze → silver → gold pattern: raw, cleaned, business-aggregated
- [ ] Performance: `OPTIMIZE`, `ZORDER`, data skipping, caching, and reading a Spark UI stage graph

## Hands-on

- [ ] Set up a Databricks workspace connected to your ADLS Gen2 account — **set auto-terminate first**
- [ ] Write a Delta table, then look inside `_delta_log/` and read one JSON file. See the mechanism for yourself.
- [ ] Build a bronze → silver → gold pipeline for the retail dataset, Delta at every stage
- [ ] Overwrite a table with rubbish, then `RESTORE` it
- [ ] Run a `MERGE` to upsert new records without a full rewrite — then run it **twice** and confirm no duplicates
- [ ] Create 1,000 tiny files on purpose, time a query, run `OPTIMIZE`, time it again
- [ ] Parameterise a notebook with `dbutils.widgets` ready for [[Unity Catalog and orchestration]]

## Certification checkpoint

Databricks Certified Data Engineer Associate maps directly to this stage. See [[Certification map]].

## Resources

- [Databricks documentation](https://docs.databricks.com/)
- [Delta Lake docs](https://docs.delta.io/latest/index.html)
- [Delta Lake: transaction log deep dive](https://www.databricks.com/blog/2019/08/21/diving-into-delta-lake-unpacking-the-transaction-log.html) — read this one properly

## Next

[[Unity Catalog and orchestration]]
