A transaction groups statements so they either all happen or none of them do.

The shape of it

```sql
BEGIN;

INSERT INTO users (email, username) VALUES ('kai@mail.com', 'kai');
INSERT INTO orders (user_id, product, amount) VALUES (5, 'Keyboard', 49.99);

COMMIT;   -- everything above becomes permanent at this point
```

ROLLBACK throws the whole lot away

```sql
BEGIN;

DELETE FROM orders WHERE user_id = 1;

-- realise that was wrong
ROLLBACK;   -- the orders are still there, nothing was written
```

A transfer, where both statements have to succeed together

```sql
BEGIN;

-- take money off sam
UPDATE users
SET balance = balance - 100
WHERE id = 1;

-- give it to alex
UPDATE users
SET balance = balance + 100
WHERE id = 2;

COMMIT;

-- if the database crashed between the two updates without a transaction,
-- the 100 would just vanish. inside BEGIN/COMMIT neither update is visible
-- until both have run
```

Guarding it with a CHECK so the transaction fails instead of going negative

```sql
ALTER TABLE users ADD CONSTRAINT balance_positive CHECK (balance >= 0);

BEGIN;
UPDATE users SET balance = balance - 100 WHERE id = 1;  -- errors if sam only has 40
UPDATE users SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- once a statement errors, Postgres puts the transaction in an aborted state
-- and every following statement fails until you ROLLBACK
```

SAVEPOINT, rolling back only part of it

```sql
BEGIN;

INSERT INTO users (email, username) VALUES ('kai@mail.com', 'kai');

SAVEPOINT after_user;

INSERT INTO orders (user_id, product, amount) VALUES (5, 'Cable', -5.00);

ROLLBACK TO after_user;   -- the bad order is undone, the user insert survives

COMMIT;
```

---

**Going deeper:** [[PostgreSQL reference]] - ACID, isolation levels and MVCC.
