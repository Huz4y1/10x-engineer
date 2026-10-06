---
tags: [postgres, sql, database, reference, cheatsheet]
---

# PostgreSQL reference

Everything you'd otherwise google. Hub: [[PostgreSQL]] · SQL concepts: [[SQL fundamentals]]

In depth: [[PostgreSQL data types]] · [[PostgreSQL SQL]] · [[pgAdmin 4]] · [[PostgreSQL with SQLAlchemy]] · Cloud: [[Azure Database for PostgreSQL]] · [[AWS RDS for PostgreSQL]]

---

## Connect

```bash
psql -h localhost -U engine -d engines           # interactive
psql "postgresql://user:pass@host:5432/db"       # URL form
PGPASSWORD=secret psql -h host -U user -d db -c "SELECT 1;"
```

### psql meta-commands (the ones worth memorising)

| Command | Does |
|---|---|
| `\l` | List databases |
| `\c dbname` | Connect to another database |
| `\dt` | List tables |
| `\d tablename` | **Describe a table** — columns, types, indexes |
| `\d+ tablename` | …plus sizes and comments |
| `\di` | List indexes |
| `\dn` | List schemas |
| `\du` | List users/roles |
| `\df` | List functions |
| `\x` | **Toggle expanded output** — essential for wide tables |
| `\timing` | Show query duration |
| `\e` | Edit the last query in `$EDITOR` |
| `\i file.sql` | Run a file |
| `\copy tbl FROM 'f.csv' CSV HEADER` | Client-side bulk load |
| `\q` | Quit |

> `\x` then `\d+ tablename` is the fastest way to understand an unfamiliar database.

---

## Data types

| Use | Type | Note |
|---|---|---|
| Whole numbers | `INTEGER`, `BIGINT` | `BIGINT` for IDs that will grow |
| **Money** | `NUMERIC(18,2)` | **Never `REAL`/`FLOAT`** |
| Measurements | `REAL`, `DOUBLE PRECISION` | Fine for sensors |
| Text | `TEXT` | No penalty vs `VARCHAR(n)` in Postgres |
| True/false | `BOOLEAN` | |
| Date | `DATE` | |
| **Timestamp** | `TIMESTAMPTZ` | **Always TZ-aware.** `TIMESTAMP` loses the offset. |
| Duration | `INTERVAL` | |
| JSON | `JSONB` | Binary, indexable. Prefer over `JSON`. |
| Array | `INTEGER[]` | Native arrays |
| UUID | `UUID` | With `gen_random_uuid()` |
| Auto ID | `BIGSERIAL` / `GENERATED ALWAYS AS IDENTITY` | |

> **`TIMESTAMPTZ` always.** Storing local time without an offset means twice a year you cannot tell 01:30 BST from 01:30 GMT. Store UTC, display local.

---

## DDL

```sql
CREATE TABLE readings (
    id          BIGGENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    unit        INTEGER      NOT NULL,
    cycle       INTEGER      NOT NULL CHECK (cycle > 0),
    temp_c      REAL,
    revenue     NUMERIC(18,2) NOT NULL DEFAULT 0,
    meta        JSONB,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT now(),

    CONSTRAINT uq_reading UNIQUE (unit, cycle)
);

ALTER TABLE readings ADD COLUMN notes TEXT;
ALTER TABLE readings ALTER COLUMN temp_c TYPE DOUBLE PRECISION;
ALTER TABLE readings DROP COLUMN notes;
ALTER TABLE readings RENAME COLUMN temp_c TO temperature_c;

DROP TABLE IF EXISTS readings CASCADE;
```

> **`UNIQUE` on the grain key is how you make loads idempotent** — pair it with `ON CONFLICT DO NOTHING` and re-running a load becomes safe ([[Project 002 — Aircraft Sensor Data Pipeline]]).

---

## Upserts

```sql
INSERT INTO readings (unit, cycle, temp_c)
VALUES (1, 5, 412.5)
ON CONFLICT (unit, cycle) DO NOTHING;              -- ignore duplicates

INSERT INTO readings (unit, cycle, temp_c)
VALUES (1, 5, 412.5)
ON CONFLICT (unit, cycle)
DO UPDATE SET temp_c = EXCLUDED.temp_c,            -- EXCLUDED = the proposed row
              updated_at = now();
```

