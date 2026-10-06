---
tags: [playbook, production, process]
status: not-started
---

# Problem framing

> **What this is:** phases 0–2 of [[The playbook]] in depth — deciding what you're actually building, checking the data exists, and building the baseline.
> **Why you care:** this is where projects are won or lost, and it happens before you write a line of model code. It's also the part everyone skips because it doesn't feel like work.

---

## The idea in plain English

Imagine someone says: *"we should use AI to improve customer retention."*

That sentence contains no information. It doesn't say what you'd predict, who'd act on it, what they'd do differently, or how you'd know it helped.

Framing is the work of turning that sentence into something a person could actually build, and then checking it's worth building. **Most of the value is in the questions you ask, not the answers.**

---

## 1. Does this need machine learning at all?

Ask this first, honestly, every time.

Machine learning is for problems where **you can't write down the rules** — either because nobody knows them, there are thousands of them, or they change constantly.

| Use rules / SQL when | Use ML when |
|---|---|
| The logic is knowable and writable | The pattern is real but nobody can articulate it |
| It rarely changes | It shifts over time |
| You need to explain every decision exactly | Approximate is fine, and you can measure it |
| You have no labelled examples | You have thousands of labelled examples |
| Errors are unacceptable | Errors are tolerable and recoverable |

```sql
-- This is not a fraud model. This is an if statement.
SELECT * FROM orders WHERE amount > 10000 AND country <> billing_country;
```

> **Rules are better than models whenever they work.** They're instant, free, exactly explainable, they never drift, they need no retraining, no GPU, no monitoring, no feature store. A rule that gets 80% of the value for 1% of the effort is the correct engineering decision, and choosing it is a sign of seniority, not laziness.
>
> The honest pattern in industry: **start with rules, and let ML earn its way in** when the rules visibly stop coping.

### The three-question filter

1. **Is there a pattern?** If outcomes are genuinely random (which horse wins), no model helps.
2. **Can you not write it down?** If you can, write it down.
3. **Do you have data with the answer in it?** If not, you have a data collection project first.

If any answer is "no", stop. Stopping here has saved more engineer-months than any optimisation ever will.

---

## 2. The framing document

One page. Written before any code. Six sections:

### 2.1 The decision

> *"Every week, the marketing team picks 500 customers to send a retention offer to. Today they pick by total spend. We want to pick the 500 most likely to churn who would stay if contacted."*

Concrete: who acts, how often, what changes.

> **If you cannot finish the sentence "because of this model, ___ will do ___ differently", the project has no purpose yet.** That's not a criticism of the idea, it's a signal that the framing isn't finished.

### 2.2 The prediction

Be precise. These are all different problems:

| Vague | Precise |
|---|---|
| "Predict churn" | "Probability a customer makes no purchase in the next 90 days, scored on the 1st of each month" |
| "Forecast sales" | "Total revenue per product category for each of the next 4 weeks, produced every Monday" |
| "Segment customers" | "Assign each customer to one of 4 named segments, refreshed monthly" |

The precise version tells you the **target**, the **grain**, and the **cadence** — which between them determine your entire pipeline.

### 2.3 The output and its consumer

- What shape? A number, a class, a probability, a ranked list?
- Who reads it — a human, a dashboard, another system?
- **A probability needs a threshold.** Who chooses it, and on what basis?

> A human consumer means you need an explanation alongside the number. A system consumer means you need an SLA. These lead to different architectures — decide now, not after.

### 2.4 The cost of being wrong

| | Predicted churn | Predicted stay |
|---|---|---|
| **Actually churned** | ✅ Sent offer, saved them | ❌ Lost the customer |
| **Actually stayed** | ⚠️ Wasted a £10 voucher | ✅ Nothing spent |

Here a false negative (losing a customer) costs far more than a false positive (a wasted voucher). So you tune for **recall** and accept more false alarms.

> **This table decides your metric.** Without it, people default to accuracy, which weights both errors equally — almost never what the business wants. Draw the table, put rough numbers in the cells, and your metric chooses itself.

### 2.5 Latency and volume

| Question | Why |
|---|---|
| How fast must the answer come back? | Batch vs. real-time — the biggest architecture fork ([[Deployment patterns]]) |
| How many predictions a day? | 500/month is a spreadsheet. 500/second is infrastructure. |
| How fresh must the input be? | Yesterday's data is often completely fine |

