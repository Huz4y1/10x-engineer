---
tags: [monitoring, drift, mlops, production, playbook]
status: not-started
---

# Monitoring and iteration

> **What this is:** phase 8 of [[The playbook]] — knowing your model is still working, and what to do when it isn't.
> **Why you care:** models decay silently. A deployed model nobody watches is a liability that looks like an asset.

For the tooling — OpenTelemetry, Grafana, Prometheus, Loki — see [[Observability for data and ML pipelines]] and [[Observability]].

---

## The idea in plain English

Ordinary software either works or throws an error. A model is different: **it keeps returning confident answers as it becomes wrong.**

Nothing crashes. No exception. The dashboard is still green. The numbers are just gradually less true, and unless you're specifically watching for it, you find out from an angry stakeholder six months later.

> **The three-part truth of ML in production:** the world changes, your data changes with it, and your model doesn't. Monitoring is how you notice the gap opening.

---

## The four layers

Watch them in this order — the top one pages you, the bottom one is the truth.

| Layer | Question | Latency of the signal |
|---|---|---|
| **1 · Operational** | Is it up and fast? | Instant |
| **2 · Input drift** | Does the incoming data look like training data? | Instant |
| **3 · Prediction drift** | Do the outputs look like they used to? | Instant |
| **4 · Quality** | Are the predictions actually right? | Days to months — sometimes never |

> **The painful asymmetry:** the layer you care about most (4) arrives last, and sometimes never arrives at all. So you monitor the proxies (2 and 3) continuously and treat them as an early warning system, then confirm with 4 when the truth eventually shows up.

---

## Layer 1 — Operational

Same as any service. If you have [[Prometheus-Mimir]] and [[Grafana]] running, this is free.

