---
tags: [project, systems, scraping, idea]
status: idea
---

# B2B webscraping tool

Index: [[App projects]] — Systems

---

## The idea

Collect structured business data from the web at scale — company details, contacts, pricing — and serve it through an API.

*(Fill in the detail — this note exists so the link resolves.)*

## Likely stack

| Layer | Option | Note |
|---|---|---|
| Scraper | Rust (`reqwest` + `scraper`) | [[Scraper]] · [[reqwest]] · [[Rust for this stack]] |
| Queue | Kafka or Postgres | [[Kafka]] |
| Storage | Postgres | [[PostgreSQL reference]] |
| Parsing | Structured output from an LLM | [[LLM and GenAI track]] |
| API | Actix or FastAPI | [[Actix Web]] · [[FastAPI fundamentals]] |
| Scheduling | Cron / Airflow | [[Adjacent tools you will meet]] |

## The engineering problems

| Problem | Where the answer is |
|---|---|
| Rate limiting and politeness | [[Networking reference]] — respect `robots.txt`, back off |
| Pages change and break parsers | [[Testing and CI-CD]] — test against saved fixtures |
| Deduplication | [[Data modeling]] — decide the grain and enforce it |
| Idempotent re-runs | [[Unity Catalog and orchestration]] |
| Extracting messy HTML into structure | [[LLM and GenAI track]] — structured outputs |

> **⚖️ Check the legal position before building.** Terms of service, `robots.txt`, GDPR for personal data, and database rights all apply. Scraping public pages is not automatically permitted, and B2B contact data is personal data under UK GDPR.

## Related

[[App projects]] · [[Rust for this stack]] · [[Actix Web]] · [[LLM and GenAI track]]
