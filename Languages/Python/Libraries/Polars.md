---
tags: [python, library, polars, data, rust]
---

# Polars

DataFrames, written in Rust. **5-30× faster than pandas**, and the API is stricter.

Index: [[Python libraries]] · Compare: [[pandas]] · [[PySpark reference]]

### Installing

> **Same commands on Windows, WSL and Linux.** The differences are called out below where they exist.

```bash
uv add polars                      # in a uv project - PREFERRED
uv pip install polars              # into the active venv, no project file
pip install polars                 # plain pip, if you're not using uv
```

**If you don't have a project yet:**

```bash
uv init myproject && cd myproject     # creates pyproject.toml and .venv
uv add polars
uv run python -c "import polars; print(polars.__version__)"           # uv run uses the project venv automatically
```

**Without uv — a plain virtual environment:**

| | Windows (PowerShell) | WSL / Linux |
|---|---|---|
| Create | `python -m venv .venv` | `python3 -m venv .venv` |
| Activate | `.venv\Scripts\Activate.ps1` | `source .venv/bin/activate` |
| Install | `pip install polars` | `pip install polars` |
| Leave | `deactivate` | `deactivate` |

**Check it worked:**

```bash
python -c "import polars; print(polars.__version__)"
```

> ⚠️ **On an older CPU, `import polars` crashes with an illegal instruction.** The default build needs AVX2. Install `polars-lts-cpu` instead:
> ```bash
> uv add polars-lts-cpu        # same API, works on pre-2015 CPUs
> ```
> This bites most often inside older VMs and CI runners.

> ⚠️ **Never `pip install` outside a virtual environment.** On WSL and Linux it fights the system Python; newer versions refuse outright with *externally-managed-environment*. On Windows it silently pollutes your global install. `uv` makes a venv for you — use it.

> **On Windows: if `python` opens the Microsoft Store**, Python isn't really installed. Get it from python.org (tick *Add to PATH*), or `winget install Python.Python.3.12`.

> **In WSL: work in `~/`, not `/mnt/c/`.** Installing into a project on the Windows drive is dramatically slower, and file-watching (`--reload`, `pytest-watch`) often doesn't fire at all. See [[Setting up a dev machine]].

```python
import polars as pl
```

---

## Why it's faster

| | pandas | Polars |
|---|---|---|
| Written in | C + Python | **Rust** |
| Threads | Mostly single | **All cores by default** |
| Memory format | NumPy | **Apache Arrow** |
| Evaluation | Eager | **Lazy — it optimises your query** |
| Nulls | NaN muddle | Proper null type |

## Eager vs lazy — the important bit

```python
import polars as pl

df = pl.read_csv("f.csv")                 # EAGER - reads now
lf = pl.scan_csv("f.csv")                 # LAZY - reads nothing yet

result = (lf
    .filter(pl.col("qty") > 0)
    .with_columns((pl.col("qty") * pl.col("price")).alias("revenue"))
    .group_by("product")
    .agg(pl.col("revenue").sum())
    .collect())                           # <- NOW it runs, optimised
```

> **Use `scan_*` and `.collect()` for anything non-trivial.** Polars sees the whole query and pushes filters into the file reader, prunes columns, and parallelises. Same idea as [[PySpark core]] — build a plan, let the engine optimise it.

```python
lf.explain()                              # see the optimised plan
```

## Reading and writing

```python
import polars as pl

pl.read_csv("f.csv", separator=",", has_header=True, null_values=["NA"],
            schema_overrides={"id": pl.Utf8}, try_parse_dates=True)
pl.read_parquet("f.parquet", columns=["a","b"])
pl.read_json("f.json") · pl.read_ndjson("f.jsonl")
pl.read_database("SELECT * FROM t", connection)

pl.scan_csv("*.csv") · pl.scan_parquet("s3://bucket/*.parquet")

df.write_csv("f.csv") · df.write_parquet("f.parquet") · df.write_ndjson("f.jsonl")
df.to_pandas() · pl.from_pandas(pdf)      # zero-copy via Arrow where possible
```

## Investigating

```python
import polars as pl

df.shape · df.columns · df.dtypes · df.schema
df.head(10) · df.tail() · df.sample(5)
df.describe()
df.glimpse()                              # transposed preview - great for wide data
df.estimated_size("mb")

df.null_count()                           # nulls per column
df.n_unique() · df["country"].value_counts()
df.select(pl.all().n_unique())
df.is_duplicated().sum()
```

## Expressions — the core idea

Everything is an **expression** applied inside a context.

```python
import polars as pl

df.select(pl.col("a"), pl.col("b") * 2)
df.with_columns((pl.col("qty") * pl.col("price")).alias("revenue"))
df.filter(pl.col("qty") > 0)
df.group_by("country").agg(pl.col("revenue").sum())
```

```python
import polars as pl

pl.col("a") · pl.col("^s\\d+$")            # regex column selection
pl.all() · pl.exclude("id")
pl.lit(1) · pl.first() · pl.last() · pl.len()

# apply to many columns at once
df.with_columns(pl.col(pl.Float64).round(2))
df.with_columns(pl.all().exclude("id").fill_null(0))
```

