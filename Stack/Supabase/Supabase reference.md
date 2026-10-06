---
tags: [supabase, postgres, auth, backend, reference]
---

# Supabase reference

Postgres plus auth, storage and realtime. Your notes: [[Supabase]] · Rust access: [[sqlx]] · SQL: [[PostgreSQL reference]] · Everything Postgres: [[PostgreSQL]]

---

## What it actually is

**Supabase is a managed Postgres database with services bolted on.** That's the key insight — underneath, it's just Postgres, so everything in [[PostgreSQL reference]] and [[SQL fundamentals]] applies directly.

| Service | What it is |
|---|---|
| **Database** | Postgres, fully accessible |
| **Auth** | Users, sessions, JWTs, OAuth providers |
| **Storage** | S3-like file storage with access rules |
| **Realtime** | Websocket subscriptions to table changes |
| **Edge Functions** | Deno serverless functions |
| **Auto REST API** | PostgREST — every table gets an endpoint |

> **You can always drop to raw SQL.** Unlike a closed backend-as-a-service, nothing is hidden — connect with `psql` or [[sqlx]] and it's an ordinary Postgres database.

## Setup

```bash
npm install @supabase/supabase-js
npx supabase init
npx supabase start          # local stack in Docker
npx supabase db push        # apply migrations
```

```ts
import { createClient } from "@supabase/supabase-js";

export const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
);
```

## ⚠️ The two keys — get this right

| Key | Safe in the browser? | Respects RLS? |
|---|---|---|
| **anon** | ✅ Yes | ✅ Yes |
| **service_role** | ❌ **Never** | ❌ **Bypasses everything** |

> ⚠️ **The `service_role` key bypasses all security policies.** Putting it in a Client Component, a `NEXT_PUBLIC_*` variable, or a mobile app hands anyone complete read/write access to your entire database. **Server-side only, always.**

## Row Level Security — the thing that makes it safe

**The anon key is public.** Anyone can read it out of your JavaScript. What stops them reading everyone's data is **RLS** — policies enforced by Postgres itself.

```sql
ALTER TABLE readings ENABLE ROW LEVEL SECURITY;

CREATE POLICY "users read own readings"
  ON readings FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "users insert own readings"
  ON readings FOR INSERT
  WITH CHECK (auth.uid() = user_id);
```

| Clause | Applies to |
|---|---|
| `USING` | Which existing rows you can see/modify (SELECT, UPDATE, DELETE) |
| `WITH CHECK` | Which new rows you may write (INSERT, UPDATE) |

> ⚠️ **A table with RLS *disabled* is readable by anyone with the anon key.** Enabling RLS with **no policies** blocks everyone (safer). **Enable RLS on every table**, then add policies.
>
> Supabase warns you about unprotected tables in the dashboard. Do not ignore it.

## Querying

```ts
const { data, error } = await supabase
  .from("readings")
  .select("id, temp_c, device:devices(name)")     // joins via foreign keys
  .eq("device_id", id)
  .gte("temp_c", 40)
  .order("created_at", { ascending: false })
  .limit(20);

if (error) throw error;
```

| Filter | SQL |
|---|---|
| `.eq(c, v)` / `.neq` | `=` / `<>` |
| `.gt` `.gte` `.lt` `.lte` | comparisons |
| `.like` / `.ilike` | `LIKE` / case-insensitive |
| `.in(c, [...])` | `IN` |
| `.is(c, null)` | `IS NULL` |
| `.contains` | array/jsonb containment |
| `.or("a.eq.1,b.eq.2")` | `OR` |
| `.range(0, 9)` | pagination |
| `.single()` | one row or error |
| `.maybeSingle()` | one row or `null` |

> ⚠️ **Always check `error`.** The client does **not** throw on failure — it returns `{ data: null, error }`. Ignoring it gives you silent `null` data and a confusing bug.

## Writing

```ts
await supabase.from("readings").insert({ device_id: "1", temp_c: 42.1 });
await supabase.from("readings").insert([{...}, {...}]);           // bulk

await supabase.from("readings").update({ temp_c: 43 }).eq("id", 1);
await supabase.from("readings").upsert({ id: 1, temp_c: 43 });     // insert or update
await supabase.from("readings").delete().eq("id", 1);

const { data } = await supabase.from("readings").insert({...}).select();   // return the row
```

