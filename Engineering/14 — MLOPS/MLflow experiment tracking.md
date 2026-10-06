---
tags: [mlflow, experiment-tracking, mlops]
status: not-started
---

# MLflow & Experiment Tracking

> **What this is:** a logbook for machine learning experiments, plus a shelf for the models that come out of them.
> **Why you care:** it's the same MLflow that ships inside Databricks. Learn it once, use it in both places. It's also the thing that separates "I trained a model" from "I can tell you exactly how, on what data, and reproduce it."

---

## The problem it solves

You train a model. Accuracy 0.82.

You change the learning rate. 0.84.

You add a layer. 0.83.

You change the batch size and the layer count together. 0.87. 

Three days later: **which combination gave you 0.87?** Which file is that model? What data was it trained on? Was that the run where you'd also fixed the null handling?

Every ML person has kept this in a spreadsheet, and every spreadsheet drifts out of sync with reality within a week. The notebook that produced the good result has been edited eleven times since.

MLflow makes logging automatic and attaches it to the model file itself.

---

## The four concepts

| Concept | What it is |
|---|---|
| **Run** | One training execution. The unit of everything. |
| **Experiment** | A named group of runs — "sales-forecasting" |
| **Parameters** | Inputs you chose: learning rate, layers, epochs. Logged once. |
| **Metrics** | Outputs measured: loss, MAE, accuracy. Can be logged per epoch. |
| **Artifacts** | Files: the model, plots, the scaler, feature lists |
| **Model Registry** | A named, versioned shelf for models you've decided to keep |

**Params vs. metrics, the rule:** a parameter is something you *set*. A metric is something you *measure*. Learning rate is a param. Validation loss is a metric. Params are single values; metrics can have a value per step, so MLflow draws a curve.

---

## 1. The minimum useful version

```python
import mlflow

mlflow.set_experiment("sales-forecasting")

with mlflow.start_run(run_name="baseline-lstm"):
    mlflow.log_params({"lr": 1e-3, "hidden": 64, "epochs": 50, "batch_size": 32})

    model, history = train(...)

    mlflow.log_metrics({"val_mae": val_mae, "test_mae": test_mae})
    mlflow.pytorch.log_model(model, "model")
```

```bash
mlflow ui        # http://localhost:5000
```

That's already enough to be genuinely useful. Everything below makes it better.

---

## 2. What to actually log

The rule: **log everything you'd need to reproduce the run or explain the result to someone three months from now.**

```python
import mlflow, torch, json, tempfile, matplotlib.pyplot as plt

mlflow.set_experiment("sales-forecasting")

with mlflow.start_run(run_name="lstm-h128-lr1e-3") as run:

    # ---- PARAMS: everything you chose ----
    mlflow.log_params({
        "model_type": "LSTM",
        "hidden_size": 128,
        "num_layers": 2,
        "dropout": 0.2,
        "learning_rate": 1e-3,
        "batch_size": 32,
        "epochs": 50,
        "optimizer": "Adam",
        "loss_fn": "MSELoss",
        "lookback_weeks": 8,
        # data provenance — the bit people forget
        "train_start": "2010-12-01",
        "train_end":   "2011-09-30",
        "delta_version": 12,               # ← see the note below
        "n_features": len(FEATURE_ORDER),
    })

    # ---- METRICS: per epoch, so you get curves ----
    for epoch in range(epochs):
        train_loss, val_loss = train_one_epoch(...)
        mlflow.log_metrics({"train_loss": train_loss, "val_loss": val_loss}, step=epoch)

    # ---- FINAL METRICS ----
    mlflow.log_metrics({
        "final_val_mae": val_mae,
        "final_val_rmse": val_rmse,
        "test_mae": test_mae,
        "test_mape": test_mape,
        "training_seconds": elapsed,
    })

    # ---- ARTIFACTS ----
    fig, ax = plt.subplots()
    ax.plot(history["train"], label="train"); ax.plot(history["val"], label="val"); ax.legend()
    mlflow.log_figure(fig, "loss_curve.png")

    with tempfile.NamedTemporaryFile("w", suffix=".json", delete=False) as f:
        json.dump(FEATURE_ORDER, f)                     # ← saves you from the feature-order bug
        mlflow.log_artifact(f.name, "features")

    mlflow.log_artifact("scaler.pkl")                   # ← serving needs this EXACT scaler

    # ---- THE MODEL ----
    mlflow.pytorch.log_model(
        model, "model",
        input_example=sample_input.numpy(),             # records the expected shape/dtype
        registered_model_name="sales-forecaster",
    )

    # ---- TAGS: for filtering later ----
    mlflow.set_tags({"stage": "experiment", "author": "huzayl", "dataset": "online-retail-ii"})

    print("run_id:", run.info.run_id)
```

### The three logs that matter more than they look

**`delta_version`** — the Delta table version you trained on ([[Databricks and Delta Lake]]). With it, you can reconstruct *exactly* the data this model saw, months later, with `VERSION AS OF`. Without it, "retrain on the same data" is guesswork. This one field is the difference between reproducible and not.

