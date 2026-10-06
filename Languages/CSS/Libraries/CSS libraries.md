---
tags: [css, libraries, tailwind, web]
---

# CSS libraries

Language: [[CSS]]

---

## Utility-first

| Library | Note |
|---|---|
| **Tailwind CSS** | [[Tailwindcss]] — **your stack's choice** |
| UnoCSS | Faster Tailwind-compatible engine |

```html
<div class="flex items-center justify-between gap-4 p-6 rounded-lg bg-slate-800">
```

> **Utility classes mean you never leave the HTML** and never invent class names. The trade: markup gets verbose. For component-based work ([[NextJs TypeScript]]) that verbosity is contained inside the component, which is why it works well there.

## Component libraries

| Library | For |
|---|---|
| **shadcn/ui** | Copy-paste React components built on Tailwind — you own the code |
| Radix UI | Unstyled, fully accessible primitives |
| Headless UI | Same idea, from the Tailwind team |
| Material UI | Google Material Design, React |
| DaisyUI | Component classes on top of Tailwind |

> **Prefer headless libraries (Radix, Headless UI) plus your own styling.** You get correct keyboard handling and ARIA — which is genuinely hard to write ([[Accessibility]]) — without inheriting someone else's visual design.

## Mobile

| Library | Note |
|---|---|
| **TWRNC** | Tailwind for React Native — [[CSS for React Native]] |
| NativeWind | Alternative Tailwind for RN |

> **React Native isn't CSS.** It's a JS style object subset — Flexbox works, most other things don't ([[Expo]]).

## Classic frameworks

| Library | Note |
|---|---|
| Bootstrap | Component-first, fast prototyping, distinctive look |
| **Bulma** | No JavaScript, pure CSS, readable class names — **full guide: [[Flask - styling with Bulma]]** |
| Pico.css | Classless — style semantic HTML with no classes at all |

## Preprocessors and tooling

| Tool | For |
|---|---|
| **PostCSS** | Plugin pipeline — Tailwind runs on it |
| Autoprefixer | Vendor prefixes automatically |
| Sass/SCSS | Nesting, variables, mixins |
| CSS Modules | Scoped class names |

> **Modern CSS has absorbed most of what Sass was for** — native nesting, `var()` custom properties, `calc()`. Reach for a preprocessor only if the project already uses one.

## Native CSS variables

```css
:root {
  --primary: #00a0a0;
  --spacing: 1rem;
}
.card { color: var(--primary); padding: var(--spacing); }

@media (prefers-color-scheme: dark) {
  :root { --bg: #0e1117; }
}
```

> **Custom properties are live** — change one in JS and everything using it updates. Sass variables are compile-time and can't do that. This is how theme switching works.

## Related

[[CSS]] · [[Tailwindcss]] · [[HTML]] · [[NextJs TypeScript]] · [[Flexbox]] · [[Grid]]