> Most people answer "real-time" reflexively. **Ask instead: could this be computed overnight and looked up?** For scoring all customers monthly, obviously yes. That single answer removes an enormous amount of complexity.

### 2.6 How you'd know it worked

Two different metrics, and you need both:

- **The model metric** — MAE, F1, AUC. What you optimise.
- **The business metric** — retention rate, revenue, hours saved. What actually matters.

> **They can move in opposite directions**, and when they do the business metric wins. A model with better AUC that flags customers nobody can contact is worse than a weaker model that flags reachable ones. State both up front so you notice the divergence.

---

## 3. Checking the data

**Do this in week one.** More projects die here than anywhere else, and dying in week one is cheap.

### The checklist

- [ ] **Does it exist?** Which system, which table, who owns it
- [ ] **Can you get it?** Access requests can take weeks — start immediately
- [ ] **Can you legally use it?** Personal data, consent, retention limits, region
- [ ] **Is there a label?** The single most common blocker
- [ ] **How much, and how far back?** Seasonality needs 2+ years
- [ ] **How dirty?** Missing, duplicated, obviously wrong
- [ ] **Will it keep arriving?** A model on a deprecated source is dead on arrival

### The label problem

Supervised learning needs examples **with the answer attached**. Most organisations have vast data and almost no labels.

| Situation | What you actually have |
|---|---|
| Historical churn recorded | ✅ Labels. Proceed. |
| "We know it when we see it" | ❌ A labelling project |
| Labels from a rule already in use | ⚠️ Your model can only learn the rule. Ceiling = the rule. |
| Labels from one person's judgement | ⚠️ You're modelling that person, biases included |

> **The trap in row 3 is subtle and common.** If your "fraud" labels came from an existing rules engine, a model trained on them learns to imitate the rules — including everything the rules currently miss. It can never exceed them. You need labels from *outcomes* (confirmed chargebacks), not from the system you're replacing.

