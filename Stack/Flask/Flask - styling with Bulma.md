---
tags: [flask, bulma, css, html, frontend, guide]
---

# Flask — styling with Bulma

**Guide 3 of 8.** Making your Flask pages look good without writing much CSS. What Bulma is, how to add it, every component you'll use, and how to wire each one to Flask data.

Hub: [[Flask]] · Previous: [[Flask - templates with Jinja]] · Next: [[Flask - connecting to a SQL database]]

---

## What Bulma is, in simple words

Writing CSS from scratch to make buttons, forms, menus and layouts look decent takes a long time.

**Bulma is a ready-made stylesheet.** You add one line to your page, and then you style things just by giving them **class names**:

```html
<button class="button is-primary">Save</button>        <!-- a green, rounded, padded button -->
<button class="button is-danger is-outlined">Delete</button>   <!-- a red outline button -->
```

You describe what you want in plain words — `button`, `is-primary`, `is-large` — and Bulma's CSS does the rest.

| | Bulma | [[Tailwindcss\|Tailwind]] | Bootstrap | Plain [[CSS]] |
|---|---|---|---|---|
| Style by | Readable component classes (`button is-primary`) | Tiny utility classes (`px-4 py-2 bg-green-500`) | Component classes | Writing it yourself |
| JavaScript | **None at all** | None | Included | — |
| Setup | One `<link>` tag | A build step | One `<link>` + a script | — |
| Good for | **Flask/Django server-rendered pages, fast** | React / Next.js apps | Same as Bulma | Full control |

**Why Bulma suits Flask:** one line, no build step, no Node.js, and the class names read like English — so your Jinja templates stay readable.

> ⚠️ **Bulma is CSS only — it contains no JavaScript.** Anything that *moves* (the mobile menu opening, a pop-up closing, a dropdown on click) needs a few lines of your own JavaScript. They're all in this guide, in [[#The JavaScript Bulma doesn't include]].

---

## Adding it to your Flask app

Put this in the `<head>` of `templates/base.html`:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bulma@1.0.2/css/bulma.min.css">
```

That's the whole install. No pip, no npm. The file loads from a CDN (a fast public file server).

**Or download it and serve it yourself** — works offline, and doesn't depend on a third-party server:

```bash
mkdir -p static/css
curl -o static/css/bulma.min.css https://cdn.jsdelivr.net/npm/bulma@1.0.2/css/bulma.min.css
```

```html
<link rel="stylesheet" href="{{ url_for('static', filename='css/bulma.min.css') }}">
```

> **Pin the version** (`bulma@1.0.2`, not `bulma@latest`). A new major version can rename classes and silently restyle your whole site.

### ⚠️ Bulma 1.x follows the user's dark mode

Bulma 1.0 automatically switches to a **dark theme** if the visitor's computer is in dark mode. That surprises everyone the first time — your page looks different on your phone than on your laptop.

```html
<html lang="en" data-theme="light">     <!-- force light everywhere -->
<html lang="en" data-theme="dark">      <!-- force dark everywhere -->
<html lang="en">                        <!-- follow the user's setting (default) -->
```

---

## The complete base template

This is the layout every page in your app extends. Copy it as your starting point:

```html
<!doctype html>
<html lang="en" data-theme="light">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">   <!-- REQUIRED for mobile -->
  <title>{% block title %}Task Tracker{% endblock %}</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bulma@1.0.2/css/bulma.min.css">
  <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">  <!-- your tweaks, AFTER Bulma -->
