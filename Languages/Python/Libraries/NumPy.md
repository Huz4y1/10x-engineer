---
tags: [python, library, numpy, arrays, maths]
---

# NumPy

Arrays and numerical computing. **Underneath pandas, scikit-learn and PyTorch.**

Index: [[Python libraries]] · Maths: [[Mathematics reference]]

### Installing

> **Same commands on Windows, WSL and Linux.** The differences are called out below where they exist.

```bash
uv add numpy                      # in a uv project - PREFERRED
uv pip install numpy              # into the active venv, no project file
pip install numpy                 # plain pip, if you're not using uv
```

**If you don't have a project yet:**

```bash
uv init myproject && cd myproject     # creates pyproject.toml and .venv
uv add numpy
uv run python -c "import numpy; print(numpy.__version__)"           # uv run uses the project venv automatically
```

**Without uv — a plain virtual environment:**

| | Windows (PowerShell) | WSL / Linux |
|---|---|---|
| Create | `python -m venv .venv` | `python3 -m venv .venv` |
| Activate | `.venv\Scripts\Activate.ps1` | `source .venv/bin/activate` |
| Install | `pip install numpy` | `pip install numpy` |
| Leave | `deactivate` | `deactivate` |

**Check it worked:**

```bash
python -c "import numpy; print(numpy.__version__)"
```

> **Wheels are prebuilt for Windows, macOS and Linux** — there's no compiler needed and no reason to build from source.

> ⚠️ **NumPy 2.x broke some C-extension compatibility.** If another library errors with *compiled using NumPy 1.x cannot be run in NumPy 2.x*, either upgrade that library or pin `numpy<2`.

> ⚠️ **Never `pip install` outside a virtual environment.** On WSL and Linux it fights the system Python; newer versions refuse outright with *externally-managed-environment*. On Windows it silently pollutes your global install. `uv` makes a venv for you — use it.

> **On Windows: if `python` opens the Microsoft Store**, Python isn't really installed. Get it from python.org (tick *Add to PATH*), or `winget install Python.Python.3.12`.

> **In WSL: work in `~/`, not `/mnt/c/`.** Installing into a project on the Windows drive is dramatically slower, and file-watching (`--reload`, `pytest-watch`) often doesn't fire at all. See [[Setting up a dev machine]].

```python
import numpy as np
```

---

## Why it exists

A Python list of a million numbers is a million separate objects with pointers. A NumPy array is **one contiguous block of memory**, so the CPU can process it with vector instructions.

```python
sum(x * 2 for x in py_list)     # ~200 ms - a million interpreter steps
(arr * 2).sum()                 # ~2 ms   - one call into compiled C
```

That 100× is why every numerical library in Python is built on it.

## 1. Creating arrays

```python
import numpy as np

np.array([1,2,3])
np.array([[1,2],[3,4]])
np.zeros((3,4)) · np.ones((2,3)) · np.full((2,2), 7)
np.empty((2,2))                          # uninitialised - faster, contains junk
np.eye(3)                                # identity matrix
np.arange(0, 10, 2)                      # like range
np.linspace(0, 1, 11)                    # 11 evenly spaced points INCLUSIVE
np.logspace(0, 3, 4)
np.zeros_like(arr) · np.ones_like(arr)

rng = np.random.default_rng(42)          # ✅ modern API
rng.random((3,3)) · rng.normal(0, 1, 100) · rng.integers(0, 10, 5)
rng.choice(items, size=5, replace=False) · rng.shuffle(arr)
```

> **Use `np.random.default_rng(seed)`, not `np.random.seed()`.** The legacy global API is discouraged and makes reproducibility harder.

## 2. Attributes and dtypes

```python
import numpy as np

arr.shape · arr.ndim · arr.size · arr.dtype · arr.itemsize · arr.nbytes
arr.astype(np.float32)
```

| dtype | Bits | Use |
|---|---|---|
| `int32` / `int64` | 32/64 | Whole numbers |
| **`float32`** | 32 | **PyTorch default** |
| `float64` | 64 | **NumPy default** |
| `bool` | 8 | Masks |

> ⚠️ **NumPy defaults to float64, PyTorch to float32.** Passing a float64 array to an ONNX model errors; passing it to PyTorch silently doubles memory. **`.astype(np.float32)`** ([[Model export and serving]]).

## 3. Indexing and slicing

```python
import numpy as np

arr[0] · arr[-1] · arr[1:5] · arr[::2] · arr[::-1]
arr[0, 1]          # 2D - row 0, col 1
arr[0]             # whole row
arr[:, 1]          # whole column
arr[1:3, 0:2]      # sub-block
arr[..., 0]        # last axis

# boolean masking - the most useful feature
arr[arr > 5]
arr[(arr > 5) & (arr < 10)]              # brackets mandatory
arr[arr > 5] = 0                         # assign through a mask

# fancy indexing
arr[[0, 2, 4]]
np.where(arr > 5)                        # indices where true
np.where(arr > 5, arr, 0)                # ternary: where/then/else
```

> **A slice is a VIEW, not a copy.** `b = a[0:3]; b[0] = 99` changes `a`. Use `a[0:3].copy()` if you need independence. Boolean and fancy indexing *do* copy.

## 4. Shape manipulation

```python
import numpy as np

arr.reshape(3, 4) · arr.reshape(-1, 2)   # -1 = infer this dimension
arr.ravel()                              # flatten (view where possible)
arr.flatten()                            # flatten (always a copy)
arr.T · arr.transpose(1, 0, 2)
arr.swapaxes(0, 1)
np.expand_dims(arr, axis=0) · arr[np.newaxis, :]     # add a dimension
np.squeeze(arr)                          # remove size-1 dimensions

np.concatenate([a, b], axis=0)
np.vstack([a, b]) · np.hstack([a, b]) · np.stack([a, b], axis=0)
np.split(arr, 3) · np.array_split(arr, 3)
np.tile(arr, 2) · np.repeat(arr, 2)
```

