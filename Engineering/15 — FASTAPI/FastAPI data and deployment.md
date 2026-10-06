---
tags: [fastapi, deployment, docker, azure]
status: not-started
---

# FastAPI: Data & Deployment

> **What this is:** wiring the API to a real database and a real model, then getting it running somewhere other than your laptop.
> **Why you care:** an API on `localhost` isn't a deliverable. This is the note that ends with a URL you can send someone.

---

## 1. The database connection

### One engine, created once

The single most important structural rule:

```python
# api/db.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from api.config import settings

engine = create_async_engine(          # ← module level. Created ONCE when imported.
    settings.database_url,
    pool_size=5,
    max_overflow=10,
    pool_recycle=1800,      # Azure drops idle connections
    pool_pre_ping=True,     # test before handing out — mandatory on Azure SQL
    echo=False,             # True to log every query — useful when debugging
)

SessionLocal = async_sessionmaker(engine, expire_on_commit=False)
```

Create the engine inside an endpoint and you create a new connection pool per request. That exhausts the database's connection limit in minutes. See [[Azure SQL Database]] for why `pool_pre_ping` isn't optional here.

### Sessions as a dependency

```python
# api/dependencies.py
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession
from api.db import SessionLocal

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with SessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

One session per request, always closed. `Depends(get_db)` in any endpoint that needs it.

### Querying — the service layer

```python
# api/services/sales.py  — NOTE: no FastAPI imports in here
from datetime import date
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

async def get_daily_sales(
    db: AsyncSession, category: str, start: date, end: date
) -> list[dict]:
    result = await db.execute(
        text("""
            SELECT date, revenue, units_sold
            FROM daily_sales
            WHERE product_category = :category
              AND date BETWEEN :start AND :end
            ORDER BY date
        """),
        {"category": category, "start": start, "end": end},
    )
    return [dict(row._mapping) for row in result]
```

```python
# api/routers/sales.py — thin
from fastapi import Depends

@router.get("/history", response_model=list[SalesPoint])
async def history(category: str, start: date, end: date, db=Depends(get_db)):
    return await get_daily_sales(db, category, start, end)
```

> **`:named` parameters, never f-strings.** `f"WHERE category = '{category}'"` lets anyone pass `'; DROP TABLE daily_sales; --`. Parameters are also *faster*, because the database caches the query plan for a reusable shape.

### Raw SQL or the ORM?

SQLAlchemy offers both. For this project, **raw SQL via `text()`**:

- Your gold tables are read-only aggregates. You're not managing object lifecycles.
- The queries came from your Databricks work — you can paste them across.
- One less abstraction to debug.

The ORM earns its keep in an app that creates, updates and relates objects (that's Django's territory, in [[Streamlit vs Django]]). A read-only serving API doesn't need it. Using SQLAlchemy Core for the pool while writing plain SQL is a completely respectable choice.

---

## 2. Loading the model — `lifespan`

### The problem

```python
@app.get("/predict")
def predict(...):
    session = onnxruntime.InferenceSession("model.onnx")   # ✗ 500ms, EVERY request
    return session.run(...)
```

Loading a model reads a file and allocates memory. Doing it per request adds hundreds of milliseconds to every call and wastes memory.

### The fix

Load once at startup, keep it in memory for the process's life.

```python
# api/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
import onnxruntime as ort
import logging

logger = logging.getLogger(__name__)
ml_models: dict[str, ort.InferenceSession] = {}

@asynccontextmanager
async def lifespan(app: FastAPI):
    # ---- startup ----
    logger.info("Loading models…")
    ml_models["forecast"]  = ort.InferenceSession(settings.forecast_model_path)
    ml_models["segment"]   = ort.InferenceSession(settings.segment_model_path)
    logger.info("Models loaded")

    yield                       # ← the app serves requests here

    # ---- shutdown ----
    ml_models.clear()
    await engine.dispose()      # close the connection pool cleanly
    logger.info("Shut down")

app = FastAPI(title="Retail Analytics API", lifespan=lifespan)
```

Everything before `yield` runs once at startup; everything after runs at shutdown.

> `@app.on_event("startup")` is the old way and is deprecated. Use `lifespan`.

> **Deliberately crash on a missing model.** If the file isn't there, let the exception propagate — the container fails to start and your deployment rolls back. The alternative is a "healthy" container returning 500s to real users. Fail fast, fail visibly.