</head>
<body>

  <!-- ===== NAVBAR ===== -->
  <nav class="navbar is-primary" role="navigation" aria-label="main navigation">
    <div class="navbar-brand">
      <a class="navbar-item has-text-weight-bold" href="{{ url_for('home') }}">Task Tracker</a>
      <!-- the "hamburger" icon that appears on phones - Bulma 1.x expects four spans -->
      <a role="button" class="navbar-burger" aria-label="menu" aria-expanded="false" data-target="main-nav">
        <span aria-hidden="true"></span><span aria-hidden="true"></span>
        <span aria-hidden="true"></span><span aria-hidden="true"></span>
      </a>
    </div>
    <div id="main-nav" class="navbar-menu">
      <div class="navbar-start">
        <a class="navbar-item {{ 'is-active' if request.endpoint == 'home' }}" href="{{ url_for('home') }}">Home</a>
        <a class="navbar-item {{ 'is-active' if request.endpoint == 'about' }}" href="{{ url_for('about') }}">About</a>
      </div>
      <div class="navbar-end">
        <div class="navbar-item">
          <a class="button is-light" href="#">Log in</a>
        </div>
      </div>
    </div>
  </nav>

  <!-- ===== PAGE ===== -->
  <section class="section">
    <div class="container">
      {% include "_flash.html" %}          <!-- one-time messages, see below -->
      {% block content %}{% endblock %}
    </div>
  </section>

  <!-- ===== FOOTER ===== -->
  <footer class="footer">
    <div class="content has-text-centered">
      <p>Built with Flask and Bulma</p>
    </div>
  </footer>

  <script src="{{ url_for('static', filename='bulma.js') }}"></script>  <!-- the menu/close JS -->
</body>
</html>
```

> **`request.endpoint`** is the name of the route function that served this page — it's how the nav bar knows which link to highlight with `is-active`.

---

## Layout — how you arrange a page

### Section and container — the page wrapper

```html
<section class="section">           <!-- vertical padding above and below -->
  <div class="container">           <!-- centred, with a sensible maximum width -->
    ...your page...
  </div>
</section>
```

| Class | Does |
|---|---|
| `section` | Adds space above and below |
| `container` | Centres content and caps its width |
| `container is-max-desktop` | Narrower — good for forms and reading |
| `container is-fluid` | Full width with small side margins |

### Columns — putting things side by side

**This is the part of Bulma you'll use most.** Wrap boxes in `columns`, and each child `column` shares the width equally:

```html
<div class="columns">
  <div class="column">One</div>
  <div class="column">Two</div>
  <div class="column">Three</div>          <!-- three equal columns -->
</div>
```

**Choosing widths:**

```html
<div class="columns">
  <div class="column is-one-third">Sidebar</div>   <!-- 1/3 of the width -->
  <div class="column">Main content</div>          <!-- takes whatever is left -->
</div>

<div class="columns">
  <div class="column is-8">Wide</div>             <!-- 8 of 12 parts -->
  <div class="column is-4">Narrow</div>           <!-- 4 of 12 parts -->
</div>
```

| Width classes | |
|---|---|
| `is-half` · `is-one-third` · `is-two-thirds` · `is-one-quarter` · `is-three-quarters` | Fractions |
| `is-1` … `is-12` | Out of 12 |
| `is-narrow` | Only as wide as its content |

**Useful column modifiers:**

```html
<div class="columns is-centered">               <!-- centre the columns -->
  <div class="column is-5">A login box</div>
</div>

<div class="columns is-multiline">              <!-- wrap onto new rows - for grids of cards -->
  {% for task in tasks %}
    <div class="column is-one-third">{{ task.title }}</div>
  {% endfor %}
</div>
```

> **Columns stack vertically on phones automatically.** Three side-by-side columns on a laptop become three stacked boxes on a phone — no extra work. Add `is-mobile` to `columns` if you want them side by side even on a phone.

> **`is-multiline` is what you want for a grid of items from a loop.** Without it, 12 cards try to squeeze onto one row.

### Hero — a big banner

```html
<section class="hero is-primary is-medium">
  <div class="hero-body">
    <p class="title">Welcome to Task Tracker</p>
    <p class="subtitle">Get things done</p>
  </div>
</section>
```

Sizes: `is-small` · `is-medium` · `is-large` · `is-fullheight`.

### Level — a row with things at each end

```html
<nav class="level">
  <div class="level-left">
    <h1 class="title">My tasks</h1>
  </div>
  <div class="level-right">
    <a class="button is-primary" href="{{ url_for('new_task') }}">New task</a>
  </div>
</nav>
```

> **Perfect for a page header** — the title on the left, the action button on the right.

---

## Text

```html
<h1 class="title">Big heading</h1>
<h2 class="subtitle">Smaller line underneath</h2>
<h1 class="title is-2">A specific size</h1>          <!-- is-1 (biggest) to is-6 -->

<div class="content">                                <!-- styles plain HTML inside it -->
  <p>A paragraph with <strong>bold</strong> text.</p>
  <ul><li>Bulleted lists get proper spacing</li></ul>