---

## Indexes

```sql
CREATE INDEX idx_readings_unit ON readings (unit);
CREATE INDEX idx_readings_unit_cycle ON readings (unit, cycle);      -- composite
CREATE UNIQUE INDEX idx_x ON t (col);
CREATE INDEX idx_recent ON readings (unit) WHERE cycle > 100;        -- partial
CREATE INDEX idx_lower ON users (lower(email));                      -- expression
CREATE INDEX idx_meta ON readings USING GIN (meta);                  -- JSONB
CREATE INDEX CONCURRENTLY idx_big ON huge_table (col);               -- no write lock

DROP INDEX idx_readings_unit;
REINDEX TABLE readings;
```

| Type | Use |
|---|---|
| **B-tree** (default) | Equality and range. 95% of cases. |
| **GIN** | JSONB, arrays, full-text |
| GiST | Geometric, ranges |
| BRIN | Huge tables with naturally ordered data (time series) |
| Hash | Equality only. Rarely worth it. |

> **Composite index column order matters.** `(unit, cycle)` helps `WHERE unit = 1` and `WHERE unit = 1 AND cycle = 5`, but **not** `WHERE cycle = 5` alone. Put the `=` column first, the range column second.

> **`CREATE INDEX CONCURRENTLY` on a live table.** The plain form takes a write lock and blocks everything.

---

## Reading a query plan

```sql
EXPLAIN SELECT * FROM readings WHERE unit = 42;
EXPLAIN ANALYZE SELECT ...;                 -- actually runs it, shows real times
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;      -- + cache hit info
```

| You see | Meaning | Worry? |
|---|---|---|
| **Seq Scan** | Reading every row | Only if the table is big and you filtered |
| **Index Scan** | Using an index, fetching rows | Good |
| **Index Only Scan** | Answered entirely from the index | Best |
| **Bitmap Heap Scan** | Many matches — index then batch fetch | Normal |
| **Nested Loop** | For each left row, scan right | 🚨 On big tables = missing index |
| **Hash Join** | Build hash of one side | Fine |
| **Merge Join** | Both sides sorted | Fine |
| `rows=` far from `actual rows` | **Stale statistics** | `ANALYZE tablename` |

> **Read plans inside-out, bottom-up** — the most indented lines run first. The number that matters is `actual time` in `EXPLAIN ANALYZE`, not `cost`.

---

## MVCC and VACUUM

Postgres never overwrites a row in place. An `UPDATE` writes a **new version** and marks the old one dead. That's **MVCC** — Multi-Version Concurrency Control — and it's why readers never block writers.

The cost: dead rows accumulate.

```sql
VACUUM readings;                -- reclaim space for reuse
VACUUM ANALYZE readings;        -- + refresh statistics
VACUUM FULL readings;           -- ⚠️ rewrites the table, takes an EXCLUSIVE lock
ANALYZE readings;               -- statistics only

SELECT relname, n_dead_tup, n_live_tup
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;
```

> **Autovacuum handles this normally.** If a table is bloated, autovacuum is probably being blocked by a **long-running transaction** — dead rows can't be removed while any transaction might still need them. Check `pg_stat_activity` for old `idle in transaction` sessions.

> **Never `VACUUM FULL` on a live production table** — it locks it completely.

---

## Transactions

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;                          -- or ROLLBACK;

BEGIN;
SAVEPOINT sp1;
-- something risky
ROLLBACK TO sp1;
COMMIT;
```

**ACID:**

| Letter | Means |
|---|---|
| **Atomicity** | All of it, or none of it |
| **Consistency** | Constraints always hold |
| **Isolation** | Concurrent transactions don't see each other's half-done work |
| **Durability** | Committed means survives a crash |

**Isolation levels:**

| Level | Prevents | Postgres default |
|---|---|---|
| Read Committed | Dirty reads | ✅ |
| Repeatable Read | + non-repeatable reads | |
| Serializable | + phantom reads | Strictest, may abort |

---

## JSONB

```sql
SELECT meta->>'sensor'        AS sensor,      -- ->> returns TEXT
       meta->'nested'->>'key' AS nested,      -- ->  returns JSONB
       meta #>> '{a,b,c}'     AS deep
