---
tags: [moc, performance, cuda, cpp, rust, systems]
status: not-started
---

# Systems performance

Where Python stops being enough, and what to reach for. Part of [[Data Engineering]].

Optional for a data engineering role. **Essential for an ML systems or research engineering role.**

## Notes

[[When to leave Python]] — **read this first.** Profiling, the optimisation ladder, and how to tell whether you have a language problem or a code problem

[[CUDA and GPU programming]] — what a GPU is, warps and memory, writing kernels, and using the GPU well from PyTorch

[[C++ for this stack]] — ONNX Runtime and LibTorch serving, custom PyTorch operators, RAII, and the traps

[[Rust for this stack]] — fast ingestion, PyO3 extensions, Actix serving, ownership and borrowing

## The one-line version

> **Measure first. Fix the algorithm. Vectorise. Only then change language.**
> Most "Python is too slow" is a Python loop around a fast library — and no rewrite fixes an I/O bottleneck.

## Where each language belongs

```mermaid
flowchart LR
    A["Ingestion / parsing<br/>RUST"] --> B["Distributed processing<br/>PYSPARK"]
    B --> C["Training<br/>PYTHON + CUDA"]
    C --> D["Inference kernels<br/>C++ / CUDA"]
    D --> E["Serving<br/>PYTHON or RUST"]
```

| Layer | Language | Why |
|---|---|---|
| Custom ingestion, parsing | **Rust** | Per-record overhead dominates; safe parallelism |
| Bulk transformation | **PySpark** | Already compiled underneath ([[PySpark core]]) |
| Model training | **Python + CUDA** | PyTorch *is* C++ and CUDA ([[Tensors, autograd and the training loop]]) |
| Custom operators | **C++ / CUDA** | When no built-in op composes |
| Serving | **Python**, or **Rust** for latency floors | [[FastAPI fundamentals]] / [[Actix Web]] |
| Embedded / edge | **C / C++ / Rust** | No Python runtime ([[Embedded]]) |

> Python stays the orchestration layer even in a heavily optimised system. That's the design PyTorch, Spark and polars all chose deliberately — not a compromise.

## Choosing between them

| | [[C]] | [[C++]] | [[Rust]] |
|---|---|---|---|
| Memory safety | Manual | RAII helps | **Compiler-enforced** |
| ML ecosystem | Small | **Huge** | Growing |
| Python bindings | ctypes | pybind11 | **PyO3** |
| Build tooling | Make | CMake | **Cargo** |

> **For new code: C++ when you must interface with LibTorch, ONNX Runtime or CUDA. Rust for everything else.**

## Related

[[PyTorch]] · [[Model export and serving]] · [[Docker deep dive]] · [[Deployment patterns]] · [[Languages]]