**When there are no labels:** hand-label a few hundred yourself (it's tedious, and it teaches you more about the data than any amount of plotting), use weak supervision, use an LLM to bootstrap labels you then spot-check ([[LLM and GenAI track]]), or reframe as unsupervised.

### Target leakage — the one that will get you

> **Leakage is when a feature contains information you would not have at prediction time.** The model finds it, scores brilliantly offline, and fails completely in production.

Real examples:

| Feature | Why it leaks |
|---|---|
| `refund_issued` predicting churn | Only known *after* they churn |
| `account_closed_date` | Same |
| `n_support_calls_total` | Includes calls made after your prediction point |
| A field back-filled after the outcome | Updated retrospectively; historical rows have future knowledge |
| Row ID correlated with the target | Data was sorted by outcome before export |

**The test, applied to every single feature:**

> *"If I were making this prediction on the morning of 1 March, would this exact value be in the database, with this exact content?"*

If the answer is anything other than a confident yes, drop it or reconstruct it as-of.

**The smell test:** if your first model scores 0.99, you have leakage. Genuinely. Real problems are hard. An implausibly good result is a bug, not a triumph — go and look for it before you tell anyone.

### The other leaks

| Leak | What happens | Fix |
|---|---|---|
| **Temporal** | Random split puts the future in training | Split by time ([[Tensors, autograd and the training loop]]) |
| **Group** | Same customer in train and test | Split by customer, not by row |
| **Preprocessing** | Scaler fitted on all data | `fit` on train only |
| **Duplicate rows** | Same record in train and test | Deduplicate before splitting |

---

## 4. Exploring it

Before modelling, sit with the data for a day. In Databricks ([[PySpark core]]):

```python
from pyspark.sql import functions as F

df.printSchema()
df.count()
df.describe().show()

# missingness per column
df.select([
    (F.count(F.when(F.col(c).isNull(), c)) / F.count("*")).alias(c)
    for c in df.columns
]).show()

# the target — is it balanced?
df.groupBy("churned").count().show()

# does it drift over time?
df.groupBy(F.year("date"), F.month("date")).agg(F.avg("target")).orderBy("year","month").show()
```

**What you're looking for:**

- A target that's 99% one class → accuracy is meaningless, and you may need resampling or class weights
- Distributions that shift over time → your test period must be the recent one
- Columns that are 90% null → probably unusable
- A column that predicts the target almost perfectly → **leakage, until proven otherwise**
- Duplicates → will inflate every metric

> **Look at 20 individual rows with your own eyes.** Not summary statistics — actual records. You will find something that no aggregate would have shown you: a placeholder date of 1900-01-01, a test customer called "asdf", a price in pence where you assumed pounds. Every time.

---

## 5. The baseline

**Twenty minutes. Before any model.**

| Problem | Baseline |
|---|---|
| Regression | Predict the mean / median |
| Time series | Predict last period's value (naive) |
| Seasonal time series | Predict same period last year |
| Classification | Predict the majority class |
| Ranking | Sort by popularity |
| **Any** | **The rule or human process in use today** |

```python
from sklearn.metrics import f1_score

# forecasting
naive_mae = (test.revenue - test.revenue_lag_1).abs().mean()

# classification
from sklearn.dummy import DummyClassifier
dummy = DummyClassifier(strategy="most_frequent").fit(X_train, y_train)
baseline_f1 = f1_score(y_test, dummy.predict(X_test), average="weighted")
```

**Why this matters so much:**

1. **It gives your metric meaning.** "MAE of 412" is noise. "MAE of 412 vs. a baseline of 498" is a result.
2. **It sometimes wins.** If the naive forecast is nearly as good, you've saved months.
3. **It tells you if the problem is learnable.** No approach beating the baseline usually means the signal isn't in your features.
4. **It's your fallback.** When the model breaks at 3am, the baseline is what you serve.

> **Then add a second baseline: simple classical ML.** Logistic regression or a small gradient-boosted tree, on your raw features, with no tuning — ten minutes. It's astonishing how often that is within 2% of the deep learning model you were about to spend three weeks on. If it is, ship it: it trains in seconds, explains itself, and needs no GPU.

**Log every baseline in MLflow** ([[MLflow experiment tracking]]) so every future run is automatically comparable.

---

## 6. The go / no-go

Before phase 3, you should be able to answer all of these:

- [ ] The decision this changes, in one sentence
- [ ] The exact prediction: target, grain, cadence
- [ ] Who consumes the output and what they do with it
- [ ] The cost of each error type, and the metric that follows
- [ ] Latency and volume requirements → batch or real-time
- [ ] Model metric **and** business metric
- [ ] Data exists, is accessible, is legal to use
- [ ] Labels exist and come from outcomes, not from the system being replaced
- [ ] Every feature passes the leakage test
- [ ] A baseline number exists

**If more than two are unanswered, don't start modelling.** Go and answer them — it's faster than building the wrong thing.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| Model scores 0.99 on the first try | Leakage | Apply the prediction-time test to every feature |
| Great offline, useless in production | Leakage, or temporal split done randomly | Split by time; audit features |
| Can't beat the baseline | Signal isn't in your features, or the problem isn't learnable | Better features; revisit whether ML is right |
| Stakeholders unhappy despite good metrics | Model metric ≠ business metric | Go back to 2.6 |
| Model can never exceed the current rules | Labels generated by those rules | Get outcome-based labels |
| 95% accuracy, worthless model | Class imbalance | F1 and confusion matrix; check the baseline |
| Weeks lost waiting for data access | Access requested late | Request in week one, always |
| Nobody uses the finished model | No named decision or consumer | 2.1 and 2.3 were skipped |

---

## Practice checklist

- [ ] The "does this need ML at all" filter and the three-question test
- [ ] Writing a precise prediction statement: target, grain, cadence
- [ ] Naming the decision, the actor, and what changes
- [ ] The error-cost table, and how it picks your metric
- [ ] Model metric vs. business metric
- [ ] The data checklist, and starting access requests immediately
- [ ] **The label problem** — especially labels generated by the system you're replacing
- [ ] **Target leakage** and the prediction-time test
- [ ] Temporal, group, and preprocessing leakage
- [ ] Looking at 20 real rows by eye
- [ ] Building the naive baseline, then the simple-classical-ML baseline

## Hands-on

- [ ] Write the one-page framing document for the capstone, retrospectively
- [ ] Draw the error-cost table for customer segmentation and decide the metric from it
- [ ] Audit the capstone features with the prediction-time test — is anything leaking?
- [ ] Compute both baselines for the forecasting model and log them
- [ ] Take a real idea of your own and run it through the go/no-go list

## Resources

- [Google: Rules of Machine Learning](https://developers.google.com/machine-learning/guides/rules-of-ml) — Rule 1 is "don't be afraid to launch a product without machine learning"
- [Kaggle: Data Leakage](https://www.kaggle.com/code/alexisbcook/data-leakage)

## Next

[[Choosing your approach]]