FROM readings
WHERE meta @> '{"status": "ok"}'              -- containment, uses GIN index
  AND meta ? 'temperature';                   -- key exists
```

> `->` gives JSONB (chainable), `->>` gives text (final). Mixing them up is the classic JSONB error.

---

## Useful admin queries

```sql
-- what's running right now
SELECT pid, now() - query_start AS duration, state, left(query, 80)
FROM pg_stat_activity
WHERE state <> 'idle' ORDER BY duration DESC;

-- kill a query
SELECT pg_cancel_backend(pid);       -- polite
SELECT pg_terminate_backend(pid);    -- forceful

-- table sizes
SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) AS size
FROM pg_catalog.pg_statio_user_tables ORDER BY pg_total_relation_size(relid) DESC;

-- unused indexes (candidates for deletion)
SELECT relname, indexrelname, idx_scan
FROM pg_stat_user_indexes WHERE idx_scan = 0;

-- blocking locks
SELECT blocked.pid AS blocked_pid, blocking.pid AS blocking_pid,
       left(blocked.query, 60) AS blocked_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));

-- cache hit ratio (want > 0.99)
SELECT sum(heap_blks_hit) / nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0)
FROM pg_statio_user_tables;
```

---

## Bulk loading

```sql
\copy readings FROM 'data.csv' WITH (FORMAT csv, HEADER true);
COPY readings FROM '/server/path.csv' WITH (FORMAT csv, HEADER true);  -- server-side
```

> **`COPY` is 10-100× faster than row-by-row `INSERT`.** For a million rows the difference is minutes vs. hours. From Python, use `psycopg`'s `copy()`.

---

## Backup and restore

```bash
pg_dump -h host -U user -d engines -F c -f backup.dump    # custom format
pg_restore -h host -U user -d engines -c backup.dump
pg_dump ... --schema-only                                  # structure only
pg_dumpall -h host -U postgres > all.sql                   # everything incl. roles
```

---

## Extensions worth knowing

```sql
CREATE EXTENSION IF NOT EXISTS pgvector;      -- vector similarity (RAG)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;  -- query performance history
CREATE EXTENSION IF NOT EXISTS postgis;       -- geospatial
CREATE EXTENSION IF NOT EXISTS timescaledb;   -- time-series
```

> **`pgvector` means you probably don't need a separate vector database** for RAG below a few million chunks ([[LLM and GenAI track]]).

---

## Postgres vs Azure SQL

| Task | Postgres | Azure SQL (T-SQL) |
|---|---|---|
| Limit | `LIMIT 10` | `SELECT TOP 10` |
| Concat | `'a' \|\| 'b'` | `'a' + 'b'` |
| Now | `NOW()` | `GETDATE()` |
| Null fallback | `COALESCE()` | `ISNULL()` / `COALESCE()` |
| Auto ID | `GENERATED ALWAYS AS IDENTITY` | `IDENTITY(1,1)` |
| Quote identifier | `"col"` | `[col]` |
| Driver | `psycopg` | `pyodbc` + ODBC Driver 18 |

> **Stick to `COALESCE`, `CAST` and standard SQL** and the same queries run on both ([[Running the whole stack locally]]).

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `connection refused` | Not running, or wrong port | `pg_isready`; check the container |
| `too many connections` | No pooling; engine per request | One pooled engine ([[Azure SQL Database]]) |
| `relation does not exist` | Wrong schema or migration not applied | `\dt`; `SET search_path` |
| Query suddenly slow | Stale stats, or missing index | `ANALYZE`; `EXPLAIN ANALYZE` |
| Table huge but few rows | Bloat from dead tuples | Check `n_dead_tup`; look for long transactions |
| `deadlock detected` | Two transactions locking in opposite order | Always lock in a consistent order |
| Index unused | Function wrapping the column | Expression index, or rewrite the predicate |
| Money off by pennies | `REAL` instead of `NUMERIC` | `NUMERIC(18,2)` |

## Related

[[06 — DATABASES]] · [[SQL fundamentals]] · [[Data modeling]] · [[Azure SQL Database]] · [[Running the whole stack locally]]
