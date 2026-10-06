---
tags: [object-storage, s3, adls, seaweedfs, moc, databases]
---

# Object storage

**Where files live in a data platform** — Parquet tables, CSV drops, images, model weights, backups. Three services here do the same job in three places:

| Where | Service | Note |
|---|---|---|
| **AWS** | Amazon S3 | **[[AWS S3]]** |
| **Azure** | Azure Data Lake Storage Gen2 | **[[ADLS Gen2]]** |
| **Your laptop / own servers** | SeaweedFS (S3-compatible) | **[[SeaweedFS]]** (what it is) · **[[Using SeaweedFS]]** (hands-on on WSL) |

Section: [[06 — DATABASES]] · Every cloud equivalent: [[Cloud comparison dictionary]]

---

## Database or object storage?

| | A database ([[PostgreSQL]]) | Object storage |
|---|---|---|
| Stores | **Rows** — look up, change, delete one at a time | **Whole files** — write once, read many times |
| Good at | "Get order 42", "update this customer" | "Read all of March's sales", "store this model" |
| Changing part of something | ✅ Update one row | ❌ Replace the whole file |
| Size | Gigabytes to a few terabytes | Practically unlimited |
| Cost per GB | Higher | Very low |
| Query language | SQL | None — you read files with Spark, pandas, DuckDB |

> **Most real systems use both.** The app's live data is in Postgres; history, raw files and analytics tables are in object storage — often as Parquet or Delta files that Spark reads ([[Databricks and Delta Lake]]). A database row can point at a file: `products.image_key = 'images/42.png'`.

---

## The same idea, three names

All three store **objects** (files) in **containers**, addressed by a **key** (the full path):

| Idea | S3 / SeaweedFS | ADLS Gen2 |
|---|---|---|
| Top-level container | **Bucket** | **Container** (inside a *storage account*) |
| A file | Object | Blob / file |
| Its name | Key | Path |
| Real folders? | ❌ Just `/` in the name | ✅ With the *hierarchical namespace* on — renaming a folder is instant |
| Access control | IAM policies, bucket policies | Azure RBAC roles + folder ACLs |

---

## The address, in every tool

The same file needs a different prefix depending on what's reading it — this trips everyone up:

| Tool | AWS S3 | SeaweedFS | ADLS Gen2 |
|---|---|---|---|
| AWS CLI | `s3://bucket/key` | `s3://bucket/key` + `--endpoint-url` | — (use `az storage` / `azcopy`) |
| boto3 | `Bucket=`, `Key=` | same + `endpoint_url=` | — |
| **pandas** (s3fs / adlfs) | `s3://bucket/key` | `s3://bucket/key` + endpoint | `abfs://container@account.dfs.core.windows.net/path` |
| **PySpark** | **`s3a://`**`bucket/key` | `s3a://` + endpoint + path-style | **`abfss://`**`container@account.dfs.core.windows.net/path` |
| Databricks on AWS | `s3://` works | — | — |

> ⚠️ **Spark uses `s3a://`, not `s3://`** (except on EMR and Databricks, which set it up for you). `s3://` in plain Spark gives *No FileSystem for scheme "s3"* ([[Using SeaweedFS]]).

> ⚠️ **ADLS uses `abfss://` with two `s`s** — the second `s` means encrypted (TLS). And the host ends in **`.dfs.core.windows.net`**, not `.blob.` ([[ADLS Gen2]]).

---

## Which one should I use?

| Situation | Use |
|---|---|
| Learning, local dev, no cloud bill | **[[SeaweedFS]]** — runs in Docker on WSL |
| Your stack is on Azure (Databricks, Synapse, Fabric) | **[[ADLS Gen2]]** |
| Your stack is on AWS | **[[AWS S3]]** |
| Code that must move between them | Write against the **S3 API**, then switch the endpoint — SeaweedFS → S3 is a config change |

> **SeaweedFS is a free rehearsal for S3.** Everything you learn in [[Using SeaweedFS]] — buckets, keys, `s3a://`, the medallion layout — transfers directly to AWS.

---

## The layout that works everywhere — bronze, silver, gold

```
bronze/   raw data, exactly as it arrived        (never edited - you can always rebuild from it)
silver/   cleaned, typed, de-duplicated
gold/     aggregated, ready for dashboards and models
```

Explained with real examples in [[Using SeaweedFS]] and [[ADLS Gen2]]; the modelling side in [[Data modeling]].

---

## Rules that apply to all three

1. **Keep containers private.** Share single files with temporary links (S3 presigned URLs, Azure SAS tokens), never by making a bucket public ([[Security in practice]])
2. **Let programs log in with roles, not keys** — IAM roles on AWS, managed identities on Azure
3. **Store Parquet, not CSV**, for anything you'll query more than once — smaller, typed, far faster
4. **Avoid millions of tiny files** — aim for roughly 100 MB–1 GB per file
5. **Set lifecycle rules** so old data moves to cheaper storage or is deleted automatically
6. **Turn on versioning or soft delete** for anything you can't recreate

## Related

[[06 — DATABASES]] · [[AWS S3]] · [[ADLS Gen2]] · [[SeaweedFS]] · [[Using SeaweedFS]] · [[PostgreSQL]] · [[Databricks and Delta Lake]] · [[Cloud comparison dictionary]] · [[PySpark reference]] · [[Pipeline setup - overview]]
