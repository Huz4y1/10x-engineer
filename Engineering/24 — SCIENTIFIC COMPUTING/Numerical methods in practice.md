---
tags: [scientific-computing, numpy, scipy, floating-point, deep-dive]
---

# Numerical methods in practice

The index is [[24 — SCIENTIFIC COMPUTING]]. This is the part that actually bites you — floating point, SciPy, and why your loss became `NaN`.

Arrays: [[NumPy]] · Maths: [[Mathematics reference]] · Speed: [[When to leave Python]]

---

## Floating point, explained properly

A computer stores numbers in binary. Some decimals have **no exact binary form**, the same way 1/3 has no exact decimal form — 0.333… never terminates.

`0.1` is one of those numbers. The computer stores the nearest value it can, which is very slightly off.

```python
0.1 + 0.2            # 0.30000000000000004
0.1 + 0.2 == 0.3     # False
```

**This is not a bug.** It's the same everywhere: Python, C, Rust, JavaScript, your calculator. It's how binary fractions work.

### The consequences

```python
import numpy as np

# NEVER compare floats with ==
if abs(a - b) < 1e-9: ...                  # do this
import math; math.isclose(a, b, rel_tol=1e-9)
np.allclose(arr1, arr2)                    # arrays
```

```python
# NEVER accumulate float error in a loop counter
x = 0.0
while x != 1.0:      # may never be exactly 1.0 -> INFINITE LOOP
    x += 0.1

for i in range(10):  # count in integers instead
    x = i / 10
```

> ⚠️ **Money is never a float.** `0.1 + 0.2 != 0.3` means pennies vanish. Use `DECIMAL` in the database ([[Data modeling]]) and `decimal.Decimal` or integer pence in code.

```python
from decimal import Decimal
Decimal("0.1") + Decimal("0.2") == Decimal("0.3")     # True
```

---

## Precision levels

| Type | Bits | ~Decimal digits | Where |
|---|---|---|---|
| `float64` | 64 | ~15 | NumPy default, scientific work |
| `float32` | 32 | ~7 | **PyTorch default**, GPUs |
| `float16` | 16 | ~3 | Mixed-precision training |
| `bfloat16` | 16 | ~2 | ML training — wider range than fp16 |

> **fp16 vs bf16:** fp16 has more precision but a much smaller *range*, so gradients underflow to zero. bf16 keeps float32's range with less precision, which is why modern training prefers it ([[CUDA and GPU programming]]).

```python
import numpy as np

np.float32(16777217)       # 16777216.0 - fp32 can't represent this integer
a = np.array([1, 2], dtype=np.float32)
b = a.astype(np.float64)   # widening is safe; narrowing loses data silently
```

> ⚠️ **Silent dtype promotion is a real bug source.** Mixing float32 and float64 in NumPy promotes to float64 and doubles your memory. Check `arr.dtype` when memory jumps unexpectedly.

---

## Catastrophic cancellation

Subtracting two nearly equal numbers destroys precision — the leading digits cancel and you're left with the error.

```python
a = 1.0000000001
b = 1.0000000000
a - b                # 1.000000082740371e-10  <- only ~2 correct digits left
```

**The classic fix — the quadratic formula:**

```python
import math

# UNSTABLE when b² >> 4ac : (-b + sqrt(b²-4ac)) cancels
x1 = (-b + math.sqrt(b*b - 4*a*c)) / (2*a)

# STABLE: compute the well-conditioned root, get the other from the product
q  = -0.5 * (b + math.copysign(math.sqrt(b*b - 4*a*c), b))
x1, x2 = q / a, c / q
```

> **Where this shows up in your work:** computing variance as `E[x²] - E[x]²` is catastrophically unstable for large means. Use `np.var`, which uses a two-pass or Welford algorithm.

---

## Numerical stability in ML

### The log-sum-exp trick

```python
import numpy as np

# NAIVE - overflows to inf for x around 800
def softmax_bad(x):
    e = np.exp(x)
    return e / e.sum()

# STABLE - subtract the max first. Mathematically identical, numerically safe.
def softmax(x):
    e = np.exp(x - x.max())
    return e / e.sum()
```

