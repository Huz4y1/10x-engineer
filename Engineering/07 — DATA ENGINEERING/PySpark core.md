---
tags: [pyspark, spark, distributed-computing]
status: not-started
---

# PySpark Core

> **What this is:** a tool for processing data that's too big for one computer, by splitting it across many.
> **Why you care:** this is the biggest conceptual jump in the whole roadmap. Everything you know about pandas is *almost* right here, and the places it's wrong are where the bugs live.

---

## The idea in plain English

You have to count the words in a library.

**Pandas:** one person reads every book. Works fine for a shelf. Impossible for a library.

**Spark:** you hand one bookshelf to each of a hundred people, they each count their own shelf, then you add the hundred numbers together.

That's it. That's distributed computing. The hundred people are **executors**, the bookshelves are **partitions**, and the person co-ordinating is the **driver**.

Two consequences follow from the analogy, and they explain nearly every Spark behaviour that seems weird:

1. **Nobody has the whole picture.** Each worker only sees its own shelf. Anything requiring the whole dataset at once (sorting, grouping, joining) means workers must **talk to each other** — and that's expensive.
2. **Handing out the work has overhead.** For a small job it's faster to just do it yourself. Spark on a 10MB file is *slower* than pandas. Spark wins at gigabytes and up.

---

## 1. The cast of characters

```
        ┌──────────────┐
        │    DRIVER    │  your Python code runs here.
        │              │  It plans the work and collects results.
        └──────┬───────┘
               │
   ┌───────────┼───────────┐
   ▼           ▼           ▼
┌────────┐ ┌────────┐ ┌────────┐
│EXECUTOR│ │EXECUTOR│ │EXECUTOR│   each holds some partitions
│ p1, p2 │ │ p3, p4 │ │ p5, p6 │   and does the actual work
└────────┘ └────────┘ └────────┘
```

- **Driver** — runs your script. Builds the plan. Has limited memory.
- **Executors** — the workers. Each holds a few **partitions** of data in memory.
- **Partition** — a chunk of rows. The unit of parallelism. One partition is processed by one core at a time.
- **Cluster** — driver + executors together.

> **The `.collect()` warning, up front:** `df.collect()` pulls *every row* from all executors into the driver's memory. On a 200GB DataFrame that crashes the driver instantly with `OutOfMemoryError`. It's the number one way beginners kill a Spark job. Use `.show(20)`, `.take(5)`, or `.limit(100).toPandas()` instead.

---

## 2. Lazy evaluation — the thing that confuses everyone

```python
df = spark.read.parquet("abfss://bronze@store.dfs.core.windows.net/sales/")
df2 = df.filter(df.quantity > 0)
df3 = df2.withColumn("revenue", df2.quantity * df2.unit_price)
df4 = df3.groupBy("product_id").sum("revenue")
```

**None of that has run anything.** Not one row has been read. Those four lines took two milliseconds.

Spark just wrote down your instructions. Then:

```python
df4.show()      # ← NOW it runs. All four steps, at once.
```

### Transformations vs. actions

| | What it does | Examples |
|---|---|---|
| **Transformation** | Adds a step to the plan. Returns a new DataFrame. **Runs nothing.** | `filter`, `select`, `withColumn`, `groupBy`, `join`, `orderBy`, `distinct` |
| **Action** | Demands a real answer. **Runs the whole plan.** | `show`, `count`, `collect`, `write`, `take`, `toPandas`, `first` |

### Why laziness is a feature

Because Spark sees the *whole* plan before running it, it can rewrite it to be faster. Two big optimisations happen automatically:

- **Predicate pushdown** — your `filter` gets moved as early as possible, sometimes all the way into the Parquet reader, so rows are skipped before they're ever read from disk.
- **Column pruning** — if you only `select` 3 of 50 columns, it never reads the other 47.

You wrote "read everything, then filter." Spark runs "read only what survives the filter." That's a 10× speedup you got for free.

### The three things laziness breaks

**1. Your timings lie.** The line that seems slow is just the first action; it's paying for everything above it.

