Single row

```sql
INSERT INTO users (email, username)
VALUES ('sam@mail.com', 'sam');

-- id and created_at are filled in for you by SERIAL and DEFAULT NOW()
```

Multiple rows in one statement

```sql
INSERT INTO users (email, username)
VALUES
    ('alex@mail.com', 'alex'),
    ('mia@mail.com',  'mia'),
    ('jo@mail.com',   'jo');

-- one round trip instead of three, much faster than looping inserts in app code
```

Getting the new row back

```sql
INSERT INTO users (email, username)
VALUES ('kai@mail.com', 'kai')
RETURNING id, created_at;
```

|id|created_at|
|---|---|
|5|2026-07-21 10:14:02+00|

RETURNING is what you want with sqlx, since you usually need the generated id straight away

```sql
INSERT INTO orders (user_id, product, amount)
VALUES ($1, $2, $3)
RETURNING *;
```

Upsert, ignore the duplicate

```sql
INSERT INTO users (email, username)
VALUES ('sam@mail.com', 'sam')
ON CONFLICT (email) DO NOTHING;

-- without this it would error, since email is UNIQUE
```

Upsert, update the existing row instead

```sql
INSERT INTO users (email, username)
VALUES ('sam@mail.com', 'sam_new')
ON CONFLICT (email)
DO UPDATE SET
    username = EXCLUDED.username;   -- EXCLUDED is the row you tried to insert
```