**`scaler.pkl`** — if you normalised features, serving must use the identical fitted scaler ([[Tensors, autograd and the training loop]]). Logging it as an artifact ties it to the run that made it, so they can't drift apart.

**`FEATURE_ORDER`** — the silent wrong-answer bug from [[FastAPI data and deployment]]. Log the order; load it at serving time.

> If you log nothing else beyond params and metrics, log these three.

### Autologging

```python
import mlflow

mlflow.pytorch.autolog()
```

Captures optimiser params, metrics, and the model automatically. Convenient — but **you still log data provenance, the scaler, and feature order yourself.** Autolog knows about your training loop; it doesn't know about your data.

---

## 3. Comparing runs

That's what the UI is for.

1. Open the experiment
2. Tick several runs → **Compare**
3. You get a table of params/metrics side by side, and overlaid metric curves

**The parallel coordinates plot** is the one worth knowing: params on vertical axes, lines coloured by metric. Patterns jump out — "every good run has `hidden_size >= 64`" — that you'd never see in a table.

### Querying runs in code

```python
import mlflow

runs = mlflow.search_runs(
    experiment_names=["sales-forecasting"],
    filter_string="metrics.test_mae < 500 and params.model_type = 'LSTM'",
    order_by=["metrics.test_mae ASC"],
    max_results=10,
)
print(runs[["run_id", "params.hidden_size", "params.learning_rate", "metrics.test_mae"]])

best_run_id = runs.iloc[0]["run_id"]
```

Returns a pandas DataFrame. Useful for automating "pick the best and register it" in a pipeline.

### A hyperparameter sweep

```python
import mlflow
import itertools

for lr, hidden, layers in itertools.product([1e-2, 1e-3, 1e-4], [32, 64, 128], [1, 2]):
    with mlflow.start_run(run_name=f"lr{lr}-h{hidden}-l{layers}"):
        mlflow.log_params({"learning_rate": lr, "hidden_size": hidden, "num_layers": layers})
        model, val_mae = train(lr=lr, hidden=hidden, layers=layers)
        mlflow.log_metric("val_mae", val_mae)
        mlflow.pytorch.log_model(model, "model")
```

18 runs, all logged, all comparable. This is the capstone's "3 different hyperparameter sets" task, done properly.

> **Choose your winner on validation, not test.** Pick the best run by `val_mae`, then report `test_mae` for that one run only. Choosing the best *test* score means you've tuned on the test set, and your reported number is optimistic — which is exactly what the test set exists to prevent.

---

## 4. The Model Registry

Experiments are messy — hundreds of runs, most of them bad. The **registry** is the curated shelf: named models, versions, and a stage for each.

```python
import mlflow

result = mlflow.register_model(f"runs:/{best_run_id}/model", "sales-forecaster")
print(result.version)      # 3
```

### Stages / aliases

Classic stages: `None` → `Staging` → `Production` → `Archived`.

Newer MLflow prefers **aliases**, which are more flexible:

```python
from mlflow import MlflowClient
client = MlflowClient()

client.set_registered_model_alias("sales-forecaster", "champion", version=3)
client.set_registered_model_alias("sales-forecaster", "challenger", version=4)
```

### Why this matters for serving

Your API loads **by name and alias**, not by file path:

```python
import mlflow

model = mlflow.pyfunc.load_model("models:/sales-forecaster@champion")
```

Promote version 4 to `champion` and the API picks it up on next restart. **No code change, no redeploy.** Rolling back is pointing the alias at the old version — seconds, not a deployment.

That decoupling is the entire point of the registry. Without it, "deploy a new model" means editing a file path in code and redeploying.

```python
# describe what changed — future you will want this
client.update_model_version(
    name="sales-forecaster", version=3,
    description="LSTM, 8-week lookback. MAE 412 vs 498 baseline. Trained on Delta v12.",
)
```

---

## 5. MLflow locally vs. on Databricks

| | Local | Databricks |
|---|---|---|
| Setup | `pip install mlflow`, `mlflow ui` | Already there |
| Where runs go | `./mlruns/` | The workspace |
| Registry | Needs a tracking server with a DB backend | Built in, Unity Catalog integrated |
| Sharing | Only you | Whole team, with permissions |

**Local:**

```python
import mlflow

mlflow.set_tracking_uri("file:./mlruns")     # the default
mlflow.set_experiment("sales-forecasting")
```

**Databricks notebook:** nothing to configure. Every run appears in the Experiments tab automatically, linked to the notebook.

**Local script logging to Databricks:**

```python
import mlflow

mlflow.set_tracking_uri("databricks")
mlflow.set_experiment("/Users/you@example.com/sales-forecasting")
```

(Requires `databricks configure` or `DATABRICKS_HOST`/`DATABRICKS_TOKEN` env vars.)

> **On Databricks, register models into Unity Catalog** — `mlflow.set_registry_uri("databricks-uc")` — so models get the same three-level naming and permissions as tables (`retail.models.sales_forecaster`). One governance model for data and models ([[Unity Catalog and orchestration]]).

