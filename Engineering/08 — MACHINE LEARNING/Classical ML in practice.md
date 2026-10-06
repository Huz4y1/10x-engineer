---
tags: [machine-learning, sklearn, deep-dive]
---

# Classical ML in practice

The index is [[08 — MACHINE LEARNING]]. This is the working version — the order you actually do things in, and the traps at each step.

API: [[scikit-learn]] · Should you use ML at all: [[Problem framing]] · Which model: [[Choosing your approach]]

---

## The idea, plainly

Normally you write rules: *if the temperature is above 90, alert.*

Machine learning flips it. You give the computer **examples with answers** — a thousand engine readings, each labelled *failed* or *fine* — and it works out the rules itself.

That's it. **Examples in, rules out.**

The catch: it can only find patterns that are actually in your examples. If nothing in your data distinguishes a failing engine from a healthy one, no algorithm will invent the difference. This is why [[Problem framing]] comes before any modelling.

---

## The order you do it in

```
1. Write down the question and how you'd measure success
2. Get the data. Look at it. Actually look at it.
3. Split it - BEFORE you touch anything else
4. Build a stupid baseline
5. Build a simple model
6. Only then try something clever
7. Check it fails safely
8. Ship it, watch it, retrain it
```

> **Most failed ML projects failed at step 1 or 2.** Not at step 6.

---

## Step 1 — The question

Bad: *"Use ML on our sensor data."*
Good: *"Given the last 24 hours of readings, predict whether this engine needs maintenance in the next 7 days. Right now maintenance is scheduled every 500 hours regardless. Success = catching 80% of failures with under 20% false alarms."*

Three things a good question has:

| Part | Example |
|---|---|
| **Input, precisely** | 24 h of readings, available at prediction time |
| **Output, precisely** | Needs maintenance in 7 days: yes/no |
| **The bar to beat** | Today's fixed schedule |

> **If you can't name what you're beating, you can't tell whether it worked.** The thing to beat is usually a rule, not a model.

---

## Step 2 — Look at the data

```python
df.shape
df.head(20)
df.describe()
df.isna().sum()
df["target"].value_counts(normalize=True)      # class balance
df.corr(numeric_only=True)["target"].sort_values()
df.duplicated().sum()
```

Ask, every time:

- **How balanced is the target?** 99/1 changes everything.
- **What's missing, and is it missing for a reason?** A blank `discharge_date` might mean "still in hospital" — which is a huge clue, not a gap.
- **Any column correlating suspiciously well?** Over about 0.95, suspect leakage before celebrating.
- **Is there a time dimension?** If yes, your splitting rules change completely.

Full toolkit: [[pandas]] · [[Polars]] · at scale [[PySpark reference]].

---

## Step 3 — Split, before anything else

```python
from sklearn.model_selection import train_test_split
X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, stratify=y, random_state=42)
```

| Set | Purpose | How often you may look |
|---|---|---|
| **Train** | Model learns from it | Constantly |
| **Validation** | Tuning and model choice | Often |
| **Test** | The final honest number | **Once, at the end** |

> ⚠️ **Split first, explore second.** If you compute the mean, pick features, or choose a model while looking at the test set, the test set is no longer honest. It has quietly become training data.

> ⚠️ **Never use a random split on time series.** It puts the future in the training set. Your score will be excellent and the model will be worthless. Split by date, or use `TimeSeriesSplit`.

> ⚠️ **Split by *entity*, not by row, when rows repeat.** Ten readings from engine #7 must all be on the same side of the split. Otherwise the model recognises engine #7 rather than learning anything.

---

## Step 4 — The stupid baseline

```python
from sklearn.dummy import DummyClassifier, DummyRegressor
DummyClassifier(strategy="most_frequent").fit(X_tr, y_tr).score(X_te, y_te)
DummyRegressor(strategy="mean").fit(X_tr, y_tr)
```

Then the domain baseline: whatever a sensible person would do without ML.

> **Do this. It takes thirty seconds and it saves projects.** If 97% of engines are fine, "always predict fine" scores 97%. Your neural network scoring 96% is *worse than a constant*. Without the baseline you'd have shipped it.

---

## Step 5 — Leakage, the thing that ruins everything

**Leakage is when information sneaks into training that won't exist at prediction time.**

You're predicting tomorrow's weather, and one of your columns is tomorrow's weather. Obvious like that — never obvious in real data.

