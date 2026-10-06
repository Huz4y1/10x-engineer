---
tags: [moc, typescript, libraries]
---

# TypeScript libraries

Language: [[TypeScript]] · Untyped base: [[JavaScript]]

---

TypeScript uses the same ecosystem as [[JavaScript libraries]] — these are the ones where types matter most.

| Library | For | Note |
|---|---|---|
| **Zod** | Runtime validation that infers static types | The TS [[Pydantic]] |
| **tRPC** | End-to-end typesafe APIs, no codegen | — |
| **Prisma** | Typed database ORM | — |
| **TanStack Query** | Typed server state | — |
| **Next.js** | React framework | [[NextJs TypeScript]] |
| **Expo** | React Native | [[Expo]] |
| **Supabase JS** | Typed Supabase client | [[Supabase]] |

## The type layer

| Tool | For |
|---|---|
| `tsc` | The compiler |
| `@types/*` | Types for untyped JS libraries |
| `tsconfig.json` | **`"strict": true`** — always |

> **`"strict": true` from day one.** Turning it on later in a large codebase is a genuinely painful migration.

> **Zod is the pattern to learn.** Define a schema once, get both runtime validation *and* a static type — exactly what [[Pydantic]] does in Python.

## Related

[[TypeScript]] · [[JavaScript libraries]] · [[NextJs TypeScript]] · [[Primary Web App Stack]]
