---
tags: [moc, pytorch]
---

# 10 — PYTORCH

> The framework that implements deep learning.

**Why it matters:** PyTorch is Python on top of C++ and CUDA. Learning it well means knowing which layer you're actually in.

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

---

> **Already built in depth.** Start at [[Tensors, autograd and the training loop]].

## The notes

[[Tensors, autograd and the training loop]] — tensors, autograd, `nn.Module`, Dataset/DataLoader, the loop, `train()` vs `eval()`, saving

[[CNNs and transfer learning]] — convolutions, pretrained backbones, fine-tuning

[[Transformers and LLM basics]] — attention, tokenisation, fine-tuning

[[MLflow experiment tracking]] — tracking runs

[[Model export and serving]] — TorchScript, ONNX, the serving contract

[[CUDA and GPU programming]] — using the GPU well

[[PyTorch]] — the short intro note

## API surface

`torch.Tensor` · `autograd` · `nn.Module` · `Dataset` · `DataLoader` · `optim` · loss functions · `model.eval()` / `model.train()` · `state_dict` · checkpointing · device management · distributed training

## The five bugs everyone hits

| Bug | Symptom |
|---|---|
| Missing `optimizer.zero_grad()` | Training silently degrades |
| Missing `model.eval()` | Noisy validation |
| Targets `(N,)` vs predictions `(N,1)` | Broadcasting to `(N,N)`, nonsense loss |
| `total += loss` not `.item()` | Memory leak |
| Random split on time series | Brilliant offline, useless live |

## Related
[[09 — DEEP LEARNING]] · [[13 — ML ENGINEERING]] · [[Systems performance]]
