Databases can store JSON in a column and query inside it. Useful for genuinely variable data, and a trap if you use it for things that should be real columns.

Examples here are Postgres, which has the best support. SQLite and MySQL notes at the end.

json vs jsonb

Postgres has two types and you almost always want the second.

```
  json      stores the exact text you gave it
            keeps whitespace, key order and duplicate keys
            reparsed on every query - slow
            no indexes

  jsonb     parsed into a binary tree on write
            loses whitespace and key order, dedupes keys
            fast to query, supports GIN indexes
            slightly slower to insert
```

Use `jsonb`. The only reason for `json` is if you must reproduce the original document byte for byte.

```sql
CREATE TABLE events (
    id          BIGSERIAL PRIMARY KEY,
    kind        TEXT NOT NULL,
    payload     JSONB NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT now()
);

INSERT INTO events (kind, payload) VALUES
  ('signup', '{"user_id": 91, "plan": "pro", "meta": {"ref": "twitter"}}'),
  ('order',  '{"user_id": 91, "total": 49.99, "items": ["a","b"]}');
```

The operators

This is the part to memorise. The distinction between `->` and `->>` is where everyone slips.

```
  ->     get a field, RESULT IS JSON
  ->>    get a field, RESULT IS TEXT
  #>     get a nested path, as JSON
  #>>    get a nested path, as TEXT
```

```sql
SELECT
  payload -> 'user_id'                AS as_json,   -- 91   (jsonb)
  payload ->> 'user_id'               AS as_text,   -- "91" (text)
  payload -> 'meta' ->> 'ref'         AS ref,       -- twitter
  payload #>> '{meta,ref}'            AS ref_path,  -- same thing
  payload -> 'items' ->> 0            AS first_item -- a
FROM events;
```

Rule of thumb: use `->` while you are still digging, `->>` on the final step where you want a usable value.

Comparing needs a cast

`->>` gives you text, so numeric comparison needs a cast or it compares as strings.

```sql
-- WRONG, string comparison: '9' > '10' is true
SELECT * FROM events WHERE payload ->> 'total' > '10';

-- right
SELECT * FROM events WHERE (payload ->> 'total')::numeric > 10;

-- also right, comparing as jsonb
SELECT * FROM events WHERE payload -> 'total' > '10'::jsonb;
```

Containment, the fast one

`@>` asks "does the left contain the right". This is the operator a GIN index can actually use.

```sql
-- events where plan is pro
SELECT * FROM events WHERE payload @> '{"plan":"pro"}';

-- nested containment works too
SELECT * FROM events WHERE payload @> '{"meta":{"ref":"twitter"}}';

-- arrays: does items contain "a"
SELECT * FROM events WHERE payload -> 'items' @> '"a"';
```

Existence operators:

```sql
payload ? 'plan'                  -- has the key "plan"
payload ?| array['plan','tier']   -- has ANY of these keys
payload ?& array['plan','tier']   -- has ALL of these keys
```

Indexing

Without an index, every query reads every row and parses the JSON. With 10 rows fine, with 10 million not fine.

```sql
-- general purpose, supports @> ? ?| ?&
CREATE INDEX idx_events_payload ON events USING GIN (payload);

-- smaller and faster if you only ever use @>
CREATE INDEX idx_events_payload ON events USING GIN (payload jsonb_path_ops);

-- if you always filter one specific field, index just that expression
CREATE INDEX idx_events_user ON events (((payload ->> 'user_id')::bigint));
```

That last form is usually the winner for a hot field. See [[Indexes (SQL)]].

Note GIN indexes do **not** help `->>` comparisons — only the containment and existence operators. If your query uses `->>`, you need the expression index.

Expanding arrays into rows

This is how you get from JSON back into relational shape.

```sql
/*
1. jsonb_array_elements turns one row with N array items into N rows
2. it goes in the FROM clause as a lateral join
3. _text variant gives text instead of jsonb
*/

SELECT e.id, item
FROM events e,
     jsonb_array_elements_text(e.payload -> 'items') AS item
WHERE e.kind = 'order';

--  id | item
--  ---+-----
--   2 | a
--   2 | b
```