### Serving a prediction

```python
from pydantic import BaseModel, Field
import numpy as np
from fastapi.concurrency import run_in_threadpool

class SegmentRequest(BaseModel):
    customer_id: str
    recency_days: int = Field(..., ge=0)
    frequency: int = Field(..., ge=1)
    monetary_value: float = Field(..., gt=0)

class SegmentResponse(BaseModel):
    customer_id: str
    segment: str
    confidence: float
    model_version: str

SEGMENT_LABELS = ["at_risk", "regular", "loyal", "champion"]

@router.post("/predict/segment", response_model=SegmentResponse)
async def predict_segment(req: SegmentRequest):
    features = np.array(
        [[req.recency_days, req.frequency, req.monetary_value]],
        dtype=np.float32,                            # ← dtype must match training exactly
    )

    session = ml_models["segment"]
    input_name = session.get_inputs()[0].name

    # ONNX inference is blocking CPU work — keep it off the event loop
    outputs = await run_in_threadpool(session.run, None, {input_name: features})

    probs = outputs[0][0]
    idx = int(np.argmax(probs))
    return SegmentResponse(
        customer_id=req.customer_id,
        segment=SEGMENT_LABELS[idx],
        confidence=float(probs[idx]),
        model_version=settings.model_version,
    )
```

Three things worth copying:

- **`dtype=np.float32`.** PyTorch defaults to float32; NumPy defaults to float64. Hand ONNX the wrong one and you get a type error — or worse, silently wrong numbers. This is the number one model-serving bug ([[Model export and serving]]).
- **`run_in_threadpool`** keeps blocking CPU work off the event loop, per the async rule in [[FastAPI fundamentals]].
- **`model_version` in the response.** When a prediction looks wrong three weeks later, you need to know which model produced it. This costs one field and saves entire afternoons.

### Feature order — the silent killer

Your model was trained on columns in a specific order. If training was `[recency, frequency, monetary]` and you serve `[frequency, recency, monetary]`, **there is no error**. The shapes match. You just get confidently wrong answers forever.

**Defend against it:** save the feature order alongside the model and build from it.

```python
import numpy as np

FEATURE_ORDER = ["recency_days", "frequency", "monetary_value"]   # exported with the model

features = np.array([[getattr(req, f) for f in FEATURE_ORDER]], dtype=np.float32)
```

> Same applies to **scaling**. If you normalised features during training, you must apply the *identical* scaler at serving time — the one fitted on training data, saved and loaded, not refitted. Refitting on live data is a classic and very hard to spot mistake.

---

## 3. Health checks

Not decoration — the orchestrator uses these to decide whether to send you traffic.

```python
from fastapi import Depends, HTTPException
from sqlalchemy import text

@router.get("/health")
async def health():
    """Liveness: am I running? Must be instant and dependency-free."""
    return {"status": "ok"}

@router.get("/ready")
async def ready(db=Depends(get_db)):
    """Readiness: can I actually serve? Checks dependencies."""
    try:
        await db.execute(text("SELECT 1"))
    except Exception:
        raise HTTPException(503, "database unavailable")
    if "forecast" not in ml_models:
        raise HTTPException(503, "models not loaded")
    return {"status": "ready"}
```

> **Keep `/health` dependency-free.** If liveness checks the database, a brief database blip makes the platform kill and restart perfectly healthy containers — turning a small outage into a restart storm. Liveness = "is the process alive". Readiness = "should I get traffic".

---

## 4. Dockerizing

```dockerfile
FROM python:3.12-slim

# Microsoft ODBC Driver 18 — required by pyodbc, NOT in the base image
RUN apt-get update && apt-get install -y --no-install-recommends curl gnupg unixodbc \
 && curl -sSL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor -o /usr/share/keyrings/microsoft.gpg \
 && echo "deb [signed-by=/usr/share/keyrings/microsoft.gpg] https://packages.microsoft.com/debian/12/prod bookworm main" > /etc/apt/sources.list.d/mssql.list \
 && apt-get update && ACCEPT_EULA=Y apt-get install -y msodbcsql18 \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# dependencies first — layer caching ([[Dev environment - Git, Docker, CLI]])
COPY pyproject.toml uv.lock ./
RUN pip install --no-cache-dir uv && uv sync --frozen --no-dev

COPY api/ ./api/
COPY models/ ./models/

# don't run as root
RUN useradd --create-home appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8000

CMD ["uv", "run", "uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
```

