---
tags: [postgresql, sqlalchemy, python, databases]
---

# PostgreSQL with SQLAlchemy

**Using PostgreSQL from Python.** General SQLAlchemy — engines, sessions, the ORM, Alembic — is in [[SQLAlchemy]]. This note is the **Postgres-specific** part: connecting (locally and in the cloud), Postgres-only column types, upserts, bulk loading and async.

Hub: [[PostgreSQL]] · Types: [[PostgreSQL data types]] · SQL: [[PostgreSQL SQL]] · Flask version: [[Flask - connecting to a SQL database]]

> ✅ **Every Python example here was run, in order, against PostgreSQL 18** with SQLAlchemy 2.0, psycopg 3 and asyncpg.

---

## Install

```bash
uv add sqlalchemy "psycopg[binary]"     # the normal driver
uv add asyncpg                          # only if you use async (FastAPI with async def)
```

> ⚠️ **Use `psycopg[binary]`, not plain `psycopg`**, especially on Windows — the plain version tries to compile against a local Postgres install ([[SQLAlchemy]]).

| Driver | URL starts with | Use |
|---|---|---|
| **psycopg 3** | `postgresql+psycopg://` | ✅ **The default** — scripts, Flask, Django, Streamlit, pandas |
| **asyncpg** | `postgresql+asyncpg://` | Async code — FastAPI with `async def` |
| psycopg2 | `postgresql+psycopg2://` | Older projects. If a URL says just `postgresql://`, SQLAlchemy picks **psycopg2** |

> ⚠️ **`postgresql://` on its own means psycopg2**, not psycopg 3. If you installed `psycopg` and get *No module named 'psycopg2'*, add `+psycopg` to the URL.

---

## Connecting

```python
import os
from sqlalchemy import create_engine, text

DATABASE_URL = os.environ.get(
    "DATABASE_URL",
    "postgresql+psycopg://postgres:secret@localhost:5432/shop",   # user:password@host:port/database
)

engine = create_engine(
    DATABASE_URL,
    pool_pre_ping=True,     # test each connection before using it - survives dropped connections
    pool_size=5,            # connections kept open
    max_overflow=10,        # extra connections allowed at busy moments
)

with engine.connect() as conn:
    print(conn.execute(text("SELECT version()")).scalar())
```

**The URL, piece by piece:**

```
postgresql+psycopg://postgres:secret@localhost:5432/shop
└──────┬─────────┘  └──┬───┘ └──┬─┘ └───┬───┘ └┬─┘ └┬─┘
 database+driver     user   password  host   port  database
```

> ⚠️ **A password containing `@`, `:` or `/` breaks the URL.** Build it safely instead of typing it:

```python
from sqlalchemy import URL

url = URL.create(
    "postgresql+psycopg",
    username="postgres",
    password="p@ss:w/rd",          # any characters - URL.create escapes them
    host="localhost",
    port=5432,
    database="shop",
)
```

> **Never write the password in your code.** Read it from an environment variable or a `.env` file ([[Flask - project structure and blueprints]], [[Security in practice]]).

### Connecting to Azure or AWS

The only differences are the host name, the username, and **requiring encryption**:

```python
AZURE_URL = (
    "postgresql+psycopg://appuser:PASSWORD"
    "@myserver.postgres.database.azure.com:5432/shop"
    "?sslmode=require"                       # encrypt the connection - required in the cloud
)

RDS_URL = (
    "postgresql+psycopg://appuser:PASSWORD"
    "@mydb.abc123xyz.eu-west-2.rds.amazonaws.com:5432/shop"
    "?sslmode=require"
)
```

Setting up those servers — firewalls, users, getting the host name — is in [[Azure Database for PostgreSQL]] and [[AWS RDS for PostgreSQL]].

---

## Raw SQL — the simplest way

You don't have to use the ORM. For scripts and data work, plain SQL through `text()` is often clearest:

```python
from sqlalchemy import text

with engine.begin() as conn:                   # begin() = a transaction that COMMITS at the end
    conn.execute(text("""
        CREATE TABLE IF NOT EXISTS readings (
            id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
            device_id   text NOT NULL,
            temp_c      double precision NOT NULL,
            recorded_at timestamptz NOT NULL DEFAULT now()
        )
    """))

    conn.execute(
        text("INSERT INTO readings (device_id, temp_c) VALUES (:device, :temp)"),
        [{"device": "esp32-1", "temp": 21.5},          # a LIST of dicts = many rows in one call
         {"device": "esp32-1", "temp": 22.0},
         {"device": "esp32-2", "temp": 19.8}],
    )

with engine.connect() as conn:                 # connect() = read only, nothing to commit
    rows = conn.execute(
        text("SELECT device_id, avg(temp_c) AS avg_temp FROM readings "
             "WHERE temp_c > :min GROUP BY device_id ORDER BY device_id"),
        {"min": 0},
    ).all()

for row in rows:
    print(row.device_id, round(row.avg_temp, 2))    # columns by name
```

