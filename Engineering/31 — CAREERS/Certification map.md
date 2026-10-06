---
tags: [certifications]
status: not-started
---

# Certification Map

> **What this is:** which certifications match which stage of this roadmap, and how to think about them.
> **Why you care:** none of these are required to be good at this stack. Treat them as checkpoints that validate what you've already learned, not a substitute for the hands-on work.

> ⚠️ **Exam names, formats, prices and retirement dates change more often than the underlying skills do.** Everything specific below should be checked against the official page before you book. The *mapping* to roadmap stages is the durable part.

---

## Do certifications actually help?

Honest answer: **it depends entirely on where you're applying.**

| Situation | Worth it? |
|---|---|
| Applying to Microsoft partners / consultancies | **Yes.** They're often contractually required to maintain certified staff, so it's a genuine hiring filter. |
| Corporate / enterprise IT, big non-tech companies | **Often.** HR screens on them. |
| Your CV lacks commercial experience in the stack | **Yes.** It's third-party evidence you know something. |
| Startups, product companies, tech-first firms | **Rarely.** They'll look at your GitHub and interview you. |
| Instead of building the capstone | **No. Never.** |

> **The ordering that matters:** build the project, then certify. A certification with no project reads as "did a course." A project with a certification reads as "can do the work, and here's independent confirmation." A project *without* a certification is still a strong position. A certification without a project is the weak one.

**The genuine benefit** isn't the badge — it's that a syllabus forces you to cover things you'd otherwise skip because your project didn't happen to need them. Security models, cost tiers, monitoring. That breadth is real.

---

## The map

| Certification | Validates | Take it after |
|---|---|---|
| **DP-900** — Azure Data Fundamentals | Broad Azure data concepts, entry level | [[Azure SQL Database]] |
| **DP-700** — Fabric Data Engineer Associate | SQL, PySpark, orchestration — the current Microsoft data-engineering credential (replaced DP-203, retired March 2025) | [[Unity Catalog and orchestration]] |
| **Databricks Certified Data Engineer Associate** | Spark, Delta Lake, Databricks platform specifically | [[Databricks and Delta Lake]] |
| **Databricks Certified Machine Learning Associate** | MLflow, Spark ML, ML lifecycle on Databricks | [[MLflow experiment tracking]] |

---

## DP-900 — Azure Data Fundamentals

**What it covers:** core data concepts (relational vs. non-relational, OLTP vs. OLAP, batch vs. streaming), and a tour of Azure's data services.

**Difficulty:** the easiest on this list by a distance. Genuinely entry-level, mostly conceptual, little hands-on required.

**Take it after** [[Azure SQL Database]] — by then you've used most of what it asks about.

**Honest assessment:** low value on its own if you already have DP-700 or a Databricks cert. It's worth taking if you want an early, cheap confidence win, or if your CV has no Azure on it at all. Skip it if you're going straight for DP-700.

**Prep:** the free Microsoft Learn path is genuinely sufficient. Don't buy a course for this one.

---

## DP-700 — Fabric Data Engineer Associate

**What it covers:** ingesting and transforming data, implementing a lakehouse, orchestrating pipelines, securing and monitoring a data platform — expressed through **Microsoft Fabric**.

**This is the current Microsoft data-engineering credential.** It replaced DP-203 (Azure Data Engineer Associate), which retired in March 2025. If you find a course or a Reddit thread recommending DP-203, it's out of date.

**The Fabric caveat, stated plainly:** Fabric is Microsoft's newer unified analytics platform. It's a different product surface from ADLS + Databricks, but the *concepts* are the ones you've learned:

| You learned | Fabric calls it |
|---|---|
| ADLS Gen2 | OneLake |
| Databricks notebook + Spark | Fabric notebook + Spark (same Spark) |
| Delta Lake tables | Delta Lake tables (identical) |
| Databricks Workflows | Data Factory pipelines |
| Azure SQL serving layer | Warehouse |
| Unity Catalog | Fabric's governance + Purview |

Bronze/silver/gold, PySpark, Delta, orchestration, security — all the same. **You'll need to spend time in a Fabric trial workspace to learn the UI and the product-specific vocabulary**, but you're not learning the subject again.

**Take it after** [[Unity Catalog and orchestration]].

**Prep:** Microsoft Learn path, plus real hands-on time in a Fabric trial. This one does test product specifics, so reading alone isn't enough. Do the practice assessment on the official page — the question style is distinctive, and DP-level exams lean on scenario questions where several answers look right.

---

## Databricks Certified Data Engineer Associate

**What it covers:** the Databricks Lakehouse platform, Delta Lake, building ETL with Spark SQL and PySpark, incremental processing, production pipelines, and the governance model.

**This is the highest-signal certification on the list for the stack you're building**, because it maps almost exactly onto stages 2–3 of this roadmap. Very little of it is knowledge you wouldn't otherwise need.

**Take it after** [[Databricks and Delta Lake]], ideally with [[Unity Catalog and orchestration]] done too.

