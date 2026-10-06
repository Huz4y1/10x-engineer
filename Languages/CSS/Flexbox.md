---
tags: [css, layout, flexbox, web]
---

# Flexbox

One-dimensional layout — a row **or** a column. Language: [[CSS]] · Two dimensions: [[Grid]]

---

## The idea

You have a container and some children. Flexbox controls how the children are **arranged along one line** and how leftover space is shared.

```css
.container {
  display: flex;
}
```

That one line puts the children in a row. Everything else is refinement.

## Container properties

```css
.container {
  display: flex;
  flex-direction: row;          /* row | row-reverse | column | column-reverse */
  justify-content: center;      /* along the MAIN axis */
  align-items: center;          /* along the CROSS axis */
  flex-wrap: wrap;              /* allow wrapping onto new lines */
  gap: 1rem;                    /* space between children - use this, not margins */
  align-content: center;        /* wrapped LINES along the cross axis */
}
```

> **The axis rule:** `justify-content` works along `flex-direction`. `align-items` works across it. Switch to `flex-direction: column` and the two swap meaning — which is the single most confusing thing about flexbox.

### justify-content

```
flex-start     |■■■        |
center         |   ■■■     |
flex-end       |        ■■■|
space-between  |■    ■    ■|      <- first and last touch the edges
space-around   | ■   ■   ■ |
space-evenly   |  ■  ■  ■  |
```

### align-items

```css
align-items: stretch;      /* default - children fill the cross axis */
align-items: center;       /* the classic vertical centring */
align-items: flex-start;
align-items: baseline;     /* align text baselines */
```

## Child properties

```css
.item {
  flex-grow: 1;             /* share of LEFTOVER space */
  flex-shrink: 1;           /* how much it may shrink */
  flex-basis: 200px;        /* starting size before growing/shrinking */
  flex: 1;                  /* shorthand for: 1 1 0  <- the one you'll use */
  align-self: flex-end;     /* override align-items for THIS child */
  order: 2;                 /* visual reorder - does NOT change tab order */
}
```

> **`flex: 1` on every child makes them equal width.** `flex: 2` on one makes it take twice the share. That's 90% of the flexbox you'll write.

> ⚠️ **`order` changes the visual order only.** Keyboard tab order still follows the HTML. Reordering with `order` can make a page confusing to navigate ([[Accessibility]]).

## The patterns you'll actually use

**Centre something, both axes**
```css
.parent { display: flex; justify-content: center; align-items: center; min-height: 100vh; }
```

**Nav bar — logo left, links right**
```css
.nav { display: flex; justify-content: space-between; align-items: center; }
```

**Sidebar + main**
```css
.layout { display: flex; gap: 1rem; }
.sidebar { flex: 0 0 250px; }    /* fixed 250px, never grows or shrinks */
.main    { flex: 1; }            /* takes everything else */
```

**Card row that wraps**
```css
.cards { display: flex; flex-wrap: wrap; gap: 1rem; }
.card  { flex: 1 1 300px; }      /* at least 300px, grow to fill, wrap when tight */
```

**Push one item to the end**
```css
.last { margin-left: auto; }     /* auto margins absorb free space */
```

## Flexbox or Grid?

| Use **Flexbox** | Use **[[Grid]]** |
|---|---|
| One direction — a row or a column | Two directions — rows **and** columns |
| Content decides the size | You define the structure |
| Nav bars, button rows, card rows | Page layouts, dashboards, galleries |

> **Rule of thumb: Flexbox for components, Grid for page layout.** They compose — a Grid page with Flexbox inside each cell is completely normal.

## Common mistakes

| Mistake | Fix |
|---|---|
| Using margins for spacing | `gap` |
| `align-items` doing nothing | The container has no cross-axis size — set a height |
| Items overflowing instead of wrapping | `flex-wrap: wrap` |
| Confused axes | Check `flex-direction` first |
| Item won't shrink below content | `min-width: 0` on the flex item |

## Related

[[CSS]] · [[Grid]] · [[Responsive design]] · [[HTML]] · [[Tailwindcss]]
