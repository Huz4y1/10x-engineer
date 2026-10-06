The types you actually end up using in Postgres:

|Type|Meaning|
|---|---|
|SERIAL|Auto-incrementing integer, good for ids|
|BIGSERIAL|Same but 64-bit, use this if the table might get big|
|INTEGER|Whole number|
|BIGINT|Large whole number|
|TEXT|String of any length|
|VARCHAR(n)|String with a max length|
|BOOLEAN|TRUE/FALSE|
|NUMERIC(10,2)|Exact decimal, use for money|
|DOUBLE PRECISION|Float, don't use for money|
|TIMESTAMPTZ|Date + time + timezone|
|DATE|Just the date|
|UUID|Random unique id|
|JSONB|JSON stored in a binary, queryable form|

A table using them

```sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       TEXT NOT NULL,
    username    VARCHAR(30),
    age         INTEGER,
    balance     NUMERIC(10, 2),   -- 10 digits total, 2 after the point
    is_active   BOOLEAN,
    created_at  TIMESTAMPTZ       -- always store timezone-aware times
);
```

NUMERIC vs floats

```sql
-- floats lose precision, this does not give exactly 0.30
SELECT 0.1::DOUBLE PRECISION + 0.2::DOUBLE PRECISION;

-- numeric is exact, this is what you want for prices
SELECT 0.1::NUMERIC + 0.2::NUMERIC;
```

---

**Going deeper:** [[PostgreSQL reference]] - type table, and why money is `DECIMAL` not `FLOAT` ([[Data modeling]]).