</div>
```

> ⚠️ **Bulma resets all default styling.** A plain `<h1>` or `<ul>` looks like normal body text with no bullets. Use `title`/`subtitle`, or wrap text in `class="content"` to get sensible typography back.

**Text helpers:**

| Class | Does |
|---|---|
| `has-text-centered` · `has-text-right` | Alignment |
| `has-text-weight-bold` · `has-text-weight-light` | Weight |
| `has-text-danger` · `has-text-success` · `has-text-grey` | Colour |
| `is-size-7` … `is-size-1` | Size (7 is smallest) |
| `is-italic` · `is-uppercase` | Style |

---

## Buttons

```html
<button class="button">Default</button>
<button class="button is-primary">Primary</button>
<button class="button is-danger is-outlined">Outlined</button>
<button class="button is-link is-light">Light</button>
<button class="button is-success is-rounded">Rounded</button>
<button class="button is-primary is-loading">Saving</button>     <!-- a spinner -->
<button class="button is-primary is-fullwidth">Full width</button>
<button class="button is-small">Small</button>
<a class="button is-primary" href="{{ url_for('new_task') }}">A link that looks like a button</a>
```

**A row of buttons with proper spacing:**

```html
<div class="buttons">
  <button class="button is-primary">Save</button>
  <button class="button">Cancel</button>
</div>
```

### The colour words — these work on almost everything

| Class | Colour | Use for |
|---|---|---|
| `is-primary` | Turquoise (default brand) | The main action |
| `is-link` | Blue | Secondary actions |
| `is-info` | Light blue | Information |
| `is-success` | Green | Success, "done" |
| `is-warning` | Yellow | Caution |
| `is-danger` | Red | Delete, errors |
| `is-dark` / `is-light` | Dark / light grey | Neutral |

The same words work on buttons, notifications, tags, navbars, heroes, messages and more — learn them once.

---

## Forms — and wiring them to Flask

Every Bulma form field uses the same three-layer structure:

```html
<div class="field">                          <!-- 1. the whole field, with spacing -->
  <label class="label">Task title</label>    <!-- 2. the label -->
  <div class="control">                      <!-- 3. wraps the actual input -->
    <input class="input" type="text" name="title" placeholder="What needs doing?">
  </div>
  <p class="help">Keep it short</p>          <!-- optional hint underneath -->
</div>
```

**Every input type:**

```html
<input class="input" type="text" name="title">
<input class="input" type="email" name="email">
<input class="input" type="password" name="password">
<input class="input" type="date" name="due">

<textarea class="textarea" name="notes" rows="4"></textarea>

<div class="select">                         <!-- a dropdown needs this wrapper div -->
  <select name="priority">
    <option value="low">Low</option>
    <option value="high">High</option>
  </select>
</div>

<label class="checkbox">
  <input type="checkbox" name="done"> Mark as done
</label>

<div class="control">
  <label class="radio"><input type="radio" name="size" value="s"> Small</label>
  <label class="radio"><input type="radio" name="size" value="l"> Large</label>
</div>
```

> ⚠️ **`<select>` needs the `<div class="select">` wrapper** or it won't be styled.

**Showing an error on a field:**

```html
<input class="input is-danger" type="email" name="email">   <!-- red border -->
<p class="help is-danger">That email isn't valid</p>         <!-- red message -->
```

**An input and button joined together:**

```html
<div class="field has-addons">
  <div class="control is-expanded">          <!-- is-expanded = take the remaining width -->
    <input class="input" name="title" placeholder="Add a task">
  </div>
  <div class="control">
    <button class="button is-primary">Add</button>
  </div>
</div>
```

**Buttons side by side at the bottom of a form:**

```html
<div class="field is-grouped">
  <div class="control"><button class="button is-primary">Save</button></div>
  <div class="control"><a class="button is-light" href="{{ url_for('home') }}">Cancel</a></div>
