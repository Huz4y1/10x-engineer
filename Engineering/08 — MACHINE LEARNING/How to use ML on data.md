---
tags: [machine-learning, deep-learning, llm, reinforcement-learning, ideas, moc]
---

# How to use ML on data

**You've been handed a dataset. What can you actually build with it?**

This note is the lookup. Find your data's shape, see every AI/ML task it supports, and follow the link to how it's done.

Index: [[08 — MACHINE LEARNING]] · Method: [[Classical ML in practice]] · Which tool: [[Choosing your approach]]

---

## Before anything: three questions

**1. What decision changes because of this?**

If nobody acts differently on the output, don't build it. "Insight" that nobody uses is a hobby.

**2. Could a rule do it?**

`if temperature > 90: alert` solves a surprising number of problems in an afternoon, runs instantly, never drifts, and everyone understands it. **Try the rule first** — it's also your baseline ([[Problem framing]]).

**3. Do I have the answer written down anywhere?**

This determines everything that follows.

| Do you have labels? | What that unlocks |
|---|---|
| **Yes, a column with the answer** | **Supervised** — classification, regression, forecasting. The most reliable |
| **No** | **Unsupervised** — clustering, anomaly detection, topic discovery |
| **No, but the data labels itself** | **Self-supervised** — "predict the next value/word/frame". How LLMs are trained |
| **No, but I can score outcomes** | **Reinforcement learning** — and read the warning in §RL first |
| **No, but a pretrained model already knows** | **Use someone else's model** — often the correct answer in 2026 |

> **The fastest win in modern AI is not training anything.** A pretrained model plus your data, via embeddings or a prompt, gets you 80% of the value in a day. Train your own when that genuinely isn't good enough.

---

## Step 1 — What shape is my data?

Find your row here. This determines everything.

