A subquery is a SELECT inside another query. A CTE is the same thing pulled out and given a name.

Subquery in WHERE

```sql
-- users who have ordered something
SELECT username
FROM users
WHERE id IN (SELECT user_id FROM orders);

-- orders above the average order value
SELECT product, amount
FROM orders
WHERE amount > (SELECT AVG(amount) FROM orders);
```

EXISTS, usually faster than IN on big tables since it stops at the first match

```sql
SELECT username
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

SELECT username
FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

Subquery in SELECT, one value per row

```sql
SELECT
    username,
    (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS order_count
FROM users u;
```

|username|order_count|
|---|---|
|sam|2|
|alex|1|
|mia|0|

CTE with WITH

```sql
WITH user_totals AS (
    SELECT user_id, SUM(amount) AS total_spent
    FROM orders
    GROUP BY user_id
)
SELECT u.username, t.total_spent
FROM users u
JOIN user_totals t ON t.user_id = u.id
WHERE t.total_spent > 60;
```

Several CTEs, each can use the ones before it

```sql
WITH user_totals AS (
    SELECT user_id, SUM(amount) AS total_spent
    FROM orders
    GROUP BY user_id
),
big_spenders AS (
    SELECT user_id FROM user_totals WHERE total_spent > 100
)
SELECT u.username
FROM users u
JOIN big_spenders b ON b.user_id = u.id;
```

The same thing as a nested subquery, which is why CTEs win

```sql
SELECT u.username
FROM users u
JOIN (
    SELECT user_id
    FROM (
        SELECT user_id, SUM(amount) AS total_spent
        FROM orders
        GROUP BY user_id
    ) t
    WHERE t.total_spent > 100
) b ON b.user_id = u.id;

-- same result, but you read it inside out and every step is unnamed
-- CTEs read top to bottom like steps, and you can SELECT * from one to debug it
```

---

**Going deeper:** [[SQL fundamentals]] - CTEs as named steps, recursive CTEs, and the debugging trick.
