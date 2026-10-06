---
tags: [project, postgresql, sql, python]
status: not-started
---

# Project 002 — Aircraft Sensor Data Pipeline

Index: [[28 — PROJECTS]] · Previous: [[Project 001 — Aircraft Engine Sensor Analyzer]]

---

## What you're building

The same analysis as Project 001, but the data now lives in **PostgreSQL** instead of a CSV, the statistics are computed in **SQL** instead of pandas, and there's a repeatable **load script** you can re-run safely.

## Why this project exists

A CSV is fine for one file on one laptop. The moment you have 100 files, or two people, or need to query yesterday's numbers, you need a database.

**Afterwards you will understand:** why databases exist, how to design a schema before writing code, what an index actually does to query time, and why re-running a load script twice must not double your data.

## Concepts used

| Concept | Note | New? |
|---|---|---|
| Schema design, keys, types | [[Data modeling]] · [[06 — DATABASES]] | New |
| SQL: joins, aggregation, CTEs, windows | [[SQL fundamentals]] | New |
| Indexes and query plans | [[SQL fundamentals]] | New |
| Postgres in Docker | [[Docker deep dive]] | New |
| Connecting from Python | [[Azure SQL Database]] | New |
| **Idempotent loads** | [[Unity Catalog and orchestration]] | New |
| Testing with a real database | [[Testing and CI-CD]] | New |

## Architecture

```mermaid
flowchart LR
    A["CSV files"] --> B["load.py<br/>bulk insert"]
    B --> C[("PostgreSQL<br/>raw_readings")]
    C --> D["transform.sql<br/>views + aggregates"]
    D --> E[("engine_summary")]
    E --> F["report.py"]
```

## The schema

Design it **before** writing any code ([[Data modeling]]).

**Grain statement:** one row per engine, per cycle.

```sql
CREATE TABLE raw_readings (
    reading_id   BIGSERIAL PRIMARY KEY,
    dataset      TEXT      NOT NULL,        -- FD001..FD004
    unit         INTEGER   NOT NULL,
    cycle        INTEGER   NOT NULL,
    setting1     REAL, setting2 REAL, setting3 REAL,
    s1 REAL, s2 REAL, s3 REAL, s4 REAL, s5 REAL, s6 REAL, s7 REAL,
    s8 REAL, s9 REAL, s10 REAL, s11 REAL, s12 REAL, s13 REAL, s14 REAL,
    s15 REAL, s16 REAL, s17 REAL, s18 REAL, s19 REAL, s20 REAL, s21 REAL,
    ingested_at  TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (dataset, unit, cycle)          -- <- this is what makes reloads safe
);

CREATE INDEX ix_readings_unit  ON raw_readings (dataset, unit);
CREATE INDEX ix_readings_cycle ON raw_readings (dataset, unit, cycle);
```

> **The `UNIQUE (dataset, unit, cycle)` constraint is the most important line.** It encodes the grain *in the database*, so a double-run is impossible rather than merely unlikely. Combined with `ON CONFLICT DO NOTHING`, your load becomes **idempotent** — run it a hundred times, get the same table.

## Build steps

- [ ] **1 — Postgres in Docker**
  ```yaml
  services:
    postgres:
      image: postgres:16
      ports: ["5432:5432"]
      environment:
        POSTGRES_USER: engine
        POSTGRES_PASSWORD: devonly
        POSTGRES_DB: engines
      volumes: [pgdata:/var/lib/postgresql/data]
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U engine"]
        interval: 5s
        retries: 10
  volumes: {pgdata: {}}
  ```
  **Done when:** `psql -h localhost -U engine -d engines -c '\dt'` connects.

- [ ] **2 — Migrations, not hand-typed DDL**
  ```bash
  uv add alembic sqlalchemy psycopg[binary]
  alembic init migrations
  alembic revision -m "create raw_readings"
  alembic upgrade head
  ```
  > A schema you typed into `psql` once is a schema nobody else can recreate. Migrations are version-controlled, reviewable and reversible ([[CI-CD pipelines]]).

- [ ] **3 — The idempotent load**
  ```python
  from pathlib import Path
  from psycopg import connect
  from psycopg.rows import dict_row

  INSERT = """
      INSERT INTO raw_readings (dataset, unit, cycle, setting1, ..., s21)
      VALUES (%(dataset)s, %(unit)s, %(cycle)s, ...)
      ON CONFLICT (dataset, unit, cycle) DO NOTHING
  """

  def load(path: Path, dataset: str, conn) -> int:
      df = load_engine_data(path)          # reuse Project 001's function
      rows = df.assign(dataset=dataset).to_dict("records")
      with conn.cursor() as cur:
          cur.executemany(INSERT, rows)    # batched
      conn.commit()
      return cur.rowcount
  ```
  **Done when:** running it twice gives the same `COUNT(*)`.