| Shape | Looks like | Jump to |
|---|---|---|
| **Tabular** | Rows and columns, a spreadsheet or SQL table | [[#Tabular data]] |
| **Time series** | Values with timestamps, in order | [[#Time series and sensor data]] |
| **Text** | Documents, messages, reviews, logs, tickets | [[#Text]] |
| **Images** | Photos, scans, frames, screenshots | [[#Images and video]] |
| **Audio** | Recordings, speech, machine noise | [[#Audio]] |
| **Events / logs** | A stream of "who did what, when" | [[#Events, clicks and user behaviour]] |
| **Graph** | Things connected to other things | [[#Graphs and networks]] |
| **Geospatial** | Coordinates, routes, regions | [[#Geospatial]] |
| **A system you can act on** | A simulator, a game, a controller | [[#Control and decision-making]] |
| **Several of these at once** | | [[#Combining data types]] |

---

## Tabular data

*Rows and columns. Sales, customers, sensors-per-hour, transactions, engine readings.*

**This is most business data, and gradient boosting beats deep learning on it far more often than the internet suggests.**

| You want to… | Task | How |
|---|---|---|
| Predict a **number** (revenue, hours-to-failure, price) | Regression | [[Classical ML in practice]] → LightGBM |
| Predict a **category** (churn/stay, fraud/ok, pass/fail) | Classification | [[Classical ML in practice]] |
| Predict **which of many** (which product, which fault) | Multi-class | [[scikit-learn]] |
| Score a **probability** (how likely to churn) | Classification + `predict_proba` | [[Classical ML in practice]] |
| Find **natural groups** with no labels | Clustering | K-means, DBSCAN → [[scikit-learn]] |
| Find **weird rows** | Anomaly detection | Isolation Forest → [[#Anomaly detection]] |
| Know **which columns matter** | Feature importance | [[Feature engineering]] |
| **Explain one prediction** to a human | SHAP values | [[Classical ML in practice]] |
| Squash 200 columns into a few | Dimensionality reduction | PCA, UMAP → [[scikit-learn]] |
| **Fill in missing values** | Imputation | [[Feature engineering]] |
| Generate realistic fake rows | Synthetic data | SDV, CTGAN |

```python
import lightgbm as lgb                                   # the right default for tabular
model = lgb.LGBMClassifier(n_estimators=500, learning_rate=0.05)
model.fit(X_train, y_train)
```

> **Start with `LogisticRegression`, then LightGBM. Stop there unless it isn't good enough.** A neural network on tabular data is usually slower to train, harder to tune, and no better ([[Choosing your approach]]).

**Ideas most people miss on tabular data:**

- **Uplift modelling** — not "who will buy?" but "who will buy *because* we contacted them?" A different and far more useful question for anything marketing-shaped.
- **Survival analysis** — "how long until X?" when some rows haven't failed yet. Standard regression handles that censoring wrongly.
- **Quantile regression** — predict a *range* (10th–90th percentile), not a point. Enormously more useful for planning than a single number.
- **Two-stage models** — classify "will this be zero?", then regress the amount for the non-zeros. Beats one model on sparse targets like insurance claims.

---

## Time series and sensor data

*Values with a timestamp, in order. Readings, prices, demand, traffic, telemetry.*

> ⚠️ **Everything here has one rule: never let the future leak backwards.** Split by date, never randomly. Windows look backwards only. See [[Feature engineering]].

| You want to… | Task | How |
|---|---|---|
| Predict **the next value** | Forecasting | Below |
| Predict **the next 30 days** | Multi-horizon forecasting | Below |
| Spot **something wrong right now** | Anomaly detection | [[#Anomaly detection]] |
| Predict **failure before it happens** | Predictive maintenance | Classification on windowed features |
| **Classify a whole window** (walking/running, normal/faulty) | Sequence classification | 1D CNN or features + boosting |
| **Estimate remaining useful life** | Regression on degradation | [[Classical ML in practice]] |
| Split a signal into **regimes** | Change point detection | ruptures, HMM |
| **Fill gaps** in a sensor feed | Interpolation / imputation | [[Numerical methods in practice]] |
| **Smooth noisy readings** | Filtering | [[PID and Kalman filters]] |
| Fuse **several unreliable sensors** | Sensor fusion | [[PID and Kalman filters]] |

**Forecasting, in the order you should try:**

| Approach | When |
|---|---|
| **Last value / seasonal naive** | **Always first.** Surprisingly hard to beat |
| Moving average, exponential smoothing | Smooth, low-volume series |
| **Lag features + LightGBM** | ✅ **The best value/effort ratio.** Handles many series and extra columns |
| Prophet | Strong seasonality, holidays, you want it fast |
| ARIMA/SARIMA | One series, you need statistical intervals |
| Deep learning (LSTM, TFT, N-BEATS) | Many related series, long horizons, and you've beaten the above |

> **"Lag features plus gradient boosting" wins most real forecasting problems.** Turn the time series into a tabular one — the value 1/7/28 periods ago, rolling means, day of week — then it's just [[Classical ML in practice]].

> ⚠️ **Always compare against the naive forecast** ("tomorrow = today"). A great many published forecasting models lose to it.

---

## Text

*Documents, reviews, tickets, emails, messages, log lines, notes.*

**This is where the last few years changed the answer completely.** You rarely train a text model from scratch now.

| You want to… | Task | Best current approach |
|---|---|---|
| **Sort text into categories** | Classification | Fine-tune a small model, **or just prompt an LLM** |
| Judge **positive/negative** | Sentiment | Pretrained model, off the shelf |
| **Pull out fields** (names, dates, amounts) | Extraction / NER | **LLM with a JSON schema** — [[Large language models]] |
| **Summarise** | Summarisation | LLM |
| **Answer questions about my documents** | **RAG** | [[#RAG — the pattern to reach for first]] |
| **Find similar documents** | Semantic search | Embeddings + vector search |
| **Group documents** with no labels | Clustering / topic modelling | Embeddings + K-means, or BERTopic |
| **Detect duplicates** | Near-duplicate detection | Embeddings + cosine similarity |
| Translate | Translation | LLM or a dedicated model |
| **Route a ticket to the right team** | Classification | Embeddings + LightGBM, or an LLM |
| Generate text | Generation | LLM |
| Redact personal data | NER + masking | [[Security in practice]] |

### RAG — the pattern to reach for first

**"Answer questions about my documents"** is the single most common real request, and the answer is almost never fine-tuning.

```
your documents -> split into chunks -> embed each chunk -> store the vectors
                                                                |
user question -> embed it -> find the most similar chunks ------+
                                    |
                     put those chunks in the prompt -> LLM answers, citing them
```

> ⚠️ **Fine-tuning does not teach a model facts.** It teaches *style, format and behaviour*. If you want the model to know your company's policies, **retrieve them and put them in the prompt** — that's RAG. This is the most common expensive mistake in applied LLM work ([[Large language models]]).

| Need | Use |
|---|---|
| Model must **know your facts** | **RAG** |
| Model must **follow a format or tone** | Fine-tuning |
| Model must **do a task well with examples** | Few-shot prompting first |
| Model must **use tools / take actions** | Agents + tool calling |

Store vectors in **pgvector** ([[PostgreSQL reference]]) before reaching for a dedicated vector database — Postgres handles millions of vectors comfortably.

### Embeddings — the underrated tool

An embedding turns any text into a list of numbers where **similar meanings are close together**.

```python
# one model, and suddenly you can do all of these
embeddings = model.encode(documents)      # -> (n_docs, 384) array
```

That single array gives you: semantic search, clustering, deduplication, recommendation, outlier detection, and **features for a normal LightGBM classifier**. Embeddings + gradient boosting often beats fine-tuning a transformer, on far less data ([[Transformers and LLM basics]]).

---

## Images and video

| You want to… | Task | How |
|---|---|---|
| **Label a whole image** (defect / ok) | Classification | [[CNNs and transfer learning]] |
| **Find and box objects** | Object detection | YOLO, DETR |
| **Outline exact shapes** | Segmentation | U-Net, SAM |
| Read text from an image | OCR | Tesseract, or a vision LLM |
| **Find similar images** | Image embeddings | CLIP |
| **Spot defects with no defect examples** | Anomaly detection | Autoencoder, PatchCore |
| Count things | Detection + counting | YOLO |
| Track something across frames | Object tracking | ByteTrack, DeepSORT |
| Estimate pose or keypoints | Pose estimation | MediaPipe |
| **Ask questions about an image** | Vision-language | A multimodal LLM |
| Generate or edit images | Diffusion | Stable Diffusion |

> **Never train an image model from scratch.** Start with a pretrained backbone and fine-tune the last layers — **transfer learning**. It works with hundreds of images instead of millions ([[CNNs and transfer learning]]).

> **With fewer than ~100 labelled images per class**, try CLIP embeddings + logistic regression before any fine-tuning. It often just works.

---

## Audio

| You want to… | Task | How |
|---|---|---|
| **Speech to text** | ASR | Whisper |
| **Classify a sound** (machine fault, alarm) | Audio classification | Spectrogram → CNN |
| Identify the speaker | Speaker ID | Speaker embeddings |
| **Detect an unusual noise** | Anomaly detection | Autoencoder on spectrograms |
| Detect a wake word | Keyword spotting | Small CNN, on-device |

> **The trick for audio: turn it into an image.** A spectrogram is a picture of sound over time — then every technique in [[#Images and video]] applies, [[CNNs and transfer learning]] included.

---

## Events, clicks and user behaviour

*"User 42 viewed product 7 at 10:03." Logs, clickstreams, transactions, app telemetry.*

| You want to… | Task | How |
|---|---|---|
| **Recommend what's next** | Recommendation | Collaborative filtering, then two-tower |
| **Predict who will leave** | Churn prediction | Aggregate to tabular → [[Classical ML in practice]] |
| Predict **lifetime value** | Regression | Aggregate to tabular |
| **Group users by behaviour** | Segmentation | RFM features + K-means |
| Find the path to conversion | Sequence mining | Markov chains, attribution models |
| **Spot fraud or abuse** | Anomaly detection | [[#Anomaly detection]] |
| Order a list well | Learning to rank | LambdaMART, LightGBM ranker |
| Decide which version wins | A/B testing, bandits | [[#Control and decision-making]] |

> **The move that unlocks all of these: aggregate events into one row per user.** Count, recency, frequency, monetary value, rate of change. Then it's a tabular problem with everything in [[Feature engineering]] available.

---

## Graphs and networks

*Things connected to things. Users↔users, parts↔assemblies, accounts↔transactions.*

| You want to… | Task |
|---|---|
| Find communities | Community detection (Louvain) |
| Find the important nodes | Centrality (PageRank) |
| **Predict a missing connection** | Link prediction |
| **Classify a node using its neighbours** | Graph neural network |
| **Spot fraud rings** | Graph anomaly detection |

> **Try graph *features* in a normal tabular model first** — a node's degree, its PageRank, its neighbours' average. That's usually most of the benefit for a fraction of the complexity of a GNN.

---

## Geospatial

| You want to… | How |
|---|---|
| Predict by area | Aggregate to a grid (H3 hexes) → tabular |
| Find hotspots | DBSCAN on coordinates |
| Optimise routes | OR-tools — **optimisation, not ML** |
| Estimate arrival time | Regression with distance/traffic features |
| Classify land use from imagery | [[CNNs and transfer learning]] |

---

## Anomaly detection

*Worth its own section — it's the most commonly wanted task and the most commonly done badly.*

**First: which kind of "weird" do you mean?**

| Kind | Example | Approach |
|---|---|---|
| **A weird value** | Temperature of 400° | A rule, or IQR bounds |
| **A weird combination** | Small order, huge shipping cost | Isolation Forest |
| **A weird moment in time** | Vibration spike | Rolling z-score, or a forecast residual |
| **A weird sequence** | Unusual click path | Autoencoder, sequence model |
| **A gradual drift** | Sensor slowly miscalibrating | Compare distributions over time |

```python
from sklearn.ensemble import IsolationForest
iso = IsolationForest(contamination=0.01, random_state=42)   # expect ~1% anomalies
scores = iso.fit_predict(X)                                  # -1 = anomaly, 1 = normal
```

> ⚠️ **Start with statistics, not ML.** A rolling z-score or IQR bound catches most real anomalies, takes ten minutes, and can be explained to anyone. Reach for Isolation Forest or an autoencoder when that genuinely isn't enough.

> ⚠️ **You almost never have labelled anomalies, so you cannot measure accuracy properly.** Decide up front how many alerts per day a human can actually action, and tune the threshold to *that*. An anomaly detector nobody trusts gets switched off in a fortnight ([[Monitoring and iteration]]).

> **If you *do* have labels, it's just imbalanced classification** — and a supervised model will beat an unsupervised one comfortably.

---

## Control and decision-making

*You don't just want a prediction — you want to choose an action.*

| You want to… | Use | Not RL? |
|---|---|---|
| Hold a value at a setpoint | **PID controller** | [[PID and Kalman filters]] |
| Plan with a known model | MPC | [[22 — CONTROL SYSTEMS]] |
| Route, schedule, pack | **Optimisation** (OR-tools) | Not ML at all |
| Choose between a few options, learn as you go | **Multi-armed bandit** | Far simpler than RL |
| Learn a complex policy by trial and error | **RL** | [[Reinforcement learning in depth]] |

> ⚠️ **Reinforcement learning is almost always the wrong first answer.** It needs a simulator or millions of cheap trials, it's unstable to train, and it will exploit any flaw in your reward function — "reward hacking" is the norm, not the exception. **A PID controller, an optimiser or a bandit solves most real problems**, and you can explain them to a regulator ([[Reinforcement learning in depth]]).

> **Where RL genuinely fits:** games, simulators, robotics with a good sim, and tuning long-horizon decisions where you can afford millions of cheap attempts.

---

## Combining data types

The interesting problems are usually multimodal.

| Combination | Approach |
|---|---|
| Tabular + text | Embed the text, add the vector as features |
| Tabular + images | Embed images with CLIP, add as features |
| Sensor + maintenance logs | Window the sensors, join the log events |
| Everything | Embed each part, concatenate, feed one model |

> **The general recipe: turn each modality into a vector, glue the vectors together, put a normal model on top.** You get most of the benefit of a bespoke multimodal architecture with a fraction of the work.

---

## Reality check — will it actually work?

Before promising anything:

| Question | Bad sign |
|---|---|
| **How many labelled examples?** | Under ~1,000 for tabular; under ~100 images per class |
| **How balanced?** | Under 1% positives — you need PR-AUC and probably resampling |
| **Will the feature exist at prediction time?** | If not, it's leakage — [[Feature engineering]] |
| **How fast must the answer be?** | Under 100 ms rules out large models |
| **How often does the world change?** | Fast drift means a retraining pipeline, not a model |
| **Does anyone need to understand it?** | Regulated → linear or trees, not deep learning |
| **What does being wrong cost?** | High → human in the loop, not automation |
| **Is there a rule that does 80% of it?** | Then build the rule |

> **The honest failure mode:** most ML projects die from not enough labels, leakage, or nobody using the output. Very few die from picking the wrong algorithm.

---

## The five-minute triage

Given any new dataset:

1. **`df.printSchema()` / `df.head()`** — what shape is this? ([[PySpark reference]] · [[pandas]])
2. **Is there a column that's the answer?** → supervised. If not → clustering, anomalies, or a pretrained model
3. **Is there a timestamp?** → forecasting is possible, and **you must split by date**
4. **How many rows, and how balanced?** → decides deep learning vs boosting vs "not yet"
5. **Find your shape in [[#Step 1 — What shape is my data?]]** → pick a task
6. **Write down the baseline you must beat** ([[Problem framing]])
7. **Build the stupid version first** ([[Classical ML in practice]])

---

## Related

[[08 — MACHINE LEARNING]] · [[Classical ML in practice]] · [[Feature engineering]] · [[Problem framing]] · [[Choosing your approach]] · [[scikit-learn]] · [[09 — DEEP LEARNING]] · [[CNNs and transfer learning]] · [[Transformers and LLM basics]] · [[Large language models]] · [[LLM and GenAI track]] · [[Reinforcement learning in depth]] · [[11 — COMPUTER VISION]] · [[PID and Kalman filters]] · [[PySpark reference]] · [[Model export and serving]] · [[Monitoring and iteration]] · [[MLflow experiment tracking]] · [[The playbook]]
