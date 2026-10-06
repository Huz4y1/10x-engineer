---
tags: [moc, troubleshooting]
---

# 30 — TROUBLESHOOTING

> How to reason about a failure, not just look up a fix.

Home: [[ULTIMATE ENGINEER]] · Method: [[_Troubleshooting template]]

---

## The method

1. **Reproduce it** — smallest input that triggers it
2. **Read the actual error** — the bottom of a traceback, not the top
3. **Bisect the pipeline** — find the first link that's wrong
4. **Ask what changed** — deploy, dependency, data, config, or time
5. **One hypothesis, one test** — change one thing
6. **Check your assumptions** — is the running code the code you edited?
7. **Write it down** — in the note for that technology

Full detail: [[_Troubleshooting template]]

## Symptom index

| Symptom | Go to |
|---|---|
| Container exits immediately | [[Docker deep dive]] |
| Connection refused to my own service | [[Docker deep dive]] |
| `403` / permission denied on Azure | [[Azure fundamentals]] |
| Pod `Pending` / `CrashLoopBackOff` | [[Kubernetes and AKS]] |
| Spark job slow or hanging | [[PySpark core]] |
| SQL result wrong after a join | [[SQL fundamentals]] |
| Model great offline, useless live | [[Problem framing]] |
| Loss is NaN / won't decrease | [[Tensors, autograd and the training loop]] |
| ONNX differs from PyTorch | [[Model export and serving]] |
| API slow under load | [[FastAPI fundamentals]] |
| Everything green, data stale | [[Observability for data and ML pipelines]] |
| Consumer not receiving messages | [[Kafka]] |
| GPU utilisation low | [[CUDA and GPU programming]] |
| CI passes locally, fails on push | [[CI-CD pipelines]] |

## The universal first five commands

```bash
git log --oneline -5          # what changed
docker logs <container>       # what did it say before dying
kubectl describe pod <pod>    # read the Events at the bottom
env | sort                    # is the config what I think
df -h && free -h              # out of disk or memory
```
