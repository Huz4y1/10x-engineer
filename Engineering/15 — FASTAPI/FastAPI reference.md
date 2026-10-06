---
tags: [fastapi, reference, cheatsheet, api, python]
status: not-started
---

# FastAPI reference

**The complete API reference.** Concepts and why: [[FastAPI fundamentals]]. Production and deployment: [[FastAPI data and deployment]].
This note is what you'd otherwise have the docs open for.

Section: [[15 — FASTAPI]]

---

## I want to…

**Find your task, click straight through to it.**

| I want to… | Go to |
|---|---|
| Understand what an API even is | [[#What FastAPI actually does]] |
| **Write my first endpoint** | [[#The smallest complete app]] |
| Know where my data comes from | [[#Where your data comes from]] |
| See the interactive docs | [[#The free interactive docs]] |
| Choose FastAPI, Streamlit or Django | [[#FastAPI vs the alternatives]] |
| Install and start the server | [[#1. Setup]] |
| **Add a GET / POST / PUT / DELETE route** | [[#The HTTP methods]] |
| Split routes across files | [[#Routers — split across files]] |
| Read a value from the URL path | [[#Path]] |
| Read `?something=value` from the URL | [[#Query]] |
| **Accept JSON in the request body** | [[#Body]] |
| Read a header, cookie or form field | [[#Headers, cookies, forms]] |
| Get the raw request (IP, method, URL) | [[#The Request object]] |
| **Define the shape of my data** | [[#4. Pydantic models]] |
| Make a field optional or nested | [[#Nested and optional]] |
| Convert a model to a dict or JSON | [[#Model methods]] |
| **Validate input (min, max, regex)** | [[#Field validators]] |
| Understand a 422 error | [[#What a validation error looks like]] |
| Control what my endpoint returns | [[#6. Responses]] |
| Return a file, HTML, or a redirect | [[#Response classes]] |
| Set a custom status code or header | [[#Custom status and headers]] |
| **Return a proper error** | [[#7. Errors]] |
| Handle an exception app-wide | [[#Custom exception handlers]] |
| Know which status code to use | [[#Status codes worth knowing]] |
| **Share logic between endpoints** | [[#8. Dependencies]] |
| Open and close a database session | [[#With cleanup (`yield`)]] |
| Apply something to every route | [[#Route / router / app level]] |
| **Know when to use `async def`** | [[#The one rule]] |
| Run blocking code without freezing the server | [[#Running blocking work without blocking]] |
| Do several slow calls at once | [[#Doing several things at once]] |
| **Add an API key** | [[#API key]] |
| Add login with JWT tokens | [[#OAuth2 + JWT]] |
| Run code on every request | [[#11. Middleware]] |
| **Fix a CORS error in the browser** | [[#12. CORS]] |
| **Load a model once at startup** | [[#13. Lifespan]] |
| Do work after returning a response | [[#14. Background tasks]] |
| Handle a file upload or download | [[#15. Files]] |
| Push live updates to a browser | [[#16. WebSockets]] |
| **Connect to a database** | [[#17. Databases]] |
| Read settings and secrets | [[#18. Config]] |
| Lay out a real project | [[#19. Project structure]] |
| **Write tests** | [[#20. Testing]] |
| Mock a dependency in a test | [[#Overriding dependencies — the clean way to mock]] |
| Customise or hide the docs | [[#21. OpenAPI]] |
| **Run it in production** | [[#22. Running it]] |
| Fix something that's broken | [[#Troubleshooting]] |

---

## Contents

0. [[#0. The basics — start here\|The basics — start here]] · 1. [[#1. Setup\|Setup]] · 2. [[#2. Routing\|Routing]] · 3. [[#3. Parameters\|Parameters]]
4. [[#4. Pydantic models\|Pydantic models]] · 5. [[#5. Validation\|Validation]] · 6. [[#6. Responses\|Responses]] · 7. [[#7. Errors\|Errors]]
8. [[#8. Dependencies\|Dependencies]] · 9. [[#9. Async\|Async]] · 10. [[#10. Authentication\|Authentication]] · 11. [[#11. Middleware\|Middleware]]
12. [[#12. CORS\|CORS]] · 13. [[#13. Lifespan\|Lifespan]] · 14. [[#14. Background tasks\|Background tasks]] · 15. [[#15. Files\|Files]]
16. [[#16. WebSockets\|WebSockets]] · 17. [[#17. Databases\|Databases]] · 18. [[#18. Config\|Config]] · 19. [[#19. Project structure\|Project structure]]
20. [[#20. Testing\|Testing]] · 21. [[#21. OpenAPI\|OpenAPI]] · 22. [[#22. Running it\|Running it]]

---

## 0. The basics — start here

### What FastAPI actually does

**An API is a way for one program to ask another program for something, over HTTP.**

Your model lives in Python. A website, a phone app or another service needs a prediction from it. FastAPI is the layer in between: it listens for requests, gives them to your Python function, and sends the answer back as JSON.

```
[browser / app / another service]
            |  HTTP request:  GET /items/42
            v
      [ FastAPI ]  ── matches the URL to your function
            |
            v
      your Python function  ->  returns a dict
            |
            v
        JSON response  {"id": 42, "name": "mug"}
```

### The smallest complete app

```python
from fastapi import FastAPI          # the framework

app = FastAPI()                      # the application object - everything hangs off this

@app.get("/hello")                   # when someone does GET /hello, run the function below
def hello():                         # the "path operation function"
    return {"message": "hi"}         # a plain dict - FastAPI turns it into JSON for you
```

```bash
uvicorn main:app --reload
#        ^file ^variable    ^restart automatically when you save
```

Then open **http://localhost:8000/hello**. That's a working API.

> **You never write JSON conversion, HTTP status codes, or request parsing.** Return a dict, a list, or a Pydantic model and FastAPI serialises it.

### The four things every endpoint has

```python
@app.post("/items/{item_id}", response_model=Item, status_code=201)
#    ^1     ^2                 ^3                   ^4
async def update(item_id: int, body: ItemIn):
    ...
```

1. **The HTTP method** — `get` read · `post` create · `put` replace · `patch` update · `delete` remove
2. **The path** — the URL, with `{braces}` for the changeable parts
3. **`response_model`** — the shape of what you send back
4. **`status_code`** — the default success code for this route

### Where your data comes from

FastAPI decides this from the **type hint**, which is why type hints are not optional here:

```python
from fastapi import Header

@app.post("/items/{item_id}")
async def f(
    item_id: int,               # in the PATH, because {item_id} is in the URL
    limit: int = 10,            # in the QUERY string, because it isn't in the path: ?limit=10
    body: ItemIn,               # in the BODY, because it's a Pydantic model
    x_key: str = Header(),      # in a HEADER
):
```

| Where it lives | How FastAPI knows |
|---|---|
| **Path** | The name appears in `{braces}` in the URL |
| **Query** | A simple type (`int`, `str`, `bool`) that isn't in the path |
| **Body** | The type is a Pydantic model |
| **Header / Cookie / Form** | You say so explicitly: `Header()`, `Cookie()`, `Form()` |

### Validation is free

```python
from pydantic import BaseModel, Field

class ItemIn(BaseModel):
    name: str = Field(min_length=1, max_length=50)
    price: float = Field(gt=0)                       # must be greater than zero
    qty: int = 1
```

Send `{"name": "", "price": -5}` and FastAPI rejects it with a **422** and a message naming the exact field — before your function runs. You write no validation code ([[Pydantic]]).

> **This is the main reason to choose FastAPI.** The type hints do triple duty: validation, JSON conversion, and the interactive docs.

### The free interactive docs

Start the server and open **http://localhost:8000/docs**.

Every endpoint is listed, with its parameters, its response shape, and a **"Try it out"** button that sends a real request. Generated entirely from your type hints — you maintain nothing.

### `def` vs `async def` — the one rule that matters

```python
import requests

@app.get("/a")
async def a():
    return await db.fetch("SELECT 1")     # everything inside is awaited - correct

@app.get("/b")
def b():
    return requests.get("...").json()     # blocking library, plain def - correct

@app.get("/c")
async def c():
    return requests.get("...").json()     # BLOCKING call inside async - WRONG
```

> ⚠️ **If you can't `await` everything inside the function, use plain `def`.** FastAPI runs `def` endpoints in a thread pool so they can't block. An `async def` containing a blocking call works perfectly with one user and falls over with ten — see §9.

### The commands you'll actually use

```bash
uv add fastapi "uvicorn[standard]"          # install
uvicorn main:app --reload                   # run in development
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4    # run in production
curl localhost:8000/items/42                # call an endpoint
curl -X POST localhost:8000/items -H "Content-Type: application/json" -d '{"name":"mug"}'
```

### FastAPI vs the alternatives

| | Use when |
|---|---|
| **FastAPI** | An API for another program to call — models, microservices, mobile backends |
| **[[Streamlit]]** | A dashboard a *human* looks at. No HTML, no routes |
| **[[Django]]** | A full website with pages, admin, sessions and templates |
| **[[Actix Web]]** | You need Rust-level performance, or it's your main backend |

> **FastAPI and Streamlit pair well:** FastAPI serves the model, Streamlit calls the API and draws the charts ([[Streamlit vs Django]]).

---

## 1. Setup

### Installing

```bash
uv add fastapi "uvicorn[standard]" pydantic pydantic-settings
uv add sqlalchemy asyncpg          # database
uv add --dev pytest httpx          # testing
```

**Without uv — a plain virtual environment:**

| | Windows (PowerShell) | WSL / Linux |
|---|---|---|
| Create | `python -m venv .venv` | `python3 -m venv .venv` |
| Activate | `.venv\Scripts\Activate.ps1` | `source .venv/bin/activate` |
| Install | `pip install fastapi "uvicorn[standard]"` | same |

```bash
python -c "import fastapi; print(fastapi.__version__)"     # check it worked
```

> **Quote the brackets in PowerShell and zsh:** `uv add "uvicorn[standard]"`. Unquoted, both shells try to expand `[standard]` as a pattern and the install fails confusingly.

> ⚠️ **`--reload` needs `uvicorn[standard]`**, not plain `uvicorn` — the extra pulls in the file watcher. Without it, `--reload` is silently ignored and you'll wonder why your edits do nothing.

> ⚠️ **In WSL, use `--host 0.0.0.0` if you want to reach the API from Windows.** Bound to `127.0.0.1` it's reachable from inside WSL only. `localhost:8000` in a Windows browser usually works via WSL2 port forwarding, but `0.0.0.0` is the reliable answer — and it's what you need in Docker too ([[Docker deep dive]]).

> **On Windows, `uvloop` and `httptools` don't install** — that's expected. `uvicorn[standard]` skips them and falls back to the standard asyncio loop. Slightly slower, entirely fine for development; production runs in Linux containers anyway.

> **`gunicorn` is Linux-only.** On Windows use `uvicorn` directly with `--workers`; in a container use gunicorn with uvicorn workers (§22).

```python
from fastapi import FastAPI

app = FastAPI(
    title="Pipeline API",
    description="Serves forecasts and segments",
    version="1.0.0",
    docs_url="/docs",            # Swagger UI;  None to disable
    redoc_url="/redoc",
    openapi_url="/openapi.json", # None to hide the schema entirely
)
```

```bash
uvicorn api.main:app --reload                    # dev only
uvicorn api.main:app --host 0.0.0.0 --port 8000 --workers 4   # prod
```

Interactive docs at **http://localhost:8000/docs** — generated from your type hints, with a working "Try it out".

---

## 2. Routing

### The HTTP methods

```python
@app.get("/items")
@app.post("/items")
@app.put("/items/{id}")
@app.patch("/items/{id}")
@app.delete("/items/{id}")
@app.head("/items")
@app.options("/items")
```

### All the decorator options

```python
@app.get(
    "/items/{item_id}",
    response_model=Item,
    status_code=200,
    tags=["items"],                  # groups them in /docs
    summary="Get one item",
    description="Longer description",
    response_description="The item",
    deprecated=False,
    include_in_schema=True,
)
async def get_item(item_id: int): ...
```

### Routers — split across files

```python
# routers/items.py
from fastapi import APIRouter, Depends

router = APIRouter(
    prefix="/items",
    tags=["items"],
    dependencies=[Depends(verify_api_key)],      # applies to EVERY route here
    responses={404: {"description": "Not found"}},
)

@router.get("/")            # -> GET /items/
async def list_items(): ...

# main.py
from api.routers import items, health
app.include_router(items.router)
app.include_router(health.router, prefix="/v1")
```

> **Route order matters.** `/items/me` must be declared **before** `/items/{item_id}`, or `me` gets captured as an `item_id` and fails to parse as an int.

---

## 3. Parameters

### Path

```python
@app.get("/items/{item_id}")
async def get(item_id: int): ...                 # auto-converted, 422 if not an int

from fastapi import Path
async def get(item_id: int = Path(..., ge=1, le=1000, description="Item ID")): ...

@app.get("/files/{file_path:path}")              # captures slashes
async def read_file(file_path: str): ...
```

### Query

Any argument not in the path becomes a query parameter.

```python
from fastapi import Query

@app.get("/items")
async def list_items(
    q: str | None = None,                        # optional
    skip: int = 0,                               # default
    limit: int = Query(10, ge=1, le=100),        # constrained
    tags: list[str] = Query(default=[]),         # ?tags=a&tags=b
    pattern: str = Query(None, min_length=3, max_length=50, pattern=r"^[a-z]+$"),
    alias_example: str = Query(None, alias="item-query"),   # hyphens in the URL
    deprecated_param: str = Query(None, deprecated=True),
): ...
```

`...` (Ellipsis) means **required**.

| Constraint | For |
|---|---|
| `ge` / `gt` / `le` / `lt` | Numbers |
| `min_length` / `max_length` | Strings and lists |
| `pattern` | Regex on strings |

### Body

```python
@app.post("/items")
async def create(item: Item): ...                # Pydantic model -> JSON body

async def create(item: Item, user: User): ...    # two models -> nested body keys

from fastapi import Body
async def create(item: Item, importance: int = Body(...)):  ...
async def create(item: Item = Body(..., embed=True)): ...    # {"item": {...}}
```

### Headers, cookies, forms

```python
from fastapi import Header, Cookie, Form

async def h(user_agent: str = Header(None),
            x_api_key: str = Header(...)):       # x_api_key -> X-API-Key automatically
    ...

async def c(session_id: str = Cookie(None)): ...

async def login(username: str = Form(...), password: str = Form(...)):  ...
```

> **`Header` converts underscores to hyphens.** `x_api_key` reads the `X-API-Key` header. Disable with `Header(..., convert_underscores=False)`.

### The Request object

```python
from fastapi import Request

@app.get("/raw")
async def raw(request: Request):
    return {
        "client": request.client.host,
        "headers": dict(request.headers),
        "query": dict(request.query_params),
        "url": str(request.url),
        "body": (await request.body()).decode(),
    }
```

---

## 4. Pydantic models

```python
from pydantic import BaseModel, Field, EmailStr, HttpUrl
from datetime import datetime, date
from decimal import Decimal
from enum import Enum

class Status(str, Enum):
    active = "active"
    archived = "archived"

class Item(BaseModel):
    id: int
    name: str = Field(..., min_length=1, max_length=100)
    price: Decimal = Field(..., gt=0, decimal_places=2)     # money -> Decimal
    quantity: int = Field(default=0, ge=0)
    tags: list[str] = Field(default_factory=list)           # NOT default=[]
    meta: dict[str, str] = Field(default_factory=dict)
    status: Status = Status.active
    email: EmailStr | None = None
    website: HttpUrl | None = None
    created_at: datetime
    due: date | None = None

    model_config = {
        "json_schema_extra": {                              # example in /docs
            "examples": [{"id": 1, "name": "Mug", "price": "2.55"}]
        }
    }
```

> **`default_factory=list`, never `default=[]`.** A mutable default is shared between every instance — the classic Python bug.

> **Money is `Decimal`, never `float`.** Same rule as [[Data modeling]] — floats can't represent 0.1 exactly.

### Nested and optional

```python
from pydantic import BaseModel

class Address(BaseModel):
    city: str
    postcode: str

class User(BaseModel):
    name: str
    address: Address                    # nested
    addresses: list[Address] = []       # list of models
    nickname: str | None = None         # optional, defaults None
    tags: set[str] = set()              # deduplicated automatically
```

### Model methods

```python
item.model_dump()                       # -> dict
item.model_dump(exclude={"id"})
item.model_dump(exclude_none=True)
item.model_dump(exclude_unset=True)     # only fields actually provided
item.model_dump_json()                  # -> JSON string
Item.model_validate(some_dict)          # dict -> model, validates
Item.model_validate_json(json_str)
Item.model_json_schema()                # the JSON Schema
item.model_copy(update={"price": 9.99})
```

> **Pydantic v2 renamed everything.** `.dict()` -> `.model_dump()`, `.json()` -> `.model_dump_json()`, `parse_obj()` -> `model_validate()`, `class Config` -> `model_config`. Most Stack Overflow answers are v1.

---

## 5. Validation

### Field validators

```python
from pydantic import field_validator, model_validator, ValidationInfo, BaseModel

class Booking(BaseModel):
    category: str
    start: date
    end: date

    @field_validator("category")
    @classmethod
    def known_category(cls, v: str) -> str:
        allowed = {"Kitchen", "Lighting", "Garden"}
        if v not in allowed:
            raise ValueError(f"must be one of {sorted(allowed)}")
        return v

    @field_validator("category", mode="before")   # runs BEFORE type coercion
    @classmethod
    def strip_it(cls, v):
        return v.strip() if isinstance(v, str) else v

    @model_validator(mode="after")                # after all fields are set
    def dates_in_order(self):
        if self.end <= self.start:
            raise ValueError("end must be after start")
        return self
```

| Mode | When |
|---|---|
| `mode="before"` | Raw input, before type conversion — for cleaning |
| `mode="after"` (default) | Typed value — for business rules |
| `model_validator(mode="after")` | Cross-field rules |

### What a validation error looks like

```json
{"detail": [{
  "type": "greater_than",
  "loc": ["body", "price"],
  "msg": "Input should be greater than 0",
  "input": -5
}]}
```

> **`loc` names the exact field.** When you see a 422, read `detail[].loc` first — it tells you precisely what was rejected.

### Custom types

```python
from typing import Annotated
from pydantic import AfterValidator, BaseModel, Field

def must_be_upper(v: str) -> str:
    if not v.isupper(): raise ValueError("must be uppercase")
    return v

CountryCode = Annotated[str, Field(min_length=2, max_length=2), AfterValidator(must_be_upper)]

class Order(BaseModel):
    country: CountryCode
```

---

## 6. Responses

```python
@app.get("/items/{id}", response_model=Item)
async def get(id: int) -> Item: ...

@app.get("/items", response_model=list[Item])
@app.post("/items", response_model=Item, status_code=201)

@app.get("/items/{id}", response_model=Item, response_model_exclude={"internal_cost"})
@app.get("/items/{id}", response_model=Item, response_model_exclude_none=True)
@app.get("/items/{id}", response_model=Item, response_model_exclude_unset=True)
```

> **`response_model` does three jobs:** documents the shape, **validates your own output**, and **filters out anything not in the model**. That last one is a security feature — return a database row containing an internal cost column and it's silently stripped.

### Response classes

```python
from fastapi import FastAPI
from fastapi.responses import FileResponse, RedirectResponse, StreamingResponse
from fastapi.responses import (JSONResponse, HTMLResponse, PlainTextResponse,
                               RedirectResponse, StreamingResponse, FileResponse,
                               ORJSONResponse)

@app.get("/html", response_class=HTMLResponse)
async def html(): return "<h1>Hi</h1>"

@app.get("/redirect")
async def r(): return RedirectResponse("/docs", status_code=307)

@app.get("/download")
async def d(): return FileResponse("report.csv", filename="report.csv",
                                   media_type="text/csv")

@app.get("/stream")
async def s():
    def gen():
        for i in range(1000):
            yield f"line {i}\n"
    return StreamingResponse(gen(), media_type="text/plain")

app = FastAPI(default_response_class=ORJSONResponse)     # faster JSON everywhere
```

### Custom status and headers

```python
from fastapi import Response

@app.post("/items")
async def create(item: Item, response: Response):
    response.status_code = 201
    response.headers["X-Custom"] = "value"
    response.set_cookie("session", "abc", httponly=True, secure=True, samesite="lax")
    return item
```

---

## 7. Errors

```python
from fastapi import HTTPException, status

@app.get("/items/{id}")
async def get(id: int):
    item = await fetch(id)
    if item is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Item {id} not found",
            headers={"X-Error": "not-found"},
        )
    return item
```

> **`raise`, don't `return`.** FastAPI converts the exception into a proper HTTP response.

### Custom exception handlers

```python
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
import logging

logger = logging.getLogger(__name__)

class ItemNotFound(Exception):
    def __init__(self, item_id: int): self.item_id = item_id

@app.exception_handler(ItemNotFound)
async def item_not_found_handler(request, exc: ItemNotFound):
    return JSONResponse(status_code=404, content={"detail": f"Item {exc.item_id} missing"})

@app.exception_handler(RequestValidationError)
async def validation_handler(request, exc):
    return JSONResponse(status_code=422, content={"detail": exc.errors()})

@app.exception_handler(Exception)                # catch-all
async def unhandled(request, exc):
    logger.exception("Unhandled error on %s", request.url.path)   # FULL trace -> logs
    return JSONResponse(status_code=500, content={"detail": "Internal server error"})
```

> ⚠️ **Never return a stack trace to the caller.** It reveals file paths, library versions and sometimes connection strings. Full detail in the logs; a generic message in the response.

### Status codes worth knowing

| Code | Meaning | Whose fault |
|---|---|---|
| 200 / 201 / 204 | OK / Created / No content | — |
| **400** | Malformed request | Caller |
| **401** | Not authenticated — *who are you?* | Caller |
| **403** | Authenticated, not allowed | Caller |
| 404 | Not found | Caller |
| 409 | Conflict (duplicate) | Caller |
| **422** | Validation failed | Caller — **Pydantic's code** |
| 429 | Rate limited | Caller |
| **500** | Your code crashed | **You** |
| 503 | Unavailable / starting | You |

---

## 8. Dependencies

```python
from fastapi import Depends

async def common_params(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}

@app.get("/items")
async def list_items(params: dict = Depends(common_params)):
    return params
```

### With cleanup (`yield`)

```python
async def get_db():
    async with SessionLocal() as session:
        try:
            yield session                # endpoint runs here
        finally:
            await session.close()        # always runs, even on exception
```

### Sub-dependencies

```python
from fastapi import Depends, Header

async def get_token(x_token: str = Header(...)):
    return x_token

async def get_user(token: str = Depends(get_token)):
    return await lookup_user(token)

@app.get("/me")
async def me(user: User = Depends(get_user)):    # chains automatically
    return user
```

### Class dependencies

```python
from fastapi import Depends, Query

class Pagination:
    def __init__(self, skip: int = 0, limit: int = Query(10, le=100)):
        self.skip, self.limit = skip, limit

@app.get("/items")
async def list_items(p: Pagination = Depends()):
    return {"skip": p.skip, "limit": p.limit}
```

### Route / router / app level

```python
from fastapi import APIRouter, Depends, FastAPI

@app.get("/x", dependencies=[Depends(verify_key)])          # result discarded
router = APIRouter(dependencies=[Depends(verify_key)])      # whole router
app = FastAPI(dependencies=[Depends(verify_key)])           # everything
```

### Annotated style (modern, reusable)

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from typing import Annotated

DB = Annotated[AsyncSession, Depends(get_db)]
CurrentUser = Annotated[User, Depends(get_user)]

@app.get("/items")
async def list_items(db: DB, user: CurrentUser): ...
```

> **Dependencies are cached per request.** If three things depend on `get_db`, it runs once. Disable with `Depends(fn, use_cache=False)`.

---

## 9. Async

### The one rule

```python
@app.get("/async")
async def a():
    return await db.fetch("SELECT 1")            # ✅ properly async

@app.get("/sync")
def s():
    return blocking_call()                       # ✅ runs in a thread pool

@app.get("/bad")
async def bad():
    return blocking_call()                       # ❌ BLOCKS THE ENTIRE SERVER
```

> **The one rule: if you can't `await` everything inside it, use plain `def`.** FastAPI runs `def` endpoints in a thread pool so they can't block the event loop. An `async def` containing a blocking call works perfectly with one user and collapses with ten.

### Running blocking work without blocking

```python
from fastapi import Request
from fastapi.concurrency import run_in_threadpool

@app.post("/predict")
async def predict(req: Request):
    result = await run_in_threadpool(model.run, None, inputs)   # CPU work off the loop
    return result
```

### Doing several things at once

```python
import asyncio
results = await asyncio.gather(fetch_a(), fetch_b(), fetch_c())   # all three run concurrently
```

> **`gather` is the payoff for async.** Three calls taking 100 ms each finish in ~100 ms total, not 300 ms.

---

## 10. Authentication

### API key

```python
from fastapi import Depends, HTTPException
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

async def verify_api_key(key: str = Depends(api_key_header)):
    if key != settings.api_key:
        raise HTTPException(401, "Invalid API key")
    return key

@app.get("/secure", dependencies=[Depends(verify_api_key)])
async def secure(): ...
```

### OAuth2 + JWT

```python
from fastapi import Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from passlib.context import CryptContext
from datetime import datetime, timedelta, timezone

pwd = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2 = OAuth2PasswordBearer(tokenUrl="token")

def create_token(data: dict, minutes: int = 30) -> str:
    payload = data | {"exp": datetime.now(timezone.utc) + timedelta(minutes=minutes)}
    return jwt.encode(payload, settings.secret_key, algorithm="HS256")

@app.post("/token")
async def login(form: OAuth2PasswordRequestForm = Depends()):
    user = await authenticate(form.username, form.password)
    if not user:
        raise HTTPException(401, "Incorrect username or password",
                            headers={"WWW-Authenticate": "Bearer"})
    return {"access_token": create_token({"sub": user.username}), "token_type": "bearer"}

async def current_user(token: str = Depends(oauth2)) -> User:
    try:
        payload = jwt.decode(token, settings.secret_key, algorithms=["HS256"])
        username = payload.get("sub")
        if username is None: raise JWTError
    except JWTError:
        raise HTTPException(401, "Could not validate credentials")
    return await get_user(username)
```

```python
pwd.hash("plaintext")             # store this
pwd.verify("plaintext", hashed)   # check it
```

> **Never store passwords, only hashes.** bcrypt or argon2 — never MD5 or SHA. See [[26 — SECURITY]].

> **Put auth in front of the app where you can** — Container Apps / Cloud Run / ALB auth is simpler and stricter than anything in Python ([[Streamlit deployment]]).

---

## 11. Middleware

```python
import time
from fastapi import Request

@app.middleware("http")
async def add_timing(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    duration = time.perf_counter() - start
    response.headers["X-Process-Time"] = f"{duration:.4f}"
    logger.info("request", extra={"extra_fields": {
        "path": request.url.path, "method": request.method,
        "status": response.status_code, "duration_ms": round(duration * 1000, 1)}})
    return response
```

```python
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware

app.add_middleware(GZipMiddleware, minimum_size=1000)
app.add_middleware(TrustedHostMiddleware, allowed_hosts=["example.com", "*.example.com"])
```

> **Middleware runs in reverse order of addition** for responses. And it runs for *every* request — keep it fast.

---

## 12. CORS

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://dashboard.example.com"],   # NOT ["*"] in production
    allow_credentials=True,
    allow_methods=["GET", "POST"],
    allow_headers=["*"],
    max_age=600,
)
```

> **A browser blocks cross-origin requests by default** — that's what a CORS error means. Your Streamlit dashboard on a different host needs its origin listed. `allow_origins=["*"]` with `allow_credentials=True` is rejected by browsers.

---

## 13. Lifespan

```python
from fastapi import FastAPI
import joblib
import onnxruntime as ort
from contextlib import asynccontextmanager

state: dict = {}

@asynccontextmanager
async def lifespan(app: FastAPI):
    # ---- startup ----
    state["model"] = ort.InferenceSession(settings.model_path)
    state["scaler"] = joblib.load(settings.scaler_path)
    logger.info("Models loaded")
    yield                                    # app serves requests here
    # ---- shutdown ----
    state.clear()
    await engine.dispose()

app = FastAPI(lifespan=lifespan)
```

> **Load models and create the DB engine here, once.** Doing it per request adds hundreds of milliseconds to every call.

> ⚠️ **`joblib.load` / `pickle.load` execute arbitrary code on load.** That is fine here because you are loading *your own* artifact, built by your own training job and shipped in your own image. **Never load a pickle from user upload, a public bucket, or anywhere you don't control** — it is remote code execution. For untrusted data use JSON, or ONNX for models ([[Model export and serving]]).
>
> **Let a missing model file crash startup.** The container fails to start and the deploy rolls back — far better than a "healthy" container returning 500s.

> `@app.on_event("startup")` is deprecated. Use `lifespan`.

---

## 14. Background tasks

```python
from fastapi import BackgroundTasks

@app.post("/predict")
async def predict(req: PredictRequest, background: BackgroundTasks):
    result = run_model(req)
    background.add_task(log_prediction, req.id, result)    # runs AFTER the response
    return result
```

> **Limits:** runs in the same process, so a crash or redeploy loses it, and a slow task consumes a worker. Fine for logging and cache warming. **Not** for model retraining — that needs a real queue (Celery, Service Bus) or a Databricks job.

---

## 15. Files

```python
from fastapi import File, UploadFile

@app.post("/upload")
async def upload(file: UploadFile = File(...)):
    content = await file.read()              # whole file into memory
    return {"filename": file.filename, "type": file.content_type, "size": len(content)}

@app.post("/upload-many")
async def many(files: list[UploadFile] = File(...)): ...

# stream a large file - don't load it all
@app.post("/upload-big")
async def big(file: UploadFile = File(...)):
    with open(f"/tmp/{file.filename}", "wb") as f:
        while chunk := await file.read(1024 * 1024):
            f.write(chunk)
```

```python
from fastapi.responses import FileResponse
@app.get("/download")
async def download():
    return FileResponse("report.csv", filename="report.csv", media_type="text/csv")
```

> **Validate size and type.** An unbounded upload is a denial-of-service hole. Check `file.content_type` and cap the size.

---

## 16. WebSockets

```python
from fastapi import WebSocket, WebSocketDisconnect

@app.websocket("/ws")
async def ws(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"echo: {data}")
    except WebSocketDisconnect:
        logger.info("client disconnected")
```

```python
class Manager:
    def __init__(self): self.active: list[WebSocket] = []
    async def connect(self, ws): await ws.accept(); self.active.append(ws)
    def disconnect(self, ws): self.active.remove(ws)
    async def broadcast(self, msg):
        for c in self.active: await c.send_text(msg)
```

---

## 17. Databases

```python
# db.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

engine = create_async_engine(                    # MODULE LEVEL - created once
    settings.database_url,
    pool_size=5, max_overflow=10,
    pool_recycle=1800,                           # cloud DBs drop idle connections
    pool_pre_ping=True,                          # test before handing out
    echo=False,
)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)
```

> **One engine, at module level.** Creating it per request creates a pool per request and exhausts the database's connection limit in minutes.
> **`pool_pre_ping` and `pool_recycle` are mandatory on cloud databases** ([[Azure SQL Database]]).

```python
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text

async def get_sales(db: AsyncSession, category: str):
    result = await db.execute(
        text("SELECT date, revenue FROM daily_sales WHERE category = :cat"),
        {"cat": category},                       # NAMED PARAMS - never f-strings
    )
    return [dict(r._mapping) for r in result]
```

> ⚠️ **Never f-string user input into SQL.** `f"WHERE cat = '{category}'"` is a SQL injection hole. Named parameters are also faster — the plan gets cached.

---

## 18. Config

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    env: str = "local"
    database_url: str                            # required - crashes if missing
    api_key: str
    model_path: str = "models/model.onnx"
    log_level: str = "INFO"
    cors_origins: list[str] = []

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

settings = Settings()
```

> **Crashing at startup on a missing variable is good.** The alternative is a 500 at 3am from a `None` config value.

---

## 19. Project structure

```
api/
├── main.py            # app, middleware, routers, lifespan
├── config.py          # Settings
├── db.py              # engine, session factory
├── dependencies.py    # get_db, verify_api_key
├── models.py          # Pydantic request/response models
├── routers/
│   ├── health.py
│   ├── sales.py
│   └── customers.py
└── services/          # ← business logic, NO FastAPI imports
    ├── forecasting.py
    └── segmentation.py
```

> **The `services/` rule pays off most.** Logic with no FastAPI imports can be unit-tested with plain function calls, reused in a batch job, and survives a framework change. **Routers stay thin: parse input, call a service, return.**

---

## 20. Testing

```python
from fastapi.testclient import TestClient
from api.main import app

client = TestClient(app)

def test_health():
    r = client.get("/health")
    assert r.status_code == 200
    assert r.json() == {"status": "ok"}

def test_validation():
    r = client.post("/items", json={"price": -1})
    assert r.status_code == 422
    assert "price" in str(r.json()["detail"])
```

### Async tests

```python
import pytest
from httpx import AsyncClient, ASGITransport

@pytest.mark.asyncio
async def test_predict():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        r = await c.post("/predict", json={"id": "1"})
    assert r.status_code == 200
```

### Overriding dependencies — the clean way to mock

```python
async def fake_db():
    return FakeSession(rows=[{"date": "2024-01-01", "revenue": 100.0}])

app.dependency_overrides[get_db] = fake_db
# ... run tests ...
app.dependency_overrides.clear()                 # always clean up
```

> **This is why dependency injection is worth using** — no `patch()`, no string paths to get wrong. See [[Testing and CI-CD]].

---

## 21. OpenAPI

```python
from fastapi import FastAPI

app = FastAPI(
    title="Pipeline API", version="1.0.0",
    openapi_tags=[{"name": "sales", "description": "Forecasting endpoints"}],
    servers=[{"url": "https://api.example.com", "description": "Production"}],
)

@app.get("/x", responses={
    200: {"description": "Success"},
    404: {"description": "Not found", "content": {"application/json": {
          "example": {"detail": "Item not found"}}}},
})
async def x(): ...
```

```python
from fastapi import FastAPI

# hide docs in production
app = FastAPI(docs_url=None, redoc_url=None, openapi_url=None)
```

```bash
curl localhost:8000/openapi.json > openapi.json   # generate a client from this
```

---

## 22. Running it

```bash
# development
uvicorn api.main:app --reload

# production
uvicorn api.main:app --host 0.0.0.0 --port 8000 --workers 4
gunicorn api.main:app -w 4 -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000
```

| Setting | Note |
|---|---|
| `--host 0.0.0.0` | **Mandatory in containers**, or nothing reaches it |
| `--workers N` | Separate processes. **Each loads its own copy of the model.** |
| `--reload` | **Development only** — slow and unsafe |
| `--proxy-headers` | Behind a load balancer, to get real client IPs |

> **Sizing workers:** a 500 MB model × 4 workers = 2 GB. Match workers to the container's memory and CPU, not to a number from a blog post.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `422 Unprocessable Entity` | Pydantic rejected the input | Read `detail[].loc` — it names the field |
| "Field required" on a body you sent | Missing `Content-Type: application/json` | `requests.post(url, json=...)` |
| Works with 1 user, dies with 10 | Blocking call inside `async def` | Plain `def`, or `run_in_threadpool` |
| `/docs` empty for an endpoint | No type hints | Add them |
| Field missing from the response | `response_model` filtered it | Add it to the model |
| CORS error in the browser | Origin not allowed | `CORSMiddleware` |
| `500` with no detail | Unhandled exception | Check logs; add the catch-all handler |
| Everything slow | DB connection per request | One module-level engine |
| `too many connections` | Engine created per request | Same fix |
| Connection refused in Docker | Bound to `127.0.0.1` | `--host 0.0.0.0` |
| No logs in the container | Buffered stdout | `ENV PYTHONUNBUFFERED=1` |
| First request slow, rest fast | Model loaded per request | Load it in `lifespan` |
| `/items/me` returns a parse error | Route order | Declare it before `/items/{id}` |
| `@validator` deprecation warnings | Pydantic v1 syntax on v2 | `@field_validator` + `@classmethod` |
| Background task never ran | Process restarted, or it raised silently | Log inside it; use a real queue |

More: [[I HAVE A PROBLEM]]

## Related

[[FastAPI fundamentals]] · [[FastAPI data and deployment]] · [[Streamlit]] · [[Testing and CI-CD]] · [[Docker deep dive]] · [[26 — SECURITY]] · [[Deployment patterns]] · [[Model export and serving]]
