---
tags: [template, troubleshooting]
---

# _Troubleshooting template

How to reason about a failure, not just look up a fix.

Home: [[ULTIMATE ENGINEER]]

---

## The method (use this before any specific guide)

Most engineers guess. Guessing works until the system is big, then it stops working entirely. This is the alternative.

### 1. Reproduce it

If you can't make it happen on demand, you can't know when you've fixed it. Find the smallest input that triggers it.

> A bug you "fixed" without reproducing it first is a bug you got lucky with.

### 2. Read the actual error

Not the first line. The **whole** thing, and the **bottom** of a Python traceback (that's the real error; the top is just the call path).

Ask: *which component produced this message?* That alone usually halves the search.

### 3. Bisect the pipeline

Every system in this vault is a chain. Find the **first** link that's wrong.

```
input → parse → transform → store → read → serve → display
```

Check the middle. Is the data correct there? Yes → the problem is downstream. No → upstream. Repeat. **Six checks finds the bug in a 64-stage pipeline.**

### 4. Ask "what changed?"

Working systems don't spontaneously break.

- Did I deploy? → `git log`, check the last commit
- Did a dependency update? → check the lockfile
- Did the data change? → check row counts and schema
- Did config change? → check env vars
- Did *time* change? → month end, DST, a certificate expiring, disk filling

> **"Nothing changed" is almost always false.** Something changed. Usually data volume, a certificate, or free disk space.

### 5. Form one hypothesis and test it

State it out loud: *"I think X is failing because Y."* Then design the **cheapest test that would prove you wrong.**

Change **one thing at a time**. Changing three and having it work teaches you nothing.

### 6. Check your assumptions

The bug is almost always in the thing you're certain about. Verify it rather than assuming:

- Is the code you're editing the code that's running? (Right container? Cached layer? Right branch?)
- Is the config being read? Print it at startup.
- Is the function actually called? Put a log line in.
- Is it the version you think? Print it.

### 7. Write down what you found

In the note for that technology. A bug you've seen once is a bug you'll see again, in eighteen months, having forgotten it entirely.

---

## The layers — check them in this order

Cheapest and most likely first.

| # | Layer | Ask |
|---|---|---|
| 1 | **My code** | Did I just change this? Does it run locally? |
| 2 | **Config** | Env vars set? Right file? Right environment? |
| 3 | **Data** | Right shape, right types, non-empty, expected volume? |
| 4 | **Dependencies** | Right versions? Lockfile committed? |
| 5 | **Container** | Right image? Layer cached? Files actually copied? |
| 6 | **Network** | DNS resolves? Port open? Firewall? Right host? |
| 7 | **Permissions** | Right identity? Right role? Data-plane vs control-plane? |
| 8 | **Resources** | Out of memory, disk, file handles, connections? |
| 9 | **The platform** | Genuinely a cloud outage? *(Check this last — it almost never is.)* |

> **Layers 6 and 7 cause most cloud problems, and 1–3 cause most local ones.** If it works locally and fails deployed, start at layer 5.

---

## The page template

```markdown
---
tags: [troubleshooting, <technology>]
---

# Troubleshooting <Technology>

## If something breaks

### Step 1 — <the cheapest check>
<command>
<what a healthy result looks like>

### Step 2 — <next>
...

### Step 3 — <next>
...

## Common errors

### `<exact error text>`

**Meaning:** <what the system is actually telling you>
**Cause:** <why it happens>
**Fix:** <what to do>
**Prevent:** <how to stop it recurring>

## How to reason about this system

<Where does data enter and leave? What's the chain?
Where would you bisect it?>

## Related
```

---

## Symptom → section

| Symptom | Go to |
|---|---|
| Container exits immediately | [[Docker deep dive]] |
| Connection refused to my own service | [[Docker deep dive]] — bind `0.0.0.0` |
| `403` / permission denied on Azure | [[Azure fundamentals]] — data-plane roles |
| Pod stuck `Pending` / `CrashLoopBackOff` | [[Kubernetes and AKS]] |
| Spark job is slow or hangs | [[PySpark core]] — skew, shuffles |
| SQL result is wrong after a join | [[SQL fundamentals]] — row counts |
| Model great offline, useless live | [[Problem framing]] — leakage |
| Loss is NaN / won't decrease | [[Tensors, autograd and the training loop]] |
| ONNX predictions differ from PyTorch | [[Model export and serving]] |
| API slow under load | [[FastAPI fundamentals]] — blocking in async |
| Everything green, data is stale | [[Observability for data and ML pipelines]] — freshness |
| Consumer isn't receiving messages | [[Kafka]] |
| GPU utilisation is low | [[CUDA and GPU programming]] |
| CI passes locally, fails on push | [[CI-CD pipelines]] |

---

## The universal first five commands

```bash
git log --oneline -5          # what changed
docker logs <container>       # what did it say before dying
kubectl describe pod <pod>    # why won't it start (read the Events)
env | sort                    # is the config what I think
df -h && free -h              # am I out of disk or memory
```

> **`kubectl describe pod` and `docker logs` answer most container questions outright.** People skip them and start editing YAML. Read the error first.
