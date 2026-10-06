---
tags: [cs, algorithms, datastructures, reference, cheatsheet]
---

# Algorithms and data structures reference

Complexity, code and when to use each. Section: [[04 — COMPUTER SCIENCE]]

---

## Big O in one line

**How does the time grow as the input grows?**

| Notation | Name | n=10 | n=1,000 | n=1,000,000 |
|---|---|---|---|---|
| $O(1)$ | Constant | 1 | 1 | 1 |
| $O(\log n)$ | Logarithmic | 3 | 10 | 20 |
| $O(n)$ | Linear | 10 | 1,000 | 1,000,000 |
| $O(n \log n)$ | Linearithmic | 33 | 10,000 | 20,000,000 |
| $O(n^2)$ | Quadratic | 100 | 1,000,000 | $10^{12}$ ❌ |
| $O(2^n)$ | Exponential | 1,024 | ❌ | ❌ |

> **The practical line is at $O(n^2)$.** A million items at $O(n)$ is instant. At $O(n^2)$ it's a trillion operations — hours. This is why [[When to leave Python]] says algorithms beat languages: fixing $O(n^2) \to O(n)$ is a 10,000× win; rewriting in Rust is 50×.

---

## Data structures

| Structure | Access | Search | Insert | Delete | Use when |
|---|---|---|---|---|---|
| **Array / list** | $O(1)$ | $O(n)$ | $O(n)$ | $O(n)$ | Indexed access, iteration |
| **Hash table (dict/set)** | — | $O(1)$* | $O(1)$* | $O(1)$* | **Lookups by key. The default.** |
| Linked list | $O(n)$ | $O(n)$ | $O(1)$† | $O(1)$† | Frequent insert/delete at known position |
| **Stack** | $O(1)$ top | — | $O(1)$ | $O(1)$ | LIFO — undo, DFS, call stack |
| **Queue** | $O(1)$ front | — | $O(1)$ | $O(1)$ | FIFO — BFS, job queues |
| **Heap** | $O(1)$ min | — | $O(\log n)$ | $O(\log n)$ | "Give me the smallest" repeatedly |
| **BST (balanced)** | $O(\log n)$ | $O(\log n)$ | $O(\log n)$ | $O(\log n)$ | Sorted order + fast lookup |
| Trie | — | $O(k)$ | $O(k)$ | $O(k)$ | Prefix search, autocomplete |
| Graph | — | varies | $O(1)$ | $O(1)$ | Relationships, networks |

\* average; $O(n)$ worst case with collisions   † once you have the node

### The one that matters most

```python
# O(n) per lookup -> O(n*m) total
if item in my_list:      # scans the whole list

# O(1) per lookup -> O(n) total
lookup = set(my_list)    # build once
if item in lookup:       # instant
```

> **Converting a list to a set before repeated membership checks is the single highest-value micro-optimisation in Python.** It turns quadratic into linear, in one line.

### Python built-ins

```python
from collections import deque, Counter, defaultdict
import heapq

deque()                  # O(1) append/pop at BOTH ends. Use for queues.
Counter(items)           # frequency counts
defaultdict(list)        # no KeyError on first access
heapq.heappush(h, x)     # min-heap
heapq.heappop(h)         # smallest
heapq.nlargest(5, data)  # top 5 without full sort
```

> `list.pop(0)` is $O(n)$ — it shifts everything. **`deque.popleft()` is $O(1)$.** Use `deque` for queues, always.

---

## Sorting