| Metric | Alert when |
|---|---|
| Error rate (5xx) | > 1% for 5 minutes |
| p95 latency | > your SLA |
| Throughput | Drops to zero (nobody's calling it — is the caller broken?) |
| Batch job success | **Any failure** |
| Prediction freshness | `max(scored_at)` older than expected |

> **`prediction_freshness` is the ML-specific one people forget.** The API is up, healthy, fast — and serving predictions from a job that stopped running eleven days ago. Everything looks perfect. Alert on the age of your newest prediction.

```python
freshness_hours = (now - max_scored_at).total_seconds() / 3600
assert freshness_hours < 30, f"Predictions are {freshness_hours:.0f}h old"
```

---

## Layer 2 — Input drift

**Are the features arriving now distributed like the ones you trained on?**

This is your earliest warning, and it's available immediately with no ground truth required.

### What to track

For each important feature, per day: mean, standard deviation, min/max, null rate, and (for categoricals) the category distribution.

```python
from pyspark.sql import functions as F

# a nightly Databricks job over your prediction log
daily = (spark.table("retail.monitoring.prediction_log")
    .filter(F.col("date") == run_date)
    .agg(*[F.avg(c).alias(f"{c}_mean") for c in FEATURES],
         *[F.stddev(c).alias(f"{c}_std") for c in FEATURES],
         *[(F.count(F.when(F.col(c).isNull(), c)) / F.count("*")).alias(f"{c}_null_rate")
           for c in FEATURES]))
daily.write.format("delta").mode("append").saveAsTable("retail.monitoring.feature_stats")
```

### Measuring drift properly

**Population Stability Index (PSI)** is the industry standard — bucket the feature and compare proportions:

```python
import numpy as np

def psi(expected, actual, buckets=10):
    breaks = np.percentile(expected, np.linspace(0, 100, buckets + 1))
    breaks[0], breaks[-1] = -np.inf, np.inf
    e = np.histogram(expected, breaks)[0] / len(expected)
    a = np.histogram(actual,   breaks)[0] / len(actual)
    e, a = np.clip(e, 1e-6, None), np.clip(a, 1e-6, None)
    return float(np.sum((a - e) * np.log(a / e)))
```

| PSI | Meaning | Action |
|---|---|---|
| < 0.1 | No meaningful shift | Nothing |
| 0.1 – 0.25 | Moderate shift | Investigate |
| > 0.25 | Major shift | **Retrain, and find out why** |

> **A sudden PSI spike is almost never the world changing — it's a pipeline bug.** An upstream schema change, a unit switch (pence to pounds), a join that started fanning out, a currency or timezone change. **Check the pipeline before you retrain.** Retraining on broken data bakes the bug in.

### Null rate — the cheapest, highest-value alarm

A feature that was 2% null and is now 60% null means an upstream system changed. That's a five-line check that catches an enormous class of silent failures.

---

## Layer 3 — Prediction drift

Same idea, applied to the outputs.

```sql
SELECT
    CAST(predicted_at AS DATE) AS day,
    COUNT(*)                   AS n,
    AVG(prediction)            AS mean_pred,
    STDEV(prediction)          AS std_pred,
    SUM(CASE WHEN segment = 'champion' THEN 1 ELSE 0 END) * 1.0 / COUNT(*) AS pct_champion
FROM prediction_log
GROUP BY CAST(predicted_at AS DATE)
ORDER BY day;
```

**What a shift means:** either the input population genuinely changed, or something upstream broke, or somebody deployed a new model and didn't tell you.

> **This is also your best canary for accidental deploys.** A step-change in the output distribution on a Tuesday afternoon, with no corresponding input change, is almost always a model version that shipped without anyone noticing.

---

## Layer 4 — Actual quality

The real answer, and the slow one.

### Getting ground truth

| Source | Delay |
|---|---|
| Natural outcome (did they churn? was the forecast right?) | Days to months |
| Human review of a sample | Days, and it costs money |
| User feedback (thumbs up/down, corrections) | Immediate but biased |
| Downstream signal (did they click, did the offer convert?) | Hours to days |

```sql
-- join predictions to outcomes once they exist
SELECT p.model_version,
       AVG(ABS(p.prediction - a.actual_revenue)) AS mae,
       COUNT(*) AS n
FROM prediction_log p
JOIN actuals a ON p.entity_id = a.entity_id AND p.target_date = a.date
WHERE p.predicted_at >= DATEADD(day, -30, GETDATE())
GROUP BY p.model_version;
```

> **Build the prediction log from day one, even before you can join it to outcomes.** You cannot retroactively find out what your model predicted last March. A log with `entity_id`, `predicted_at`, `prediction`, `model_version`, and the input features is the single most valuable thing you can build for the future of the project — and it's one table.

### Always slice

Overall metrics hide the failures that matter. Break down by segment, region, customer size, and time:

```sql
GROUP BY p.model_version, c.country, c.segment
```

> A model that's 92% accurate overall and 45% accurate on your highest-value customers is a broken model with a good average. **The average is the least informative number you have.**

---

## The prediction log

One table. Write to it on every prediction. It powers layers 2, 3 and 4.

```sql
CREATE TABLE prediction_log (
    prediction_id   BIGINT IDENTITY(1,1) PRIMARY KEY,
    entity_id       NVARCHAR(50)  NOT NULL,   -- customer/product it's about
    predicted_at    DATETIME2     NOT NULL,
    target_date     DATE          NULL,       -- what period it's predicting
    prediction      DECIMAL(18,4) NOT NULL,
    confidence      DECIMAL(5,4)  NULL,
    model_name      NVARCHAR(100) NOT NULL,
    model_version   NVARCHAR(20)  NOT NULL,
    mlflow_run_id   NVARCHAR(50)  NULL,       -- traces back to the training run
    features_json   NVARCHAR(MAX) NULL,       -- what it actually saw
    latency_ms      INT           NULL
);
```

> `mlflow_run_id` closes the loop: a suspicious prediction in production traces to the exact training run, which traces to the exact Delta table version ([[MLflow experiment tracking]]). That chain is what lets you answer "why did it say that?" months later.

**Don't write this synchronously in the request path** — use a background task or a queue, so logging can never slow down or break a user request.

---

## When to retrain

| Trigger | Good default |
|---|---|
| **Scheduled** | Monthly or quarterly. Simple, predictable. **Start here.** |
| **Drift-triggered** | PSI > 0.25 on an important feature |
| **Performance-triggered** | Live metric degrades > 10% vs. the validation number |
| **Data-triggered** | A meaningful volume of new labelled data |
| **Event-triggered** | You know the world changed — a pricing change, a new market |

> **Scheduled retraining is underrated.** It's predictable, easy to automate, and easy to reason about. Drift-triggered retraining sounds smarter and adds a whole feedback system that can itself misfire. Start scheduled; add triggers when you have evidence you need them.

### Retraining is a pipeline, not a person

```
new data → retrain → evaluate vs. current champion → register → shadow → canary → promote
```

**With a gate that can say no:**

```python
if new_metrics["mae"] > champion_metrics["mae"] * 0.98:
    raise Exception(f"New model MAE {new_metrics['mae']:.1f} does not beat "
                    f"champion {champion_metrics['mae']:.1f} by enough — not promoting")
```

> **Automatic retraining without an automatic quality gate is a way to automatically deploy a worse model.** The gate is the important half. And the gate must compare against the *current champion*, not against the original baseline.

Automating this is [[MLOps and CI-CD]].

---

## The feedback loop trap

Your model changes the world it's measuring. This is real and it's subtle.

**Example:** you predict churn, so you send those customers an offer, so they don't churn. Your model now looks wrong — it predicted churn that didn't happen. It was right; the intervention worked.

**Worse:** you retrain on that data. The model learns those customers don't churn. It stops flagging them. They churn. The model destroyed its own signal.

| Loop | Mitigation |
|---|---|
| Intervention prevents the outcome | **Hold out a control group** who get no intervention. Evaluate on them. |
| Recommendations shape what's clicked | Inject exploration — some random recommendations |
| Only reviewing what the model flagged | Randomly sample the unflagged too, or you never learn about false negatives |

> **The control group is the honest answer, and it costs something real** — you deliberately don't help some customers. That's a business decision, not a technical one, so raise it early with whoever owns the metric.

---

## The dashboard

One page, four sections, matching the four layers:

1. **Health** — requests/min, error rate, p95 latency, prediction freshness
2. **Input** — PSI per feature over time, null rates
3. **Output** — prediction distribution over time, by segment
4. **Quality** — live metric vs. validation metric, by model version, sliced

Build it in Grafana ([[Grafana]]) if you have the observability stack, or in Streamlit ([[Streamlit vs Django]]) against the same Azure SQL tables everything else uses. The pipeline that feeds it is just another bronze→silver→gold job where the source is your own prediction log.

---

## When it breaks: the runbook

Write this before you need it.

```markdown
## Forecast model — degraded

1. Is the batch job running?         → Databricks Workflows run history
2. Are predictions fresh?            → SELECT MAX(scored_at) FROM predictions
3. Did the inputs change?            → PSI dashboard. Spike = check pipeline FIRST.
4. Did the model change?             → MLflow registry: which version is champion?
5. Is it real degradation?           → live MAE vs. validation MAE, sliced

Mitigations, in order:
  a. Roll back to the previous champion (alias flip — seconds)
  b. Fall back to the naive baseline (always keep it deployed)
  c. Disable the feature, serve the pre-model behaviour
```

> **Keep the naive baseline deployed permanently.** It costs nothing and it means "turn the model off" is a real option that still returns sensible numbers, rather than an outage.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| Model quietly got worse over months | No drift monitoring | Layers 2–3, and a prediction log |
| API healthy, predictions ancient | Batch job silently failing | Alert on job failure **and** prediction freshness |
| PSI spiked overnight | Almost certainly a pipeline bug, not the world | Check upstream schema/units **before** retraining |
| Retrained and it got worse | Trained on broken or drifted-in-a-bad-way data | Quality gate vs. champion; validate the input first |
| Live metric worse than validation | Normal offline/online gap, or leakage in training | Expected — but audit for leakage ([[Problem framing]]) |
| Good overall metric, complaints anyway | Not slicing | Break down by segment |
| Can't tell which model made a prediction | No `model_version` logged | Add it to the log and the response |
| Model appears wrong but intervention worked | Feedback loop | Hold out a control group |
| Never learn about false negatives | Only reviewing what was flagged | Randomly sample unflagged cases |
| No idea what it predicted last quarter | No prediction log | Build it today — you can't backfill it |

---

## Practice checklist

- [ ] Why models fail silently, unlike ordinary software
- [ ] The four layers, and why quality arrives last
- [ ] **Prediction freshness** — the ML-specific alert everyone forgets
- [ ] Input drift, PSI, and the thresholds
- [ ] **A PSI spike is usually a pipeline bug, not the world changing**
- [ ] Null-rate alerting as the cheapest high-value check
- [ ] Prediction drift as a canary for accidental deploys
- [ ] Sources of ground truth and their delays
- [ ] **The prediction log** — build it day one, you can't backfill it
- [ ] Slicing metrics by segment
- [ ] Retraining triggers, and why scheduled is a good default
- [ ] **The quality gate against the current champion**
- [ ] Feedback loops and the control group
- [ ] Writing the runbook before the incident
- [ ] Keeping the naive baseline permanently deployed

## Hands-on

- [ ] Add a `prediction_log` table to the capstone and write to it from every prediction
- [ ] Build the nightly feature-stats job and compute PSI against the training distribution
- [ ] Deliberately shift a feature (multiply by 10) and confirm PSI catches it
- [ ] Add a prediction-freshness alert and trigger it by disabling the job
- [ ] Build the four-section monitoring dashboard
- [ ] Write the runbook for your forecast model
- [ ] Add a quality gate that refuses to promote a model that doesn't beat the champion

## Resources

- [Evidently AI](https://docs.evidentlyai.com/) — open-source drift detection and reports
- [Google: Rules of Machine Learning](https://developers.google.com/machine-learning/guides/rules-of-ml) — Rules 8–14 on monitoring
- [[Observability for data and ML pipelines]] — wiring this into OpenTelemetry and Grafana

## Next

[[Containers and deployment]]
