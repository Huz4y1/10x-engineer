---
tags: [moc, cloud]
---

# 19 — CLOUD

> Four stacks, one architecture.

**Why it matters:** Local, Azure, AWS and GCP implement the same concepts with different names. Learn the concepts and the cloud becomes a config change.

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

---

## Build it end to end

**[[Pipeline setup - overview]]** - what all four guides build, and the order to build it in

| Guide | |
|---|---|
| [[Pipeline setup - Local]] | Your laptop. Free. **Start here.** |
| [[Pipeline setup - Azure]] | |
| [[Pipeline setup - AWS]] | |
| [[Pipeline setup - GCP]] | |

## Start here

**[[Cloud comparison dictionary]]** — the master translation table plus all four pipelines drawn out.

## The four stacks

| Stack | Notes |
|---|---|
| **Local** *(your laptop is the server)* | [[Running the whole stack locally]] — SeaweedFS, Postgres, Spark, Delta, MLflow, Airflow, k3s |
| **Azure** | [[Data Engineering]] — built out in depth |
| **AWS** | EC2, ECS, EKS, RDS, S3, Kinesis, EMR, SageMaker, IoT Core, CloudWatch, IAM |
| **GCP** | Compute Engine, Cloud Run, GKE, Cloud SQL, GCS, Pub/Sub, Dataproc, BigQuery, Vertex AI |

## What actually differs

The middle of every pipeline — Spark, Delta, PyTorch, MLflow, Docker, FastAPI — is **identical**. Only the edges differ: ingest, storage endpoints, **identity**, monitoring.

> **Identity is the hard one.** Entra ID, AWS IAM and GCP IAM have genuinely different models. Everything else ports easily.

## Related
[[18 — TERRAFORM]] · [[17 — KUBERNETES]] · [[Azure fundamentals]] · [[26 — SECURITY]]
