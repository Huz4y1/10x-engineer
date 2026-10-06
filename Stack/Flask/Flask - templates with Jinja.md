---
tags: [flask, jinja, html, templates, guide]
---

# Flask — templates with Jinja

**Guide 2 of 8.** Returning real HTML pages instead of plain text: templates, variables, loops, if-statements, a shared layout, and your own CSS and images.

Hub: [[Flask]] · Previous: [[Flask - your first app]] · Next: [[Flask - styling with Bulma]]

---

## The problem templates solve

Returning a web page from Python like this is miserable:

```python
return "<html><body><h1>Hello " + name + "</h1><ul><li>" + ...   # unreadable, and unsafe
```

**A template is an HTML file with gaps in it.** Python fills the gaps with data, and the finished page goes to the browser.

```
templates/tasks.html            +   data from Python          =   the page the browser gets
────────────────────                ──────────────────            ─────────────────────────
<h1>Hi {{ name }}</h1>              name = "Huz"                  <h1>Hi Huz</h1>
```

Flask's template language is called **Jinja**. It's just HTML plus a few extra symbols.

---

## Where templates live

Flask looks in a folder called exactly **`templates`**, next to your app:

```
tasktracker/
├── app.py
├── templates/          <- HTML files go here - the name is not optional
│   ├── base.html
│   └── index.html
└── static/             <- CSS, images and JavaScript go here
    ├── style.css
    └── logo.png
```

> ⚠️ **It must be `templates` (plural, lowercase).** `template/` or `Templates/` gives you *TemplateNotFound*.

---

## Rendering a template

`templates/index.html`:

```html
<!doctype html>
<html>
  <body>
    <h1>Hello, {{ name }}!</h1>      <!-- {{ }} prints a value -->
    <p>You have {{ count }} tasks.</p>
  </body>
</html>
```

`app.py`:

```python
from flask import Flask, render_template

app = Flask(__name__)

@app.route("/")
def home():
    return render_template(
        "index.html",         # which file in templates/
        name="Huz",           # every keyword argument becomes a variable in the template
        count=3,
    )
```

The browser receives `<h1>Hello, Huz!</h1><p>You have 3 tasks.</p>`.

---

## The three Jinja symbols

That's all the syntax there is:

| Symbol | Does | Example |
|---|---|---|
| `{{ ... }}` | **Print** a value | `{{ user.name }}` |
| `{% ... %}` | **Do** something — a loop, an if, a block | `{% for t in tasks %}` |
| `{# ... #}` | A **comment** — never sent to the browser | `{# TODO: fix this #}` |

---

## Printing values

```html
{{ name }}                          <!-- a plain variable -->
{{ user.email }}                    <!-- an attribute of an object -->
{{ task["title"] }}                 <!-- a key in a dict -->
{{ tasks | length }}                <!-- how many items - | applies a FILTER -->
{{ name | upper }}                  <!-- HUZ -->
{{ price | round(2) }}              <!-- 9.99 -->
{{ description | truncate(50) }}    <!-- cut to 50 characters with ... -->
{{ nickname | default("Anonymous") }}   <!-- fallback if missing -->
{{ created.strftime("%d %b %Y") }}  <!-- format a date: 11 Sep 2026 -->
```

**Filters** are small functions you chain with `|`:

| Filter | Does |
|---|---|
| `upper` / `lower` / `title` / `capitalize` | Change case |
| `length` | How many items |
| `default("x")` | Use "x" if the value is missing |
| `round(2)` | Round a number |
| `truncate(50)` | Shorten long text |
| `join(", ")` | List → `"a, b, c"` |
| `replace("a", "b")` | Swap text |
| `safe` | **Turn off escaping** — see the warning below |

### Why your HTML is safe by default

```python
from flask import render_template

render_template("index.html", name="<script>alert('hacked')</script>")
```

Jinja prints that as **harmless text**, not as a script. It converts `<` into `&lt;` automatically. This is called **autoescaping**, and it's what stops someone typing JavaScript into your form and having it run in other people's browsers (an *XSS* attack).

> ⚠️ **`| safe` switches that protection off.** Only ever use it on HTML *you* wrote. **Never** on anything a user typed ([[Security in practice]]).

---

## Loops

```html
<ul>
  {% for task in tasks %}              <!-- repeat for each item -->
    <li>{{ task.title }}</li>
  {% else %}                           <!-- runs if the list was EMPTY -->
    <li>No tasks yet.</li>
  {% endfor %}                         <!-- every block must be closed -->
</ul>
```

Inside a loop you get a free `loop` variable:

| Variable | Gives you |
|---|---|
| `loop.index` | 1, 2, 3 … (counting from 1) |
| `loop.index0` | 0, 1, 2 … |
| `loop.first` / `loop.last` | True on the first / last item |
| `loop.length` | Total number of items |

```html
{% for task in tasks %}
  <p>{{ loop.index }}. {{ task.title }}{% if not loop.last %},{% endif %}</p>
{% endfor %}
```

---

## If statements

```html
{% if current_user.is_authenticated %}
  <p>Welcome back, {{ current_user.email }}</p>
{% elif guest %}
  <p>Hello, guest</p>
{% else %}
  <a href="/login">Log in</a>
{% endif %}

{% if tasks %}                 <!-- an empty list counts as False -->
  You have {{ tasks | length }} tasks
{% endif %}

<!-- a one-line inline if -->
<td class="{{ 'done' if task.done else 'todo' }}">{{ task.title }}</td>
```

> **Jinja's `if` works like Python's.** Empty lists, empty strings, `0` and `None` are all false.

---

## Template inheritance — write the layout once

Every page on your site shares the same `<head>`, navigation bar and footer. Copy-pasting that into every file means changing ten files to fix one link.

