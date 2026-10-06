---
tags: [css, layout, grid, web]
---

# Grid

Two-dimensional layout — rows **and** columns at once. Language: [[CSS]] · One dimension: [[Flexbox]]

---

## The idea

Flexbox lays things out along a line. **Grid gives you a proper table of cells** and lets you place things into them.

```css
.container {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;   /* three equal columns */
  gap: 1rem;
}
```

## Defining the grid

```css
grid-template-columns: 200px 1fr 200px;         /* fixed, flexible, fixed */
grid-template-columns: repeat(3, 1fr);          /* three equal */
grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));   /* responsive, no media query */
grid-template-rows: auto 1fr auto;              /* header, body, footer */
gap: 1rem;  /  row-gap: 1rem;  column-gap: 2rem;
```

> **`fr` is "one share of the free space".** `1fr 2fr` gives a 1:2 split. It's the unit Grid was designed around.

> **`repeat(auto-fit, minmax(250px, 1fr))` is the single most useful line in CSS Grid.** It gives you a responsive card layout — as many columns as fit, each at least 250px, no media queries at all.

## Placing items

```css
.item {
  grid-column: 1 / 3;          /* from line 1 to line 3 = spans 2 columns */
  grid-column: span 2;         /* span 2 from wherever it lands */
  grid-row: 1 / -1;            /* first line to LAST line - full height */
  grid-area: header;           /* named area */
}
```

> **Grid lines are numbered from 1, and `-1` is the last line.** `1 / -1` means "full width", whatever the column count.

## Named areas — the readable way

```css
.layout {
  display: grid;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
  grid-template-columns: 250px 1fr;
  grid-template-rows: auto 1fr auto;
  min-height: 100vh;
  gap: 1rem;
}

.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
.main    { grid-area: main; }
.footer  { grid-area: footer; }
```

> **You can read the layout straight out of the CSS.** For page-level structure this is far clearer than line numbers.

## Alignment

```css
justify-items: center;    /* items within their cell, horizontally */
align-items: center;      /* items within their cell, vertically */
place-items: center;      /* shorthand for both - the easiest centring in CSS */

justify-content: center;  /* the whole GRID within the container */
align-content: center;

justify-self / align-self / place-self    /* per item */
```

```css
.centre { display: grid; place-items: center; min-height: 100vh; }
```

> **`display: grid; place-items: center;` is the shortest way to centre anything.** Two lines.

## The patterns

**Responsive cards, no media queries**
```css
.cards {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.5rem;
}
```

**Dashboard**
```css
.dashboard {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 1rem;
}
.chart-wide { grid-column: span 2; }
.chart-full { grid-column: 1 / -1; }
```

**Holy grail layout** — the named-areas example above.

## Grid or Flexbox?

| **Grid** | **[[Flexbox]]** |
|---|---|
| Two dimensions | One dimension |
| You define the structure | Content decides |
| Page layouts, dashboards, galleries | Nav bars, button groups, card contents |

> **They work together.** Grid for the page skeleton, Flexbox inside each region. That's the normal combination.

## Common mistakes

| Mistake | Fix |
|---|---|
| Using `%` widths | Use `fr` |
| Media queries for card layouts | `auto-fit` + `minmax` |
| Content overflowing a cell | `minmax(0, 1fr)` — `1fr` won't shrink below content |
| Forgetting `gap` | It's the whole spacing mechanism |
| Grid for a single row of buttons | That's [[Flexbox]] |

## Related

[[CSS]] · [[Flexbox]] · [[Responsive design]] · [[HTML]] · [[Tailwindcss]]