`exp(1000)` is `inf`. `inf / inf` is `NaN`. Subtracting the max makes the largest exponent `exp(0) == 1`, and the answer is unchanged because the constant cancels top and bottom.

> **This is why `BCEWithLogitsLoss` beats `Sigmoid` + `BCELoss` in PyTorch**, and why `CrossEntropyLoss` takes raw logits rather than softmax output. The fused version does this trick internally. **Always pass logits** ([[Tensors, autograd and the training loop]]).

### Log space

```python
import math

p = 0.001
p ** 1000                       # 0.0 - underflow, information destroyed
1000 * math.log(p)              # -6907.75 - fine

from scipy.special import logsumexp
logsumexp(log_probs)            # add probabilities in log space, safely
```

> **Multiplying many probabilities underflows to zero.** Work in log space and add. Every probabilistic model does this.

### Division and logs

```python
import numpy as np

np.log(x + 1e-8)                # guard against log(0) = -inf
a / (b + 1e-8)                  # guard against divide-by-zero
np.clip(p, 1e-7, 1 - 1e-7)      # keep probabilities off the boundary
```

> **Where does `NaN` come from?** `0/0`, `inf - inf`, `log(0)`, `sqrt(-1)`. And **`NaN` is contagious** — one `NaN` in a gradient poisons every weight on the next step.

```python
import numpy as np
import torch

np.isnan(arr).any()
torch.isnan(loss)
with np.errstate(invalid="raise", divide="raise"):    # turn warnings into exceptions
    ...
```

---

## Conditioning

A problem is **ill-conditioned** when a tiny change in the input causes a huge change in the output. No algorithm can fix it — the problem itself is fragile.

```python
import numpy as np

np.linalg.cond(A)      # < 1e3 fine | ~1e8 losing half your digits | > 1e12 don't trust it
```

```python
import numpy as np

x = np.linalg.solve(A, b)        # ALWAYS this
x = np.linalg.inv(A) @ b         # NEVER this - slower AND less accurate
```

> **Never invert a matrix to solve a system.** `solve` uses LU decomposition: fewer operations and better conditioning. Explicitly inverting is one of the clearest signs of numerical inexperience.

> **This is also why you standardise features.** A feature ranging 0–1 alongside one ranging 0–1,000,000 makes the problem ill-conditioned, and gradient descent zigzags ([[Classical ML in practice]]).

---

## SciPy — the toolbox

```bash
uv add scipy
```

| Module | For |
|---|---|
| `scipy.optimize` | Minimisation, curve fitting, root finding |
| `scipy.integrate` | Integration, ODE solving |
| `scipy.linalg` | Decompositions beyond NumPy's |
| `scipy.signal` | Filtering, FFT, convolution |
| `scipy.stats` | Distributions, hypothesis tests |
| `scipy.interpolate` | Filling between known points |
| `scipy.spatial` | Distances, KD-trees, convex hulls |
| `scipy.sparse` | Matrices that are mostly zeros |

### Curve fitting

```python
import numpy as np
from scipy.optimize import curve_fit

def model(t, a, tau, c):
    return a * np.exp(-t / tau) + c

params, cov = curve_fit(model, t_data, y_data, p0=[1.0, 10.0, 0.0])
errors = np.sqrt(np.diag(cov))          # 1-sigma uncertainty on each parameter
```

> **Always give `p0`.** Without a starting guess the optimiser begins at all-ones and frequently fails to converge on anything physical.

> **`np.sqrt(np.diag(cov))` is your error bar.** A fitted parameter without an uncertainty is half an answer.

### Optimisation

```python
from scipy.optimize import minimize

res = minimize(cost, x0, method="L-BFGS-B", bounds=[(0, None), (0, 10)])
if not res.success:
    raise RuntimeError(res.message)     # CHECK THIS
print(res.x, res.fun, res.nit)
```

| Method | Use |
|---|---|
| `L-BFGS-B` | Smooth, with bounds — **the default choice** |
| `Nelder-Mead` | No gradients available, low dimension |
| `SLSQP` | Constraints |
| `differential_evolution` | Global search, many local minima |

> ⚠️ **`minimize` returns a result whether or not it worked.** Check `res.success` every time — a silently unconverged fit looks exactly like a good one.

