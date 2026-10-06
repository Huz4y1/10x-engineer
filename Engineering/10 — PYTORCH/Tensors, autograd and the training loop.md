---
tags: [pytorch, deep-learning, fundamentals]
status: not-started
---

# Tensors, Autograd & the Training Loop

> **What this is:** the mechanics everything else in PyTorch builds on.
> **Why you care:** every high-level trainer is a wrapper around the six lines in section 4. Write them by hand once and nothing later is mysterious.

---

## The idea in plain English

A neural network is a machine with millions of adjustable dials.

You feed it an input. It produces an output. You compare that output to the right answer and get a number for how wrong it was — the **loss**.

Then the clever bit: you work out, **for every single dial**, "if I turned this one slightly, would the wrongness go up or down, and by how much?" That set of answers is the **gradient**.

Then you turn every dial a tiny amount in the direction that reduces wrongness. Repeat a few thousand times.

That's it. That's deep learning. The rest is engineering.

The magic that makes it practical is **autograd** — PyTorch computes those millions of "which way should this dial turn" answers automatically, no calculus from you.

---

## 1. Tensors

A tensor is an array with extra powers: it can live on a GPU, and it can remember how it was created so gradients can flow back through it.

```python
import torch

torch.tensor([1.0, 2.0, 3.0])          # from a list
torch.zeros(3, 4)                       # 3 rows, 4 cols of 0
torch.ones(2, 3)
torch.randn(2, 3)                       # normal distribution — the usual for random init
torch.arange(0, 10, 2)                  # [0, 2, 4, 6, 8]
torch.from_numpy(np_array)              # from NumPy (SHARES memory — mutating one changes both)
```

### Shape is the thing you'll actually spend time on

```python
import torch

x = torch.randn(32, 10)
x.shape          # torch.Size([32, 10])  — 32 rows ("batch"), 10 features each
x.dtype          # torch.float32         — the default
x.device         # cpu
```

> **Read shapes as "batch first."** `(32, 10)` = 32 samples, 10 features. `(32, 3, 224, 224)` = 32 images, 3 colour channels, 224×224 pixels. Nearly every PyTorch shape error is a mismatch you can find by printing `.shape` at each step.

**Reshaping:**

```python
import torch

x = torch.randn(32, 10)
x.view(320)                # flatten (needs contiguous memory)
x.reshape(320)             # like view, but copies if it has to. Safer default.
x.T                        # transpose → (10, 32)
x.unsqueeze(0).shape       # (1, 32, 10) — add a dimension
x.squeeze().shape          # remove all size-1 dimensions
x.flatten(start_dim=1)     # keep batch dim, flatten the rest — the CNN→Linear move
```

`-1` means "work it out":

```python
x.reshape(-1, 5).shape     # (64, 5) — PyTorch computes 64 from 320/5
```

### Broadcasting

Operations on different-shaped tensors auto-expand where possible:

```python
import torch

x = torch.randn(32, 10)
b = torch.randn(10)
(x + b).shape              # (32, 10) — b added to every row
```

**The rule:** align shapes from the right. Dimensions must be equal, or one of them must be 1.

```
(32, 10)  +  (10,)   →  ✓   (10 vs 10 ✓, then 32 vs nothing → keep)
(32, 10)  +  (32,)   →  ✗   (10 vs 32 ✗)
(32, 10)  +  (32, 1) →  ✓
```

> **Broadcasting causes silent bugs.** If your predictions are `(32, 1)` and targets are `(32,)`, `pred - target` broadcasts into `(32, 32)` — a matrix of every prediction minus every target. The loss computes fine and is complete nonsense. **When a loss is weirdly large or won't go down, print `pred.shape` and `target.shape` first.**

### Devices

```python
import torch

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

model = model.to(device)
x = x.to(device)
```

> **Everything in one operation must be on the same device**, or you get `Expected all tensors to be on the same device`. Move the model once at the start; move each batch inside the loop.

`.to(device)` returns a *new* tensor for data (`x = x.to(device)` — reassign), but moves a model **in place** (`model.to(device)` is enough, though reassigning is harmless).

---

## 2. Autograd

### The mechanism

```python
import torch

x = torch.tensor([2.0], requires_grad=True)
y = x ** 2 + 3 * x
y.backward()
print(x.grad)        # tensor([7.])   -- dy/dx = 2x + 3 = 7 at x=2
```

`requires_grad=True` tells PyTorch to record every operation involving `x` into a **computational graph** — a record of what happened, so it can be replayed backwards.

`.backward()` walks that graph in reverse applying the chain rule, and deposits the result in `.grad` on every leaf tensor.

