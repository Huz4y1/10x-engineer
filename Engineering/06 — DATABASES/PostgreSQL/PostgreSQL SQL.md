---
tags: [postgresql, sql, databases, reference]
---

# PostgreSQL SQL

**The SQL that's special to PostgreSQL** — the features other databases don't have or do differently. Standard SQL (joins, `GROUP BY`, window functions) is in [[SQL fundamentals]]; this note assumes you know that and shows what Postgres adds on top.

Hub: [[PostgreSQL]] · Types: [[PostgreSQL data types]] · Quick lookup: [[PostgreSQL reference]] · From Python: [[PostgreSQL with SQLAlchemy]]

> ✅ **Every example here was run, top to bottom, on PostgreSQL 18.** Outputs shown are real. Paste the setup below into [[pgAdmin 4]]'s Query Tool (or `psql`) and follow along.

---

## The example database

Everything below uses this small shop. Run it once:

```sql
CREATE TABLE customers (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email       text NOT NULL UNIQUE,
    name        text NOT NULL,
    country     text NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE products (
    id     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku    text NOT NULL UNIQUE,
    name   text NOT NULL,
    price  numeric(10, 2) NOT NULL CHECK (price >= 0),
    stock  integer NOT NULL DEFAULT 0,
    tags   text[] NOT NULL DEFAULT '{}',
    attrs  jsonb NOT NULL DEFAULT '{}'
);

CREATE TABLE orders (
    id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id  bigint NOT NULL REFERENCES customers(id),
    status       text NOT NULL DEFAULT 'pending'
                 CHECK (status IN ('pending', 'paid', 'shipped', 'cancelled')),
    ordered_at   timestamptz NOT NULL
);

CREATE TABLE order_items (
    order_id    bigint NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id  bigint NOT NULL REFERENCES products(id),
    qty         integer NOT NULL CHECK (qty > 0),
    unit_price  numeric(10, 2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);

INSERT INTO customers (email, name, country) VALUES
  ('ada@example.com',   'Ada',   'GB'),
  ('linus@example.com', 'Linus', 'FI'),
  ('grace@example.com', 'Grace', 'US');

INSERT INTO products (sku, name, price, stock, tags, attrs) VALUES
  ('MUG-01',  'Mug',        8.50, 40, '{kitchen}',          '{"colour": "blue", "dishwasher": true}'),
  ('LAMP-01', 'Desk lamp', 34.00, 12, '{office,lighting}',  '{"colour": "black", "watts": 9}'),
  ('PEN-01',  'Pen',        1.20, 500, '{office}',          '{"colour": "black"}'),
  ('KETTLE-01', 'Old kettle', 25.00, 0, '{kitchen}',        '{"colour": "white"}');

INSERT INTO orders (customer_id, status, ordered_at) VALUES
  (1, 'paid',    '2026-03-01 10:00+00'),
  (1, 'shipped', '2026-03-05 09:30+00'),
  (2, 'paid',    '2026-03-05 14:00+00'),
  (3, 'pending', '2026-03-08 16:45+00');

INSERT INTO order_items (order_id, product_id, qty, unit_price) VALUES
  (1, 1, 2,  8.50), (1, 3, 10, 1.20),
  (2, 2, 1, 34.00),
  (3, 1, 1,  8.50), (3, 2, 1, 34.00),
  (4, 3, 5,  1.20);
```

---

## `RETURNING` — get back what you just changed

Most databases make you run a second `SELECT` to see the row you inserted. Postgres gives it back straight away:

```sql
INSERT INTO customers (email, name, country)
VALUES ('alan@example.com', 'Alan', 'GB')
RETURNING id, created_at;
```

It works on `UPDATE` and `DELETE` too:

```sql
UPDATE products SET price = price * 1.10
WHERE 'office' = ANY(tags)
RETURNING sku, price;
```

```
   sku   | price
---------+-------
 LAMP-01 | 37.40
 PEN-01  |  1.32
```

