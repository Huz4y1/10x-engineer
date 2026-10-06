---
tags: [fastapi, api, python]
status: not-started
---

# FastAPI Fundamentals

> **What this is:** a Python framework for building web APIs — programs that answer questions over HTTP.
> **Why you care:** it's the one door everything downstream goes through. Dashboards don't query the lake or load the model; they ask the API.

---

## The idea in plain English

An **API** is a waiter.

You (the dashboard) don't walk into the kitchen and cook. You tell the waiter what you want. The waiter goes to the kitchen (the database, the model), gets it, and brings it back in a standard format.

**Why not let the dashboard go into the kitchen?** Because then every dashboard needs the database password, every dashboard has to know the table layout, and changing the schema breaks all of them at once. One waiter, one menu, one place to change things.

Concretely, your Streamlit dashboard does this:

```python
import requests

requests.get("https://retail-api.azure.com/sales/forecast?category=Kitchen&weeks=4").json()
```

Not this:

```python
pyodbc.connect("Server=retail-sql-01...;Password=...")   # ✗ password in the dashboard
onnxruntime.InferenceSession("model.onnx")               # ✗ model loaded in the dashboard
```

---

## The HTTP vocabulary you need

An API call is a **request** and a **response**.

**The request has:**

| Part | Example | Purpose |
|---|---|---|
| Method | `GET`, `POST`, `PUT`, `DELETE` | What kind of action |
| Path | `/customers/17850/segment` | Which resource |
| Query string | `?category=Kitchen&weeks=4` | Options/filters |
| Headers | `Authorization: Bearer abc123` | Metadata, auth |
| Body | `{"features": [1, 2, 3]}` | Data being sent (POST/PUT only) |

**Methods, and when to use which:**

| Method | Meaning | Has a body? |
|---|---|---|
| `GET` | Read something. Changes nothing. | No |
| `POST` | Create something, or run an action | Yes |
| `PUT` | Replace something entirely | Yes |
| `PATCH` | Update part of something | Yes |
| `DELETE` | Remove something | No |

> **Rule of thumb:** `GET` must be safe to repeat. A `GET` that changes data is a bug — browsers, proxies and caches all assume `GET` is harmless and may repeat it without asking.

**Status codes** — the response's headline:

| Code | Means | Typical cause |
|---|---|---|
| `200 OK` | Worked | — |
| `201 Created` | Made something | Successful POST |
| `204 No Content` | Worked, nothing to return | Successful DELETE |
| `400 Bad Request` | **Your** input was malformed | — |
| `401 Unauthorized` | You didn't say who you are | Missing/invalid token |
| `403 Forbidden` | You said who you are; you're not allowed | Permissions |
| `404 Not Found` | That thing doesn't exist | — |
| `422 Unprocessable Entity` | Input was well-formed but failed validation | **FastAPI's Pydantic errors** |
| `429 Too Many Requests` | Rate limited | — |
| `500 Internal Server Error` | **Your code crashed** | Unhandled exception |
| `503 Service Unavailable` | Server temporarily down | Startup, overload |

The 4xx/5xx split matters: **4xx is the caller's fault, 5xx is yours.** If you're returning 500s for bad input, you're blaming the user for your missing validation.

---

## 1. Hello world

```python
# api/main.py
from fastapi import FastAPI

app = FastAPI(title="Retail Analytics API", version="1.0.0")

@app.get("/health")
def health():
    return {"status": "ok"}
```

```bash
uvicorn api.main:app --reload
```

`api.main:app` means "in the module `api.main`, the object called `app`". `--reload` restarts on file changes — **development only**, it's slow and unsafe in production.

Now open **http://localhost:8000/docs**. There's a full interactive documentation page, generated from your code, with a working "Try it out" button. You wrote no documentation. This is FastAPI's headline feature and it's genuinely why people pick it.

---

## 2. Parameters — the three kinds

### Path parameters — part of the URL, identify a thing

```python
@app.get("/customers/{customer_id}/segment")
def get_segment(customer_id: int):
    return {"customer_id": customer_id, "segment": "high_value"}
```

`GET /customers/17850/segment` → `customer_id = 17850`, **as an `int`**.

That type hint isn't decoration. FastAPI reads it and:
1. Converts `"17850"` (URLs are always strings) into the integer `17850`
2. Returns `422` automatically if someone sends `/customers/banana/segment`
3. Documents it in `/docs`

### Query parameters — after the `?`, options and filters

Any function argument that isn't in the path becomes a query parameter:

```python
from fastapi import Query

@app.get("/sales/forecast")
def forecast(
    category: str,                                    # required
    weeks: int = 4,                                   # optional, defaults to 4
    include_history: bool = False,
):
    return {"category": category, "weeks": weeks}
```