</div>
```

### A complete form in Flask

```html
<form method="post" action="{{ url_for('new_task') }}">
  <div class="field">
    <label class="label" for="title">Title</label>
    <div class="control">
      <input class="input" id="title" name="title" value="{{ request.form.get('title', '') }}" required>
      <!-- value= refills the box if the form comes back with an error -->
    </div>
  </div>

  <div class="field">
    <label class="label" for="priority">Priority</label>
    <div class="control">
      <div class="select">
        <select id="priority" name="priority">
          {% for p in ["low", "medium", "high"] %}
            <option value="{{ p }}" {{ 'selected' if request.form.get('priority') == p }}>{{ p | title }}</option>
          {% endfor %}
        </select>
      </div>
    </div>
  </div>

  <div class="field">
    <div class="control"><button class="button is-primary">Create</button></div>
  </div>
</form>
```

> **With Flask-WTF, one macro styles every field** — no copying these blocks for every input. See [[Flask - forms and user input]].

---

## Box and card — containers for content

**Box** — a white panel with a shadow. The simplest way to group something:

```html
<div class="box">
  <h2 class="title is-5">Log in</h2>
  ...a form...
</div>
```

**Card** — a structured panel with a header, body and footer:

```html
<div class="card">
  <header class="card-header">
    <p class="card-header-title">{{ task.title }}</p>
  </header>
  <div class="card-content">
    <div class="content">{{ task.notes }}</div>
  </div>
  <footer class="card-footer">
    <a href="{{ url_for('edit_task', task_id=task.id) }}" class="card-footer-item">Edit</a>
    <a href="{{ url_for('show_task', task_id=task.id) }}" class="card-footer-item">View</a>
  </footer>
</div>
```

**A grid of cards from your data:**

```html
<div class="columns is-multiline">
  {% for task in tasks %}
    <div class="column is-one-third">
      <div class="card">
        <div class="card-content">
          <p class="title is-5">{{ task.title }}</p>
          <span class="tag {{ 'is-success' if task.done else 'is-warning' }}">
            {{ 'Done' if task.done else 'To do' }}
          </span>
        </div>
      </div>
    </div>
  {% else %}
    <div class="column"><p class="has-text-grey">No tasks yet.</p></div>
  {% endfor %}
</div>
```

---

## Tables — showing rows from the database

```html
<table class="table is-fullwidth is-striped is-hoverable">
  <thead>
    <tr><th>Title</th><th>Status</th><th>Created</th><th></th></tr>
  </thead>
  <tbody>
    {% for task in tasks %}
      <tr>
        <td>{{ task.title }}</td>
        <td><span class="tag {{ 'is-success' if task.done else 'is-light' }}">{{ 'Done' if task.done else 'Open' }}</span></td>
        <td>{{ task.created_at.strftime('%d %b') }}</td>
        <td class="has-text-right">
          <a class="button is-small" href="{{ url_for('edit_task', task_id=task.id) }}">Edit</a>
        </td>
      </tr>
    {% endfor %}
  </tbody>
</table>
```

| Class | Does |
|---|---|
| `is-fullwidth` | Stretch to the container width |
| `is-striped` | Alternate row shading |
| `is-hoverable` | Highlight the row under the mouse |
| `is-bordered` | Borders on every cell |
| `is-narrow` | Tighter padding |

> ⚠️ **Wide tables break phone layouts.** Wrap them so they scroll sideways: `<div class="table-container"><table class="table">...</table></div>`.

---

## Tags — small labels

```html
<span class="tag">Default</span>
<span class="tag is-success">Done</span>
<span class="tag is-danger is-light">Overdue</span>
<span class="tag is-rounded">Rounded</span>

<div class="tags">                       <!-- a group, with spacing -->
  {% for label in task.labels %}<span class="tag is-info">{{ label }}</span>{% endfor %}
</div>
```

---

## Notifications — Flask's flash messages

Flask's `flash()` shows a one-time message on the next page ("Task saved!"). Bulma's `notification` is exactly the right box for it.

In Python, pass a **Bulma colour class as the category**:

```python
from flask import flash

flash("Task saved!", "is-success")          # the second argument becomes the CSS class
flash("Something went wrong", "is-danger")
flash("Please log in first", "is-warning")
```

`templates/_flash.html`:

```html
{% with messages = get_flashed_messages(with_categories=true) %}
  {% for category, message in messages %}
    <div class="notification {{ category }} is-light">
      <button class="delete" aria-label="close"></button>   <!-- the little x -->
      {{ message }}
    </div>
  {% endfor %}
{% endwith %}
```

> **The trick is using the Bulma class *as* the flash category.** Then `flash("...", "is-danger")` automatically produces a red box with no extra mapping code.

**Message** — a notification with a header:

```html
<article class="message is-warning">
  <div class="message-header"><p>Heads up</p></div>
  <div class="message-body">You have 3 overdue tasks.</div>
