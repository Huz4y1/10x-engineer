---
tags: [pytorch, onnx, torchscript, model-serving]
status: not-started
---

# Model Export & Serving

> **What this is:** getting a trained model out of a notebook and into a process that has never seen your training code.
> **Why you care:** this is where models break. The training was the easy part; the bugs here are silent, and they produce confidently wrong answers rather than errors.

---

## The problem

Your model works in the notebook. Now serve it from a Docker container running FastAPI.

```python
import torch

model = torch.load("model.pth")     # ✗ ModuleNotFoundError: No module named 'train'
```

A `.pth` file saved with `torch.save(model)` is a **pickle**. Pickles store a *reference* to the class, not the class itself. Loading requires:

- The exact class definition importable at the same path
- The same PyTorch version (roughly)
- The same Python version (roughly)
- The whole `torch` package installed — ~2GB in your container, to run a 5MB model

Even `state_dict` (the right way to save, per [[Tensors, autograd and the training loop]]) needs the class definition. That couples your serving container to your training code forever. Refactor the training repo and production breaks.

**The fix: export to a self-contained format that describes the computation itself.**

---

## The two formats

| | **TorchScript** | **ONNX** |
|---|---|---|
| What it is | PyTorch's own serialised format | An open standard, many frameworks and runtimes |
| Needs at load time | `libtorch` / PyTorch | `onnxruntime` (~50MB) |
| Runs in | Python, C++ | Python, C++, C#, Java, JS, mobile, browser |
| Handles Python control flow | Yes (with `script`) | Only what's traced |
| Speed | Fast | Often faster on CPU (graph optimisations) |
| Ecosystem | PyTorch only | Everything |

### Which to pick

**ONNX**, for this project and for most serving.

- Your container needs `onnxruntime`, not the full PyTorch stack. Image goes from ~2.5GB to ~300MB. On Container Apps that means faster cold starts and lower cost.
- Genuinely faster CPU inference — ONNX Runtime fuses operations and optimises the graph.
- Framework-independent, which is the honest reason it exists.

**TorchScript** when: you're staying inside a PyTorch shop, you need Python control flow that tracing can't capture, or you're using PyTorch's own serving stack.

> Sensible default: **export to ONNX, keep the `state_dict` as the source of truth.** ONNX is the deployment artifact; the state dict is what you'd retrain or re-export from.

---

## 1. Exporting to ONNX

```python
import torch

model.eval()                                   # ← MANDATORY. See below.

dummy_input = torch.randn(1, n_features, dtype=torch.float32)

torch.onnx.export(
    model,
    dummy_input,
    "models/forecast.onnx",
    input_names=["features"],
    output_names=["prediction"],
    dynamic_axes={                             # allow variable batch size
        "features":   {0: "batch_size"},
        "prediction": {0: "batch_size"},
    },
    opset_version=17,
    do_constant_folding=True,
)
```

### The four things that matter

**`model.eval()` first.** In training mode, dropout is randomly zeroing activations and batchnorm is using batch statistics. Export in training mode and you bake that randomness into the graph permanently. Your served model then gives **different answers for identical inputs**, forever, with no error. This is the worst bug in this note — silent, intermittent, and baffling.

**`dynamic_axes`.** Without it, the exported model accepts *exactly* the batch size of your dummy input. Export with batch 1, and a request for 32 predictions fails. Mark the batch dimension dynamic.

**`input_names` / `output_names`.** Otherwise you get `input.1` and `1832`, and your serving code becomes unreadable.

**`opset_version`.** The ONNX operator set version. Higher supports more operations; too high may outrun your runtime. 17 is a safe modern choice.

### Tracing, and its limitation

`torch.onnx.export` **traces**: it runs your model once with the dummy input and records the operations that happened.

That means **data-dependent branches are baked in**:

```python
def forward(self, x):
    if x.sum() > 0:          # ✗ only ONE branch gets traced
        return self.a(x)
    return self.b(x)
```

Whichever branch the dummy input took is the only one in the exported model. No warning worth noticing.

Also: Python loops are unrolled, `.item()` calls freeze into constants, and `print` statements vanish.

> **Rule of thumb:** keep `forward()` free of data-dependent Python control flow. If you can't, use TorchScript's `torch.jit.script` (which compiles the actual code, branches included) instead of tracing.

### Verification — non-optional

**Never trust an export you haven't verified.**

