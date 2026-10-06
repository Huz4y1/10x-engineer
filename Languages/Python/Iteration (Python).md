---
tags: [python, language, basics]
---

# Iteration (Python)

Language: [[Python]]

---

```python
for item in items: ...
for i in range(10): ...
for i in range(2, 10, 2): ...
for i, item in enumerate(items): ...
for i, item in enumerate(items, start=1): ...
for a, b in zip(list_a, list_b): ...
for k, v in d.items(): ...

while condition: ...
```

## Control

```python
break            # exit the loop
continue         # next iteration
else:            # runs if the loop finished WITHOUT break
    ...
```

```python
for x in items:
    if x == target:
        break
else:
    print("not found")        # only runs if no break
```

## Idioms

```python
zip(a, b)                         # stops at the shorter
zip(a, b, strict=True)            # 3.10+ - raises if lengths differ
zip(*matrix)                      # transpose
reversed(items)
sorted(items, key=lambda x: x.age, reverse=True)
enumerate(items)
any(x > 0 for x in items)
all(x > 0 for x in items)
sum(x.price for x in items)
min(items, key=len) · max(items, key=len)

from itertools import chain, groupby, product, combinations, islice, accumulate
chain(a, b)                       # flatten iterables
product(a, b)                     # cartesian product
combinations(items, 2)
islice(gen, 10)                   # first 10 of a generator
accumulate([1,2,3])               # running total
```

## Don't mutate while iterating

```python
for x in items:
    if bad(x):
        items.remove(x)           # ✗ skips elements

items = [x for x in items if not bad(x)]     # ✓
```

## Related

[[Python]] · [[Comprehensions and generators]] · [[Selection (Python)]]
