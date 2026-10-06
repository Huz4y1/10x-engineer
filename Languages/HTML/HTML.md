---
tags: [html, language, web, moc]
---

# HTML

The structure of a web page. Language index: [[Languages]] · Styling: [[CSS]] · Behaviour: [[JavaScript]]

**Deeper notes:** [[Forms (HTML)]] · [[Accessibility]] · [[HTML tooling]]

---

## The idea in plain English

HTML says **what things are**. CSS says what they look like. JavaScript says what they do.

```html
<h1>Engine Report</h1>          <!-- this is a heading -->
<p>Temperature is high.</p>     <!-- this is a paragraph -->
<button>Refresh</button>        <!-- this is a button -->
```

Nothing about colour, size or position — that's [[CSS]]'s job. Keeping them separate is the whole point.

## The skeleton

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Engine Dashboard</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <h1>Engine Dashboard</h1>
  <script src="app.js"></script>
</body>
</html>
```

| Line | Why |
|---|---|
| `<!DOCTYPE html>` | Tells the browser to use standards mode. Omit it and layout breaks in odd ways. |
| `lang="en"` | Screen readers and translation need it |
| `charset="UTF-8"` | **Or accented characters render as mojibake** |
| `viewport` | **Or the page doesn't work on phones** |
| `<script>` at the end of body | So the HTML renders before JS runs |

> **Those two meta tags are non-negotiable.** Missing `charset` gives you `Ã©` instead of `é`; missing `viewport` gives you a desktop page zoomed out on a phone.

## Semantic structure

```html
<header>   <!-- site or section header -->
<nav>      <!-- navigation links -->
<main>     <!-- the main content - ONE per page -->
<section>  <!-- a thematic grouping -->
<article>  <!-- standalone content -->
<aside>    <!-- sidebar, related content -->
<footer>   <!-- footer -->
```

```html
<body>
  <header><nav>...</nav></header>
  <main>
    <section>
      <h2>Readings</h2>
      <article>...</article>
    </section>
  </main>
  <footer>...</footer>
</body>
```

> **Use semantic tags, not `<div>` for everything.** Screen readers navigate by landmark, search engines read structure, and your CSS gets simpler. `<div>` means "I have no better word for this".

## Text

```html
<h1> ... <h6>              <!-- ONE h1 per page, don't skip levels -->
<p>Paragraph</p>
<strong>important</strong>  <!-- meaning: important -->
<em>emphasis</em>           <!-- meaning: stressed -->
<b>bold</b> <i>italic</i>   <!-- appearance only - prefer strong/em -->
<br>  <hr>
<blockquote cite="...">Quote</blockquote>
<code>inline code</code>
<pre><code>block of code</code></pre>
<span>inline, no meaning</span>
<div>block, no meaning</div>
```

## Links and images

```html
<a href="/page">Internal</a>
<a href="https://x.com" target="_blank" rel="noopener noreferrer">External</a>
<a href="#section">Jump to an id</a>
<a href="mailto:a@b.com">Email</a>

<img src="chart.png" alt="Revenue by month" width="600" height="400" loading="lazy">
```

> ⚠️ **`rel="noopener noreferrer"` on every `target="_blank"`.** Without it the new page can manipulate the page that opened it — a real security issue ([[26 — SECURITY]]).

> ⚠️ **`alt` is not optional.** It's what a blind user hears and what shows when the image fails. Decorative image? `alt=""` — empty, but present.

> **`width`/`height` on images** stop the page jumping around as they load. `loading="lazy"` defers off-screen ones.

## Lists and tables

```html
<ul><li>Unordered</li></ul>
<ol><li>Ordered</li></ol>
<dl><dt>Term</dt><dd>Definition</dd></dl>

<table>
  <caption>Daily readings</caption>
  <thead>
    <tr><th scope="col">Date</th><th scope="col">Temp</th></tr>
  </thead>
  <tbody>
    <tr><td>2026-01-01</td><td>42.1</td></tr>
  </tbody>
</table>
```

> **`<th scope="col">` and `<caption>`** are what make a table readable to a screen reader. A table without them is a wall of unlabelled numbers.

> **Never use tables for layout.** That's [[Flexbox]] and [[Grid]]. Tables are for tabular data.

## Media

```html
<video src="v.mp4" controls width="600" poster="thumb.jpg"></video>
<audio src="a.mp3" controls></audio>

<picture>
  <source srcset="wide.webp" media="(min-width: 800px)" type="image/webp">
  <img src="fallback.jpg" alt="Description">
</picture>
```

## Attributes worth knowing

```html
id="unique"            <!-- ONE element per page -->
class="a b c"          <!-- many elements, the CSS hook -->
data-device-id="42"    <!-- custom data, readable from JS -->
title="tooltip"
hidden
aria-label="Close"     <!-- accessible name when there's no visible text -->
role="alert"
```

```js
element.dataset.deviceId    // reads data-device-id
```

## Where HTML appears in this vault

| Context | Note |
|---|---|
| React / JSX | [[NextJs TypeScript]] · [[TSX]] |
| Django templates | [[Django]] |
| Streamlit (rarely — it generates HTML for you) | [[Streamlit]] |
| Scraping targets | [[B2B webscraping tool]] |

> **You will rarely write raw HTML in this stack.** React writes it, Django templates write it, Streamlit writes it. But when the layout is wrong, you're debugging HTML — so the structure and semantics matter.

## Common mistakes

| Mistake | Consequence |
|---|---|
| Missing `charset` / `viewport` | Mojibake / unusable on mobile |
| `<div>` for everything | No semantics, worse accessibility and SEO |
| Missing `alt` | Inaccessible, fails audits |
| `target="_blank"` without `rel` | Security hole |
| Multiple `<h1>`, or skipping levels | Confuses screen readers |
| Inline `style="..."` | Unmaintainable — use [[CSS]] |
| Tables for layout | Breaks on mobile, inaccessible |
| Unclosed tags | Browser guesses, and guesses differently |

## Related

[[Languages]] · [[CSS]] · [[JavaScript]] · [[Forms (HTML)]] · [[Accessibility]] · [[NextJs TypeScript]]