```
esp32-1 21.75
esp32-2 19.8
```

| Use | When |
|---|---|
| `engine.begin()` | You're **changing** data — commits at the end, rolls back if anything fails |
| `engine.connect()` | You're **reading** — or you'll call `conn.commit()` yourself |

> ⚠️ **Always pass values as `:name` parameters, never with f-strings.** `text(f"... WHERE device_id = '{device}'")` is **SQL injection** — a device name of `x' OR '1'='1` returns every row ([[Security in practice]]).

> ⚠️ **`engine.connect()` does not commit.** Inserting inside `with engine.connect()` and not calling `conn.commit()` means the rows vanish when the block ends. Use `engine.begin()` for writes.

---

## ORM models with Postgres-only types

The standard types (`String`, `Integer`, `Numeric`…) work on every database. For Postgres features, import from `sqlalchemy.dialects.postgresql`:

```python
import uuid
from datetime import datetime
from decimal import Decimal
from sqlalchemy import String, Numeric, DateTime, func, text
from sqlalchemy.dialects.postgresql import JSONB, ARRAY, UUID
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Product(Base):
    __tablename__ = "products"

    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), primary_key=True,
        server_default=text("uuidv7()"),            # Postgres generates it (PG 18; use gen_random_uuid() before)
    )
    sku: Mapped[str] = mapped_column(String(40), unique=True)
    name: Mapped[str]                               # plain str -> text column
    price: Mapped[Decimal] = mapped_column(Numeric(10, 2))            # money: exact
    tags: Mapped[list[str]] = mapped_column(ARRAY(String), server_default="{}")
    attrs: Mapped[dict] = mapped_column(JSONB, server_default="{}")
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),                    # -> timestamptz
        server_default=func.now(),                  # the DATABASE sets the time
    )


Base.metadata.create_all(engine)                    # create the table (use Alembic for real projects)
```

| Postgres type | SQLAlchemy | Python value |
|---|---|---|
| `timestamptz` | `DateTime(timezone=True)` | `datetime` **with** a time zone |
| `numeric(10,2)` | `Numeric(10, 2)` | `Decimal` |
| `uuid` | `UUID(as_uuid=True)` | `uuid.UUID` |
| `jsonb` | `JSONB` | `dict` / `list` |
| `text[]` | `ARRAY(String)` | `list[str]` |
| `text` | `Mapped[str]` with no length | `str` |

> ⚠️ **`DateTime()` without `timezone=True` creates a plain `timestamp`** — no time zone, and every problem from [[PostgreSQL data types]]. Always pass `timezone=True`.

> **`server_default` vs `default`:** `server_default=func.now()` makes the **database** fill it in, so rows inserted by other tools or raw SQL get it too. `default=...` only works when inserting through SQLAlchemy.

### Using the model

```python
from sqlalchemy import select
from sqlalchemy.orm import Session

with Session(engine) as session:
    session.add_all([
        Product(sku="MUG-01", name="Mug", price=Decimal("8.50"),
                tags=["kitchen"], attrs={"colour": "blue"}),
        Product(sku="LAMP-01", name="Desk lamp", price=Decimal("34.00"),
                tags=["office", "lighting"], attrs={"colour": "black", "watts": 9}),
    ])
    session.commit()

    lamp = session.scalars(select(Product).where(Product.sku == "LAMP-01")).one()
    print(lamp.id, lamp.price, lamp.tags, lamp.attrs["watts"], lamp.created_at.tzinfo is not None)
```

---

## Querying JSONB and arrays

```python
with Session(engine) as session:
    # JSONB: attrs ->> 'colour' = 'black'
    black = session.scalars(
        select(Product.sku).where(Product.attrs["colour"].astext == "black")
    ).all()

    # JSONB contains: attrs @> '{"watts": 9}'
    nine_watt = session.scalars(
        select(Product.sku).where(Product.attrs.contains({"watts": 9}))
    ).all()

    # array contains: tags @> ARRAY['office']
    office = session.scalars(
        select(Product.sku).where(Product.tags.contains(["office"]))
    ).all()

    # any element equals: 'kitchen' = ANY(tags)
    kitchen = session.scalars(
        select(Product.sku).where(Product.tags.any("kitchen"))
    ).all()

print(black, nine_watt, office, kitchen)
```