`GET /sales/forecast?category=Kitchen&weeks=8`

**With constraints** — this is where it gets good:

```python
from fastapi import Query

@app.get("/sales/forecast")
def forecast(
    category: str = Query(..., min_length=1, max_length=50, description="Product category"),
    weeks:    int = Query(4, ge=1, le=52, description="Weeks to forecast"),
):
    ...
```

`...` (Ellipsis) means **required**. `ge`/`le` are ≥ and ≤. Send `weeks=1000` and you get a `422` with a clear message — **and you wrote no validation code.**

> This is the core pitch of FastAPI: the type hints you'd write anyway for readability become your validation, your parsing, and your documentation. One thing, three jobs.

### Request bodies — structured data, for POST/PUT

```python
from pydantic import BaseModel, Field

class PredictionRequest(BaseModel):
    customer_id: str = Field(..., min_length=1)
    recency_days: int = Field(..., ge=0, le=3650)
    frequency: int = Field(..., ge=1)
    monetary_value: float = Field(..., gt=0)

@app.post("/predict/segment")
def predict_segment(request: PredictionRequest):
    return {"customer_id": request.customer_id, "segment": "..."}
```

Because `PredictionRequest` is a Pydantic model, FastAPI knows it comes from the JSON body.

---

## 3. Pydantic — validation without validation code

### What you'd write without it

```python
def predict(data):
    if "customer_id" not in data:
        return {"error": "customer_id required"}, 400
    if not isinstance(data["customer_id"], str):
        return {"error": "customer_id must be a string"}, 400
    if "recency_days" not in data:
        return {"error": "recency_days required"}, 400
    try:
        recency = int(data["recency_days"])
    except ValueError:
        return {"error": "recency_days must be an integer"}, 400
    if recency < 0:
        return {"error": "recency_days must be >= 0"}, 400
    # ...twenty more lines, for four fields
```

Every API without Pydantic has that code. It's tedious, it's inconsistent between endpoints, and someone always forgets a check.

### What you write with it

```python
from pydantic import BaseModel, Field

class PredictionRequest(BaseModel):
    customer_id: str = Field(..., min_length=1)
    recency_days: int = Field(..., ge=0, le=3650)
```

Done. Missing field, wrong type, out of range — all handled, with a precise error message naming the exact field:

```json
{
  "detail": [{
    "type": "greater_than_equal",
    "loc": ["body", "recency_days"],
    "msg": "Input should be greater than or equal to 0",
    "input": -5
  }]
}
```

### Response models — the other half

```python
from datetime import datetime
from pydantic import BaseModel

class ForecastPoint(BaseModel):
    week_starting: date
    predicted_revenue: float
    lower_bound: float
    upper_bound: float

class ForecastResponse(BaseModel):
    category: str
    generated_at: datetime
    model_version: str
    forecast: list[ForecastPoint]

@app.get("/sales/forecast", response_model=ForecastResponse)
def forecast(category: str, weeks: int = 4) -> ForecastResponse:
    ...
```

`response_model` does three things:

1. **Documents** the exact response shape in `/docs`
2. **Validates** your own output — if you accidentally return a `None` where a float belongs, you find out immediately instead of the dashboard breaking
3. **Filters** — any field not in the model is stripped from the response

Point 3 is a security feature. Return a database row containing an internal cost column, and the response model silently drops it. Accidental data leaks through over-fetching are common; this prevents them structurally.

### Custom validation

```python
from pydantic import field_validator, model_validator, BaseModel

class ForecastRequest(BaseModel):
    category: str
    start_date: date
    end_date: date

    @field_validator("category")
    @classmethod
    def category_must_be_known(cls, v: str) -> str:
        allowed = {"Kitchen", "Lighting", "Garden"}
        if v not in allowed:
            raise ValueError(f"category must be one of {sorted(allowed)}")
        return v

    @model_validator(mode="after")
    def dates_in_order(self):
        if self.end_date <= self.start_date:
            raise ValueError("end_date must be after start_date")
        return self
```

`field_validator` checks one field. `model_validator(mode="after")` checks relationships *between* fields, once they're all populated.

> **Pydantic v1 vs v2:** v2 renamed things — `@validator` → `@field_validator`, `.dict()` → `.model_dump()`, `class Config` → `model_config`. Most Stack Overflow answers are v1. If a snippet errors with "deprecated", that's why.

---

## 4. Async — and when it actually matters

### The idea

Normal (synchronous) Python does one thing at a time. When it waits on a database query for 50ms, it does *nothing* — the whole process is blocked.

`async` lets the process go and serve another request during that wait.

