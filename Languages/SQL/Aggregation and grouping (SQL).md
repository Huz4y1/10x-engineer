Aggregates squash many rows into one number.

|Function|Gives you|
|---|---|
|COUNT(*)|how many rows|
|SUM(col)|total|
|AVG(col)|mean|
|MIN(col) / MAX(col)|smallest / largest|

Over the whole table

```sql
SELECT COUNT(*) FROM orders;

SELECT
    SUM(amount)  AS total,
    AVG(amount)  AS average,
    MIN(amount)  AS cheapest,
    MAX(amount)  AS priciest
FROM orders;

-- COUNT(*) counts rows, COUNT(user_id) skips NULLs, they are not the same
SELECT COUNT(*), COUNT(user_id) FROM orders;
```

GROUP BY, one row per group instead

```sql
SELECT
    user_id,
    COUNT(*)    AS order_count,
    SUM(amount) AS total_spent
FROM orders
GROUP BY user_id;
```

|user_id|order_count|total_spent|
|---|---|---|
|1|2|69.98|
|2|1|180.00|

With a join so you get names, not ids

```sql
SELECT
    u.username,
    COUNT(o.id)              AS order_count,
    COALESCE(SUM(o.amount), 0) AS total_spent   -- SUM of nothing is NULL, so default it
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.username     -- every non-aggregated column has to be in the GROUP BY
ORDER BY total_spent DESC;
```

|username|order_count|total_spent|
|---|---|---|
|alex|1|180.00|
|sam|2|69.98|
|mia|0|0|
|jo|0|0|

HAVING vs WHERE

```sql
SELECT user_id, SUM(amount) AS total_spent
FROM orders
WHERE amount > 10          -- filters individual rows BEFORE grouping
GROUP BY user_id
HAVING SUM(amount) > 60;   -- filters the groups AFTER grouping
```

The rule is WHERE can't see aggregates, because at that point the groups don't exist yet

```sql
-- this errors
SELECT user_id FROM orders
GROUP BY user_id
WHERE SUM(amount) > 60;

-- this works
SELECT user_id FROM orders
GROUP BY user_id
HAVING SUM(amount) > 60;
```

---

**Going deeper:** [[SQL fundamentals]] - `GROUP BY` vs `HAVING`, and window functions that aggregate without collapsing rows.
