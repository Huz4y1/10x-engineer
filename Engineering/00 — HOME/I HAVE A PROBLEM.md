---
tags: [home, troubleshooting, router, moc]
---

# I HAVE A PROBLEM

Start here when something is wrong or you don't know what to do next. Find your row, follow the link.

Home: [[ULTIMATE ENGINEER]] · Method: [[_Troubleshooting template]]

---

## First, five commands

Before reading anything, run these. They answer most questions outright.

```bash
git log --oneline -5          # what changed recently
docker logs <container>       # what did it say before dying
kubectl describe pod <pod>    # read the Events at the bottom
env | sort                    # is the config what I think it is
df -h && free -h              # am I out of disk or memory
```

> **"Nothing changed" is almost always false.** Something changed: a deploy, a dependency, the data volume, a certificate, or free disk space.

---

## 🔴 Something is broken

### It won't start

| Symptom | Go to |
|---|---|
| Container exits immediately | [[Docker deep dive]] — `docker logs` first |
| `CrashLoopBackOff` | [[Kubernetes and AKS]] — `logs --previous` |
| Pod stuck `Pending` | [[Kubernetes and AKS]] — insufficient CPU/memory |
| `ImagePullBackOff` | [[Docker deep dive]] — wrong tag or no registry access |
| App crashes on startup, no logs | [[Docker deep dive]] — `PYTHONUNBUFFERED=1` |
| Startup fails: model file missing | [[FastAPI data and deployment]] — that's correct behaviour |

### It won't connect

| Symptom | Go to |
|---|---|
| Connection refused to my own service | [[Docker deep dive]] — bind `0.0.0.0`, not `127.0.0.1` |
| One container can't reach another | [[Docker deep dive]] — use the **service name**, not `localhost` |
| `403` / `AuthorizationPermissionMismatch` on Azure | [[Azure fundamentals]] — you need a **data-plane** role |
| `401` vs `403` — which is which? | [[Networking reference]] — 401 = who are you, 403 = no |
| Database connection times out | [[Azure SQL Database]] — firewall, your IP changed |
| Random "connection reset" | [[Azure SQL Database]] — `pool_pre_ping=True` |
| `too many connections` | [[FastAPI data and deployment]] — one engine, not one per request |
| Service resolves but times out | [[Kubernetes and AKS]] — `get endpoints` shows `<none>` |
| CORS error in the browser | [[FastAPI fundamentals]] |
| Certificate expired / TLS error | [[Networking reference]] |

### It's wrong

| Symptom | Go to |
|---|---|
| Row count exploded after a join | [[SQL fundamentals]] — fan-out |
| Totals are double what they should be | [[Data modeling]] — mixed grain |
| A percentage sums to 4000% | [[Data modeling]] — non-additive measure |
| Re-running the pipeline duplicated rows | [[Unity Catalog and orchestration]] — idempotency |
| Predictions plausible but wrong | [[Model export and serving]] — **feature order or scaler** |
| ONNX differs from PyTorch | [[Model export and serving]] — `model.eval()` before export |
| Same input, different answer each time | [[Model export and serving]] — dropout baked into the export |
| Model brilliant offline, useless live | [[Problem framing]] — **leakage** |
| Money totals off by pennies | [[Data modeling]] — use `DECIMAL`, never `FLOAT` |
| `0.1 + 0.2 != 0.3` | [[24 — SCIENTIFIC COMPUTING]] |
| Historical report changed after an update | [[Data modeling]] — needed SCD Type 2 |

### It's slow

| Symptom | Go to |
|---|---|
| Python loop is slow | [[When to leave Python]] — **profile first, then vectorise** |
| pandas takes minutes | [[When to leave Python]] — `iterrows` is almost always a bug |
| SQL query suddenly slow | [[SQL fundamentals]] · [[PostgreSQL reference]] — `EXPLAIN` |
| Spark job hangs or crawls | [[PySpark core]] — skew, shuffles, `.explain()` |
| One Spark task takes 50× the others | [[PySpark core]] — data skew |
| API slow under load | [[FastAPI fundamentals]] — blocking call inside `async def` |
| API slow but the model is fast | [[Deployment patterns]] — it's the database |
| GPU utilisation under 30% | [[CUDA and GPU programming]] — the data loader is starving it |
| Docker rebuild takes minutes every time | [[Docker deep dive]] — layer ordering |
| CI takes 20 minutes | [[CI-CD pipelines]] — caching, split fast/slow tests |
| First request after idle takes 10s | [[Deployment patterns]] — scale-to-zero cold start |

### Training won't work