> **The killer feature: expressions are reusable objects.**
> ```python
> import polars as pl
>
> revenue = (pl.col("qty") * pl.col("price")).alias("revenue")
> df.with_columns(revenue)          # use the same expression anywhere
> ```

## Filtering

```python
import polars as pl

df.filter(pl.col("qty") > 0)
df.filter((pl.col("qty") > 0) & (pl.col("country") == "UK"))
df.filter(pl.col("country").is_in(["UK","FR"]))
df.filter(pl.col("a").is_null()) · df.filter(pl.col("a").is_not_null())
df.filter(pl.col("name").str.contains("mug"))
df.filter(pl.col("price").is_between(10, 100))
```

## Cleaning

```python
import polars as pl

df.drop_nulls() · df.drop_nulls(subset=["a"])
df.fill_null(0) · df.fill_null(strategy="forward")
df.with_columns(pl.col("a").fill_null(pl.col("a").mean()))
df.unique() · df.unique(subset=["id"], keep="last")

df.with_columns(pl.col("qty").cast(pl.Int64, strict=False))   # strict=False -> null on failure
df.with_columns(pl.col("country").str.strip_chars().str.to_titlecase())
df.rename({"old":"new"}) · df.drop("a")
```

## Aggregation

```python
import polars as pl

df.group_by("country").agg(
    pl.col("revenue").sum().alias("total"),
    pl.col("revenue").mean().alias("avg"),
    pl.len().alias("n"),
    pl.col("customer_id").n_unique().alias("customers"),
    pl.col("revenue").filter(pl.col("qty") > 10).sum().alias("big"),   # conditional agg
)

df.group_by(["a","b"]).agg(pl.all().sum())
df.group_by_dynamic("date", every="1d").agg(pl.col("revenue").sum())   # time buckets
```

> **Conditional aggregation is far cleaner than pandas** — `.filter()` inside `.agg()` needs no separate mask column.

## Window functions

```python
import polars as pl

df.with_columns(
    pl.col("revenue").sum().over("customer_id").alias("customer_total"),
    pl.col("revenue").rank(descending=True).over("customer_id").alias("rank"),
    pl.col("revenue").shift(1).over("customer_id").alias("prev"),
    pl.col("revenue").rolling_mean(7).over("customer_id").alias("ma7"),
    pl.col("revenue").cum_sum().over("customer_id"),
)
```

> `.over()` is the equivalent of SQL `PARTITION BY` and pandas `groupby().transform()` ([[SQL fundamentals]]).

## Joins and reshaping

```python
import polars as pl

a.join(b, on="id", how="inner")           # inner|left|outer|semi|anti|cross
a.join(b, left_on="a_id", right_on="b_id")
a.join_asof(b, on="time", by="id")        # time-series join
pl.concat([a, b]) · pl.concat([a, b], how="horizontal")

df.pivot(index="date", on="country", values="revenue", aggregate_function="sum")
df.unpivot(index="date", on=["uk","fr"], variable_name="country", value_name="revenue")
df.explode("tags") · df.transpose()
```

## String, date, list namespaces

```python
import polars as pl

pl.col("s").str.to_uppercase() · .str.strip_chars() · .str.contains("x")
pl.col("s").str.replace_all(r"\\d", "") · .str.extract(r"(\\d+)", 1)
pl.col("s").str.split(",") · .str.len_chars() · .str.slice(0, 5)

pl.col("d").dt.year() · .dt.month() · .dt.weekday() · .dt.strftime("%Y-%m")
pl.col("d").dt.offset_by("7d") · .dt.truncate("1mo")

pl.col("arr").list.len() · .list.first() · .list.sum() · .list.contains("x")
```

## Conditionals

```python
import polars as pl

df.with_columns(
    pl.when(pl.col("revenue") > 1000).then(pl.lit("high"))
      .when(pl.col("revenue") > 100).then(pl.lit("medium"))
      .otherwise(pl.lit("low")).alias("band")
)
```

## pandas → Polars

| pandas | Polars |
|---|---|
| `df[df.a > 0]` | `df.filter(pl.col("a") > 0)` |
| `df["b"] = df.a * 2` | `df.with_columns((pl.col("a")*2).alias("b"))` |
| `df.groupby("k").sum()` | `df.group_by("k").agg(pl.all().sum())` |
| `df.merge(o, on="id")` | `df.join(o, on="id")` |
| `df.rename(columns={...})` | `df.rename({...})` |
| `df.groupby("k")["v"].transform("sum")` | `pl.col("v").sum().over("k")` |
| `len(df)` | `df.height` |
| `df.apply(...)` | **Use an expression** |

> **There is no index in Polars.** No `set_index`, no `reset_index`, no `.loc`. That removes an entire category of pandas confusion.

## When to use which

| Use | When |
|---|---|
| **pandas** | Small data, huge ecosystem, existing code |
| **Polars** | Anything slow in pandas; new code; multi-GB files |
| **DuckDB** | You'd rather write SQL ([[DuckDB]]) |
| **PySpark** | Genuinely bigger than one machine ([[PySpark reference]]) |

## Related

[[Python libraries]] · [[pandas]] · [[DuckDB]] · [[PySpark reference]] · [[Rust]] · [[When to leave Python]]
