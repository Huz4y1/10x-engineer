---
tags: [project, from-scratch, llm, transformers]
status: not-started
---

# Project 03 - LLM from Scratch

Track: [[Machine Learning Research Engineer]] · Previous: [[Project 02 - Neural Network from Scratch]]

---

## What you're building

A small GPT-style transformer, trained on text you choose, generating text. Tokeniser, attention, transformer blocks, training loop — all written by you.

## Why

[[Large language models]] tells you what an LLM does. This makes you *build* one. Attention stops being a diagram and becomes forty lines of code you understand.

## Concepts used

| Concept | Note |
|---|---|
| Attention, Q/K/V | [[Transformers and LLM basics]] |
| Tokenisation | [[Large language models]] |
| Softmax, cross-entropy | [[Mathematics reference]] |
| Training loop, Adam | [[Tensors, autograd and the training loop]] |
| GPU usage | [[CUDA and GPU programming]] |

## Build steps

- [ ] **1 — Character-level tokeniser.** Simplest possible: map each character to an integer.
- [ ] **2 — Bigram baseline.** Predict the next character from the current one alone. Terrible output — **that's your baseline** ([[Problem framing]]).
- [ ] **3 — Single-head self-attention.**
  ```python
  import math
  import torch

  q, k, v = x @ Wq, x @ Wk, x @ Wv
  scores = (q @ k.transpose(-2, -1)) / math.sqrt(head_dim)
  scores = scores.masked_fill(mask == 0, float('-inf'))   # causal: can't see the future
  out = torch.softmax(scores, dim=-1) @ v
  ```
- [ ] **4 — Multi-head.** Run several in parallel, concatenate.
- [ ] **5 — A full block.** Attention → residual + LayerNorm → feed-forward → residual + LayerNorm.
- [ ] **6 — Positional encoding.** Without it the model can't tell word order at all.
- [ ] **7 — Stack blocks, train.** 4-6 layers on a few MB of text.
- [ ] **8 — Generate.** Sample with temperature and top-k.
- [ ] **9 — BPE tokeniser.** Upgrade from characters to sub-words.

> **The causal mask is the bit to get right.** Without it, the model can see the answer while predicting it — loss drops beautifully and generation is nonsense. It's [[Problem framing|leakage]] in architectural form.

## Checkpoints

- [ ] Attention output shape matches input shape
- [ ] Attention weights per row sum to 1
- [ ] Causal mask verified — position *i* attends only to ≤ *i*
- [ ] Loss beats the bigram baseline
- [ ] Generated text is recognisably English-shaped
- [ ] You can explain Q, K and V without notes

## What will go wrong

| Symptom | Cause | Fix |
|---|---|---|
| Loss suspiciously low, output garbage | Causal mask wrong | Print the mask; check it's lower-triangular |
| Loss is NaN | No `/sqrt(d)` scaling | Scale the scores |
| No word order sensitivity | Missing positional encoding | Add it |
| Training unstable | No LayerNorm / residuals | Add both |
| Repetitive output | Temperature too low | Raise it; add top-k |
| Out of memory | Sequence length — attention is O(n²) | Shorter context, smaller batch |

## Resources

Karpathy's *Neural Networks: Zero to Hero* builds exactly this. See [[32 — RESOURCES]].

## Related

[[Transformers and LLM basics]] · [[Large language models]] · [[09 — DEEP LEARNING]] · [[10 — PYTORCH]] · [[Project 05 - Notes Intelligence Assistant]]