```python
import torch
import numpy as np, onnx, onnxruntime as ort

onnx_model = onnx.load("models/forecast.onnx")
onnx.checker.check_model(onnx_model)                     # structurally valid?

session = ort.InferenceSession("models/forecast.onnx", providers=["CPUExecutionProvider"])

test_input = torch.randn(8, n_features, dtype=torch.float32)      # batch of 8, not 1

model.eval()
with torch.no_grad():
    torch_out = model(test_input).numpy()

onnx_out = session.run(None, {"features": test_input.numpy()})[0]

np.testing.assert_allclose(torch_out, onnx_out, rtol=1e-4, atol=1e-5)
print("✓ ONNX matches PyTorch — max diff:", np.abs(torch_out - onnx_out).max())
```

Points worth copying:

- **Test with a batch size different from the dummy** — that's what catches missing `dynamic_axes`.
- **`rtol=1e-4`, not exact equality.** Different operation ordering produces tiny floating-point differences. That's expected. A difference of `1e-2` is not.
- **Make this a real test** in your suite ([[Testing and CI-CD]]). It runs in a second and catches a whole category of production bugs.

---

## 2. TorchScript, briefly

```python
import torch

model.eval()

# tracing — same limitation as ONNX
traced = torch.jit.trace(model, dummy_input)
traced.save("models/forecast_traced.pt")

# scripting — compiles the actual code, control flow included
scripted = torch.jit.script(model)
scripted.save("models/forecast_scripted.pt")

# loading — no class definition needed
loaded = torch.jit.load("models/forecast_scripted.pt")
loaded.eval()
with torch.no_grad():
    out = loaded(x)
```

`script` handles branches and loops but is fussier — it type-checks your Python and rejects a lot of ordinary code. `trace` accepts anything and silently loses control flow. Try `trace` first; fall back to `script` when you need real branching.

---

## 3. The contracts — where the real bugs live

Everything so far was mechanics. This section is where models silently produce wrong answers.

### The four things that must match training exactly

**1. Feature order**

```python
FEATURE_ORDER = ["recency_days", "frequency", "monetary_value", "avg_basket", "tenure_days"]
```

Train on that order, serve `[frequency, recency_days, ...]`, and **nothing errors.** Shapes match. The model confidently returns nonsense forever.

Save the order with the model and build inputs from it — never by hand:

```python
import numpy as np

features = np.array([[payload[name] for name in FEATURE_ORDER]], dtype=np.float32)
```

**2. Scaling**

If you scaled during training, you must apply the **same fitted scaler**:

```python
import joblib
import numpy as np

scaler = joblib.load("models/scaler.pkl")      # the one from training, not a new one
features = scaler.transform(raw_features).astype(np.float32)
```

Refitting a scaler on live data is a classic silent failure: the transform changes as traffic changes, and the model sees a moving target.

**3. dtype**

PyTorch is float32. NumPy defaults to float64.

```python
import numpy as np

np.array([[1, 2, 3]])                        # int64  → error
np.array([[1.0, 2.0, 3.0]])                  # float64 → error
np.array([[1.0, 2.0, 3.0]], dtype=np.float32)  # ✓
```

ONNX Runtime is strict and *will* error here — which is a mercy. Transformer inputs need `int64` ([[Transformers and LLM basics]]).

**4. Shape**

The model expects a batch. One sample is `(1, n_features)`, not `(n_features,)`.

```python
import numpy as np

np.array([1.0, 2.0, 3.0], dtype=np.float32)          # (3,)   ✗
np.array([[1.0, 2.0, 3.0]], dtype=np.float32)        # (1, 3) ✓
```

### Ship the contract with the model

Don't leave these in comments. Export them as a file:

```python
import json

metadata = {
    "model_name": "sales-forecaster",
    "version": "v3",
    "mlflow_run_id": run_id,
    "feature_order": FEATURE_ORDER,
    "input_shape": [None, len(FEATURE_ORDER)],
    "input_dtype": "float32",
    "output_names": ["predicted_revenue"],
    "scaler_path": "scaler.pkl",
    "trained_on_delta_version": 12,
    "training_date": "2026-09-06",
}
with open("models/forecast_metadata.json", "w") as f:
    json.dump(metadata, f, indent=2)
```

Load it at startup and build inputs from it. Now the contract is code, not folklore, and a mismatch fails loudly instead of quietly.

---

## 4. Inference

### The basics

