---
tags: [moc, scicomp]
---

# 24 — SCIENTIFIC COMPUTING

> Doing mathematics on a computer, correctly.

**Why it matters:** Computers don't do real numbers — they do floating point. Knowing where that breaks is why your loss became NaN.

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

> **The working detail: [[Numerical methods in practice]]** — floating point, SciPy, and where NaN comes from.

---

## Tools

**NumPy** — arrays, broadcasting, vectorisation · **SciPy** — integration, optimisation, signal processing, linear algebra

## Numerical methods

Linear algebra (solving systems, decompositions) · numerical integration · ODEs · optimisation · Monte Carlo · simulation

## Floating point — the part that bites

```python
0.1 + 0.2 == 0.3        # False
0.1 + 0.2               # 0.30000000000000004
```

| Concept | Why it matters |
|---|---|
| **Precision** | float32 vs float64 vs bfloat16 ([[CUDA and GPU programming]]) |
| **Catastrophic cancellation** | Subtracting near-equal numbers destroys precision |
| **Numerical stability** | Why softmax subtracts the max before exponentiating |
| **Conditioning** | Small input change, huge output change |

> **This is why money uses `DECIMAL` not `FLOAT`** ([[Data modeling]]), why `BCEWithLogitsLoss` beats sigmoid+BCE, and why gradients underflow in float16.

## Vectorisation

The single highest-value skill: replace loops with array operations. 10-100x, no language change. See [[When to leave Python]].

## Related
[[Numerical methods in practice]] · [[NumPy]] · [[02 — MATHEMATICS]] · [[09 — DEEP LEARNING]] · [[22 — CONTROL SYSTEMS]] · [[Systems performance]]
