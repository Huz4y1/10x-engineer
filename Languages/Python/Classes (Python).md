---
tags: [python, language, oop]
---

# Classes (Python)

Language: [[Python]]

---

```python
class Sensor:
    kind = "generic"                          # class attribute - SHARED

    def __init__(self, device_id: str, unit: str = "C"):
        self.device_id = device_id            # instance attribute
        self.unit = unit
        self._readings: list[float] = []      # _ = internal by convention

    def add(self, value: float) -> None:
        self._readings.append(value)

    @property
    def mean(self) -> float:                  # accessed as .mean, no parens
        return sum(self._readings) / len(self._readings)

    @staticmethod
    def c_to_f(c: float) -> float:            # no self - a plain function
        return c * 9 / 5 + 32

    @classmethod
    def from_config(cls, cfg: dict) -> "Sensor":   # alternative constructor
        return cls(cfg["id"], cfg.get("unit", "C"))

    def __repr__(self) -> str:                # for developers
        return f"Sensor({self.device_id!r}, n={len(self._readings)})"
```

## Dunder methods

| Method | Enables |
|---|---|
| `__init__` | Construction |
| `__repr__` | `repr(x)` — **always define this** |
| `__str__` | `str(x)`, `print(x)` |
| `__eq__` / `__hash__` | `==`, use in sets/dict keys |
| `__lt__` etc. | Sorting and comparison |
| `__len__` | `len(x)` |
| `__getitem__` / `__setitem__` | `x[key]` |
| `__iter__` / `__next__` | `for x in obj` |
| `__contains__` | `x in obj` |
| `__call__` | `obj()` |
| `__enter__` / `__exit__` | `with obj:` |

> **Define `__repr__` on every class.** It's what you see in a debugger, a log line and a traceback. Without it you get `<Sensor object at 0x7f...>`, which tells you nothing.

## Dataclasses — use these

```python
from dataclasses import dataclass, field

@dataclass
class Reading:
    device_id: str
    value: float
    tags: list[str] = field(default_factory=list)    # NOT default=[]

@dataclass(frozen=True, slots=True)      # immutable + memory-efficient
class Point:
    x: float
    y: float
```

Gives you `__init__`, `__repr__` and `__eq__` free.

> **`@dataclass` for internal data, [[Pydantic]] `BaseModel` for anything crossing a trust boundary** — API input, config, external files. Dataclasses don't validate.

## Inheritance

```python
class TempSensor(Sensor):
    def __init__(self, device_id: str, calibration: float = 0.0):
        super().__init__(device_id, unit="C")
        self.calibration = calibration

    def add(self, value: float) -> None:      # override
        super().add(value + self.calibration)
```

```python
from abc import ABC, abstractmethod

class Model(ABC):
    @abstractmethod
    def predict(self, x): ...                 # subclasses MUST implement
```

> **Prefer composition over inheritance.** Deep hierarchies are hard to follow. An object holding another object is usually clearer than one inheriting from it.

## Protocols — structural typing

```python
from typing import Protocol

class Predictor(Protocol):
    def predict(self, x: list[float]) -> float: ...

def run(model: Predictor):        # anything with .predict() satisfies this
    return model.predict([1.0])
```

## Context managers

```python
import time

class Timer:
    def __enter__(self):
        self.start = time.perf_counter(); return self
    def __exit__(self, exc_type, exc, tb):
        print(f"{time.perf_counter() - self.start:.3f}s")

with Timer():
    slow_thing()
```

```python
import time
from contextlib import contextmanager

@contextmanager
def timer():
    start = time.perf_counter()
    try:
        yield
    finally:
        print(f"{time.perf_counter() - start:.3f}s")
```

## Related

[[Python]] · [[Type hints]] · [[Pydantic]] · [[Decorators]]
