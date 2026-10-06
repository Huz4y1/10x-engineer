---
tags: [nextjs, react, typescript, reference, web]
---

# Next.js reference

The complete API. Your notes: [[NextJs TypeScript]] · Stack: [[Primary Web App Stack]] · Styling: [[Tailwindcss]]

```bash
npx create-next-app@latest myapp --typescript --tailwind --app
```

---

## App Router — file-based routing

```
app/
├── layout.tsx           # root layout - REQUIRED, wraps everything
├── page.tsx             # route: /
├── loading.tsx          # shown while the page loads
├── error.tsx            # shown when it throws
├── not-found.tsx        # 404
├── dashboard/
│   ├── layout.tsx       # nested layout
│   └── page.tsx         # route: /dashboard
├── items/
│   └── [id]/page.tsx    # route: /items/42
├── (marketing)/         # GROUP - organises files, adds NO url segment
│   └── about/page.tsx   # route: /about
└── api/
    └── items/route.ts   # route: /api/items
```

| File | Purpose |
|---|---|
| `page.tsx` | The route's UI |
| `layout.tsx` | Wraps children, **preserves state across navigation** |
| `loading.tsx` | Automatic Suspense boundary |
| `error.tsx` | Error boundary — must be a Client Component |
| `route.ts` | API endpoint |
| `[id]` | Dynamic segment |
| `[...slug]` | Catch-all |
| `(name)` | Group — no URL segment |

## Server vs Client Components

**This is the concept that matters most in modern Next.js.**

```tsx
// app/items/page.tsx - a SERVER component by default
async function ItemsPage() {
  const items = await db.query("SELECT * FROM items");   // runs on the SERVER
  return <ItemList items={items} />;
}
export default ItemsPage;
```

```tsx
"use client";                    // <- opts into the browser
import { useState } from "react";

export function Counter() {
  const [n, setN] = useState(0);
  return <button onClick={() => setN(n + 1)}>{n}</button>;
}
```

| | Server Component (default) | Client Component (`"use client"`) |
|---|---|---|
| Runs | On the server | In the browser |
| Can `await` directly | ✅ | ❌ |
| Can use `useState`/`useEffect` | ❌ | ✅ |
| Can use `onClick` | ❌ | ✅ |
| Ships JS to the browser | **No** | Yes |
| Can read secrets / query the DB | ✅ | ❌ **Never** |

> **Default to Server Components. Add `"use client"` only where you need interactivity** — state, effects, event handlers, browser APIs. Every `"use client"` adds JavaScript the user downloads.

> ⚠️ **Never put a secret in a Client Component.** It ships to the browser in plain text. Only `NEXT_PUBLIC_*` env vars reach the client, and those are public by definition.

**The pattern:** keep pages as Server Components, push `"use client"` down to the smallest leaf that needs it.

## Data fetching

```tsx
// Server Component - just await
async function Page() {
  const res = await fetch("https://api.example.com/items", {
    next: { revalidate: 60 },        // cache for 60s
  });
  const items = await res.json();
  return <ul>{items.map((i) => <li key={i.id}>{i.name}</li>)}</ul>;
}
```

| Option | Behaviour |
|---|---|
| `cache: "force-cache"` | Cache indefinitely (default in older versions) |
| `cache: "no-store"` | Never cache — always fresh |
| `next: { revalidate: 60 }` | ISR — regenerate at most every 60s |
| `next: { tags: ["items"] }` | Tag for on-demand invalidation |

```tsx
import { revalidateTag, revalidatePath } from "next/cache";
revalidateTag("items");
revalidatePath("/dashboard");
```

> **Caching defaults changed between Next.js versions.** If data is unexpectedly stale — or unexpectedly not cached — check your version's default before debugging anything else.

## Server Actions — mutations without an API route

```tsx
// app/actions.ts
"use server";

export async function createItem(formData: FormData) {
  const name = formData.get("name") as string;
  await db.insert({ name });
  revalidatePath("/items");
}
```

```tsx
import { createItem } from "./actions";

export default function Page() {
  return (
    <form action={createItem}>
      <input name="name" required />
      <button type="submit">Add</button>
    </form>
  );
}
```

> **This works without JavaScript enabled** — it's a real form post. Progressive enhancement for free.

> ⚠️ **A Server Action is a public HTTP endpoint.** Anyone can call it with any payload. **Validate and authorise inside it**, exactly as you would an API route — [[Pydantic]]'s TypeScript equivalent is Zod ([[TypeScript libraries]]).

## Route handlers (API routes)

```ts
// app/api/items/route.ts
import { NextRequest, NextResponse } from "next/server";

export async function GET(req: NextRequest) {
  const category = req.nextUrl.searchParams.get("category");
  const items = await db.getItems(category);
  return NextResponse.json(items);
}

export async function POST(req: NextRequest) {
  const body = await req.json();
  const parsed = ItemSchema.safeParse(body);          // Zod validation
  if (!parsed.success) {
    return NextResponse.json({ error: parsed.error.issues }, { status: 400 });
  }
  return NextResponse.json(await db.create(parsed.data), { status: 201 });
}
```

