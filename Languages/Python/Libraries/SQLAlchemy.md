---
tags: [python, library, sqlalchemy, database]
---

# SQLAlchemy

Database access, connection pooling, and an optional ORM.

Index: [[Python libraries]] · SQL: [[SQL fundamentals]] · [[PostgreSQL reference]]

> **Using it with PostgreSQL** — connection URLs for Azure/AWS, `JSONB`/`ARRAY`/`UUID` columns, upserts, `COPY`, async: **[[PostgreSQL with SQLAlchemy]]**

### Installing

> **Same commands on Windows, WSL and Linux.** The differences are called out below where they exist.

```bash
uv add sqlalchemy "psycopg[binary]"                      # in a uv project - PREFERRED
uv pip install sqlalchemy "psycopg[binary]"              # into the active venv, no project file
pip install sqlalchemy "psycopg[binary]"                 # plain pip, if you're not using uv
```

**If you don't have a project yet:**

```bash
uv init myproject && cd myproject     # creates pyproject.toml and .venv
uv add sqlalchemy "psycopg[binary]"
uv run python -c "import sqlalchemy; print(sqlalchemy.__version__)"           # uv run uses the project venv automatically
```

**Without uv — a plain virtual environment:**

| | Windows (PowerShell) | WSL / Linux |
|---|---|---|
| Create | `python -m venv .venv` | `python3 -m venv .venv` |
| Activate | `.venv\Scripts\Activate.ps1` | `source .venv/bin/activate` |
| Install | `pip install sqlalchemy "psycopg[binary]"` | `pip install sqlalchemy "psycopg[binary]"` |
| Leave | `deactivate` | `deactivate` |

**Check it worked:**

```bash
python -c "import sqlalchemy; print(sqlalchemy.__version__)"
```

**Pick the driver for your database:**

```bash
uv add sqlalchemy "psycopg[binary]"      # Postgres, sync
uv add sqlalchemy asyncpg                # Postgres, async
uv add sqlalchemy aioodbc pyodbc         # Azure SQL / SQL Server
uv add sqlalchemy pymysql                # MySQL / MariaDB
# SQLite needs no driver - it's in the standard library
```

> ⚠️ **Use `psycopg[binary]`, not plain `psycopg`.** The `[binary]` extra ships a prebuilt libpq. Without it, pip tries to compile against a local Postgres install — which fails on a clean Windows box, and needs `sudo apt install libpq-dev python3-dev gcc` on WSL.

> ⚠️ **`pyodbc` needs a system ODBC driver, which pip does not install.**
> - **Windows:** `winget install Microsoft.msodbcsql.18`
> - **WSL/Ubuntu:** follow Microsoft's apt repository steps for `msodbcsql18`, then `sudo apt install unixodbc-dev`
> Without it you get *Can't open lib 'ODBC Driver 18 for SQL Server'*.

> ⚠️ **Never `pip install` outside a virtual environment.** On WSL and Linux it fights the system Python; newer versions refuse outright with *externally-managed-environment*. On Windows it silently pollutes your global install. `uv` makes a venv for you — use it.

> **On Windows: if `python` opens the Microsoft Store**, Python isn't really installed. Get it from python.org (tick *Add to PATH*), or `winget install Python.Python.3.12`.

> **In WSL: work in `~/`, not `/mnt/c/`.** Installing into a project on the Windows drive is dramatically slower, and file-watching (`--reload`, `pytest-watch`) often doesn't fire at all. See [[Setting up a dev machine]].

---

## Why use it even if you write raw SQL

**The connection pool.** Opening a database connection costs 50-200 ms (TCP + TLS + auth). A pool opens N once and lends them out in microseconds.

```python
from sqlalchemy import create_engine, text

engine = create_engine(                   # MODULE LEVEL - created ONCE
    "postgresql+psycopg://user:pw@host:5432/db",
    pool_size=5,
    max_overflow=10,
    pool_recycle=1800,                    # cloud DBs drop idle connections
    pool_pre_ping=True,                   # test before handing out
    echo=False,                           # True logs every query
)
```

> ⚠️ **One engine, at module level.** Creating it per request creates a pool per request and exhausts the database's connection limit in minutes.

> ⚠️ **`pool_pre_ping=True` and `pool_recycle` are mandatory on cloud databases.** Azure and AWS silently drop idle connections; without these you get random "connection reset" errors after quiet periods ([[Azure SQL Database]]).

## Raw SQL — usually what you want

```python
from sqlalchemy import text

with engine.connect() as conn:
    rows = conn.execute(
        text("SELECT date, revenue FROM daily_sales WHERE category = :cat"),
        {"cat": "Kitchen"},               # NAMED PARAMS - never f-strings
    ).fetchall()

    for r in rows:
        print(r.date, r.revenue)          # attribute access
    dicts = [dict(r._mapping) for r in rows]

with engine.begin() as conn:              # transaction - commits on exit
    conn.execute(text("UPDATE t SET x = :x"), {"x": 1})
```