```python
@app.get("/sales")
async def get_sales():
    rows = await db.fetch("SELECT ...")     # ← during this wait, other requests get served
    return rows
```

### The rule that actually matters

FastAPI supports both, and picks the right behaviour — **if you're honest about which you're writing.**

| Your endpoint | What FastAPI does |
|---|---|
| `async def` with `await` on real async I/O | Runs on the event loop. Fast, concurrent. ✅ |
| plain `def` | Runs in a **thread pool**, so it can't block the loop. Also fine. ✅ |
| `async def` containing a **blocking** call (`time.sleep`, sync `pyodbc`, `requests.get`) | **Blocks the entire server.** Every other request waits. ❌ |

> **The one rule:** if you can't `await` everything inside it, use plain `def` and let FastAPI's thread pool handle it. A plain `def` endpoint is *never* wrong. An `async def` with a blocking call inside is a serious, hard-to-spot performance bug — the API works perfectly with one user and collapses with ten.

```python
import asyncio
import time

@app.get("/bad")
async def bad():
    time.sleep(5)                 # ✗ blocks EVERY request for 5 seconds
    return {"ok": True}

@app.get("/fine")
def fine():
    time.sleep(5)                 # ✓ thread pool — other requests unaffected
    return {"ok": True}

@app.get("/best")
async def best():
    await asyncio.sleep(5)        # ✓ properly async
    return {"ok": True}
```

**Where this bites in the capstone:** ONNX inference is CPU-bound and synchronous. Put it in a plain `def` endpoint (or `await run_in_threadpool(...)`), not naked inside `async def`.

---

## 5. Dependency injection — shared logic, declared

`Depends` says "before running this endpoint, run that function and give me the result."

```python
from fastapi import Depends

async def get_db():
    async with SessionLocal() as session:
        yield session                    # ← endpoint runs here
    # cleanup after the response is sent

@app.get("/sales")
async def get_sales(db = Depends(get_db)):
    return await db.execute(...)
```

The `yield` form is a context manager: setup, hand over, tear down. The session is always closed, even if the endpoint raises.

### Why not just call the function?

Because dependencies are:
- **Composable** — dependencies can have dependencies
- **Overridable in tests** — swap the real database for a fake with one line ([[Testing and CI-CD]])
- **Documented** — a security dependency shows as a lock icon in `/docs`
- **Cached per request** — used by three things in one request, it runs once

### Auth as a dependency

```python
import os
from fastapi import Header, HTTPException, status, APIRouter, Depends

async def verify_api_key(x_api_key: str = Header(...)):
    if x_api_key != os.environ["API_KEY"]:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid API key")
    return x_api_key

@app.get("/sales/forecast", dependencies=[Depends(verify_api_key)])
def forecast(...):
    ...

# or protect a whole group at once
router = APIRouter(prefix="/admin", dependencies=[Depends(verify_api_key)])
```

> A plain API key in a header is the minimum viable auth, fine for a portfolio project behind HTTPS. Real systems use OAuth2/JWT — FastAPI has `fastapi.security` for that. Know the ladder exists; don't build OAuth for a capstone.

---

## 6. Errors

```python
from fastapi import HTTPException, Depends

@app.get("/customers/{customer_id}/segment")
async def get_segment(customer_id: str, db = Depends(get_db)):
    row = await fetch_customer(db, customer_id)
    if row is None:
        raise HTTPException(status_code=404, detail=f"Customer {customer_id} not found")
    return row
```

`raise`, don't `return`. FastAPI turns it into a proper HTTP response.

### Don't leak internals

```python
from fastapi import Request
from fastapi.responses import JSONResponse
import logging

logger = logging.getLogger(__name__)

@app.exception_handler(Exception)
async def unhandled(request: Request, exc: Exception):
    logger.exception("Unhandled error on %s", request.url.path)   # full trace → logs
    return JSONResponse(status_code=500, content={"detail": "Internal server error"})
```

> **Never return a stack trace to the caller.** It reveals your file paths, library versions, and sometimes connection strings — a genuine gift to an attacker. Full detail in the logs, generic message in the response.

---

## 7. Background tasks

For work that shouldn't make the caller wait:

```python
from fastapi import BackgroundTasks

@app.post("/predict/segment")
async def predict(req: PredictionRequest, background: BackgroundTasks):
    result = run_model(req)
    background.add_task(log_prediction, req.customer_id, result)   # after the response
    return result
```

The response goes out immediately; the logging happens after.

> **The limit:** background tasks run **in the same process**. If it crashes or redeploys, the task is lost, and a slow task still consumes a worker. Fine for logging and cache warming. **Not** for model retraining or anything that takes minutes — that needs a real queue (Celery, Azure Service Bus) or a Databricks job.