**2. Errors surface late.** A typo in a column name on line 3 throws on line 40 where the action is. Read the stack trace for the *transformation* that's wrong, not the action that triggered it.

**3. Work gets repeated.** This is the expensive one:

```python
expensive = df.join(other, "id").filter(...).groupBy(...).agg(...)

expensive.count()          # runs the whole thing
expensive.show()           # runs the WHOLE THING AGAIN, from scratch
expensive.write.parquet()  # and AGAIN
```

Spark doesn't keep results unless told. Three actions = three full recomputations.

**The fix — `.cache()`:**

```python
expensive = df.join(other, "id").filter(...).groupBy(...).agg(...).cache()
expensive.count()          # computes and stores the result in memory
expensive.show()           # instant — reads the cache
expensive.unpersist()      # free the memory when done
```

> **When to cache:** a DataFrame used by **2 or more actions**, that was **expensive to build**, and **fits in memory**. Caching everything is worse than caching nothing — it evicts things you actually needed. Cache deliberately, and `unpersist()` when finished.

---

## 3. The DataFrame API

If you know pandas, you know 80% of this. The syntax differs; the ideas don't.

```python
from pyspark.sql import functions as F

df = spark.read.parquet("abfss://bronze@store.dfs.core.windows.net/sales/")

result = (
    df
    .filter(F.col("quantity") > 0)                                   # WHERE
    .withColumn("revenue", F.col("quantity") * F.col("unit_price"))  # new column
    .withColumn("year", F.year("invoice_date"))
    .groupBy("product_id", "year")                                   # GROUP BY
    .agg(
        F.sum("revenue").alias("total_revenue"),
        F.count("*").alias("n_sales"),
        F.countDistinct("customer_id").alias("n_customers"),
    )
    .filter(F.col("total_revenue") > 1000)                           # HAVING
    .orderBy(F.desc("total_revenue"))                                # ORDER BY
)

result.show(20, truncate=False)
```

> Wrap the chain in parentheses so you can put each step on its own line. Every Spark codebase does this — it makes long chains readable and lets you comment out a step to debug.

### Pandas → PySpark translation table

| pandas | PySpark |
|---|---|
| `df[df.qty > 0]` | `df.filter(F.col("qty") > 0)` |
| `df["a"] = df.b * 2` | `df = df.withColumn("a", F.col("b") * 2)` |
| `df.rename(columns={"a":"b"})` | `df.withColumnRenamed("a", "b")` |
| `df.groupby("k").sum()` | `df.groupBy("k").agg(F.sum("v"))` |
| `df.merge(o, on="id")` | `df.join(o, "id")` |
| `df.head()` | `df.show()` |
| `len(df)` | `df.count()` |
| `df.dropna()` | `df.na.drop()` |
| `df.fillna(0)` | `df.na.fill(0)` |
| `df.drop_duplicates()` | `df.dropDuplicates()` |
| `df.sort_values("x")` | `df.orderBy("x")` |
| `df.describe()` | `df.describe().show()` |
| `df.dtypes` | `df.printSchema()` |

### Three things that catch pandas users out

**1. DataFrames are immutable.** Every operation returns a *new* DataFrame. `df.withColumn(...)` without reassigning does nothing.

```python
df.withColumn("revenue", ...)       # ✗ discarded
df = df.withColumn("revenue", ...)  # ✓
```

**2. No row-by-row access.** There's no `df.iloc[5]`, no index, no meaningful row order unless you sort. Rows live on different machines.

**3. Use `F.col()`, not Python operators on raw strings.** `F.col("a") > 5` builds a *description* of a comparison, evaluated later on the cluster. It's not a Python comparison.

Also: `&`, `|`, `~` instead of `and`, `or`, `not` — **and the brackets are mandatory**, because `&` binds tighter than `>`:

```python
from pyspark.sql import functions as F

df.filter((F.col("qty") > 0) & (F.col("country") == "France"))   # ✓
df.filter(F.col("qty") > 0 & F.col("country") == "France")       # ✗ cryptic error
```

### You can just write SQL