> **This is how your app learns a new row's ID** — `INSERT ... RETURNING id` in one round trip. SQLAlchemy uses it under the hood ([[PostgreSQL with SQLAlchemy]]).

---

## Upsert — insert, or update if it already exists

"Upsert" = **up**date or in**sert**. You try to insert; if a row with that key already exists, you update it instead.

```sql
INSERT INTO products (sku, name, price, stock)
VALUES ('MUG-01', 'Mug', 9.00, 55)                -- MUG-01 already exists
ON CONFLICT (sku) DO UPDATE                       -- conflict on the UNIQUE sku column
SET price = EXCLUDED.price,                       -- EXCLUDED = the row you tried to insert
    stock = EXCLUDED.stock
RETURNING sku, price, stock;
```

```
  sku   | price | stock
--------+-------+-------
 MUG-01 |  9.00 |    55
```

**Insert only if it's new — otherwise do nothing:**

```sql
INSERT INTO customers (email, name, country)
VALUES ('ada@example.com', 'Ada L.', 'GB')
ON CONFLICT (email) DO NOTHING;                   -- Ada exists: nothing happens, no error
```

> ⚠️ **`ON CONFLICT` needs a `UNIQUE` constraint or primary key on those columns.** Without one you get *there is no unique or exclusion constraint matching the ON CONFLICT specification*.

> **This is the pattern for re-runnable data loads.** Loading the same file twice doesn't create duplicates — the second run just updates the same rows ([[Data modeling]]).

---

## `MERGE` — sync a table from another one

When you have a batch of changes (a staging table from a daily load), `MERGE` applies inserts, updates and deletes in one statement:

```sql
CREATE TABLE product_updates (sku text, name text, price numeric(10,2), discontinued boolean);
INSERT INTO product_updates VALUES
  ('PEN-01',   'Pen',          1.50, false),   -- existing: update the price
  ('KETTLE-01', 'Old kettle', 25.00, true),    -- existing: discontinued, delete it
  ('BOOK-01',  'Notebook',     4.00, false);   -- new: insert it

MERGE INTO products AS p
USING product_updates AS u ON p.sku = u.sku
WHEN MATCHED AND u.discontinued THEN DELETE
WHEN MATCHED THEN UPDATE SET price = u.price
WHEN NOT MATCHED THEN INSERT (sku, name, price) VALUES (u.sku, u.name, u.price)
RETURNING merge_action(), p.sku;                  -- RETURNING on MERGE: PostgreSQL 17+
```

```
 merge_action |    sku
--------------+-----------
 UPDATE       | PEN-01
 DELETE       | KETTLE-01
 INSERT       | BOOK-01
```

| Use | When |
|---|---|
| `INSERT ... ON CONFLICT` | One row or a batch of rows, insert-or-update |
| `MERGE` | Syncing from another table, and you also need deletes or conditions |

> `MERGE` is available from PostgreSQL 15; `RETURNING` and `merge_action()` on it from 17.

---

## `UPDATE ... FROM` and `DELETE ... USING` — change rows based on another table

```sql
UPDATE products AS p
SET stock = p.stock - s.sold
FROM (SELECT product_id, sum(qty) AS sold
      FROM order_items GROUP BY product_id) AS s
WHERE p.id = s.product_id
RETURNING p.sku, p.stock;
```

```sql
DELETE FROM orders AS o
USING customers AS c
WHERE o.customer_id = c.id
  AND c.email = 'grace@example.com'
  AND o.status = 'pending'
RETURNING o.id;
```

> ⚠️ **In `UPDATE ... FROM`, if the other table matches a row more than once, Postgres updates it with one of the matches — which one is undefined.** Group or de-duplicate the other side first, as above.

---

## `DISTINCT ON` — the first row of each group

"Each customer's **most recent** order" is awkward in standard SQL. Postgres has a shortcut:

```sql
SELECT DISTINCT ON (customer_id)
       customer_id, id AS order_id, status, ordered_at
FROM orders
ORDER BY customer_id, ordered_at DESC;           -- DISTINCT ON keeps the FIRST row of each group
```