```ts
// app/api/items/[id]/route.ts
export async function GET(req: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;                        // params is async in Next 15+
  return NextResponse.json(await db.get(id));
}
```

> **In your stack the backend is [[Actix Web]]**, so route handlers are mainly for webhooks, auth callbacks, and proxying — not your main API.

## Navigation

```tsx
import Link from "next/link";
import { useRouter, usePathname, useSearchParams } from "next/navigation";
import { redirect } from "next/navigation";

<Link href="/dashboard" prefetch>Dashboard</Link>

// client components only
const router = useRouter();
router.push("/items/42");
router.replace("/login");
router.refresh();               // re-fetch server components, keep client state

// server components
redirect("/login");
```

> **Use `<Link>`, not `<a>`.** It prefetches on hover and does client-side navigation. A plain `<a>` triggers a full page reload.

## Rendering strategies

| Strategy | How | Use for |
|---|---|---|
| **Static (SSG)** | Default when there's no dynamic data | Marketing pages, docs |
| **ISR** | `next: { revalidate: n }` | Content that changes occasionally |
| **Dynamic (SSR)** | `cache: "no-store"`, or using `cookies()`/`headers()` | Per-user pages |
| **Client** | `"use client"` + `useEffect` | Highly interactive widgets |

```tsx
export const dynamic = "force-dynamic";      // force SSR for this route
export const revalidate = 3600;              // ISR for this route
```

```tsx
export async function generateStaticParams() {
  const items = await db.getAll();
  return items.map((i) => ({ id: String(i.id) }));   // pre-render these paths
}
```

## Metadata and SEO

```tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: "Dashboard",
  description: "Engine analytics",
  openGraph: { title: "Dashboard", images: ["/og.png"] },
};

export async function generateMetadata({ params }): Promise<Metadata> {
  const item = await db.get((await params).id);
  return { title: item.name };
}
```

## Images and fonts

```tsx
import Image from "next/image";
<Image src="/chart.png" alt="Revenue" width={800} height={600} priority />

import { Inter } from "next/font/google";
const inter = Inter({ subsets: ["latin"] });
<body className={inter.className}>
```

> **`next/image` gives automatic resizing, lazy loading and modern formats.** `width`/`height` are required — they prevent layout shift ([[HTML]]).

## Environment variables

```bash
# .env.local  (gitignored)
DATABASE_URL=postgres://...          # server only
NEXT_PUBLIC_API_URL=https://api...   # shipped to the BROWSER
```

> ⚠️ **`NEXT_PUBLIC_*` is public.** It's baked into the JavaScript bundle. Anything without that prefix is server-only — and that's where secrets go.

## Middleware

```ts
// middleware.ts (project root)
import { NextResponse, type NextRequest } from "next/server";

export function middleware(req: NextRequest) {
  if (!req.cookies.get("token") && req.nextUrl.pathname.startsWith("/dashboard")) {
    return NextResponse.redirect(new URL("/login", req.url));
  }
  return NextResponse.next();
}

export const config = { matcher: ["/dashboard/:path*"] };
```

> Middleware runs on the **edge runtime** — no Node APIs, no database drivers. Keep it to redirects, headers and cookie checks.

## Loading and errors

```tsx
// app/dashboard/loading.tsx
export default function Loading() { return <p>Loading…</p>; }

// app/dashboard/error.tsx
"use client";
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return <><p>{error.message}</p><button onClick={reset}>Retry</button></>;
}
```

```tsx
import { Suspense } from "react";
<Suspense fallback={<Skeleton />}><SlowComponent /></Suspense>
```

## Common mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| `useState` in a Server Component | Build error | `"use client"` |
| Secret in a Client Component | **Leaked to the browser** | Keep it server-side |
| `"use client"` at the top of every page | Ships far too much JS | Push it to leaves |
| `<a>` instead of `<Link>` | Full page reloads | `<Link>` |
| Data unexpectedly stale | Caching defaults | Set `cache`/`revalidate` explicitly |
| Server Action without validation | **Public unvalidated endpoint** | Validate with Zod |
| `params` used without `await` (Next 15+) | Runtime error | `await params` |
| Missing `key` in a list | React warning, subtle bugs | `key={item.id}` |
| `NEXT_PUBLIC_` on a secret | Public | Drop the prefix |

## Related

[[NextJs TypeScript]] · [[TypeScript]] · [[TypeScript libraries]] · [[Tailwindcss]] · [[HTML]] · [[CSS]] · [[Actix Web]] · [[Primary Web App Stack]] · [[Accessibility]]
