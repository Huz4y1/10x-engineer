---
tags: [moc, sql, databases]
---

# SQL dialects and tools

SQL basics: [[SQL]] · Deep reference: [[SQL fundamentals]]

---

SQL doesn't have libraries — it has **dialects** and **engines**. This is what actually differs between them.

## Engines

| Engine | Note |
|---|---|
| **PostgreSQL** | [[PostgreSQL reference]] — the default |
| **Azure SQL / SQL Server** | [[Azure SQL Database]] — T-SQL |
| **Spark SQL** | [[PySpark reference]] · [[PySpark core]] |
| **DuckDB** | [[DuckDB]] — SQL over local files |
| BigQuery | [[Cloud comparison dictionary]] — billed per byte scanned |
| SQLite | Embedded, single file |

## The differences that bite

| Task | Postgres / Spark | T-SQL (Azure SQL) |
|---|---|---|
| Limit rows | `LIMIT 10` | `SELECT TOP 10` |
| Concat | `'a' \|\| 'b'` | `'a' + 'b'` |
| Now | `NOW()` | `GETDATE()` |
| Null fallback | `COALESCE()` | `ISNULL()` / `COALESCE()` |
| Auto key | `GENERATED ALWAYS AS IDENTITY` | `IDENTITY(1,1)` |
| Quote identifier | `"col"` | `[col]` |

> **Stick to `COALESCE`, `CAST` and standard SQL** and the same queries run everywhere. Where you can't, keep the dialect-specific SQL in one module rather than scattered through the codebase.

## Python access

| Library | For | Note |
|---|---|---|
| **SQLAlchemy** | Pooling, sessions, optional ORM | [[SQLAlchemy]] |
| psycopg | Postgres driver | [[PostgreSQL reference]] |
| pyodbc | SQL Server driver | [[Azure SQL Database]] |
| DuckDB | Query files directly | [[DuckDB]] |

## Related

[[SQL]] · [[SQL fundamentals]] · [[06 — DATABASES]] · [[Data modeling]]
