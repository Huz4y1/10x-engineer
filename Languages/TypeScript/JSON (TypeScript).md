The runtime behaviour is identical to [[JSON (JavaScript)]] — same `JSON.parse`, same array methods. What TypeScript adds is the problem of *types you cannot trust*.

The core issue

```ts
const data = JSON.parse(text);        // type is `any`
data.naem.toUpperCase();              // compiles fine, crashes at runtime
```

`JSON.parse` returns `any`, which switches off type checking entirely. Same with `res.json()`. Every typo and every wrong assumption about the API sails straight through the compiler.

There are three honest answers to this, in increasing order of safety.

Option 1, assert the type

```ts
interface User {
    id: number;
    name: string;
    email: string | null;      // null in some responses, so allow it
    tags: string[];
}

const user = JSON.parse(text) as User;
```

Fast, zero dependencies, and a **lie**. You have told the compiler what you hope is true. If the API changes, nothing warns you — you just get `undefined` somewhere far away.

Fine for a script. Not fine for anything that runs unattended.

Option 2, unknown plus a type guard

```ts
/*
1. `unknown` forces you to check before using
2. the `x is User` return type tells the compiler the check succeeded
3. no dependencies, and it's honest
*/

function isUser(x: unknown): x is User {
    if (typeof x !== "object" || x === null) return false;
    const u = x as Record<string, unknown>;

    return typeof u.id === "number"
        && typeof u.name === "string"
        && (typeof u.email === "string" || u.email === null)
        && Array.isArray(u.tags)
        && u.tags.every(t => typeof t === "string");
}

const parsed: unknown = JSON.parse(text);
if (!isUser(parsed)) throw new Error("unexpected shape");
// from here, parsed is User
```

Correct, but tedious, and the guard drifts out of sync with the interface as soon as you edit one and not the other.

Option 3, a schema library

This is what to reach for in a real project. Zod defines the shape once and gives you both the runtime check and the static type.

```ts
/*
1. the schema is the single source of truth
2. z.infer derives the TypeScript type from it, so they can't drift
3. parse throws on bad data, safeParse returns a result object
*/

import { z } from "zod";

const User = z.object({
    id: z.number(),
    name: z.string(),
    email: z.string().email().nullable(),
    tags: z.array(z.string()),
    created_at: z.coerce.date(),        // ISO string -> Date, automatically
});

type User = z.infer<typeof User>;       // the type, for free

const user = User.parse(JSON.parse(text));       // throws on mismatch
```

Non-throwing version, which is usually what you want at an API boundary:

```ts
const result = User.safeParse(JSON.parse(text));

if (!result.success) {
    console.error(result.error.issues);   // exactly which field was wrong
    return null;
}

const user = result.data;               // fully typed
```

The error tells you the path and the expected type, so a broken API gives you a clear message instead of a mystery `undefined` three files away.

A typed fetch wrapper

Worth writing once per project.

```ts
/*
1. takes a schema, returns data already validated against it
2. one place that handles HTTP errors and shape errors
3. every caller gets a real type with no casting
*/

import { z } from "zod";

async function fetchJson<T>(url: string, schema: z.ZodType<T>): Promise<T> {
    const res = await fetch(url);

    if (!res.ok) {
        throw new Error(`HTTP ${res.status} for ${url}`);
    }

    const raw = await res.json();
    const parsed = schema.safeParse(raw);

    if (!parsed.success) {
        throw new Error(`Bad shape from ${url}: ${parsed.error.message}`);
    }

    return parsed.data;
}

// usage
const users = await fetchJson("/api/users", z.array(User));
//    ^ User[], guaranteed to actually be that
```

Typing the filters

Once the data is typed, everything from [[JSON (JavaScript)]] works with full inference.

```ts
const adults: User[] = users.filter(u => u.age > 30);

// map to a narrower type
const names: string[] = users.map(u => u.name);

// TypeScript infers this as { id: number; name: string }[]
const summary = users.map(u => ({ id: u.id, name: u.name }));

// filter + narrow: this predicate removes null from the type
const withEmail = users.filter(
    (u): u is User & { email: string } => u.email !== null
);
// withEmail[0].email is `string`, not `string | null`
```

That last pattern is the one worth learning. A plain `.filter(u => u.email !== null)` still gives you `User[]` with `email: string | null` — TypeScript cannot see through the callback. The `u is ...` annotation fixes it.

Typing unknown-shaped JSON

For genuinely arbitrary JSON, this is the honest recursive type:

```ts
type Json =
    | string
    | number
    | boolean
    | null
    | Json[]
    | { [key: string]: Json };
```

Better than `any` because it forces you to narrow before use, and better than `object` because it actually describes JSON.

Optional vs nullable

These are different and APIs use both.

```ts
interface User {
    email: string | null;      // key is always present, value may be null
    phone?: string;            // key may be missing entirely
    fax?: string | null;       // may be missing OR null. joy.
}
```

In Zod:

```ts
import { z } from "zod";

z.object({
    email: z.string().nullable(),      // string | null
    phone: z.string().optional(),      // string | undefined
    fax:   z.string().nullish(),       // string | null | undefined
});
```

Check your saved samples to see which one you actually have — the `to_entries` trick in [[Investigating an API]] shows you when a field comes back null.

Generating types from a sample

```bash
npx quicktype -s json -o types.ts --lang ts samples/users.json
npx json-schema-to-typescript schema.json > types.ts
```

Always review the output. Generators mark a field required if it was present in the one sample you fed them, which is usually too strict.

Importing JSON files

```ts
// needs "resolveJsonModule": true in tsconfig
import config from "./config.json";
```

TypeScript infers the type from the file's literal contents, which is genuinely useful for static config. See [[tsconfig and compiling (TypeScript)]].

satisfies, for config objects

```ts
const config = {
    port: 3000,
    host: "localhost",
} satisfies Record<string, string | number>;

config.port.toFixed();     // still known to be `number`, not `string | number`
```

`satisfies` checks the object against a type without widening it, so you keep the precise literal types. Better than `as` or an annotation for anything you both validate and read.

See [[Unions and Narrowing (TypeScript)]] for the narrowing rules behind the type guards, and [[jq]] for exploring the response before you write the schema.