Points that matter:

- **The ODBC driver block.** `pyodbc` fails at runtime with "Data source name not found" without it. This is the number one "works locally, breaks in Docker" failure for Azure SQL.
- **`--host 0.0.0.0`.** Non-negotiable.
- **`--workers 2`.** Each worker is a separate process with its **own copy of the model in memory**. A 500MB model × 4 workers = 2GB. Match workers to your container's memory and CPU, not to a number you saw online.
- **Non-root user.** Cheap, and every security review asks for it.
- **No `--reload`.** Development only.

```bash
docker build -t retail-api .
docker run -p 8000:8000 --env-file .env retail-api
curl http://localhost:8000/health
```

> Model files in the image or mounted at runtime? **In the image** for the capstone — it's simpler and guarantees the code and model versions match. Downside: a 500MB image and a rebuild per model update. Big models in production usually get downloaded from blob storage at startup instead.

---

## 5. Deploying to Azure Container Apps

**Container Apps vs. App Service:** Container Apps is the modern container-native option with **scale-to-zero** (free when nobody's calling it). Use it.

```bash
RG=retail-rg
LOC=uksouth
ACR=retailacr$RANDOM

# 1. container registry
az acr create --name $ACR --resource-group $RG --sku Basic --admin-enabled true
az acr login --name $ACR

# 2. build and push
docker build -t $ACR.azurecr.io/retail-api:v1 .
docker push $ACR.azurecr.io/retail-api:v1

# 3. environment (once — both api and dashboard live in it)
az containerapp env create --name retail-env --resource-group $RG --location $LOC

# 4. deploy
az containerapp create \
  --name retail-api --resource-group $RG --environment retail-env \
  --image $ACR.azurecr.io/retail-api:v1 \
  --registry-server $ACR.azurecr.io \
  --target-port 8000 --ingress external \
  --min-replicas 0 --max-replicas 3 \
  --cpu 1 --memory 2Gi \
  --secrets "db-url=<connection-string>" \
  --env-vars "DATABASE_URL=secretref:db-url" "MODEL_VERSION=v1"

az containerapp show --name retail-api --resource-group $RG \
  --query properties.configuration.ingress.fqdn -o tsv
```

That last command prints your public URL.

**Choices explained:**

| Flag | Why |
|---|---|
| `--ingress external` | Reachable from the internet. `internal` = only from inside the environment. |
| `--min-replicas 0` | Scales to zero → **£0 when idle**. Cost of a ~10s cold start on the first request. |
| `--min-replicas 1` | No cold start, but you pay continuously. Switch to this only for a demo. |
| `--secrets` + `secretref:` | Values stored encrypted by the platform, injected as env vars, never in the image |

### Updating

```bash
docker build -t $ACR.azurecr.io/retail-api:v2 .
docker push $ACR.azurecr.io/retail-api:v2
az containerapp update --name retail-api --resource-group $RG \
  --image $ACR.azurecr.io/retail-api:v2
```

> **Tag with versions, never rely on `:latest`.** With `:latest` you can't tell what's running and can't roll back. Container Apps keeps revisions, so `az containerapp revision list` + activating the previous one is your rollback — but only if the tags are distinguishable.

### Firewall

The container app needs to reach Azure SQL. Add the "allow Azure services" rule (`0.0.0.0 → 0.0.0.0`) from [[Azure SQL Database]], or set up a VNet with a private endpoint for a proper production setup.

### Secrets, properly

Better than `--secrets`: give the container app a **managed identity** and read from Key Vault ([[Azure fundamentals]]).

```bash
az containerapp identity assign --name retail-api --resource-group $RG --system-assigned

PRINCIPAL=$(az containerapp identity show --name retail-api --resource-group $RG --query principalId -o tsv)

az role assignment create --assignee $PRINCIPAL \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/<sub>/resourceGroups/$RG/providers/Microsoft.KeyVault/vaults/retail-kv-01
```

Then `DefaultAzureCredential()` in the app just works, and there is no secret anywhere in your config.

---

## 6. Logging

`print()` doesn't survive contact with production. Use structured logging so you can search it.

```python
import logging, json, sys, time

class JsonFormatter(logging.Formatter):
    def format(self, record):
        return json.dumps({
            "time": self.formatTime(record),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            **getattr(record, "extra_fields", {}),
        })

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JsonFormatter())
logging.basicConfig(handlers=[handler], level=settings.log_level)
```

```python
import time

@app.middleware("http")
async def log_requests(request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    duration_ms = (time.perf_counter() - start) * 1000
    logger.info(
        "request",
        extra={"extra_fields": {
            "path": request.url.path,
            "method": request.method,
            "status": response.status_code,
            "duration_ms": round(duration_ms, 1),
        }},
    )
    return response
```

```bash
az containerapp logs show --name retail-api --resource-group $RG --follow
```

> **Log to stdout, never to a file.** Containers are ephemeral; a log file dies with the container. The platform collects stdout. And never log secrets, tokens, or full request bodies containing personal data.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| `pyodbc`: "Data source name not found" in Docker | ODBC Driver 18 missing from the image | Add the apt block above |
| Container starts then dies | Model file missing, or bad env var | `az containerapp logs show` — the traceback is there |
| Works locally, times out in Azure | SQL firewall blocking the container app | Add the "allow Azure services" rule |
| First request after idle takes 10s | Scale-to-zero cold start (+ serverless SQL waking) | Expected. `--min-replicas 1` if it matters. |
| Random "connection reset" | Azure dropped an idle pooled connection | `pool_pre_ping=True`, `pool_recycle=1800` |
| "Too many connections" | Engine created per request | One module-level engine |
| Memory limit exceeded / OOMKilled | `--workers N` × model size > container memory | Fewer workers, or more memory |
| Predictions look plausible but wrong | Feature order or scaling differs from training | Pin `FEATURE_ORDER`; load the *saved* scaler |
| ONNX: "Unexpected input data type" | float64 vs float32 | `dtype=np.float32` |
| API slow under load | Blocking inference on the event loop | `run_in_threadpool` |
| Dashboard gets a CORS error | Origin not allowed | `CORSMiddleware` with the dashboard's URL |
| Can't tell what's deployed | Everything tagged `:latest` | Version your tags |
| Secrets visible in `az containerapp show` | Passed via `--env-vars` instead of `--secrets` | Use `--secrets` + `secretref:`, or Key Vault |

---

## Practice checklist

- [ ] One engine at module level; sessions per request via `Depends`
- [ ] Async SQLAlchemy against Azure SQL — session management, connection pooling, `pool_pre_ping`
- [ ] Parameterised queries, never f-strings
- [ ] Raw SQL vs. ORM, and why raw SQL is the right call for a read-only serving API
- [ ] Loading a trained model once at startup (`lifespan` events) instead of per-request
- [ ] Serving predictions, with input validation matching the model's expected shape
- [ ] **Feature order and dtype contracts** — the silent wrong-answer bug
- [ ] `run_in_threadpool` for blocking inference
- [ ] Liveness vs. readiness health checks, and why liveness must be dependency-free
- [ ] Containerizing a FastAPI app with Docker, including the ODBC driver
- [ ] Deploying to Azure Container Apps; scale-to-zero and the cold-start trade-off
- [ ] Environment variables and secrets in the cloud, not the image
- [ ] Structured logging to stdout

## Hands-on

- [ ] Wire an endpoint to Azure SQL and return real query results
- [ ] Load a trained model at startup and serve predictions from a `/predict` endpoint
- [ ] Add `/health` and `/ready`, and make `/ready` fail by stopping the database
- [ ] Containerize the app; confirm it works with no local Python
- [ ] Deploy to Azure Container Apps; confirm it's reachable over the internet
- [ ] Deploy a `v2` tag, then roll back to `v1` by activating the previous revision
- [ ] Deliberately shuffle your feature order and observe that nothing errors — this is the lesson

## Resources

- [FastAPI: SQL (Relational) Databases](https://fastapi.tiangolo.com/tutorial/sql-databases/)
- [FastAPI: Lifespan events](https://fastapi.tiangolo.com/advanced/events/)
- [Azure Container Apps overview](https://learn.microsoft.com/azure/container-apps/overview)
- [SQLAlchemy asyncio docs](https://docs.sqlalchemy.org/en/latest/orm/extensions/asyncio.html)

## Next

[[Streamlit vs Django]]
