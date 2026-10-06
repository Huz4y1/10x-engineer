---
tags: [python, language, decorators]
---

# Decorators

Language: [[Python]]

---

## What they are

A decorator is a function that wraps another function.

```python
@timer
def slow(): ...

# is exactly the same as
def slow(): ...
slow = timer(slow)
```

## Writing one

```python
import time
from functools import wraps

def timer(func):
    @wraps(func)                        # preserves __name__, __doc__, signature
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__}: {time.perf_counter() - start:.3f}s")
        return result
    return wrapper
```

> **Always `@wraps(func)`.** Without it the decorated function reports the wrapper's name, loses its docstring, and breaks anything that inspects signatures — including [[FastAPI reference|FastAPI]], which reads type hints to build routes.

## With arguments

```python
import time

def retry(times: int = 3):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception:
                    if attempt == times - 1:
                        raise
                    time.sleep(2 ** attempt)
        return wrapper
    return decorator

@retry(times=5)
def call_api(): ...
```

Three levels: `retry(5)` returns `decorator`, which returns `wrapper`.

## Built-in decorators you'll use constantly

| Decorator | Does | Where |
|---|---|---|
| `@property` | Method accessed as an attribute | [[Classes (Python)]] |
| `@staticmethod` / `@classmethod` | No `self` / gets `cls` | [[Classes (Python)]] |
| `@dataclass` | Generates `__init__`, `__repr__`, `__eq__` | [[Classes (Python)]] |
| `@lru_cache` | Memoise a pure function | [[Functions (Python)]] |
| `@cached_property` | Compute once per instance | — |
| `@abstractmethod` | Subclasses must implement | [[Classes (Python)]] |
| `@contextmanager` | Turn a generator into a `with` block | [[Classes (Python)]] |

## Framework decorators in this vault

```python
from pyspark.sql import functions as F
from pyspark.sql import types as T
import jax
import pytest
import streamlit as st

@app.get("/items")                # [[FastAPI reference]]
@st.cache_data(ttl=300)           # [[Streamlit]]
@F.udf(returnType=T.StringType()) # [[PySpark reference]]
@pytest.fixture                   # [[pytest]]
@pytest.mark.parametrize(...)     # [[pytest]]
@jax.jit                          # [[JAX]]
```

> Every one of these is the same mechanism: a function that takes your function and returns a wrapped version. Understanding decorators makes all of them stop being magic.

## Class decorators and stacking

```python
@decorator_a          # applied SECOND (outermost)
@decorator_b          # applied FIRST
def f(): ...
# equivalent to: f = decorator_a(decorator_b(f))
```

## Related

[[Python]] · [[Functions (Python)]] · [[Classes (Python)]]