- [ ] **4 — Rewrite the analysis in SQL**

  RUL, as a window function instead of a pandas groupby:
  ```sql
  SELECT
      unit, cycle,
      MAX(cycle) OVER (PARTITION BY dataset, unit) - cycle AS rul
  FROM raw_readings
  WHERE dataset = 'FD001';
  ```
  > Compare this with `groupby().transform("max")` from Project 001. **Same idea, same name — "partition by".** Window functions and pandas groupby-transform are the same concept in two dialects ([[SQL fundamentals]]).

  Per-engine summary with a CTE:
  ```sql
  WITH per_engine AS (
      SELECT dataset, unit,
             MAX(cycle) AS total_cycles,
             AVG(s2) AS mean_s2, STDDEV(s2) AS std_s2,
             AVG(s11) AS mean_s11
      FROM raw_readings
      GROUP BY dataset, unit
  )
  SELECT * FROM per_engine ORDER BY total_cycles ASC LIMIT 10;
  ```

  Sensor correlation with RUL:
  ```sql
  WITH with_rul AS (
      SELECT *, MAX(cycle) OVER (PARTITION BY dataset, unit) - cycle AS rul
      FROM raw_readings WHERE dataset = 'FD001'
  )
  SELECT
      CORR(s2,  rul) AS corr_s2,
      CORR(s4,  rul) AS corr_s4,
      CORR(s11, rul) AS corr_s11,
      CORR(s15, rul) AS corr_s15
  FROM with_rul;
  ```

- [ ] **5 — A materialised summary table**
  ```sql
  CREATE MATERIALIZED VIEW engine_summary AS
  SELECT dataset, unit, MAX(cycle) AS total_cycles, AVG(s11) AS mean_s11
  FROM raw_readings GROUP BY dataset, unit;

  REFRESH MATERIALIZED VIEW engine_summary;
  ```
  > This is your first **gold table** — precomputed so reads are instant. Exactly the pattern in [[Deployment patterns]].

- [ ] **6 — Prove indexes matter**
  ```sql
  EXPLAIN ANALYZE SELECT * FROM raw_readings WHERE unit = 42;
  DROP INDEX ix_readings_unit;
  EXPLAIN ANALYZE SELECT * FROM raw_readings WHERE unit = 42;   -- Seq Scan now
  CREATE INDEX ix_readings_unit ON raw_readings (dataset, unit);
  ```
  **Done when:** you can point at `Seq Scan` vs `Index Scan` and state the time difference.

- [ ] **7 — Tests against a real database**
  ```python
  import pytest

  @pytest.fixture(scope="session")
  def db():
      conn = connect("postgresql://engine:devonly@localhost/engines_test")
      yield conn
      conn.close()

  def test_load_is_idempotent(db, tmp_csv):
      load(tmp_csv, "TEST", db)
      first = count_rows(db)
      load(tmp_csv, "TEST", db)          # again
      assert count_rows(db) == first     # <- the whole point
  ```

## Checkpoints

- [ ] Loading twice does **not** duplicate rows
- [ ] SQL RUL matches Project 001's pandas RUL exactly, for the same engine
- [ ] You can show `EXPLAIN` output before and after dropping an index
- [ ] Migrations recreate the schema from empty

## Make it fail deliberately

- [ ] Drop the `UNIQUE` constraint, load twice, watch counts double — then restore it
- [ ] Write `WHERE unit = '42'` (string vs integer) and see the plan change
- [ ] Kill the load halfway; confirm the transaction rolled back and left nothing partial

## What will go wrong

| Symptom | Cause | Fix |
|---|---|---|
| `connection refused` | Postgres not ready yet | Healthcheck + `condition: service_healthy` |
| Load takes minutes | Row-by-row inserts | `executemany`, or `COPY` for bulk |
| Second load duplicates | No unique constraint | Add it; use `ON CONFLICT` |
| `relation does not exist` | Migration not applied | `alembic upgrade head` |
| Query slow after load | No index, or stale stats | Create index; `ANALYZE raw_readings` |
| Numbers differ from Project 001 | `REAL` vs `float64` rounding | Expected; compare with a tolerance |

## Stretch goals

- [ ] Use `COPY FROM` instead of `INSERT` — 10-100x faster for bulk
- [ ] Add a `dim_engine` dimension table and make `raw_readings` reference it ([[Data modeling]])
- [ ] Add `pgvector` and store a sensor embedding per engine

## What you learned

*Fill in afterwards.*

## Next project

[[Project 003 — Distributed Sensor Pipeline]]
