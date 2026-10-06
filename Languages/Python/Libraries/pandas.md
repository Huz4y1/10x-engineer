---
tags: [python, library, pandas, data]
---

# pandas

DataFrames in memory. Index: [[Python libraries]] · Bigger data: [[Polars]] · [[PySpark reference]]

### Installing

> **Same commands on Windows, WSL and Linux.** The differences are called out below where they exist.

```bash
uv add pandas pyarrow                      # in a uv project - PREFERRED
uv pip install pandas pyarrow              # into the active venv, no project file
pip install pandas pyarrow                 # plain pip, if you're not using uv
```

**If you don't have a project yet:**

```bash
uv init myproject && cd myproject     # creates pyproject.toml and .venv
uv add pandas pyarrow
uv run python -c "import pandas; print(pandas.__version__)"           # uv run uses the project venv automatically
```

**Without uv — a plain virtual environment:**

| | Windows (PowerShell) | WSL / Linux |
|---|---|---|
| Create | `python -m venv .venv` | `python3 -m venv .venv` |
| Activate | `.venv\Scripts\Activate.ps1` | `source .venv/bin/activate` |
| Install | `pip install pandas pyarrow` | `pip install pandas pyarrow` |
| Leave | `deactivate` | `deactivate` |

**Check it worked:**

```bash
python -c "import pandas; print(pandas.__version__)"
```

> **Always install `pyarrow` alongside pandas.** It's what makes Parquet, and the string dtype that saves large amounts of memory, actually work.

> ⚠️ **Never `pip install` outside a virtual environment.** On WSL and Linux it fights the system Python; newer versions refuse outright with *externally-managed-environment*. On Windows it silently pollutes your global install. `uv` makes a venv for you — use it.

> **On Windows: if `python` opens the Microsoft Store**, Python isn't really installed. Get it from python.org (tick *Add to PATH*), or `winget install Python.Python.3.12`.

> **In WSL: work in `~/`, not `/mnt/c/`.** Installing into a project on the Windows drive is dramatically slower, and file-watching (`--reload`, `pytest-watch`) often doesn't fire at all. See [[Setting up a dev machine]].

```python
import pandas as pd
import numpy as np
```

---

## 1. Creating and reading

```python
import pandas as pd

pd.DataFrame({"a": [1,2], "b": ["x","y"]})
pd.DataFrame(list_of_dicts)
pd.Series([1,2,3], name="col")

pd.read_csv("f.csv")
pd.read_csv("f.csv", sep=";", usecols=["a","b"], nrows=1000,
            dtype={"id": str},                       # <- keep IDs as strings
            parse_dates=["date"], na_values=["N/A", "-"],
            encoding="utf-8", low_memory=False)
pd.read_csv("f.csv", chunksize=100_000)              # -> iterator of DataFrames

pd.read_parquet("f.parquet", columns=["a","b"])      # column pruning
pd.read_json("f.json", lines=True)                   # JSONL
pd.read_excel("f.xlsx", sheet_name="Sheet1")
pd.read_sql("SELECT * FROM t WHERE d > %(d)s", conn, params={"d": start})
pd.read_sql_table("t", engine)
```

> **Always pass `dtype={"id": str}` for identifiers.** pandas will happily turn `007` into `7`.

> **Use Parquet, not CSV, for anything intermediate.** 5-10× smaller, types preserved, column pruning ([[07 — DATA ENGINEERING]]).

## 2. Investigating data

**Do this first, every time.**

```python
df.shape                     # (rows, cols)
df.info()                    # dtypes, non-null counts, memory
df.head(10) · df.tail() · df.sample(5)
df.describe()                # numeric summary
df.describe(include="all")   # + categorical
df.dtypes · df.columns · df.index
df.memory_usage(deep=True).sum() / 1e6      # MB
```

```python
# nulls
df.isna().sum()                              # count per column
df.isna().mean().sort_values(ascending=False)   # RATE per column  <- most useful
df.isna().any(axis=1).sum()                  # rows with any null

# duplicates
df.duplicated().sum()
df.duplicated(subset=["invoice","product"]).sum()
df[df.duplicated(keep=False)].sort_values("invoice")    # see them

# uniqueness / cardinality
df.nunique()
df["country"].value_counts()
df["country"].value_counts(normalize=True)   # proportions
df["country"].unique()

# correlation
df.corr(numeric_only=True)
df[["a","b"]].corr().iloc[0,1]
```

> **`df.isna().mean()` is the single most valuable profiling line.** A column that was 2% null and is now 60% null means something upstream broke ([[Observability for data and ML pipelines]]).

## 3. Selecting