**Instead: one base template with gaps (`blocks`), and every page fills them in.**

`templates/base.html` — the skeleton:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{% block title %}Task Tracker{% endblock %}</title>   <!-- a gap, with a default -->
  <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
<body>
  <nav>
    <a href="{{ url_for('home') }}">Home</a>
    <a href="{{ url_for('about') }}">About</a>
  </nav>

  <main>
    {% block content %}{% endblock %}         <!-- each page puts its content here -->
  </main>

  <footer>Made with Flask</footer>
</body>
</html>
```

`templates/index.html` — one page:

```html
{% extends "base.html" %}                     <!-- MUST be the first line -->

{% block title %}My tasks{% endblock %}       <!-- replace the title gap -->

{% block content %}                           <!-- fill the content gap -->
  <h1>My tasks</h1>
  <ul>
    {% for task in tasks %}<li>{{ task.title }}</li>{% endfor %}
  </ul>
{% endblock %}
```

```
base.html                     index.html                  the page you get
─────────                     ──────────                  ────────────────
<nav>...</nav>                                            <nav>...</nav>
{% block content %}    ◄──    <h1>My tasks</h1>    ──►    <h1>My tasks</h1>
{% endblock %}                <ul>...</ul>                <ul>...</ul>
<footer>...</footer>                                      <footer>...</footer>
```

> **This is the most useful thing in Jinja.** Change the nav bar once in `base.html` and every page updates.

> ⚠️ **Anything in a child template *outside* a `{% block %}` is thrown away.** If some HTML isn't appearing, check it's inside a block that exists in the base.

**Adding to a block instead of replacing it:**

```html
{% block content %}
  {{ super() }}                  <!-- keep whatever the base template had here -->
  <p>...and this extra bit</p>
{% endblock %}
```

---

## Including a small piece

For a chunk you reuse in several places — a card, a footer, a flash-message box:

```html
{% include "_flash_messages.html" %}      <!-- paste another template in, right here -->
```

> **Convention:** name partial templates with a leading underscore (`_navbar.html`) so it's obvious they're pieces, not whole pages.

## Macros — functions for HTML

When you'd otherwise copy the same 8 lines of form-field HTML five times:

`templates/_macros.html`:

```html
{% macro task_card(task) %}                  <!-- define it once, with arguments -->
  <div class="card">
    <h3>{{ task.title }}</h3>
    <p>{{ 'Done' if task.done else 'To do' }}</p>
  </div>
{% endmacro %}
```

Using it:

```html
{% from "_macros.html" import task_card %}   <!-- import it at the top -->

{% for task in tasks %}
  {{ task_card(task) }}                      <!-- call it like a function -->
{% endfor %}
```

---

## Static files — your CSS, images and JavaScript

Anything in the `static/` folder is served as-is. **Always build the link with `url_for`:**

```html
<link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
<img src="{{ url_for('static', filename='images/logo.png') }}" alt="Logo">
<script src="{{ url_for('static', filename='app.js') }}"></script>
```

> **Why not just write `/static/style.css`?** It works — until you deploy under a sub-path, or put static files on a CDN. `url_for` handles both. It's a habit worth having from day one.

> ⚠️ **CSS changes not showing?** Your browser cached the old file. **Ctrl+F5** forces a full reload.

---

## Links between pages

```html
<a href="{{ url_for('home') }}">Home</a>
<a href="{{ url_for('show_task', task_id=task.id) }}">View</a>
<a href="{{ url_for('search', q='mugs') }}">Search</a>     <!-- extra args become ?q=mugs -->
```

---

## Variables available in every template

You don't pass these — Flask provides them automatically:

| Variable | What it is |
|---|---|
| `request` | The current request — `request.path`, `request.args` |
| `session` | The user's session data |
| `url_for()` | Build URLs |
| `get_flashed_messages()` | One-time messages — see [[Flask - forms and user input]] |
| `current_user` | The logged-in user, once you add [[Flask - user accounts and login\|Flask-Login]] |
| `config` | Your app's config |

```html
<!-- highlight the nav link for the page you're on -->
<a href="{{ url_for('home') }}" class="{{ 'active' if request.path == '/' }}">Home</a>
```

---

## Setting variables in a template

```html
{% set total = tasks | length %}
{% set done = tasks | selectattr("done") | list | length %}
<p>{{ done }} of {{ total }} done</p>
```

> **Keep logic out of templates.** Anything more complex than a count or a simple `if` belongs in the Python route. Templates should mostly *display* data, not calculate it.

---

## Common mistakes

| Mistake | What you'll see | Fix |
|---|---|---|
| Folder named `template` | `TemplateNotFound` | It must be `templates` |
| `{% extends %}` isn't the first line | Page renders wrong | Put it on line 1 |
| Content outside a `{% block %}` in a child | It just disappears | Wrap it in a block that exists in the base |
| Forgot `{% endfor %}` / `{% endif %}` | `TemplateSyntaxError: Unexpected end of template` | Close every block |
| A variable you forgot to pass | Renders as nothing, no error | Check `render_template(...)` passes it |
| `{{ user_input \| safe }}` | **XSS vulnerability** | Remove `\| safe` |
| Hardcoded `/static/...` links | Breaks when deployed | `url_for('static', filename=...)` |
| CSS edit not appearing | Old styles | Ctrl+F5 |

> **Missing variables fail silently.** `{{ usre.name }}` (a typo) prints nothing — no error. If something is blank, check the spelling and that the route passed it.

## Related

[[Flask]] · [[Flask - your first app]] · [[Flask - styling with Bulma]] · [[Flask reference]] · [[HTML]] · [[CSS]] · [[Security in practice]]