```python
import onnxruntime as ort
import numpy as np

session = ort.InferenceSession("models/forecast.onnx", providers=["CPUExecutionProvider"])

print([(i.name, i.shape, i.type) for i in session.get_inputs()])
print([(o.name, o.shape, o.type) for o in session.get_outputs()])

outputs = session.run(None, {"features": features})    # None = "all outputs"
prediction = outputs[0]
```

> **Create the `InferenceSession` once, at startup.** It reads the file, builds the optimised graph, and allocates memory — hundreds of milliseconds. Per request, that dominates your latency. `lifespan` in [[FastAPI data and deployment]].

**Thread settings matter in a container:**

```python
import onnxruntime as ort

opts = ort.SessionOptions()
opts.intra_op_num_threads = 2          # match your container's CPU allocation
opts.graph_optimization_level = ort.GraphOptimizationLevel.ORT_ENABLE_ALL

session = ort.InferenceSession("models/forecast.onnx", opts, providers=["CPUExecutionProvider"])
```

ONNX Runtime defaults to using every core it can see — which in a container is the *host's* core count, not your limit. It then oversubscribes and gets *slower*. Set it explicitly.

### Batching

One request at a time wastes most of the compute — matrix operations are far more efficient on many rows at once.

```python
import numpy as np

# 32 separate calls
for row in rows:
    session.run(None, {"features": row.reshape(1, -1)})     # ~32 × overhead

# one call
session.run(None, {"features": np.vstack(rows)})            # ~1 × overhead
```

Same total maths, a fraction of the overhead. Your `/predict` endpoint should accept a **list** of items and run one inference:

```python
from fastapi.concurrency import run_in_threadpool
from pydantic import BaseModel, Field
import numpy as np

class BatchRequest(BaseModel):
    customers: list[CustomerFeatures] = Field(..., max_length=1000)   # cap it

@router.post("/predict/segments")
async def predict_batch(req: BatchRequest):
    features = np.array(
        [[getattr(c, f) for f in FEATURE_ORDER] for c in req.customers],
        dtype=np.float32,
    )
    probs = (await run_in_threadpool(session.run, None, {"features": features}))[0]
    return [
        {"customer_id": c.customer_id,
         "segment": LABELS[int(p.argmax())],
         "confidence": float(p.max())}
        for c, p in zip(req.customers, probs)
    ]
```

> **Cap the batch size** with `max_length`. Without it, someone posts 10 million rows and your container runs out of memory. That's a denial-of-service hole, and it's one line to close.

### Latency

| Stage | Typical |
|---|---|
| Cold container start | 2–10s |
| Loading the ONNX session | 50–500ms |
| Warm inference (small tabular model) | 0.1–5ms |
| Warm inference (DistilBERT, 128 tokens) | 10–50ms |
| Database round trip | 5–50ms |

**Inference is rarely your bottleneck.** For a small tabular model, the database query and network are 10–100× the model time. Measure before optimising — people spend days quantizing a model that accounts for 2% of request latency.

Measure with the middleware from [[FastAPI data and deployment]], and time the inference separately:

```python
import time

t0 = time.perf_counter()
out = session.run(None, inputs)
logger.info("inference", extra={"extra_fields": {"ms": (time.perf_counter()-t0)*1000}})
```

---

## 5. Versioning models in production

```
models/
├── forecast_v3.onnx
├── forecast_v3_metadata.json
├── scaler_v3.pkl
└── segment_v2.onnx
```

Three rules:

1. **Version in the filename**, never `model.onnx`.
2. **Return the version in every response** — `model_version: "v3"`. When a prediction is questioned weeks later, you can say which model made it.
3. **Log the version with every prediction.** Combined with MLflow's run ID, that traces a live prediction all the way back to the training run and the data version.

The cleanest version is loading from the MLflow registry by alias ([[MLflow experiment tracking]]):

```python
import mlflow

model = mlflow.pyfunc.load_model("models:/sales-forecaster@champion")
```

Promotion becomes an alias change instead of a redeploy.

### Where the file lives

| Approach | Pros | Cons |
|---|---|---|
| **In the Docker image** | Simple; code and model versions can't drift | Big image; rebuild per model update |
| Downloaded at startup from blob storage | Small image; update model without rebuilding | Startup dependency; must handle failure |
| From MLflow registry at startup | Versioned, promotable, auditable | Needs registry access from the container |

**For the capstone: in the image.** Simplest, and guarantees the code and model match.