```
 customer_id | order_id | status  |       ordered_at
-------------+----------+---------+------------------------
           1 |        2 | shipped | 2026-03-05 09:30:00+00
           2 |        3 | paid    | 2026-03-05 14:00:00+00
```

> **Rule: the `ORDER BY` must start with the `DISTINCT ON` columns**, then say which row wins (`ordered_at DESC` = newest). The portable alternative is `row_number()` in a window — see [[SQL fundamentals]].

---

## Aggregates Postgres adds

```sql
SELECT c.name,
       count(*)                                         AS orders,
       count(*) FILTER (WHERE o.status = 'paid')        AS paid_orders,    -- count only some rows
       string_agg(o.status, ', ' ORDER BY o.ordered_at) AS statuses,       -- join text together
       array_agg(o.id ORDER BY o.id)                    AS order_ids,      -- collect into an array
       bool_or(o.status = 'shipped')                    AS anything_shipped
FROM customers c
JOIN orders o ON o.customer_id = c.id
GROUP BY c.name
ORDER BY c.name;
```

```
 name  | orders | paid_orders |   statuses    | order_ids | anything_shipped
-------+--------+-------------+---------------+-----------+------------------
 Ada   |      2 |           1 | paid, shipped | {1,2}     | t
 Linus |      1 |           1 | paid          | {3}       | f
```

> **`FILTER (WHERE ...)` replaces `sum(CASE WHEN ... THEN 1 ELSE 0 END)`.** Same result, much easier to read. It works on any aggregate: `sum(amount) FILTER (WHERE status = 'paid')`.

---

## `generate_series` — make rows out of nothing

Makes a list of numbers or dates. Its main job: **filling gaps**, so days with no orders show as `0` instead of disappearing.

```sql
SELECT d::date                                   AS day,
       count(o.id)                               AS orders
FROM generate_series(date '2026-03-01', date '2026-03-08', interval '1 day') AS d
LEFT JOIN orders o ON o.ordered_at::date = d::date
GROUP BY d
ORDER BY d;
```

```
    day     | orders
------------+--------
 2026-03-01 |      1
 2026-03-02 |      0
 2026-03-03 |      0
 2026-03-04 |      0
 2026-03-05 |      2
 2026-03-06 |      0
 2026-03-07 |      0
 2026-03-08 |      0
```

> **Without the series, days with no orders are simply missing** — and a chart drawn from that data joins the dots straight across the gap, which looks like steady sales. Always build the full date range first, then `LEFT JOIN` the data onto it.

---

## `LATERAL` — "for each row, run this subquery"

A normal subquery in `FROM` can't see the other tables. `LATERAL` can — it runs once per row. Classic use: **top N per group**.

```sql
SELECT c.name, recent.id AS order_id, recent.ordered_at
FROM customers c
CROSS JOIN LATERAL (
    SELECT o.id, o.ordered_at
    FROM orders o
    WHERE o.customer_id = c.id                   -- refers to c from OUTSIDE - that's what LATERAL allows
    ORDER BY o.ordered_at DESC
    LIMIT 2                                      -- each customer's 2 latest orders
) AS recent
ORDER BY c.name, recent.ordered_at DESC;
```

> **`CROSS JOIN LATERAL` drops customers with no orders; `LEFT JOIN LATERAL (...) ON true` keeps them.**

---

## Recursive CTEs — trees and hierarchies

For data that points at itself: categories with sub-categories, managers and reports, folders.

```sql
CREATE TABLE categories (id int PRIMARY KEY, name text, parent_id int REFERENCES categories(id));
INSERT INTO categories VALUES
  (1, 'Home', NULL), (2, 'Kitchen', 1), (3, 'Mugs', 2), (4, 'Office', 1), (5, 'Pens', 4);

WITH RECURSIVE tree AS (
    SELECT id, name, parent_id, name AS path, 0 AS depth
    FROM categories WHERE parent_id IS NULL                  -- 1. start at the top
    UNION ALL
    SELECT c.id, c.name, c.parent_id, t.path || ' > ' || c.name, t.depth + 1
    FROM categories c JOIN tree t ON c.parent_id = t.id      -- 2. then repeatedly add the children
)
SELECT repeat('  ', depth) || name AS category, path FROM tree ORDER BY path;
```

