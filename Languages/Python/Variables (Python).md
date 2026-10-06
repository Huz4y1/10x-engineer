---
tags: [python, language, basics]
---

# Variables (Python)

Language: [[Python]]

---

## Assignment

```python
x = 5                    # no type declaration needed
x, y = 1, 2              # tuple unpacking
x, y = y, x              # swap - no temp variable
a = b = c = 0            # chained
first, *rest = [1,2,3,4] # star unpacking -> first=1, rest=[2,3,4]
*init, last = [1,2,3,4]  # init=[1,2,3], last=4
```

## Names are references, not boxes

```python
a = [1, 2, 3]
b = a                    # b points at the SAME list
b.append(4)
print(a)                 # [1, 2, 3, 4]  <- a changed too

b = a.copy()             # shallow copy - now independent
import copy
b = copy.deepcopy(a)     # nested structures too
```

> **This is the single most common Python surprise.** `=` never copies. For mutable objects (lists, dicts, sets, custom classes) both names point at the same thing.

## Mutable vs immutable

| Immutable | Mutable |
|---|---|
| `int`, `float`, `str`, `bool`, `tuple`, `frozenset`, `bytes` | `list`, `dict`, `set`, `bytearray`, most objects |

```python
def add_item(items=[]):          # ✗ THE CLASSIC BUG
    items.append(1)
    return items
add_item()  # [1]
add_item()  # [1, 1]  <- the default is created ONCE and shared

def add_item(items=None):        # ✓
    items = items if items is not None else []
    items.append(1)
    return items
```

> **Never use a mutable default argument.** The default object is created once, when the function is defined, and reused on every call. Same rule as `default_factory` in [[Pydantic]].

## Scope

```python
x = "global"

def f():
    x = "local"          # a NEW local variable, doesn't touch the global
def g():
    global x
    x = "changed"        # modifies the global
def outer():
    y = 1
    def inner():
        nonlocal y       # modifies the enclosing function's variable
        y = 2
```

**LEGB lookup order:** Local -> Enclosing -> Global -> Builtins.

## Constants and naming

```python
MAX_RETRIES = 3          # UPPER_SNAKE by convention - not enforced
_internal = "private"    # single underscore = "don't touch" convention
__mangled = "x"          # double underscore in a class = name mangling
snake_case               # variables and functions
PascalCase               # classes
```

## Walrus operator

```python
if (n := len(data)) > 10:
    print(f"{n} items")              # assign and test in one expression

while (line := f.readline()):
    process(line)
```

## Related

[[Python]] · [[Types (Python)]] · [[Functions (Python)]]
