---
tags: [python, library, http, requests, httpx]
---

# Requests and httpx

Calling other people's APIs. Index: [[Python libraries]] · Building APIs: [[FastAPI reference]]

### Installing

> **Same commands on Windows, WSL and Linux.** The differences are called out below where they exist.

```bash
uv add httpx                      # in a uv project - PREFERRED
uv pip install httpx              # into the active venv, no project file
pip install httpx                 # plain pip, if you're not using uv
```

**If you don't have a project yet:**

```bash
uv init myproject && cd myproject     # creates pyproject.toml and .venv
uv add httpx
uv run python -c "import httpx; print(httpx.__version__)"           # uv run uses the project venv automatically
```

**Without uv — a plain virtual environment:**

| | Windows (PowerShell) | WSL / Linux |
|---|---|---|
| Create | `python -m venv .venv` | `python3 -m venv .venv` |
| Activate | `.venv\Scripts\Activate.ps1` | `source .venv/bin/activate` |
| Install | `pip install httpx` | `pip install httpx` |
| Leave | `deactivate` | `deactivate` |

**Check it worked:**

```bash
python -c "import httpx; print(httpx.__version__)"
```

```bash
uv add httpx        # modern, sync AND async - the default choice
uv add requests     # the classic, sync only
uv add "httpx[http2]"   # if you need HTTP/2
```

> ⚠️ **Corporate networks and SSL:** behind a proxy that inspects TLS you'll get `SSLCertVerificationError`. The fix is to point at the corporate CA bundle, **not** to disable verification:
> ```bash
> export SSL_CERT_FILE=/path/to/corp-ca.pem      # WSL/Linux
> $env:SSL_CERT_FILE="C:\path\corp-ca.pem"       # Windows PowerShell
> ```
> ⚠️ **Never ship `verify=False`.** It disables certificate checking entirely and makes the connection trivially interceptable ([[Security in practice]]).

> ⚠️ **Never `pip install` outside a virtual environment.** On WSL and Linux it fights the system Python; newer versions refuse outright with *externally-managed-environment*. On Windows it silently pollutes your global install. `uv` makes a venv for you — use it.

> **On Windows: if `python` opens the Microsoft Store**, Python isn't really installed. Get it from python.org (tick *Add to PATH*), or `winget install Python.Python.3.12`.

> **In WSL: work in `~/`, not `/mnt/c/`.** Installing into a project on the Windows drive is dramatically slower, and file-watching (`--reload`, `pytest-watch`) often doesn't fire at all. See [[Setting up a dev machine]].

---

## Which one

| | requests | httpx |
|---|---|---|
| Sync | ✅ | ✅ |
| **Async** | ❌ | ✅ |
| HTTP/2 | ❌ | ✅ |
| API | The original | **Almost identical to requests** |
| Testing FastAPI | — | ✅ Required for async tests |

> **Use httpx for new code.** The API is deliberately the same as requests, and you get async when you need it. `requests` is fine and everywhere, but it can never be awaited.

## Basics — identical in both

```python
import httpx
import httpx   # or: import requests as httpx

r = httpx.get("https://api.example.com/items")
r = httpx.get(url, params={"category": "Kitchen", "weeks": 4})
r = httpx.post(url, json={"a": 1})                # sets Content-Type automatically
r = httpx.post(url, data={"form": "field"})        # form-encoded
r = httpx.post(url, content=b"raw bytes")
r = httpx.put(url, json=payload)
r = httpx.delete(url)

r.status_code · r.text · r.json() · r.content · r.headers
r.raise_for_status()                               # raise on 4xx/5xx
r.ok                                               # requests only
```

> ⚠️ **`r.json()` does not raise on a 500.** A failed request often returns an HTML error page, and `.json()` then throws a confusing parse error. **Always `raise_for_status()` first.**

## Headers, auth, timeouts

```python
import httpx

r = httpx.get(url,
    headers={"Authorization": f"Bearer {token}", "Accept": "application/json"},
    timeout=10.0,                                  # <- NEVER omit this
)

httpx.get(url, auth=("user", "pass"))              # basic auth
httpx.get(url, timeout=httpx.Timeout(10.0, connect=5.0))
```

