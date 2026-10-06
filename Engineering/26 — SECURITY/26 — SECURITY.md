---
tags: [moc, security]
---

# 26 — SECURITY

> Assuming someone hostile is reading.

**Why it matters:** Security isn't a feature you add. It's a set of defaults you either chose or didn't.

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

> **The code, the commands and the checklist: [[Security in practice]].** This page is the map; that one is the work.

---

## Identity

**Authentication** — who are you? · **Authorisation** — what may you do? · IAM · OAuth2 · JWT · sessions

> These are different. 401 = we don't know who you are. 403 = we know, and no.

## Cryptography

TLS · encryption at rest vs in transit · **hashing vs encryption** (hashing is one-way) · password storage (bcrypt/argon2, never MD5) · key management

## Secrets

Never in code · never in images (`docker history` reveals them) · never in git (rotate if committed — deleting doesn't help) · use Key Vault / Secrets Manager · **prefer managed identity so no secret exists at all** ([[Azure fundamentals]])

## Network

Firewalls · private endpoints · VNet/VPC · least-privilege network rules

## Application

Input validation ([[FastAPI fundamentals]] — Pydantic) · **SQL injection** (parameterised queries, never f-strings) · CORS · rate limiting · never returning stack traces

## Containers and Kubernetes

Non-root users · image scanning · **Kubernetes Secrets are base64, not encrypted** · pod security · network policies

## The principle

> **Least privilege.** Smallest role, smallest scope, shortest lifetime. Start narrow and widen deliberately — never start wide.

## Related
[[Security in practice]] · [[04 — COMPUTER SCIENCE]] · [[Azure fundamentals]] · [[17 — KUBERNETES]] · [[CI-CD pipelines]]
