---
tags: [css, responsive, mobile, web]
---

# Responsive design

Making one page work on a phone and a monitor. Language: [[CSS]]

---

## The one tag that makes it possible

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

> **Without this, a phone renders your page at 980px wide and zooms out.** Every responsive rule below does nothing until this tag is present. It goes in every [[HTML]] `<head>`.

## Mobile first

```css
/* base styles = MOBILE */
.container { padding: 1rem; }

/* then ADD complexity for bigger screens */
@media (min-width: 768px)  { .container { padding: 2rem; } }
@media (min-width: 1200px) { .container { max-width: 1140px; margin: 0 auto; } }
```

> **Write mobile styles first, then `min-width` queries to enhance.** The alternative (desktop first with `max-width`) means overriding things repeatedly and fighting yourself.

## Breakpoints

```css
@media (min-width: 640px)  { }   /* large phone */
@media (min-width: 768px)  { }   /* tablet */
@media (min-width: 1024px) { }   /* laptop */
@media (min-width: 1280px) { }   /* desktop */
```

> **Choose breakpoints where your layout breaks, not by device.** Resize the browser until it looks wrong — that's your breakpoint.

Other useful queries:
```css
@media (prefers-color-scheme: dark) { }
@media (prefers-reduced-motion: reduce) { *, *::before { animation: none !important; } }
@media print { nav, .no-print { display: none; } }
@media (hover: hover) { .card:hover { transform: translateY(-2px); } }
```

> **`prefers-reduced-motion` matters.** Animation can cause genuine nausea for some people ([[Accessibility]]).

## Fluid without media queries

```css
width: min(100%, 1200px);           /* never wider than 1200, shrinks below */
width: clamp(300px, 50%, 800px);    /* min, preferred, max */
font-size: clamp(1rem, 2.5vw, 2rem);   /* scales with the viewport, bounded */
padding: max(1rem, 3vw);
```

> **`clamp()` removes most media queries.** One line replaces three breakpoints for typography and spacing.

## Units

| Unit | Relative to | Use for |
|---|---|---|
| `px` | Nothing | Borders, tiny fixed things |
| **`rem`** | Root font size | **Spacing and type — the default** |
| `em` | The element's own font size | Padding that scales with its text |
| `%` | Parent | Widths |
| `vw` / `vh` | Viewport | Full-screen sections |
| `dvh` | **Dynamic** viewport height | **Mobile — accounts for the browser bar** |
| `fr` | Free space in a grid | [[Grid]] |
| `ch` | Width of "0" | Line length (`max-width: 65ch`) |

> **Use `rem` for spacing and type.** It respects the user's browser font size; `px` ignores it, which breaks the page for anyone who has increased their default.

> **`100vh` is broken on mobile** — it doesn't account for the address bar, so content gets cut off. Use `100dvh`.

## Responsive layout without queries

```css
/* cards: as many as fit, min 280px each */
.cards { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 1rem; }

/* sidebar that stacks when narrow */
.layout { display: flex; flex-wrap: wrap; gap: 1rem; }
.sidebar { flex: 1 1 250px; }
.main    { flex: 3 1 500px; }
```

> **[[Grid]]'s `auto-fit` + `minmax` and [[Flexbox]]'s `flex-wrap` handle most responsiveness with no media queries at all.**

## Images

```html
<img src="photo.jpg" alt="..." loading="lazy" width="800" height="600">
```

```css
img { max-width: 100%; height: auto; display: block; }
```

> **`max-width: 100%` on all images**, or one wide image gives the whole page a horizontal scrollbar.

## Touch targets

```css
button, a { min-height: 44px; min-width: 44px; }
```

> **44×44px minimum for anything tappable.** Smaller and people miss it on a phone.

## Testing

| Method | Catches |
|---|---|
| **Resize the browser slowly** | Where the layout breaks |
| DevTools device toolbar | Specific device sizes |
| **A real phone** | Touch, performance, the address bar |
| Zoom to 200% | Accessibility requirement |

## Checklist

- [ ] Viewport meta tag
- [ ] Mobile-first `min-width` queries
- [ ] `rem` for spacing and type
- [ ] `max-width: 100%` on images
- [ ] No horizontal scroll at 320px wide
- [ ] Touch targets ≥ 44px
- [ ] `dvh` not `vh` for full-height on mobile
- [ ] `prefers-reduced-motion` respected

## Related

[[CSS]] · [[Flexbox]] · [[Grid]] · [[HTML]] · [[Accessibility]] · [[Tailwindcss]]