---

## 6. What happens after deployment

Not in the capstone scope, but you will be asked about it.

**Data drift** — the input distribution shifts. Post-pandemic buying patterns don't match 2011. Detect by logging input feature distributions and comparing to training.

**Concept drift** — the relationship between inputs and outputs changes. Same features, different meaning.

**Monitoring you'd actually add:**
- Prediction distribution over time (a sudden shift means something broke upstream)
- Input feature statistics vs. training statistics
- Actuals vs. predictions, once ground truth arrives
- Inference latency and error rate

> **The honest summary:** models degrade. A model deployed and never re-evaluated is a liability, not an asset. The retraining pipeline matters more than the model.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| `ModuleNotFoundError` loading a `.pth` | Pickle needs the class definition | Export to ONNX; save `state_dict` not the model |
| Same input gives different answers | Exported without `model.eval()` — dropout baked in | Re-export in eval mode |
| Works with batch 1, fails with batch 32 | No `dynamic_axes` | Re-export with the batch dim marked dynamic |
| `Unexpected input data type` | float64 vs float32 (or int32 vs int64) | `.astype(np.float32)` |
| `Invalid rank for input` | Missing batch dimension | `reshape(1, -1)` |
| ONNX and PyTorch outputs differ slightly (~1e-6) | Floating-point op ordering | Normal. Use `rtol=1e-4`. |
| ONNX and PyTorch differ a lot (>1e-2) | Real export bug — eval mode, or an unsupported op | Check `model.eval()`; try a different opset |
| Predictions plausible but wrong | Feature order or scaling mismatch | Load `feature_order` and the saved scaler from metadata |
| First request very slow, rest fast | Session created per request, or cold start | Load in `lifespan` |
| Slower in the container than locally | ONNX Runtime oversubscribing threads | Set `intra_op_num_threads` to your CPU limit |
| Container OOM under load | Unbounded batch size, or `--workers N` × model size | Cap batch length; fewer workers |
| Model file missing at startup | Not copied into the image | `COPY models/ ./models/`; check `.dockerignore` |
| Only one branch of an `if` ever runs | Tracing baked in one path | Remove data-dependent control flow, or use `torch.jit.script` |
| Can't tell which model made a prediction | No version logged | Return and log `model_version` |

---

## Practice checklist

- [ ] Why a pickled `.pth` is a bad deployment artifact
- [ ] TorchScript vs. ONNX — what each is for, and when to use which
- [ ] Exporting with `torch.onnx.export`: `model.eval()`, `dynamic_axes`, names, opset
- [ ] **Why `model.eval()` before export is mandatory**
- [ ] Tracing vs. scripting, and what tracing silently loses
- [ ] **Verifying the export numerically** — with a different batch size
- [ ] Input/output **shape, dtype, feature order and scaling contracts** — the most common source of production bugs
- [ ] Shipping a metadata file so the contract is code, not folklore
- [ ] Creating the `InferenceSession` once at startup
- [ ] Thread settings in a container
- [ ] Batching predictions vs. serving one request at a time — and capping batch size
- [ ] Basic latency awareness: cold load vs. warm, and where the time actually goes
- [ ] Versioning: filenames, response fields, registry aliases
- [ ] Data drift and why deployment isn't the end

## Hands-on

- [ ] Export a model trained earlier in this section to ONNX
- [ ] Load it in a clean script with `onnxruntime` and confirm predictions match PyTorch — using a **different batch size**
- [ ] Export **without** `model.eval()` and observe the outputs changing between runs
- [ ] Export **without** `dynamic_axes` and watch a batch of 32 fail
- [ ] Write the numerical-equivalence check as a real `pytest` test
- [ ] Shuffle `FEATURE_ORDER` at serving time and confirm nothing errors — the lesson is the silence
- [ ] Time single-row vs. batched inference for 1,000 rows
- [ ] Wire it into the FastAPI `/predict` endpoint from [[FastAPI data and deployment]]

## Resources

- [TorchScript docs](https://pytorch.org/docs/stable/jit.html)
- [PyTorch: exporting to ONNX](https://pytorch.org/tutorials/beginner/onnx/export_simple_model_to_onnx_tutorial.html)
- [ONNX Runtime docs](https://onnxruntime.ai/docs/)
- [Netron](https://netron.app/) — drag an `.onnx` file in and see the graph. Excellent for debugging exports.

## Next

[[Testing and CI-CD]]
