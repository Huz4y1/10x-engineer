---
tags: [dictionary, tensorflow, keras, deep-learning]
status: not-started
---

# TensorFlow and Keras

Template: [[_Dictionary template]] · Section: [[09 — DEEP LEARNING]]

---

## One sentence

TensorFlow is Google's deep learning framework; Keras is the high-level API most people actually write.

## In simple words

If [[Tensors, autograd and the training loop|PyTorch]] is a manual car — you write the training loop yourself and see every gear change — **Keras is an automatic**. You describe the model, call `.fit()`, and it drives.

Both get you there. One teaches you what's happening; the other gets you there in ten lines.

## The problem it solves

Before these frameworks, a neural network meant hand-writing the derivative of every layer. Get one wrong and the model silently learns nothing.

TensorFlow (2015) provided automatic differentiation, GPU execution, and — critically — a **deployment story**: TF Serving, TF Lite for phones, TF.js for browsers. That production tooling is why it dominated industry for years.

## PyTorch or TensorFlow?

**Honest answer for you: stay on PyTorch.** Learn enough TensorFlow to read it.

| | PyTorch | TensorFlow / Keras |
|---|---|---|
| Research papers | **~90%+** | Rare now |
| Feels like | Normal Python | A framework you configure |
| Debugging | Standard Python debugger | Harder inside `@tf.function` |
| Training loop | You write it | `.fit()` writes it |
| Mobile / embedded | ExecuTorch (newer) | **TF Lite (mature)** |
| Browser | ONNX.js | **TF.js (mature)** |
| Job adverts | Increasingly dominant | Legacy systems, some big companies |

> **Where TensorFlow still genuinely wins: deployment to phones, microcontrollers and browsers.** TF Lite Micro runs on an ESP32 ([[20 — EMBEDDED]]) — that's directly relevant to [[Project 008 — Physical Intelligent Engine Monitor]].

> **Keras 3 is multi-backend.** It now runs on TensorFlow, JAX **or** PyTorch. So Keras is no longer a reason to choose TensorFlow — you can write Keras and run it on a PyTorch backend.

## How it works

**Eager execution (default in TF2):** operations run immediately, like NumPy and PyTorch.

**Graph mode (`@tf.function`):** TensorFlow traces your Python function once, builds a static computation graph, and runs that. Faster, and deployable without Python.

```python
import tensorflow as tf

@tf.function          # traced ONCE, then the graph is reused
def train_step(x, y):
    with tf.GradientTape() as tape:
        loss = loss_fn(model(x, training=True), y)
    grads = tape.gradient(loss, model.trainable_variables)
    optimizer.apply_gradients(zip(grads, model.trainable_variables))
    return loss
```

> **The `@tf.function` trap:** the Python body runs only during tracing. `print()` fires once and never again; Python `if` on a tensor bakes in one branch. This is the same limitation as ONNX tracing in [[Model export and serving]] — and it confuses everyone once.

## Important vocabulary

| Term | Meaning |
|---|---|
| **Tensor** | N-dimensional array (same idea as PyTorch) |
| **Variable** | A tensor that can be updated — a weight |
| **GradientTape** | Records operations so gradients can be computed |
| **`@tf.function`** | Compile a Python function into a graph |
| **Keras** | The high-level model API |
| **SavedModel** | TF's deployment format (a directory, not a file) |
| **TF Lite** | Slimmed runtime for mobile/embedded |
| **TF Serving** | Production model server |

## Code

### Level 1 — the ten-line version

```python
import tensorflow as tf
from tensorflow import keras

model = keras.Sequential([
    keras.layers.Dense(64, activation="relu", input_shape=(10,)),
    keras.layers.Dropout(0.2),
    keras.layers.Dense(1),
])
model.compile(optimizer="adam", loss="mse", metrics=["mae"])
model.fit(X_train, y_train, validation_data=(X_val, y_val), epochs=50, batch_size=32)
model.evaluate(X_test, y_test)
```

> That's the whole of [[Tensors, autograd and the training loop]] in six lines. **Which is exactly why you should write the loop by hand first** — otherwise `.fit()` is magic you can't debug.

### Level 2 — practical, with callbacks

```python
from tensorflow import keras

callbacks = [
    keras.callbacks.EarlyStopping(patience=10, restore_best_weights=True),  # <- restore!
    keras.callbacks.ReduceLROnPlateau(factor=0.5, patience=5),
    keras.callbacks.ModelCheckpoint("best.keras", save_best_only=True),
    keras.callbacks.TensorBoard(log_dir="./logs"),
]
history = model.fit(train_ds, validation_data=val_ds, epochs=100, callbacks=callbacks)
```

> `restore_best_weights=True` is the equivalent of reloading your best checkpoint in the PyTorch loop. Without it you keep the *last* (overfitted) model.

### Level 3 — custom training loop

When `.fit()` isn't enough, TensorFlow looks much more like PyTorch:

```python
from tensorflow import keras
import tensorflow as tf

optimizer = keras.optimizers.Adam(1e-3)
loss_fn = keras.losses.MeanSquaredError()

for epoch in range(epochs):
    for x, y in train_ds:
        with tf.GradientTape() as tape:                 # ~ requires_grad
            preds = model(x, training=True)
            loss = loss_fn(y, preds)
        grads = tape.gradient(loss, model.trainable_variables)   # ~ loss.backward()
        optimizer.apply_gradients(zip(grads, model.trainable_variables))  # ~ step()
```

