---
tags: [moc, k8s]
---

# 17 — KUBERNETES

> Running containers across many machines, and keeping them running.

**Why it matters:** Docker runs a container on one machine. Kubernetes runs containers across many and repairs them when they fail.

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

---

> **Already built.** [[Kubernetes and AKS]] — including whether you actually need it.

## What problem does it solve?

You have 40 containers across 6 machines. One machine dies. Who notices? Who restarts them, and where? How do the survivors find each other when IPs change? How do you deploy a new version without downtime?

Kubernetes is a **control loop** answering all of that: compare desired state to actual state, act to close the gap, forever.

## Objects

Cluster · node · **pod** · container · **deployment** · ReplicaSet · **service** · ingress · ConfigMap · secret · namespace · persistent volume · StatefulSet · **Job** · **CronJob** · scheduling · autoscaling · health checks · rolling deployments · rollbacks

## The two things that cause most incidents

1. **Resource requests and limits** — omitting them is worse than guessing
2. **Liveness vs readiness** — never check a dependency in liveness, or one DB blip restarts every pod

## Do you need it?

Probably not. Azure Container Apps *is* Kubernetes with the complexity hidden. See [[Kubernetes and AKS]].

## Related
[[16 — DOCKER]] · [[04 — COMPUTER SCIENCE]] · [[19 — CLOUD]] · [[18 — TERRAFORM]]