```
['LAMP-01'] ['LAMP-01'] ['LAMP-01'] ['MUG-01']
```

| You want | SQLAlchemy | SQL it becomes |
|---|---|---|
| A key's value as text | `Product.attrs["colour"].astext` | `attrs ->> 'colour'` |
| A key's value as JSON | `Product.attrs["colour"]` | `attrs -> 'colour'` |
| JSON contains | `Product.attrs.contains({...})` | `attrs @> '{...}'` |
| Has a key | `Product.attrs.has_key("watts")` | `attrs ? 'watts'` |
| Array contains all | `Product.tags.contains([...])` | `tags @> ARRAY[...]` |
| Array has this value | `Product.tags.any("x")` | `'x' = ANY(tags)` |

> ⚠️ **Changing a key inside a JSONB value in Python isn't noticed.** `lamp.attrs["watts"] = 12` then `commit()` saves nothing, because the dict object is the same one. Assign a new dict — `lamp.attrs = {**lamp.attrs, "watts": 12}` — or wrap the column type in `MutableDict.as_mutable(JSONB)`.

---

## Upserts — `ON CONFLICT` from Python

Postgres's insert-or-update ([[PostgreSQL SQL]]) through SQLAlchemy. Note the `insert` comes from the **postgresql** dialect:

```python
from sqlalchemy.dialects.postgresql import insert

incoming = [
    {"sku": "MUG-01", "name": "Mug", "price": Decimal("9.00")},          # exists -> update
    {"sku": "PEN-01", "name": "Pen", "price": Decimal("1.20")},          # new -> insert
]

stmt = insert(Product).values(incoming)
stmt = stmt.on_conflict_do_update(
    index_elements=["sku"],                                   # the UNIQUE column that clashes
    set_={"name": stmt.excluded.name, "price": stmt.excluded.price},   # excluded = the incoming row
).returning(Product.sku, Product.price)

with engine.begin() as conn:
    print(conn.execute(stmt).all())
```

```
[('MUG-01', Decimal('9.00')), ('PEN-01', Decimal('1.20'))]
```

**Insert only the new ones:**

```python
stmt = insert(Product).values(sku="MUG-01", name="Mug", price=Decimal("1.00"))
stmt = stmt.on_conflict_do_nothing(index_elements=["sku"])     # MUG-01 exists: skipped, no error
stmt = stmt.returning(Product.sku)                             # hand back what WAS inserted

with engine.begin() as conn:
    print("inserted:", conn.execute(stmt).scalars().all())
```

```
inserted: []
```

> ⚠️ **Don't rely on `result.rowcount` to count what an upsert did** — for this statement it came back as `-1` ("not available"), not `0`. Use `.returning(...)` and count the rows you get back.

> ⚠️ **`from sqlalchemy import insert` doesn't have `on_conflict_do_update`.** You need `from sqlalchemy.dialects.postgresql import insert` — the error *'Insert' object has no attribute 'on_conflict_do_update'* means you imported the generic one.

---

## Loading lots of rows fast

### From pandas

```python
import pandas as pd

df = pd.DataFrame({
    "device_id": ["esp32-3"] * 1000,
    "temp_c": [20 + i / 100 for i in range(1000)],
})

df.to_sql(
    "readings", engine,
    if_exists="append",       # add to the existing table ("replace" DROPS it first - careful)
    index=False,              # don't write the pandas index as a column
    method="multi",           # many rows per INSERT - much faster than one at a time
    chunksize=500,            # rows per INSERT statement
)

back = pd.read_sql("SELECT device_id, count(*) AS n FROM readings GROUP BY device_id ORDER BY device_id", engine)
print(back)
```

```
  device_id     n
0   esp32-1     2
1   esp32-2     1
2   esp32-3  1000
```

> ⚠️ **`if_exists="replace"` drops the table and recreates it from the DataFrame** — losing your column types, primary key, indexes and constraints. Use `"append"` into a table you created properly.

### Really big loads — `COPY`

For hundreds of thousands of rows, Postgres's `COPY` is far faster than any `INSERT`. psycopg 3 exposes it directly:

```python
rows = [("esp32-4", 18.0 + i / 10) for i in range(10_000)]

with engine.begin() as conn:
    raw = conn.connection.driver_connection         # the underlying psycopg connection
    with raw.cursor() as cur:
        with cur.copy("COPY readings (device_id, temp_c) FROM STDIN") as copy:
            for row in rows:
                copy.write_row(row)                  # streams straight into the table

with engine.connect() as conn:
    print(conn.execute(text("SELECT count(*) FROM readings WHERE device_id = 'esp32-4'")).scalar())
```

