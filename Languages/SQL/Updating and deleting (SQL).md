UPDATE with a WHERE

```sql
UPDATE users
SET username = 'sam_smith'
WHERE id = 1;

-- several columns at once
UPDATE users
SET username = 'sam_smith',
    is_active = FALSE
WHERE id = 1;
```

Updating based on the current value

```sql
UPDATE users
SET login_count = login_count + 1
WHERE id = 1;

UPDATE orders
SET amount = amount * 1.2
WHERE product = 'Monitor';
```

Forgetting the WHERE

```sql
-- this sets EVERY user's username to sam, all of them, no undo
UPDATE users SET username = 'sam';
```

So check first with a SELECT using the same WHERE, then swap it out

```sql
SELECT * FROM users WHERE id = 1;   -- is this the row I mean?

UPDATE users SET is_active = FALSE WHERE id = 1;
```

RETURNING works here too, handy for confirming what changed

```sql
UPDATE orders
SET amount = 55.00
WHERE id = 1
RETURNING id, product, amount;
```

DELETE

```sql
DELETE FROM orders WHERE id = 4;

DELETE FROM orders WHERE user_id = 3;

-- same trap, no WHERE means the whole table
DELETE FROM orders;
```

TRUNCATE, empties the table fast

```sql
TRUNCATE orders;

-- also resets the SERIAL counter back to 1
TRUNCATE orders RESTART IDENTITY;

-- needed if another table has a foreign key pointing at this one
TRUNCATE users, orders RESTART IDENTITY CASCADE;

-- TRUNCATE can't take a WHERE and doesn't go row by row, so it's much quicker
-- than DELETE on a big table, but it's all or nothing
```

Soft delete, often better than actually removing the row

```sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;

UPDATE users SET deleted_at = NOW() WHERE id = 3;

SELECT * FROM users WHERE deleted_at IS NULL;
```
