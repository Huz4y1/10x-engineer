---
tags: [python, language, types]
---

# Types (Python)

Language: [[Python]] · Static typing: [[Type hints]]

---

## Built-in types

| Type | Example | Mutable |
|---|---|---|
| `int` | `42`, `1_000_000` | No |
| `float` | `3.14`, `1e-5` | No |
| `str` | `"text"` | No |
| `bool` | `True` | No |
| `bytes` | `b"raw"` | No |
| `list` | `[1, 2]` | **Yes** |
| `tuple` | `(1, 2)` | No |
| `dict` | `{"a": 1}` | **Yes** |
| `set` | `{1, 2}` | **Yes** |
| `frozenset` | `frozenset({1})` | No |
| `None` | `None` | — |

## Strings

```python
s = "hello"
s.upper() · s.lower() · s.title() · s.capitalize()
s.strip() · s.lstrip() · s.rstrip()
s.split(",") · ",".join(["a","b"])
s.replace("a", "b")
s.startswith("he") · s.endswith("lo")
s.find("l")          # -1 if absent
s.index("l")         # raises if absent
s.count("l")
s.zfill(5) · s.ljust(10) · s.rjust(10) · s.center(10)
s.isdigit() · s.isalpha() · s.isalnum()
s.encode("utf-8")    # -> bytes

f"{value:.2f}"       # 2 decimal places
f"{value:,}"         # thousands separator
f"{value:>10}"       # right-align in 10 chars
f"{value:%}"         # percentage
f"{name=}"           # debug: prints "name=value"
```

## Numbers

```python
7 / 2                # 3.5   true division
7 // 2               # 3     floor division
7 % 2                # 1     modulo
2 ** 10              # 1024  power
divmod(7, 2)         # (3, 1)
round(3.567, 2)      # 3.57
abs(-5) · min(a,b) · max(a,b) · sum(items)

from decimal import Decimal
Decimal("0.1") + Decimal("0.2") == Decimal("0.3")   # True
0.1 + 0.2 == 0.3                                    # False
```

> **Money uses `Decimal`, never `float`.** Floats can't represent 0.1 exactly. Same rule as `DECIMAL` in [[Data modeling]] and [[24 — SCIENTIFIC COMPUTING]].

## Lists

```python
xs = [3, 1, 2]
xs.append(4) · xs.extend([5,6]) · xs.insert(0, 0)
xs.remove(3)         # by VALUE, raises if absent
xs.pop() · xs.pop(0) # by INDEX, returns it
xs.sort() · xs.sort(key=len, reverse=True)   # in place
sorted(xs)           # returns a new list
xs.reverse() · xs.index(2) · xs.count(1) · xs.clear()

xs[0] · xs[-1] · xs[1:3] · xs[::2] · xs[::-1]        # slicing
```

> **`list.pop(0)` is O(n)** — it shifts every element. Use `collections.deque` for a queue ([[Algorithms and data structures reference]]).

## Dicts

```python
d = {"a": 1}
d["b"] = 2
d.get("c")               # None if absent - no exception
d.get("c", 0)            # with default
d.setdefault("c", 0)
d.keys() · d.values() · d.items()
d.pop("a") · d.update({"x": 9})
d | {"y": 10}            # merge (3.9+)
{**d, "y": 10}           # merge, older syntax

for k, v in d.items(): ...
```

## Sets

```python
s = {1, 2, 3}
s.add(4) · s.remove(1) · s.discard(9)     # discard doesn't raise
a | b   # union
a & b   # intersection
a - b   # difference
a ^ b   # symmetric difference
a <= b  # subset
```

> **`x in some_set` is O(1); `x in some_list` is O(n).** Converting a list to a set before repeated membership checks is the highest-value one-line optimisation in Python.

## Conversion and checking

```python
int("42") · float("3.14") · str(42) · list("abc") · dict(pairs) · set([1,1,2])

type(x) is int
isinstance(x, int)                # ✅ use this - respects inheritance
isinstance(x, (int, float))
x is None                         # ✅ identity, not ==
```

## Useful collections

```python
from collections import defaultdict, Counter, deque, namedtuple

defaultdict(list)                 # no KeyError on first access
Counter(items).most_common(3)     # frequency counts
deque(maxlen=100)                 # O(1) at both ends
```

## Related

[[Python]] · [[Type hints]] · [[Variables (Python)]] · [[Algorithms and data structures reference]]
