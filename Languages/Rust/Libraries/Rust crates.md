---
tags: [moc, rust, libraries, crates]
---

# Rust crates

The crates worth knowing, grouped by job. Language: [[Rust]]

---

## Async and web

| Crate | For | Note |
|---|---|---|
| **tokio** | The async runtime everything builds on | [[tokio]] |
| **actix-web** | Web framework | [[Actix Web]] |
| axum | Alternative web framework, tokio-native | — |
| **reqwest** | HTTP client | [[reqwest]] |
| tower | Middleware layers | — |

## Data and serialisation

| Crate | For | Note |
|---|---|---|
| **serde** | Serialise/deserialise anything | [[Serde]] |
| **polars** | DataFrames — the same engine as Python polars | [[Polars]] |
| csv | CSV reading/writing | — |
| sqlx | Async SQL, compile-time checked | [[sqlx]] |

## Performance and concurrency

| Crate | For | Note |
|---|---|---|
| **rayon** | Data parallelism — `.iter()` to `.par_iter()` | [[Rust for this stack]] |
| **pyo3** | Python extensions in Rust | [[Rust for this stack]] |
| maturin | Build and publish pyo3 extensions | [[Rust for this stack]] |

## Errors and CLI

| Crate | For |
|---|---|
| **anyhow** | Easy error handling in applications |
| **thiserror** | Custom error types in libraries |
| **clap** | Command-line argument parsing |
| tracing | Structured logging and spans |

## ML and inference

| Crate | For | Note |
|---|---|---|
| **ort** | ONNX Runtime bindings | [[Rust for this stack]] |
| candle | Pure-Rust ML framework | — |

> **The pattern to notice:** polars, ruff, uv, pydantic-core and tokenizers are all **Rust with a Python wrapper**. That's the highest-value way to use Rust in this stack — rewrite one hot function, keep Python around it ([[Rust for this stack]]).

## Related

[[Rust]] · [[Rust for this stack]] · [[Crates, Modules and modularisation]] · [[33 — SYSTEMS PERFORMANCE]]
