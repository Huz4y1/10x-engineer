---
tags: [moc, mle]
---

# 13 — ML ENGINEERING

> Turning a model into a system.

**Why it matters:** A trained model in a notebook has no value. ML engineering is everything between that notebook and something people rely on.

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

---

> Covered in depth across [[The playbook]] and the [[Data Engineering]] section.

## The lifecycle

[[Problem framing]] then [[Choosing your approach]] then training then [[MLflow experiment tracking]] then [[Model export and serving]] then [[Deployment patterns]] then [[Monitoring and iteration]]

## Topics

Experiment tracking · dataset versioning · model versioning · model registry · reproducibility · training pipelines · evaluation · deployment · batch vs real-time inference · monitoring · data drift · concept drift · retraining · rollbacks · A/B testing

## The three things that version independently

```
CODE          DATA          MODEL
```

Change any one and behaviour changes. Reproducing a result means pinning all three. That's the whole of [[14 — MLOPS]] in one line.

## The serving decision

**Batch by default.** Precompute, store, look up. Real-time only when you can't know the input in advance. See [[Deployment patterns]].

## Related
[[14 — MLOPS]] · [[15 — FASTAPI]] · [[10 — PYTORCH]]