> ⚠️ **Always set a timeout.** Without one, a hung server hangs *your* process forever. `requests` has **no default timeout at all** — this is the single most common production bug with it.

## Clients — reuse the connection

```python
import httpx

with httpx.Client(
    base_url="https://api.example.com",
    headers={"Authorization": f"Bearer {token}"},
    timeout=10.0,
) as client:
    a = client.get("/items")                       # connection reused
    b = client.get("/customers")
```

> **A `Client` reuses the TCP+TLS connection.** Calling `httpx.get()` in a loop performs a full handshake every time — 50-200 ms each ([[Networking reference]]). For repeated calls this is the biggest single speedup available.

## Async

```python
import asyncio, httpx

async def fetch_all(urls):
    async with httpx.AsyncClient(timeout=10.0) as client:
        tasks = [client.get(u) for u in urls]
        responses = await asyncio.gather(*tasks)    # CONCURRENT
        return [r.json() for r in responses]
```

> **This is the reason to use httpx.** Ten sequential requests at 200 ms each take 2 seconds; concurrently they take 200 ms ([[Async (Python)]]).

```python
import asyncio

sem = asyncio.Semaphore(10)                        # don't hammer the server
async def limited(client, url):
    async with sem:
        return await client.get(url)
```

## Error handling

```python
import httpx

try:
    r = httpx.get(url, timeout=10.0)
    r.raise_for_status()
    data = r.json()
except httpx.TimeoutException:
    logger.warning("timed out")
except httpx.HTTPStatusError as e:
    logger.error("HTTP %s: %s", e.response.status_code, e.response.text[:200])
except httpx.RequestError as e:
    logger.error("connection failed: %s", e)
```

| requests | httpx |
|---|---|
| `requests.Timeout` | `httpx.TimeoutException` |
| `requests.HTTPError` | `httpx.HTTPStatusError` |
| `requests.ConnectionError` | `httpx.ConnectError` |
| `requests.RequestException` | `httpx.RequestError` |

## Retries

```python
import httpx

transport = httpx.HTTPTransport(retries=3)         # connection errors only
client = httpx.Client(transport=transport)
```

```python
import httpx
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

@retry(stop=stop_after_attempt(3),
       wait=wait_exponential(multiplier=1, max=10),
       retry=retry_if_exception_type((httpx.TimeoutException, httpx.ConnectError)))
def fetch(url):
    r = httpx.get(url, timeout=10.0)
    r.raise_for_status()
    return r.json()
```

> **Retry transient failures only** — timeouts, connection errors, 429, 503. Retrying a 400 or 404 just wastes time; the answer won't change ([[CI-CD pipelines]]).

## Streaming large responses

```python
import httpx

with httpx.stream("GET", url) as r:
    r.raise_for_status()
    with open("big.csv", "wb") as f:
        for chunk in r.iter_bytes(chunk_size=8192):
            f.write(chunk)
```

> **Don't `r.content` a 5 GB download** — it loads the whole thing into memory.

## Files and multipart

```python
import httpx

with open("data.csv", "rb") as f:
    r = httpx.post(url, files={"file": ("data.csv", f, "text/csv")})
```

## Testing FastAPI

```python
import pytest
from httpx import AsyncClient, ASGITransport

@pytest.mark.asyncio
async def test_predict():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://t") as c:
        r = await c.post("/predict", json={"id": "1"})
    assert r.status_code == 200
```

> **`ASGITransport` calls the app in-process** — no server, no network, no port. Fast and reliable ([[pytest]]).

## In Streamlit

```python
import httpx
import streamlit as st

@st.cache_data(ttl=300)                            # <- or every interaction re-fetches
def api_get(path: str, **params):
    r = httpx.get(f"{API_URL}{path}", params=params, timeout=15.0)
    r.raise_for_status()
    return r.json()
```

## Checklist for any API call

- [ ] **Timeout set**
- [ ] `raise_for_status()` before `.json()`
- [ ] `Client` reused for repeated calls
- [ ] Specific exceptions caught, most-specific first
- [ ] Retries on transient failures only
- [ ] Secrets from config, never hardcoded
- [ ] Response cached where sensible

## Related

[[Python libraries]] · [[FastAPI reference]] · [[Async (Python)]] · [[Networking reference]] · [[Streamlit]] · [[Error handling (Python)]]