```python
df["a"]                      # Series
df[["a","b"]]                # DataFrame
df.a                         # breaks on names with spaces - avoid

df.loc[0]                    # by LABEL
df.loc[0:5, "a":"c"]         # label slicing is INCLUSIVE of the end
df.loc[df.qty > 0, ["a","b"]]
df.iloc[0]                   # by POSITION
df.iloc[0:5, 0:3]            # position slicing is EXCLUSIVE
df.at[0, "a"]                # single value, fast
df.iat[0, 0]

df.query("qty > 0 and country == 'UK'")
df.query("qty > @threshold")                 # @ references a Python variable
df.filter(like="price")                      # columns containing "price"
df.filter(regex="^s\\d+$")
```

> **`.loc` is inclusive, `.iloc` is exclusive.** `df.loc[0:5]` gives 6 rows; `df.iloc[0:5]` gives 5. This catches everyone.

## 4. Filtering

```python
df[df.qty > 0]
df[(df.qty > 0) & (df.country == "UK")]      # PARENTHESES MANDATORY
df[df.country.isin(["UK","FR"])]
df[~df.country.isin(["UK"])]
df[df.name.str.contains("mug", case=False, na=False)]
df[df.price.between(10, 100)]
df[df.date.dt.year == 2024]
df[df.a.isna()] · df[df.a.notna()]
```

> **`&` `|` `~`, not `and` `or` `not`** — and brackets are mandatory, same as [[PySpark reference]].
> **`na=False` on `.str` methods**, or nulls propagate and break the mask.

## 5. Cleaning and preprocessing

```python
import pandas as pd

# nulls
df.dropna() · df.dropna(subset=["a"]) · df.dropna(how="all") · df.dropna(thresh=3)
df.fillna(0) · df.fillna({"a": 0, "b": "unknown"})
df.fillna(df.mean(numeric_only=True))
df["a"].ffill() · df["a"].bfill()            # forward / backward fill
df["a"].interpolate()

# duplicates
df.drop_duplicates()
df.drop_duplicates(subset=["invoice","product"], keep="last")
df.sort_values("updated_at").drop_duplicates("id", keep="last")   # keep a SPECIFIC row

# types
df["qty"] = df["qty"].astype("int64")
df["qty"] = pd.to_numeric(df["qty"], errors="coerce")    # bad values -> NaN
df["date"] = pd.to_datetime(df["date"], format="%Y-%m-%d", errors="coerce")
df["cat"] = df["cat"].astype("category")                 # big memory saving
df.convert_dtypes()                                      # nullable dtypes

# text
df["country"] = df["country"].str.strip().str.title()
df["phone"] = df["phone"].str.replace(r"[^0-9]", "", regex=True)

# rename / drop / reorder
df.rename(columns={"old":"new"})
df.rename(columns=str.lower)
df.drop(columns=["a","b"])
df[["b","a"]]                                # reorder
df.reset_index(drop=True)
df.set_index("id")
```

> **`errors="coerce"` turns bad values into NaN silently.** Always count them afterwards: `df.qty.isna().sum()`.

## 6. Creating columns

```python
import numpy as np
import pandas as pd

df["revenue"] = df.qty * df.price
df = df.assign(revenue=lambda d: d.qty * d.price,        # chainable
               margin=lambda d: d.revenue * 0.2)

# conditional
df["band"] = np.where(df.revenue > 1000, "high", "low")
df["band"] = np.select(
    [df.revenue > 1000, df.revenue > 100],
    ["high", "medium"],
    default="low")
df["band"] = pd.cut(df.revenue, bins=[0,100,1000,np.inf], labels=["low","mid","high"])
df["decile"] = pd.qcut(df.revenue, 10, labels=False)     # equal-sized buckets

# map / replace
df["region"] = df.country.map({"UK":"Europe","US":"Americas"})
df["country"] = df.country.replace({"U.K.":"UK"})
```

> **`np.select` for multi-condition logic, not chained `apply`.** It's vectorised and orders of magnitude faster.

## 7. Aggregation

```python
df.groupby("country").size()
df.groupby("country")["revenue"].sum()
df.groupby(["country","segment"]).agg(
    total=("revenue","sum"),
    mean=("revenue","mean"),
    n=("revenue","size"),
    customers=("customer_id","nunique"),
    first_date=("date","min"),
).reset_index()

df.groupby("country").agg({"revenue":["sum","mean"], "qty":"sum"})
df.groupby("country")["revenue"].transform("sum")     # broadcast back to rows
df.groupby("country").apply(lambda g: g.nlargest(3, "revenue"))
df.groupby("country").filter(lambda g: g.revenue.sum() > 1000)

# top N per group
df.sort_values("revenue", ascending=False).groupby("country").head(3)
```

> **`.transform()` broadcasts the aggregate back to every row** — the pandas equivalent of a SQL window function ([[SQL fundamentals]]).

> **`.reset_index()` after `groupby`** or the keys stay in the index and later merges get confusing.

## 8. Joining and combining

