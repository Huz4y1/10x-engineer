Everything

```sql
SELECT * FROM users;
```

|id|email|username|is_active|
|---|---|---|---|
|1|sam@mail.com|sam|true|
|2|alex@mail.com|alex|true|
|3|mia@mail.com|mia|false|
|4|jo@mail.com|jo|true|

Just the columns you need

```sql
SELECT username, email FROM users;

-- * is fine when poking around, but name the columns in real code
-- otherwise adding a column later silently changes what your query returns
```

Aliases with AS

```sql
SELECT
    username AS name,
    created_at AS joined
FROM users;
```

|name|joined|
|---|---|
|sam|2026-07-01|
|alex|2026-07-03|

Aliasing the table too, saves typing once joins show up

```sql
SELECT u.username, o.product
FROM users AS u
JOIN orders AS o ON o.user_id = u.id;
```

Calculated columns

```sql
SELECT
    product,
    amount,
    amount * 0.2 AS vat        -- expressions need an alias or the column is called "?column?"
FROM orders;
```

DISTINCT, unique values only

```sql
SELECT DISTINCT product FROM orders;

-- DISTINCT on more than one column means unique combinations
SELECT DISTINCT user_id, product FROM orders;
```

LIMIT, don't pull 50k rows to look at 5

```sql
SELECT * FROM orders
ORDER BY created_at DESC
LIMIT 5;
```

---

**Going deeper:** [[SQL fundamentals]] - the order SQL actually evaluates clauses in, and why aliases fail in `WHERE`.