### ODEs — simulating physics

```python
import numpy as np
from scipy.integrate import solve_ivp

def spring(t, y, k, m, c):
    x, v = y
    return [v, (-k * x - c * v) / m]

sol = solve_ivp(spring, [0, 10], y0=[1.0, 0.0], args=(10.0, 1.0, 0.5),
                t_eval=np.linspace(0, 10, 500), rtol=1e-8)
```

> **`solve_ivp`, not the old `odeint`.** Adaptive step size, event detection, choice of method.

> **Stiff systems** — where fast and slow dynamics coexist — need `method="Radau"` or `"BDF"`. If your simulation takes forever or blows up, that's usually why ([[22 — CONTROL SYSTEMS]], [[PID and Kalman filters]]).

### Signal processing

```python
from scipy import signal

b, a = signal.butter(4, 10, btype="low", fs=1000)      # 4th-order, 10 Hz cutoff
clean = signal.filtfilt(b, a, noisy)                   # zero phase shift

f, Pxx = signal.welch(x, fs=1000)                      # power spectrum
peaks, _ = signal.find_peaks(x, height=0.5, distance=20)
```

> **`filtfilt` runs the filter forwards and backwards, so it introduces no phase lag** — essential for offline sensor data. For real-time you must use `lfilter` and accept the delay, because you can't filter backwards through time you haven't seen ([[Sensors overview]]).

> ⚠️ **Nyquist:** you must sample at more than **twice** the highest frequency present, or that frequency aliases and appears as a false low-frequency signal. Sampling a 60 Hz vibration at 100 Hz shows you a ghost at 40 Hz that isn't real.

---

## Vectorisation — the highest-value habit

```python
# 1000x slower
out = []
for i in range(len(a)):
    out.append(a[i] * b[i] + c[i])

# fast, and clearer
out = a * b + c
```

```python
import numpy as np

np.where(cond, x, y)              # vectorised if/else
np.select([c1, c2], [v1, v2], default=0)
arr[mask]                         # boolean indexing instead of filtering in a loop
np.einsum("ij,jk->ik", A, B)      # explicit, readable tensor contractions
```

> **Before rewriting anything in C or Rust, vectorise.** A NumPy rewrite typically gives 10–100× because the loop moves into compiled code. Only after that is a language change worth considering ([[When to leave Python]], [[Systems performance]]).

```python
import numpy as np

np.add(a, b, out=a)               # in-place: no new allocation
a += b                            # same
```

> **For big arrays, allocation dominates.** In-place operations avoid copying gigabytes.

---

## Debugging numerical code

```python
import numpy as np

np.isnan(x).any(), np.isinf(x).any()
np.abs(x).max()                                    # is anything exploding?
np.linalg.cond(A)                                  # is the problem itself fragile?
np.testing.assert_allclose(got, want, rtol=1e-6)   # comparing floats in tests
```

> **When a result is wrong, check in this order:** dtype → `NaN`/`inf` → magnitude → conditioning → algorithm. Most "the maths is broken" turns out to be float32 where you assumed float64.

---

## Common mistakes

| Mistake | Consequence |
|---|---|
| `==` on floats | Comparison that's false for no visible reason |
| Float loop counter | Infinite loop or off-by-one |
| Float for money | Missing pennies |
| `exp` without subtracting the max | `inf`, then `NaN` |
| Multiplying many probabilities | Underflow to zero |
| `inv(A) @ b` | Slower and less accurate than `solve` |
| Unscaled features | Ill-conditioned, slow convergence |
| Not checking `res.success` | Unconverged fit that looks fine |
| `odeint` on a stiff system | Hangs or explodes |
| Sampling below Nyquist | Aliasing — a signal that isn't there |
| Python loop over an array | 10–100× slower than needed |
| Assuming float64 | It's float32 in PyTorch |

## Related

[[24 — SCIENTIFIC COMPUTING]] · [[NumPy]] · [[Mathematics reference]] · [[When to leave Python]] · [[Tensors, autograd and the training loop]] · [[CUDA and GPU programming]] · [[PID and Kalman filters]] · [[22 — CONTROL SYSTEMS]] · [[Data modeling]] · [[Classical ML in practice]]