> **No `zero_grad()`.** `GradientTape` is scoped — it records only inside the `with` block, then is discarded. That whole class of bug simply doesn't exist here.

### Level 4 — `tf.data` pipelines

```python
import tensorflow as tf

ds = (tf.data.Dataset.from_tensor_slices((X, y))
      .shuffle(10_000)
      .batch(32)
      .prefetch(tf.data.AUTOTUNE))     # <- overlap data prep with training
```

`prefetch(AUTOTUNE)` is `tf.data`'s answer to `num_workers` — it stops the GPU starving ([[CUDA and GPU programming]]).

## PyTorch → TensorFlow translation

| PyTorch | TensorFlow / Keras |
|---|---|
| `nn.Module` | `keras.Model` / `keras.layers.Layer` |
| `nn.Linear(i, o)` | `keras.layers.Dense(o)` *(input inferred)* |
| `nn.Conv2d(i, o, k)` | `keras.layers.Conv2D(o, k)` |
| `forward()` | `call()` |
| `loss.backward()` | `tape.gradient(loss, vars)` |
| `optimizer.step()` | `optimizer.apply_gradients(...)` |
| `optimizer.zero_grad()` | *(not needed)* |
| `model.train()` / `.eval()` | `training=True` / `False` |
| `DataLoader` | `tf.data.Dataset` |
| `.to(device)` | Automatic |
| `state_dict()` | `model.save_weights()` |
| **`(B, C, H, W)`** | **`(B, H, W, C)`** ← channels **last** |

> **The channels-last difference bites everyone.** PyTorch images are `(batch, channels, height, width)`; TensorFlow is `(batch, height, width, channels)`. Converting a model between them without permuting is a silent, confusing failure.

## Deployment — where TF earns its place

```python
import tensorflow as tf

model.save("model.keras")                # Keras format
model.export("saved_model/")             # SavedModel — for TF Serving

# TF Lite for mobile / embedded
converter = tf.lite.TFLiteConverter.from_saved_model("saved_model/")
converter.optimizations = [tf.lite.Optimize.DEFAULT]     # quantise to int8
open("model.tflite", "wb").write(converter.convert())
```

**TF Lite Micro** runs a quantised model on a microcontroller with no OS — kilobytes of RAM. That's the path to running your engine model *on the ESP32* rather than in the cloud ([[Project 008 — Physical Intelligent Engine Monitor]]).

## When to use it

- Deploying to phones, browsers or microcontrollers
- An existing TensorFlow codebase
- You want `.fit()` and don't need a custom loop
- TPU training (TF and JAX have the best support)

## When NOT to use it

| Situation | Use instead |
|---|---|
| Following research papers | **PyTorch** — that's where the code is |
| Learning how training works | PyTorch — write the loop by hand |
| Serving on a server | ONNX Runtime ([[Model export and serving]]) — smaller and faster |
| You want maximum speed on TPUs with custom maths | [[JAX]] |

## Common mistakes

| Mistake | What happens | Fix |
|---|---|---|
| `print()` inside `@tf.function` | Fires once, during tracing | `tf.print()` |
| Python `if` on a tensor in `@tf.function` | One branch baked in | `tf.cond` |
| Channels-first input | Silent shape mismatch | `(B, H, W, C)` |
| Forgetting `training=True/False` | Dropout/BatchNorm behave wrongly | Pass it explicitly |
| No `prefetch` | GPU starved by data loading | `.prefetch(AUTOTUNE)` |
| `EarlyStopping` without `restore_best_weights` | You keep the overfitted model | Set it `True` |
| Mixing Keras 2 and 3 APIs | Import errors | Check `keras.__version__` |

## Debugging

1. **Run eagerly first** — comment out `@tf.function`. If the bug disappears, it's a tracing problem.
2. `tf.config.run_functions_eagerly(True)` — global switch, no code changes.
3. `model.summary()` — layer shapes and parameter counts.
4. `tf.debugging.assert_shapes([...])` — catch shape errors at the right line.
5. **TensorBoard** — loss curves, graph, profiling.

## Performance

| Technique | Effect |
|---|---|
| `@tf.function` | Graph compilation — significant |
| Mixed precision | ~2× on modern GPUs |
| `tf.data` + `prefetch` + `cache` | Stops GPU starvation |
| XLA (`jit_compile=True`) | Fuses ops — big on TPU |

```python
from tensorflow import keras

keras.mixed_precision.set_global_policy("mixed_float16")
model.compile(..., jit_compile=True)
```

## Cloud equivalents

| Local | Azure | AWS | GCP |
|---|---|---|---|
| TF + TF Serving | Azure ML | SageMaker | **Vertex AI** (TF is first-class) |

## Prerequisites

[[09 — DEEP LEARNING]] · [[Tensors, autograd and the training loop]] · [[Mathematics reference]]

## Learning progression

- **Beginner:** `Sequential`, `.compile()`, `.fit()`
- **Intermediate:** functional API, callbacks, `tf.data`, custom loops with `GradientTape`
- **Advanced:** `@tf.function` semantics, custom layers/losses, distribution strategies, TF Lite quantisation
- **Research:** XLA compilation, TPU pods

## Related

[[09 — DEEP LEARNING]] · [[10 — PYTORCH]] · [[JAX]] · [[Model export and serving]] · [[20 — EMBEDDED]] · [[CUDA and GPU programming]] · [[Large language models]]
