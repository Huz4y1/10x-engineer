Constraints are rules the database enforces for you, so bad data can't get in even if the app code has a bug.

|Constraint|What it stops|
|---|---|
|PRIMARY KEY|Duplicate or missing ids|
|FOREIGN KEY|Rows pointing at a user that doesn't exist|
|NOT NULL|Empty values|
|UNIQUE|Two rows with the same email|
|DEFAULT|Having to pass a value every insert|
|CHECK|Values that make no sense, like a negative price|

All of them on the users table

```sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,             -- unique + not null, automatically
    email       TEXT NOT NULL UNIQUE,              -- no two users share an email
    username    TEXT NOT NULL,
    age         INTEGER CHECK (age >= 13),         -- reject anyone under 13
    is_active   BOOLEAN NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Foreign key linking orders to users

```sql
CREATE TABLE orders (
    id       BIGSERIAL PRIMARY KEY,
    user_id  BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    product  TEXT NOT NULL,
    amount   NUMERIC(10, 2) NOT NULL CHECK (amount > 0)
);

-- ON DELETE CASCADE  -> deleting a user deletes their orders too
-- ON DELETE SET NULL -> the order stays, user_id becomes NULL
-- default (RESTRICT)  -> Postgres refuses to delete a user who has orders
```

What it looks like when you break one

```sql
INSERT INTO orders (user_id, product, amount)
VALUES (999, 'Keyboard', 49.99);
-- ERROR: insert or update on table "orders" violates foreign key constraint
-- there is no user with id 999
```

Adding constraints later

```sql
ALTER TABLE users ADD CONSTRAINT users_email_unique UNIQUE (email);

ALTER TABLE orders ADD CONSTRAINT amount_positive CHECK (amount > 0);

ALTER TABLE orders DROP CONSTRAINT amount_positive;
```

A composite primary key, when the pair is what's unique

```sql
CREATE TABLE order_items (
    order_id   BIGINT REFERENCES orders(id),
    product_id BIGINT,
    quantity   INTEGER NOT NULL DEFAULT 1,
    PRIMARY KEY (order_id, product_id)   -- same product can't be added twice to one order
);
```

---

**Going deeper:** [[Data modeling]] - natural vs surrogate keys, and using `UNIQUE` to make loads idempotent.
