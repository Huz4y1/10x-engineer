---
tags: [moc, ml]
---

# 08 — MACHINE LEARNING

> Learning patterns from data instead of writing the rules yourself.

**Why it matters:** Classical ML is where you should start and where most tabular problems should end. Gradient boosting beats deep learning on structured data far more often than the internet suggests.

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

> **The step-by-step working method: [[Classical ML in practice]].** This page is the concepts; that one is the order you actually do them in.

---

## Core concepts

Supervised · unsupervised · reinforcement ([[12 — REINFORCEMENT LEARNING]]) · regression · classification · clustering · dimensionality reduction · feature engineering · training/validation/testing · overfitting · underfitting · **bias-variance** · regularisation · cross-validation · metrics · hyperparameters · optimisation

## Algorithms

| Algorithm | Idea | Use |
|---|---|---|
| Linear regression | Fit a line, minimise squared error | Baseline for regression |
| Logistic regression | Linear, squashed to a probability | Baseline for classification |
| Decision trees | Recursive if/else splits | Interpretable |
| Random forests | Many trees, averaged | Robust default |
| **Gradient boosting** | Trees fixing previous trees' errors | ✅ **Best default on tabular** |
| SVM | Maximum-margin separator | Small, high-dimensional data |
| k-means | Iterative cluster centres | Segmentation |
| PCA | Project onto top eigenvectors | Dimensionality reduction |

> **Start with logistic/linear regression as a baseline, then LightGBM.** See [[Choosing your approach]] for why this beats jumping to neural networks.

## The one thing that matters most

**Bias vs variance.** High bias = too simple, underfits, both errors high. High variance = too complex, overfits, memorises training data. Everything you tune is trading one for the other.

## Practical
**[[How to use ML on data]]** — *what can I build with this dataset?*

[[Classical ML in practice]] · **[[Feature engineering]]** · [[Problem framing]] · [[Choosing your approach]] · [[scikit-learn]] · [[MLflow experiment tracking]]

## Related
[[02 — MATHEMATICS]] · [[09 — DEEP LEARNING]] · [[13 — ML ENGINEERING]]
