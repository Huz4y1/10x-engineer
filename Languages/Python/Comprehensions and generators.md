---
tags: [python, language, idioms]
---

# Comprehensions and generators

Language: [[Python]]

---

## Comprehensions

```python
[x ** 2 for x in items]                          # list
[x for x in items if x > 0]                      # with filter
[x if x > 0 else 0 for x in items]               # with ternary (note position)
[(x, y) for x in a for y in b]                   # nested loops
{k: v for k, v in pairs}                         # dict
{x for x in items}                               # set
```

> **Filter goes at the end; ternary goes at the front.** `[x for x in xs if cond]` vs `[a if cond else b for x in xs]`. Mixing them up is the most common comprehension error.

**Readability limit:** one loop and one condition. Beyond that, write a loop — a comprehension nobody can read is worse than four lines.

## Generators — lazy, memory-efficient

```python
(x ** 2 for x in items)                          # generator expression
sum(x.price for x in items)                      # no intermediate list built
```

```python
def read_lines(path):
    with open(path, encoding="utf-8") as f:
        for line in f:
            yield line.rstrip("\n")              # produces one at a time

for line in read_lines("huge.txt"):              # constant memory
    process(line)
```

> **A generator holds one item in memory; a list holds all of them.** For a 10 GB file that's the difference between working and an `OutOfMemoryError`.

```python
def counter():
    n = 0
    while True:                                   # infinite is fine - it's lazy
        yield n
        n += 1

from itertools import islice
list(islice(counter(), 5))                        # [0,1,2,3,4]
```

## yield from

```python
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)              # delegate to a sub-generator
        else:
            yield item
```

## When to use which

| Use | When |
|---|---|
| **List comprehension** | You need the whole result, and it fits in memory |
| **Generator** | Streaming, huge data, or you only need the first few |
| **`for` loop** | The logic needs more than one condition, or has side effects |

> **Generators can only be consumed once.** After iterating, it's exhausted. If you need it twice, make it a list — or call the generator function again.

## Related

[[Python]] · [[Iteration (Python)]] · [[When to leave Python]]
