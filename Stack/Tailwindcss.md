---
tags: [css, tailwind, styling, web]
---

# Tailwind CSS

Utility-first styling. Language: [[CSS]] · Libraries: [[CSS libraries]] · Mobile: [[CSS for React Native]]

[[TWRNC (Tailwind for React Native)]]

---

## The idea

Instead of writing CSS in a separate file and inventing class names, you compose **utility classes** in the markup.

```html
<!-- traditional -->
<div class="card">...</div>
<style>.card { display:flex; align-items:center; gap:1rem; padding:1.5rem; border-radius:.5rem; }</style>

<!-- Tailwind -->
<div class="flex items-center gap-4 p-6 rounded-lg">...</div>
```

> **You never leave the HTML and never invent a class name.** The cost is verbose markup — which is fine inside a component ([[NextJs TypeScript]]), because the verbosity is contained.

## Setup

```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

```js
// tailwind.config.js
export default {
  content: ["./app/**/*.{ts,tsx}", "./components/**/*.{ts,tsx}"],   // <- MUST match your files
  theme: {
    extend: {
      colors: { brand: { DEFAULT: "#00a0a0", dark: "#007070" } },
    },
  },
};
```

```css
/* globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

> ⚠️ **`content` must match where your classes actually are.** Tailwind scans those files and deletes every unused class. If styles vanish in production but work in dev, this is why.

> ⚠️ **Never build class names dynamically** — `` `text-${color}-500` `` doesn't exist in the source, so Tailwind strips it. Use complete strings and pick between them.

## The classes you'll use constantly

**Layout** ([[Flexbox]] · [[Grid]])
```
flex  grid  block  hidden  inline-flex
flex-row  flex-col  flex-wrap
items-center  justify-between  justify-center
gap-2  gap-4  gap-8
grid-cols-3  col-span-2
```

**Spacing** — the scale is `1 = 0.25rem = 4px`
```
p-4  px-6  py-2  pt-8        padding
m-4  mx-auto  mt-2           margin
space-y-4                    gap between children
```

**Sizing**
```
w-full  w-1/2  w-64  max-w-4xl  min-h-screen
h-full  h-screen  size-10
```

**Text**
```
text-sm  text-lg  text-2xl
font-medium  font-bold
text-slate-900  text-white
text-center  truncate  leading-relaxed
```

**Colour and borders**
```
bg-slate-800  bg-brand  bg-transparent
border  border-2  border-slate-700
rounded  rounded-lg  rounded-full
shadow  shadow-lg
```

**State and responsive** — the prefix system
```
hover:bg-slate-700
focus:ring-2  focus-visible:outline-none
disabled:opacity-50
dark:bg-slate-900
md:flex-row  lg:grid-cols-3
group-hover:text-white
```

> **Prefixes stack and are mobile-first**: `flex-col md:flex-row` means column on phones, row from medium screens up. Same principle as [[Responsive design]].

## A realistic component

```tsx
export function Card({ title, value, delta }: Props) {
  return (
    <div className="flex flex-col gap-2 rounded-lg border border-slate-700
                    bg-slate-800 p-6 shadow-lg transition hover:border-slate-500">
      <span className="text-sm font-medium text-slate-400">{title}</span>
      <span className="text-3xl font-semibold text-white">{value}</span>
      <span className={delta >= 0 ? "text-emerald-400" : "text-rose-400"}>
        {delta >= 0 ? "+" : ""}{delta}%
      </span>
    </div>
  );
}
```

## Managing long class lists

```bash
npm install clsx tailwind-merge
```

```ts
import clsx from "clsx";
import { twMerge } from "tailwind-merge";

export const cn = (...inputs: any[]) => twMerge(clsx(inputs));
```

```tsx
<button className={cn(
  "rounded px-4 py-2 font-medium",
  variant === "primary" && "bg-brand text-white",
  variant === "ghost"   && "bg-transparent text-slate-300",
  disabled && "opacity-50 cursor-not-allowed",
  className,                                   // caller can override
)} />
```

> **`twMerge` resolves conflicts.** `cn("p-2", "p-4")` gives `p-4` — without it you'd get both and CSS order decides, unpredictably.

## Reusable patterns

```css
@layer components {
  .btn { @apply rounded px-4 py-2 font-medium transition; }
  .btn-primary { @apply btn bg-brand text-white hover:bg-brand-dark; }
}
```

> **Use `@apply` sparingly.** It recreates the CSS-file problem Tailwind exists to avoid. Prefer a **component** with the classes inside it.

## Tooling

| Tool | For |
|---|---|
| **Tailwind CSS IntelliSense** (VS Code) | Autocomplete and hover previews — essential |
| `prettier-plugin-tailwindcss` | Sorts classes into a consistent order |
| **shadcn/ui** | Copy-paste accessible components built on Tailwind |

## Common mistakes

| Mistake | Fix |
|---|---|
| Styles missing in production | `content` paths in the config |
| Dynamic class names stripped | Use complete literal strings |
| Conflicting classes | `tailwind-merge` |
| Enormous class lists | Extract a component |
| `@apply` everywhere | Components instead |
| Arbitrary values everywhere (`w-[347px]`) | Use the scale |

## Related

[[CSS]] · [[CSS libraries]] · [[NextJs TypeScript]] · [[Flexbox]] · [[Grid]] · [[Responsive design]] · [[HTML]]
