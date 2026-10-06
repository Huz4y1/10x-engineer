---
tags: [maths, reference, cheatsheet]
---

# Mathematics reference

Every formula this vault relies on, in one place. Section: [[02 — MATHEMATICS]]

> Use this as a lookup, not a course. Each entry says **what it is**, **the formula**, and **where it appears** in the stack.

---

## 1. Linear algebra

### Vectors

| Operation | Formula | Meaning |
|---|---|---|
| Dot product | $a \cdot b = \sum_i a_i b_i$ | How aligned two vectors are |
| Magnitude (L2 norm) | $\|a\| = \sqrt{\sum_i a_i^2}$ | Length |
| L1 norm | $\|a\|_1 = \sum_i \|a_i\|$ | Manhattan distance; Lasso |
| Cosine similarity | $\dfrac{a \cdot b}{\|a\|\,\|b\|}$ | Angle only, ignores magnitude |
| Unit vector | $\hat{a} = a / \|a\|$ | Direction only |

> **Cosine similarity is how vector search works** — embeddings in RAG, recommendation systems ([[LLM and GenAI track]]). Two documents about the same topic point the same direction regardless of length.

### Matrices

**Matrix multiplication** — `(m×n) · (n×p) = (m×p)`. The inner dimensions must match.

$$(AB)_{ij} = \sum_k A_{ik} B_{kj}$$

> **This single operation is 95% of the compute in deep learning.** A neural network layer is `output = input @ weights + bias`. A GPU exists to do this fast ([[CUDA and GPU programming]]).

| Property | Note |
|---|---|
| $AB \neq BA$ | **Not commutative.** Order matters. |
| $(AB)^T = B^T A^T$ | Transpose reverses order |
| $(AB)C = A(BC)$ | Associative — this is why you can fuse layers |
| $A I = A$ | Identity matrix |
| $A A^{-1} = I$ | Inverse (only if square and non-singular) |

**Shapes are the thing you'll actually debug.** `(32, 784) @ (784, 128) = (32, 128)`: 32 samples, 784 features in, 128 out.

### Eigenvalues and eigenvectors

$$A v = \lambda v$$

A vector $v$ whose *direction* is unchanged by $A$; $\lambda$ is how much it's scaled.

| Where it appears | Why |
|---|---|
| **PCA** | Eigenvectors of the covariance matrix are the principal components |
| **Stability analysis** | Eigenvalues with positive real part = unstable system ([[22 — CONTROL SYSTEMS]]) |
| **PageRank** | The dominant eigenvector |

### Decompositions

| Name | Form | Use |
|---|---|---|
| **SVD** | $A = U \Sigma V^T$ | Works on any matrix. PCA, compression, pseudo-inverse |
| Eigendecomposition | $A = Q \Lambda Q^{-1}$ | Square matrices only |
| LU | $A = LU$ | Solving linear systems fast |
| Cholesky | $A = LL^T$ | Symmetric positive-definite; Kalman filters |

---

## 2. Calculus

### Derivatives

| Function | Derivative |
|---|---|
| $x^n$ | $n x^{n-1}$ |
| $e^x$ | $e^x$ |
| $\ln x$ | $1/x$ |
| $\sin x$ | $\cos x$ |
| $\cos x$ | $-\sin x$ |

**Rules:**

| Rule | Formula |
|---|---|
| Sum | $(f+g)' = f' + g'$ |
| Product | $(fg)' = f'g + fg'$ |
| Quotient | $(f/g)' = \dfrac{f'g - fg'}{g^2}$ |
| **Chain** | $\dfrac{dy}{dx} = \dfrac{dy}{du}\cdot\dfrac{du}{dx}$ |

> **The chain rule IS backpropagation.** A network is nested functions $f(g(h(x)))$. To know how a weight deep inside affects the loss, you multiply the derivatives along the path. That's all `loss.backward()` does ([[Tensors, autograd and the training loop]]).

### Gradients

For $f(x_1, \dots, x_n)$:

$$\nabla f = \left[\frac{\partial f}{\partial x_1}, \dots, \frac{\partial f}{\partial x_n}\right]$$

The gradient points in the direction of **steepest increase**. So to *minimise*, step in the **negative** gradient direction:

$$\theta_{t+1} = \theta_t - \eta \nabla_\theta L$$

