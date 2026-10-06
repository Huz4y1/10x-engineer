---
tags: [moc, python, language]
---

# Python

The language everything in this vault is orchestrated in.

Section: [[Languages]] · Choosing between languages: [[03 — PROGRAMMING]]

---

## Core language

[[Variables (Python)]]

[[Types (Python)]]

[[Selection (Python)]]

[[Iteration (Python)]]

[[Functions (Python)]]

[[Classes (Python)]]

[[Error handling (Python)]]

[[File handling (Python)]]

[[Modules and packages (Python)]]

[[JSON (Python)]]

## Python-specific power tools

[[Comprehensions and generators]]

[[Decorators]]

[[Type hints]]

[[Async (Python)]]

## Working practices

[[Packaging and environments]] — uv, virtualenvs, lockfiles

[[Testing (Python)]] — pytest

## 📚 Libraries

**[[Python libraries]]** — the index. pandas, NumPy, Polars, Pydantic, SQLAlchemy, pytest, PyTorch, PySpark, FastAPI, Streamlit and the rest, each with the full API.

---

## Why Python, and where it stops

**Python is the orchestration layer.** You write the pipeline in Python and it calls fast things — PySpark is Scala/JVM, NumPy is C, PyTorch is C++/CUDA, polars is Rust.

That's not a compromise; it's the design every one of those tools chose deliberately.

| Python is right for | Reach for something else when |
|---|---|
| Data pipelines, ML, APIs, scripting, glue | Running on a microcontroller -> [[C]] |
| Anything where developer time > CPU time | You've **measured** a CPU bottleneck -> [[Rust]] / [[C++]] |
| | You need a latency floor in microseconds |

> **Before rewriting anything for speed, read [[When to leave Python]].** Most "Python is slow" is a Python loop wrapped around a fast library, and no rewrite fixes an I/O bottleneck.

## Where Python appears in this vault

| Use | Note |
|---|---|
| Data pipelines | [[PySpark reference]] · [[PySpark core]] |
| Model training | [[Tensors, autograd and the training loop]] |
| APIs | [[FastAPI reference]] |
| Dashboards | [[Streamlit]] |
| Experiment tracking | [[MLflow experiment tracking]] |
| Scientific computing | [[24 — SCIENTIFIC COMPUTING]] |

## Related

[[Languages]] · [[03 — PROGRAMMING]] · [[Python libraries]] · [[Systems performance]]
