---
tags: [moc, docker, containers, deployment]
status: not-started
---

# Containers and deployment

How code gets from your laptop to something other people can use. Part of [[Data Engineering]].

The short version lives in [[Dev environment - Git, Docker, CLI]]; these are the deep versions.

## Notes

[[Docker deep dive]] — images, layers, multi-stage builds, networking, volumes, Compose, and the data/ML-specific traps

[[Running the whole stack locally]] — the entire pipeline with no Azure account: SeaweedFS, Postgres, Spark, Delta, MLflow, Airflow

[[Kubernetes and AKS]] — the orchestrator you'll meet in big companies, and whether you actually need it

## The deployment ladder

Stop at the first rung that works.

| Rung | Option | Use when |
|---|---|---|
| 1 | **Docker Compose on one box** | Local dev, demos, small internal tools |
| 2 | **Azure Container Apps** | ✅ **The default for this stack.** HTTP services, scale-to-zero, revisions and traffic splitting free. |
| 3 | Azure App Service | A single web app, no container expertise wanted |
| 4 | **AKS / Kubernetes** | Dozens of services, custom networking, a team to run it |

> Container Apps *is* Kubernetes underneath, with the complexity hidden. The capstone deploys there, and that's the correct choice — see [[FastAPI data and deployment]].

## Where each piece of the stack runs

| Component | Deployed as |
|---|---|
| Bronze → silver → gold pipeline | Databricks Workflow ([[Unity Catalog and orchestration]]) |
| Model training | Databricks job, or a GPU container |
| Nightly batch scoring | Databricks job → Delta → Azure SQL |
| FastAPI | Container on Azure Container Apps |
| Streamlit dashboard | Container on Azure Container Apps |
| Monitoring pipeline | Databricks job over the prediction log |

## Related

[[Deployment patterns]] — batch vs. real-time vs. streaming, and rollout strategies

[[MLOps and CI-CD]] — automating all of this

[[Observability for data and ML pipelines]] — knowing it's working

[[Git-GitHub]] — the version control underneath it all