| Symptom | Go to |
|---|---|
| Loss doesn't decrease at all | [[Tensors, autograd and the training loop]] — check all six loop lines |
| Loss gets *worse* over epochs | [[Tensors, autograd and the training loop]] — missing `zero_grad()` |
| Loss becomes `NaN` | [[Tensors, autograd and the training loop]] — LR too high, or NaN input |
| Shape mismatch in the loss | [[Tensors, autograd and the training loop]] — `(N,)` vs `(N,1)` |
| Out of GPU memory | [[CUDA and GPU programming]] — batch size, or `.item()` |
| OOM after several epochs | [[Tensors, autograd and the training loop]] — accumulating loss tensors |
| Validation noisier than training | [[Tensors, autograd and the training loop]] — missing `model.eval()` |
| Can't beat the baseline | [[Problem framing]] — the signal isn't in your features |
| Score is 0.99 on the first try | [[Problem framing]] — **that's leakage, go and find it** |
| RL agent does something absurd | [[Reinforcement learning in depth]] — reward hacking |
| Fine-tuned LLM still doesn't know my facts | [[Large language models]] — fine-tuning ≠ knowledge, use RAG |

### It silently stopped

| Symptom | Go to |
|---|---|
| Everything green, data is days old | [[Observability for data and ML pipelines]] — **prediction freshness** |
| Pipeline failed and nobody noticed | [[Unity Catalog and orchestration]] — failure alerts |
| Model quietly got worse over months | [[Monitoring and iteration]] — drift |
| Bad data reached the dashboard | [[Azure ML and the MLOps stack]] — quality gates |
| Kafka consumer receives nothing | [[Kafka]] — `kafka-consumer-groups --describe` |
| Stream reprocesses everything on restart | [[Kafka]] — lost checkpoint |

---

## 🟡 How do I…?

| Task | Go to |
|---|---|
| Start a new project properly | [[Dev environment - Git, Docker, CLI]] |
| Undo something in git | [[Command reference]] — and `git reflog` |
| Design a database schema | [[Data modeling]] — write the **grain** first |
| Write a fast SQL query | [[SQL fundamentals]] |
| Process data too big for memory | [[PySpark core]] · [[When to leave Python]] |
| Store files in a lake | [[Object storage]] — [[SeaweedFS]] (local) · [[ADLS Gen2]] (Azure) · [[AWS S3]] (AWS) |
| **Set up a PostgreSQL database** | **[[PostgreSQL]]** — local, [[Azure Database for PostgreSQL]], [[AWS RDS for PostgreSQL]] |
| Use pgAdmin | [[pgAdmin 4]] |
| **Build a website with Python, start to finish** | **[[Flask SQLite stack]]** — then [[Flask SQLite - notes app]] |
| **Show database rows on an HTML page** | **[[Flask SQLite - showing data on the page]]** |
| Format money, dates or NULLs on a page | [[Flask SQLite - showing data on the page]] |
| Make a page with real data faster | [[Flask SQLite - showing data on the page]] (the N+1 trap) |
| Add log-ins to a site | [[Flask SQLite - notes app]] · [[Flask - user accounts and login]] |
| Use a database with no server to install | [[Flask SQLite stack]] (SQLite) |
| Join tables, count and average them | [[Flask SQLite - book tracker]] · [[SQL fundamentals]] |
| Split a Flask app into files | [[Flask SQLite - book tracker]] · [[Flask - project structure and blueprints]] |
| Pick the right column type | [[PostgreSQL data types]] |
| Insert-or-update (upsert) in Postgres | [[PostgreSQL SQL]] |
| Connect to Postgres from Python | [[PostgreSQL with SQLAlchemy]] |
| Upload or download files on AWS | [[AWS S3]] |
| **Actually set up and use SeaweedFS** | **[[Using SeaweedFS]]** |
| Save a PySpark DataFrame to a bucket | [[Using SeaweedFS]] |
| Understand what a bucket is | [[Using SeaweedFS]] |
| Get ACID on a data lake | [[Databricks and Delta Lake]] |
| Schedule a pipeline | [[Unity Catalog and orchestration]] |
| Train a model | [[Tensors, autograd and the training loop]] |
| Work out what to build from a dataset | [[How to use ML on data]] |
| Build features for a model | [[Feature engineering]] |
| Model works offline, fails live | [[Feature engineering]] — training-serving skew |
| Track experiments | [[MLflow experiment tracking]] |
| Get a model out of a notebook | [[Model export and serving]] |
| Build an API | [[FastAPI fundamentals]] |
| **Build a Python website with a database** | **[[Flask]]** → [[Flask - full project walkthrough]] |
| Add login to a Flask app | [[Flask - user accounts and login]] |
| Style HTML pages without much CSS | [[Flask - styling with Bulma]] |
| Build a dashboard | [[Streamlit vs Django]] |
| Containerise something | [[Docker deep dive]] |
| Deploy to the cloud | [[FastAPI data and deployment]] · [[Cloud comparison dictionary]] |
| **Build the whole pipeline end to end** | **[[Pipeline setup - overview]]** |
| Run everything with no cloud account | [[Running the whole stack locally]] |
| Set up CI/CD | [[CI-CD pipelines]] |
| Add monitoring | [[Observability for data and ML pipelines]] |
| Write tests | [[Testing and CI-CD]] |
| Make code faster | [[When to leave Python]] |
| Use a GPU properly | [[CUDA and GPU programming]] |
| Build a RAG system | [[LLM and GenAI track]] |
| Run an LLM locally | [[Large language models]] |
| Control a motor | [[PID and Kalman filters]] |
| Fuse sensor readings | [[PID and Kalman filters]] |
| Read a sensor on a microcontroller | [[20 — EMBEDDED]] · [[MQTT]] |
| Take an idea to production | [[The playbook]] |
| Read a research paper | [[27 — ML RESEARCH]] |

