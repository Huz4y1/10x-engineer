---
tags: [roadmap, tracker]
---

# Progress tracker

Update `Status` as you go: `Not started` → `In progress` → `Done`.

Hub: [[Data Engineering]]

| # | Stage | Key notes | Status | Target date |
|---|---|---|---|---|
| 1 | Foundations | [[SQL fundamentals]], [[Dev environment - Git, Docker, CLI]] | Not started | |
| 2 | Storage & databases | [[Azure fundamentals]], [[ADLS Gen2]], [[Azure SQL Database]] | Not started | |
| 3 | Distributed processing | [[PySpark core]], [[Databricks and Delta Lake]], [[Unity Catalog and orchestration]] | Not started | |
| 4 | Serving layer | [[FastAPI fundamentals]], [[FastAPI data and deployment]] | Not started | |
| 5 | Dashboards | [[Streamlit vs Django]] | Not started | |
| 6 | ML & deep learning | [[Tensors, autograd and the training loop]] … [[Model export and serving]] | Not started | |
| 7 | Engineering practices | [[Testing and CI-CD]], [[Data modeling]], [[Adjacent tools you will meet]] | Not started | |
| 8 | Containers & deployment | [[Docker deep dive]], [[Running the whole stack locally]], [[Kubernetes and AKS]] | Not started | |
| 9 | MLOps & CI-CD | [[CI-CD pipelines]], [[Azure ML and the MLOps stack]], [[Observability for data and ML pipelines]] | Not started | |
| 10 | Systems performance *(optional)* | [[When to leave Python]], [[CUDA and GPU programming]], [[C++ for this stack]], [[Rust for this stack]] | Not started | |
| 11 | Certifications | [[Certification map]] | Not started | |
| 12 | Capstone | [[Capstone overview and architecture]], [[Capstone build guide]] | Not started | |
| 13 | Idea to production *(reference)* | [[The playbook]], [[Problem framing]], [[Choosing your approach]], [[LLM and GenAI track]], [[Deployment patterns]], [[Monitoring and iteration]] | Not started | |

## Capstone milestones

- [ ] **Part 0** — budget alert set, repo structure, Azure resources provisioned
- [ ] **Part 1** — raw → bronze → silver → gold in Delta, gold tables in Azure SQL, orchestrated and idempotent
- [ ] **Part 2** — both models trained, baselines beaten, tracked in MLflow, exported to ONNX and verified
- [ ] **Part 3** — FastAPI serving, Streamlit dashboard, containerised, deployed, CI green
- [ ] **Part 4** — README, screenshot, `DECISIONS.md`, two-minute architecture explanation

## Optional depth

- [ ] Run the whole pipeline locally with no Azure ([[Running the whole stack locally]])
- [ ] Rewrite one hot function in Rust with PyO3 ([[Rust for this stack]])
- [ ] Serve the ONNX model from C++ and compare image size ([[C++ for this stack]])
- [ ] Write and profile a CUDA kernel ([[CUDA and GPU programming]])
- [ ] Wire OpenTelemetry through the whole pipeline ([[Observability for data and ML pipelines]])