```
  category   |          path
-------------+------------------------
 Home        | Home
   Kitchen   | Home > Kitchen
     Mugs    | Home > Kitchen > Mugs
   Office    | Home > Office
     Pens    | Home > Office > Pens
```

> ⚠️ **A loop in the data (a category that is its own grandparent) makes a recursive query run forever.** Add `WHERE t.depth < 20` to the second part as a safety net.

---

## Text matching

```sql
SELECT name FROM products WHERE name ILIKE '%lamp%';       -- ILIKE = LIKE, ignoring case
SELECT email FROM customers WHERE email ~ '^[a-z]+@';      -- ~ = regular expression match
SELECT email FROM customers WHERE email ~* 'EXAMPLE';      -- ~* = regex, ignoring case
SELECT split_part('ada@example.com', '@', 2) AS domain;    -- 'example.com'
```

| Operator | Means |
|---|---|
| `LIKE` / `ILIKE` | Pattern: `%` = anything, `_` = one character. `ILIKE` ignores case |
| `~` / `~*` | Regular expression, case-sensitive / not |
| `!~` / `!~*` | Does *not* match the regex |

> ⚠️ **`ILIKE '%lamp%'` can't use a normal index** — it reads the whole table. For fast "contains" search, add the `pg_trgm` extension and a trigram index (see [[#Indexes]]).

---

## JSONB — querying JSON

```sql
SELECT sku, attrs->>'colour' AS colour
FROM products
WHERE attrs @> '{"colour": "black"}';            -- @> "contains": the JSON includes this

SELECT sku FROM products WHERE attrs ? 'watts';  -- ?  "has this key"
```

| Operator | Means | Example |
|---|---|---|
| `->` | Get a key, **as JSON** | `attrs -> 'size'` |
| `->>` | Get a key, **as text** | `attrs ->> 'colour'` |
| `#>>` | Get a nested path, as text | `attrs #>> '{dims,width}'` |
| `@>` | Contains | `attrs @> '{"colour":"black"}'` |
| `?` | Has key | `attrs ? 'watts'` |
| `?|` / `?&` | Has any / all of these keys | `attrs ?| array['watts','volts']` |
| `\|\|` | Merge two objects | `attrs \|\| '{"new":1}'` |
| `-` | Remove a key | `attrs - 'watts'` |

**Changing JSON:**

```sql
UPDATE products
SET attrs = jsonb_set(attrs, '{dishwasher}', 'false')     -- set one key
WHERE sku = 'MUG-01';

UPDATE products
SET attrs = attrs || '{"material": "ceramic"}'            -- add or overwrite keys
WHERE sku = 'MUG-01';

SELECT attrs FROM products WHERE sku = 'MUG-01';
```

**Turning a JSON array into rows:**

```sql
SELECT d.sensor
FROM (VALUES ('{"sensors": ["temp", "humidity", "co2"]}'::jsonb)) AS t(info),
     jsonb_array_elements_text(t.info -> 'sensors') AS d(sensor);
```

**Make `@>` and `?` fast on a big table:**

```sql
CREATE INDEX products_attrs_gin ON products USING gin (attrs);
```

---

## Arrays

```sql
SELECT sku, tags FROM products WHERE 'office' = ANY(tags);     -- has this one tag
SELECT sku FROM products WHERE tags @> ARRAY['office'];        -- contains all of these (can use a GIN index)
SELECT sku, unnest(tags) AS tag FROM products;                 -- one row per tag
UPDATE products SET tags = array_append(tags, 'sale') WHERE sku = 'PEN-01';
```