| Type | Example | Tell |
|---|---|---|
| **Target leakage** | `days_until_failure` used to predict failure | Accuracy near 1.0 |
| **Time leakage** | Random split on time-ordered data | Great offline, bad live |
| **Preprocessing leakage** | Scaler fitted on all data before splitting | Small, quiet overestimate |
| **Group leakage** | Same patient in train and test | Model memorises the entity |
| **Availability leakage** | `final_invoice_amount` when predicting at order time | Field is filled in *later* |

> ⚠️ **A score of 0.99 on your first attempt is not success. It is a leakage alarm.** Real problems are hard. Go and find the leak — it's there.

**The test that catches most of it:** for every column, ask *"would I actually have this value, filled in, at the moment I need the prediction?"* If not, drop it.

The structural defence is a [[scikit-learn]] `Pipeline` — it makes preprocessing leakage impossible during cross-validation.

---

## Step 6 — Features

Feature engineering usually beats model choice. **Full detail: [[Feature engineering]]**.

```python
import numpy as np

# Time
df["hour"] = df["ts"].dt.hour
df["dow"] = df["ts"].dt.dayofweek
df["is_weekend"] = df["dow"] >= 5

# Cyclical - so 23:00 and 00:00 are close, not far apart
df["hour_sin"] = np.sin(2 * np.pi * df["hour"] / 24)
df["hour_cos"] = np.cos(2 * np.pi * df["hour"] / 24)

# Rolling - the shape of recent history
df["temp_mean_24h"] = df.groupby("engine")["temp"].transform(lambda s: s.rolling(24).mean())
df["temp_std_24h"]  = df.groupby("engine")["temp"].transform(lambda s: s.rolling(24).std())
df["temp_delta"]    = df.groupby("engine")["temp"].diff()

# Ratios often carry more signal than the raw parts
df["load_per_rpm"] = df["load"] / df["rpm"].replace(0, np.nan)
```

> **Encode the physics you know.** "Temperature rose 15° in an hour" is a far stronger signal than the raw temperature, and no model will construct it for you from a single column ([[Physics]], [[PID and Kalman filters]]).

> ⚠️ **A rolling window must only look backwards.** `rolling(24)` is fine; anything centred on the current point uses the future and is leakage.

| Feature type | Treatment |
|---|---|
| Numeric | Scale for linear models / KNN / PCA; **trees don't need it** |
| Categorical, few values | One-hot, `handle_unknown="ignore"` |
| Categorical, many values | Target encoding — **fit inside the CV fold or it leaks** |
| Ordered categorical | Ordinal encoding |
| Text | TF-IDF, or embeddings ([[Large language models]]) |
| Dates | Explode into parts; **never feed a raw timestamp** |

---

## Step 7 — Pick a model

```python
from sklearn.linear_model import LogisticRegression
import lightgbm as lgb

LogisticRegression(max_iter=1000, class_weight="balanced")     # always run this first
lgb.LGBMClassifier(n_estimators=500, learning_rate=0.05)       # then this
```

| Data | Use |
|---|---|
| **Tabular** (rows and columns) | **Gradient boosting** — LightGBM/XGBoost |
| Tabular, must explain every decision | Logistic regression or a shallow tree |
| Images, audio, text, sequences | Deep learning — [[Tensors, autograd and the training loop]] |
| Bigger than one machine | [[PySpark reference]] MLlib |
| Genuinely tiny data (<1000 rows) | Linear model, and be sceptical |

> **On tabular data, gradient boosting beats deep learning most of the time.** Faster, less tuning, handles missing values and mixed types natively, and it tells you which features mattered. The internet over-recommends neural networks ([[Choosing your approach]]).

---

## Step 8 — Measure the right thing

```python
from sklearn.metrics import classification_report, confusion_matrix
print(confusion_matrix(y_te, y_pred))
print(classification_report(y_te, y_pred))
```

```
                 predicted fine    predicted fail
actually fine         950 (TN)          20 (FP)   <- false alarm
actually fail          10 (FN)          20 (TP)   <- MISSED FAILURE
```

| Metric | Plain English | Use when |
|---|---|---|
| **Precision** | Of my alarms, how many were real? | False alarms are costly |
| **Recall** | Of the real events, how many did I catch? | **Misses are costly** |
| **F1** | Balance of the two | General imbalanced case |
| **ROC-AUC** | Ranking quality across all thresholds | Comparing models |
| **PR-AUC** | Same, for rare positives | **Under ~5% positive rate** |
| MAE | Average error, in real units | Regression you must explain |
| RMSE | Penalises large errors more | Big errors are disproportionately bad |

