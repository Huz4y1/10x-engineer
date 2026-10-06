---
tags: [azure, sql, database]
status: not-started
---

# Azure SQL Database

> **What this is:** Microsoft SQL Server, run for you in the cloud, with the boring parts (patching, backups, failover) handled.
> **Why you care:** it's the fast, small, curated store your API and dashboards read from. The lake holds everything; this holds the answers.

---

## Why not just query the lake?

You already have all the data in ADLS. Why copy the results into a database?

**Because they're good at opposite things.**

| Question | Lake (Spark on Parquet) | Azure SQL |
|---|---|---|
| "Total revenue by category over 2 years" | ~seconds. Built for this. | Slow if the table is huge |
| "Give me customer 17850's segment" | ~10–30 seconds — spins up a cluster, scans files | **~5 milliseconds** |
| Cost to hold 40TB | Cheap | Very expensive |
| 200 dashboard users clicking at once | Falls over | Fine |

A dashboard needs the second row. A user clicking a filter cannot wait 20 seconds, and you can't put a Spark cluster behind a web request.

So: **Spark does the heavy thinking once, writes small result tables to Azure SQL, and the API serves from there.** Your gold tables are maybe a few hundred thousand rows — nothing for a database.

---

## 1. Provisioning

### The two things you create

1. A **logical server** (`retail-sql-01`) — a connection endpoint and an admin login. Not a machine you pay for.
2. A **database** (`retaildb`) on that server — this is what costs money.

```bash
RG=retail-rg
LOC=uksouth
SERVER=retail-sql-$RANDOM

az sql server create \
  --name $SERVER --resource-group $RG --location $LOC \
  --admin-user sqladmin --admin-password '<a-long-random-password>'

az sql db create \
  --name retaildb --server $SERVER --resource-group $RG \
  --edition GeneralPurpose --compute-model Serverless \
  --family Gen5 --capacity 1 \
  --auto-pause-delay 60          # pause after 60 min idle → stops billing compute
```

> **Use the Serverless tier while learning.** It auto-pauses when idle and you stop paying for compute. Provisioned tiers bill 24/7 whether you touch them or not, and that's how a learning project quietly costs £80/month.
>
> The trade-off: the first query after a pause takes ~30 seconds while it wakes up. Harmless for you; you'd disable auto-pause in production.

### The firewall — your first error

By default, nothing can connect. Not even you.

```bash
# let your current IP in
az sql server firewall-rule create \
  --resource-group $RG --server $SERVER \
  --name my-laptop --start-ip-address <your-ip> --end-ip-address <your-ip>

# let other Azure services (Databricks, Container Apps) in
az sql server firewall-rule create \
  --resource-group $RG --server $SERVER \
  --name allow-azure --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
```

That `0.0.0.0 → 0.0.0.0` rule is a special value meaning **"allow Azure-internal traffic"**, not "allow the whole internet". Confusing, but that's the API.

> Home IP addresses change. When a connection that worked yesterday times out today, check your IP first — it's the most common cause and takes ten seconds to rule out.

### Private endpoints (know it exists)

A firewall rule still means traffic goes over the public internet, just filtered. A **private endpoint** gives the database a private IP inside your virtual network, so traffic never touches the public internet at all. That's the production answer. Overkill for the capstone, but you'll be asked about it in interviews.

---

## 2. Connecting from Python

### The drivers, and which to pick

- **`pyodbc`** — the standard synchronous driver. Needs the Microsoft ODBC Driver 18 installed on the machine (or in your Docker image).
- **`aioodbc`** — async wrapper around pyodbc.
- **`SQLAlchemy`** — not a driver; a layer *on top* of one. Gives you connection pooling, a query builder, and an ORM.

**Use SQLAlchemy over pyodbc.** Not for the ORM — for the **connection pool** (section 3). You can still write raw SQL through it.

### Connection string

```python
import os
from sqlalchemy import create_engine, text

# note the URL-encoding of the odbc string
conn_str = (
    "mssql+pyodbc://sqladmin:PASSWORD@retail-sql-01.database.windows.net:1433/retaildb"
    "?driver=ODBC+Driver+18+for+SQL+Server&Encrypt=yes&TrustServerCertificate=no"
)

engine = create_engine(os.environ["AZURE_SQL_URL"], pool_size=5, max_overflow=10)

with engine.connect() as conn:
    rows = conn.execute(text(
        "SELECT TOP 10 product_category, revenue FROM daily_sales ORDER BY revenue DESC"
    )).fetchall()
```

Better: skip the password entirely with a managed identity, as in [[Azure fundamentals]]:

```python
from sqlalchemy import create_engine
from azure.identity import DefaultAzureCredential
import struct

def get_token():
    token = DefaultAzureCredential().get_token("https://database.windows.net/.default")
    tok = token.token.encode("utf-16-le")
    return struct.pack("<i", len(tok)) + tok

engine = create_engine(
    "mssql+pyodbc://retail-sql-01.database.windows.net:1433/retaildb"
    "?driver=ODBC+Driver+18+for+SQL+Server",
    connect_args={"attrs_before": {1256: get_token()}},   # 1256 = SQL_COPT_SS_ACCESS_TOKEN
)
```