</article>
```

---

## Modal — a pop-up

```html
<button class="button is-danger js-modal-trigger" data-target="confirm-delete">Delete</button>

<div id="confirm-delete" class="modal">
  <div class="modal-background"></div>             <!-- the dark overlay -->
  <div class="modal-card">
    <header class="modal-card-head">
      <p class="modal-card-title">Delete this task?</p>
      <button class="delete" aria-label="close"></button>
    </header>
    <section class="modal-card-body">This can't be undone.</section>
    <footer class="modal-card-foot">
      <form method="post" action="{{ url_for('delete_task', task_id=task.id) }}">
        <button class="button is-danger">Delete</button>
      </form>
      <button class="button">Cancel</button>
    </footer>
  </div>
</div>
```

A modal is hidden until it gets the class `is-active` — which needs JavaScript, below.

> **For a simple "are you sure?" you don't need a modal at all:** `<form method="post" onsubmit="return confirm('Delete this task?')">`. The browser's own dialog, zero Bulma, one attribute.

---

## Tabs, pagination and other components

```html
<!-- tabs - the links are just normal Flask routes -->
<div class="tabs">
  <ul>
    <li class="{{ 'is-active' if filter == 'all' }}"><a href="{{ url_for('home', filter='all') }}">All</a></li>
    <li class="{{ 'is-active' if filter == 'open' }}"><a href="{{ url_for('home', filter='open') }}">Open</a></li>
    <li class="{{ 'is-active' if filter == 'done' }}"><a href="{{ url_for('home', filter='done') }}">Done</a></li>
  </ul>
</div>
```

> **Make tabs normal links to Flask routes** (`?filter=open`) rather than JavaScript that hides and shows content. It needs no JS, the back button works, and the URL can be bookmarked.

```html
<!-- pagination, driven by Flask-SQLAlchemy's paginate() -->
<nav class="pagination is-centered" role="navigation">
  <a class="pagination-previous" {% if not page.has_prev %}disabled{% endif %}
     href="{{ url_for('home', page=page.prev_num) }}">Previous</a>
  <a class="pagination-next" {% if not page.has_next %}disabled{% endif %}
     href="{{ url_for('home', page=page.next_num) }}">Next</a>
  <ul class="pagination-list">
    {% for n in page.iter_pages() %}
      {% if n %}
        <li><a class="pagination-link {{ 'is-current' if n == page.page }}" href="{{ url_for('home', page=n) }}">{{ n }}</a></li>
      {% else %}
        <li><span class="pagination-ellipsis">&hellip;</span></li>
      {% endif %}
    {% endfor %}
  </ul>
