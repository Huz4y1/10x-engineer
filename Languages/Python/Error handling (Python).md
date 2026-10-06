---
tags: [python, language, errors]
---

# Error handling (Python)

Language: [[Python]]

---

```python
try:
    result = risky()
except ValueError as e:
    logger.warning("bad value: %s", e)
except (KeyError, IndexError) as e:
    handle(e)
except Exception as e:
    logger.exception("unexpected")      # logs the FULL traceback
    raise                               # re-raise
else:
    print("no exception")               # runs only if try succeeded
finally:
    cleanup()                           # ALWAYS runs
```

## The rules that matter

```python
except Exception:        # ✅ catches errors, not KeyboardInterrupt/SystemExit
except:                  # ✗ bare except catches Ctrl+C too - never do this
    pass                 # ✗✗ silently swallowing errors is how bugs hide
```

> **Catch the most specific exception you can.** `except Exception` around a whole function hides the one bug you needed to see.

> **`logger.exception()` inside an `except` block logs the full traceback automatically.** Use it instead of `logger.error(str(e))`, which throws away the stack.

## Raising

```python
raise ValueError(f"weeks must be 1-52, got {weeks}")
raise RuntimeError("failed") from original_error      # preserves the cause
```

## Custom exceptions

```python
class PipelineError(Exception):
    """Base for this package."""

class DataQualityError(PipelineError):
    def __init__(self, table: str, rows: int):
        self.table, self.rows = table, rows
        super().__init__(f"{table} has only {rows} rows")
```

> **Give a package one base exception.** Callers can then `except PipelineError` and catch everything you raise without catching everything else.

## Common built-ins

| Exception | Raised when |
|---|---|
| `ValueError` | Right type, wrong value |
| `TypeError` | Wrong type |
| `KeyError` / `IndexError` | Missing key / index out of range |
| `AttributeError` | No such attribute |
| `FileNotFoundError` | Missing file |
| `ZeroDivisionError` | Division by zero |
| `TimeoutError` | Timed out |
| `NotImplementedError` | Abstract method not overridden |

## Assertions

```python
assert n > 0, f"expected positive, got {n}"
```

> ⚠️ **`assert` is removed when Python runs with `-O`.** Never use it for validation of external input or security checks — use `if ... raise`. It's fine for internal invariants and tests.

## Retrying

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, max=10))
def call_api(): ...
```

> **Retry transient failures only** — network blips, rate limits. A bug fails identically three times and just wastes time ([[CI-CD pipelines]]).

## Warnings

```python
import warnings
warnings.warn("deprecated, use new_fn", DeprecationWarning, stacklevel=2)
```

## Related

[[Python]] · [[Testing (Python)]] · [[FastAPI reference]] · [[_Troubleshooting template]]
