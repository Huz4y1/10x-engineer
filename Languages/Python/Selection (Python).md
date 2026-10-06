---
tags: [python, language, basics]
---

# Selection (Python)

Language: [[Python]]

---

```python
if x > 10:
    ...
elif x > 5:
    ...
else:
    ...

value = "big" if x > 10 else "small"          # ternary
```

## Truthiness

**Falsy:** `False`, `None`, `0`, `0.0`, `""`, `[]`, `{}`, `set()`, `()`

Everything else is truthy.

```python
if items:            # ✅ pythonic - "if the list is non-empty"
if len(items) > 0:   # works, more verbose

if x is None:        # ✅ for None specifically
if not x:            # ✗ also true for 0, "", []
```

> **`if not x` and `if x is None` are different.** `0`, `""` and `[]` are all falsy but not `None`. When a function can legitimately return `0`, this distinction is a real bug source.

## Comparison and boolean operators

```python
a == b        # value equality
a is b        # SAME OBJECT - use only for None, True, False
a != b · a < b · a <= b
1 < x < 10                        # chaining works
a and b · a or b · not a

x = value or "default"            # short-circuit default
x = a and a.method()              # guard against None
```

## match (3.10+)

```python
match command.split():
    case ["go", direction]:
        move(direction)
    case ["drop", *items]:
        drop(items)
    case {"action": "quit"}:       # dict pattern
        quit()
    case Point(x=0, y=0):          # class pattern
        origin()
    case _:
        unknown()
```

> `match` is structural pattern matching, not a C-style switch — it destructures. For simple value dispatch a dict is usually clearer.

## Related

[[Python]] · [[Iteration (Python)]] · [[Types (Python)]]
