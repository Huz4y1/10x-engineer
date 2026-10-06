---
tags: [moc, performance, cuda, cpp, rust, systems]
---

# 33 — SYSTEMS PERFORMANCE

> Where Python stops being enough, and what to reach for.

**Why it matters:** optional for a data engineering role, essential for an ML *systems* or research engineering role. Also the section that stops you rewriting things that didn't need rewriting.

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

---

## Notes

[[Systems performance]] — the hub

[[When to leave Python]] — **read this first.** Profiling, the 8-rung optimisation ladder, and how to tell a language problem from a code problem

[[CUDA and GPU programming]] — warps, memory coalescing, writing kernels, using the GPU well from PyTorch

[[C++ for this stack]] — ONNX Runtime and LibTorch serving, custom PyTorch operators, RAII

[[Rust for this stack]] — fast ingestion, PyO3 extensions, Actix serving, ownership

## The one-line version

> **Measure first. Fix the algorithm. Vectorise. Only then change language.**
> Most "Python is too slow" is a Python loop around a fast library — and no rewrite fixes an I/O bottleneck.

## Where each language belongs

| Layer | Language | Why |
|---|---|---|
| Ingestion, parsing | **Rust** | Per-record overhead dominates; safe parallelism |
| Bulk transformation | **PySpark** | Already compiled underneath ([[PySpark core]]) |
| Training | **Python + CUDA** | PyTorch *is* C++ and CUDA |
| Custom operators | **C++ / CUDA** | When no built-in op composes |
| Serving | **Python**, or **Rust** for latency floors | [[FastAPI fundamentals]] / [[Actix Web]] |
| Embedded / edge | **C / C++ / Rust** | No Python runtime ([[20 — EMBEDDED]]) |

## Related

[[03 — PROGRAMMING]] · [[10 — PYTORCH]] · [[24 — SCIENTIFIC COMPUTING]] · [[C]] · [[C++]] · [[Rust]] · [[Languages]]