```
10000
```

| Method | Rough speed | Use |
|---|---|---|
| `session.add()` in a loop | Slowest | A few rows |
| `conn.execute(text(...), [list of dicts])` | Good | Hundreds to thousands |
| `df.to_sql(method="multi")` | Good | DataFrames |
| **`COPY`** | **Fastest** | Hundreds of thousands and up |

---

## Reading big results without running out of memory

```python
with engine.connect() as conn:
    result = conn.execution_options(stream_results=True, yield_per=2000).execute(
        text("SELECT device_id, temp_c FROM readings")
    )
    total = 0
    for partition in result.partitions():           # 2,000 rows at a time
        total += len(partition)
print(total, "rows read in chunks")
```

> **`stream_results=True` uses a server-side cursor** — rows arrive in batches instead of all at once. Without it, `SELECT *` on a 50-million-row table tries to load all 50 million into Python.

---

## Async — for FastAPI

```python
import asyncio
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

ASYNC_URL = DATABASE_URL.replace("+psycopg", "+asyncpg")   # same database, async driver

async def main():
    async_engine = create_async_engine(ASYNC_URL, pool_pre_ping=True)
    SessionLocal = async_sessionmaker(async_engine, expire_on_commit=False)

    async with SessionLocal() as session:
        products = (await session.scalars(select(Product).order_by(Product.sku))).all()
        print([p.sku for p in products])

    await async_engine.dispose()                    # close the pool cleanly

asyncio.run(main())
```

```
['LAMP-01', 'MUG-01', 'PEN-01']
```

> ⚠️ **asyncpg uses `ssl=require`, not `sslmode=require`** in the URL. Copying a psycopg URL with `?sslmode=require` to asyncpg gives *unexpected keyword argument 'sslmode'* — change it to `?ssl=require`.

> ⚠️ **Async only helps inside async code.** In a normal script, Flask or Streamlit, use the plain engine — async adds complexity for no gain there ([[FastAPI reference]]).

---

## Connection pooling in the cloud

Every open connection uses memory on the Postgres server, and cloud databases have a **maximum number of connections** (smaller servers allow fewer).

| Setting | Meaning | Guidance |
|---|---|---|
| `pool_size` | Connections each app process keeps open | 5 is plenty for most apps |
| `max_overflow` | Extra connections at busy moments | 10 |
| `pool_pre_ping=True` | Test a connection before using it | ✅ Always — cloud networks drop idle connections |
| `pool_recycle=1800` | Replace connections older than 30 min | Useful behind load balancers that cut long connections |

> ⚠️ **Connections multiply.** 4 Gunicorn workers × (5 + 10) = up to 60 connections from **one** app. Add a second app and a dashboard and a small cloud server runs out: *FATAL: remaining connection slots are reserved*. Put **PgBouncer** in front — built into Azure Flexible Server, and **RDS Proxy** on AWS ([[Azure Database for PostgreSQL]], [[AWS RDS for PostgreSQL]]).

---

## Common mistakes

| Mistake | What you see | Fix |
|---|---|---|
| `postgresql://` with only psycopg 3 installed | *No module named 'psycopg2'* | `postgresql+psycopg://` |
| Password with `@` or `:` in the URL | Wrong host / auth failure | `URL.create(...)` |
| Writing inside `engine.connect()` | Rows disappear | `engine.begin()`, or `conn.commit()` |
| f-string SQL | **SQL injection** | `:name` parameters |
| `DateTime()` without `timezone=True` | Naive `timestamp` column | `DateTime(timezone=True)` |
| Generic `insert` for upserts | *no attribute 'on_conflict_do_update'* | `sqlalchemy.dialects.postgresql.insert` |
| Editing a JSONB dict in place | Change not saved | Assign a new dict, or `MutableDict` |
| `to_sql(if_exists="replace")` | Keys and indexes lost | `if_exists="append"` |
| `SELECT *` on a huge table | Memory error | `stream_results=True` |
| `sslmode=require` with asyncpg | *unexpected keyword argument* | `ssl=require` |
| Too many connections | *remaining connection slots are reserved* | Smaller pools, PgBouncer / RDS Proxy |

## Related

[[PostgreSQL]] · [[SQLAlchemy]] · [[PostgreSQL SQL]] · [[PostgreSQL data types]] · [[Azure Database for PostgreSQL]] · [[AWS RDS for PostgreSQL]] · [[Flask - connecting to a SQL database]] · [[FastAPI reference]] · [[pandas]] · [[Security in practice]]
