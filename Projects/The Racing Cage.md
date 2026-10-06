---
tags: [project, web, idea]
status: idea
---

# The Racing Cage

Index: [[App projects]] — Web apps

---

## The idea

A sim-racing companion: track your lap times, compare setups, and see where you're actually losing time.

*(Fill in the detail — this note exists so the link resolves and the project has a home.)*

## Likely stack

| Layer | Option | Note |
|---|---|---|
| Frontend | Next.js + TypeScript | [[NextJs TypeScript]] |
| Backend | Actix Web | [[Actix Web]] |
| Database | Postgres / Supabase | [[PostgreSQL reference]] · [[Supabase]] |
| Telemetry ingest | UDP from the game | [[Networking reference]] |
| Analysis | Python | [[08 — MACHINE LEARNING]] |

## The interesting engineering problem

Sim racing games broadcast telemetry over **UDP at 60Hz** — that's a real-time stream, not a form submission. Which makes it the same shape as [[Live F1 racing telemetry in C++]] and, at heart, [[Project 010 — Advanced Aerospace Intelligent System]].

**Start by asking:** does this need streaming, or is it a batch job after each session? ([[Deployment patterns]] — usually batch.)

## Related

[[App projects]] · [[Live F1 racing telemetry in C++]] · [[Primary Web App Stack]]
