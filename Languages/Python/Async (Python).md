---
tags: [python, async, concurrency]
---

# Async (Python)

Language: [[Python]] · In practice: [[FastAPI reference]]

---

## The idea

Normal Python does one thing at a time. Waiting 50 ms on a database call means doing **nothing** for 50 ms.

`async` lets the process go and do something else during that wait.

```python
import httpx
import asyncio

async def fetch(url: str) -> str:
    async with httpx.AsyncClient() as client:
        r = await client.get(url)          # <- during this wait, others run
        return r.text

async def main():
    results = await asyncio.gather(        # concurrent, not sequential
        fetch(a), fetch(b), fetch(c)
    )

asyncio.run(main())
```

## Async helps I/O, not CPU

| Workload | Use |
|---|---|
| **Waiting** — network, database, disk | **async** |
| **Computing** — maths, parsing, model inference | **processes** (`ProcessPoolExecutor`) |

> **Async does not make CPU work faster.** It only helps when you're *waiting*. For CPU-bound work the GIL means threads don't help either — use processes ([[When to leave Python]]).

## The rules

```python
import asyncio
import time

await something()                # only inside `async def`
asyncio.run(main())              # entry point from sync code

async def f():
    time.sleep(1)                # ✗ BLOCKS the entire event loop
    await asyncio.sleep(1)       # ✓
```

> **One blocking call inside `async def` freezes everything.** This is the number one async bug, and it works perfectly with one user ([[FastAPI reference]] §9).

```python
from asyncio import to_thread
result = await to_thread(blocking_function, arg)   # push it to a thread
```

## Patterns

```python
import asyncio

await asyncio.gather(*tasks)                       # all, concurrently
await asyncio.gather(*tasks, return_exceptions=True)   # don't fail fast

async with asyncio.TaskGroup() as tg:              # 3.11+ - preferred
    t1 = tg.create_task(fetch(a))
    t2 = tg.create_task(fetch(b))

await asyncio.wait_for(fetch(url), timeout=5.0)

sem = asyncio.Semaphore(10)                        # limit concurrency
async def limited(url):
    async with sem:
        return await fetch(url)
```

## Async iteration and context managers

```python
import asyncio

async for row in cursor: ...

async with SessionLocal() as session: ...

async def gen():
    for i in range(10):
        await asyncio.sleep(0.1)
        yield i
```

## Async libraries

| Sync | Async |
|---|---|
| `requests` | `httpx` ([[Requests and httpx]]) |
| `psycopg` | `asyncpg`, `psycopg` async |
| `SQLAlchemy` | `create_async_engine` ([[SQLAlchemy]]) |
| `time.sleep` | `asyncio.sleep` |
| `open` | `aiofiles` |

> **Don't mix sync and async drivers.** A sync `psycopg` call inside an async endpoint blocks the loop — worse than being fully synchronous.

## Related

[[Python]] · [[FastAPI reference]] · [[SQLAlchemy]] · [[Requests and httpx]]
