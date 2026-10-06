---
tags: [html, accessibility, a11y, web]
---

# Accessibility

Language: [[HTML]] · Also called **a11y**

---

## Why it matters

Roughly 1 in 5 people have a disability. Accessibility is also **legally required** for many public and commercial sites in the UK and EU.

And it's cheap: most of it is using the right HTML tag rather than adding anything.

## The five that fix most problems

**1. Semantic HTML.** `<button>` not `<div onclick>`. A real button is focusable, keyboard-activatable and announced as a button — for free.

**2. `alt` on every image.**
```html
<img src="chart.png" alt="Revenue rose 8% in Q3">   <!-- meaningful -->
<img src="divider.png" alt="">                       <!-- decorative -->
```

**3. Labels on every input.**
```html
<label for="email">Email</label><input id="email">
```

**4. Colour contrast.** At least **4.5:1** for body text, 3:1 for large text. Never use colour *alone* to convey meaning — add an icon or text.

**5. Keyboard navigable.** Tab through your whole page. If you can't reach or activate something, it's broken.

> **Test it: unplug your mouse and use the site.** That single exercise finds most keyboard problems in about two minutes.

## ARIA — only when HTML can't

```html
<button aria-label="Close">×</button>                <!-- no visible text -->
<div role="alert">Saved</div>                         <!-- announce immediately -->
<button aria-expanded="false" aria-controls="menu">Menu</button>
<input aria-describedby="help"><small id="help">1-100</small>
<div aria-live="polite">Updated 3 seconds ago</div>   <!-- announce on change -->
<span aria-hidden="true">🔥</span>                     <!-- hide decoration -->
```

> **The first rule of ARIA: don't use ARIA.** A native `<button>` beats `<div role="button" tabindex="0">` every time. ARIA is for filling gaps HTML genuinely can't cover — live regions, custom widgets.

## Focus

```css
:focus-visible {
  outline: 2px solid #0066cc;
  outline-offset: 2px;
}
```

> ⚠️ **Never `outline: none` without a replacement.** Removing the focus ring makes a site unusable by keyboard — and it's one of the most common accessibility failures on the web.

```html
<a href="#main" class="skip-link">Skip to content</a>
```

## Headings

```html
<h1>Page title</h1>          <!-- ONE per page -->
  <h2>Section</h2>
    <h3>Subsection</h3>
```

> **Don't skip levels** (h1 → h3). Screen reader users navigate by heading list — skipping breaks the outline. Choose heading level by *structure*, size by [[CSS]].

## Checking it

| Tool | For |
|---|---|
| **Keyboard only** | The fastest real test |
| Browser DevTools → Lighthouse | Automated audit |
| axe DevTools | Browser extension |
| NVDA (Win) / VoiceOver (Mac) | Actual screen readers |
| WebAIM Contrast Checker | Colour ratios |

> **Automated tools catch maybe 30%.** They can't tell whether your `alt` text is *meaningful* or your tab order makes sense. Use them as a floor, not a pass.

## Quick checklist

- [ ] `lang` on `<html>`
- [ ] One `<h1>`, no skipped levels
- [ ] `alt` on every image
- [ ] `<label for>` on every input
- [ ] Keyboard reaches everything
- [ ] Visible focus indicator
- [ ] Contrast ≥ 4.5:1
- [ ] Colour isn't the only signal
- [ ] Semantic landmarks (`<main>`, `<nav>`)
- [ ] Forms announce their errors

## Related

[[HTML]] · [[Forms (HTML)]] · [[CSS]] · [[NextJs TypeScript]]