Model parameters have `requires_grad=True` automatically. So `loss.backward()` fills in `.grad` for every weight in the network, however deep. That's the whole trick.

### The three rules

**1. Gradients accumulate. You must zero them.**

```python
for epoch in range(3):
    loss = compute_loss()
    loss.backward()          # ADDS to .grad, doesn't replace it
```

After three epochs `.grad` holds the sum of three gradients, and your updates are nonsense.

```python
optimizer.zero_grad()        # ← at the top of every batch. Non-negotiable.
```

> **Forgetting `zero_grad()` is the most common PyTorch bug.** It doesn't crash. Training just gets worse and worse in a confusing way. (The accumulate-by-default behaviour is deliberate — it lets you simulate a large batch on a small GPU by accumulating over several small batches.)

**2. The graph is freed after `backward()`.** Calling `.backward()` twice on the same loss errors. That's fine — you compute a fresh loss each batch.

**3. Turn autograd off when you're not training.**

```python
import torch

with torch.no_grad():
    predictions = model(x_test)
```

Building the graph costs memory and time. During evaluation or inference you don't need gradients, so switch it off. Forget this and evaluation is slower and can run out of memory.

### `.detach()` and `.item()`

```python
total_loss += loss                  # ✗ keeps the whole graph alive → memory leak
total_loss += loss.item()           # ✓ plain Python float
predictions = model(x).detach()     # ✓ a tensor with no graph attached
```

> **The classic memory leak:** accumulating loss tensors across a loop keeps every batch's computational graph in memory. By epoch three you're out of RAM. **Always `.item()` when logging a scalar.**

---

## 3. Building a model

### `nn.Module`

```python
import torch.nn as nn

class SalesForecastNet(nn.Module):
    def __init__(self, n_features: int, hidden: int = 64):
        super().__init__()                                # ← always first
        self.net = nn.Sequential(
            nn.Linear(n_features, hidden),
            nn.ReLU(),
            nn.Dropout(0.2),
            nn.Linear(hidden, hidden // 2),
            nn.ReLU(),
            nn.Linear(hidden // 2, 1),
        )

    def forward(self, x):
        return self.net(x)

model = SalesForecastNet(n_features=10)
print(model)
print(sum(p.numel() for p in model.parameters()), "parameters")
```