</nav>
```

See [[Flask - connecting to a SQL database]] for `paginate()`.

```html
<progress class="progress is-success" value="{{ done }}" max="{{ total }}"></progress>
```

---

## Spacing helpers

Instead of writing CSS for margins and padding:

| Class | Means |
|---|---|
| `m-0` … `m-6` | Margin on all sides (0 = none, 6 = most) |
| `mt-4` · `mb-4` · `ml-4` · `mr-4` | Margin top / bottom / left / right |
| `mx-4` · `my-4` | Margin left+right / top+bottom |
| `p-4` · `pt-2` · `px-3` … | The same pattern for **padding** |

```html
<h1 class="title mb-2">Tight to the next line</h1>
<div class="box p-6">Lots of inner space</div>
```

**Show and hide by screen size:**

```html
<p class="is-hidden-mobile">Only on bigger screens</p>
<p class="is-hidden-tablet">Only on phones</p>
```

---

## The JavaScript Bulma doesn't include

Save this as `static/bulma.js` and load it at the bottom of `base.html` (as in the base template above):

```javascript
document.addEventListener("DOMContentLoaded", () => {

  // 1. Mobile navbar - open and close the menu when the hamburger is clicked
  document.querySelectorAll(".navbar-burger").forEach(burger => {
    burger.addEventListener("click", () => {
      const menu = document.getElementById(burger.dataset.target);
      burger.classList.toggle("is-active");
      menu.classList.toggle("is-active");
      burger.setAttribute("aria-expanded", burger.classList.contains("is-active"));
    });
  });

  // 2. Notifications - the little x removes the message
  document.querySelectorAll(".notification .delete").forEach(btn => {
    btn.addEventListener("click", () => btn.parentNode.remove());
  });

  // 3. Modals - open from any .js-modal-trigger, close on background / x / Cancel / Escape
  const closeAll = () => document.querySelectorAll(".modal.is-active")
                                  .forEach(m => m.classList.remove("is-active"));
  document.querySelectorAll(".js-modal-trigger").forEach(trigger => {
    trigger.addEventListener("click", () =>
      document.getElementById(trigger.dataset.target).classList.add("is-active"));
  });
  document.querySelectorAll(".modal-background, .modal .delete, .modal-card-foot .button:not(.is-danger)")
    .forEach(el => el.addEventListener("click", closeAll));
  document.addEventListener("keydown", e => { if (e.key === "Escape") closeAll(); });

  // 4. Dropdowns - toggle on click
  document.querySelectorAll(".dropdown:not(.is-hoverable) .dropdown-trigger").forEach(t => {
    t.addEventListener("click", e => {
      e.stopPropagation();
      t.closest(".dropdown").classList.toggle("is-active");
    });
  });
  document.addEventListener("click", () =>
    document.querySelectorAll(".dropdown.is-active").forEach(d => d.classList.remove("is-active")));
});
```

> **Without #1, your navigation disappears on phones.** The menu collapses behind the hamburger icon on small screens, and clicking it does nothing until this script is loaded. Test on a narrow window before you call it finished.

See [[JavaScript]] for how these event listeners work.

---

## Your own tweaks on top

Put your own CSS in `static/style.css` and link it **after** Bulma, so yours wins:

```css
.task-done { text-decoration: line-through; opacity: 0.6; }
.navbar-brand .navbar-item { font-size: 1.25rem; }
```

**Changing Bulma's colours** — Bulma 1.x is built on CSS variables, so you can re-colour it without rebuilding anything:

```css
:root {
  --bulma-primary-h: 262deg;      /* hue - change the primary colour to purple */
  --bulma-primary-s: 60%;
  --bulma-primary-l: 50%;
}
```

> **Change the variables, not the classes.** Overriding `.button.is-primary` directly means fighting Bulma's own rules for hover, focus and disabled states. The variables change all of them consistently.

**Icons** — Bulma has no icon set of its own. Add Font Awesome and use Bulma's `icon` wrapper:

```html
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.2/css/all.min.css">

<button class="button is-primary">
  <span class="icon"><i class="fas fa-plus"></i></span>
  <span>Add task</span>
</button>
```

---

## Common mistakes

| Mistake | What you'll see | Fix |
|---|---|---|
| No `<meta name="viewport">` | Tiny zoomed-out page on phones | Add it to `<head>` |
| Menu vanishes on phones | Hamburger does nothing | Load the navbar JavaScript |
| `<select>` looks unstyled | Plain browser dropdown | Wrap in `<div class="select">` |
| Plain `<h1>`/`<ul>` look like body text | No heading sizes, no bullets | Use `title`, or wrap in `class="content"` |
| `column` without a `columns` parent | Nothing lines up | Always nest `column` inside `columns` |
| A grid of 12 cards in one squashed row | Tiny cards | Add `is-multiline` to `columns` |
| Page is dark on some devices | Bulma 1.x follows dark mode | `data-theme="light"` on `<html>` |
| Your CSS doesn't apply | Bulma overrides it | Link `style.css` *after* Bulma |
| Wide table breaks the phone layout | Page scrolls sideways | Wrap in `div.table-container` |
| Styles unchanged after an edit | Browser cache | Ctrl+F5 |

## Related

[[Flask]] · [[Flask - templates with Jinja]] · [[Flask - forms and user input]] · [[Flask reference]] · [[CSS]] · [[HTML]] · [[CSS libraries]] · [[JavaScript]] · [[Tailwindcss]] · [[Accessibility]]
