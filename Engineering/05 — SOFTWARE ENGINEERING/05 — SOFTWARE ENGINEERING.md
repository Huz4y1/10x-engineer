---
tags: [moc, swe]
---

# 05 — SOFTWARE ENGINEERING

> Writing code that other people — including future you — can work with.

**Why it matters:** Anyone can make something work once. Software engineering is making it keep working while five people change it.

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

---

## Version control
[[Git-GitHub]] · [[Dev environment - Git, Docker, CLI]] — branching, rebase vs merge, conflicts, pull requests, code review

## Testing
[[Testing and CI-CD]] — unit, integration, end-to-end, fixtures, mocking, coverage as a diagnostic not a target

## Operating
Debugging · logging · error handling · documentation

## Design
API design · design patterns · SOLID · clean architecture · dependency injection

## Architecture
Monoliths · microservices · event-driven · distributed systems · message queues ([[Kafka]]) · pub/sub · REST · gRPC

## The qualities
Reliability · scalability · maintainability · security ([[26 — SECURITY]])

## When architecture becomes over-engineering

| Symptom | Reality |
|---|---|
| Microservices before 10 engineers | You've bought distributed-systems problems for nothing |
| An interface with one implementation | Delete it |
| A config option nobody changes | Delete it |
| A framework for a two-page app | Delete it |
| Kubernetes for one container | [[Kubernetes and AKS]] says use Container Apps |

> **The test: does this abstraction remove more complexity than it adds?** Usually not. Complexity you added to be flexible is still complexity, and speculative flexibility is the most expensive kind.

## Related
[[04 — COMPUTER SCIENCE]] · [[Testing and CI-CD]] · [[CI-CD pipelines]] · [[15 — FASTAPI]]