Ugly, but it's copy-paste once and there's no password anywhere.

### Async, for FastAPI

FastAPI is async. If you make blocking database calls inside an `async def` endpoint, you freeze the entire event loop and your API serves one request at a time.

```python
from sqlalchemy import text
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

engine = create_async_engine("mssql+aioodbc://...", pool_size=5, max_overflow=10)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_forecast_rows(category: str):
    async with SessionLocal() as session:
        result = await session.execute(
            text("SELECT * FROM daily_sales WHERE product_category = :cat"),
            {"cat": category},
        )
        return result.fetchall()
```

> **Two rules, always:**
> 1. **Never f-string a value into SQL.** `f"WHERE id = {user_input}"` is a SQL injection hole. Use `:named` parameters — the driver escapes them properly and it's faster too (the plan gets cached).
> 2. **If the endpoint is `async def`, every DB call inside must be `await`ed.** Mixing a sync driver into async code is worse than being fully synchronous.

---

## 3. Connection pooling — why FastAPI needs it

Opening a database connection is expensive: TCP handshake, TLS negotiation, authentication. Roughly 50–200ms. If every API request opens one, you've added 200ms to every request before doing any work.

A **pool** opens N connections at startup and lends them out. Borrowing from a pool takes microseconds.

```python
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(
    url,
    pool_size=5,        # connections kept open permanently
    max_overflow=10,    # extras allowed under load, closed afterwards
    pool_timeout=30,    # seconds to wait for a free one before erroring
    pool_recycle=1800,  # recycle after 30 min — Azure kills idle connections
    pool_pre_ping=True, # test a connection before handing it out
)
```

> **`pool_pre_ping=True` and `pool_recycle` are not optional on Azure SQL.** Azure silently drops idle connections, and serverless databases pause entirely. Without these you get random "connection reset" errors on the first request after a quiet period — an intermittent bug that's miserable to diagnose. Set them from day one.

**One engine per application, created once at startup.** Creating an engine per request creates a pool per request, which defeats the entire point and eventually exhausts the database's connection limit.

---

## 4. T-SQL — where it differs from what you learned

Azure SQL speaks **T-SQL**. Spark SQL and Postgres speak something closer to standard SQL. These are the differences that will bite you when moving a query between them:

| Task | T-SQL (Azure SQL) | Spark SQL / Postgres |
|---|---|---|
| Limit rows | `SELECT TOP 10 ...` | `... LIMIT 10` |
| Limit with offset | `OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY` | `LIMIT 10 OFFSET 20` |
| String concat | `'a' + 'b'` | `'a' \|\| 'b'` / `CONCAT()` |
| Null fallback | `ISNULL(x, 0)` | `COALESCE(x, 0)` ← works in both, prefer it |
| Now | `GETDATE()` / `SYSDATETIME()` | `NOW()` / `current_timestamp()` |
| Date maths | `DATEADD(day, -7, d)` | `d - INTERVAL 7 DAYS` |
| Auto-increment key | `id INT IDENTITY(1,1) PRIMARY KEY` | `GENERATED ALWAYS AS IDENTITY` |
| Cast | `CAST(x AS DECIMAL(10,2))` | same |
| Quote an identifier | `[my column]` | `"my column"` or `` `my column` `` |

> `TOP` without `ORDER BY` returns *arbitrary* rows, not the "first" ones — there is no inherent order in a table. Always pair them.

### Types worth choosing deliberately

| Use | Type | Why |
|---|---|---|
| Money | `DECIMAL(18,2)` | **Never `FLOAT`.** Floats can't represent 0.1 exactly; your totals will be off by pennies and nobody will trust the dashboard. |
| Text | `NVARCHAR(100)` | `N` = Unicode. Product descriptions have accents and symbols. |
| Dates | `DATE` or `DATETIME2` | Not the legacy `DATETIME` — worse precision and range |
| True/false | `BIT` | 0/1 |

---

## 5. Getting the gold tables in from Spark

Databricks writes to Azure SQL via JDBC:

```python
(gold_daily_sales.write
    .format("jdbc")
    .option("url", "jdbc:sqlserver://retail-sql-01.database.windows.net:1433;database=retaildb")
    .option("dbtable", "daily_sales")
    .option("user", dbutils.secrets.get("retail-scope", "sql-user"))
    .option("password", dbutils.secrets.get("retail-scope", "sql-password"))
    .option("batchsize", 10000)          # ← massively faster than the default
    .mode("overwrite")
    .save())
```

> **`batchsize` matters enormously.** The default sends rows in small batches, and writing 200,000 rows can take twenty minutes. At 10,000 it's under a minute. This is the first thing to check when a JDBC write is inexplicably slow.

