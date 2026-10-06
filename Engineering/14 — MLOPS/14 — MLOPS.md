---
tags: [moc, mlops]
---

# 14 — MLOPS

> Keeping models working after they're deployed.

**Why it matters:** Ordinary software fails loudly. Models fail silently — they keep returning confident answers as they slowly become wrong.

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

---

> **Already built.** See [[MLOps and CI-CD]], [[Azure ML and the MLOps stack]], [[Monitoring and iteration]].

## MLflow

Tracking · runs · experiments · artifacts · parameters · metrics · models · **model registry**

Full note: [[MLflow experiment tracking]]

## The gates that don't exist in normal DevOps

| Gate | Question |
|---|---|
| **Data validation** | Is the input sane? |
| **Model quality** | Does it beat the current champion? |
| **Drift** | Is the world still the same? |

## Monitoring layers

1. Operational — is it up?
2. Input drift — does the data look like training data? (PSI)
3. Prediction drift — do outputs look normal?
4. Quality — are predictions actually right? *(arrives late)*

> **Prediction freshness is the alert everyone forgets.** Everything green, all jobs succeeded, data eleven days old.

## Related
[[13 — ML ENGINEERING]] · [[CI-CD pipelines]] · [[25 — OBSERVABILITY]] · [[Observability for data and ML pipelines]]