Two rules: call `super().__init__()` first, and define the layers as attributes in `__init__` (that's how PyTorch finds the parameters).

> **Call `model(x)`, never `model.forward(x)`.** `model(x)` runs hooks and internal bookkeeping around `forward`. Calling `forward` directly skips them and breaks some features silently.

### The layers you need

| Layer | Does | Use for |
|---|---|---|
| `nn.Linear(in, out)` | `y = xW + b` | The workhorse. Fully connected. |
| `nn.ReLU()` | `max(0, x)` | Default activation. Fast, works. |
| `nn.Dropout(p)` | Randomly zeroes p% of activations during training | Reduce overfitting |
| `nn.BatchNorm1d(n)` | Normalises across the batch | Faster, more stable training |
| `nn.Sigmoid()` | Squashes to (0, 1) | Binary output probability |
| `nn.Softmax(dim=1)` | Turns scores into probabilities summing to 1 | Multi-class output |

**Why activations at all?** Without them, stacking `Linear` layers is pointless — a chain of linear functions collapses into one linear function. The non-linearity is what lets a network learn curves instead of straight lines.

> **Don't put `Softmax` at the end of a classifier.** `nn.CrossEntropyLoss` applies it internally (in a numerically stable way). Doing it twice makes gradients tiny and training stalls. Output **raw logits**; let the loss handle it. This trips up nearly everyone once.

### Losses

| Task | Loss | Model output |
|---|---|---|
| Regression (predict a number) | `nn.MSELoss()` | 1 raw value |
| Regression, robust to outliers | `nn.L1Loss()` / `nn.HuberLoss()` | 1 raw value |
| Binary classification | `nn.BCEWithLogitsLoss()` | 1 raw logit |
| Multi-class classification | `nn.CrossEntropyLoss()` | N raw logits |

> Use `BCEWithLogitsLoss`, not `Sigmoid` + `BCELoss` — same maths, numerically stable.
>
> `CrossEntropyLoss` expects targets as **class indices** (`0, 1, 2`), not one-hot vectors, and as `dtype=torch.long`.

### Optimizers

```python
import torch

optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)
```

| Optimizer | Notes |
|---|---|
| `SGD(params, lr, momentum=0.9)` | Classic. Often the best final result, needs more tuning. |
| **`Adam(params, lr=1e-3)`** | **Adaptive. The sensible default.** Start here, always. |
| `AdamW(params, lr=1e-3)` | Adam with correct weight decay. Best for transformers. |

**Learning rate is the hyperparameter that matters most.** Too high: loss explodes to `NaN`. Too low: nothing happens. Start at `1e-3` for Adam; try `3e-4` and `1e-2` if it misbehaves.

---

## 4. The training loop

Here it is. Six lines of actual mechanism.

```python
for epoch in range(n_epochs):
    model.train()                                   # 1. training mode
    for X_batch, y_batch in train_loader:
        X_batch, y_batch = X_batch.to(device), y_batch.to(device)

        optimizer.zero_grad()                       # 2. clear old gradients
        predictions = model(X_batch)                # 3. forward pass
        loss = criterion(predictions, y_batch)      # 4. how wrong were we
        loss.backward()                             # 5. compute gradients
        optimizer.step()                            # 6. adjust the weights
```

Read it as a sentence: **clear the gradients, make a guess, measure the error, work out which way each weight should move, move them.**

### The complete, real version

```python
import torch
import torch.nn as nn

def train(model, train_loader, val_loader, epochs=50, lr=1e-3, patience=5):
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
    model = model.to(device)

    criterion = nn.MSELoss()
    optimizer = torch.optim.Adam(model.parameters(), lr=lr)
    scheduler = torch.optim.lr_scheduler.ReduceLROnPlateau(optimizer, patience=3)

    history = {"train": [], "val": []}
    best_val, epochs_without_improvement = float("inf"), 0

    for epoch in range(epochs):
        # ---------- TRAIN ----------
        model.train()
        train_loss = 0.0
        for X, y in train_loader:
            X, y = X.to(device), y.to(device)

            optimizer.zero_grad()
            pred = model(X)
            loss = criterion(pred, y)
            loss.backward()
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            optimizer.step()

            train_loss += loss.item() * X.size(0)     # .item() — no graph kept
        train_loss /= len(train_loader.dataset)

        # ---------- VALIDATE ----------
        model.eval()
        val_loss = 0.0
        with torch.no_grad():
            for X, y in val_loader:
                X, y = X.to(device), y.to(device)
                val_loss += criterion(model(X), y).item() * X.size(0)
        val_loss /= len(val_loader.dataset)

        scheduler.step(val_loss)
        history["train"].append(train_loss)
        history["val"].append(val_loss)
        print(f"epoch {epoch:3d} | train {train_loss:.4f} | val {val_loss:.4f}")

        # ---------- EARLY STOPPING ----------
        if val_loss < best_val:
            best_val, epochs_without_improvement = val_loss, 0
            torch.save(model.state_dict(), "best_model.pth")
        else:
            epochs_without_improvement += 1
            if epochs_without_improvement >= patience:
                print(f"Early stopping at epoch {epoch}")
                break

    model.load_state_dict(torch.load("best_model.pth"))    # restore the best, not the last
    return model, history
```

### `model.train()` vs `model.eval()`

They don't train or evaluate anything. They flip a switch that changes the behaviour of two kinds of layer:

| Layer | `train()` | `eval()` |
|---|---|---|
| `Dropout` | Randomly zeroes activations | Passes everything through |
| `BatchNorm` | Uses this batch's statistics | Uses running averages from training |

> **Forgetting `model.eval()` before validation** gives you noisy, pessimistic validation numbers that jump around — because dropout is still randomly deleting parts of your network. **Forgetting `model.train()` after** means dropout never applies and your model overfits.
>
> `model.eval()` and `torch.no_grad()` are **different things** and you need both: one changes layer behaviour, the other stops gradient tracking.

### The extras, explained

- **`clip_grad_norm_`** — caps the size of gradients. Prevents one bad batch producing an enormous update that destroys the model ("exploding gradients"). Essential for RNNs/LSTMs, cheap insurance elsewhere.
- **`ReduceLROnPlateau`** — when validation loss stops improving, cut the learning rate. Big steps early to get close, small steps later to fine-tune.
- **Early stopping** — stop when validation loss stops improving, and **restore the best checkpoint**. Without the restore you keep the last (overfitted) model, which defeats the point.

### `state_dict`, not the whole model

```python
import torch

torch.save(model.state_dict(), "model.pth")               # ✓ just the weights

model = SalesForecastNet(n_features=10)                    # rebuild the architecture
model.load_state_dict(torch.load("model.pth"))
model.eval()
```

```python
import torch

torch.save(model, "model.pth")                             # ✗ pickles your class
```

Saving the whole model pickles the Python class definition. Rename the file or the class and it won't load. `state_dict` is a plain dictionary of tensors — portable and stable.

---

## 5. Data: `Dataset` and `DataLoader`

**`Dataset`** knows how to get one item. **`DataLoader`** batches, shuffles, and loads in parallel.

```python
import torch
from torch.utils.data import Dataset, DataLoader

class SalesDataset(Dataset):
    def __init__(self, features, targets):
        self.X = torch.tensor(features, dtype=torch.float32)
        self.y = torch.tensor(targets,  dtype=torch.float32).unsqueeze(1)   # (N,) → (N,1)

    def __len__(self):
        return len(self.X)

    def __getitem__(self, idx):
        return self.X[idx], self.y[idx]

train_loader = DataLoader(train_ds, batch_size=64, shuffle=True,  num_workers=4, pin_memory=True)
val_loader   = DataLoader(val_ds,   batch_size=64, shuffle=False)
```

Three methods, always the same: `__init__`, `__len__`, `__getitem__`.

> **`.unsqueeze(1)`** turns targets from `(N,)` to `(N, 1)` to match a model outputting `(N, 1)`. Skip it and broadcasting silently produces an `(N, N)` loss. This is *the* shape bug in regression.

**DataLoader settings:**

| Setting | Meaning |
|---|---|
| `batch_size` | Samples per update. 32–256 typical. Bigger = smoother but more memory. |
| `shuffle=True` | **Training only.** Never for validation/test. |
| `num_workers` | Parallel loading processes. Try 4. On Windows, must be inside `if __name__ == "__main__":`. |
| `pin_memory=True` | Faster CPU→GPU transfer. Only worth it with a GPU. |
| `drop_last=True` | Discard the final partial batch. Needed if BatchNorm chokes on a batch of 1. |

### Splitting — get this right or your metrics are fiction

```python
from torch.utils.data import random_split
train_ds, val_ds, test_ds = random_split(dataset, [0.7, 0.15, 0.15])
```

| Split | For |
|---|---|
| **Train** | Learning the weights |
| **Validation** | Choosing hyperparameters and when to stop |
| **Test** | The final honest number. **Touch it once, at the very end.** |

> **⏰ For time series — and the sales forecast IS time series — never use `random_split`.** Random splitting puts future data in the training set, so the model learns from data it couldn't possibly have had. Your validation score looks brilliant and the model is useless in production. This is **data leakage**, and it's the single most common serious mistake in applied ML.
>
> **Split by time:** train on Jan–Sep, validate on Oct, test on Nov–Dec. Always.

### Scaling features

Neural networks want inputs roughly in the -1..1 range. A `monetary_value` of 5,000 alongside a `frequency` of 3 makes training unstable.

```python
from sklearn.preprocessing import StandardScaler
import joblib

scaler = StandardScaler()
X_train = scaler.fit_transform(X_train_raw)     # fit on TRAIN ONLY
X_val   = scaler.transform(X_val_raw)           # transform only
X_test  = scaler.transform(X_test_raw)

joblib.dump(scaler, "scaler.pkl")               # ← save it. Serving needs the SAME one.
```

> **`fit_transform` on train, `transform` on everything else.** Fitting the scaler on all your data leaks the test set's mean and standard deviation into training — another leak, quieter but just as real.
>
> And **save the scaler**. At serving time you must apply the identical transform ([[FastAPI data and deployment]]). Refitting on live data silently changes what your model sees.

---

## 6. Reading the loss curve

```python
import matplotlib.pyplot as plt
plt.plot(history["train"], label="train")
plt.plot(history["val"], label="val")
plt.legend(); plt.xlabel("epoch"); plt.ylabel("loss")
```

| What you see | Meaning | Do |
|---|---|---|
| Both fall, then flatten together | ✅ Healthy | Nothing |
| Train falls, **val rises** | **Overfitting** | More dropout, more data, early stopping, smaller model |
| Both stay high and flat | **Underfitting** | Bigger model, higher LR, train longer, better features |
| Loss goes to `NaN` | Exploding gradients / LR too high | Lower LR 10×, add grad clipping, check for NaNs in input |
| Loss is wildly spiky | LR too high, or batch size too small | Lower LR, bigger batches |
| Loss doesn't move at all | LR too low, or gradients aren't flowing | Raise LR; check `zero_grad`/`backward`/`step` are all there and in order |

> **Always plot it.** A single final number tells you almost nothing. The shape of the curve tells you exactly what's wrong.

---

## 7. The sanity check that saves days

Before training on real data:

```python
# Take ONE batch and train until the model memorises it.
X, y = next(iter(train_loader))
X, y = X.to(device), y.to(device)

for i in range(500):
    optimizer.zero_grad()
    loss = criterion(model(X), y)
    loss.backward()
    optimizer.step()
    if i % 100 == 0:
        print(i, loss.item())
```

**The loss must go to nearly zero.** A model that cannot overfit a single batch has a bug — wrong shapes, gradients not flowing, a broken loss, a missing `zero_grad`. Finding that in 30 seconds instead of after a 4-hour training run is the highest-value habit in this note.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| Loss doesn't decrease at all | Missing `optimizer.step()` or `loss.backward()`; LR far too low | Check all six loop lines are present and ordered |
| Loss gets worse over epochs | Missing `optimizer.zero_grad()` | Add it at the top of the batch loop |
| Loss becomes `NaN` | LR too high, or NaN/inf in the input data | Lower LR 10×; `torch.isnan(X).any()`; add grad clipping |
| `Expected all tensors on same device` | Model on GPU, data on CPU (or vice versa) | `.to(device)` on both |
| Shape mismatch in the loss | Targets `(N,)` vs predictions `(N,1)` | `.unsqueeze(1)` on targets |
| Loss suspiciously large, model won't learn | Silent broadcasting to `(N,N)` | Print both shapes |
| Validation loss noisier than training | `model.eval()` not called | Call it before validating |
| Model overfits despite dropout | `model.train()` not called after validating | Call it at the top of each epoch |
| Out of memory | Batch too big; accumulating loss tensors; no `no_grad()` in eval | Smaller batch; `.item()`; wrap eval in `no_grad()` |
| Great validation score, useless in production | Data leakage — random split on time series, or scaler fit on all data | Split by time; fit the scaler on train only |
| Multi-class model won't train | `Softmax` applied before `CrossEntropyLoss` | Output raw logits |
| `expected scalar type Long but found Float` | `CrossEntropyLoss` targets must be `long` class indices | `y.long()` |
| Can't load a saved model | Saved the whole model, then renamed the class | Save `state_dict()` instead |
| DataLoader hangs on Windows | `num_workers > 0` outside a main guard | Wrap in `if __name__ == "__main__":` or set `num_workers=0` |
| Results differ every run | No seed | `torch.manual_seed(42)`, `np.random.seed(42)` |

---

## Practice checklist

- [ ] Tensors: shapes, dtypes, broadcasting, moving between CPU/GPU (`.to(device)`)
- [ ] Reading a shape as "batch first", and debugging by printing shapes
- [ ] Autograd: `requires_grad`, `.backward()`, and what a computational graph actually is
- [ ] **Why gradients accumulate and `zero_grad()` is mandatory**
- [ ] `torch.no_grad()`, `.detach()`, `.item()` — and the memory leak they prevent
- [ ] `nn.Module`: defining a model as a class, `forward()`, calling `model(x)` not `model.forward(x)`
- [ ] Loss functions and optimizers — which loss for which task, and why no `Softmax` before `CrossEntropyLoss`
- [ ] `model.train()` vs `model.eval()` — and that it's different from `no_grad()`
- [ ] `Dataset` and `DataLoader` — batching, shuffling, custom datasets
- [ ] Train/val/test splits, and **why time series must split by time**
- [ ] Feature scaling: fit on train only, save the scaler
- [ ] The training loop itself: zero grad → forward → loss → backward → step
- [ ] Early stopping, LR scheduling, gradient clipping
- [ ] Saving `state_dict()`, not the whole model
- [ ] Reading a loss curve

## Hands-on

- [ ] Write a full training loop by hand (no high-level wrapper) for linear regression on synthetic data
- [ ] Delete `optimizer.zero_grad()` and watch training degrade — so you recognise the symptom
- [ ] Overfit a single batch to near-zero loss before doing anything else
- [ ] Repeat for a small classifier on a real tabular dataset — plot the loss curve
- [ ] Deliberately overfit (tiny dataset, big model) and see train/val diverge on the plot
- [ ] Build the `SalesDataset` for the capstone, split **by time**, and scale correctly

## Resources

- [PyTorch: Learn the Basics](https://pytorch.org/tutorials/beginner/basics/intro.html)
- [Deep Learning with PyTorch: 60-Minute Blitz](https://pytorch.org/tutorials/beginner/deep_learning_60min_blitz.html)
- [Karpathy: A Recipe for Training Neural Networks](https://karpathy.github.io/2019/04/25/recipe/) — read this once a year

## Next

[[CNNs and transfer learning]]