> **`= ANY(tags)` reads well; `tags @> ARRAY[...]` is the one a GIN index speeds up.** Use the second on big tables.

---

## Full-text search

`ILIKE` finds exact letters. **Full-text search** understands words: searching `lamps` finds *lamp*, and it can rank results.

```sql
ALTER TABLE products
    ADD COLUMN search tsvector
    GENERATED ALWAYS AS (to_tsvector('english', name)) STORED;

CREATE INDEX products_search_gin ON products USING gin (search);

SELECT sku, name
FROM products
WHERE search @@ websearch_to_tsquery('english', 'lamps OR notebook');
```

| Piece | What it is |
|---|---|
| `tsvector` | The text broken into word stems: *"desk lamps"* → `'desk' 'lamp'` |
| `tsquery` | The search, also stemmed |
| `@@` | "matches" |
| `websearch_to_tsquery` | Understands Google-style input: `"exact phrase"`, `OR`, `-exclude` — safe to pass user input to |
| `ts_rank(search, query)` | A relevance score to `ORDER BY` |

> ⚠️ **A generated column can only use "immutable" functions** — ones that always give the same output for the same input. `array_to_string` and `concat()` aren't, and fail with *generation expression is not immutable*. Join plain text columns with `||` and `coalesce(col, '')` instead.

> **A generated `tsvector` column plus a GIN index is the whole setup.** For a small or medium app it's good enough to skip a separate search engine entirely.

---

## Indexes

An index is like a book's index — it lets Postgres jump to the rows instead of reading every one. Full explanation of *why* in [[SQL fundamentals]]; here are the kinds Postgres offers:

```sql
CREATE INDEX orders_customer_idx ON orders (customer_id);                  -- B-tree: the default
CREATE INDEX orders_cust_date_idx ON orders (customer_id, ordered_at DESC); -- multi-column: order matters
CREATE INDEX orders_pending_idx ON orders (ordered_at) WHERE status = 'pending';   -- partial
CREATE INDEX customers_email_lower_idx ON customers (lower(email));        -- expression
CREATE INDEX orders_cover_idx ON orders (customer_id) INCLUDE (status);    -- covering
CREATE INDEX orders_date_brin ON orders USING brin (ordered_at);           -- BRIN

CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX products_name_trgm ON products USING gin (name gin_trgm_ops); -- makes ILIKE '%x%' fast
```

| Kind | Use when |
|---|---|
| **B-tree** *(default)* | `=`, `<`, `>`, `BETWEEN`, `ORDER BY`. **This is 90% of indexes** |
| Multi-column `(a, b)` | You filter on `a`, or on `a` **and** `b`. It does **not** help a filter on `b` alone |
| **Partial** `... WHERE status = 'pending'` | You only ever query a small slice — much smaller and faster |
| **Expression** `(lower(email))` | You query with a function: `WHERE lower(email) = ...` |
| `INCLUDE (...)` | Lets a query get its answer from the index alone, without visiting the table |
| **GIN** | `jsonb`, arrays, full-text, trigram `ILIKE` |
| **BRIN** | Huge tables where rows are written in time order (logs, sensor readings) — tiny and cheap |

**On a live production table, always build indexes without locking writes:**

```sql
CREATE INDEX CONCURRENTLY orders_status_idx ON orders (status);
```

> ⚠️ **A plain `CREATE INDEX` blocks every insert and update to that table until it finishes** — on a big table that's minutes of downtime. `CONCURRENTLY` takes longer but lets the app keep writing. (It can't run inside a transaction block.)

> ⚠️ **Every index slows down writes and uses disk.** Add them for queries you actually run slowly, not "just in case". Unused indexes are listed in [[PostgreSQL reference]].