That's gradient descent. $\eta$ is the learning rate.

### Activation derivatives

| Activation | $f(x)$ | $f'(x)$ |
|---|---|---|
| Sigmoid | $\sigma(x) = \dfrac{1}{1+e^{-x}}$ | $\sigma(x)(1-\sigma(x))$ |
| Tanh | $\tanh x$ | $1 - \tanh^2 x$ |
| ReLU | $\max(0, x)$ | $1$ if $x>0$, else $0$ |
| Leaky ReLU | $\max(\alpha x, x)$ | $1$ if $x>0$, else $\alpha$ |

> **Why ReLU won:** sigmoid's derivative peaks at 0.25, so multiplying it through 20 layers gives $0.25^{20} \approx 10^{-12}$ — the **vanishing gradient problem**. ReLU's derivative is exactly 1 for positive inputs, so gradients survive depth.

---

## 3. Probability and statistics

### Basics

| Concept | Formula |
|---|---|
| Expectation | $E[X] = \sum_i x_i p_i$ |
| Variance | $\mathrm{Var}(X) = E[(X-\mu)^2] = E[X^2] - (E[X])^2$ |
| Std deviation | $\sigma = \sqrt{\mathrm{Var}(X)}$ |
| Covariance | $\mathrm{Cov}(X,Y) = E[(X-\mu_X)(Y-\mu_Y)]$ |
| Correlation | $\rho = \dfrac{\mathrm{Cov}(X,Y)}{\sigma_X \sigma_Y}$ — always in $[-1,1]$ |

### Bayes' theorem

$$P(A|B) = \frac{P(B|A)\,P(A)}{P(B)}$$

> **The medical-test intuition:** a 99%-accurate test for a disease affecting 1 in 10,000 gives a *positive* result that is still ~1% likely to be true. The **prior** dominates. This is why rare-event models need care ([[08 — MACHINE LEARNING]]).

### Distributions

| Distribution | Use | Parameters |
|---|---|---|
| **Normal / Gaussian** | Measurement noise, CLT | $\mu, \sigma$ |
| Bernoulli | One yes/no trial | $p$ |
| Binomial | $n$ yes/no trials | $n, p$ |
| Poisson | Events per interval | $\lambda$ |
| Exponential | Time between events | $\lambda$ |
| Uniform | Equal probability | $a, b$ |

Normal PDF:
$$f(x) = \frac{1}{\sigma\sqrt{2\pi}} e^{-\frac{(x-\mu)^2}{2\sigma^2}}$$

> **Central Limit Theorem:** the mean of many independent samples is approximately normal, whatever the underlying distribution. This is why the normal distribution is everywhere and why standard errors work.

---

## 4. Loss functions

| Loss | Formula | Use |
|---|---|---|
| **MSE** | $\frac{1}{n}\sum (y - \hat{y})^2$ | Regression. Punishes big errors hard. |
| **MAE** | $\frac{1}{n}\sum \|y - \hat{y}\|$ | Regression. Robust to outliers. |
| Huber | MSE near 0, MAE far away | Best of both |
| **Binary cross-entropy** | $-\frac{1}{n}\sum [y\log\hat{y} + (1-y)\log(1-\hat{y})]$ | Binary classification |
| **Categorical cross-entropy** | $-\sum_i y_i \log \hat{y}_i$ | Multi-class |

**Softmax** turns scores into probabilities:
$$\hat{y}_i = \frac{e^{z_i}}{\sum_j e^{z_j}}$$

> **Numerically stable softmax subtracts the max first**: $e^{z_i - \max z}$. Without it, $e^{1000}$ overflows to infinity. This is why `CrossEntropyLoss` applies softmax internally and you pass it raw logits ([[Tensors, autograd and the training loop]]).

---

## 5. Metrics

### Classification

With TP, FP, TN, FN:

| Metric | Formula | Answers |
|---|---|---|
| Accuracy | $\dfrac{TP+TN}{TP+TN+FP+FN}$ | Overall correct. **Misleading if imbalanced.** |
| **Precision** | $\dfrac{TP}{TP+FP}$ | Of those I flagged, how many were right? |
| **Recall** | $\dfrac{TP}{TP+FN}$ | Of the real ones, how many did I catch? |
| **F1** | $2\cdot\dfrac{P \cdot R}{P + R}$ | Harmonic mean. Default for imbalanced data. |
| Specificity | $\dfrac{TN}{TN+FP}$ | True negative rate |

