---
tags: [moc, maths]
---

# 02 — MATHEMATICS

> The language everything else is written in.

**Why it matters:** Linear algebra **is** neural networks. Calculus **is** backpropagation. Probability **is** machine learning. Skip this and every later section becomes memorisation instead of understanding.

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

---

## Reference

**[[Mathematics reference]]** — every formula in one place: linear algebra, calculus, probability, losses, metrics, optimisation, floating point.

## Topics

**Algebra & functions** — equations, functions, notation

**Linear algebra** — vectors, matrices, matrix multiplication, dot products, transpose, inverse, rank, eigenvalues and eigenvectors, SVD

**Calculus** — derivatives, partial derivatives, gradients, the chain rule, integrals

**Probability** — random variables, distributions, expectation, variance, covariance, Bayes' theorem

**Statistics** — sampling, hypothesis testing, confidence intervals, correlation vs causation

**Optimisation** — gradient descent, convexity, local vs global minima, learning rates

**Numerical methods** — floating point, numerical stability, conditioning

## Why each matters here

| Maths | Powers |
|---|---|
| Matrix multiplication | Every forward pass ([[Tensors, autograd and the training loop]]) |
| Chain rule | `loss.backward()` — backpropagation *is* the chain rule |
| Gradients | Every optimiser step |
| Eigenvalues | PCA, stability analysis ([[22 — CONTROL SYSTEMS]]) |
| Probability | Loss functions, uncertainty, Bayesian methods |
| Optimisation | Training, and [[22 — CONTROL SYSTEMS]] |
| Floating point | Why your loss became NaN ([[24 — SCIENTIFIC COMPUTING]]) |

## Learning progression

- **Beginner:** vectors, matrices, derivatives, mean/variance
- **Intermediate:** matrix calculus, the chain rule through a network, distributions
- **Advanced:** SVD, eigendecomposition, convex optimisation, information theory
- **Research:** measure theory, optimisation theory, statistical learning theory

## Related
[[09 — DEEP LEARNING]] · [[08 — MACHINE LEARNING]] · [[24 — SCIENTIFIC COMPUTING]] · [[22 — CONTROL SYSTEMS]] · [[Physics]]
