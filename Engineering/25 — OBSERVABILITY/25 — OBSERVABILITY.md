---
tags: [moc, observability]
---

# 25 — OBSERVABILITY

> Knowing what your system is doing, and why.

**Why it matters:** Monitoring answers questions you knew to ask. Observability answers ones you didn't.

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

---

> Existing notes: [[Observability]] · [[OpenTelemetry]] · [[Grafana]] · [[Prometheus-Mimir]] · [[Loki]] · [[Tempo]] · [[Clickhouse]] · [[Alloy or Otel Collector]]
> For data/ML pipelines specifically: **[[Observability for data and ML pipelines]]**

## The pillars

| Pillar | Question | Tool |
|---|---|---|
| **Logs** | What happened? | [[Loki]] |
| **Metrics** | How much, how fast? | [[Prometheus-Mimir]] |
| **Traces** | Where did the time go? | [[Tempo]] |
| **Freshness** | Is the data recent? | Custom |
| **Quality** | Is the data right? | Assertions |

The last two are data-specific and are the ones that catch silent failures.

## Concepts

Monitoring · alerting · health checks · **SLIs / SLOs / SLAs** · RED method (Rate, Errors, Duration) · USE method

## The rule about alerts

> **Every alert must name an action.** An alert nobody acts on trains everyone to ignore all alerts, including the real one.

## Cloud
Azure Monitor · CloudWatch · Cloud Monitoring — see [[Cloud comparison dictionary]]

## Related
[[05 — SOFTWARE ENGINEERING]] · [[14 — MLOPS]] · [[30 — TROUBLESHOOTING]]