**Where the questions concentrate** (based on the published exam guide — check it for current weightings):
- Delta Lake mechanics: the transaction log, `MERGE`, time travel, `OPTIMIZE`/`ZORDER`, `VACUUM` and its retention trade-off
- The medallion architecture and what belongs in each layer
- Incremental processing: Auto Loader, Structured Streaming basics
- Unity Catalog: the three-level namespace, grants, managed vs. external tables
- Workflows: tasks, dependencies, and production deployment

> **Everything in that list is a section of [[Databricks and Delta Lake]] and [[Unity Catalog and orchestration]].** If you built the capstone pipeline properly, you have already done most of the preparation. The gaps are likely to be Auto Loader and Delta Live Tables, which the capstone doesn't force you to use.

**Prep:** Databricks Academy has free self-paced courses covering the syllabus. Do the official practice exam — Databricks' question style tends to be quite specific about exact syntax and default behaviours (what `VACUUM`'s default retention is, what `mode("overwrite")` does to a schema).

---

## Databricks Certified Machine Learning Associate

**What it covers:** the ML lifecycle on Databricks — MLflow tracking and the registry, feature engineering, Spark ML, AutoML, and model deployment.

**The gap to be aware of:** it assumes **Spark ML** (`pyspark.ml`), which this roadmap deliberately skips in favour of PyTorch. Spark ML is a different API — `Pipeline`, `VectorAssembler`, `CrossValidator`, distributed training of classical models like gradient-boosted trees.

That's a real gap, and it's the main extra work for this exam. It's not conceptually hard — it's scikit-learn-shaped, distributed — but you do have to learn the API.

**Take it after** [[MLflow experiment tracking]], plus deliberate Spark ML practice.

**Honest assessment:** take this one only if you're targeting **ML engineering on Databricks** specifically. If your target is data engineering with some ML, the Data Engineer Associate is a better use of the same time. Two Databricks certifications is not twice as convincing as one.

---

## Suggested order

If you're doing more than one:

1. **Databricks Data Engineer Associate** — highest signal, maps most directly onto what you've built
2. **DP-700** — broad Microsoft credential, valuable in Azure shops, complements rather than duplicates
3. **DP-900** — only if you want an early confidence win, or your CV shows no Azure at all
4. **Databricks ML Associate** — only for an explicitly ML-engineering target

> **Or none of them.** A working, deployed capstone with a public URL, a clean repo, tests, and CI is stronger evidence than any certification on this list. If your time is limited, that's where it goes.

---

## How to prepare (for any of them)

**1. Build first.** You'll retain vastly more from a syllabus when you've already hit the problems it describes. Reading about `VACUUM` retention is abstract; having lost time travel to it is memorable.

**2. Read the official exam guide.** Every one publishes a topic breakdown with percentage weightings. It tells you exactly what's tested and roughly how much. Use it as a checklist against your own knowledge.

**3. Do the official practice assessment.** Not primarily for the score — for the *question style*. These exams have a house style, and scenario questions where three of four answers are technically true but only one is best need practice to read correctly.

**4. Fill gaps deliberately.** Practice test results tell you which topics are weak. Go and *build* something with that feature rather than re-reading about it.

**5. Book the exam before you feel ready.** An open booking creates a deadline. Without one, "I'll take it when I'm ready" means never.

> **Exam-technique note that's worth more than it sounds:** on Microsoft exams especially, read the *last sentence of the question first*. Scenarios are long, and the actual question ("which should you use to minimise cost?" vs. "…to minimise latency?") changes which answer is right. Two answers are usually both workable, and the qualifier decides.

---

## What to do with it afterwards

- **LinkedIn** — add it to Licenses & Certifications with the credential ID. Recruiters filter on this.
- **CV** — one line, with the date. Don't give it its own section above your projects.
- **Renewal** — Microsoft associate-level certifications generally need annual renewal via a free online assessment. Databricks certifications expire after a set period and require retaking. Diary it; letting one lapse silently is an avoidable annoyance.

> **Where it goes on your CV:** below your projects, not above. The capstone is what gets you the interview. The certification is what stops HR filtering you out before a human reads it. Different jobs — keep them in that order.

---

## Practice checklist

- [ ] Decide, honestly, whether certifications help for the jobs *you* are targeting
- [ ] Check the current official exam guide for each one you're considering — format, price and topics change
- [ ] Confirm the roadmap stage that maps to it is genuinely finished first
- [ ] Identify the gaps between what you built and what the exam guide lists
- [ ] Take the official practice assessment before deciding you're ready
- [ ] Book the exam to create a deadline

## Resources

- [Microsoft Learn: DP-700 overview](https://learn.microsoft.com/credentials/certifications/fabric-data-engineer-associate/)
- [Microsoft Learn: DP-900 overview](https://learn.microsoft.com/credentials/certifications/azure-data-fundamentals/)
- [Databricks certification overview](https://www.databricks.com/learn/certification)
- [Databricks Academy](https://www.databricks.com/learn/training/home) — free self-paced courses

## Next

[[Capstone overview and architecture]]
