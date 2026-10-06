Joins pull columns from two tables into one result by matching a key. Here users has 4 rows, orders has 4 rows, 3 of which belong to a real user and 1 has a NULL user_id.

users

|id|username|
|---|---|
|1|sam|
|2|alex|
|3|mia|
|4|jo|

orders

|id|user_id|product|amount|
|---|---|---|---|
|1|1|Keyboard|49.99|
|2|1|Mouse|19.99|
|3|2|Monitor|180.00|
|4|NULL|Cable|5.00|

INNER JOIN, only rows that match on both sides

```sql
SELECT u.username, o.product, o.amount
FROM users u
INNER JOIN orders o ON o.user_id = u.id;
-- JOIN on its own means INNER JOIN
```

|username|product|amount|
|---|---|---|
|sam|Keyboard|49.99|
|sam|Mouse|19.99|
|alex|Monitor|180.00|

LEFT JOIN, every user, orders where they exist

```sql
SELECT u.username, o.product
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;
```

|username|product|
|---|---|
|sam|Keyboard|
|sam|Mouse|
|alex|Monitor|
|mia|NULL|
|jo|NULL|

Which is how you find users with no orders

```sql
SELECT u.username
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;   -- no matching order, so the right side came back NULL
```

RIGHT JOIN, every order, users where they exist

```sql
SELECT u.username, o.product
FROM users u
RIGHT JOIN orders o ON o.user_id = u.id;

-- same as flipping the tables and using LEFT JOIN, which is why you rarely see RIGHT JOIN
```

FULL OUTER JOIN, everything from both sides

```sql
SELECT u.username, o.product
FROM users u
FULL OUTER JOIN orders o ON o.user_id = u.id;
```

How the row counts differ

|Join|Rows|What you get|
|---|---|---|
|INNER|3|only matched pairs|
|LEFT|5|matched pairs + mia and jo with NULLs|
|RIGHT|4|matched pairs + the orphan Cable order|
|FULL OUTER|6|matched pairs + both sets of leftovers|

In practice it's INNER JOIN and LEFT JOIN nearly every time. INNER when you only care about users who actually ordered something, LEFT when you're building a list of all users and want their orders attached if they have any. RIGHT is just a LEFT written backwards, and FULL OUTER mostly turns up when reconciling two datasets that should match but don't.

Joining three tables is the same pattern chained

```sql
SELECT u.username, o.product, oi.quantity
FROM users u
JOIN orders o      ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id
WHERE u.is_active;
```

---

**Going deeper:** [[SQL fundamentals]] - join types, the row-count trap, and why totals silently double.
