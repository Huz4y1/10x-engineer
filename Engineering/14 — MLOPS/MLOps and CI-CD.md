---
tags: [moc, mlops, cicd, azure]
status: not-started
---

# MLOps and CI-CD

Automating the path from a commit to production, for code **and** for models. Part of [[Data Engineering]].

## Notes

[[CI-CD pipelines]] — GitHub Actions and Azure DevOps, environments and approvals, OIDC auth, deploying notebooks, migrations and infrastructure

[[Azure ML and the MLOps stack]] — model registry, quality gates, feature stores, Azure ML, and the maturity ladder

[[Observability for data and ML pipelines]] — traces, metrics, logs, freshness and quality across the whole pipeline

## The one-line version

> Ship code with CI/CD. Ship models with a **registry and a quality gate**. Know both are still working with **observability**.

## What makes this different from normal DevOps

Three things version independently, not one:

```
   CODE              DATA               MODEL
 (the script)   (what it learned    (the weights that
                    from)             came out)
```

So the pipeline needs three extra gates that a web app never needs:

| Gate | Question | Where |
|---|---|---|
| **Data validation** | Is the input sane? | Before training |
| **Model quality** | Does it beat the current champion? | Before registering |
| **Drift** | Is the world still the same? | Continuously, after deploy |

## Related

[[Testing and CI-CD]] — writing the tests these pipelines run

[[Containers and deployment]] — what gets deployed

[[Deployment patterns]] — shadow, canary, rollback

[[Monitoring and iteration]] — drift, retraining triggers, runbooks

[[MLflow experiment tracking]] — the registry these pipelines write to

[[Unity Catalog and orchestration]] — scheduling and governance

[[Git-GitHub]] — the version control underneath

[[Observability]] — the tooling stack