**Which write mode:**
- `overwrite` — drops and recreates the table. Simple; fine for gold tables you fully rebuild. Beware: it also throws away your indexes, so recreate them after.
- `append` — adds rows. Careful about running twice and duplicating.
- For genuine upserts, write to a staging table and run a `MERGE` in SQL.

---

## 6. Indexes for the serving layer

Your gold tables exist to be queried by the API in specific ways. Index exactly those ways.

```sql
-- API does: WHERE product_category = ? AND date BETWEEN ? AND ?
CREATE INDEX idx_daily_sales_cat_date ON daily_sales (product_category, date);

-- API does: WHERE customer_id = ?
CREATE INDEX idx_customer_rfm_id ON customer_rfm (customer_id);
```

**Column order in a composite index matters.** `(product_category, date)` works for a filter on `product_category` alone, or on both. It does **not** help a filter on `date` alone. Think of a phone book sorted by surname then first name: useless for finding everyone called "James".

> Put the column you filter with `=` first, the range/`BETWEEN` column second.

**`INCLUDE` for covering indexes:** if the index contains every column the query needs, the database never touches the table at all.

```sql
CREATE INDEX idx_cat_date ON daily_sales (product_category, date) INCLUDE (revenue, units_sold);
```

---

## 7. Backups and tiers

**Backups are automatic.** Full weekly, differential every 12–24 hours, transaction log every 5–10 minutes. Retained 7 days by default (configurable to 35).

**Point-in-time restore** restores to any moment in that window — but into a **new database**, never over the existing one. So recovering means: restore to `retaildb-restored`, verify, then swap names.

```bash
az sql db restore --dest-name retaildb-restored \
  --name retaildb --server $SERVER --resource-group $RG \
  --time "2026-09-01T14:30:00"
```

**Tiers, briefly:**

| Model | How it's sold | Use |
|---|---|---|
| **DTU** | A blended bundle of CPU+IO+memory (Basic/Standard/Premium) | Legacy, simple |
| **vCore** | You pick cores and memory separately | Current default, more predictable |
| **Serverless** (a vCore option) | Auto-scales, auto-pauses when idle | ✅ **This one, for learning** |

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| Connection times out | Firewall — your IP changed | Add a firewall rule for your current IP |
| Databricks/Container App can't connect | "Allow Azure services" rule missing | Add the `0.0.0.0 → 0.0.0.0` rule |
| First query after a break takes 30s | Serverless database resuming from pause | Normal. Raise `--auto-pause-delay` or disable it |
| "Login failed for user" | Wrong password, or connecting to `master` instead of your DB | Check the database name in the connection string |
| Random "connection reset" errors | Azure dropped an idle pooled connection | `pool_pre_ping=True`, `pool_recycle=1800` |
| "Too many connections" | Engine created per request instead of once | One engine at app startup, module-level |
| `pyodbc` "Data source name not found" | ODBC Driver 18 not installed | Install it — and add it to your `Dockerfile` |
| API is slow, database looks idle | Sync driver inside an `async def` endpoint | Use `aioodbc` + `await`, or make the endpoint `def` |
| Money totals off by pennies | Column is `FLOAT` | `DECIMAL(18,2)` |
| Query slow after `mode("overwrite")` | Overwrite dropped the table and its indexes | Recreate indexes after the load |
| `TOP 10` returns different rows each time | No `ORDER BY` | Add one |

---

## Practice checklist

- [ ] Provisioning a database and server, firewall rules, private endpoints
- [ ] Why you copy gold tables out of the lake into a database at all
- [ ] Connecting from Python: `pyodbc` vs. `SQLAlchemy` (async and sync)
- [ ] T-SQL specifics that differ from standard SQL (`TOP` vs `LIMIT`, `IDENTITY`, `ISNULL`)
- [ ] `DECIMAL` not `FLOAT` for money
- [ ] Connection pooling — why it matters once FastAPI is involved, and why `pool_pre_ping` is mandatory on Azure
- [ ] Composite index column order
- [ ] Backups, point-in-time restore, basic cost/performance tiers (DTU vs. vCore vs. Serverless)

## Hands-on

- [ ] Provision a **Serverless** Azure SQL Database and connect from a local Python script via SQLAlchemy
- [ ] Design and create the capstone schema (read [[Data modeling]] first)
- [ ] Write a parameterised query — then try the f-string version and understand exactly why it's dangerous
- [ ] Load a table without an index, time a filtered query, add the index, time it again
- [ ] Do a point-in-time restore to a new database name

## Resources

- [Azure SQL Database docs](https://learn.microsoft.com/azure/azure-sql/database/)
- [SQLAlchemy: Engine configuration](https://docs.sqlalchemy.org/en/latest/core/engines.html)
- [SQLAlchemy: connection pooling](https://docs.sqlalchemy.org/en/latest/core/pooling.html)

## Next

[[PySpark core]]