### Is my index being used? — `EXPLAIN`

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 1;
```

| In the output | Means |
|---|---|
| `Seq Scan` | Read the whole table — fine for small tables, slow for big ones |
| `Index Scan` / `Index Only Scan` | Used an index ✅ |
| `Bitmap Heap Scan` | Used an index for many matching rows ✅ |
| `actual time=...` | Real milliseconds (only with `ANALYZE`) |
| `rows=` estimated vs actual | A big mismatch means stale statistics — run `ANALYZE orders;` |

> **On tiny tables Postgres ignores your index on purpose** — reading 4 rows directly is faster than using an index. Test plans on realistic data volumes. Reading a plan in detail: [[PostgreSQL reference]]. In [[pgAdmin 4]], **F7** shows the plan as a diagram.

> ⚠️ **`EXPLAIN ANALYZE` actually runs the query.** On an `UPDATE` or `DELETE` it really changes data — wrap it: `BEGIN; EXPLAIN ANALYZE ...; ROLLBACK;`

---

## Views and materialized views

```sql
CREATE VIEW order_totals AS                        -- a saved query: always up to date, recalculated every time
SELECT o.id, o.customer_id, sum(i.qty * i.unit_price) AS total
FROM orders o JOIN order_items i ON i.order_id = o.id
GROUP BY o.id, o.customer_id;

CREATE MATERIALIZED VIEW daily_sales AS            -- the RESULT is stored: fast to read, goes stale
SELECT ordered_at::date AS day, count(*) AS orders
FROM orders GROUP BY 1;

CREATE UNIQUE INDEX ON daily_sales (day);          -- needed for REFRESH ... CONCURRENTLY
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales;   -- recompute without blocking readers

SELECT * FROM order_totals ORDER BY id;
```

| | View | Materialized view |
|---|---|---|
| Stores data | No — runs the query each time | Yes — a saved copy |
| Always current | ✅ | ❌ until you `REFRESH` |
| Speed | As fast as the query | As fast as reading a table |
| Use for | Hiding complexity, permissions | Slow dashboard queries refreshed on a schedule |

---

## Generated columns

A column **calculated from other columns**, kept in sync automatically:

```sql
ALTER TABLE order_items
    ADD COLUMN line_total numeric(12, 2) GENERATED ALWAYS AS (qty * unit_price) STORED;

SELECT order_id, qty, unit_price, line_total FROM order_items ORDER BY order_id LIMIT 3;
```

> **`STORED`** computes the value when the row is written. PostgreSQL 18 also supports **`VIRTUAL`** (computed when read, uses no disk) — and virtual is the default if you write neither.

---

## Schemas — folders for tables

A **schema** is a namespace inside a database — like a folder. Everything so far went into the default schema, `public`.

```sql
CREATE SCHEMA reporting;

CREATE TABLE reporting.monthly_revenue (month date PRIMARY KEY, revenue numeric(14, 2));

SET search_path TO reporting, public;          -- look in reporting first, then public
SELECT count(*) FROM monthly_revenue;           -- no prefix needed now
RESET search_path;
```

> **Use schemas to separate areas** — `raw`, `staging`, `reporting` for a data pipeline; `app` and `audit` for an application. Permissions can then be granted per schema.

---

## Users, roles and permissions

In Postgres a **role** is either a user (it can log in) or a group (a bundle of permissions). Give each app and each person **only what they need**.

```sql
CREATE ROLE app_readonly NOLOGIN;                                       -- a group
GRANT USAGE ON SCHEMA public TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO app_readonly;   -- future tables too

