WHERE

```sql
SELECT * FROM users
WHERE is_active = TRUE;

SELECT * FROM orders
WHERE amount > 50;

-- one = for comparison, not two
SELECT * FROM users WHERE username = 'sam';
```

AND, OR, NOT

```sql
SELECT * FROM orders
WHERE amount > 50 AND product = 'Keyboard';

SELECT * FROM users
WHERE username = 'sam' OR username = 'mia';

SELECT * FROM users
WHERE NOT is_active;

-- brackets matter, AND binds tighter than OR
SELECT * FROM orders
WHERE (product = 'Mouse' OR product = 'Keyboard') AND amount > 20;
```

IN, tidier than a pile of ORs

```sql
SELECT * FROM users
WHERE username IN ('sam', 'mia', 'jo');

SELECT * FROM orders
WHERE user_id NOT IN (1, 2);
```

BETWEEN, inclusive on both ends

```sql
SELECT * FROM orders
WHERE amount BETWEEN 20 AND 100;

SELECT * FROM orders
WHERE created_at BETWEEN '2026-07-01' AND '2026-07-31';
```

LIKE, pattern matching

```sql
SELECT * FROM users WHERE email LIKE '%@mail.com';  -- % is any number of characters
SELECT * FROM users WHERE username LIKE 's%';       -- starts with s
SELECT * FROM users WHERE username LIKE '_am';      -- _ is exactly one character

-- ILIKE is the case insensitive version, Postgres only
SELECT * FROM users WHERE username ILIKE 'SAM';
```

NULL, which is not a value so = won't work on it

```sql
SELECT * FROM orders WHERE user_id IS NULL;

SELECT * FROM orders WHERE user_id IS NOT NULL;

-- this returns nothing, ever
SELECT * FROM orders WHERE user_id = NULL;
```

ORDER BY

```sql
SELECT * FROM orders ORDER BY amount DESC;

SELECT * FROM users ORDER BY username ASC;   -- ASC is the default

-- sort by one column, break ties with another
SELECT * FROM orders ORDER BY user_id ASC, amount DESC;

-- NULLs sort last by default on DESC, first on ASC, override it if it matters
SELECT * FROM orders ORDER BY amount DESC NULLS LAST;
```

LIMIT and OFFSET for pagination

```sql
-- page 1
SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 0;

-- page 2
SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 10;

-- always ORDER BY when paginating, without it the order isn't guaranteed
-- and you can get the same row on two different pages
```
