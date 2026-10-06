---
tags: [pyspark, spark, reference, cheatsheet, data-engineering]
status: not-started
---

# PySpark reference

**The complete API reference.** Concepts and why it works this way: [[PySpark core]].
This note is what you'd otherwise have twelve documentation tabs open for.

Section: [[07 — DATA ENGINEERING]]

---

## I want to…

**Find your task, click straight through to it.**

| I want to…                                    | Go to                                                                     |
| --------------------------------------------- | ------------------------------------------------------------------------- |
| Understand what a DataFrame even is           | [[#0. The basics — start here]]                                           |
| **Do a simple everyday thing**                | [[#Simple things you'll want on day one]]                                 |
| **Start Spark (simplest)**                    | [[#Creating a SparkSession — the simple version]]                         |
| Know which Spark settings I actually need     | [[#Adding settings — only when you need them]]                            |
| **Read a CSV (simplest)**                     | [[#CSV — the simple version]]                                             |
| Know which CSV options I actually need        | [[#CSV — every option, and when you need it]]                             |
| **Find rows that failed to load**             | [[#⚠️ Bad rows are silently turned into nulls]]                           |
| Load a CSV / Parquet / Delta / database table | [[#2. Reading data]]                                                      |
| See what columns and types I have             | [[#`df.printSchema()` — reading the output]]                              |
| Understand a type I don't recognise           | [[#Every type, explained]]                                                |
| **Look at my data**                           | [[#Looking at rows]]                                                      |
| **Print a wide table readably**               | [[#`vertical=True` — the one to remember for wide tables]]                |
| **Show only certain columns**                 | [[#Displaying only the columns you want]]                                 |
| **Find the unique values in a column**        | [[#Finding unique values]]                                                |
| Count nulls in every column                   | [[#Nulls — the most important check]]                                     |
| Find duplicate rows                           | [[#Duplicates]]                                                           |
| Add a new calculated column                   | [[#Computing new columns]]                                                |
| **Delete a column**                           | [[#Deleting columns]]                                                     |
| Tell `drop` from `na.drop`                    | [[#⚠️ `drop`, `na.drop` and `dropDuplicates` are three different things]] |
| **Keep only some rows**                       | [[#6. Filtering]]                                                         |
| Filter on a list of values                    | [[#Is the value one of a list?]]                                          |
| Filter out nulls                              | [[#Filtering on nulls]]                                                   |
| Match text or use a wildcard                  | [[#Text matching]]                                                        |
| Fill in missing values                        | [[#Nulls]]                                                                |
| Remove duplicate rows                         | [[#Deduplication]]                                                        |
| Change a column's type                        | [[#Casting]]                                                              |
| Clean up messy text                           | [[#Trimming and standardising text]]                                      |
| Split or join strings                         | [[#Cutting strings up]]                                                   |
| Use a regular expression                      | [[#Regular expressions]]                                                  |
| Turn a string into a date                     | [[#Converting between strings and dates]]                                 |
| Get the year / month / hour out               | [[#Pulling out parts of a date]]                                          |
| Add days or months to a date                  | [[#Date arithmetic]]                                                      |
| Group by month or hour                        | [[#Rounding a date down (truncating)]]                                    |
| Round a number                                | [[#Rounding]]                                                             |
| Write if/else logic                           | [[#CASE WHEN — if/else for columns]]                                      |
| Map many values to categories                 | [[#Mapping many values at once]]                                          |
| **Average of one column**                     | [[#One column at a time — the simple case]]                               |
| **Get that average as a Python number**       | [[#Getting it out as an actual Python number]]                            |
| Average per group                             | [[#The average of one column, per group]]                                 |
| Average of only some rows                     | [[#The average of only *some* rows]]                                      |
| Sum / count / median by category              | [[#Grouping and aggregating]]                                             |
| Running total or moving average               | [[#Aggregates over a window]]                                             |
| Compare a row to the previous one             | [[#Offset]]                                                               |
| Rank rows, or take the top N per group        | [[#The top-N-per-group pattern]]                                          |
| **Combine two tables**                        | [[#What a join is]]                                                       |
| Choose the right join type                    | [[#Choosing `how` — the join types]]                                      |
| Find rows with no match                       | [[#semi and anti — filtering with a join]]                                |
| **Check a join didn't duplicate rows**        | [[#⚠️ The mistake that silently corrupts your numbers]]                   |
| Stack two tables (rows, not columns)          | [[#Stacking tables on top of each other (not side by side)]]              |
| Pivot rows into columns                       | [[#Pivot]]                                                                |
| Sort                                          | [[#Sorting and ordering]]                                                 |
| Work with arrays or JSON                      | [[#12. Arrays, maps and JSON]]                                            |
| Write my own function                         | [[#17. UDFs]]                                                             |
| **Save my results**                           | [[#Writing a file or table]]                                              |
| Make a job safe to re-run                     | [[#Re-runnable writes (idempotency)]]                                     |
| Update or merge a Delta table                 | [[#Delta operations]]                                                     |
| Just write SQL instead                        | [[#Running SQL against a DataFrame]]                                      |
| **Make a slow job faster**                    | [[#20. Performance]]                                                      |
| Work out why one task is slow                 | [[#Handling skew]]                                                        |
| Read from Kafka                               | [[#21. Structured Streaming]]                                             |
| Train a model on Spark                        | [[#22. MLlib]]                                                            |

---

## Contents

0. [[#0. The basics — start here\|The basics — start here]] · 1. [[#1. Setup\|Setup]] · 2. [[#2. Reading data\|Reading data]]
3. [[#3. Schemas and types\|Schemas and types]] · 4. [[#4. Investigating data\|Investigating data]] · 5. [[#5. Selecting columns\|Selecting columns]]
6. [[#6. Filtering\|Filtering]] · 7. [[#7. Cleaning and preprocessing\|Cleaning and preprocessing]] · 8. [[#8. String functions\|String functions]]
9. [[#9. Date and time\|Date and time]] · 10. [[#10. Math and stats\|Math and stats]] · 11. [[#11. Conditional logic\|Conditional logic]]
12. [[#12. Arrays, maps and JSON\|Arrays, maps and JSON]] · 13. [[#13. Aggregation\|Aggregation]] · 14. [[#14. Window functions\|Window functions]]
15. [[#15. Joins\|Joins]] · 16. [[#16. Reshaping\|Reshaping]] · 17. [[#17. UDFs\|UDFs]]
18. [[#18. Writing data\|Writing data]] · 19. [[#19. Spark SQL\|Spark SQL]] · 20. [[#20. Performance\|Performance]]
21. [[#21. Structured Streaming\|Structured Streaming]] · 22. [[#22. MLlib\|MLlib]]

```python
from pyspark.sql import SparkSession, Window, DataFrame
# SparkSession = your entry point · Window = for window functions · DataFrame = the type, for hints
from pyspark.sql import functions as F   # EVERY built-in column function lives here. Always "as F".
from pyspark.sql import types as T       # every data type (StringType, IntegerType...). Always "as T".
```

> **Always `import functions as F`.** Never `from pyspark.sql.functions import *` — it shadows Python builtins like `sum`, `min`, `max`, `abs` and `round`, and the resulting errors are baffling.

---

## 0. The basics — start here

### What a DataFrame actually is

**A DataFrame is a table.** Rows and columns, like a spreadsheet or a SQL table.

The difference from a spreadsheet is that Spark **splits it into chunks (partitions) and works on them on different machines at the same time**. You write code as if it's one table; Spark runs it in parallel.

```
        your DataFrame (6 million rows)
   ┌──────────┬──────────┬──────────┬──────────┐
   │ chunk 1  │ chunk 2  │ chunk 3  │ chunk 4  │   <- partitions
   │ machine A│ machine B│ machine C│ machine D│   <- all working at once
   └──────────┴──────────┴──────────┴──────────┘
```

> **You never think about the chunks** — until something is slow, and then it's almost always because the chunks are uneven (*skew*, §20) or because Spark had to shuffle everything between machines.

### Nothing happens until you ask for an answer

This is the single most surprising thing about Spark.

```python
from pyspark.sql import functions as F

df = spark.read.parquet("sales/")     # nothing read yet
df = df.filter(F.col("qty") > 0)      # nothing filtered yet
df = df.select("id", "price")         # still nothing

df.show()                             # NOW Spark runs all three, in one optimised pass
```

**Transformations are lazy** — they just build a plan. **Actions run the plan.**

| Transformations (lazy — build the plan) | Actions (run it) |
|---|---|
| `select` `filter` `withColumn` `groupBy` | `show()` `count()` `collect()` |
| `join` `orderBy` `drop` `distinct` | `first()` `take(n)` `toPandas()` |
| `limit` `repartition` `agg` | `write.parquet()` `foreach()` |

> **Why this is good:** because Spark sees all three steps before running any of them, it can push your filter down into the file reader and never load the rows you were going to throw away.

> ⚠️ **Why it bites:** every action re-runs the whole chain from the start. Calling `df.count()` then `df.show()` reads your files **twice**. If you'll use a DataFrame more than once and it was expensive to build, `df.cache()` it (§20).

### DataFrames never change

```python
from pyspark.sql import functions as F

df.filter(F.col("qty") > 0)          # this does NOTHING to df - result is thrown away
df = df.filter(F.col("qty") > 0)     # you must REASSIGN
clean = df.filter(F.col("qty") > 0)  # or name the new one
```

> **Every operation returns a *new* DataFrame.** There is no in-place modification in Spark, ever. Forgetting to reassign is the most common beginner mistake, and it fails silently — no error, your filter just didn't happen.

### The twelve commands you'll actually use

```python
from pyspark.sql import functions as F

df = spark.read.parquet("path/")     # 1. load data
df.printSchema()                     # 2. what columns and types are there?  (free, no job)
df.show(5)                           # 3. what does it look like?
df.show(2, vertical=True, truncate=False)            # 4. same, one field per line - for wide tables, show all characters in the field
df.count()                           # 5. how many rows?
df.columns                           # 6. just the column names, as a Python list

df.select("name", "price")           # 7. pick columns
df.filter(F.col("price") > 10)       # 8. pick rows
df.withColumn("vat", F.col("price") * 0.2)     # 9. add a computed column
df.groupBy("country").agg(F.avg("price"))      # 10. summarise
df.orderBy(F.desc("price"))          # 11. sort
df.write.mode("overwrite").parquet("out/")     # 12. save
```

**That's most of Spark.** Everything else in this reference is a variation on those twelve.

### Simple things you'll want on day one

**Rename a column**
```python
df = df.withColumnRenamed("old_name", "new_name")            # one
df = df.withColumnsRenamed({"a": "x", "b": "y"})             # several
df = df.toDF(*[c.lower().replace(" ", "_") for c in df.columns])   # fix ALL names at once
```

**Count rows and columns**
```python
df.count()                       # rows
len(df.columns)                  # columns
print(f"{df.count():,} rows x {len(df.columns)} columns")
```

**Sort**
```python
from pyspark.sql import functions as F

df.orderBy("price")                          # smallest first
df.orderBy(F.desc("price"))                  # biggest first
df.orderBy("country", F.desc("price"))       # by country, then price within it
df.orderBy(F.desc("price")).limit(10)        # the top 10
```

**See the biggest / smallest rows**
```python
from pyspark.sql import functions as F

df.orderBy(F.desc("price")).show(5)                    # 5 most expensive
df.orderBy("price").show(5)                            # 5 cheapest
```

**Count how many of each value** (pandas' `value_counts`)
```python
from pyspark.sql import functions as F

df.groupBy("country").count().orderBy(F.desc("count")).show()
```

**Add a constant column**
```python
from pyspark.sql import functions as F

df = df.withColumn("source", F.lit("taxi_2026"))       # F.lit for a fixed value
df = df.withColumn("loaded_at", F.current_timestamp())
```

**Add a row number**
```python
from pyspark.sql import Window, functions as F
df = df.withColumn("row_num", F.row_number().over(Window.orderBy("date")))
```

**Do maths between columns**
```python
from pyspark.sql import functions as F

df = df.withColumn("total", F.col("qty") * F.col("price"))
df = df.withColumn("with_vat", F.round(F.col("total") * 1.2, 2))
df = df.withColumn("diff", F.col("actual") - F.col("expected"))
```

**Replace values**
```python
df = df.replace("UK", "United Kingdom", subset=["country"])       # one value
df = df.na.fill({"qty": 0, "country": "unknown"})                 # fill nulls
```

**Convert to pandas (small results only)**
```python
pdf = df.limit(1000).toPandas()          # ALWAYS limit first
sdf = spark.createDataFrame(pdf)         # and back again
```

**Make a small DataFrame to test with**
```python
df = spark.createDataFrame(
    [(1, "a", 10.0), (2, "b", 20.0)],    # the rows
    ["id", "name", "price"],             # the column names
)
```

**Save and load locally, no cloud**
```python
df.write.mode("overwrite").parquet("output/")      # a FOLDER of part files
spark.read.parquet("output/")
```

**Chain a filter and a count in one line**
```python
from pyspark.sql import functions as F

df.filter(F.col("price") > 100).count()            # how many expensive rows?
```

**Check for a column before using it**
```python
from pyspark.sql import functions as F

if "price" in df.columns:
    df = df.withColumn("with_vat", F.col("price") * 1.2)
```

**Empty or not**
```python
df.isEmpty()                     # True/False - cheaper than count() == 0
```

### Chaining them together

Because each step returns a new DataFrame, you chain them. Wrap in brackets so you can break lines:

```python
from pyspark.sql import functions as F

result = (df                                        # brackets let you split across lines
    .filter(F.col("quantity") > 0)                  # keep valid rows
    .withColumn("revenue", F.col("quantity") * F.col("price"))   # derive a column
    .groupBy("country")                             # one output row per country
    .agg(F.sum("revenue").alias("total"))           # sum revenue in each
    .orderBy(F.desc("total"))                       # biggest first
)

result.show()                                       # only NOW does any of it run
```

> **Read a chain top to bottom as a sentence:** take df, keep the valid rows, work out revenue, group by country, total it up, sort it. That's the whole mental model.

### The two words that explain most error messages

| Word | Means | Why you care |
|---|---|---|
| **Driver** | The machine running *your* Python code | `collect()` and `toPandas()` pull data **here** — it has limited memory and will crash |
| **Executor** | The machines doing the actual work | Where your `filter` and `groupBy` run, in parallel |

> ⚠️ **`OutOfMemoryError` on the driver** almost always means you called `.collect()` or `.toPandas()` on something big. Use `.show()`, `.take(5)`, or `.limit(100).toPandas()`.

### Spark vs pandas — the same operations

| Task | pandas | PySpark |
|---|---|---|
| Read | `pd.read_parquet(p)` | `spark.read.parquet(p)` |
| Look | `df.head()` | `df.show(5)` |
| Types | `df.dtypes` | `df.printSchema()` |
| Pick columns | `df[["a","b"]]` | `df.select("a","b")` |
| Filter | `df[df.qty > 0]` | `df.filter(F.col("qty") > 0)` |
| New column | `df["c"] = df.a * 2` | `df.withColumn("c", F.col("a") * 2)` |
| Group | `df.groupby("k").sum()` | `df.groupBy("k").agg(F.sum("x"))` |
| Rename | `df.rename(columns=...)` | `df.withColumnRenamed("a","b")` |
| Join | `df1.merge(df2, on="id")` | `df1.join(df2, "id")` |
| Rows | `len(df)` | `df.count()` |

> **The big differences:** Spark has no index, no in-place edits, and is lazy. If your data fits comfortably in memory, **[[pandas]] or [[Polars]] will be faster and simpler** — Spark only pays off above roughly one machine's worth of data ([[When to leave Python]]).

---

## 1. Setup

### Creating a SparkSession — the simple version

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.getOrCreate()   # that's it - start Spark (or reuse the one already running)
```

**This is all you need to start.** A `SparkSession` is your connection to Spark: you use `spark` to read files, run SQL and build DataFrames.

`getOrCreate()` means *"give me the Spark session that's already running, or start a new one if there isn't one."* That's why running the cell twice in a notebook doesn't start Spark twice.

> **In Databricks, Microsoft Fabric or most hosted notebooks, `spark` already exists.** Skip this step entirely and just use `spark`.

### Adding settings — only when you need them

Each `.something()` between `.builder` and `.getOrCreate()` changes one setting. You add only the ones you need:

```python
from pyspark.sql import SparkSession

spark = (SparkSession.builder
    .appName("taxi-cleaning")                        # a name so you can find this job in the Spark UI
    .config("spark.driver.memory", "4g")             # more memory, if you get "Java heap space" errors
    .config("spark.sql.shuffle.partitions", "8")     # fewer pieces after a shuffle, for small data on a laptop
    .getOrCreate())

spark.sparkContext.setLogLevel("WARN")               # hide the wall of INFO messages
```

**What each setting does, and when you'd actually want it:**

| Setting | What it does | Add it when… | If you leave it out |
|---|---|---|---|
| `.appName("…")` | Names the job | You want to find it in the Spark UI at http://localhost:4040 | Called `pyspark-shell` |
| `.master("local[*]")` | Where Spark runs. `local[*]` = this computer, all CPU cores | Almost never locally. **Never on a real cluster** (the cluster sets it) | Already `local[*]` when you run a plain Python script |
| `.config("spark.driver.memory", "4g")` | Memory for the process running *your* code | You get `OutOfMemoryError: Java heap space`, or you call `.toPandas()` on something big | 1 GB |
| `.config("spark.sql.shuffle.partitions", "8")` | How many pieces data is split into after a `groupBy` or `join` | Small data on a laptop, where 200 tiny pieces waste time | 200 (Spark usually shrinks this automatically — see below) |
| `setLogLevel("WARN")` | How chatty Spark is in your terminal | Always, when working locally | Pages of INFO lines hide your own output |

> **Settings you'll see in tutorials but don't need on Spark 3.2+:** `spark.sql.adaptive.enabled` and `spark.sql.adaptive.skewJoin.enabled` are **already `true` by default** — adding them changes nothing. Adaptive execution (AQE) also merges tiny partitions for you, which is why the default of 200 hurts less than it used to. *(Checked on Spark 4.2.)*

> ⚠️ **`spark.driver.memory` only works before Spark starts.** Set it in the builder in a fresh script. Once a session is running (for example in a notebook where you already ran a cell), changing it does nothing — restart the kernel.

### Session housekeeping

| Task | Code |
|---|---|
| See a setting | `spark.conf.get("spark.sql.shuffle.partitions")` |
| Change a setting while running | `spark.conf.set("spark.sql.shuffle.partitions", "8")` |
| Spark version | `spark.version` |
| Stop Spark | `spark.stop()` |
| Web UI (jobs, stages, what's slow) | http://localhost:4040 |

---

## 2. Reading data

### CSV — the simple version

```python
df = spark.read.option("header", True).csv("data/sales.csv")   # first row = column names
```

That's usually enough to **look** at a file. Two things to know about it:

- **Every column comes in as text (`string`).** `price` is `"4.50"`, not `4.5`.
- Without `header`, the first row is treated as data and your columns are called `_c0`, `_c1`, …

**Next step up — let Spark guess the types** (fine for exploring, not for pipelines):

```python
df = (spark.read
    .option("header", True)        # first row = column names
    .option("inferSchema", True)   # read the file an extra time and guess each column's type
    .csv("data/sales.csv"))
```

> ⚠️ **Why `inferSchema` is only for exploring:** it reads the whole file twice (slow on big data), and it guesses wrong in ways that quietly damage data. On a test file, an ID of `001` came back as the number `1`, and one `NA` in a price column made the **whole column** text.

**The proper version — tell Spark the types yourself:**

```python
schema = "order_id STRING, product STRING, price DOUBLE, order_date DATE"   # column name + type

df = (spark.read
    .option("header", True)
    .schema(schema)                 # use these types - no guessing, no second read
    .csv("data/sales.csv"))
```

> **IDs, postcodes and phone numbers are `STRING`, not numbers** — they can have leading zeros and you never do maths on them. All the types are in §3.

### CSV — every option, and when you need it

Only add an option when your file needs it. This is the full list:

```python
df = (spark.read
    .option("header", "true")              # first line holds the column names, not data
    .option("sep", ",")                    # field separator - use "\t" for TSV, "|" for pipe-delimited
    .option("inferSchema", "false")        # do NOT guess types - we give a schema below
    .option("nullValue", "NA")             # treat the literal text "NA" as a real null
    .option("nanValue", "NaN")             # treat "NaN" as not-a-number in float columns
    .option("dateFormat", "yyyy-MM-dd")            # how dates are written in the file
    .option("timestampFormat", "yyyy-MM-dd HH:mm:ss")   # how timestamps are written
    .option("quote", '"')                  # character wrapping fields that contain the separator
    .option("escape", "\\")                # character that escapes a quote inside a quoted field
    .option("multiLine", "true")           # allow a quoted field to contain newlines (slower)
    .option("encoding", "UTF-8")           # file encoding - Windows exports are often "ISO-8859-1"
    .option("mode", "PERMISSIVE")          # what to do with bad rows: PERMISSIVE | DROPMALFORMED | FAILFAST
    .schema(my_schema)                     # declare the types yourself - fast and predictable
    .csv("path/*.csv"))                    # a glob works; Spark reads every matching file as one table
```

| Option | Default | You need it when… |
|---|---|---|
| `header` | `false` | **Almost always** — the first row is column names |
| `sep` | `,` | The file uses tabs (`\t`), pipes (`\|`) or semicolons (`;` — common in European Excel exports) |
| `inferSchema` | `false` | Quick exploring only — see above |
| `nullValue` | empty string | Missing values are written as `NA`, `NULL`, `-` or similar |
| `nanValue` | `NaN` | Rarely — only if not-a-number is written differently |
| `dateFormat` / `timestampFormat` | ISO (`2026-01-31`) | Dates are written some other way, e.g. `dd/MM/yyyy` |
| `quote` / `escape` | `"` and `\` | Rarely — only for unusual files |
| `multiLine` | `false` | A text field contains line breaks (e.g. a comment or address). **Slower** — only turn on if needed |
| `encoding` | `UTF-8` | You see garbled characters like `Ã©` — try `ISO-8859-1` or `windows-1252` |
| `mode` | `PERMISSIVE` | You want bad rows to stop the job (`FAILFAST`) — see below |

> **Date format letters:** `yyyy` year · `MM` month · `dd` day · `HH` hour (24h) · `mm` minutes · `ss` seconds. Capital `MM` is month, lowercase `mm` is minutes — mixing them up is the classic mistake.

### ⚠️ Bad rows are silently turned into nulls

This is the one that catches people. In the default `PERMISSIVE` mode, a value that doesn't fit its type — `"notanumber"` in a `DOUBLE` column — **quietly becomes `null`**. No error, no warning. You lose the value and don't know.

To **see** the broken rows, add a column called `_corrupt_record` to your schema. Spark then puts the original raw line there:

```python
from pyspark.sql import functions as F

schema = "order_id STRING, product STRING, price DOUBLE, order_date DATE, _corrupt_record STRING"   # every column + the extra one

df = spark.read.option("header", True).schema(schema).csv("data/sales.csv")
df.cache()                                              # needed before filtering on _corrupt_record

bad = df.filter(F.col("_corrupt_record").isNotNull())
print(bad.count(), "bad rows")
bad.show(truncate=False)                                # the raw text of each broken line
```

| `mode` | A bad row… | Use when |
|---|---|---|
| `PERMISSIVE` *(default)* | Keeps the row, bad values become `null` (raw line in `_corrupt_record` **if that column is in your schema**) | Normally — then count the bad rows |
| `DROPMALFORMED` | Throws the row away, silently | Almost never — you won't know what you lost |
| `FAILFAST` | Stops the whole job with an error | The data must be perfect, e.g. finance |

> ⚠️ **List every column in the file, then add `_corrupt_record` at the end.** If your schema has fewer columns than the file, Spark treats *every* row as broken — on a test file, leaving out one column flagged the perfectly good rows too.

> **Without `_corrupt_record` in the schema you can't see which rows were bad** — `columnNameOfCorruptRecord` only renames that column, it doesn't create it. *(Checked on Spark 4.2: without the column, the bad price just became `null`.)*

> **Why `.cache()` before filtering on it?** Without it Spark refuses with `QUERY_ONLY_CORRUPT_RECORD_COLUMN` — it won't run a query that uses *only* that column straight from the file. Caching reads the file once first, and then the filter works.

> ⚠️ **`df.count()` does not detect bad rows.** To count rows Spark doesn't need to read any column values, so it never notices they're broken: with `DROPMALFORMED` the count still includes the bad rows, and `FAILFAST` doesn't fail. The mode only takes effect when the data is actually read — `.show()`, `.collect()`, `.write`. *(Checked on Spark 4.2: 2 rows from `count()`, 1 from `collect()`.)*

### Parquet, Delta, JSON, ORC, Avro

```python
spark.read.parquet("path/")            # schema is stored INSIDE the file - no options needed
spark.read.format("delta").load("path/")                       # a Delta table by path
spark.read.format("delta").option("versionAsOf", 3).load("path/")   # TIME TRAVEL to version 3
spark.read.format("delta").option("timestampAsOf", "2026-01-01").load("path/")   # as it was that day

spark.read.option("multiLine", "true").json("path/")   # default expects ONE JSON object per line;
                                                       # multiLine=true for a pretty-printed array
spark.read.orc("path/")                # ORC - columnar, common in the Hive world
spark.read.format("avro").load("path/")# Avro - row-based, common with Kafka
spark.read.text("path/")               # raw lines: one row per line, single column called "value"
```

### JDBC (databases)

```python
df = (spark.read.format("jdbc")
    .option("url", "jdbc:postgresql://host:5432/db")     # connection string for the database
    .option("dbtable", "public.sales")     # a table name, or "(SELECT ...) AS t" to push work to the DB
    .option("user", user).option("password", pw)         # credentials - from a secret store, never literals
    .option("driver", "org.postgresql.Driver")           # the JDBC driver class; the jar must be on the path
    # --- parallel read: without these it's SINGLE-THREADED ---
    .option("partitionColumn", "id")       # the column Spark splits on - numeric, date or timestamp
    .option("lowerBound", "1").option("upperBound", "1000000")   # the RANGE to divide up
    .option("numPartitions", "8")          # into 8 concurrent queries -> 8 connections to the database
    .option("fetchsize", "10000")          # rows per network round trip; the default is far too small
    .load())
```

> **A JDBC read without `partitionColumn` uses one connection and one core**, however big your cluster. On a large table that's the whole bottleneck. `partitionColumn` must be numeric, date or timestamp, and evenly distributed.

### Tables and paths

```python
spark.table("catalog.schema.table")    # read a table registered in the catalog (Unity, Hive, Glue)
spark.read.load("path", format="delta")# generic form - format given as an argument
spark.read.parquet("s3a://bucket/year=2024/*/*.parquet")   # globs work; * matches one path level
spark.read.option("recursiveFileLookup", "true").parquet("dir/")  # every file in every subfolder
spark.read.option("pathGlobFilter", "*.parquet").load("dir/")     # only files matching this pattern
spark.read.option("modifiedAfter", "2024-01-01T00:00:00").load("dir/")  # only recently changed files
```

### Creating DataFrames directly

```python
spark.createDataFrame([(1, "a"), (2, "b")], ["id", "name"])   # from Python tuples + column names
spark.createDataFrame(pandas_df)             # from a pandas DataFrame - great for small test fixtures
spark.range(0, 1000, step=1, numPartitions=4)# 1000 rows, one column called "id" - handy for testing
spark.createDataFrame([], schema)            # an EMPTY DataFrame that still has the right columns
```

---

## 3. Schemas and types

A **schema** is the list of columns, their types, and whether each may be null. Spark either infers it or you declare it.

```python
from pyspark.sql import types as T

schema = T.StructType([                                          # StructType = the whole table's shape
    T.StructField("invoice_no",   T.StringType(),    True),      # name, type, nullable=True
    T.StructField("quantity",     T.IntegerType(),   True),      # whole number, 32-bit
    T.StructField("unit_price",   T.DoubleType(),    True),      # decimal measurement, 64-bit
    T.StructField("amount",       T.DecimalType(18, 2), True),   # exact money: 18 digits, 2 after the point
    T.StructField("invoice_date", T.TimestampType(), True),      # date AND time
    T.StructField("is_active",    T.BooleanType(),   True),      # true / false
    T.StructField("tags",         T.ArrayType(T.StringType()), True),                  # a list inside one cell
    T.StructField("meta",         T.MapType(T.StringType(), T.StringType()), True),    # key->value inside one cell
    T.StructField("address", T.StructType([                      # a nested record inside one cell
        T.StructField("city", T.StringType()),                   # nullable defaults to True
        T.StructField("post", T.StringType()),
    ])),
])

df = spark.read.schema(schema).csv("data.csv", header=True)      # declare it - fast and safe
```

> **Always declare the schema in production.** `inferSchema=True` reads the file twice (slow) and can guess differently next month when the data changes — a column that was `IntegerType` becomes `StringType` and everything downstream breaks.

---

### `df.printSchema()` — reading the output

```python
df.printSchema()      # prints the schema as an indented tree. Costs nothing - no job runs.
```

```
root
 |-- invoice_no: string (nullable = true)
 |-- quantity: integer (nullable = true)
 |-- amount: decimal(18,2) (nullable = true)
 |-- invoice_date: timestamp (nullable = true)
 |-- tags: array (nullable = true)
 |    |-- element: string (containsNull = true)
 |-- address: struct (nullable = true)
 |    |-- city: string (nullable = true)
 |    |-- post: string (nullable = true)
```

**How to read it:**

| Part | Means |
|---|---|
| `root` | The top of the tree — the row itself |
| `\|--` | One column |
| `\|    \|--` | A field **nested inside** the column above it |
| `string`, `integer` | The column's type (lowercase here, `StringType()` in code) |
| `nullable = true` | This column is allowed to contain nulls |
| `containsNull = true` | On an array: the **elements** may be null |
| `valueContainsNull` | On a map: the **values** may be null |

> **`nullable = true` is a promise Spark makes to itself, not a constraint it enforces.** Spark will not reject a null in a `nullable = false` column — it just optimises assuming there aren't any, which produces wrong answers if there are. **Never lie in your schema.**

---

### Every type, explained

#### Text

| Type | Python equivalent | Notes |
|---|---|---|
| `StringType()` | `str` | Text of any length. **All identifiers go here.** |
| `VarcharType(n)` / `CharType(n)` | `str` | Length-limited. Only meaningful in table DDL; Spark treats them as strings in a DataFrame |

> **Identifiers are strings, always.** A customer ID stored as an integer loses leading zeros (`00123` → `123`) and invites nonsense like `AVG(customer_id)`. Postcodes, phone numbers, product codes and order references are text, not numbers.

#### Whole numbers

| Type | Bytes | Range | Use for |
|---|---|---|---|
| `ByteType()` | 1 | −128 to 127 | Tiny flags, enum codes |
| `ShortType()` | 2 | ±32,767 | Small counters |
| **`IntegerType()`** | 4 | ±2.1 billion | **The normal choice** |
| **`LongType()`** | 8 | ±9.2 quintillion | **Big IDs, epoch milliseconds, row counts** |

> ⚠️ **An `IntegerType` overflows silently at about 2.1 billion.** It does not raise an error — it wraps around to a negative number. Any counter that could grow, and anything holding milliseconds since 1970, must be `LongType`.

#### Decimal numbers

| Type | Bytes | Precision | Use for |
|---|---|---|---|
| `FloatType()` | 4 | ~7 digits | Rarely worth it — save the memory elsewhere |
| **`DoubleType()`** | 8 | ~15 digits | **Measurements: temperature, weight, latitude** |
| **`DecimalType(p, s)`** | varies | **Exact** | **Money, tax, anything that must balance** |

`DecimalType(18, 2)` means 18 total digits, 2 of them after the decimal point — so up to `9999999999999999.99`.

> ⚠️ **Never store money in a `DoubleType`.** Binary floating point cannot represent `0.1` exactly, so `0.1 + 0.2` gives `0.30000000000000004`. Sum a million transactions and the total is wrong by real pennies that a real accountant will find. `DecimalType` stores the digits exactly ([[Numerical methods in practice]], [[Data modeling]]).

> ⚠️ **Decimal arithmetic grows the precision.** Multiply `DecimalType(18,2)` by `DecimalType(18,2)` and you get `DecimalType(38,4)`. Exceed 38 digits and Spark returns **null**, silently. Cast back down after multiplying.

#### Dates and times

| Type | Holds | Use for |
|---|---|---|
| `DateType()` | Year, month, day only | Birthdays, invoice dates, partition keys |
| **`TimestampType()`** | Date + time, **adjusted to session time zone** | Events, log lines |
| `TimestampNTZType()` | Date + time, **no time zone at all** | Wall-clock readings where the zone is irrelevant |

> ⚠️ **`TimestampType` is time-zone sensitive.** Spark stores it as an absolute instant and renders it in `spark.sql.session.timeZone`. Two people running the same query in different zones see different dates, and rows land in different date partitions. **Set the session time zone explicitly to UTC** and treat everything as UTC until display:

```python
spark.conf.set("spark.sql.session.timeZone", "UTC")   # do this at the top of every job
```

> **Never store a date as a string.** `"2026-01-05"` and `"05/01/2026"` won't sort, compare or partition correctly, and date functions won't work on them.

#### Boolean and binary

| Type | Notes |
|---|---|
| `BooleanType()` | `True` / `False` — **and `null`, which is a third state** |
| `BinaryType()` | Raw bytes: images, serialised blobs, hashes |

> ⚠️ **A nullable boolean has three values, not two.** `F.col("flag") == False` does **not** match null rows. Use `F.col("flag").eqNullSafe(False)`, or fill the nulls first.

#### Nested types

| Type | Shape | Access it with |
|---|---|---|
| `ArrayType(elem)` | An ordered list in one cell | `F.col("tags")[0]`, `F.explode`, `F.size` |
| `MapType(key, val)` | Key→value pairs in one cell | `F.col("meta")["source"]` |
| `StructType([...])` | A nested record | `F.col("address.city")` — **dot notation** |

```python
from pyspark.sql import functions as F

df.select("address.city")                        # reach into a struct
df.select(F.col("tags")[0].alias("first_tag"))   # first element of an array
df.select(F.explode("tags").alias("tag"))        # one ROW per array element
df.select(F.size("tags").alias("n_tags"))        # array length (-1 if the array is null)
df.select(F.col("meta")["source"])               # look up a map key
```

> **Structs are free; arrays and maps cost you.** A struct is just columns with a prefix, and Spark reads only the sub-fields you ask for. Arrays and maps block many optimisations — flatten them early unless the nesting is genuinely part of your data ([[Data modeling]]).

#### The unhelpful types

| Type | Why you'll see it |
|---|---|
| `NullType()` | A column that was **entirely null** when inferred — Spark has no idea what it is. **Cast it, or writing to Parquet will fail.** |
| `VoidType()` | Same thing, from `F.lit(None)` — always cast: `F.lit(None).cast("string")` |

---

### Checking and changing types

```python
import json
from pyspark.sql import types as T

df.printSchema()          # the tree view above - what you use 95% of the time
df.dtypes                 # [('invoice_no', 'string'), ('quantity', 'int'), ...]
df.columns                # ['invoice_no', 'quantity', ...] - just the names
df.schema                 # the StructType object itself
df.schema["quantity"].dataType        # DataType of one column
df.schema.fieldNames()                # same as df.columns
df.schema.json()                      # serialise the schema to JSON text
T.StructType.fromJson(json.loads(s))  # rebuild it from that JSON
```

```python
df.select([c for c, ty in df.dtypes if ty in ("int", "bigint", "double")])   # all numeric columns
df.select([c for c, ty in df.dtypes if ty == "string"])                      # all text columns
```

**Casting:**

```python
from pyspark.sql import functions as F
from pyspark.sql import types as T

df.withColumn("quantity", F.col("quantity").cast("int"))                 # string -> int
df.withColumn("amount", F.col("amount").cast(T.DecimalType(18, 2)))      # -> exact decimal
df.withColumn("d", F.to_date("date_str", "yyyy-MM-dd"))                  # string -> date
df.withColumn("ts", F.to_timestamp("ts_str", "yyyy-MM-dd HH:mm:ss"))     # string -> timestamp
```

> ⚠️ **A failed cast produces `null`, it does not raise an error.** `"abc".cast("int")` is `null`. Casting a whole column and silently turning 40% of it into nulls is a classic quiet data-loss bug. **Always count the nulls before and after a cast:**

```python
from pyspark.sql import functions as F

before = df.filter(F.col("quantity").isNotNull()).count()                # non-null before
after = df.withColumn("q", F.col("quantity").cast("int")) \
          .filter(F.col("q").isNotNull()).count()                        # non-null after
print(f"cast lost {before - after} values")                              # should be 0
```

**Schema shortcuts:**

```python
schema = "invoice_no STRING, quantity INT, unit_price DOUBLE"   # DDL string - quick and readable
df = spark.read.schema(schema).csv("data.csv", header=True)

df2.schema == df1.schema                     # do two DataFrames match exactly?
set(df1.columns) - set(df2.columns)          # which columns are missing from df2?
```


---

## 4. Investigating data

**The first thing you do with any new dataset.** Do this before writing a single transformation.

### The first five commands

```python
df.printSchema()                 # what columns and types
df.count()                       # how many rows
df.show(20, truncate=False)      # what does it look like
df.describe().show()             # min/max/mean/stddev/count per numeric column
len(df.columns)                  # how wide
```

### Looking at rows

```python
df.show()                                # first 20 rows, columns cut off at 20 characters
df.show(5)                               # first 5 rows only
df.show(5, truncate=False)               # 5 rows, show long strings in FULL (nothing cut)
df.show(5, truncate=30)                  # 5 rows, cut each value at 30 characters
df.show(2, vertical=True)                # 2 rows, ONE FIELD PER LINE - see below
df.show(5, truncate=False, vertical=True)  # all three together

df.head(5)                               # returns a LIST of 5 Row objects (data, not printed)
df.head()                                # one Row, or None if the DataFrame is empty
df.first()                               # one Row - raises an error if empty
df.take(5)                               # same as head(5) - a list of Rows
df.tail(5)                               # the LAST 5 rows - must scan everything, use sparingly
df.limit(10).toPandas()                  # 10 rows as a pandas DataFrame - nice table in a notebook
display(df)                              # Databricks / Fabric only - interactive sortable grid
```

#### `vertical=True` — the one to remember for wide tables

Normally `show()` prints a grid. With 30 columns that grid wraps across your screen and becomes unreadable.

```python
df.show(2, vertical=True)                # 2 rows, printed one field per line
```

```
-RECORD 0-------------------------
 invoice_no   | 536365
 quantity     | 6
 unit_price   | 2.55
 invoice_date | 2010-12-01 08:26:00
 country      | United Kingdom
-RECORD 1-------------------------
 invoice_no   | 536365
 quantity     | 6
 unit_price   | 3.39
 invoice_date | 2010-12-01 08:26:00
 country      | United Kingdom
```

Instead of one row per line, you get **one field per line**, with each record separated by a `-RECORD n-` header.

> **Use `vertical=True` whenever you have more than about eight columns.** It is the difference between reading your data and squinting at wrapped text. Keep the row count small — `df.show(2, vertical=True)` or `df.show(1, vertical=True)` — because each row now takes up as many lines as you have columns.

**Where it's genuinely essential:**

```python
from pyspark.sql import functions as F

df.show(1, vertical=True)                                    # inspect one full record

# a one-row summary with 40 columns is unreadable any other way
df.select([F.count(F.when(F.col(c).isNull(), c)).alias(c)
           for c in df.columns]).show(vertical=True)         # null count per column

df.describe().show(vertical=True)                            # stats on a wide table
df.filter(F.col("invoice_no") == "536365").show(2, vertical=True)   # debug specific rows
```

#### What each one actually returns

| Call | Returns | Runs a Spark job? |
|---|---|---|
| `df.show(n)` | **Nothing** — prints to the console | Yes |
| `df.head(n)` / `df.take(n)` | A Python **list of `Row`** objects | Yes |
| `df.first()` | A single `Row` | Yes |
| `df.limit(n)` | **Another DataFrame** — still lazy | No, not until you act on it |
| `df.toPandas()` | A pandas DataFrame **in driver memory** | Yes |
| `df.printSchema()` | Nothing — prints the schema | **No** — it's free |

```python
row = df.first()                         # one Row
row["invoice_no"]                        # access by column name
row.invoice_no                           # or by attribute
row.asDict()                             # turn it into a normal Python dict
```

> ⚠️ **`df.collect()` pulls every row to the driver and will crash it on a large DataFrame.** Use `show`, `take`, or `limit(n).toPandas()`. This is the number one way beginners kill a Spark job.

> ⚠️ **`df.toPandas()` does the same thing.** It loads the entire DataFrame into the driver's memory. Always put a `.limit()` in front of it unless you have already aggregated down to something small.

### Profiling

```python
df.describe().show()      # count, mean, stddev, min, max for every numeric column
df.summary().show()       # the same PLUS the 25%, 50% (median) and 75% percentiles
df.summary("count", "min", "25%", "75%", "max").show()   # pick exactly which stats you want

df.select("price").summary().show()               # restrict it to one column
# for the mean/sum/max of ONE column as a Python number, see section 13
df.approxQuantile("price", [0.25, 0.5, 0.75], 0.01)   # quartiles; 0.01 = allow 1% error, much faster
df.stat.corr("price", "quantity")   # Pearson correlation, -1 to 1. Only finds STRAIGHT-LINE relationships.
df.stat.cov("price", "quantity")    # covariance - same idea but unscaled, so harder to interpret
df.stat.crosstab("country", "segment").show()     # counts for every country x segment combination
df.stat.freqItems(["country"], support=0.3).show()# values appearing in at least 30% of rows
```

### Counting and distinctness

```python
from pyspark.sql import functions as F

df.count()                                   # total rows. Scans everything - not free on big data.
df.distinct().count()                        # rows that are unique across ALL columns
df.select("country").distinct().show()       # the list of distinct countries
df.select("country").distinct().count()      # how many distinct countries
df.select(F.countDistinct("customer_id")).show()   # exact unique count - needs a full shuffle
df.select(F.approx_count_distinct("customer_id", rsd=0.01)).show()
                                             # approximate, ~1% error, DRAMATICALLY faster on big data
df.groupBy("country").count().orderBy(F.desc("count")).show()
                                             # value counts, most common first - pandas value_counts()
```

### Finding unique values

**"What are all the different values in this column?"** This is pandas' `df["country"].unique()`. In Spark you pick the column, then call `.distinct()`.

**The simple version:**

```python
df.select("country").distinct().orderBy("country").show()
```

```
+-------+
|country|
+-------+
|   NULL|        <- a missing value counts as one of the unique values
|     FR|
|     UK|
|     US|
+-------+
```

> **Add `.orderBy(...)` when you want the list sorted.** `distinct()` returns the values in whatever order Spark finds them, and that order can change from run to run.

**Everything else you'll want to know about unique values:**

```python
from pyspark.sql import functions as F

# 1. As a Python list, e.g. to loop over or print
countries = [row["country"] for row in df.select("country").distinct().orderBy("country").collect()]
# -> [None, 'FR', 'UK', 'US']        None is Python's name for a null

# 2. HOW MANY unique values - careful, the two ways disagree about nulls
df.select("country").distinct().count()             # 4 - counts NULL as a value
df.select(F.countDistinct("country")).first()[0]    # 3 - skips nulls
df.select(F.approx_count_distinct("country")).first()[0]
                                                     # 3 - an estimate, much faster on huge data

# 3. Each unique value AND how often it appears (pandas value_counts)
total = df.count()
(df.groupBy("country").count()                                   # one row per unique value, with its count
   .withColumn("pct", F.round(F.col("count") / total * 100, 1))  # its share of all rows, as a %
   .orderBy(F.desc("count"), "country")                          # most common first, ties A-Z
   .show())

# 4. Unique COMBINATIONS of several columns
df.select("country", "product").distinct().show()  # every (country, product) pair that exists

# 5. Values that appear exactly ONCE
df.groupBy("product").count().filter(F.col("count") == 1).show()

# 6. The unique values inside each group
(df.groupBy("country")
   .agg(F.array_sort(F.collect_set("product")).alias("products"),  # the list of different products, sorted
        F.countDistinct("product").alias("n_products"))             # how many different products
   .orderBy("country")
   .show(truncate=False))                                           # truncate=False = don't cut off the lists

# 7. Is this column a unique key? (every value appears once)
n_rows = df.count()
n_unique = df.select("order_id").distinct().count()
print("unique" if n_rows == n_unique else f"{n_rows - n_unique} duplicate values")

# 8. Unique values inside an ARRAY column, e.g. tags = ["office", "lighting"]
df.select(F.explode("tags").alias("tag")).distinct().orderBy("tag").show()
                                                     # explode = one row per array item, then distinct as normal

# 9. Unique values of several columns at once - small categorical columns only
for c in ["country", "product"]:
    print(c, [row[0] for row in df.select(c).distinct().collect()])
```

Output of 3, 6 and 9 on a small orders table:

```
+-------+-----+----+          +-------+-----------+----------+
|country|count| pct|          |country|products   |n_products|
+-------+-----+----+          +-------+-----------+----------+
|     UK|    3|42.9|          |NULL   |[mug]      |1         |
|     FR|    2|28.6|          |FR     |[mug, pen] |2         |
|   NULL|    1|14.3|          |UK     |[lamp, mug]|2         |
|     US|    1|14.3|          |US     |[pen]      |1         |
+-------+-----+----+          +-------+-----------+----------+

country ['UK', 'FR', 'US', None]      <- no orderBy, so in whatever order Spark found them
product ['mug', 'lamp', 'pen']
```

> ⚠️ **`.collect()` pulls every unique value into your Python program.** That's fine for `country` (a few hundred values). On `customer_id` it can be millions of values and crash the driver. Check how many there are first with `countDistinct`. If it's large, look at a sample with `.distinct().show(50)` or `.distinct().limit(1000).collect()`.

> ⚠️ **`distinct()` counts NULL as a value. `countDistinct` ignores it.** That's why the two gave 4 and 3 above. If the numbers don't match, the column has nulls ([[#Nulls — the most important check]]).

> **`distinct()` shuffles data across the cluster** ([[#The two words that explain most error messages]]). It's fine to run once when you're exploring. Don't leave it inside a job that runs every hour unless you actually need it.

| pandas | PySpark |
|---|---|
| `df["c"].unique()` | `df.select("c").distinct().collect()` |
| `df["c"].nunique()` | `df.select(F.countDistinct("c")).first()[0]` |
| `df["c"].value_counts()` | `df.groupBy("c").count().orderBy(F.desc("count"))` |
| `df[["a", "b"]].drop_duplicates()` | `df.select("a", "b").distinct()` |
| `df["c"].is_unique` | `df.count() == df.select("c").distinct().count()` |

### Nulls — the most important check

```python
from pyspark.sql import functions as F

# null count per column, in ONE pass over the data
df.select([
    F.count(F.when(F.col(c).isNull(), c)).alias(c)   # count() ignores nulls, so when() flips the test:
    for c in df.columns                              # emit the column name when null, nothing otherwise
]).show(vertical=True)                               # vertical=True because this is 1 row x N columns

# null RATE per column - far more useful than a raw count
total = df.count()                                   # denominator, computed once
df.select([
    F.round(F.count(F.when(F.col(c).isNull(), c)) / total * 100, 2).alias(c)   # as a % to 2dp
    for c in df.columns
]).show(vertical=True)

# rows with ANY null in them
df.filter(F.greatest(*[F.col(c).isNull() for c in df.columns])).count()
# greatest() over booleans = true if ANY is true, so this counts rows with at least one null
```

> **Null rate per column is the single most valuable profiling output.** A column that was 2% null and is now 60% null means something upstream changed ([[Observability for data and ML pipelines]]).

### Duplicates

```python
from pyspark.sql import functions as F

df.count() - df.distinct().count()      # how many rows are exact duplicates of another row

# duplicates on a KEY - far more important, this is what breaks joins
(df.groupBy("invoice_no", "product_id").count()   # count rows per key
   .filter(F.col("count") > 1)                    # keep only keys appearing more than once
   .orderBy(F.desc("count")).show())              # worst offenders first
# if this returns anything, that key is NOT unique - joining on it will multiply your rows
```

### Cardinality and skew

```python
from pyspark.sql import functions as F

# how many distinct values per column - tells you what is categorical vs continuous
df.select([F.countDistinct(c).alias(c) for c in df.columns]).show(vertical=True)
# 2-50 distinct = categorical · thousands = an ID · nearly all rows = continuous or a key

# is a key SKEWED? (skew is what kills Spark jobs)
(df.groupBy("customer_id").count()
   .orderBy(F.desc("count")).show(10))   # the 10 biggest keys
# if #1 has 100x the rows of #10, one task will do most of the work and everyone waits

# actual partition sizes - wildly uneven numbers mean skew
df.rdd.glom().map(len).collect()   # glom() = rows per partition as a list; collect() pulls the SIZES only
```

### Numeric ranges and outliers

```python
from pyspark.sql import functions as F

df.select(F.min("price"), F.max("price"), F.mean("price"), F.stddev("price")).show()
# check min and max FIRST - a negative price or a year-3000 date shows up here immediately

# IQR outlier bounds - the standard statistical definition of "unusually far out"
q1, q3 = df.approxQuantile("price", [0.25, 0.75], 0.01)   # 25th and 75th percentile
iqr = q3 - q1                                             # inter-quartile range = the middle 50%
lower, upper = q1 - 1.5 * iqr, q3 + 1.5 * iqr             # 1.5x IQR beyond each quartile
df.filter((F.col("price") < lower) | (F.col("price") > upper)).count()   # how many fall outside

# impossible values - these are DATA BUGS, not outliers. Different problem, different fix.
df.filter(F.col("quantity") < 0).count()   # negative quantity: a return, or a broken feed?
df.filter(F.col("price") <= 0).count()     # free items, or a missing value written as 0?
```

### Execution plan

```python
df.explain()                   # the PHYSICAL plan - what Spark will actually run
df.explain(True)               # all four stages: parsed, analysed, optimised, physical
df.explain(mode="formatted")   # the same information, laid out readably. Start here.
df.rdd.getNumPartitions()      # how many chunks the work is split into
df.isEmpty()                   # true/false - MUCH cheaper than count() == 0, it stops at row 1
df.storageLevel                # is this DataFrame cached, and where (memory/disk)?

# in the plan, look for: Exchange = a SHUFFLE (expensive) · BroadcastHashJoin = good, no shuffle
# · SortMergeJoin = a shuffle on both sides · PushedFilters = your filter reached the files (good)
```

---

## 5. Selecting columns

### Displaying only the columns you want

**This is the most common thing you'll do.** `select` picks columns; chain `.show()` to print them.

```python
from pyspark.sql import functions as F

df.select("invoice_no", "quantity").show()            # only these 2 columns, first 20 rows
df.select("invoice_no", "quantity").show(5)           # only these 2 columns, first 5 rows
df.select("invoice_no", "quantity").show(2, vertical=True)   # 2 rows, one field per line

df.select(["invoice_no", "quantity", "price"]).show() # a list works too
df.select(F.col("price")).show()                      # F.col() form - identical result
df.select("*").show()                                 # every column (the default anyway)
```

> **`select` returns a NEW DataFrame — it does not change `df`.** Nothing in Spark modifies a DataFrame in place. If you want to keep the result, assign it: `small = df.select("a", "b")`.

**Picking columns without typing them all out:**

```python
df.select(df.columns[:5]).show()                              # the FIRST 5 columns
df.select(df.columns[-3:]).show()                             # the LAST 3 columns
df.select(sorted(df.columns)).show()                          # every column, alphabetical

df.select([c for c in df.columns if c.startswith("cust")]).show()     # by name prefix
df.select([c for c in df.columns if "date" in c.lower()]).show()      # by name contains
df.select([c for c, ty in df.dtypes if ty == "string"]).show()        # all TEXT columns
df.select([c for c, ty in df.dtypes if ty in ("int", "bigint", "double")]).show()  # all NUMERIC

df.drop("big_json_blob", "internal_id").show()                # everything EXCEPT these
```

> **`drop` is the inverse of `select`.** With 50 columns and 2 you don't want, `drop` is far shorter than listing 48. `drop` also ignores names that don't exist, so it never crashes on a typo — which is convenient and occasionally hides a mistake.

**Renaming while you select:**

```python
from pyspark.sql import functions as F

df.select(F.col("invoice_no").alias("invoice")).show()        # rename in the output only
df.select([F.col(c).alias(c.lower()) for c in df.columns])    # lowercase EVERY column name
df.select([F.col(c).alias(c.replace(" ", "_")) for c in df.columns])   # spaces -> underscores
df.withColumnRenamed("old_name", "new_name")                  # rename one, keep the rest
df.withColumnsRenamed({"a": "x", "b": "y"})                   # rename several, keep the rest
```

> **`alias` renames only in the DataFrame you're building; `withColumnRenamed` renames within the existing set of columns.** Use `alias` inside a `select`, `withColumnRenamed` when you just want to fix one name.

### Deleting columns

**The form to use when you're dropping several — one name per line, reassigned to `df`:**

```python
df = df.drop(
    "RatecodeID",
    "store_and_fwd_flag",
    "fare_amount",
    "extra",
    "mta_tax",
    "tolls_amount",
    "improvement_surcharge",
    "congestion_surcharge"
)
```

**Two things make that work, and both are easy to get wrong:**

1. **`df = df.drop(...)`** — you must reassign. `df.drop(...)` on its own builds a new DataFrame and throws it away; `df` is untouched and no error appears.
2. **Each name is its own argument**, separated by commas — not a list. `df.drop(["a", "b"])` does **not** work; `drop` expects the names spread out, not wrapped in brackets.

> **Put each column on its own line once you're past about three.** It's easier to read, easier to comment out one line while testing, and gives a clean one-line-per-column diff in git.

**The shorter forms, for fewer columns:**

```python
from pyspark.sql import functions as F

df = df.drop("internal_id")                          # remove ONE column
df = df.drop("internal_id", "big_json_blob")         # remove two
df = df.drop(F.col("internal_id"))                   # the F.col form also works
```

**If your column names are already in a Python list**, unpack it with `*`:

```python
junk = ["RatecodeID", "store_and_fwd_flag", "fare_amount"]

df = df.drop(*junk)          # ✅ the * spreads the list into separate arguments
df = df.drop(junk)           # ❌ passing the list itself - does nothing
```

> ⚠️ **This is the same trap as above.** `drop` never takes a list. `*` unpacks it into the comma-separated arguments `drop` actually wants — and because `drop` ignores names it doesn't recognise, passing the list raises no error and silently drops nothing.

**Dropping by a rule instead of by name:**

```python
df.drop(*[c for c in df.columns if c.startswith("tmp_")])       # every temp column
df.drop(*[c for c in df.columns if c.endswith("_raw")])         # every _raw column
df.drop(*[c for c, ty in df.dtypes if ty == "binary"])          # every binary column
df.drop(*[c for c in df.columns if c not in KEEP])              # keep only a known list
```

**Dropping empty or useless columns:**

```python
from pyspark.sql import functions as F

# columns that are 100% null
total = df.count()
nulls = df.select([F.count(F.when(F.col(c).isNull(), c)).alias(c) for c in df.columns]).first()
all_null = [c for c in df.columns if nulls[c] == total]         # names of the empty columns
df = df.drop(*all_null)

# columns with only ONE distinct value - they carry no information
counts = df.select([F.countDistinct(c).alias(c) for c in df.columns]).first()
constant = [c for c in df.columns if counts[c] <= 1]
df = df.drop(*constant)
```

> ⚠️ **`drop` silently ignores names that don't exist.** `df.drop("typo_name")` raises no error and changes nothing. Convenient, but it means a misspelled column name fails quietly — check with `assert "col" in df.columns` if it matters.

**Keeping instead of dropping:**

```python
df.select("id", "name", "price")       # KEEP these three - the inverse of drop
df.drop("internal_id", "debug_blob")   # keep everything EXCEPT these two
```

> **Use whichever is shorter.** With 50 columns and 2 you don't want, `drop` beats listing 48. With 50 columns and 3 you *do* want, `select` beats listing 47.

**Dropping a duplicate column after a join:**

```python
joined = a.join(b, a.id == b.id, "left")    # this leaves TWO columns called "id"
joined.drop(b.id)                            # drop the RIGHT one specifically - use the DataFrame prefix
```

> **`joined.drop("id")` by name would drop both.** Passing `b.id` targets exactly one. Better still, join with `a.join(b, "id")` so only one `id` is produced in the first place (§15).

**Dropping a field inside a struct:**

```python
from pyspark.sql import functions as F

df.withColumn("address", F.col("address").dropFields("post"))   # remove a nested field
df.drop("address")                                              # remove the whole struct
```

---

### ⚠️ `drop`, `na.drop` and `dropDuplicates` are three different things

The names look alike and they do completely unrelated jobs:

| Call | Removes | Think of it as |
|---|---|---|
| **`df.drop("col")`** | A **column** | Delete a column |
| **`df.na.drop()`** | **Rows** containing nulls | Delete incomplete rows |
| **`df.dropDuplicates()`** | **Rows** that repeat | Delete duplicate rows |
| `df.drop_duplicates()` | Same as `dropDuplicates` | An alias |

```python
df.drop("price")                     # the price COLUMN is gone
df.na.drop(subset=["price"])         # ROWS with a null price are gone
df.dropDuplicates(["id"])            # ROWS with a repeated id are gone
```

> **`drop` takes column names; `na.drop(subset=[...])` and `dropDuplicates([...])` take column names too — but to decide which *rows* to remove.** Same argument, opposite effect. This catches people constantly.

### Computing new columns

```python
from pyspark.sql import functions as F

df.select("*", F.lit(1).alias("one"))                    # keep everything, add a constant column
df.select("qty", "price",
          (F.col("qty") * F.col("price")).alias("revenue"))   # computed column, named

df.withColumn("revenue", F.col("qty") * F.col("price"))  # ADD a column, keep all the others
df.withColumn("qty", F.col("qty").cast("int"))           # REPLACE a column (same name = overwrite)
df.withColumns({                                          # add several in one pass
    "revenue": F.col("qty") * F.col("price"),
    "is_bulk": F.col("qty") > 100,
})

df.selectExpr("qty",                                      # SQL strings instead of F.col()
              "qty * price as revenue",
              "cast(price as int) as price_int")
```

> ⚠️ **Don't chain `withColumn` twenty times in a loop.** Each call builds another layer on the query plan, and a long chain gets genuinely slow to analyse. Use **one `select`** with a list, or `withColumns({...})`, when adding many columns at once.

### Referring to a column — five equivalent ways

```python
from pyspark.sql import functions as F

F.col("price")        # ALWAYS works - use this one
df["price"]           # works, including names with spaces
df.price              # breaks on names with spaces, dots, or DataFrame method names
"price"               # works wherever a plain string is accepted
F.expr("price * 2")   # a SQL expression as a string
```

> **Use `F.col()`.** `df.price` silently breaks on columns named `count`, `filter`, `join` or `describe`, because those are DataFrame methods — you get the method object instead of the column, and the error message points somewhere unrelated.

```python
from pyspark.sql import functions as F

df.select(F.col("`odd name`"))     # backticks for names with spaces or dots
```

### Column order and prefixes

```python
from pyspark.sql import functions as F

df.select("id", *[c for c in df.columns if c != "id"])       # move "id" to the front
df.select([F.col(c).alias(f"src_{c}") for c in df.columns])  # prefix every column (before a join)
```

> **Prefixing every column before a join is the cleanest fix for ambiguous names.** Otherwise `df1.join(df2, "id").select("name")` fails with `Reference 'name' is ambiguous` when both sides have a `name`.


---

## 6. Filtering

### One condition

```python
from pyspark.sql import functions as F

df.filter(F.col("qty") > 0)                      # keep rows where qty > 0
df.where(F.col("qty") > 0)                       # identical - "where" is a SQL-style alias
df.filter("qty > 0")                             # same thing written as a SQL string
```

### Combining conditions (AND, OR, NOT)

```python
from pyspark.sql import functions as F

df.filter((F.col("qty") > 0) & (F.col("country") == "UK"))   # AND - both must be true
df.filter((F.col("a") > 1) | (F.col("b") < 2))               # OR  - either can be true
df.filter(~F.col("is_cancelled"))                            # NOT - invert a boolean column
df.filter("qty > 0 AND country = 'UK'")                      # or write the whole thing as SQL
```

> ⚠️ **`&`, `|`, `~` — not `and`, `or`, `not`.** And the brackets around each condition are **mandatory**, because `&` binds tighter than `>`. `F.col("a") > 0 & F.col("b") == 1` gives a cryptic error.

### Is the value one of a list?

```python
from pyspark.sql import functions as F

df.filter(F.col("country").isin("UK", "FR", "DE"))   # value is one of these
df.filter(F.col("country").isin(my_list))            # same, from a Python list
df.filter(~F.col("country").isin(my_list))           # value is NOT in the list
```

### Filtering on nulls

```python
from pyspark.sql import functions as F

df.filter(F.col("x").isNull())                   # keep rows where x IS null
df.filter(F.col("x").isNotNull())                # keep rows where x is NOT null
df.filter(F.col("x").isNaN())                    # "not a number" - floats only, different from null
```

> ⚠️ **`F.col("x") == None` does not work.** Null is not a value you can compare to — use `isNull()`. Equally, `F.col("x") != "UK"` **drops the null rows too**, because `null != "UK"` is null, not true. If you want them, write `(F.col("x") != "UK") | F.col("x").isNull()`.

### Ranges

```python
from pyspark.sql import functions as F

df.filter(F.col("price").between(10, 100))       # 10 <= price <= 100, INCLUSIVE at both ends
df.filter((F.col("d") >= "2026-01-01") & (F.col("d") < "2026-02-01"))   # a month of dates
```

### Text matching

```python
from pyspark.sql import functions as F

df.filter(F.col("name").like("%mug%"))           # SQL LIKE: % = any characters, _ = exactly one
df.filter(F.col("name").ilike("%MUG%"))          # same but case-INsensitive
df.filter(F.col("name").rlike("^[A-Z]{2}\\d+"))  # regex: 2 capitals then digits
df.filter(F.col("name").startswith("C"))         # begins with "C"
df.filter(F.col("name").endswith("X"))           # ends with "X"
df.filter(F.col("name").contains("mug"))         # contains "mug" anywhere
```

### Taking a subset (limit and sampling)

```python
df.limit(100)                                    # first 100 rows - arbitrary WHICH 100
df.sample(fraction=0.1, seed=42)                 # roughly 10% of rows; seed makes it repeatable
df.sample(withReplacement=False, fraction=0.1, seed=42)            # no row picked twice
df.sampleBy("country", fractions={"UK": 0.5, "FR": 0.1}, seed=42)  # stratified: different % per group
```

> **`fraction` is approximate, not exact.** `sample(fraction=0.1)` on 1000 rows gives you *about* 100, not exactly 100. For an exact number use `df.limit(n)` after an `orderBy`.

---

## 7. Cleaning and preprocessing

### Nulls

```python
from pyspark.sql import functions as F

df.na.drop()                                     # DROP a row if ANY column is null
df.na.drop(how="all")                            # drop only if EVERY column is null
df.na.drop(subset=["customer_id", "qty"])        # drop if null in THESE columns only
df.na.drop(thresh=3)                             # drop rows with fewer than 3 non-null values

df.na.fill(0)                                    # replace null with 0 in every NUMERIC column
df.na.fill("unknown")                            # replace null with "unknown" in every STRING column
df.na.fill({"qty": 0, "country": "unknown"})     # a different fill value per column

df.na.replace(["N/A", "NULL", ""], None)         # turn these junk strings INTO real nulls
df.na.replace({"UK": "United Kingdom"}, subset=["country"])   # map values in one column

# fill with the column mean
mean_price = df.select(F.mean("price")).first()[0]   # .first() = one Row, [0] = its first field
df.na.fill({"price": mean_price})                    # substitute that number for nulls

# coalesce - returns the FIRST non-null of its arguments, left to right
df.withColumn("x", F.coalesce("a", "b", F.lit(0)))   # a, else b, else 0
```

### Deduplication

```python
from pyspark.sql import functions as F
from pyspark.sql import Window

df.distinct()                                    # remove rows identical across ALL columns
df.dropDuplicates()                              # exactly the same thing
df.dropDuplicates(["invoice_no", "product_id"])  # dedupe on a KEY - keeps an ARBITRARY row

# keep a SPECIFIC row per key - the pattern you actually want
w = Window.partitionBy("invoice_no", "product_id") \
          .orderBy(F.desc("updated_at"))         # group by key, newest first within each group
(df.withColumn("_rn", F.row_number().over(w))    # number the rows 1,2,3... inside each group
   .filter(F.col("_rn") == 1)                    # keep only #1 = the newest
   .drop("_rn"))                                 # remove the helper column
```

> **`dropDuplicates(keys)` keeps whichever row Spark happens to see first — that is not deterministic.** If it matters which row survives, use the window + `row_number` pattern.

### Casting

```python
from pyspark.sql import functions as F
from pyspark.sql import types as T

df.withColumn("qty", F.col("qty").cast("int"))                # string -> integer, by name
df.withColumn("qty", F.col("qty").cast(T.IntegerType()))      # same, using the type object
df.withColumn("amt", F.col("amt").cast(T.DecimalType(18, 2))) # -> exact decimal for money
df.withColumn("d",   F.to_date("date_str", "yyyy-MM-dd"))     # string -> DateType, given the format
df.withColumn("ts",  F.to_timestamp("ts_str", "yyyy-MM-dd HH:mm:ss"))   # string -> TimestampType

# cast every column listed
for c in ["a", "b", "c"]:                        # loop over the column names
    df = df.withColumn(c, F.col(c).cast("double"))   # reassign df each time - DataFrames are immutable
```

> **A failed cast produces `null`, silently.** Always count nulls after casting:
> `df.filter(F.col("qty").isNull()).count()`

### Trimming and standardising text

```python
from pyspark.sql import functions as F

df.withColumn("country", F.trim(F.col("country")))               # strip leading/trailing spaces
df.withColumn("country", F.initcap(F.trim(F.lower("country"))))  # trim, lowercase, Then Capitalise
df.withColumn("code", F.upper(F.trim("code")))                   # trim then UPPERCASE
df.withColumn("name", F.regexp_replace("name", r"\s+", " "))      # collapse runs of whitespace to one space
df.withColumn("digits", F.regexp_replace("phone", r"[^0-9]", "")) # delete every non-digit character
```

### Outliers

```python
from pyspark.sql import functions as F

# clip - pull extreme values back to the boundary instead of removing the row
df.withColumn("price", F.when(F.col("price") > upper, upper)      # too high -> upper bound
                        .when(F.col("price") < lower, lower)      # too low  -> lower bound
                        .otherwise(F.col("price")))               # otherwise leave it alone

# drop - remove the rows entirely
df.filter(F.col("price").between(lower, upper))                   # keep only what's in range

# flag rather than drop - usually better, because you keep the evidence
df.withColumn("is_outlier", (F.col("price") > upper) | (F.col("price") < lower))
```

### A realistic cleaning function

```python
from pyspark.sql import DataFrame
from pyspark.sql import functions as F

def clean_sales(df: DataFrame) -> DataFrame:                  # takes a DataFrame, returns a new one
    return (df
        .filter(~F.col("invoice_no").startswith("C"))         # drop cancellations (they start with C)
        .filter(F.col("quantity") > 0)                        # drop returns and zero-quantity rows
        .filter(F.col("unit_price") > 0)                      # drop free/broken price rows
        .filter(F.col("customer_id").isNotNull())             # we cannot attribute anonymous rows
        .withColumn("country", F.initcap(F.trim("country")))  # standardise " uk " -> "Uk"
        .withColumn("revenue", F.round(F.col("quantity") * F.col("unit_price"), 2))  # derive revenue
        .dropDuplicates(["invoice_no", "product_id", "customer_id"]))    # one row per line item
```

> **Keep cleaning as a pure `DataFrame -> DataFrame` function** with no `spark.read` or `.write` inside. That's what makes it testable on three rows ([[Testing and CI-CD]]).

---

## 8. String functions

### Joining strings together

```python
from pyspark.sql import functions as F

F.concat("a", "b")                     # glue columns together; NULL if ANY input is null
F.concat_ws("-", "a", "b", "c")        # glue with a separator; SKIPS nulls instead of poisoning
F.repeat("s", 3)                       # "ab" -> "ababab"
```

> **Prefer `concat_ws`.** With `concat`, one null anywhere makes the whole result null — a common way to silently lose a column.

### Cutting strings up

```python
from pyspark.sql import functions as F

F.substring("s", 1, 5)                 # 5 chars starting at position 1. 1-INDEXED, not 0
F.substring_index("s", "-", 2)         # everything before the 2nd "-" (negative counts from the right)
F.length("s")                          # number of characters
F.split("s", ",")                      # "a,b,c" -> ARRAY ["a","b","c"]
F.split("s", ",").getItem(0)           # first element of that array
F.reverse("s")                         # characters backwards
```

> ⚠️ **`substring` is 1-indexed**, unlike Python. `F.substring("s", 1, 3)` gives the first three characters.

### Case and whitespace

```python
from pyspark.sql import functions as F

F.upper("s") · F.lower("s") · F.initcap("s")     # UPPER / lower / Capitalise Each Word
F.trim("s") · F.ltrim("s") · F.rtrim("s")        # strip spaces: both ends / left / right
F.lpad("s", 10, "0") · F.rpad("s", 10, " ")      # pad to width 10 on the left / right
```

> **`lpad(col, 8, "0")` is how you restore leading zeros** to an ID that was wrongly stored as a number.

### Replacing

```python
from pyspark.sql import functions as F

F.translate("s", "abc", "xyz")         # character-by-character swap: a->x, b->y, c->z
F.overlay("s", "REPL", 3, 4)           # replace 4 chars starting at position 3 with "REPL"
F.regexp_replace("s", r"\d+", "N")     # replace EVERY run of digits with "N"
```

### Regular expressions

```python
from pyspark.sql import functions as F

F.regexp_replace("s", r"\s+", " ")     # collapse runs of whitespace into one space
F.regexp_extract("s", r"(\d+)", 1)     # pull out capture group 1; "" if no match
F.regexp_extract_all("s", r"(\d+)", 1) # every match, as an array
F.regexp_count("s", r"\d")             # how many matches
F.col("name").rlike(r"^[A-Z]{2}\d+")   # boolean: does it match? (use in a filter)
```

> ⚠️ **`regexp_extract` returns an empty string when nothing matches, not null.** Check with `== ""`, or wrap it: `F.nullif(F.regexp_extract(...), F.lit(""))`.

### Searching and fuzzy matching

```python
from pyspark.sql import functions as F

F.instr("s", "sub")                    # position of "sub" - 1-indexed, 0 if absent
F.locate("sub", "s", pos=1)            # same, but you can choose where to start looking
F.levenshtein("a", "b")                # edit distance - how many changes to turn a into b
F.soundex("s")                         # phonetic code - "Smith" and "Smyth" match
```

### Hashing

```python
from pyspark.sql import functions as F

F.md5("s") · F.sha2("s", 256) · F.crc32("s")     # deterministic fingerprints of a value
F.hash("a", "b") · F.xxhash64("a")               # fast non-cryptographic hashes, for bucketing
F.base64("s") · F.unbase64("s")                  # encode / decode bytes as text
```

> ⚠️ **These are for fingerprinting and bucketing, never for passwords.** Password hashing needs bcrypt or argon2 ([[Security in practice]]).

### Formatting numbers as text

```python
from pyspark.sql import functions as F

F.format_string("%s-%d", "a", "b")     # printf-style formatting
F.format_number(1234.5678, 2)          # "1,234.57" - thousands separators, returns a STRING
```

---

## 9. Date and time

### Now

```python
from pyspark.sql import functions as F

F.current_date() · F.current_timestamp()   # today / right now, evaluated on the CLUSTER
```

### Converting between strings and dates

```python
from pyspark.sql import functions as F

F.to_date("s", "yyyy-MM-dd")               # parse a STRING into DateType using this format
F.to_timestamp("s", "yyyy-MM-dd HH:mm:ss") # parse a string into TimestampType
F.date_format("d", "yyyy-MM")              # the opposite: a date -> a formatted STRING
F.make_date(2024, 1, 15)                   # build a date from separate year/month/day columns
```

> ⚠️ **A failed parse gives null, not an error.** If your format string doesn't match the data, the whole column silently becomes null. Count them straight after: `df.filter(F.col("d").isNull()).count()`.

> **Format letters:** `yyyy` year · `MM` month · `dd` day · `HH` hour (24h) · `mm` minute · `ss` second. Capital `MM` is month, lowercase `mm` is minute — mixing them up is the classic mistake.

### Pulling out parts of a date

```python
from pyspark.sql import functions as F

F.year("d") · F.month("d") · F.dayofmonth("d")   # pull out the parts as integers
F.dayofweek("d")        # 1 = SUNDAY, 7 = Saturday. Not Monday. This trips everyone up.
F.dayofyear("d") · F.weekofyear("d") · F.quarter("d")   # 1-366 / 1-53 / 1-4
F.hour("t") · F.minute("t") · F.second("t")             # time parts, needs a timestamp
```

> ⚠️ **`dayofweek` returns 1 for Sunday**, but pandas' `dayofweek` returns 0 for Monday. If you build a weekend flag in Spark and rebuild it in pandas at serving time, they will disagree ([[Feature engineering]]).

### Date arithmetic

```python
from pyspark.sql import functions as F

F.date_add("d", 7) · F.date_sub("d", 7)    # 7 days later / earlier
F.add_months("d", 3)                       # 3 months later; clamps 31 Jan + 1 month -> 28/29 Feb
F.datediff("end", "start")                 # whole DAYS between two dates (end minus start)
F.months_between("end", "start")           # months as a DOUBLE, so 1.5 is possible
F.last_day("d") · F.next_day("d", "Mon")   # last day of that month / the next Monday after d
F.date_diff("end", "start")                # Spark 3.5+ spelling of datediff
```

### Rounding a date down (truncating)

```python
from pyspark.sql import functions as F

F.trunc("d", "month")                      # round a DATE down: 2026-03-17 -> 2026-03-01
F.trunc("d", "year") · F.trunc("d", "week")
F.date_trunc("hour", "t")                  # round a TIMESTAMP down: 10:47:23 -> 10:00:00
F.date_trunc("day", "t") · F.date_trunc("minute", "t")
```

> **This is how you group by month or hour.** `groupBy(F.trunc("d", "month"))` gives you monthly totals.

### Epoch seconds

```python
from pyspark.sql import functions as F

F.unix_timestamp("s", "yyyy-MM-dd")        # string -> seconds since 1970-01-01 (epoch)
F.from_unixtime("epoch", "yyyy-MM-dd")     # epoch seconds -> formatted string
```

> ⚠️ **Epoch values are often in MILLIseconds** (JavaScript, Kafka, most APIs). `from_unixtime` expects **seconds** — divide by 1000 first, or you land in the year 55000.

### Time zones

```python
from pyspark.sql import functions as F

spark.conf.set("spark.sql.session.timeZone", "UTC")   # do this at the top of every job
F.to_utc_timestamp("t", "Europe/London")   # treat t as London time, convert TO UTC
F.from_utc_timestamp("t", "Europe/London") # treat t as UTC, convert to London time
```

> ⚠️ **`TimestampType` renders in the session time zone.** Two people running the same query in different zones see different dates, and rows land in different date partitions. Set the session zone to UTC explicitly and convert only for display.

### Generating a range of dates

```python
from pyspark.sql import functions as F

(spark.sql("SELECT sequence(to_date('2024-01-01'), to_date('2024-12-31'), interval 1 day) AS d")
      .select(F.explode("d").alias("date")))
```

> **Use this to build a date dimension**, then left-join your data onto it so days with no activity still appear as zero rather than vanishing.

---

## 10. Math and stats

### Rounding

```python
from pyspark.sql import functions as F

F.round("x", 2)                        # round to 2 decimal places, .5 rounds AWAY from zero
F.bround("x", 2)                       # banker's rounding: .5 goes to the nearest EVEN digit
F.ceil("x") · F.floor("x")             # always round up / always round down
F.abs("x") · F.signum("x")             # magnitude / -1, 0 or 1
```

### Powers, roots and logs

```python
from pyspark.sql import functions as F

F.sqrt("x") · F.pow("x", 2) · F.exp("x")   # square root / x squared / e^x
F.log("x") · F.log10("x") · F.log2("x")    # natural / base-10 / base-2
F.log1p("x")                               # log(1+x) - stays accurate when x is near 0
F.factorial("x") · F.hypot("a", "b")       # x! / sqrt(a²+b²) without overflowing
```

> ⚠️ **`F.log(0)` is `-Infinity` and `F.log(-1)` is null.** Guard it: `F.log(F.col("x") + 1)`, or use `log1p` ([[Numerical methods in practice]]).

### Trigonometry

```python
from pyspark.sql import functions as F

F.sin/cos/tan/asin/acos/atan/atan2/sinh/cosh/tanh   # all in RADIANS
F.degrees("x") · F.radians("x")                     # convert between degrees and radians
```

> **Cyclical encoding for ML uses these** — `sin` and `cos` of the hour so that 23:00 and 00:00 come out close together ([[Feature engineering]]).

### Comparing across columns in one row

```python
from pyspark.sql import functions as F

F.greatest("a", "b", "c")              # the largest of these columns, per row; skips nulls
F.least("a", "b", "c")                 # the smallest, per row
```

> **`F.greatest` is row-wise; `F.max` is column-wise.** `greatest("a","b")` compares two columns within one row. `F.max("a")` finds the biggest value down the whole column. Confusing these is common.

### Random numbers and row IDs

```python
from pyspark.sql import functions as F

F.rand(seed=42)                        # uniform 0-1; the seed makes it reproducible
F.randn(seed=42)                       # normal distribution, mean 0
F.monotonically_increasing_id()        # unique 64-bit id - increasing but NOT consecutive
F.spark_partition_id()                 # which partition this row is in - for debugging skew
```

> ⚠️ **`monotonically_increasing_id()` is not a row number.** It's unique and ascending, but it has gaps and depends on partitioning. For 1,2,3... use `F.row_number().over(window)` (§14).

> ⚠️ **Always pass a `seed` to `rand`** if you need the same split twice. Without one, re-running gives you different rows and your results won't reproduce.

### Arithmetic that won't blow up

```python
from pyspark.sql import functions as F

F.nanvl("x", 0)                        # replace NaN (not null) with 0
F.try_divide("a", "b")                 # returns null on divide-by-zero instead of erroring
F.nullif("b", F.lit(0))                # turn 0 into null so a / b gives null, not an error
```

---

## 11. Conditional logic

### CASE WHEN — if/else for columns

```python
from pyspark.sql import functions as F

# conditions are checked IN ORDER, first match wins
df.withColumn("band",
    F.when(F.col("revenue") > 1000, "high")      # if revenue > 1000 -> "high"
     .when(F.col("revenue") > 100,  "medium")    # else if > 100     -> "medium"
     .otherwise("low"))                          # else              -> "low"
```

> ⚠️ **Order matters.** Put the narrowest condition first. If `> 100` came first, nothing would ever reach `> 1000`.

```python
from pyspark.sql import functions as F

# without .otherwise() any unmatched row gets NULL, not an error
F.when(F.col("x") > 0, "pos")                    # x <= 0 and x IS NULL both give null
```

> **That null behaviour is useful on purpose** — it's what makes conditional aggregates work: `F.avg(F.when(cond, F.col("price")))` averages only the matching rows (§13).

### Boolean columns

```python
from pyspark.sql import functions as F

df.withColumn("is_big", F.col("revenue") > 1000)     # a comparison IS a boolean - no when() needed
df.withColumn("ok", (F.col("qty") > 0) & F.col("price").isNotNull())
```

### Handling nulls

```python
from pyspark.sql import functions as F

F.coalesce("a", "b", F.lit(0))     # first non-null of a, b, 0
F.ifnull("a", F.lit(0))            # if a is null use 0 - coalesce with exactly 2 arguments
F.nullif("a", F.lit(0))            # turn 0 INTO null - useful before dividing
F.isnull("a") · F.isnan("a")       # true if null / true if NaN - these are different things
```

### Mapping many values at once

```python
from pyspark.sql import functions as F

# a lookup instead of 50 chained .when() calls
mapping = {"UK": "Europe", "FR": "Europe", "US": "Americas"}
expr = F.create_map([F.lit(x) for kv in mapping.items() for x in kv])   # flatten to k1,v1,k2,v2,...
df.withColumn("region", expr[F.col("country")])  # look the country up; unknown keys -> null
```

> **Above roughly a dozen values, use a lookup table and a join instead.** It's easier to update and doesn't bake the mapping into your code.

### The same thing as SQL

```python
from pyspark.sql import functions as F

F.expr("CASE WHEN x > 0 THEN 'p' ELSE 'n' END")  # sometimes just clearer to read
```

---

## 12. Arrays, maps and JSON

### Arrays

```python
from pyspark.sql import functions as F

F.array("a", "b", "c")                 # build an array from three COLUMNS
F.array_contains("arr", "x")           # true if the array holds "x"
F.size("arr") · F.array_distinct("arr") · F.sort_array("arr")   # length (-1 if null) / dedupe / sort
F.array_union("a", "b") · F.array_intersect("a", "b") · F.array_except("a", "b")  # set operations
F.array_position("arr", "x")           # position of "x" - 1-indexed, 0 if absent
F.array_remove("arr", "x") · F.array_repeat("x", 3)   # drop every "x" / make ["x","x","x"]
F.slice("arr", 1, 3) · F.flatten("arr_of_arr")        # 3 elements from position 1 / nested -> flat
F.arrays_zip("a", "b") · F.array_join("arr", ",")     # pair up elementwise / array -> "a,b,c" string
F.array_max("arr") · F.array_min("arr")               # largest / smallest element
F.element_at("arr", 1)                 # element at position 1. 1-INDEXED; negative counts from the end
F.sequence(F.lit(1), F.lit(10))        # generate [1,2,3,...,10]

F.explode("arr")                       # ONE ROW PER ELEMENT - turns 1 row of 3 tags into 3 rows
F.explode_outer("arr")                 # same, but keeps rows whose array is empty/null (as null)
F.posexplode("arr")                    # same as explode, plus a "pos" column with the index

# higher-order functions - run a lambda on every element, WITHOUT a slow Python UDF
F.transform("arr", lambda x: x * 2)                  # map: double every element
F.filter("arr", lambda x: x > 0)                     # keep only elements passing the test
F.aggregate("arr", F.lit(0), lambda acc, x: acc + x) # reduce: start at 0, sum the elements
F.exists("arr", lambda x: x > 5)                     # true if ANY element matches
F.forall("arr", lambda x: x > 0)                     # true if EVERY element matches
F.zip_with("a", "b", lambda x, y: x + y)             # combine two arrays elementwise
```

> **`explode` silently drops rows with empty or null arrays.** Use `explode_outer` if you need to keep them — a classic silent row-loss bug.

### Maps and structs

```python
from pyspark.sql import functions as F

F.create_map(F.lit("k"), F.col("v"))             # build a map from alternating key, value
F.map_keys("m") · F.map_values("m") · F.map_entries("m")   # keys array / values array / (k,v) structs
F.map_from_arrays("keys", "vals") · F.map_concat("m1", "m2")   # zip 2 arrays into a map / merge maps
F.col("m")["key"] · F.element_at("m", "key")     # look up one key - both return null if absent

F.struct("a", "b").alias("s")                    # bundle columns a and b into one nested column "s"
F.col("s.field")                                 # reach INTO a struct with dot notation
df.select("s.*")                                 # flatten a struct back into top-level columns
```

### JSON

```python
from pyspark.sql import functions as F
from pyspark.sql import types as T

F.from_json("json_str", schema)        # parse a JSON STRING into a real struct, using your schema
F.to_json("struct_col")                # the reverse: struct -> JSON string
F.get_json_object("json_str", "$.field.sub")   # pull ONE value out by path, no schema needed
F.json_tuple("json_str", "a", "b")     # pull several top-level keys out at once
F.schema_of_json(F.lit(sample_json))   # infer a schema from one sample - use to WRITE the schema, not in prod

# parse a JSON column into real columns - the standard Kafka/event pattern
schema = T.StructType([T.StructField("temp", T.DoubleType())])   # declare what you expect
df.select(F.from_json("value", schema).alias("d")).select("d.*")  # parse, then flatten to columns
```

---

## 13. Aggregation

### One column at a time — the simple case

**"What is the average price?"** This is the thing you will do most often, so it comes first.

```python
from pyspark.sql import functions as F

df.select(F.avg("price")).show()          # average of ONE column, across the whole DataFrame
df.agg(F.avg("price")).show()             # identical - .agg() and .select() both work here
```

```
+-----------------+
|       avg(price)|
+-----------------+
|4.611113626083177|
+-----------------+
```

> **`show()` prints; it does not give you the number.** The result is a one-row DataFrame, not a float.

#### Getting it out as an actual Python number

```python
from pyspark.sql import functions as F

avg_price = df.select(F.avg("price")).first()[0]     # .first() = the Row, [0] = its first field
print(avg_price)                                     # 4.611113626083177 - a real Python float

avg_price = df.select(F.avg("price")).collect()[0][0]          # same thing, more typing
avg_price = df.select(F.avg("price").alias("a")).first()["a"]  # by name - clearer with several stats
```

> **This is the one people get stuck on.** `df.select(F.avg("price"))` is a DataFrame. You need `.first()[0]` to turn it into a number you can use in an `if`, an f-string, or a `na.fill()`.

#### Every statistic, for one column

```python
from pyspark.sql import functions as F

df.select(F.avg("price")).first()[0]        # mean
df.select(F.sum("price")).first()[0]        # total
df.select(F.min("price")).first()[0]        # smallest
df.select(F.max("price")).first()[0]        # largest
df.select(F.count("price")).first()[0]      # NON-NULL count for that column
df.select(F.countDistinct("price")).first()[0]          # how many different values
df.select(F.stddev("price")).first()[0]                 # standard deviation
df.select(F.variance("price")).first()[0]               # variance
df.select(F.percentile_approx("price", 0.5)).first()[0] # median
df.select(F.skewness("price")).first()[0]               # is the distribution lopsided?
df.select(F.kurtosis("price")).first()[0]               # how heavy are the tails?
```

| Function | Gives you |
|---|---|
| `F.avg` / `F.mean` | Mean — **the same function, two names** |
| `F.sum` | Total |
| `F.min` / `F.max` | Smallest / largest |
| `F.count("col")` | Rows where that column is **not null** |
| `F.count("*")` | **All** rows, nulls included |
| `F.countDistinct` | Unique values (exact, needs a shuffle) |
| `F.stddev` / `F.variance` | Spread |
| `F.percentile_approx(c, 0.5)` | Median (there is no `F.median` in older Spark) |
| `F.first` / `F.last` | First/last value — arbitrary unless ordered |

#### Several statistics on the same column, in one pass

```python
from pyspark.sql import functions as F

stats = df.select(
    F.avg("price").alias("mean"),          # name each one, or the columns come out as "avg(price)"
    F.min("price").alias("min"),
    F.max("price").alias("max"),
    F.stddev("price").alias("sd"),
    F.count("price").alias("n"),
).first()

print(stats["mean"], stats["max"])         # access by the alias you gave it
print(stats.asDict())                      # {'mean': 4.61, 'min': 0.0, ...}
```

> **Do it in one `select`, not five.** Each separate `.first()` triggers its own full scan of the data. One `select` with five functions reads the data once.

#### The built-in profile of one column

```python
df.select("price").describe().show()       # count, mean, stddev, min, max
df.describe("price").show()                # identical, shorter
df.describe("price", "quantity").show()    # two named columns
df.select("price").summary().show()        # the above PLUS 25%, 50%, 75% percentiles
df.summary("count", "mean", "50%", "max").show()   # choose exactly which stats
```

> **`describe()` with no arguments profiles every numeric column** — often far more than you wanted. **Naming the column is usually what you mean.**

#### The average of one column, per group

```python
from pyspark.sql import functions as F

df.groupBy("country").avg("price").show()                        # shortcut form
df.groupBy("country").agg(F.avg("price").alias("avg_price")).show()   # named - do this
df.groupBy("country").agg(F.avg("price")).orderBy(F.desc("avg(price)")).show()

df.groupBy("country").mean("price", "quantity").show()           # two columns, same function
df.groupBy("country").agg(F.avg("price"), F.sum("quantity")).show()   # different functions
```

> **Use `.agg()` with `.alias()`.** The shortcut `.avg("price")` names the output column `avg(price)`, with brackets in the name — awkward to reference later.

#### The average of only *some* rows

Two ways, and they mean different things:

```python
from pyspark.sql import functions as F

# 1 - filter the ROWS first, then average what's left
df.filter(F.col("country") == "UK").select(F.avg("price")).first()[0]

# 2 - conditional aggregate: average over the whole DataFrame, but only count matching rows
df.select(F.avg(F.when(F.col("country") == "UK", F.col("price"))).alias("uk_avg")).first()[0]
```

> **Use the `when()` form when you want several conditional measures side by side** — it reads the data once instead of once per filter:

```python
from pyspark.sql import functions as F

df.select(
    F.avg(F.when(F.col("country") == "UK", F.col("price"))).alias("uk_avg"),
    F.avg(F.when(F.col("country") == "FR", F.col("price"))).alias("fr_avg"),
    F.count(F.when(F.col("price") > 100, True)).alias("expensive_items"),
).show()
```

`F.when()` with no `.otherwise()` returns **null** for non-matching rows, and `avg` ignores nulls — which is exactly what makes this work.

#### ⚠️ The null trap

```python
from pyspark.sql import functions as F

df.select(F.avg("price")).first()[0]      # SUM of non-nulls / COUNT of non-nulls
df.select(F.sum("price") / F.count("*")).first()[0]   # SUM of non-nulls / ALL rows - DIFFERENT
```

> ⚠️ **`avg` ignores nulls entirely — it does not treat them as zero.** With 100 rows where 40 are null, `avg` divides by 60, not 100. If a missing price should count as £0, fill it first:

```python
from pyspark.sql import functions as F

df.na.fill({"price": 0}).select(F.avg("price")).first()[0]   # now nulls count as 0
```

> ⚠️ **`count("price")` and `count("*")` are different numbers.** The first skips nulls; the second doesn't. Comparing them is how you measure a column's null rate.

#### Rounding the result

```python
from pyspark.sql import functions as F

df.select(F.round(F.avg("price"), 2).alias("avg_price")).show()   # round INSIDE Spark
round(df.select(F.avg("price")).first()[0], 2)                    # or in Python afterwards
```

> ⚠️ **Never use Python's built-in `sum`, `min`, `max` or `round` on a Spark column** — they don't work, and the error is confusing. That's why the import is `functions as F`: `F.sum` is Spark's, `sum` is Python's.

---

### Grouping and aggregating

```python
from pyspark.sql import functions as F

df.groupBy("country").count()                    # rows per country - the simplest aggregate
df.groupBy("country", "segment").agg(            # group by TWO columns, then compute many measures
    F.sum("revenue").alias("total"),             # add up revenue in each group
    F.avg("revenue").alias("mean"),              # average; IGNORES nulls (doesn't count them)
    F.min("revenue"), F.max("revenue"),          # smallest and largest
    F.count("*").alias("n"),                     # rows in the group. count("col") skips nulls - different!
    F.countDistinct("customer_id").alias("customers"),   # unique customers - exact but needs a shuffle
    F.stddev("revenue"), F.variance("revenue"),  # spread of the values
    F.first("x"), F.last("x"),                   # first/last row in the group - ARBITRARY unless ordered
    F.collect_list("product"),                   # every value as an array (keeps duplicates)
    F.collect_set("product"),                    # every DISTINCT value as an array
    F.percentile_approx("revenue", 0.5).alias("median"),   # approximate median - exact is expensive
    F.sum(F.when(F.col("qty") > 10, 1).otherwise(0)).alias("big_orders"),  # count rows meeting a condition
)

df.agg(F.sum("revenue")).first()[0]              # aggregate the WHOLE DataFrame, pull out the number

# aggregate every numeric column at once
df.groupBy("k").agg({c: "sum" for c in numeric_cols})   # dict form: {column: function name}
df.groupBy("k").sum()                            # sum EVERY numeric column automatically
df.groupBy("k").mean() · .max() · .min() · .count()     # the same shortcut for other functions
```

**Filtering groups (HAVING):**
```python
from pyspark.sql import functions as F

# there is no HAVING in PySpark - you aggregate first, then filter the result
df.groupBy("country").agg(F.sum("revenue").alias("t")).filter(F.col("t") > 1000)
```

**Rollup, cube, grouping sets:**
```python
from pyspark.sql import functions as F

df.rollup("country", "city").agg(F.sum("revenue"))   # per city, per country subtotal, AND grand total
df.cube("country", "city").agg(F.sum("revenue"))     # every combination incl. city-only totals
df.groupBy(F.grouping_sets([["a"], ["b"], []])).agg(F.sum("x"))   # name exactly the groupings you want
# in the extra subtotal rows the grouped-out column is NULL - F.grouping_id() tells you which level
```

---

## 14. Window functions

**The most powerful feature in Spark SQL.** Compute across related rows **without collapsing them**.

```python
from pyspark.sql import Window

w = Window.partitionBy("customer_id").orderBy("date")
#              ^ split rows into groups      ^ order WITHIN each group
# partitionBy = "restart the calculation for each customer"; orderBy = "in this sequence"
```

### Ranking

```python
from pyspark.sql import functions as F

F.row_number().over(w)      # 1,2,3,4 - ALWAYS unique, ties broken arbitrarily
F.rank().over(w)            # ties share a rank, then SKIP:      1,1,3
F.dense_rank().over(w)      # ties share a rank, no skipping:    1,1,2
F.percent_rank().over(w)    # rank as 0.0-1.0 - where in the group this row sits
F.ntile(4).over(w)          # split the group into 4 equal buckets - quartiles
```

### Offset

```python
from pyspark.sql import functions as F

F.lag("revenue", 1).over(w)      # value from 1 row BACK - null on the group's first row
F.lag("revenue", 1, 0).over(w)   # same, but use 0 instead of null on the first row
F.lead("revenue", 1).over(w)     # value from 1 row FORWARD - null on the last row
F.first("x").over(w) · F.last("x").over(w)   # first / last value in the window frame
F.nth_value("x", 2).over(w)                  # the 2nd value in the frame

# the classic use: change since the previous period
F.col("revenue") - F.lag("revenue", 1).over(w)   # absolute change vs last time
```

### Aggregates over a window

```python
from pyspark.sql import functions as F
from pyspark.sql import Window

w_all  = Window.partitionBy("customer_id")                   # NO orderBy = the whole group
w_cum  = Window.partitionBy("customer_id").orderBy("date")   # WITH orderBy = start..current row
w_roll = (Window.partitionBy("customer_id").orderBy("date")
                .rowsBetween(-6, 0))                         # this row + the 6 before it

F.sum("revenue").over(w_all)      # that customer's TOTAL, repeated on every one of their rows
F.sum("revenue").over(w_cum)      # RUNNING total - grows down the group
F.avg("revenue").over(w_roll)     # 7-period MOVING average

# why w_all is useful: share of total, without a join back
F.col("revenue") / F.sum("revenue").over(w_all)   # this row as a fraction of the customer total
```

### Frames

```python
from pyspark.sql import Window

Window.unboundedPreceding · Window.currentRow · Window.unboundedFollowing
# = "the start of the group"  ·  "this row"  ·  "the end of the group"

.rowsBetween(-6, 0)                              # 6 rows back .. this row - counts ROWS
.rangeBetween(-7 * 86400, 0)                     # 7 days back .. now - counts VALUES (seconds here)
.rowsBetween(Window.unboundedPreceding, Window.currentRow)   # everything so far = a running total
```

> **`rowsBetween` counts rows; `rangeBetween` counts values.** For a true "last 7 days" window with missing days, you need `rangeBetween` over a numeric timestamp — `rowsBetween(-6,0)` gives you the last 7 *rows*, which is different.

### The top-N-per-group pattern

```python
from pyspark.sql import functions as F
from pyspark.sql import Window

w = Window.partitionBy("customer_id").orderBy(F.desc("revenue"))   # per customer, biggest first
(df.withColumn("rn", F.row_number().over(w))    # step 1: number the rows inside each customer
   .filter(F.col("rn") == 1)                    # step 2: keep #1 = that customer's biggest order
   .drop("rn"))                                 # step 3: throw the helper column away
# change to .filter(F.col("rn") <= 3) for the top 3 per customer
```

> **You cannot filter on a window function directly** — compute it in one step, filter in the next. Memorise this shape; you'll write it constantly.

---

## 15. Joins

### What a join is

**A join glues two tables together side by side, matching rows by a shared column.**

You have orders, and you have customers. The orders table only stores a `customer_id` — not the name. A join looks up each order's customer and brings their details across.

```python
orders = spark.createDataFrame([                 # small tables for the examples below
    (1, "C1", 50.0),
    (2, "C2", 30.0),
    (3, "C9", 10.0),          # C9 does NOT exist in customers
], ["order_id", "customer_id", "amount"])

customers = spark.createDataFrame([
    ("C1", "Alice", "UK"),
    ("C2", "Bob",   "FR"),
    ("C3", "Carol", "DE"),    # C3 has never ordered
], ["customer_id", "name", "country"])
```

```python
orders.join(customers, "customer_id").show()     # the join, on the shared column name
```

```
+-----------+--------+------+-----+-------+
|customer_id|order_id|amount| name|country|
+-----------+--------+------+-----+-------+
|         C1|       1|  50.0|Alice|     UK|
|         C2|       2|  30.0|  Bob|     FR|
+-----------+--------+------+-----+-------+
```

Order 3 vanished (C9 has no customer) and Carol vanished (no orders). **That's an inner join** — it keeps only rows that match on both sides.

### The three arguments

```python
left.join(right, on, how)
#          ^      ^   ^
#          |      |   └── the TYPE: "inner" (default), "left", "outer", "left_anti"...
#          |      └────── WHAT to match on: a column name, a list, or a condition
#          └───────────── the other DataFrame
```

### Choosing `on` — three forms

```python
a.join(b, "id")                    # shared column NAME -> ONE "id" column out. Best form, use this.
a.join(b, ["id", "date"])          # match on TWO columns - both must be equal
a.join(b, a.id == b.id)            # condition form -> TWO "id" columns in the output. Avoid.

# when the key is named differently on each side:
orders.join(customers, orders.customer_id == customers.cust_ref, "left")

# non-equality conditions are allowed, but slow (no hash join possible):
a.join(b, (a.id == b.id) & (a.ts.between(b.start, b.end)), "left")
```

> ⚠️ **Prefer `a.join(b, "id")` over `a.join(b, a.id == b.id)`.** The string form collapses the key into one output column. The condition form leaves **two columns both called `id`**, and any later `select("id")` fails with *Reference 'id' is ambiguous*.

**If you're stuck with duplicate names, prefix them before joining:**

```python
from pyspark.sql import functions as F

b2 = b.select([F.col(c).alias(f"b_{c}") for c in b.columns])   # rename every column
a.join(b2, a.id == b2.b_id, "left").drop("b_id")               # now nothing is ambiguous
```

### Choosing `how` — the join types

| Type | Keeps | Ask it when |
|---|---|---|
| **`"inner"`** *(default)* | Only rows matching on **both** sides | You need both halves to exist |
| **`"left"`** / `"left_outer"` | **Every** left row; right columns null if no match | **Enriching a table** — the usual choice |
| `"right"` | Every right row | Rare — just swap the tables and use left |
| `"outer"` / `"full"` | Everything from both sides | Reconciling two systems |
| **`"left_semi"`** | Left rows that **have** a match. **No right columns.** | Filtering, not enriching |
| **`"left_anti"`** | Left rows with **no** match | Finding what's missing |
| `"cross"` | Every row paired with every row | Almost never on purpose |

**The same example, each way:**

```python
orders.join(customers, "customer_id", "inner").count()   # 2 - only C1 and C2
orders.join(customers, "customer_id", "left").count()    # 3 - keeps order 3, name is NULL
orders.join(customers, "customer_id", "outer").count()   # 4 - keeps order 3 AND Carol
```

```python
orders.join(customers, "customer_id", "left").show()
```
```
+-----------+--------+------+-----+-------+
|customer_id|order_id|amount| name|country|
+-----------+--------+------+-----+-------+
|         C1|       1|  50.0|Alice|     UK|
|         C2|       2|  30.0|  Bob|     FR|
|         C9|       3|  10.0| NULL|   NULL|   <- kept, but nothing matched
+-----------+--------+------+-----+-------+
```

> **`left` is what you want most of the time.** You have a fact table and you're adding descriptive columns to it — you must not lose facts just because a lookup is missing.

> **A `left` join that produces nulls is telling you something.** Those rows are orphans: a bad ID, a late-arriving dimension, or a real data quality problem. Count them: `result.filter(F.col("name").isNull()).count()`.

### semi and anti — filtering with a join

These two are underused and very useful. They return **only the left table's columns**.

```python
# "which orders have a valid customer?" - a FILTER, not an enrichment
orders.join(customers, "customer_id", "left_semi").show()     # order 1 and 2. No name/country columns.

# "which customers have NEVER ordered?" - the missing ones
customers.join(orders, "customer_id", "left_anti").show()     # Carol

# "which orders point at a customer that doesn't exist?" - a data quality check
orders.join(customers, "customer_id", "left_anti").show()     # order 3 (C9)
```

> **`left_semi` cannot duplicate rows**, which is why it's the right tool for "does a match exist?". A plain inner join used as a filter will multiply your rows if the right side has duplicates — a silent, common bug.

### ⚠️ The mistake that silently corrupts your numbers

**If one left row matches three right rows, you get three output rows.** Your `SUM` is then triple-counted, and nothing warns you.

```python
before = orders.count()
after   = orders.join(customers, "customer_id").count()
print(before, after)
```

| Result | Meaning |
|---|---|
| `after == before` | Clean 1-to-1 join |
| `after > before` | **Duplicate keys on the right — your totals are now wrong** |
| `after < before` | Inner join dropped non-matching rows (use `left` if you wanted to keep them) |

**Check the right side is unique before you join:**

```python
from pyspark.sql import functions as F

(customers.groupBy("customer_id").count()
          .filter(F.col("count") > 1).show())      # returns nothing = the key is unique. Good.
```

**If it isn't unique, deduplicate deliberately — pick which row wins:**

```python
from pyspark.sql import functions as F
from pyspark.sql import Window

w = Window.partitionBy("customer_id").orderBy(F.desc("updated_at"))   # newest first
dim = (customers.withColumn("_rn", F.row_number().over(w))
                .filter(F.col("_rn") == 1).drop("_rn"))               # keep only the newest
orders.join(dim, "customer_id", "left")                               # now safe
```

> **Do this check every time you join.** It takes ten seconds and it catches the single most expensive class of data bug.

### Joining more than two tables

```python
result = (orders
    .join(customers, "customer_id", "left")      # add customer details
    .join(products,  "product_id",  "left")      # add product details
    .join(stores,    "store_id",    "left"))     # add store details
```

> **Check the row count after *each* join, not just at the end.** If the count jumps at step two, you know exactly which table has the duplicate key.

### Making joins fast

```python
from pyspark.sql import functions as F

spark.conf.get("spark.sql.autoBroadcastJoinThreshold")   # auto-broadcast below this (default 10MB)
orders.join(F.broadcast(customers), "customer_id")       # force it - the big win
```

**Why broadcasting matters:** normally Spark must *shuffle* — send every row across the network so that matching keys land on the same machine. That's the expensive part of a join. Broadcasting instead copies the **small** table to every machine, so no shuffle happens at all.

| Situation | What to do |
|---|---|
| Big fact × small dimension (< ~100 MB) | **`F.broadcast(small)`** |
| Both tables big | Shuffle is unavoidable; make sure the key isn't skewed (§20) |
| One key has far more rows than the rest | Salt it, or enable `adaptive.skewJoin` (§20) |
| Joining the same tables repeatedly | `.cache()` the built side |

```python
df.explain(mode="formatted")   # BroadcastHashJoin = good · SortMergeJoin = both sides shuffled
```

> **Filter before you join, not after.** Halving each side before the shuffle is far cheaper than joining everything and discarding the result.

### Stacking tables on top of each other (not side by side)

A join adds **columns**. A union adds **rows** — same shape, more data.

```python
jan.union(feb)                   # stack by POSITION - column ORDER must match exactly
jan.unionByName(feb)             # stack by column NAME - safer, use this
jan.unionByName(feb, allowMissingColumns=True)   # tolerate different columns; missing become null

jan.intersect(feb)               # rows present in BOTH (distinct)
jan.exceptAll(feb)               # rows in jan not in feb, keeping duplicates
jan.subtract(feb)                # same, but removes duplicates
```

> ⚠️ **`union` matches by position, not by name.** If one table has `(id, name)` and the other `(name, id)`, it will happily stack names into the id column with no error at all. **Use `unionByName`.**

```python
from functools import reduce                                  # stacking many at once
all_months = reduce(lambda a, b: a.unionByName(b), monthly_dfs)
```

### Joining a stream to a table

```python
from pyspark.sql import functions as F

stream.join(F.broadcast(dim), "device_id", "left")     # stream-static: fine, dim is re-read each batch
```

> **Stream-to-stream joins need a watermark on both sides**, or Spark buffers state forever and eventually dies. Stream-to-static, as above, is the easy and common case (§21).

---

## 16. Reshaping

### Pivot

```python
from pyspark.sql import functions as F

(df.groupBy("date")                # one output ROW per date
   .pivot("country")                # one output COLUMN per country value
   .agg(F.sum("revenue")))          # what goes in each cell

df.groupBy("date").pivot("country", ["UK","FR","DE"]).agg(F.sum("revenue"))
#                                   ^ naming the values skips a whole extra scan
```

> **Always pass the value list to `pivot`.** Without it Spark runs an extra job to find the distinct values first.

### Unpivot / melt

```python
from pyspark.sql import functions as F

df.unpivot(ids=["date"],                   # columns to KEEP as they are
           values=["uk","fr","de"],        # columns to fold DOWN into rows
           variableColumnName="country",   # new column holding the old column NAME
           valueColumnName="revenue")      # new column holding the old column VALUE

# older Spark - stack(n, name1, col1, name2, col2, ...) where n = number of pairs
F.expr("stack(3, 'uk', uk, 'fr', fr, 'de', de) as (country, revenue)")
```

### Sorting and ordering

```python
from pyspark.sql import functions as F

df.orderBy("a") · df.orderBy(F.desc("a")) · df.sort("a", F.desc("b"))  # ascending / descending / mixed
df.orderBy(F.col("a").asc_nulls_last())      # ascending, nulls pushed to the BOTTOM
df.orderBy(F.col("a").desc_nulls_first())    # descending, nulls at the TOP
df.sortWithinPartitions("a")                 # sort inside each partition only - no full shuffle, cheap

# orderBy on the whole DataFrame is a FULL SHUFFLE. Sorting 500M rows just to eyeball 20
# is a common and expensive mistake - use df.orderBy(...).limit(20) so Spark can optimise it.
```

---

## 17. UDFs

```python
from pyspark.sql import functions as F
from pyspark.sql import types as T

@F.udf(returnType=T.StringType())      # you MUST declare the return type - Spark can't infer it
def categorise(desc):                  # `desc` arrives as a plain Python value, one row at a time
    if desc is None: return None       # HANDLE NULLS - Spark passes them straight through
    return "mug" if "MUG" in desc.upper() else "other"

df.withColumn("cat", categorise("description"))    # call it like any other column function
```

> ⚠️ **Python UDFs are 10–100× slower than built-ins.** Every row is serialised to a Python process and back, and the optimiser cannot see inside — so predicate pushdown stops working.
>
> **Search `pyspark.sql.functions` before writing one.** The above is:
> ```python
> from pyspark.sql import functions as F
>
> F.when(F.upper("description").contains("MUG"), "mug").otherwise("other")
> ```

### Pandas UDFs — when you genuinely need Python

Vectorised via Arrow; far faster than row-at-a-time.

```python
from pyspark.sql import functions as F
from pyspark.sql import types as T
import pandas as pd

@F.pandas_udf(T.DoubleType())                    # declare the return type as before
def normalise(s: pd.Series) -> pd.Series:        # receives a whole pandas SERIES, not one value
    return (s - s.mean()) / s.std()              # vectorised - the entire batch at once

df.withColumn("z", normalise("price"))
# WARNING: s.mean() here is the mean of ONE BATCH, not the whole column. For a true
# column-wide z-score, compute the mean and stddev with F.mean/F.stddev first.
```

```python
import pandas as pd

# grouped map - your function receives ONE WHOLE GROUP as a pandas DataFrame
def fit_group(pdf: pd.DataFrame) -> pd.DataFrame:   # pdf = every row for a single id
    pdf["pred"] = pdf["x"] * 2                      # do anything pandas can do
    return pdf                                      # must match the declared schema exactly

df.groupBy("id").applyInPandas(fit_group, schema="id string, x double, pred double")
# ⚠️ each group must FIT IN ONE EXECUTOR'S MEMORY. A skewed group will OOM the job.
```

```python
spark.conf.set("spark.sql.execution.arrow.pyspark.enabled", "true")   # speeds up toPandas()
```

---

## 18. Writing data

### Writing a file or table

```python
(df.write
   .mode("overwrite")            # overwrite = DELETE what's there first | append | ignore | error
   .partitionBy("year", "month") # write into year=2026/month=03/ folders - lets readers skip files
   .option("compression", "snappy")   # snappy = fast; zstd/gzip = smaller but slower
   .parquet("path/"))            # writes a FOLDER of part files, not a single file

df.write.format("delta").mode("overwrite").save("path/")                  # Delta table by path
df.write.format("delta").mode("overwrite").saveAsTable("cat.schema.table")# registered in the catalog
df.write.option("header", True).csv("path/")     # CSV with a header row
df.write.json("path/")                            # one JSON object per line
```

### The four write modes

| Mode | Does |
|---|---|
| `"overwrite"` | **Deletes what's there**, then writes |
| `"append"` | Adds to what's there |
| `"ignore"` | Does nothing if the path exists |
| `"error"` *(default)* | Fails if the path exists |

> ⚠️ **`overwrite` deletes the whole target first.** Overwriting a partitioned table without `replaceWhere` destroys every partition, not just today's. That is a genuine data-loss incident and it has happened to everyone once.

### Re-runnable writes (idempotency)

```python
# replace ONE partition - THE idempotent pattern. Re-running the job cannot duplicate rows.
(df.write.format("delta").mode("overwrite")
   .option("replaceWhere", f"date = '{run_date}'")   # overwrite ONLY today's rows, leave the rest
   .save(path))
```

> **This is the single most useful thing in this section.** A job you can safely re-run is a job you can recover from. Without it, a retry after a half-finished run duplicates data.

### Letting the schema change

```python
df.write.format("delta").option("mergeSchema", "true").mode("append").save(path)
```

> ⚠️ **`mergeSchema` also silently accepts a typo'd column name as a brand new column.** Turn it on deliberately, not by default.

### Writing to a database (JDBC)

```python
(df.write.format("jdbc")
   .option("url", url).option("dbtable", "t")        # connection string and target table
   .option("user", u).option("password", p)          # use a secret store, never a literal
   .option("batchsize", 10000)                       # rows per round trip - default is 1000, far too slow
   .mode("overwrite").save())
```

### Controlling how many output files you get
```python
df.coalesce(1).write.parquet(path)     # ONE output file. No shuffle, but forces ALL data through
                                       # one task - fine for small results, fatal for large ones.
df.repartition(10).write.parquet(path) # exactly 10 files. Full shuffle, but evenly balanced.
df.repartition("country").write.partitionBy("country").parquet(path)
# repartition BY the same column you partitionBy = one file per country instead of one per task
```

> **Don't `partitionBy` a high-cardinality column.** `partitionBy("customer_id")` on 50,000 customers creates 50,000 directories of tiny files — the small-file problem at its worst. Partition on something with tens-to-hundreds of values.

### Delta operations

```python
from pyspark.sql import functions as F
from delta.tables import DeltaTable   # the Python API for merge/delete/update/history/vacuum

dt = DeltaTable.forPath(spark, path)   # get a handle on an existing Delta table
dt.toDF().show()                       # read it back as a normal DataFrame
dt.history().show()                    # EVERY version, who wrote it and what operation it was
dt.vacuum(168)                         # delete old files no longer referenced; 168 hours = 7 days
# ⚠️ vacuum destroys your ability to time-travel further back than the retention window.

# MERGE = "upsert": update the rows that exist, insert the ones that don't
(dt.alias("t").merge(updates.alias("s"), "t.id = s.id")   # t = target table, s = source of changes
   .whenMatchedUpdateAll()                                # id already there -> overwrite the row
   .whenNotMatchedInsertAll()                             # id is new       -> insert it
   .execute())

dt.delete(F.col("qty") < 0)            # delete matching rows - a real DELETE, tracked in history
dt.update(condition=F.col("c") == "UK", set={"c": F.lit("United Kingdom")})   # conditional update
dt.restoreToVersion(3)                 # roll the WHOLE TABLE back to version 3. Time travel.

spark.sql("OPTIMIZE delta.`path` ZORDER BY (customer_id)")
# OPTIMIZE compacts thousands of small files into a few big ones.
# ZORDER co-locates similar customer_id values so filters on it skip more files.
```

---

## 19. Spark SQL

### Running SQL against a DataFrame

```python
df.createOrReplaceTempView("sales")    # give this DataFrame a SQL-visible name (this session only)
spark.sql("SELECT country, SUM(revenue) FROM sales GROUP BY country").show()   # returns a DataFrame

result = spark.sql("""
    SELECT country, SUM(revenue) AS total
    FROM sales
    WHERE quantity > 0
    GROUP BY country
    HAVING SUM(revenue) > 1000
    ORDER BY total DESC
""")                                   # triple quotes for multi-line SQL
result.show()
```

> **The result of `spark.sql()` is an ordinary DataFrame** — you can carry on with `.filter()`, `.join()` and the rest. You can mix the two styles freely in one pipeline.

### Mixing SQL and DataFrame code

```python
from pyspark.sql import functions as F

df.createOrReplaceTempView("sales")
top = spark.sql("SELECT * FROM sales WHERE revenue > 100")   # SQL for the readable bit
top.groupBy("country").agg(F.avg("revenue")).show()          # DataFrame API for the rest
```

### Managing views and the catalog

```python
df.createOrReplaceTempView("sales")    # session-scoped view - disappears when the session ends
df.createOrReplaceGlobalTempView("g")  # visible to other sessions, addressed as global_temp.g

spark.catalog.listTables()             # what tables/views can I see right now?
spark.catalog.dropTempView("sales")    # remove the view
spark.catalog.tableExists("db.t")      # true/false - check before reading
spark.catalog.listColumns("db.t")      # column names and types without reading any data
spark.catalog.clearCache()             # drop everything cached in memory
```

### Expressions inside DataFrame code

```python
from pyspark.sql import functions as F

df.selectExpr("qty", "qty * price AS revenue")      # SQL strings in a select
df.filter("qty > 0 AND country = 'UK'")             # a SQL string as a filter
df.withColumn("band", F.expr("CASE WHEN revenue > 100 THEN 'high' ELSE 'low' END"))
```

> **Identical performance to the DataFrame API** — both go through the same Catalyst optimiser. Use whichever reads better for the task. Everything in [[SQL fundamentals]] works here.

---

## 20. Performance

### Partitions

```python
df.rdd.getNumPartitions()          # how many chunks is this DataFrame split into right now?
df.repartition(200)                # force exactly 200 partitions - FULL SHUFFLE, even sizes
df.repartition("country")          # shuffle so all rows for a country land together
df.repartition(50, "country")      # both: 50 partitions, grouped by country
df.coalesce(10)                    # REDUCE to 10 by merging neighbours - no shuffle, can be uneven
```

> **`repartition` shuffles; `coalesce` doesn't.** Use `coalesce` to reduce file count before writing; `repartition` to fix skew or increase parallelism. Target ~128 MB per partition, and 2–4× your core count.

### Caching

```python
from pyspark import StorageLevel

df.cache()                          # keep the result in memory, spilling to disk if needed
df.persist(StorageLevel.MEMORY_ONLY)# same idea, but you choose the storage level
df.unpersist()                      # release it - do this as soon as you're finished
spark.catalog.clearCache()          # release EVERYTHING cached

# caching is LAZY - nothing is stored until an action runs:
df.cache()
df.count()                          # this is the call that actually fills the cache
```

> **Cache only a DataFrame used by 2+ actions that was expensive to build.** Caching everything evicts what you actually needed. Always `unpersist()` when done.

### Key configs

```python
spark.conf.set("spark.sql.shuffle.partitions", "200")   # partitions AFTER a shuffle. Default 200 -
                                                        # far too many on a laptop, set 4-8 locally.
spark.conf.set("spark.sql.adaptive.enabled", "true")    # AQE: re-plan mid-job using real row counts
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")  # auto-merge tiny partitions
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")           # auto-split skewed join keys
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 10*1024*1024)    # broadcast tables under 10MB
spark.conf.set("spark.sql.files.maxPartitionBytes", 128*1024*1024)      # target ~128MB read per task
spark.conf.set("spark.sql.execution.arrow.pyspark.enabled", "true")     # fast toPandas() and pandas UDFs
```

### Handling skew

```python
from pyspark.sql import functions as F

# Skew = one key has far more rows than the rest, so ONE task does most of the work
# and everyone waits for it. Spot it with: df.groupBy(key).count().orderBy(F.desc("count")).show()

# 1 - broadcast the small side. No shuffle at all, so skew stops mattering. TRY THIS FIRST.
big.join(F.broadcast(small), "id")

# 2 - salt the hot key: split one huge key into 10 smaller ones
salted = big.withColumn("salt", (F.rand() * 10).cast("int"))     # tag each big row 0-9 at random
small_x = small.crossJoin(spark.range(10)
                               .withColumnRenamed("id", "salt")) # duplicate the small side 10x, once per salt
salted.join(small_x, ["id", "salt"])                             # now the work spreads over 10 tasks

# 3 - let AQE detect and split skewed partitions for you
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
```

### Diagnosing

| Symptom | Cause | Fix |
|---|---|---|
| One task 50× slower than the median | **Skew** | Broadcast, salt, or AQE |
| Job runs 3× | Multiple actions recomputing | `.cache()` |
| `OutOfMemoryError` on the driver | `.collect()` / `.toPandas()` | `show()`, `take()`, `limit()` |
| Thousands of tiny output files | Too many partitions | `coalesce()` before write |
| `BroadcastNestedLoopJoin` in the plan | Non-equality join condition | Rewrite as an equi-join |
| Filter reads everything | Not partitioned on that column | `partitionBy`, or Z-order |

Spark UI at **:4040** — check **Stages** (each `Exchange` is a shuffle), then **Tasks** (max vs median duration = skew), then the **SQL tab** (the plan with real row counts).

---

## 21. Structured Streaming

**A stream is a DataFrame that never stops growing.** You write the same transformations; Spark re-runs them on each new batch and keeps track of what it has already seen.

### Reading a stream

```python
stream = (spark.readStream                        # readStream, not read - this never finishes
    .format("kafka")
    .option("kafka.bootstrap.servers", "localhost:9092")   # where the broker lives
    .option("subscribe", "telemetry")                      # which topic to consume
    .option("startingOffsets", "latest")   # "latest" = only new messages; "earliest" = the whole topic
    .load())                               # Kafka gives you key/value/topic/partition/offset as BYTES
```

### Parsing and windowing it

```python
from pyspark.sql import functions as F

parsed = (stream
    .select(F.from_json(F.col("value").cast("string"), schema).alias("d"))  # bytes -> string -> struct
    .select("d.*")                                # flatten the struct into real columns
    .withWatermark("ts", "10 minutes")            # accept data up to 10 min late, then stop waiting
    .groupBy(F.window("ts", "5 minutes"), "device_id")   # 5-minute tumbling buckets, per device
    .agg(F.avg("temp")))                          # average temperature in each bucket
```

### Writing it out

```python
query = (parsed.writeStream
    .format("delta")
    .outputMode("append")     # append = only new rows | update = changed rows | complete = whole result
    .option("checkpointLocation", "s3a://bronze/_checkpoints/telemetry")   # ESSENTIAL - see below
    .trigger(processingTime="1 minute")   # run a micro-batch every minute; availableNow=True = catch up once and stop
    .start("s3a://bronze/telemetry/"))    # start() launches it in the background

query.awaitTermination()                  # block here so the program doesn't exit
query.status · query.lastProgress · query.stop()   # what is it doing / throughput and lag / shut it down
```

### Sources and sinks

Other sources/sinks: `.format("delta")`, `.format("rate")` (generates test data), `.foreachBatch(fn)` to write anywhere Spark has no built-in sink for.

### Checkpoints

> **`checkpointLocation` is how the stream remembers its offsets.** Delete it and you reprocess everything from the beginning. Back it up; never delete it casually. See [[Kafka]].

---

## 22. MLlib

Spark's own ML library — distributed, and separate from [[10 — PYTORCH|PyTorch]].

### The imports

```python
from pyspark.ml import Pipeline                  # chains the stages so they fit/apply together
from pyspark.ml.feature import VectorAssembler, StringIndexer, OneHotEncoder, StandardScaler
                                                  # the preprocessing steps - see Feature engineering
from pyspark.ml.regression import LinearRegression, GBTRegressor          # predict a NUMBER
from pyspark.ml.classification import LogisticRegression, RandomForestClassifier   # predict a CLASS
from pyspark.ml.evaluation import RegressionEvaluator     # turns predictions into a score
from pyspark.ml.tuning import CrossValidator, ParamGridBuilder   # hyperparameter search
```

### Building a pipeline

```python
from pyspark.ml import Pipeline
from pyspark.ml.feature import OneHotEncoder, StandardScaler, StringIndexer, VectorAssembler
from pyspark.ml.regression import GBTRegressor

pipeline = Pipeline(stages=[      # stages run IN ORDER, each feeding the next
    StringIndexer(inputCol="country", outputCol="country_idx", handleInvalid="keep"),
        # text category -> a number. handleInvalid="keep" stops unseen values crashing at predict time
    OneHotEncoder(inputCols=["country_idx"], outputCols=["country_vec"]),
        # number -> a 0/1 vector, so the model doesn't think country 3 > country 1
    VectorAssembler(inputCols=["qty", "price", "country_vec"], outputCol="features_raw"),
        # glue every input column into ONE vector column - MLlib requires this
    StandardScaler(inputCol="features_raw", outputCol="features"),
        # rescale to mean 0, stddev 1 so no feature dominates by unit alone
    GBTRegressor(featuresCol="features", labelCol="revenue"),
        # the model itself: features in, revenue out
])
```

### Training, predicting and scoring

```python
from pyspark.ml.evaluation import RegressionEvaluator

train, test = df.randomSplit([0.8, 0.2], seed=42)   # 80/20 split; seed makes it repeatable
model = pipeline.fit(train)        # LEARN from the training set only - every stage is fitted here
preds = model.transform(test)      # APPLY to unseen data - adds a "prediction" column

RegressionEvaluator(labelCol="revenue", metricName="rmse").evaluate(preds)   # one score

model.write().overwrite().save("s3a://models/gbt")   # saves the WHOLE pipeline, encoders included
```

> **`VectorAssembler` is mandatory** — every MLlib estimator expects a single vector column called `features`. That surprises everyone once.

> **Use MLlib when the data genuinely doesn't fit on one machine.** Otherwise scikit-learn or LightGBM on a sample is faster to iterate and usually just as good ([[Choosing your approach]]).

---

## Related

Building ML features with these window functions: **[[Feature engineering]]**

Saving DataFrames to local S3 buckets: **[[Using SeaweedFS]]**


[[PySpark core]] — the concepts and why lazy evaluation matters
[[Databricks and Delta Lake]] · [[SQL fundamentals]] · [[Data modeling]] · [[Kafka]] · [[When to leave Python]] · [[Testing and CI-CD]] · [[07 — DATA ENGINEERING]]