```python
df.createOrReplaceTempView("sales")
spark.sql("""
    SELECT product_id, SUM(quantity * unit_price) AS revenue
    FROM sales
    WHERE quantity > 0
    GROUP BY product_id
    ORDER BY revenue DESC
""").show()
```

Identical performance — both go through the same optimiser. Use whichever is clearer for the task. Everything in [[SQL fundamentals]], window functions included, works here.

### RDDs — what to know

RDDs are the old, low-level API: no schema, no optimiser, you write the map/reduce yourself. **You will not need them.** Know that they exist, that DataFrames are built on them, and that seeing `.rdd` in modern code is usually a smell. That's enough.

---

## 4. Partitions and the shuffle

### Partitions

Your data is split into chunks. Each chunk is processed by one core.

```python
df.rdd.getNumPartitions()          # how many chunks?
df = df.repartition(200)           # reshuffle into 200 (expensive, but even)
df = df.coalesce(10)               # merge down to 10 (cheap, no shuffle, can be uneven)
```

**Rules of thumb:**
- Aim for partitions of **roughly 128MB**.
- Target **2–4× your total core count** so every core gets work.
- `repartition` when increasing or rebalancing. `coalesce` when decreasing (it avoids a shuffle).

### The shuffle — the thing that costs you

Some operations need rows that are *related* to end up on the *same machine*. To group by `product_id`, every row for product P100 must be together.

Moving rows between executors to achieve that is a **shuffle**. It means writing to disk, sending over the network, and reading back. It's the single most expensive thing Spark does — orders of magnitude slower than anything happening in memory.

**Causes a shuffle:** `groupBy`, `join`, `distinct`, `orderBy`, `repartition`, window functions.
**No shuffle:** `filter`, `select`, `withColumn`, `union`, `coalesce`.

> You can't avoid all shuffles. You *can* avoid pointless ones: filter **before** joining, not after. Fewer rows to shuffle = faster. Spark's optimiser often does this for you, but not always.

### Skew — when one partition is enormous

Imagine grouping sales by country, and 90% of your rows are `United Kingdom`. One partition gets 90% of the data. 199 cores finish in seconds; one core grinds for an hour. The whole job waits for it.

**How to spot it:** in the Spark UI, one task's duration is wildly longer than the median for its stage. That's skew, every time.

**Fixes:**
1. **Broadcast join** (below) — no shuffle, so no skew.
2. **Filter out the dominant key** and process it separately.
3. **Salting** — add a random suffix to the hot key to split it across partitions, then aggregate twice.
4. **Adaptive Query Execution** — on by default in Spark 3+, and it handles many skew cases automatically. Check it's enabled before doing anything clever:

```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
```

---

## 5. Joins — shuffle vs. broadcast

This is the highest-leverage performance knowledge in Spark.

### Shuffle (sort-merge) join — the default

Both tables get shuffled so matching keys land on the same executor, then joined. Correct for any size. Expensive.

### Broadcast join — the fast one

If one table is **small**, Spark can send a full copy to every executor. Then each executor joins its own partitions against its local copy. **No shuffle at all.**

This is the classic shape: a huge fact table joined to a small dimension table (500 million sales × 4,000 products).

```python
from pyspark.sql.functions import broadcast

result = sales.join(broadcast(products), "product_id")
```

Spark does this automatically when it *knows* the table is small:

```python
spark.conf.get("spark.sql.autoBroadcastJoinThreshold")   # default 10MB
```

But it often doesn't know — the size estimate is wrong for a table built by earlier transformations. So hint it explicitly with `broadcast()`.