> ⚠️ **Accuracy is a trap with imbalanced classes.** 99% accuracy on a 1%-failure dataset is what you get by predicting "fine" forever.

> **Decide which error hurts more before you look at the scores.** Missing an engine failure costs an engine. A false alarm costs an hour of inspection. That ratio, not the maths, sets your threshold.

### The threshold is a business decision

```python
proba = model.predict_proba(X_te)[:, 1]
y_pred = (proba > 0.30).astype(int)      # 0.5 is a default, not a law
```

> **`predict()` silently uses 0.5.** Lower it to catch more (more false alarms); raise it to be more certain (more misses). Tune it on validation against the actual cost of each error.

---

## Step 9 — Validate honestly

```python
from sklearn.model_selection import cross_val_score, RandomizedSearchCV

cross_val_score(pipeline, X_tr, y_tr, cv=5, scoring="f1")      # pipeline, not bare model

search = RandomizedSearchCV(pipeline, param_dist, n_iter=50, cv=5,
                            scoring="f1", n_jobs=-1, random_state=42)
```

> **Always cross-validate the whole Pipeline, never a bare model.** Otherwise the scaler saw the validation fold and your score is optimistic.

> **`RandomizedSearchCV` beats `GridSearchCV` beyond about three hyperparameters** — comparable results in a fraction of the time.

> **Report the standard deviation, not just the mean.** `0.84 ± 0.09` across folds means you don't really know your score to two decimal places.

---

## Step 10 — Understand what it learned

```python
from sklearn.inspection import permutation_importance
r = permutation_importance(model, X_te, y_te, n_repeats=10, random_state=42)
```

> **Permutation importance is more trustworthy than tree `feature_importances_`,** which is biased toward high-cardinality columns.

> **Read the top features and ask if they make sense.** If `row_id` or `customer_id` is your best feature, you have leakage or a sorted dataset. This check catches bugs nothing else does.

---

## Step 11 — Ship it

```python
import joblib
joblib.dump(pipeline, "model.joblib")     # the WHOLE pipeline, not just the estimator
```

> ⚠️ **Save the pipeline, never the bare model.** The scaler and encoder are part of the model. Saving only the estimator is how you get "plausible but wrong" predictions in production ([[Model export and serving]]).

> ⚠️ **`joblib.load` executes arbitrary code — only load files you produced yourself** ([[Security in practice]]).

**The contract you must write down:** exact feature order, exact dtypes, exact category values, the scaler, and the threshold. Getting the feature order wrong produces predictions that look reasonable and are nonsense — the hardest bug in ML to spot.

Serving: [[FastAPI reference]] · Tracking: [[MLflow experiment tracking]] · Monitoring: [[Monitoring and iteration]]

---

## Step 12 — It gets worse over time

**Drift.** The world changes; your model doesn't.

| Kind | What changed | Example |
|---|---|---|
| **Data drift** | The inputs | New sensor model reads 2° higher |
| **Concept drift** | Input→output relationship | New engine design fails differently |
| **Upstream drift** | The pipeline | A column silently became a string |

> **Monitor input distributions, not just accuracy** — you usually get labels months late, but you can detect the inputs shifting today ([[Observability for data and ML pipelines]]).

---

## Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| No baseline | Can't tell if the model helps | `DummyClassifier` first |
| Explored before splitting | Optimistic test score | Split first |
| Random split on time data | Great offline, useless live | Split by date |
| Leakage | Score near 1.0 | Audit every column for availability |
| Accuracy on imbalanced data | 99%, useless model | F1, PR-AUC, confusion matrix |
| Scaler fitted on all data | Quiet overestimate | Use a `Pipeline` |
| Saved model without preprocessing | Plausible but wrong output | Save the whole pipeline |
| Used the default 0.5 threshold | Wrong error trade-off | Tune it against real costs |
| Tuned against the test set | Test set no longer honest | Validation set for tuning |
| Never looked at feature importance | Missed obvious leakage | Always look |
| Deployed and forgot | Quiet decay over months | Monitor drift |

## Related

[[08 — MACHINE LEARNING]] · [[scikit-learn]] · [[Problem framing]] · [[Choosing your approach]] · [[pandas]] · [[NumPy]] · [[MLflow experiment tracking]] · [[Model export and serving]] · [[Monitoring and iteration]] · [[09 — DEEP LEARNING]] · [[Mathematics reference]] · [[Security in practice]]
