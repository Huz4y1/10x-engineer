---
tags: [project, azure, aws, gcp, terraform, cloud]
status: not-started
---

# Project 007 — Cloud Deployment

Index: [[28 — PROJECTS]] · Previous: [[Project 006 — Complete ML Platform]]

---

## What you're building

The **same platform**, deployed to Azure, AWS and GCP, with infrastructure defined in **Terraform** and one codebase that runs on all four targets (including local) by changing configuration only.

## Why this project exists

Anyone can follow one cloud's tutorial. Deploying the same architecture three ways proves you understand the *concepts* rather than one vendor's console.

**Afterwards you will understand:** what genuinely differs between clouds (identity, mostly), what doesn't (nearly everything else), and why infrastructure belongs in version control.

## Concepts used

| Concept | Note | New? |
|---|---|---|
| Cloud service equivalence | [[Cloud comparison dictionary]] | New |
| Infrastructure as Code | [[Terraform]] · [[18 — TERRAFORM]] | New |
| Identity and least privilege | [[Azure fundamentals]] · [[26 — SECURITY]] | New |
| Managed container hosting | [[Deployment patterns]] | Revision |
| CI/CD with OIDC | [[CI-CD pipelines]] | New |

## The translation

| Layer | Local | Azure | AWS | GCP |
|---|---|---|---|---|
| Object storage | SeaweedFS | ADLS Gen2 | S3 | Cloud Storage |
| Database | Postgres | Azure DB for PostgreSQL | RDS | Cloud SQL |
| Streaming | Kafka | Event Hubs | Kinesis | Pub/Sub |
| Spark | Local | Databricks | EMR | Dataproc |
| Containers | k3d | Container Apps / AKS | ECS / EKS | Cloud Run / GKE |
| Registry | local | ACR | ECR | Artifact Registry |
| Secrets | `.env` | Key Vault | Secrets Manager | Secret Manager |
| Monitoring | Prometheus | Azure Monitor | CloudWatch | Cloud Monitoring |

Full detail: [[Cloud comparison dictionary]].

## Build steps

- [ ] **0 — 💸 Budget alerts on all three accounts. First. Before anything.**
  This project can genuinely cost money if something is left running.

- [ ] **1 — One codebase, four targets**
  ```python
  from pydantic_settings import BaseSettings

  class Settings(BaseSettings):
      env: str = "local"                 # local | azure | aws | gcp
      lake_root: str                     # s3a://bronze | abfss://... | s3://... | gs://...
      database_url: str
      mlflow_tracking_uri: str
      model_config = {"env_file": ".env"}
  ```
  > **Only the edges know where they're running.** Transformations, models, API and dashboard are byte-identical across all four. If you find yourself writing `if env == "azure"` deep in business logic, the abstraction is in the wrong place.

- [ ] **2 — Terraform, structured for reuse**
  ```
  infra/
  ├── modules/
  │   ├── storage/     # bucket + lifecycle + access
  │   ├── database/    # postgres + firewall
  │   └── compute/     # container hosting
  └── envs/
      ├── azure/main.tf
      ├── aws/main.tf
      └── gcp/main.tf
  ```
  ```bash
  cd infra/envs/azure
  terraform init
  terraform plan -out=tfplan      # <- READ THIS
  terraform apply tfplan
  ```

- [ ] **3 — Azure first** (you already know it from [[Data Engineering]])
  ACR, Container Apps, Azure DB for PostgreSQL, ADLS Gen2, Key Vault, managed identity.
  ```bash
  az containerapp create --name rul-api --resource-group ml-rg \
    --environment ml-env --image $ACR/rul-api:v1 \
    --target-port 8000 --ingress external \
    --min-replicas 0 --max-replicas 3
  ```

- [ ] **4 — AWS**
  ECR, ECS Fargate (or App Runner), RDS, S3, Secrets Manager, IAM task roles.
  > **The IAM model is genuinely different.** Roles attach to *tasks*, trust policies decide who may assume them. Budget real time for this — it's the hardest part of the project.

- [ ] **5 — GCP**
  Artifact Registry, Cloud Run, Cloud SQL, Cloud Storage, Secret Manager, Workload Identity.
  > Cloud Run is the closest thing to Container Apps — scale-to-zero, one command, HTTPS included.

- [ ] **6 — CI/CD with no stored secrets**
  ```yaml
  permissions: {id-token: write, contents: read}
  steps:
    - uses: azure/login@v2
      with: {client-id: ..., tenant-id: ..., subscription-id: ...}
  ```
  > **OIDC federated credentials** mean GitHub proves its identity and receives a short-lived token. No client secret exists to leak. All three clouds support this and it is strictly better than storing keys ([[CI-CD pipelines]]).

- [ ] **7 — Compare them honestly**
  Record for each: time to deploy, monthly cost at your volume, cold-start latency, developer experience, what surprised you.

- [ ] **8 — Destroy everything**
  ```bash
  terraform destroy
  ```
  **Done when:** all three accounts show zero running resources and you have the cost report.

## Checkpoints

- [ ] Three public URLs, all serving the same predictions
- [ ] Zero code differences between them — only config
- [ ] All infrastructure in Terraform, in Git, reviewable
- [ ] No long-lived cloud credentials anywhere
- [ ] A written comparison table with real numbers
- [ ] **Everything destroyed, and the final bill is what you expected**

## What will go wrong

| Symptom | Cause | Fix |
|---|---|---|
| `403` on blob/object read | Missing **data-plane** role | Azure: `Storage Blob Data Contributor` |
| Container can't reach the database | Firewall / security group | Allow the compute service |
| OIDC login fails | Subject doesn't match repo+branch exactly | Fix the federated credential |
| Terraform wants to destroy the DB | A `-/+` change (immutable field) | Read the plan; use `lifecycle` blocks |
| Image pull denied | Registry role missing | Grant AcrPull / ECR read / AR reader |
| Surprise bill | Something left running | `terraform destroy`; budget alerts |
| Works on Azure, fails on AWS | Hardcoded paths or dialect | Config-driven; standard SQL |

## Stretch goals

- [ ] One GitHub Actions workflow deploying to all three in parallel
- [ ] Measure and chart cold-start latency across the three
- [ ] Cost-per-1000-predictions comparison
- [ ] `terraform plan` posted automatically as a PR comment

## What you learned

*Fill in afterwards.*

## Next project

[[Project 008 — Physical Intelligent Engine Monitor]]
