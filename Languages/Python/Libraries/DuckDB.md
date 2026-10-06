---
tags: [python, library, duckdb, sql, analytics]
---

# DuckDB

**SQL over local files.** An analytics database that runs inside your process — no server, no setup.

Index: [[Python libraries]] · SQL: [[SQL fundamentals]]

### Installing

> **Same commands on Windows, WSL and Linux.** The differences are called out below where they exist.

```bash
uv add duckdb                      # in a uv project - PREFERRED
uv pip install duckdb              # into the active venv, no project file
pip install duckdb                 # plain pip, if you're not using uv
```

**If you don't have a project yet:**

```bash
uv init myproject && cd myproject     # creates pyproject.toml and .venv
uv add duckdb
uv run python -c "import duckdb; print(duckdb.__version__)"           # uv run uses the project venv automatically
```

**Without uv — a plain virtual environment:**

| | Windows (PowerShell) | WSL / Linux |
|---|---|---|
| Create | `python -m venv .venv` | `python3 -m venv .venv` |
| Activate | `.venv\Scripts\Activate.ps1` | `source .venv/bin/activate` |
| Install | `pip install duckdb` | `pip install duckdb` |
| Leave | `deactivate` | `deactivate` |

**Check it worked:**

```bash
python -c "import duckdb; print(duckdb.__version__)"
```

> **No server, no service, no configuration.** DuckDB is a single library — this is the whole installation.

```bash
winget install DuckDB.cli          # Windows: the standalone CLI, optional
curl https://install.duckdb.org | sh   # WSL / Linux CLI
```

> ⚠️ **Never `pip install` outside a virtual environment.** On WSL and Linux it fights the system Python; newer versions refuse outright with *externally-managed-environment*. On Windows it silently pollutes your global install. `uv` makes a venv for you — use it.

> **On Windows: if `python` opens the Microsoft Store**, Python isn't really installed. Get it from python.org (tick *Add to PATH*), or `winget install Python.Python.3.12`.

> **In WSL: work in `~/`, not `/mnt/c/`.** Installing into a project on the Windows drive is dramatically slower, and file-watching (`--reload`, `pytest-watch`) often doesn't fire at all. See [[Setting up a dev machine]].

---

## The idea

SQLite is a transactional database in a file. **DuckDB is an *analytical* database in a file** — columnar, vectorised, built for `GROUP BY` over millions of rows.

```python
import duckdb

duckdb.sql("SELECT * FROM 'sales.parquet' WHERE qty > 0").df()
```

> **You just queried a Parquet file with SQL, with no database, no schema, no loading step.** That's the entire pitch.

## Querying files directly

```python
import duckdb

duckdb.sql("SELECT * FROM 'data.csv'")
duckdb.sql("SELECT * FROM 'data/*.parquet'")            # globs
duckdb.sql("SELECT * FROM read_csv_auto('f.csv')")
duckdb.sql("SELECT * FROM read_parquet('s3://bucket/*.parquet')")
duckdb.sql("SELECT * FROM read_json_auto('f.jsonl')")
```

## Querying DataFrames

```python
import duckdb
import pandas as pd, polars as pl
df = pd.read_csv("sales.csv")

duckdb.sql("SELECT country, SUM(revenue) FROM df GROUP BY country").df()
```

> **It finds `df` in your Python scope automatically.** No registration, no copying — it reads the DataFrame's memory directly. Works with pandas, [[Polars]] and Arrow.

## Output formats

```python
import duckdb

r = duckdb.sql("SELECT 1 AS x")
r.df()          # pandas
r.pl()          # polars
r.arrow()       # Arrow table
r.fetchall()    # list of tuples
r.fetchone()
r.show()
```

## Persistent databases

```python
import duckdb

con = duckdb.connect("analytics.db")
con.sql("CREATE TABLE sales AS SELECT * FROM 'sales.parquet'")
con.sql("CREATE VIEW gold AS SELECT country, SUM(revenue) r FROM sales GROUP BY country")
con.sql("SELECT * FROM gold").df()
con.close()

with duckdb.connect("analytics.db") as con:      # context manager
    ...
```

## Writing

```python
import duckdb

duckdb.sql("COPY (SELECT * FROM df) TO 'out.parquet' (FORMAT PARQUET)")
duckdb.sql("COPY (SELECT * FROM df) TO 'out.csv' (HEADER, DELIMITER ',')")
duckdb.sql("COPY (SELECT * FROM df) TO 'out' (FORMAT PARQUET, PARTITION_BY (year))")
```

## Parameters — never f-string

```python
con.execute("SELECT * FROM sales WHERE country = ?", ["UK"]).df()
con.execute("SELECT * FROM sales WHERE d > $d", {"d": start}).df()
```

> **Same rule as everywhere: parameters, not string interpolation** ([[SQL fundamentals]]).

## Cloud storage

```python
import duckdb

duckdb.sql("INSTALL httpfs; LOAD httpfs;")
duckdb.sql("SET s3_region='eu-west-2'; SET s3_access_key_id='...'; SET s3_secret_access_key='...'")
duckdb.sql("SELECT * FROM read_parquet('s3://bucket/data/*.parquet')")
```

Works against S3, [[SeaweedFS]], Azure and GCS.

## Useful extensions

```sql
INSTALL httpfs;   LOAD httpfs;      -- S3/HTTP
INSTALL delta;    LOAD delta;       -- read Delta Lake tables
INSTALL spatial;  LOAD spatial;     -- geospatial
INSTALL json;     LOAD json;
```

## Analytical SQL it does well

```sql
SELECT country,
       SUM(revenue)                                        AS total,
       SUM(revenue) OVER (PARTITION BY country)            AS country_total,
       revenue - LAG(revenue) OVER (ORDER BY date)         AS change,
       PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY revenue) AS median
FROM sales
QUALIFY ROW_NUMBER() OVER (PARTITION BY country ORDER BY revenue DESC) <= 3
```

> **`QUALIFY` filters on a window function without a subquery** — a genuinely nice extension that Postgres doesn't have ([[SQL fundamentals]]).

## When to use it

| Use DuckDB | Use something else |
|---|---|
| Analytics over local Parquet/CSV | Transactional app -> [[PostgreSQL reference]] |
| You'd rather write SQL than pandas | Genuinely distributed -> [[PySpark reference]] |
| Ad-hoc exploration of files | Concurrent multi-user writes |
| Replacing a slow pandas pipeline | |

> **DuckDB and [[Polars]] solve the same problem** — fast single-machine analytics. Pick by whether you'd rather write SQL or method chains. They interoperate freely via Arrow.

## Related

[[Python libraries]] · [[SQL fundamentals]] · [[Polars]] · [[pandas]] · [[PostgreSQL reference]] · [[06 — DATABASES]]
