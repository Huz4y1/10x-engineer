---
tags: [moc, databases]
---

# 06 — DATABASES

> Where state lives.

**Why it matters:** Every system stores something. The database is usually the hardest thing to change later and the first thing to become a bottleneck.

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

---

## What's in this section

```
06 — DATABASES
├── PostgreSQL/          rows and tables - the database for your app
│   ├── PostgreSQL                     ← start here: reading order
│   ├── pgAdmin 4                      the point-and-click app
│   ├── PostgreSQL data types          every type, and which to use
│   ├── PostgreSQL SQL                 what Postgres adds to standard SQL
│   ├── PostgreSQL with SQLAlchemy     using it from Python
│   ├── Azure Database for PostgreSQL  on Azure
│   ├── AWS RDS for PostgreSQL         on AWS
│   └── PostgreSQL reference           quick lookup
├── Object storage/      files - data lakes, backups, models
│   ├── Object storage                 ← start here: S3 vs ADLS vs SeaweedFS
│   ├── AWS S3                         on AWS
│   ├── ADLS Gen2                      on Azure
│   ├── SeaweedFS                      on your laptop - what it is
│   └── Using SeaweedFS                on your laptop - hands-on on WSL
├── SQL fundamentals     standard SQL, in depth
├── Data modeling        designing tables and star schemas
└── Azure SQL Database   Microsoft's SQL Server in the cloud
```

## PostgreSQL

**Start at [[PostgreSQL]]** — it gives the reading order.

| Need | Note |
|---|---|
| Click around, run SQL, import a CSV, back up | **[[pgAdmin 4]]** |
| Which type for a column | **[[PostgreSQL data types]]** |
| Upserts, JSONB, `DISTINCT ON`, full-text search, indexes, permissions | **[[PostgreSQL SQL]]** |
| Connect from Python, bulk-load, async | **[[PostgreSQL with SQLAlchemy]]** |
| Run it on Azure | **[[Azure Database for PostgreSQL]]** |
| Run it on AWS | **[[AWS RDS for PostgreSQL]]** |
| Look up a `psql` command or admin query | **[[PostgreSQL reference]]** |

## Object storage — files, not rows

**Start at [[Object storage]]** — what it is, and how the three compare.

| Where | Note |
|---|---|
| AWS | **[[AWS S3]]** |
| Azure | **[[ADLS Gen2]]** |
| Your laptop / WSL | **[[SeaweedFS]]** · **[[Using SeaweedFS]]** |

## SQLite — the database that is just a file

No server, no install, built into Python. Best for one-machine apps, prototypes and tools: **[[Flask SQLite stack]]**, with worked SQL in **[[Flask SQLite - book tracker]]**.

## SQL
[[SQL]] (basics) · [[SQL fundamentals]] (deep — window functions, execution order, query plans, indexes)

SELECT · INSERT · UPDATE · DELETE · JOINs · GROUP BY · HAVING · ORDER BY · aggregations · subqueries · CTEs · window functions · transactions · ACID · constraints · primary keys · foreign keys · indexes · query optimisation

## Inside PostgreSQL

Architecture · tables · schemas · indexes · transactions · **MVCC** · locks · JSON/JSONB · extensions (pgvector, PostGIS) · replication · backups · performance · connection pooling

> **MVCC in one line:** Postgres keeps multiple versions of a row so readers never block writers. That's why it handles concurrency well, and why `VACUUM` exists to clean up old versions.

## NoSQL

| Type | Example | Good at |
|---|---|---|
| Key-value | Redis | Caching, sessions |
| Document | MongoDB | Flexible schemas |
| Column | Cassandra | Huge writes, time series |
| Graph | Neo4j | Relationships, traversal |

## SQL or NoSQL?

| Use SQL when | Use NoSQL when |
|---|---|
| Data has relationships | Documents are self-contained |
| You need transactions | Eventual consistency is fine |
| Queries are varied/unknown | Access pattern is fixed and known |
| Correctness matters most | Scale/latency matters most |

> **Default to PostgreSQL.** It does JSON, full-text search, vectors and time-series adequately. Reach for something specialised when you've measured that Postgres can't cope — which is later than most people think.

## Also
[[Azure SQL Database]] · [[Data modeling]] (star schemas, SCD) · [[Cloud comparison dictionary]]

## Related
[[PostgreSQL]] · [[Object storage]] · [[07 — DATA ENGINEERING]] · [[SQL fundamentals]]
