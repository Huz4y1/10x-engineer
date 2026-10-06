---
tags: [python, types, mypy]
---

# Type hints

Language: [[Python]] · Runtime validation: [[Pydantic]]

---

```python
def process(name: str, count: int = 0, ratio: float = 1.0) -> bool:
    ...

items: list[str] = []
mapping: dict[str, int] = {}
pairs: tuple[int, str] = (1, "a")
maybe: str | None = None                    # 3.10+ ; older: Optional[str]
either: int | str = 5                       # older: Union[int, str]
```

> **Type hints are not enforced at runtime.** Python ignores them. They exist for your editor, for `mypy`, and for humans. [[Pydantic]] and [[FastAPI reference|FastAPI]] are the exception — they read hints and *do* enforce them.

## Common annotations

```python
from typing import Any, Callable, Iterator, Iterable, Sequence, Literal, TypeVar, Protocol
from collections.abc import Callable, Iterator      # preferred in modern Python

def apply(fn: Callable[[int], str], xs: Iterable[int]) -> list[str]: ...
def gen() -> Iterator[int]: ...
Mode = Literal["fast", "accurate"]                  # only these two values
def run(mode: Mode) -> None: ...

T = TypeVar("T")
def first(xs: Sequence[T]) -> T | None:
    return xs[0] if xs else None
```

## Protocols — duck typing, checked

```python
from typing import Protocol

class Predictor(Protocol):
    def predict(self, x: list[float]) -> float: ...

def evaluate(model: Predictor) -> float:            # ANY object with .predict()
    return model.predict([1.0])
```

> **Protocols beat abstract base classes** when you don't control the other class. Nothing needs to inherit from anything.

## Annotated — hints plus metadata

```python
from typing import Annotated
from fastapi import Depends, Query

Weeks = Annotated[int, Query(ge=1, le=52)]
DB = Annotated[AsyncSession, Depends(get_db)]

async def forecast(weeks: Weeks, db: DB): ...       # reusable, readable
```

## mypy

```bash
uv add --dev mypy
uv run mypy src/
```

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.12"
ignore_missing_imports = true       # third-party libs without stubs
warn_return_any = true
warn_unused_ignores = true
# strict = true                     # turn on for new projects
```

```python
x = something()  # type: ignore[assignment]        # narrow escape hatch
```

> **`strict = true` from day one on a new project.** Retrofitting strict typing onto a large untyped codebase is genuinely painful.

## Forward references

```python
from __future__ import annotations       # all hints become strings - always safe

class Node:
    def add(self, child: Node) -> None:  # works without quotes
        ...
```

## Related

[[Python]] · [[Pydantic]] · [[FastAPI reference]] · [[Testing (Python)]]