> ⚠️ **An `update` or `delete` without a filter affects every row.** There's no confirmation prompt. Always chain `.eq()` — and RLS is your backstop if you forget.

## Auth

```ts
await supabase.auth.signUp({ email, password });
await supabase.auth.signInWithPassword({ email, password });
await supabase.auth.signInWithOAuth({ provider: "github" });
await supabase.auth.signOut();

const { data: { user } } = await supabase.auth.getUser();
const { data: { session } } = await supabase.auth.getSession();

supabase.auth.onAuthStateChange((event, session) => { /* ... */ });
```

> **Use `getUser()` on the server, not `getSession()`.** `getUser()` verifies the JWT with the auth server; `getSession()` just reads local storage, which a client can forge.

Auth users live in `auth.users`. Reference them from your own tables:

```sql
CREATE TABLE profiles (
  id uuid PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  username text UNIQUE,
  created_at timestamptz DEFAULT now()
);
```

## Storage

```ts
await supabase.storage.from("avatars").upload(`${userId}/pic.png`, file);
const { data } = supabase.storage.from("avatars").getPublicUrl(path);
const { data } = await supabase.storage.from("private").createSignedUrl(path, 3600);
await supabase.storage.from("avatars").remove([path]);
```

> **Storage buckets have their own RLS policies.** A public bucket is readable by anyone with the URL — fine for avatars, wrong for documents.

## Realtime

```ts
const channel = supabase
  .channel("readings")
  .on("postgres_changes",
      { event: "INSERT", schema: "public", table: "readings" },
      (payload) => console.log(payload.new))
  .subscribe();

await supabase.removeChannel(channel);      // always clean up
```

> **Enable realtime per table** in the dashboard, and remember **RLS applies** — subscribers only receive rows they're allowed to see.

## Database functions (RPC)

```sql
CREATE FUNCTION daily_totals(start_date date)
RETURNS TABLE(day date, total numeric)
LANGUAGE sql SECURITY INVOKER AS $$
  SELECT date_trunc('day', created_at)::date, SUM(revenue)
  FROM sales WHERE created_at >= start_date GROUP BY 1;
$$;
```

```ts
const { data } = await supabase.rpc("daily_totals", { start_date: "2026-01-01" });
```

> **`SECURITY INVOKER` (the default) runs as the caller and respects RLS.** `SECURITY DEFINER` runs as the function owner and **bypasses RLS** — only use it deliberately, and set `search_path` when you do.

## Migrations

```bash
npx supabase migration new create_readings
# edit supabase/migrations/<ts>_create_readings.sql
npx supabase db push
npx supabase db reset            # local: rebuild from migrations
npx supabase gen types typescript --linked > types/supabase.ts
```

```ts
import type { Database } from "@/types/supabase";
const supabase = createClient<Database>(url, key);      // fully typed queries
```

> **Generate types after every schema change.** That's what turns Supabase queries into typed TypeScript ([[TypeScript]]).

## Accessing it from Rust

Your backend is [[Actix Web]], so connect with [[sqlx]] directly — no Supabase client needed:

```
postgres://postgres:[PASSWORD]@db.[PROJECT].supabase.co:5432/postgres
```

> **A server-side connection bypasses RLS** (it's a direct Postgres connection as a privileged user). Your Actix handlers become responsible for authorisation.

## Common mistakes

| Mistake | Consequence |
|---|---|
| **`service_role` key in the browser** | **Total database compromise** |
| RLS not enabled | Anyone with the anon key reads everything |
| Not checking `error` | Silent `null` data |
| `update`/`delete` with no filter | Every row affected |
| `getSession()` for server auth | Forgeable |
| Forgetting to regenerate types | Type errors that don't match reality |
| Not unsubscribing from channels | Memory leaks |
| `SECURITY DEFINER` by habit | Silently bypasses RLS |

## Related

[[Supabase]] · [[sqlx]] · [[PostgreSQL reference]] · [[SQL fundamentals]] · [[Actix Web]] · [[NextJs TypeScript]] · [[26 — SECURITY]] · [[Primary Web App Stack]]