---

## 6. Metrics that mean something

Logging the wrong metric well is still useless. For the capstone's two models:

### Forecasting (regression)

| Metric | Meaning | Use when |
|---|---|---|
| **MAE** | Average absolute error, in £ | **Default.** Interpretable: "off by £412 on average." |
| **RMSE** | Like MAE but punishes big misses harder | Large errors are disproportionately costly |
| **MAPE** | Average % error | Comparing across categories of different sizes. **Breaks if any actual is 0.** |
| R² | Fraction of variance explained | Quick sanity check; poor for time series |

> **Always log a baseline.** "MAE 412" means nothing alone. Compare against *predict last week's value* — the naive forecast. If your LSTM can't beat that, it isn't a model, it's a random number generator with extra steps. Log `baseline_mae` in every run.

### Segmentation (classification)

| Metric | Meaning |
|---|---|
| Accuracy | % correct. **Misleading with imbalanced classes.** |
| Precision | Of those I flagged, how many were right |
| Recall | Of the real ones, how many did I catch |
| **F1** | Harmonic mean of the two. **The default for imbalanced data.** |
| Confusion matrix | Which classes get confused with which. Log it as an artifact. |

> If 95% of customers are "regular", a model predicting "regular" always scores **95% accuracy** and is completely worthless. Log F1 and the confusion matrix, and look at them.

```python
import matplotlib.pyplot as plt
import mlflow
from sklearn.metrics import confusion_matrix, ConfusionMatrixDisplay

cm = confusion_matrix(y_true, y_pred)
fig, ax = plt.subplots()
ConfusionMatrixDisplay(cm, display_labels=LABELS).plot(ax=ax)
mlflow.log_figure(fig, "confusion_matrix.png")
```

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| Runs don't appear in the UI | Wrong tracking URI, or `mlflow ui` started elsewhere | Run `mlflow ui` from the folder containing `mlruns/` |
| "Run already active" | A previous `start_run` never closed | Use `with mlflow.start_run():`; or `mlflow.end_run()` |
| Metric shows one point, no curve | Logged without `step=` | `mlflow.log_metrics({...}, step=epoch)` |
| Can't reproduce an old result | Data provenance not logged | Log `delta_version` / date range in every run |
| Loaded model gives wrong predictions | Scaler or feature order not logged with it | Log both as artifacts; load them at serving |
| `log_model` fails on a custom class | Class not importable at load time | Use `code_paths=`, or export to ONNX ([[Model export and serving]]) |
| Registry empty in Databricks | Registering to the workspace, not UC | `mlflow.set_registry_uri("databricks-uc")` |
| `mlruns/` is enormous | Every run stores a full model copy | Only `log_model` for runs worth keeping; add `mlruns/` to `.gitignore` |
| Best model by test score disappoints in production | Selected on the test set | Select on validation, report on test |
| 95% accuracy, useless model | Class imbalance | Log F1 + confusion matrix |
| Two runs, same params, different results | No seed logged or set | `torch.manual_seed(42)` and log the seed |

---

## Practice checklist

- [ ] Runs, parameters, metrics, and artifacts — what to log and why
- [ ] Params vs. metrics — the "set" vs. "measured" rule
- [ ] Logging metrics with `step=` so you get curves
- [ ] **Logging data provenance** (Delta version / date range) — the reproducibility key
- [ ] **Logging the scaler and feature order** alongside the model
- [ ] Autologging, and what it doesn't cover
- [ ] Comparing runs in the UI; parallel coordinates
- [ ] `mlflow.search_runs` for programmatic selection
- [ ] Choosing a winner on validation, reporting on test
- [ ] The Model Registry: versioning, stages/aliases, and **loading by alias so promotion needs no redeploy**
- [ ] Logging from a Databricks notebook vs. from a local script
- [ ] Picking metrics that mean something — MAE vs MAPE, F1 vs accuracy, and always logging a baseline

## Hands-on

- [ ] Instrument your training loop from [[CNNs and transfer learning]] with MLflow logging
- [ ] Run the same training with 3 different hyperparameter sets and compare them in the MLflow UI
- [ ] Log a loss-curve figure and a confusion matrix as artifacts
- [ ] Log a naive baseline metric and confirm your model actually beats it
- [ ] Register the best model in the Model Registry and set a `champion` alias
- [ ] Load a model by alias (`models:/name@champion`) instead of by file path
- [ ] Promote a different version to `champion` and confirm the loading code needed no change

## Certification checkpoint

Databricks Certified Machine Learning Associate assumes this plus Spark ML familiarity. See [[Certification map]].

## Resources

- [MLflow docs](https://mlflow.org/docs/latest/index.html)
- [MLflow Model Registry](https://mlflow.org/docs/latest/model-registry.html)
- [Databricks: MLflow guide](https://docs.databricks.com/mlflow/index.html)

## Next

[[Model export and serving]]
