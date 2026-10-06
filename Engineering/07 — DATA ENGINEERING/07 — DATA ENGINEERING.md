---
tags: [moc, dataeng]
---

# 07 — DATA ENGINEERING

> Getting data from where it is to where it's useful, reliably.

**Why it matters:** Models are the easy part. Getting correct, fresh, well-shaped data to them is most of the work in any real ML system.

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

---

> **This section is already built in depth.** Go to **[[Data Engineering]]** — 43 notes covering the full Azure/Databricks stack, plus a local no-cloud mirror.

## Complete reference

**[[PySpark reference]]** — the full API: reading/writing, schemas, **investigating data**, **cleaning and preprocessing**, string/date/math functions, aggregation, windows, joins, reshaping, UDFs, SQL, performance, streaming, MLlib.

## Core concepts

ETL vs ELT · batch vs stream · pipelines · warehouses · lakes · **lakehouses** · data quality · schema evolution · lineage · governance · partitioning · serialization · compression

## Formats

| Format | Use |
|---|---|
| **Parquet** | Columnar, compressed. Default for analytics. |
| **Delta** | Parquet + transaction log = ACID ([[Databricks and Delta Lake]]) |
| CSV | Only at system edges |
| JSON | APIs, config, semi-structured |
| Avro | Row-based, schema evolution, Kafka |
| Protobuf | Compact binary, gRPC |

## Streaming
[[Kafka]] · [[MQTT]] · [[Adjacent tools you will meet]]

## Processing
[[PySpark core]] · [[Databricks and Delta Lake]] · [[Unity Catalog and orchestration]]

## Storage
[[ADLS Gen2]] · [[Cloud comparison dictionary]]

> **Storage notes live in [[06 — DATABASES]]:** [[Object storage]] (S3, ADLS Gen2, SeaweedFS) and [[PostgreSQL]].

## Modelling
[[Data modeling]] — star schemas, grain, slowly changing dimensions

## Related
[[06 — DATABASES]] · [[08 — MACHINE LEARNING]] · [[14 — MLOPS]]
