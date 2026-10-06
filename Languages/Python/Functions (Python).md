---
tags: [python, language, functions]
---

# Functions (Python)

Language: [[Python]]

---

```python
def greet(name: str, greeting: str = "Hello") -> str:
    """Return a greeting.

    Args:
        name: Who to greet.
        greeting: The greeting word.
    """
    return f"{greeting}, {name}"
```

## Arguments

```python
def f(a, b=1, *args, c, d=2, **kwargs): ...
#     |  |      |      |   |      |
#     |  |      |      |   |      +-- extra keyword args as a dict
#     |  |      |      |   +--------- keyword-only with default
#     |  |      |      +------------- keyword-only, REQUIRED
#     |  |      +-------------------- extra positional args as a tuple
#     |  +--------------------------- positional with default
#     +------------------------------ positional

def f(a, b, /, c, *, d): ...
#            ^        ^
#            |        +-- everything after * is keyword-only
#            +----------- everything before / is positional-only
```

```python
f(*my_list)              # unpack a list into positional args
f(**my_dict)             # unpack a dict into keyword args
```

> **Force keyword-only arguments with `*` for anything ambiguous.** `resize(img, 100, 200)` is unreadable; `resize(img, width=100, height=200)` is not.

## Return values

```python
def f(): return 1, 2, 3           # returns a tuple
a, b, c = f()

def f(): pass                     # returns None implicitly
```

## Lambdas

```python
square = lambda x: x ** 2         # avoid - use def
sorted(items, key=lambda x: x.age)   # ✅ fine inline
```

## Closures and first-class functions

```python
def multiplier(n):
    def inner(x):
        return x * n              # captures n
    return inner

double = multiplier(2)
double(5)                         # 10
```

## Useful patterns

```python
from functools import lru_cache, partial, reduce, wraps

@lru_cache(maxsize=None)          # memoise - turns exponential into linear
def fib(n): ...

add_five = partial(add, 5)        # pre-fill an argument
reduce(lambda a, b: a + b, items)
```

> `@lru_cache` is the laziest possible dynamic programming and works on any **pure** function ([[Algorithms and data structures reference]]).

## Related

[[Python]] · [[Decorators]] · [[Type hints]] · [[Comprehensions and generators]]