Once expanded you can group and aggregate normally — see [[Aggregation and grouping (SQL)]].

```sql
-- most common items across all orders
SELECT item, count(*)
FROM events e,
     jsonb_array_elements_text(e.payload -> 'items') AS item
GROUP BY item
ORDER BY count(*) DESC;
```

Expanding objects

```sql
-- one row per key/value pair
SELECT key, value
FROM events, jsonb_each_text(payload)
WHERE id = 1;

-- just the keys
SELECT jsonb_object_keys(payload) FROM events WHERE id = 1;
```

`jsonb_each_text` is the SQL equivalent of the `to_entries` trick in [[Investigating an API]] — good for finding out what is actually in a column.

Building JSON out

The other direction: turn query results into JSON, which is handy for APIs.

```sql
-- one object per row
SELECT jsonb_build_object(
  'id',   id,
  'kind', kind,
  'user', payload -> 'user_id'
) FROM events;

-- the whole row, automatically
SELECT to_jsonb(e) FROM events e;

-- aggregate many rows into one array
SELECT jsonb_agg(to_jsonb(e)) FROM events e;

-- nested: users with their events embedded
SELECT jsonb_build_object(
  'user_id', u.id,
  'events',  (SELECT jsonb_agg(to_jsonb(e))
              FROM events e
              WHERE (e.payload ->> 'user_id')::bigint = u.id)
) FROM users u;
```

That last pattern replaces a lot of application-side joining. One query, JSON out, straight to the client.

Modifying JSON

```sql
-- set a field (creates it if missing)
UPDATE events SET payload = jsonb_set(payload, '{plan}', '"enterprise"');

-- nested path
UPDATE events SET payload = jsonb_set(payload, '{meta,ref}', '"google"');

-- merge two objects, right side wins
UPDATE events SET payload = payload || '{"verified": true}';

-- delete a key
UPDATE events SET payload = payload - 'plan';

-- delete a nested path
UPDATE events SET payload = payload #- '{meta,ref}';
```

`||` for merging is the quickest way to add a field to every row.

JSON path queries

Postgres 12+ has a proper query language for deeper work.

```sql
-- does any item match
SELECT * FROM events
WHERE payload @@ '$.total > 20';

-- extract with a filter
SELECT jsonb_path_query(payload, '$.items[*] ? (@ != "a")')
FROM events WHERE kind = 'order';
```

Closer to [[jq]] in feel. Worth knowing but the operators above cover most real work.

When NOT to use a JSON column

```
  USE JSONB FOR                      USE REAL COLUMNS FOR
  ─────────────                      ────────────────────
  genuinely variable payloads        anything you filter on often
  third-party API responses          anything you join on
  event/audit logs                   anything with a foreign key
  user-defined custom fields         anything you need constrained
  settings blobs                     anything you aggregate constantly
```

The failure mode is putting `email` in a JSON column, then needing a unique constraint on it, then discovering you cannot have one without an expression index and a lot of regret. If you know the field exists on every row, make it a column. See [[Constraints and keys (SQL)]].

A good middle path is both: real columns for the fields you query, plus a `payload jsonb` for everything else the API sent.

SQLite

```sql
-- SQLite 3.38+ has -> and ->> too
SELECT payload ->> 'user_id' FROM events;

-- older syntax, still works
SELECT json_extract(payload, '$.user_id') FROM events;

-- expanding arrays
SELECT value FROM events, json_each(events.payload, '$.items');

-- 3.45+ has jsonb, a faster binary format
```

Note SQLite's `->>` returns a proper SQL value (integer stays integer), unlike Postgres where it is always text.

MySQL

```sql
SELECT payload -> '$.user_id' FROM events;      -- JSON result
SELECT payload ->> '$.user_id' FROM events;     -- unquoted text
SELECT JSON_EXTRACT(payload, '$.user_id') FROM events;

-- MySQL can't index JSON directly, use a generated column
ALTER TABLE events
  ADD COLUMN user_id BIGINT AS (payload ->> '$.user_id') STORED,
  ADD INDEX (user_id);
```

MySQL always uses `$.` path syntax, and needs the generated-column trick for indexing.