CREATE ROLE dashboard LOGIN PASSWORD 'change-me-please';               -- a user
GRANT app_readonly TO dashboard;                                        -- joins the group
```

| Command | Does |
|---|---|
| `GRANT SELECT ON table TO role` | Read one table |
| `GRANT SELECT ON ALL TABLES IN SCHEMA s TO role` | Read every **existing** table |
| `ALTER DEFAULT PRIVILEGES ... GRANT ...` | Also cover tables **created later** — easy to forget |
| `GRANT USAGE ON SCHEMA s` | Needed before any table in it can be used |
| `REVOKE ...` | Take it back |

> ⚠️ **Never connect your app as the `postgres` superuser.** Give it its own role that can only touch its own tables. If the app is ever compromised, the damage stops there ([[Security in practice]]).

> ⚠️ **Forgetting `ALTER DEFAULT PRIVILEGES` is the classic mistake:** everything works until someone creates a new table, and then the dashboard gets *permission denied*.

---

## Transactions and locking

```sql
BEGIN;
UPDATE products SET stock = stock - 2 WHERE sku = 'MUG-01';
INSERT INTO orders (customer_id, ordered_at) VALUES (2, now());
COMMIT;                                            -- both happen, or (on error / ROLLBACK) neither does
```

**Locking a row you're about to change:**

```sql
BEGIN;
SELECT stock FROM products WHERE sku = 'MUG-01' FOR UPDATE;   -- others wait until this transaction ends
UPDATE products SET stock = stock - 1 WHERE sku = 'MUG-01';
COMMIT;
```

### A job queue in plain SQL — `SKIP LOCKED`

Several workers take jobs from one table without two of them grabbing the same job:

```sql
CREATE TABLE jobs (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
                   payload jsonb, done boolean NOT NULL DEFAULT false);
INSERT INTO jobs (payload) VALUES ('{"task": "email"}'), ('{"task": "resize"}');

BEGIN;
SELECT id, payload FROM jobs
WHERE NOT done
ORDER BY id
FOR UPDATE SKIP LOCKED                              -- skip jobs another worker already holds
LIMIT 1;
-- ...do the work, then:
UPDATE jobs SET done = true WHERE id = 1;
COMMIT;
```

> **`FOR UPDATE SKIP LOCKED` turns a table into a reliable work queue.** Each worker grabs a different job and nobody waits. For many apps this replaces a separate queue service entirely.

---

## Loading and exporting data — `COPY`

`COPY` is the fast way to move lots of rows in or out — far faster than thousands of `INSERT`s.

From a terminal (the file is on **your** computer):

```bash
psql -h localhost -U postgres -d shop -c "\copy products (sku, name, price) FROM 'products.csv' WITH (FORMAT csv, HEADER true)"
psql -h localhost -U postgres -d shop -c "\copy (SELECT * FROM orders) TO 'orders.csv' WITH (FORMAT csv, HEADER true)"
```

| Command | The file is on… |
|---|---|
| `\copy` (psql) | **Your** computer — what you usually want |
| `COPY` (SQL) | The **database server's** disk — needs special permission, and on Azure/AWS you can't reach that disk |

In [[pgAdmin 4]] the same thing is right-click a table → **Import/Export Data…**

---

## Common mistakes

| Mistake | What happens | Fix |
|---|---|---|
| `ON CONFLICT` without a unique constraint | Error | Add `UNIQUE` on the conflict columns |
| `DISTINCT ON` with the wrong `ORDER BY` | Error, or the wrong row per group | Start `ORDER BY` with the `DISTINCT ON` columns |
| Missing days in a time series | Charts join across gaps | `generate_series` + `LEFT JOIN` |
| `ILIKE '%x%'` on a big table | Slow full scan | `pg_trgm` GIN index |
| Plain `CREATE INDEX` in production | Writes blocked | `CREATE INDEX CONCURRENTLY` |
| `EXPLAIN ANALYZE` on a `DELETE` | Data really deleted | Wrap in `BEGIN; ... ROLLBACK;` |
| Recursive CTE on data with a loop | Runs forever | Add a depth limit |
| App connects as `postgres` | Full access if compromised | A dedicated least-privilege role |
| No `ALTER DEFAULT PRIVILEGES` | *permission denied* on new tables | Set default privileges |
| Thousands of single `INSERT`s | Very slow loads | `\copy`, or batched inserts |

## Related

[[PostgreSQL]] · [[PostgreSQL data types]] · [[PostgreSQL reference]] · [[SQL fundamentals]] · [[pgAdmin 4]] · [[PostgreSQL with SQLAlchemy]] · [[Data modeling]] · [[Security in practice]]