```python
import pandas as pd

pd.merge(a, b, on="id", how="inner")          # inner|left|right|outer|cross
pd.merge(a, b, left_on="a_id", right_on="b_id")
pd.merge(a, b, on="id", suffixes=("_left","_right"))
pd.merge(a, b, on="id", indicator=True)       # adds _merge column - great for debugging
pd.merge_asof(a, b, on="time", by="id", direction="backward")   # time-series join

a.join(b, on="id")                            # index-based
pd.concat([a, b])                             # stack rows
pd.concat([a, b], axis=1)                     # side by side
pd.concat([a, b], ignore_index=True)
```

> ⚠️ **Always check row counts around a merge.**
> ```python
> before = len(df); after = len(df.merge(dim, on="key")); print(before, after)
> ```
> Up = duplicates on the right. Down = an inner join dropped rows. `indicator=True` shows you which.

## 9. Reshaping

```python
df.pivot_table(index="date", columns="country", values="revenue",
               aggfunc="sum", fill_value=0, margins=True)
df.pivot(index="date", columns="country", values="revenue")   # no aggregation

df.melt(id_vars=["date"], value_vars=["uk","fr"],
        var_name="country", value_name="revenue")

df.stack() · df.unstack()
df.explode("tags")                            # one row per list element
df.T                                          # transpose
```

## 10. Time series

```python
import pandas as pd

df["date"] = pd.to_datetime(df.date)
df = df.set_index("date")

df.resample("D").sum()                        # D H W M Q Y
df.resample("W-MON").agg({"revenue":"sum"})
df.rolling(7).mean()
df.rolling("7D").mean()                       # time-based window
df.expanding().sum()                          # cumulative
df.shift(1) · df.diff() · df.pct_change()

df.index.year · df.date.dt.month · df.date.dt.dayofweek
df.date.dt.to_period("M")
df.between_time("09:00","17:00")
df.tz_localize("UTC").tz_convert("Europe/London")
```

## 11. Sorting and ranking

```python
df.sort_values("revenue", ascending=False)
df.sort_values(["country","revenue"], ascending=[True, False])
df.sort_values("a", na_position="first")
df.nlargest(10, "revenue") · df.nsmallest(10, "revenue")
df.rank(method="dense", ascending=False)
```

## 12. Apply — and why to avoid it

```python
df["x"] = df.a.apply(lambda v: v * 2)         # ✗ slow, row by row
df["x"] = df.a * 2                            # ✓ vectorised, ~100x faster

df.apply(lambda row: f(row.a, row.b), axis=1)  # ✗✗ slowest of all
```

```python
for i, row in df.iterrows():                  # ✗✗✗ almost always a bug
    df.at[i, "x"] = row.a * 2
```

> **`iterrows` creates a Series per row.** On a million rows it's minutes versus milliseconds. If you're reaching for it, there is a vectorised way — `np.where`, `np.select`, `.map`, `.merge`, `groupby().transform()` ([[When to leave Python]]).

## 13. Writing

```python
df.to_csv("f.csv", index=False)               # index=False almost always
df.to_parquet("f.parquet", index=False, compression="snappy")
df.to_json("f.jsonl", orient="records", lines=True)
df.to_sql("table", engine, if_exists="replace", index=False, chunksize=10_000)
df.to_excel("f.xlsx", sheet_name="Data", index=False)
df.to_dict(orient="records")                  # -> list of dicts
```

## 14. Performance

| Technique | Effect |
|---|---|
| **Vectorise** — no `apply`/`iterrows` | 10-100× |
| `category` dtype for repeated strings | Large memory saving |
| Read only the columns you need | Proportional |
| Parquet instead of CSV | 5-10× smaller and faster |
| Downcast numerics | `pd.to_numeric(s, downcast="integer")` |
| `chunksize` for big CSVs | Constant memory |
| Switch to [[Polars]] / [[DuckDB]] | 5-30× |

```python
import pandas as pd

pd.options.display.max_columns = 100
pd.options.display.width = 200
pd.options.mode.copy_on_write = True          # pandas 2+ - avoids SettingWithCopyWarning
```

## Common mistakes

| Mistake | Fix |
|---|---|
| `SettingWithCopyWarning` | `df = df.copy()`, or enable copy-on-write |
| `and`/`or` in a filter | `&` / `\|` with brackets |
| IDs turned into ints | `dtype={"id": str}` |
| `iterrows` | Vectorise |
| Forgetting `index=False` on `to_csv` | Adds a junk column |
| Merge silently duplicating rows | Check counts; `indicator=True` |
| `.loc` vs `.iloc` slice ends | `.loc` inclusive, `.iloc` exclusive |
| `.str` methods failing on nulls | `na=False` |
| Chained assignment `df[a][b] = x` | Use `.loc[b, a] = x` |

## Related

[[Python libraries]] · [[NumPy]] · [[Polars]] · [[DuckDB]] · [[PySpark reference]] · [[Matplotlib and Plotly]] · [[When to leave Python]]