| Algorithm | Best | Average | Worst | Space | Stable |
|---|---|---|---|---|---|
| **Timsort** (Python's) | $O(n)$ | $O(n\log n)$ | $O(n\log n)$ | $O(n)$ | ✅ |
| Quicksort | $O(n\log n)$ | $O(n\log n)$ | $O(n^2)$ | $O(\log n)$ | ❌ |
| Mergesort | $O(n\log n)$ | $O(n\log n)$ | $O(n\log n)$ | $O(n)$ | ✅ |
| Heapsort | $O(n\log n)$ | $O(n\log n)$ | $O(n\log n)$ | $O(1)$ | ❌ |
| Bubble | $O(n)$ | $O(n^2)$ | $O(n^2)$ | $O(1)$ | ✅ |

> **Use `sorted()`.** Python's Timsort is adaptive — nearly-sorted data sorts in near-linear time. You will essentially never beat it by hand.
> **"Stable"** means equal elements keep their original order — which matters when sorting by multiple keys in sequence.

---

## Searching

```python
# Binary search - O(log n), requires SORTED input
import bisect
i = bisect.bisect_left(sorted_list, target)
found = i < len(sorted_list) and sorted_list[i] == target
```

> Binary search on a million items takes **20 comparisons**. Linear search takes 500,000 on average. But it only works on sorted data — and sorting costs $O(n\log n)$, so sort once if you'll search many times.

---

## Graph algorithms

```python
graph = {"A": ["B", "C"], "B": ["D"], "C": ["D"], "D": []}
```

### BFS — shortest path in an *unweighted* graph

```python
from collections import deque

def bfs(graph, start, goal):
    queue = deque([(start, [start])])
    seen = {start}
    while queue:
        node, path = queue.popleft()          # QUEUE -> breadth first
        if node == goal:
            return path
        for nxt in graph[node]:
            if nxt not in seen:
                seen.add(nxt)
                queue.append((nxt, path + [nxt]))
    return None
```

### DFS — explore deeply, cycle detection, topological sort

```python
def dfs(graph, node, seen=None):
    seen = seen or set()
    seen.add(node)
    for nxt in graph[node]:
        if nxt not in seen:
            dfs(graph, nxt, seen)             # STACK (recursion) -> depth first
    return seen
```

> **BFS uses a queue, DFS uses a stack. That single difference is the whole distinction.** BFS finds the shortest path; DFS goes deep and is better for exhaustive exploration.

### Dijkstra — shortest path with weights

```python
import heapq

def dijkstra(graph, start):
    dist = {start: 0}
    pq = [(0, start)]
    while pq:
        d, node = heapq.heappop(pq)           # always expand the closest
        if d > dist.get(node, float("inf")):
            continue
        for nxt, weight in graph[node]:
            nd = d + weight
            if nd < dist.get(nxt, float("inf")):
                dist[nxt] = nd
                heapq.heappush(pq, (nd, nxt))
    return dist
```

Complexity: $O((V+E)\log V)$. **Requires non-negative weights.**

### A* — Dijkstra with a hint

Same as Dijkstra, but the priority is `cost_so_far + heuristic(node, goal)`.

With an **admissible** heuristic (never overestimates — e.g. straight-line distance), A* is guaranteed optimal and explores far less of the graph.

> **A* is what a robot uses to plan a path** across an occupancy grid ([[21 — ROBOTICS]], [[Project 009 — Autonomous Robotics System]]).

| Algorithm | Use |
|---|---|
| BFS | Shortest path, unweighted |
| DFS | Cycles, topological sort, exhaustive search |
| Dijkstra | Shortest path, weighted, non-negative |
| **A\*** | Shortest path with a good heuristic — pathfinding |
| Bellman-Ford | Weighted, **negative** edges allowed |
| Union-Find | Connected components, Kruskal's MST |

---

## Recursion and dynamic programming

**Recursion** — a function calling itself, with a base case.

```python
def fib(n):
    if n <= 1: return n
    return fib(n-1) + fib(n-2)      # O(2^n) - recomputes constantly
```

**Memoisation** — cache results:

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    if n <= 1: return n
    return fib(n-1) + fib(n-2)      # O(n) - one line changed
```

> **`@lru_cache` turns exponential into linear for one decorator.** It's the laziest possible dynamic programming, and it works whenever the function is pure.

**Bottom-up DP** — build the table iteratively, no recursion depth limit:

```python
def fib(n):
    a, b = 0, 1
    for _ in range(n): a, b = b, a + b
    return a                        # O(n) time, O(1) space
```

> Python's default recursion limit is 1000. Deep recursion needs the iterative form.

---

## Where these appear in this vault

| Concept | Appears in |
|---|---|
| Hash tables | Every dict; Spark shuffle partitioning ([[PySpark core]]) |
| Heaps | Top-N queries; Dijkstra; scheduling |
| B-trees | **Database indexes** ([[SQL fundamentals]]) |
| Graphs + topological sort | DAG orchestration ([[Unity Catalog and orchestration]]) |
| A* | Robot path planning ([[21 — ROBOTICS]]) |
| Big O | Why $O(n^2)$ joins explode ([[SQL fundamentals]]) |
| Sorting | `ORDER BY`, Spark shuffles |

## Related

[[04 — COMPUTER SCIENCE]] · [[Networking reference]] · [[When to leave Python]] · [[03 — PROGRAMMING]]
