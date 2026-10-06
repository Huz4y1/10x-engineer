An index is a lookup structure the database keeps on the side so it doesn't have to scan every row.

Creating one

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- speeds up anything filtering or joining on user_id
SELECT * FROM orders WHERE user_id = 1;

CREATE UNIQUE INDEX idx_users_email ON users(email);

-- doesn't lock the table while it builds, use this on a live database
CREATE INDEX CONCURRENTLY idx_orders_created_at ON orders(created_at);

DROP INDEX idx_orders_created_at;
```

When it helps and when it costs you

|Situation|Worth it?|
|---|---|
|Column used in WHERE or JOIN a lot|yes|
|Foreign key columns like orders.user_id|yes, Postgres doesn't index these for you|
|Column you ORDER BY constantly|yes|
|Table with heavy INSERT/UPDATE traffic|every index makes writes slower|
|Small table, a few hundred rows|no, a full scan is already fast|
|Column with only 2 or 3 distinct values|usually not, the planner ignores it anyway|

PRIMARY KEY and UNIQUE already create an index, no need to add your own

```sql
-- users.id and users.email are already indexed by their constraints
```

Composite index, column order matters

```sql
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at);

-- used
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 1 ORDER BY created_at DESC;

-- NOT used, it's a left-to-right prefix thing
SELECT * FROM orders WHERE created_at > '2026-07-01';
```

Checking it's actually being used

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 1;

-- Seq Scan on orders   -> reading the whole table, index not used
-- Index Scan using idx_orders_user_id -> good, it's being used
```

Things that stop an index being used

```sql
-- wrapping the column in a function
SELECT * FROM users WHERE LOWER(email) = 'sam@mail.com';

-- fix it with an expression index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- leading wildcard, nothing to anchor on
SELECT * FROM users WHERE email LIKE '%mail.com';
```

---

**Going deeper:** [[SQL fundamentals]] - what stops an index being used. [[PostgreSQL reference]] - index types and `EXPLAIN`.
