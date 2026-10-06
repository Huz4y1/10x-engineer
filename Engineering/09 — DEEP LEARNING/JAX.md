---
tags: [dictionary, jax, deep-learning, autodiff, research]
status: not-started
---

# JAX

Template: [[_Dictionary template]] · Section: [[09 — DEEP LEARNING]]

---

## One sentence

JAX is NumPy that can differentiate itself, compile itself to GPU/TPU, and vectorise itself automatically.

## In simple words

Write a normal maths function in NumPy syntax. Then JAX hands you four transformations you can apply to it:

| Transform | Does |
|---|---|
| **`grad`** | Give me the derivative of this function |
| **`jit`** | Compile it to run fast on GPU/TPU |
| **`vmap`** | Run it on a whole batch, automatically |
| **`pmap`** | Run it across many devices |

They **compose**. `jit(grad(vmap(f)))` is a compiled, batched gradient — and you wrote none of that machinery.

## The problem it solves

PyTorch and TensorFlow bundle autodiff *with* a neural-network framework. If your research isn't shaped like a standard neural network — physics simulation, optimisation, Bayesian inference, custom numerical methods — you're fighting a `nn.Module` abstraction you don't want.

JAX unbundles it: **autodiff and compilation as transformations over pure functions**. Neural networks become one application among many.

> **The other reason it exists: TPUs.** JAX is what Google DeepMind uses, and its compilation model (XLA) targets TPUs better than anything else.

## The one rule: functions must be pure

**No side effects. No in-place mutation. Same input → same output, always.**

```python
# ✗ NOT allowed - JAX arrays are immutable
x[0] = 5

# ✓ returns a NEW array
x = x.at[0].set(5)
```

This is the price of admission, and everything else follows from it: purity is what makes `jit`, `grad` and `vmap` safe to apply automatically.

> **Consequence: JAX has no hidden state.** Model parameters are just a dict (a "PyTree") you pass in and get back. There is no `self.weights`, no `.to(device)` on a module, no `optimizer.zero_grad()`. Some people find this liberating; others find it exhausting.

## Randomness is explicit

```python
import jax

key = jax.random.PRNGKey(42)
key, subkey = jax.random.split(key)         # you MUST split
noise = jax.random.normal(subkey, (3, 3))
```

> **There is no global random seed.** Every random call needs a key, and reusing a key gives identical results. This is deliberate: it makes randomness reproducible under `jit` and `vmap`, where a global seed would be ambiguous. It also catches everyone at first.

## Code

### Level 1 — the four transformations

```python
import jax, jax.numpy as jnp

def f(x):
    return jnp.sum(x ** 2)

# derivative
df = jax.grad(f)
df(jnp.array([1.0, 2.0, 3.0]))       # [2., 4., 6.]

# compiled
f_fast = jax.jit(f)

# batched: apply to each row automatically
f_batched = jax.vmap(f)
f_batched(jnp.ones((100, 3)))        # shape (100,)

# all three at once
fast_batched_grad = jax.jit(jax.vmap(jax.grad(f)))
```

> **`vmap` is the one with no PyTorch equivalent.** Write a function for a *single* example — no batch dimension anywhere — and `vmap` makes it batched. No `unsqueeze`, no broadcasting puzzles, no `(N,1)` vs `(N,)` bugs.

### Level 2 — training a model by hand

```python
import jax
import jax.numpy as jnp

def init_params(key, sizes):
    params = []
    for i in range(len(sizes) - 1):
        key, k = jax.random.split(key)
        w = jax.random.normal(k, (sizes[i], sizes[i+1])) * jnp.sqrt(2 / sizes[i])
        params.append({"w": w, "b": jnp.zeros(sizes[i+1])})
    return params

def forward(params, x):
    for layer in params[:-1]:
        x = jax.nn.relu(x @ layer["w"] + layer["b"])
    last = params[-1]
    return x @ last["w"] + last["b"]

def loss_fn(params, x, y):
    return jnp.mean((forward(params, x) - y) ** 2)

@jax.jit
def update(params, x, y, lr=1e-3):
    grads = jax.grad(loss_fn)(params, x, y)
    return jax.tree.map(lambda p, g: p - lr * g, params, grads)   # <- PyTree magic
```

> **`jax.tree.map` applies a function across a whole nested structure** of parameters. That one line is the optimiser. Params are just nested dicts — no `nn.Module`, no `state_dict`.

### Level 3 — the real ecosystem

Nobody hand-writes optimisers in production JAX:

| Library | Does | ~PyTorch equivalent |
|---|---|---|
| **Flax** (`flax.nnx`) | Neural network layers | `torch.nn` |
| **Optax** | Optimisers, schedules, clipping | `torch.optim` |
| **Orbax** | Checkpointing | `torch.save` |
| Equinox | Alternative NN library, very "JAX-native" | — |

