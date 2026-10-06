---
tags: [project, from-scratch, maths, rust, c, cpp]
status: not-started
---

# Project 01 - Linear Regression from Scratch

Track: [[Machine Learning Research Engineer]] · Next: [[Project 02 - Neural Network from Scratch]]

---

## What you're building

Linear regression with gradient descent, implemented from nothing. No `sklearn`, no `numpy.linalg.lstsq`. First in Python to get it right, then in Rust, C and C++ to see what the language costs you.

## The whole algorithm

Fit `y = wx + b` by nudging `w` and `b` downhill.

```
prediction:  ŷ = wx + b
loss (MSE):  L = (1/n) Σ (y - ŷ)²
gradients:   dL/dw = -(2/n) Σ x(y - ŷ)
             dL/db = -(2/n) Σ (y - ŷ)
update:      w -= lr · dL/dw
             b -= lr · dL/db
```

That's it. Everything in [[09 — DEEP LEARNING]] is this, repeated with more parameters.

## Concepts used

| Concept | Note |
|---|---|
| Derivatives and gradients | [[Mathematics reference]] |
| MSE, gradient descent, learning rate | [[Mathematics reference]] |
| Vectorisation | [[When to leave Python]] |
| Rust / C / C++ basics | [[Rust]] · [[C]] · [[C++]] |
| Benchmarking honestly | [[When to leave Python]] |

## Build steps

- [ ] **1 — Python, loops.** Write it with explicit `for` loops. Slow and obvious.
- [ ] **2 — Python, vectorised.** Same maths in NumPy. Time both. *(Expect 50-200×.)*
- [ ] **3 — Verify.** Compare against the closed-form solution `w = (XᵀX)⁻¹Xᵀy`. They should agree.
- [ ] **4 — Multiple features.** Generalise to `y = Xw + b`. Now it's matrix multiplication.
- [ ] **5 — Rust.** Port it. Use `--release`.
- [ ] **6 — C and C++.** Port again. Compile with `-O3`.
- [ ] **7 — Benchmark honestly.** Compare against **vectorised NumPy**, not your Python loop.

> **The lesson in step 7:** NumPy is already compiled C calling SIMD instructions. Your hand-written C will likely *tie* it, not beat it. That's the real lesson about [[When to leave Python]] — you don't beat a good library by rewriting; you beat a bad algorithm.

## Checkpoints

- [ ] Loss decreases monotonically with a sensible learning rate
- [ ] Learned `w`, `b` match the closed-form solution to ~4 decimal places
- [ ] Too-high learning rate makes it diverge — **watch it happen**
- [ ] All four language implementations agree on the same data
- [ ] A timing table you'd defend

## What will go wrong

| Symptom | Cause | Fix |
|---|---|---|
| Loss goes to infinity/NaN | Learning rate too high | Lower it 10× |
| Loss barely moves | Learning rate too low, or features unscaled | Raise it; standardise features |
| Rust slower than NumPy | Debug build | `cargo build --release` |
| C slower than Python | No optimisation | `-O3` |
| Diverges with multiple features | Features on different scales | Standardise |

## Related

[[Mathematics reference]] · [[08 — MACHINE LEARNING]] · [[When to leave Python]] · [[Rust for this stack]] · [[C++ for this stack]]