> **The limit:** the small table must fit in **each executor's memory**, and in the driver's (it's collected there first). Under ~100MB is comfortable. Broadcast something too big and you crash the driver. If a broadcast fails with OOM, that's why.

### Reading the plan

```python
result.explain(mode="formatted")
```

Look for the join strategy in the output:

| You see | Meaning |
|---|---|
| `BroadcastHashJoin` | ✅ Fast path, no shuffle |
| `SortMergeJoin` | Standard shuffle join. Fine for two big tables. |
| `BroadcastNestedLoopJoin` | 🚨 **Danger.** Usually means your join condition isn't an equality. Can be catastrophically slow. |
| `Exchange` | A shuffle. Count them; each one costs. |

### Other join gotchas

```python
sales.join(products, "product_id")                            # ✓ one product_id column
sales.join(products, sales.product_id == products.product_id) # ✗ TWO ambiguous columns
```

The string form deduplicates the key column. The expression form keeps both, and any later `select("product_id")` throws "Reference is ambiguous". Use the string form when the column names match.

**Join types** are the same as SQL: `"inner"` (default), `"left"`, `"right"`, `"full"`, plus two Spark-specific ones:
- `"left_semi"` — rows from the left that *have* a match, no columns from the right. Like SQL `EXISTS`. Faster than a join + distinct.
- `"left_anti"` — rows from the left with *no* match. Great for "which customers never bought anything."

> And the row-count check from [[SQL fundamentals]] applies identically here. Count before, count after.

---

## 6. Reading and writing

```python
# read
df = spark.read.parquet("abfss://bronze@store.dfs.core.windows.net/sales/")
df = spark.read.format("delta").load("abfss://silver@store.dfs.core.windows.net/sales/")

df = (spark.read
      .option("header", "true")
      .option("inferSchema", "false")      # ← see below
      .schema(my_schema)
      .csv("abfss://raw@store.dfs.core.windows.net/online_retail.csv"))

# write
(df.write
   .mode("overwrite")              # overwrite | append | ignore | error
   .partitionBy("year", "month")
   .parquet("abfss://silver@store.dfs.core.windows.net/sales/"))
```

### Always declare the schema for CSV

`inferSchema=true` reads the entire file once just to guess types, then reads it again to load. Double the work. Worse, it guesses *wrong* — a customer ID that's all digits becomes an integer, leading zeros vanish, and a column that's mostly numbers with one stray "N/A" becomes a string.

```python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, DoubleType, TimestampType

schema = StructType([
    StructField("invoice_no",   StringType(),    True),   # STRING — they start with "C" for cancellations
    StructField("product_id",   StringType(),    True),
    StructField("description",  StringType(),    True),
    StructField("quantity",     IntegerType(),   True),
    StructField("invoice_date", TimestampType(), True),
    StructField("unit_price",   DoubleType(),    True),
    StructField("customer_id",  StringType(),    True),   # STRING — it's an identifier, not a number
    StructField("country",      StringType(),    True),
])

df = spark.read.option("header", "true").schema(schema).csv(path)
```

> **Identifiers are strings, always.** You never do arithmetic on a customer ID. Storing it as an integer loses leading zeros and invites nonsense like `AVG(customer_id)`.

### Partitioned writes

`partitionBy("year", "month")` creates the `year=2011/month=01/` directory layout from [[ADLS Gen2]], enabling partition pruning on read.

> **Don't partition by a high-cardinality column.** `partitionBy("customer_id")` on 5,000 customers creates 5,000 directories each holding a handful of rows — the small file problem at its worst. Partition by something with tens to low-hundreds of distinct values: year, month, country, category.

---

## 7. UDFs — and why to avoid them

A UDF is your own Python function applied to a column.

```python
from pyspark.sql import functions as F

@F.udf(returnType=StringType())
def categorise(desc):
    return "mug" if "MUG" in desc.upper() else "other"

df = df.withColumn("cat", categorise("description"))   # works, but slow
```

**Why it's slow:** Spark runs on the JVM. A Python UDF forces every row to be serialised out to a Python process, executed, and serialised back. That's 10–100× slower than a built-in, and the optimiser can't see inside it, so predicate pushdown stops working.

**Almost always there's a built-in:**

```python
from pyspark.sql import functions as F

df = df.withColumn(
    "cat",
    F.when(F.upper("description").contains("MUG"), "mug").otherwise("other")
)
```

`pyspark.sql.functions` has hundreds of these. Before writing a UDF, search it. Genuinely need Python? Use a **pandas UDF** (`@F.pandas_udf`), which processes whole batches with Arrow and is far faster than a row-at-a-time UDF.

---

## 8. Running locally

You don't need a cluster to learn Spark. `pip install pyspark`, and it runs on your laptop's cores.

```python
from pyspark.sql import SparkSession

spark = (SparkSession.builder
         .appName("local-practice")
         .master("local[*]")                                  # all cores
         .config("spark.driver.memory", "4g")
         .config("spark.sql.adaptive.enabled", "true")
         .getOrCreate())
```

The Spark UI is at **http://localhost:4040** while the session is alive. Learn to read it now, because on Databricks it's the only way to diagnose a slow job.

**What to look at, in order:**
1. **Jobs** — which action is slow?
2. **Stages** — a stage boundary is a shuffle. Lots of stages = lots of shuffles.
3. **Tasks within a stage** — compare min / median / max duration. Max ≫ median means **skew**.
4. **SQL tab** — the visual query plan, with actual row counts. The best screen in the UI.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| `OutOfMemoryError` on the driver | `.collect()` / `.toPandas()` on a big DataFrame | `.show()`, `.take(n)`, or `.limit(n).toPandas()` |
| Job runs three times slower than expected | Multiple actions recomputing the same chain | `.cache()` the shared DataFrame |
| One task takes 50× longer than the rest | Data skew | Broadcast join, salting, or enable AQE skew handling |
| Join is very slow | Sort-merge join where a broadcast would do | `broadcast(small_df)`; check `.explain()` |
| `BroadcastNestedLoopJoin` in the plan | Non-equality join condition | Rewrite as an equality join if at all possible |
| Broadcast crashes the driver | The "small" table isn't small | Remove the hint; let it sort-merge |
| "Reference 'x' is ambiguous" | Joined on an expression, so both key columns survived | Join with the string form: `df.join(o, "id")` |
| Error points at a line with no bug | Lazy evaluation — the error is in an earlier transformation | Read the plan; add `.show()` after each step to bisect |
| Thousands of tiny output files | Too many partitions on write | `.coalesce(n)` before writing |
| Filter runs but reads everything | Not partitioned on that column, or filter is inside a UDF | Partition on it; replace the UDF |
| Wrong types after reading CSV | `inferSchema` guessed | Declare an explicit schema |
| `AnalysisException: Path does not exist` | Typo in the `abfss://` path, or missing permissions | Check container-before-`@`; check the data role |
| Everything is slower than pandas | Dataset is small | That's correct. Use pandas below ~1GB. |

---

## Practice checklist

- [ ] Driver / executor / partition — who does what
- [ ] DataFrame API — the standard way to work with Spark today (skip RDDs beyond knowing what they are)
- [ ] Transformations vs. actions, and why nothing runs until an action is called
- [ ] Why lazy evaluation makes things fast (pushdown, pruning) and makes errors confusing
- [ ] `.cache()` — when it helps and when it hurts
- [ ] The DAG — how Spark plans and executes a chain of transformations
- [ ] Partitioning — how data is split across executors, and why skew kills performance
- [ ] Joins in a distributed context: shuffle joins vs. broadcast joins
- [ ] Why UDFs are slow, and what to use instead
- [ ] Reading and writing Parquet/Delta from ADLS Gen2 paths, with an explicit schema

## Hands-on

- [ ] Load a multi-GB dataset locally with `pyspark` (no cluster needed yet) and profile it
- [ ] Chain five transformations, time them, then time the first action — see laziness directly
- [ ] Run three actions on one expensive DataFrame; add `.cache()`; compare total time
- [ ] Force a shuffle join, look at the plan with `.explain()`, then force a broadcast join and compare
- [ ] Create skew on purpose (duplicate one key heavily) and find the straggler task in the Spark UI
- [ ] Write the same logic as a Python UDF and as built-in functions; time both

## Resources

- [PySpark: getting started (official docs)](https://spark.apache.org/docs/latest/api/python/getting_started/index.html)
- [Learning Spark, 2nd Edition (free from Databricks)](https://www.databricks.com/resources/ebook/learning-spark-2nd-edition)
- [PySpark SQL functions reference](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/functions.html) — check here before writing any UDF

## Next

[[Databricks and Delta Lake]]
