---
tags: [html, forms, web]
---

# Forms (HTML)

Language: [[HTML]] · Validation on the server: [[FastAPI reference]]

---

```html
<form action="/submit" method="post">
  <label for="email">Email</label>
  <input type="email" id="email" name="email" required>

  <label for="qty">Quantity</label>
  <input type="number" id="qty" name="qty" min="1" max="100" value="1">

  <button type="submit">Send</button>
</form>
```

> **Every input needs a `<label for="...">` matching its `id`.** Clicking the label focuses the input, and screen readers announce it. An input without a label is unusable for some people.

> **`name` is what gets submitted**, `id` is what the label points at. Forgetting `name` means the field silently isn't sent.

## Input types

```html
<input type="text">
<input type="email">          <!-- validates format, mobile shows @ keyboard -->
<input type="password">
<input type="number" min="0" max="100" step="1">
<input type="tel">            <!-- numeric keypad on mobile -->
<input type="url">
<input type="date"> <input type="time"> <input type="datetime-local">
<input type="checkbox" checked>
<input type="radio" name="group" value="a">
<input type="file" accept=".csv,.parquet" multiple>
<input type="range" min="0" max="100">
<input type="search"> <input type="color"> <input type="hidden">
```

> **The right `type` gives you free validation and the right mobile keyboard.** `type="email"` on a phone shows the `@` key; `type="tel"` shows a numeric pad. Small change, real usability difference.

## Other controls

```html
<textarea name="notes" rows="4" cols="50"></textarea>

<select name="category">
  <optgroup label="Home">
    <option value="kitchen" selected>Kitchen</option>
    <option value="lighting">Lighting</option>
  </optgroup>
</select>

<input list="options" name="c">
<datalist id="options"><option value="Kitchen"></option></datalist>
```

## Built-in validation

```html
<input required>
<input minlength="3" maxlength="50">
<input min="1" max="100">
<input pattern="[A-Z]{2}\d{4}" title="Two letters then four digits">
```

```css
input:invalid { border-color: red; }
input:valid   { border-color: green; }
```

> ⚠️ **Client-side validation is a convenience, never a security control.** Anyone can bypass it with curl. **Always validate again on the server** — that's what [[Pydantic]] and [[FastAPI reference]] are for.

## Grouping and accessibility

```html
<fieldset>
  <legend>Delivery address</legend>
  <label for="city">City</label>
  <input id="city" name="city">
</fieldset>
```

```html
<label for="qty">Quantity</label>
<input id="qty" name="qty" aria-describedby="qty-help">
<small id="qty-help">Between 1 and 100</small>
```

## Submitting

```html
<form action="/upload" method="post" enctype="multipart/form-data">
  <input type="file" name="file">
</form>
```

| Method | Use |
|---|---|
| `GET` | Search/filter — values appear in the URL, shareable, **safe to repeat** |
| `POST` | Anything that changes data |

> **`enctype="multipart/form-data"` is required for file uploads.** Without it only the filename is sent, not the file — a genuinely confusing bug.

## With JavaScript

```js
const form = document.querySelector("form");
form.addEventListener("submit", async (e) => {
  e.preventDefault();                        // stop the page reloading
  const data = Object.fromEntries(new FormData(form));
  const r = await fetch("/api/submit", {
    method: "POST",
    headers: {"Content-Type": "application/json"},
    body: JSON.stringify(data),
  });
});
```

## Common mistakes

| Mistake | Consequence |
|---|---|
| No `<label for>` | Inaccessible |
| Missing `name` | Field not submitted |
| Trusting client validation | Security hole |
| Missing `enctype` for files | File never arrives |
| No `e.preventDefault()` | Page reloads mid-submit |
| `GET` for something that changes data | Repeated by caches and browsers |

## Related

[[HTML]] · [[Accessibility]] · [[CSS]] · [[FastAPI reference]] · [[Pydantic]] · [[26 — SECURITY]]