## 5. Broadcasting

Operations on different-shaped arrays auto-expand.

```
(3, 4) + (4,)    -> ✓  (4 vs 4 match, 3 kept)
(3, 4) + (3, 1)  -> ✓  (1 stretches to 4)
(3, 4) + (3,)    -> ✗  (4 vs 3 mismatch)
```

**The rule: align shapes from the RIGHT. Dimensions must be equal, or one must be 1.**

```python
arr - arr.mean(axis=0)                   # centre each column
arr / arr.sum(axis=1, keepdims=True)     # keepdims preserves the shape for broadcasting
```

> ⚠️ **Broadcasting causes silent bugs.** `(32,1) - (32,)` broadcasts to `(32,32)` — a full matrix instead of a vector. Your loss computes fine and is nonsense. **When a result is weirdly shaped, print both shapes first** ([[Tensors, autograd and the training loop]]).

## 6. Maths

```python
import numpy as np

a + b · a - b · a * b · a / b            # ELEMENT-WISE
a @ b                                    # MATRIX multiplication
np.dot(a, b) · np.matmul(a, b)

np.sqrt · np.exp · np.log · np.log10 · np.log1p
np.sin · np.cos · np.tan
np.abs · np.round · np.floor · np.ceil · np.clip(arr, 0, 1)
np.sign · np.power(a, 2) · np.mod(a, b)
```

> **`*` is element-wise; `@` is matrix multiplication.** Confusing them is the classic linear-algebra bug ([[Mathematics reference]]).

## 7. Aggregation

```python
import numpy as np

arr.sum() · arr.mean() · arr.std() · arr.var()
arr.min() · arr.max() · arr.argmin() · arr.argmax()
arr.sum(axis=0)     # collapse ROWS -> one value per column
arr.sum(axis=1)     # collapse COLUMNS -> one value per row
arr.cumsum() · arr.cumprod()
np.median(arr) · np.percentile(arr, [25, 50, 75]) · np.quantile(arr, 0.5)
np.corrcoef(a, b) · np.cov(a, b)
arr.any() · arr.all() · np.count_nonzero(arr > 5)
np.unique(arr, return_counts=True)
```

> **`axis=0` collapses rows (result is per-column); `axis=1` collapses columns.** Remember it as "the axis that disappears".

## 8. NaN handling

```python
import numpy as np

np.nan · np.inf · np.isnan(arr) · np.isinf(arr) · np.isfinite(arr)
np.nansum · np.nanmean · np.nanstd · np.nanmax        # ignore NaN
np.nan_to_num(arr, nan=0.0, posinf=1e10)

np.nan == np.nan                         # False! Always use np.isnan()
```

## 9. Sorting and searching

```python
import numpy as np

np.sort(arr) · arr.sort()                # copy vs in place
np.argsort(arr)                          # indices that would sort it
arr[np.argsort(arr[:, 1])]               # sort rows by column 1
np.searchsorted(sorted_arr, value)       # binary search - O(log n)
np.partition(arr, 5) · np.argpartition(arr, 5)       # top-k without full sort
np.isin(arr, [1,2,3])
```

## 10. Linear algebra

```python
import numpy as np

np.linalg.inv(A) · np.linalg.det(A) · np.linalg.matrix_rank(A)
np.linalg.solve(A, b)                    # ✅ better than inv(A) @ b
np.linalg.eig(A) · np.linalg.eigh(A)     # eigenvalues/vectors
np.linalg.svd(A)
np.linalg.norm(v) · np.linalg.norm(v, ord=1)
np.trace(A) · np.linalg.qr(A) · np.linalg.cholesky(A)
```

> **`solve(A, b)` not `inv(A) @ b`.** Inverting is slower and numerically worse ([[24 — SCIENTIFIC COMPUTING]]).

## 11. Saving and loading

```python
import numpy as np

np.save("a.npy", arr) · np.load("a.npy")
np.savez("a.npz", x=a, y=b) · np.load("a.npz")["x"]
np.savetxt("a.csv", arr, delimiter=",") · np.loadtxt("a.csv", delimiter=",")
```

> ⚠️ **`np.load(..., allow_pickle=True)` executes arbitrary code.** Never use it on files you didn't create. Plain `.npy` of numeric arrays is safe ([[26 — SECURITY]]).

## 12. Performance

```python
import numpy as np

# ✗ ~500 ms
result = np.array([x * 2 + 1 for x in arr])
# ✓ ~1 ms
result = arr * 2 + 1

arr += 1                                 # in place - no new allocation
np.add(a, b, out=c)                      # write into an existing array
np.ascontiguousarray(arr)                # fix a slow non-contiguous array
```

| Rule | Why |
|---|---|
| **Never loop over an array in Python** | That's the whole point of NumPy |
| Use `out=` and in-place ops | Avoids allocation in hot loops |
| Prefer `float32` for ML | Half the memory and bandwidth |
| Keep arrays contiguous | Strided access is much slower |

## Where NumPy sits

```
pandas  ·  scikit-learn  ·  SciPy  ·  Matplotlib
                    |
                  NumPy
                    |
              C / BLAS / LAPACK
```

```python
import pandas as pd
import torch

df.to_numpy() · pd.DataFrame(arr)
torch.from_numpy(arr) · tensor.numpy()   # SHARES memory - mutating one changes both
```

## Related

[[Python libraries]] · [[pandas]] · [[Mathematics reference]] · [[24 — SCIENTIFIC COMPUTING]] · [[Tensors, autograd and the training loop]] · [[scikit-learn]]