> **Precision vs recall is a trade-off you choose deliberately**, and the error-cost table in [[Problem framing]] is how you choose. Spam filter: precision matters (don't bin real mail). Cancer screening: recall matters (don't miss a case).

### Regression

| Metric | Formula | Note |
|---|---|---|
| MAE | $\frac{1}{n}\sum\|y-\hat y\|$ | In original units. Interpretable. |
| RMSE | $\sqrt{\frac{1}{n}\sum (y-\hat y)^2}$ | Punishes large errors |
| MAPE | $\frac{100}{n}\sum\left\|\frac{y-\hat y}{y}\right\|$ | **Breaks if any $y = 0$** |
| $R^2$ | $1 - \dfrac{SS_{res}}{SS_{tot}}$ | Fraction of variance explained |

---

## 6. Optimisation

### Gradient descent variants

| Method | Update | Note |
|---|---|---|
| SGD | $\theta \mathrel{-}= \eta \nabla L$ | Simple, noisy |
| Momentum | $v = \beta v + \nabla L$; $\theta \mathrel{-}= \eta v$ | Smooths oscillation |
| **Adam** | Adaptive per-parameter rates | ✅ The default. `lr=1e-3`. |
| AdamW | Adam + correct weight decay | Best for transformers |

Adam, in full:
$$m_t = \beta_1 m_{t-1} + (1-\beta_1)g_t \quad\quad v_t = \beta_2 v_{t-1} + (1-\beta_2)g_t^2$$
$$\hat{m}_t = \frac{m_t}{1-\beta_1^t} \quad\quad \hat{v}_t = \frac{v_t}{1-\beta_2^t} \quad\quad \theta_t = \theta_{t-1} - \eta\frac{\hat{m}_t}{\sqrt{\hat{v}_t}+\epsilon}$$

Defaults: $\beta_1 = 0.9$, $\beta_2 = 0.999$, $\epsilon = 10^{-8}$.

### Regularisation

| Method | Effect |
|---|---|
| **L1 (Lasso)** | $+\lambda\sum\|w\|$ — drives weights to **exactly zero** (feature selection) |
| **L2 (Ridge / weight decay)** | $+\lambda\sum w^2$ — shrinks weights smoothly |
| Dropout | Randomly zero activations during training |
| Early stopping | Stop when validation loss rises |

---

## 7. Numbers on a computer

| Type | Bits | Range | Use |
|---|---|---|---|
| float64 | 64 | ±1.8e308, ~15 digits | NumPy default, scientific |
| **float32** | 32 | ±3.4e38, ~7 digits | **PyTorch default** |
| bfloat16 | 16 | Same range as f32, ~3 digits | Mixed precision. Preferred. |
| float16 | 16 | ±65504, ~3 digits | Mixed precision. Can overflow. |

```python
0.1 + 0.2 == 0.3     # False
0.1 + 0.2            # 0.30000000000000004
```

> **Never use floats for money** — `DECIMAL(18,2)` ([[Data modeling]]). Never compare floats with `==` — use a tolerance. See [[24 — SCIENTIFIC COMPUTING]].

---

## 8. Where each bit of maths shows up

| Maths | Appears in |
|---|---|
| Matrix multiplication | Every forward pass ([[Tensors, autograd and the training loop]]) |
| Chain rule | `loss.backward()` |
| Gradients | Every optimiser step |
| Eigenvalues | PCA; control stability ([[22 — CONTROL SYSTEMS]]) |
| Cosine similarity | Vector search / RAG ([[LLM and GenAI track]]) |
| Bayes | Kalman filters ([[21 — ROBOTICS]]); rare-event modelling |
| Cross-entropy | Every classifier |
| Variance / covariance | PCA, Kalman filters, drift detection (PSI) |
| Floating point | NaN losses, money bugs, mixed precision |

## Related

[[02 — MATHEMATICS]] · [[09 — DEEP LEARNING]] · [[08 — MACHINE LEARNING]] · [[24 — SCIENTIFIC COMPUTING]] · [[22 — CONTROL SYSTEMS]]
