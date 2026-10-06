---
tags: [project, from-scratch, deep-learning, backprop]
status: not-started
---

# Project 02 - Neural Network from Scratch

Track: [[Machine Learning Research Engineer]] · Previous: [[Project 01 - Linear Regression from Scratch]] · Next: [[Project 03 - LLM from Scratch]]

---

## What you're building

A multi-layer neural network with **backpropagation you wrote yourself**. Then a tiny autograd engine that builds the computation graph automatically — your own miniature PyTorch.

## Why this is the most important project in the track

After this, `loss.backward()` stops being magic. You will have written it.

**Backpropagation is the chain rule applied to a computation graph.** Nothing more. But you only truly believe that after implementing it.

## Concepts used

| Concept | Note |
|---|---|
| **Chain rule** | [[Mathematics reference]] |
| Matrix multiplication, shapes | [[Mathematics reference]] |
| Activation functions and their derivatives | [[Mathematics reference]] |
| Forward pass, loss, backward pass | [[Tensors, autograd and the training loop]] |
| Optimisers | [[Mathematics reference]] |

## The maths

Forward, for one layer:
```
z = xW + b
a = σ(z)
```

Backward — the chain rule, layer by layer:
```
dL/dW = xᵀ · (dL/dz)
dL/db = sum(dL/dz)
dL/dx = (dL/dz) · Wᵀ        ← this is what you pass to the previous layer
where dL/dz = dL/da · σ'(z)
```

> **`dL/dx` is the whole trick.** Each layer computes its own gradients *and* hands the gradient-with-respect-to-its-input backwards. That chaining is backpropagation.

## Build steps

- [ ] **1 — Forward pass only.** Two layers, ReLU, random weights. Confirm shapes.
- [ ] **2 — Loss.** MSE for regression or cross-entropy for classification.
- [ ] **3 — Backward, by hand.** Derive and code the gradients for a 2-layer net.
- [ ] **4 — Gradient check.** *Non-negotiable:*
  ```python
  numerical = (loss(w + eps) - loss(w - eps)) / (2 * eps)
  assert abs(numerical - analytical) < 1e-5
  ```
- [ ] **5 — Train it.** XOR first (the classic non-linear test), then MNIST.
- [ ] **6 — Generalise to N layers.**
- [ ] **7 — Build a tiny autograd engine.** A `Value` class holding data, gradient, and a `_backward` closure. Topologically sort the graph and call them in reverse.
- [ ] **8 — Compare against PyTorch.** Same data, same init, same LR — gradients should match.

> **Step 4 catches nearly every bug.** If the numerical and analytical gradients disagree, your derivative is wrong. Do it before training anything.

## Checkpoints

- [ ] Gradient check passes for every layer
- [ ] It solves XOR — proving non-linearity works
- [ ] >90% on MNIST
- [ ] Your autograd's gradients match PyTorch's to ~1e-6
- [ ] You can explain why a network without activations is just one linear layer

## What will go wrong

| Symptom | Cause | Fix |
|---|---|---|
| Gradient check fails | Wrong derivative, or a transpose | Check one layer at a time |
| Shapes don't match | Row-vector vs column-vector convention | Pick one and write it down |
| XOR won't converge | No activation, or too few hidden units | ReLU + ≥2 hidden units |
| Loss is NaN | Exploding gradients, or `log(0)` | Lower LR; clip; stable softmax |
| Works on XOR, fails on MNIST | Weights initialised badly | He/Xavier init |

## Related

[[Tensors, autograd and the training loop]] · [[09 — DEEP LEARNING]] · [[Mathematics reference]] · [[10 — PYTORCH]]