```python
import jax
import optax
from flax import nnx

optimizer = optax.chain(
    optax.clip_by_global_norm(1.0),
    optax.adamw(learning_rate=1e-3),
)
opt_state = optimizer.init(params)

@jax.jit
def train_step(params, opt_state, x, y):
    loss, grads = jax.value_and_grad(loss_fn)(params, x, y)   # loss AND grads, one pass
    updates, opt_state = optimizer.update(grads, opt_state, params)
    return optax.apply_updates(params, updates), opt_state, loss
```

> `jax.value_and_grad` gives you the loss and its gradient in a single forward-backward pass. Using `grad` and then recomputing the loss separately does the work twice.

## What happens under the hood

When you call a `@jax.jit` function:

1. JAX **traces** it with abstract placeholder values (shape + dtype, no actual numbers).
2. It records the operations into a graph (`jaxpr`).
3. **XLA** compiles that graph — fusing operations, allocating buffers, optimising for the target device.
4. The compiled binary is **cached**, keyed on input shapes and dtypes.
5. Subsequent calls with the same shapes skip straight to the compiled code.

> **This is why the first call is slow and the rest are fast.** And why **changing input shape triggers full recompilation** — feed it 100 different batch sizes and you compile 100 times. Pad to fixed shapes in production.

## When to use it

- Research needing custom gradients or unusual maths
- Physics simulation, differentiable optimisation, Bayesian inference
- TPU training
- Large-scale distributed training where `pmap`/sharding matters
- You want `vmap`

## When NOT to use it

| Situation | Use instead |
|---|---|
| Standard supervised learning | **PyTorch** — more examples, more libraries |
| You want pretrained models | PyTorch / Hugging Face |
| Production serving | ONNX ([[Model export and serving]]) |
| Team unfamiliar with functional programming | PyTorch — JAX has a real learning curve |
| Deploying to mobile/browser | [[TensorFlow and Keras]] |

> **Honest positioning: JAX is a research and large-scale-training tool.** For this vault's stack, PyTorch is the right default. Learn JAX if you go into research ([[27 — ML RESEARCH]]) or hit a problem PyTorch's abstractions fight you on.

## Common mistakes

| Mistake | What happens | Fix |
|---|---|---|
| In-place mutation `x[0] = 5` | `TypeError` — arrays are immutable | `x.at[0].set(5)` |
| Reusing a PRNG key | Identical "random" numbers | `jax.random.split` every time |
| Python `if` on a traced value | `ConcretizationTypeError` | `jax.lax.cond` |
| Python loop over data in `jit` | Unrolls into a giant graph; slow compile | `jax.lax.scan` / `fori_loop` |
| Varying input shapes | Recompiles every call | Pad to fixed shapes |
| `print()` inside `jit` | Prints once, during tracing | `jax.debug.print` |
| Expecting `.grad` attributes | JAX has no stateful gradients | `grad` returns a new structure |
| Not using `value_and_grad` | Double the compute | Use it |

## Debugging

1. **Disable jit** — `jax.disable_jit()` context, or remove the decorator. If the bug vanishes, it's a tracing issue.
2. `jax.debug.print("{x}", x=x)` — works inside compiled code.
3. `jax.make_jaxpr(f)(x)` — see the traced graph.
4. `jax.config.update("jax_debug_nans", True)` — raise at the operation that produced a NaN, not later.
5. Check for recompilation — if it's mysteriously slow, shapes are probably varying.

## Performance

| Technique | Effect |
|---|---|
| `jit` the **outermost** function | Fuses the most operations |
| `vmap` instead of a Python loop | Vectorised |
| `lax.scan` instead of a Python loop | Compiles to a real loop, not an unrolled graph |
| `donate_argnums` | Reuse input buffers — saves memory |
| `bfloat16` | ~2× on TPU/modern GPU |

> **Jit the biggest sensible unit.** Jitting each small function separately prevents XLA from fusing across them, which is where most of the speedup comes from.

## Cloud equivalents

| Local | Azure | AWS | GCP |
|---|---|---|---|
| JAX on GPU | Azure ML (GPU) | SageMaker (GPU/Trainium) | **Vertex AI + TPU** ← JAX's natural home |

## Prerequisites

[[09 — DEEP LEARNING]] · [[Mathematics reference]] · [[24 — SCIENTIFIC COMPUTING]] · comfort with pure functions

## Learning progression

- **Beginner:** `jnp` as NumPy; `grad`, `jit`, `vmap` on scalar functions
- **Intermediate:** PyTrees, `value_and_grad`, Optax, a hand-written training loop
- **Advanced:** `lax.scan`/`cond`, custom VJPs, `pmap` and sharding, TPU
- **Research:** differentiable simulation, implicit differentiation, neural ODEs

## Related

[[09 — DEEP LEARNING]] · [[10 — PYTORCH]] · [[TensorFlow and Keras]] · [[24 — SCIENTIFIC COMPUTING]] · [[Mathematics reference]] · [[27 — ML RESEARCH]] · [[Large language models]]