---

## 8. Project layout

Keep everything in `main.py` and it becomes unmanageable around endpoint six.

```
api/
├── main.py            # app creation, middleware, router registration
├── config.py          # settings from env vars (pydantic-settings)
├── dependencies.py    # get_db, verify_api_key
├── models.py          # Pydantic request/response models
├── db.py              # engine, session factory
├── routers/
│   ├── sales.py
│   ├── customers.py
│   └── health.py
└── services/          # ← the actual logic, with NO FastAPI imports
    ├── forecasting.py
    └── segmentation.py
```

```python
# routers/sales.py
from fastapi import APIRouter, FastAPI
router = APIRouter(prefix="/sales", tags=["sales"])

@router.get("/forecast", response_model=ForecastResponse)
async def forecast(...): ...

# main.py
from api.routers import sales, customers, health
app = FastAPI(title="Retail Analytics API")
app.include_router(sales.router)
app.include_router(customers.router)
app.include_router(health.router)
```

> **The `services/` rule is the one that pays off.** Business logic in there imports nothing from FastAPI. That means you can unit-test it with plain function calls, reuse it in a batch job, and swap the web framework without rewriting it. Routers should be thin: parse input, call a service, return the result.

### Config from environment

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    api_key: str
    model_path: str = "models/forecast.onnx"
    log_level: str = "INFO"

    model_config = {"env_file": ".env"}

settings = Settings()      # reads env vars, validates, crashes loudly if one is missing
```

Crashing at startup on a missing variable is *good*. The alternative is a 500 error at 3am from a `None` config value.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| `422 Unprocessable Entity` | Pydantic rejected the input | Read `detail[].loc` — it names the exact field |
| API works with one user, dies with ten | Blocking call inside `async def` | Use plain `def`, or make the call properly async |
| `/docs` shows nothing for an endpoint | No type hints, or no `response_model` | Add them |
| Response is missing a field you returned | `response_model` filtered it | Add the field to the model |
| "Field required" on a body you did send | Missing `Content-Type: application/json` | Set the header; `requests.post(url, json=...)` does it for you |
| Dashboard blocked by CORS | Browser blocks cross-origin by default | Add `CORSMiddleware` with your dashboard's origin |
| `500` with no useful message | Unhandled exception | Check logs; add the exception handler above |
| Everything is slow | Opening a DB connection per request | Pool it; see [[Azure SQL Database]] |
| Works locally, 404 in Docker | Bound to `127.0.0.1` | `--host 0.0.0.0` ([[Dev environment - Git, Docker, CLI]]) |
| `@validator` deprecation warnings | Pydantic v1 syntax on v2 | `@field_validator` + `@classmethod` |
| Background task never ran | Process restarted, or it raised silently | Log inside the task; use a real queue for anything important |

**CORS**, since it will happen:

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://my-dashboard.azurecontainerapps.io"],   # not ["*"] in production
    allow_methods=["GET", "POST"],
    allow_headers=["*"],
)
```

---

## Practice checklist

- [ ] HTTP basics: methods, status codes, and the 4xx-vs-5xx split
- [ ] Path and query parameters, request bodies — and how type hints do the parsing
- [ ] Pydantic models for request/response validation — and why this replaces manual input checking
- [ ] `response_model` — documentation, output validation, and field filtering as a safety net
- [ ] Custom validators: `field_validator` vs. `model_validator`
- [ ] Async `def` endpoints vs. sync — **and why a blocking call inside `async def` is a real bug**
- [ ] Dependency injection (`Depends`) for shared logic like auth or DB sessions
- [ ] Raising `HTTPException`, and never leaking stack traces
- [ ] Background tasks for work that shouldn't block the response — and their limits
- [ ] Project layout: routers thin, services framework-free
- [ ] Automatic OpenAPI docs (`/docs`) — read them, don't just generate them

## Hands-on

- [ ] Build a small API with 3+ endpoints, full Pydantic validation, and at least one async endpoint
- [ ] Break it on purpose: send a string where an int belongs, read the 422 body
- [ ] Add a `response_model` that omits a field you return, and confirm it's stripped
- [ ] Write one dependency (`get_db` or `verify_api_key`) shared across two endpoints
- [ ] Put `time.sleep(5)` in an `async def`, fire 5 concurrent requests, and watch them queue — then fix it

## Resources

- [FastAPI official tutorial](https://fastapi.tiangolo.com/tutorial/) — genuinely one of the best docs in Python
- [Pydantic v2 docs](https://docs.pydantic.dev/latest/)
- [MDN: HTTP status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)

## Next

[[FastAPI data and deployment]]
