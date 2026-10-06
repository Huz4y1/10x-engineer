The two tables the rest of these notes use.

Users

```sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       TEXT NOT NULL,
    username    TEXT NOT NULL,
    is_active   BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);
```

Orders

```sql
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT REFERENCES users(id),  -- links a row back to a user
    product     TEXT NOT NULL,
    amount      NUMERIC(10, 2) NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);
```

Only create it if it isn't already there

```sql
CREATE TABLE IF NOT EXISTS users (
    id BIGSERIAL PRIMARY KEY
);
```

Changing a table after the fact

```sql
-- add a column (existing rows get NULL, or the default if you give one)
ALTER TABLE users ADD COLUMN country TEXT;

ALTER TABLE users ADD COLUMN login_count INTEGER DEFAULT 0;

-- rename it
ALTER TABLE users RENAME COLUMN country TO country_code;

-- change its type
ALTER TABLE users ALTER COLUMN login_count TYPE BIGINT;

-- drop it, the data in it is gone
ALTER TABLE users DROP COLUMN country_code;
```

Dropping

```sql
DROP TABLE orders;

-- doesn't error if the table isn't there
DROP TABLE IF EXISTS orders;

-- CASCADE also drops anything depending on it (like foreign keys pointing at it)
DROP TABLE users CASCADE;
```