> ⚠️ **Never f-string user input into SQL.** `f"WHERE cat = '{cat}'"` is a SQL injection hole. Parameters are also faster — the plan gets cached ([[26 — SECURITY]]).

## Async

```python
from sqlalchemy import text
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

engine = create_async_engine("postgresql+asyncpg://...", pool_pre_ping=True)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_sales(category: str):
    async with SessionLocal() as session:
        result = await session.execute(
            text("SELECT * FROM sales WHERE category = :c"), {"c": category})
        return [dict(r._mapping) for r in result]
```

> **Match the driver to the endpoint.** A sync driver inside an `async def` blocks the event loop — worse than being fully synchronous ([[FastAPI reference]]).

## The ORM

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from sqlalchemy import ForeignKey, String, Numeric
from decimal import Decimal

class Base(DeclarativeBase): pass

class Customer(Base):
    __tablename__ = "customers"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    country: Mapped[str | None]
    orders: Mapped[list["Order"]] = relationship(back_populates="customer")

class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)
    customer_id: Mapped[int] = mapped_column(ForeignKey("customers.id"))
    total: Mapped[Decimal] = mapped_column(Numeric(18, 2))     # money -> Numeric
    customer: Mapped["Customer"] = relationship(back_populates="orders")

Base.metadata.create_all(engine)          # dev only - use migrations in prod
```

## Querying with the ORM

```python
from sqlalchemy.orm import Session
from sqlalchemy import select, func

with Session(engine) as session:
    stmt = select(Customer).where(Customer.country == "UK").order_by(Customer.name).limit(10)
    customers = session.scalars(stmt).all()

    one = session.get(Customer, 1)

    stmt = (select(Customer.country, func.sum(Order.total).label("total"))
            .join(Order)
            .group_by(Customer.country)
            .having(func.sum(Order.total) > 1000))
    for country, total in session.execute(stmt):
        print(country, total)

    session.add(Customer(name="New"))
    session.commit()
```

### The N+1 problem

```python
from sqlalchemy import select

for c in session.scalars(select(Customer)):     # 1 query
    print(c.orders)                              # +1 query PER customer = N+1

from sqlalchemy.orm import selectinload, joinedload
stmt = select(Customer).options(selectinload(Customer.orders))   # 2 queries total
```

> **N+1 is the classic ORM performance bug.** It looks like clean code and issues a thousand queries. `selectinload` for collections, `joinedload` for single relations.

## ORM or raw SQL?

| Use the ORM | Use raw SQL |
|---|---|
| CRUD-heavy app, objects with lifecycles | Read-only analytics/serving |
| You want relationships managed | Complex aggregations and windows |
| Migrations from models | You already have the query from SQL work |

> **For the serving APIs in this vault, raw SQL via `text()` is the right call** — the gold tables are read-only aggregates, and the queries came from your Spark work. Use SQLAlchemy for the pool, not the ORM ([[FastAPI data and deployment]]).

## Migrations — Alembic

```bash
uv add alembic
alembic init migrations
alembic revision --autogenerate -m "create sales"
alembic upgrade head
alembic downgrade -1
alembic current · alembic history
```

> **Migrations must be backwards compatible with the running app** — during a deploy both versions are live. Add a nullable column, deploy code using it, backfill, then add the constraint: **expand → migrate → contract** ([[CI-CD pipelines]]).

## Connection URLs

| Database | URL |
|---|---|
| Postgres (sync) | `postgresql+psycopg://user:pw@host:5432/db` |
| Postgres (async) | `postgresql+asyncpg://user:pw@host:5432/db` |
| Azure SQL | `mssql+pyodbc://...?driver=ODBC+Driver+18+for+SQL+Server` |
| SQLite | `sqlite:///file.db` |
| DuckDB | `duckdb:///file.db` |

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `too many connections` | Engine per request | One module-level engine |
| Random "connection reset" | Idle connection dropped | `pool_pre_ping=True`, `pool_recycle=1800` |
| `QueuePool limit reached` | Sessions not closed | Use `with` / a `yield` dependency |
| Very slow list page | N+1 queries | `selectinload` |
| `ObjectDeletedError` after commit | Object expired | `expire_on_commit=False` |
| Async endpoint blocks | Sync driver in async code | Use `asyncpg` / `aioodbc` |
| Money off by pennies | `Float` column | `Numeric(18,2)` |

## Related

[[Python libraries]] · [[FastAPI reference]] · [[PostgreSQL reference]] · [[Azure SQL Database]] · [[SQL fundamentals]] · [[Data modeling]]