---

## 🟢 Which should I use?

| Question | Answer lives in |
|---|---|
| **I have data — what AI can I build with it?** | **[[How to use ML on data]]** |
| Do I even need machine learning? | [[Problem framing]] — **usually no** |
| Rules, classical ML, deep learning, or an LLM? | [[Choosing your approach]] |
| Neural network or gradient boosting? | [[Choosing your approach]] — tabular → boosting |
| PyTorch, TensorFlow or JAX? | [[TensorFlow and Keras]] · [[JAX]] — default PyTorch |
| Fine-tune or RAG? | [[Large language models]] — facts → RAG |
| RL or classical control? | [[Reinforcement learning in depth]] — **learn control first** |
| Batch or real-time serving? | [[Deployment patterns]] — **batch by default** |
| SQL or NoSQL? | [[06 — DATABASES]] |
| Postgres or a vector database? | [[PostgreSQL reference]] — pgvector first |
| Kafka or MQTT? | [[Kafka]] · [[MQTT]] — transport vs. durable log |
| Spark, polars or DuckDB? | [[When to leave Python]] — Spark only above one machine |
| Container Apps or Kubernetes? | [[Kubernetes and AKS]] — **probably not Kubernetes** |
| Streamlit or Django? | [[Streamlit vs Django]] |
| Flask, Django or FastAPI? | [[Flask]] — *When to choose Flask* |
| Python, C++ or Rust? | [[Systems performance]] |
| SeaweedFS or MinIO? | [[SeaweedFS]] |
| Azure, AWS or GCP? | [[Cloud comparison dictionary]] |
| Which certification? | [[Certification map]] |
| Which career? | [[31 — CAREERS]] |

---

## 🔵 I don't understand a word

[[29 — DICTIONARY]] — A–Z index. Every page: *problem it solves → how it works → code → under the hood → when NOT to use → mistakes → debugging → cloud equivalents.*

| Need | Go to |
|---|---|
| A maths formula | [[Mathematics reference]] |
| Big O / an algorithm | [[Algorithms and data structures reference]] |
| A port number, HTTP code, network term | [[Networking reference]] |
| A CLI command | [[Command reference]] |
| **A PySpark function** | **[[PySpark reference]]** |
| **A FastAPI feature** | **[[FastAPI reference]]** |
| **A Streamlit widget** | **[[Streamlit]]** |
| A SQL or psql command | [[PostgreSQL reference]] · [[PostgreSQL SQL]] |
| A Postgres data type | [[PostgreSQL data types]] |

---

## The method, when nothing above fits

1. **Reproduce it** — smallest input that triggers it
2. **Read the actual error** — the *bottom* of a Python traceback
3. **Bisect the pipeline** — find the first link that's wrong
4. **Ask what changed** — deploy, dependency, data, config, or time
5. **One hypothesis, one test** — change one thing
6. **Check your assumptions** — is the running code the code you edited?
7. **Write it down** — in the note for that technology

Full detail: [[_Troubleshooting template]]

> **The layers to check, cheapest first:** my code → config → data → dependencies → container → network → permissions → resources → the platform. Layers 6-7 cause most cloud problems; 1-3 cause most local ones. If it works locally and fails deployed, start at the container.
