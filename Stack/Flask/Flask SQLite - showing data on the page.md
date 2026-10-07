---
tags: [flask, sqlite, jinja, html, python, templates, cookbook]
---

# Flask SQLite - showing data on the page

**Twenty worked examples of getting rows out of the database and onto a web page.** Every one is a real route, a real template, and the HTML it actually produced — I ran all of them before writing this down.

Stack: [[Flask SQLite stack]] · Projects: [[Flask SQLite - notes app]] · [[Flask SQLite - book tracker]] · Hub: [[Flask]]

> ✅ **Run on Flask 3.1.3, Python 3.14, SQLite 3.50.4.** Every "what the page gets" block below is copied from the real response, not written by hand.

---

## The one rule

```
the view        fetches the data and gets it into the right shape
the template    only displays what it was handed
```

**Templates never query the database.** They loop, they format, they decide what to show — and that's all. The moment a template starts fetching things, you can no longer tell what a page costs, and the N+1 problem in example 11 is unavoidable.

Everything below is a variation on four lines:

```python
@app.route("/products")                                   # 1. a URL
def many_rows():
    products = rows("SELECT * FROM products ORDER BY name")  # 2. ask SQLite
    return render_template("products.html", products=products)  # 3. hand them over
#                                           ^^^^^^^^^^^^^^^^^^ 4. the template loops
```

> **In this note the templates are strings next to their routes**, so each example sits in one place while you read it. In a real app every one of them is its own file in `templates/`, rendered with `render_template("name.html", ...)` instead of `render_template_string(...)`.

---

## The setup these examples share

A small shop: customers, products, tags, orders and order items. It deliberately contains the awkward things real data has — a column that's sometimes `NULL`, money as whole pence, dates as text, and `0`/`1` instead of true/false.

```sql
-- A small shop. Deliberately includes the awkward things real databases have:
-- a column that is sometimes NULL, money as whole pence, dates as text, and a
-- 0/1 column standing in for true/false.

DROP TABLE IF EXISTS order_items;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS product_tags;
DROP TABLE IF EXISTS tags;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS customers;

CREATE TABLE customers (
    id        INTEGER PRIMARY KEY AUTOINCREMENT,
    name      TEXT    NOT NULL,
    email     TEXT,                                  -- nullable on purpose: some have none
    joined_at TEXT    NOT NULL,                      -- 'YYYY-MM-DD HH:MM:SS'
    is_active INTEGER NOT NULL DEFAULT 1             -- SQLite has no boolean: 0 / 1
);

CREATE TABLE products (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    name        TEXT    NOT NULL,
    category    TEXT    NOT NULL,
    price_pence INTEGER NOT NULL,                    -- money as WHOLE PENCE, never a float
    stock       INTEGER NOT NULL DEFAULT 0,
    notes       TEXT                                 -- nullable
);

CREATE TABLE tags (
    id   INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE
);

-- A "join table": it exists only to connect products and tags, many to many.
CREATE TABLE product_tags (
    product_id INTEGER NOT NULL REFERENCES products (id) ON DELETE CASCADE,
    tag_id     INTEGER NOT NULL REFERENCES tags (id)     ON DELETE CASCADE,
    PRIMARY KEY (product_id, tag_id)
);

CREATE TABLE orders (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    customer_id INTEGER NOT NULL REFERENCES customers (id) ON DELETE CASCADE,
    placed_at   TEXT    NOT NULL,
    status      TEXT    NOT NULL DEFAULT 'new'        -- new / paid / shipped
);

CREATE TABLE order_items (
    id               INTEGER PRIMARY KEY AUTOINCREMENT,
    order_id         INTEGER NOT NULL REFERENCES orders (id)   ON DELETE CASCADE,
    product_id       INTEGER NOT NULL REFERENCES products (id) ON DELETE CASCADE,
    quantity         INTEGER NOT NULL,
    -- the price AT THE TIME of the order: if the product's price changes later,
    -- old orders must not change. Copying it here is the standard fix.
    unit_price_pence INTEGER NOT NULL
);

CREATE INDEX idx_orders_customer ON orders (customer_id);
CREATE INDEX idx_items_order ON order_items (order_id);
```

### The plumbing, and three helpers

```python
"""Twenty worked examples of getting rows out of SQLite and onto an HTML page.

Each example is a route plus the template it renders. The templates are written
here as strings only so each example stays in one place while you read it - in a
real app every one of them is a file in templates/.
"""

import os
import sqlite3
from datetime import datetime
from itertools import groupby

from flask import Flask, abort, g, render_template_string, request

# markupsafe comes with Flask. Markup("...") means "this string really is HTML";
# escape("...") means "turn any HTML in this into visible text".
from markupsafe import Markup, escape

app = Flask(__name__)
app.config["DATABASE"] = os.path.join(app.root_path, "shop.db")


# ---------------------------------------------------------------------------
# the usual plumbing (same as every other note in this stack)
# ---------------------------------------------------------------------------
def get_db():
    if "db" not in g:
        g.db = sqlite3.connect(app.config["DATABASE"])
        g.db.row_factory = sqlite3.Row
        g.db.execute("PRAGMA foreign_keys = ON")
    return g.db


@app.teardown_appcontext
def close_db(exception=None):
    db = g.pop("db", None)
    if db is not None:
        db.close()


def rows(sql, params=()):
    """Every matching row, as a list."""
    return get_db().execute(sql, params).fetchall()


def row(sql, params=()):
    """The first matching row, or None if there wasn't one."""
    return get_db().execute(sql, params).fetchone()


def value(sql, params=()):
    """One single number or string out of a one-row, one-column query."""
    result = row(sql, params)
    return None if result is None else result[0]
```

**Those three helpers are what keep the examples short.** Pick by the shape of the answer you want:

| Helper | Use for | Gives you | If nothing matches |
|---|---|---|---|
| `rows(sql)` | a list — a table, a loop | list of row objects | `[]`, which is falsy |
| `row(sql)` | one thing — a detail page | one row object | `None` |
| `value(sql)` | a single number — a count, a total | an `int` or `str` | `None` |

### What a row actually is

`sqlite3.Row` is not a dict and not a tuple. It's a thin wrapper around one result row:

```python
r = row("SELECT id, name, price_pence FROM products WHERE id = 2")

r["name"]           # 'Desk lamp'   - by column name. Use this.
r[1]                # 'Desk lamp'   - by position. Fragile: breaks if you add a column
r.keys()            # ['id', 'name', 'price_pence']
dict(r)             # {'id': 2, 'name': 'Desk lamp', 'price_pence': 3450}  - for JSON
len(r)              # 3
r["nope"]           # IndexError - a column that was not SELECTed
```

> ⚠️ **Use `row["x"]`, not `row.x` — and the reason is not the one you'd guess.** In a *template* `{{ product.name }}` does work, because Jinja tries attribute access and then falls back to `product["name"]`. It breaks on columns whose name collides with one of Row's own methods: a column called `keys` makes `{{ row.keys }}` print `<built-in method keys of sqlite3.Row object at 0x...>`, while `{{ row["keys"] }}` gives the value. In *Python* code `row.name` raises `AttributeError` outright. One form always works, so use it.

> ⚠️ **You can only read columns you asked for.** `SELECT id, name` then `row["price_pence"]` is an error — and `SELECT *` is the usual reason a working page breaks after someone renames a column.

---

## Formatting in one place: custom filters

Three filters used throughout. Each one is written once and used everywhere, which beats repeating the same maths in fifteen templates.

```python
@app.template_filter("money")
def money(pence):
    """3450 -> '£34.50'. Writing it once here beats repeating the maths.

    The format spec {:,.2f} means: thousands separators, exactly 2 decimals.
    """
    if pence is None:
        return "—"
    return f"£{pence / 100:,.2f}"


@app.template_filter("nice_date")
def nice_date(text, fmt="%-d %b %Y"):
    """'2026-10-03 16:20:00' -> '3 Oct 2026'.

    SQLite hands dates over as plain text, so parse it into a real datetime
    first - then any format you like is one call away.
    """
    if not text:
        return "—"
    stamp = datetime.strptime(text, "%Y-%m-%d %H:%M:%S")
    try:
        return stamp.strftime(fmt)
    except ValueError:
        # Windows' strftime has no %-d (strip-the-leading-zero) flag
        return stamp.strftime(fmt.replace("%-d", "%d")).lstrip("0")


@app.template_filter("highlight")
def highlight(text, term):
    """Wrap each occurrence of `term` in <mark>, ignoring capitals.

    Why this is a Python function and not a one-liner in the template: the
    obvious version,

        {{ name|e|replace(term, "<mark>" ~ term ~ "</mark>")|safe }}

    does not work. |e produces a Markup object, and Markup.replace() escapes
    whatever you pass it, so the page shows the literal text &lt;mark&gt;.
    Dropping the |e instead would "work" and hand any visitor a way to inject
    their own HTML. Doing it here, piece by piece, is both correct and safe:
    every piece of the product name is escaped, and only the <mark> tags we
    wrote ourselves are real HTML.
    """
    text = "" if text is None else str(text)
    if not term:
        return escape(text)

    pieces, haystack, needle, at = [], text.lower(), term.lower(), 0
    while (found := haystack.find(needle, at)) != -1:
        pieces.append(escape(text[at:found]))                              # before the match
        pieces.append(Markup("<mark>{}</mark>").format(text[found:found + len(needle)]))
        at = found + len(needle)
    pieces.append(escape(text[at:]))                                       # whatever is left
    return Markup("").join(pieces)
```

| Filter | In the template | Out |
|---|---|---|
| `money` | `{{ 3450\|money }}` | `£34.50` |
| `nice_date` | `{{ "2026-10-03 16:20:00"\|nice_date }}` | `3 Oct 2026` |
| `nice_date` with a format | `{{ o\|nice_date("%d/%m/%Y %H:%M") }}` | `03/10/2026 16:20` |
| `highlight` | `{{ "Desk lamp"\|highlight("lamp") }}` | `Desk <mark>lamp</mark>` |

> ⚠️ **Never store money as a float.** `0.1 + 0.2` is `0.30000000000000004` in every language that uses binary floats, and those errors accumulate into real missing pennies. Store **whole pence as an integer**, and divide by 100 only when displaying. Same for the database column: `INTEGER`, not `REAL`.

> **The `%-d` dance in `nice_date`** is a genuine Windows difference: `%-d` ("day without the leading zero") works on macOS and Linux and raises `ValueError` on Windows, so the filter falls back and strips the zero itself.

---

## The examples


### 1. One single number on the page

**Start here, because it's the whole idea in miniature.** A `COUNT` or a `SUM` comes back as one row with one column, so you pull the bare number out and hand that to the template.

**The route:**

```python
@app.route("/count")
def count_only():
    # COUNT and SUM come back as a single row with a single column. Asking for
    # [0] gets the bare number, so the template gets an int, not a row object.
    product_count = value("SELECT COUNT(*) FROM products")
    stock_value = value("SELECT SUM(price_pence * stock) FROM products")
    return render_template_string(T_COUNT, product_count=product_count, stock_value=stock_value)
```

**The template:**

```html
<p>We sell <strong>{{ product_count }}</strong> products,
   worth <strong>{{ stock_value|money }}</strong> in stock.</p>
```

**What the page gets:**

```html
<p>We sell <strong>8</strong> products,
   worth <strong>£3,707.93</strong> in stock.</p>
```

**Why `value()` ends in `[0]`.** `SELECT COUNT(*)` doesn't return `5` — it returns one row *containing* 5. Forget the `[0]` and the page prints something like `<sqlite3.Row object at 0x...>`, which is the most common "why is my page showing gibberish" moment.

> ⚠️ **`SUM` of no rows is `NULL`, not 0.** An empty table gives `None`, and `{{ None|money }}` would print "None" — which is why the `money` filter has a `None` branch. Either guard the filter, as here, or ask SQLite: `SELECT COALESCE(SUM(x), 0)`.

---

### 2. One row - a detail page

**A page about one thing.** `row()` gives you a single row — or `None`, which is the case everybody forgets.

**The route:**

```python
@app.route("/product/<int:product_id>")
def one_row(product_id):
    product = row("SELECT * FROM products WHERE id = ?", (product_id,))

    # fetchone() returns None when nothing matched, and leaving that unchecked is
    # worse than a crash: Jinja prints NOTHING for each missing value, so
    # /product/999 would return a blank page with a cheerful 200 OK. It only
    # raises (UndefinedError) if some filter then tries maths on a blank.
    if product is None:
        abort(404)

    return render_template_string(T_ONE, product=product)
```

**The template:**

```html
<h1>{{ product["name"] }}</h1>
<p>{{ product["category"] }} · {{ product["price_pence"]|money }}</p>
<p>{{ product["stock"] }} in stock</p>
```

**What the page gets:**

```html
<h1>Desk lamp</h1>
<p>Lighting · £34.50</p>
<p>27 in stock</p>
```

**The `if product is None` check is not optional, and what happens without it is worse than a crash.** Jinja prints *nothing* for a value it can't resolve, so `/product/999` would return a blank page with a cheerful `200 OK` — no error anywhere, nothing in the log. I checked exactly what Jinja does with a `None` row:

```python
render_template_string('[{{ p["name"] }}]', p=None)          # '[]'        - silently empty
render_template_string('[{{ p["name"]|upper }}]', p=None)    # '[]'        - still silent
render_template_string('[{{ p["price"]|money }}]', p=None)   # UndefinedError
```

So it stays quiet until some filter does real work on a blank — in that last case the `money` filter trying to divide. A page that is *sometimes* blank and *sometimes* a 500 is far harder to track down than an honest 404.

| URL | What happens | Why |
|---|---|---|
| `/product/2` | the page, 200 | the row exists |
| `/product/999` | a clean 404 | `abort(404)` on `None` |
| `/product/abc` | a 404, no code runs | `<int:product_id>` only matches digits |

All three are checked in the test run.

---

### 3. Many rows - a table

**The bread and butter: a list of rows in a table.** `{% for %}` repeats the markup inside it once per row.

**The route:**

```python
@app.route("/products")
def many_rows():
    products = rows("SELECT * FROM products ORDER BY name")
    return render_template_string(T_TABLE, products=products)
```

**The template:**

```html
<table>
  <thead><tr><th>Product</th><th>Category</th><th>Price</th></tr></thead>
  <tbody>
    {% for p in products %}
      <tr>
        <td>{{ p["name"] }}</td>
        <td>{{ p["category"] }}</td>
        <td>{{ p["price_pence"]|money }}</td>
      </tr>
    {% endfor %}
  </tbody>
</table>
```

**What the page gets:**

```html
<table>
  <thead><tr><th>Product</th><th>Category</th><th>Price</th></tr></thead>
  <tbody>

      <tr>
        <td>Desk lamp</td>
        <td>Lighting</td>
        <td>£34.50</td>
      </tr>

      <tr>
        <td>Filing cabinet</td>
        <td>Furniture</td>
        <td>£89.99</td>
      </tr>

      <tr>
        <td>Floor lamp</td>
        <td>Lighting</td>
        <td>£79.99</td>
      </tr>

      <tr>
        <td>Keyboard tray</td>
        <td>Desk gear</td>
        <td>£18.99</td>
      </tr>

      <tr>
        <td>LED bulb</td>
        <td>Lighting</td>
        <td>£5.99</td>
      </tr>

      <tr>
        <td>Monitor stand</td>
        <td>Desk gear</td>
        <td>£22.50</td>
      </tr>

      <tr>
        <td>Oak desk</td>
        <td>Furniture</td>
        <td>£249.99</td>
      </tr>

      <tr>
        <td>Office chair</td>
        <td>Furniture</td>
        <td>£159.99</td>
      </tr>

  </tbody>
</table>
```

**The structure matters here.** The `<tr>` lives *inside* the loop and the `<table>`, `<thead>` and `<tbody>` live *outside* it — put the table tags inside the loop by accident and you get one table per row, which looks like broken spacing.

`ORDER BY` belongs in the SQL, not in Python. SQLite sorts far faster than a Python loop can, and it's one phrase instead of several lines.

---

### 4. No rows at all - the empty state

**An empty result is normal, not an error** — a new user, a search with no matches, a filtered view. Without a message the visitor sees a blank page and assumes the site is broken.

**The route:**

```python
@app.route("/category/<name>")
def empty_state(name):
    products = rows("SELECT * FROM products WHERE category = ?", (name,))
    return render_template_string(T_EMPTY, products=products)
```

**The template:**

```html
{% if products %}
  <ul>{% for p in products %}<li>{{ p["name"] }}</li>{% endfor %}</ul>
{% else %}
  <p class="empty">No products in that category yet.</p>
{% endif %}

{# Jinja's own shorthand for the same thing: #}
<ul>
  {% for p in products %}
    <li>{{ p["name"] }}</li>
  {% else %}
    <li class="empty">Nothing here.</li>
  {% endfor %}
</ul>
```

**A category with rows:**

```html
<ul><li>Desk lamp</li><li>LED bulb</li><li>Floor lamp</li></ul>

<ul>

    <li>Desk lamp</li>

    <li>LED bulb</li>

    <li>Floor lamp</li>

</ul>
```

**A category with none:**

```html
<p class="empty">No products in that category yet.</p>

<ul>

    <li class="empty">Nothing here.</li>

</ul>
```

**Two ways to write it, both above.** The second is Jinja's `{% for %} ... {% else %}`, where the `else` runs when there was nothing to loop over. (Confusingly unlike Python's `for/else`, which runs the `else` when the loop *didn't* break.)

**An empty list is falsy**, so `{% if products %}` is all you need — no `|length > 0`.

---

### 5. Columns that are sometimes NULL

**`NULL` means "no value" and it arrives in Python as `None`.** Printed raw, your page says `None` to the visitor. Three ways to handle it, and they are not interchangeable.

**The route:**

```python
@app.route("/customers")
def nulls():
    customers = rows("SELECT * FROM customers ORDER BY id")
    return render_template_string(T_NULLS, customers=customers)
```

**The template:**

```html
<table>
  <tbody>
    {% for c in customers %}
      <tr>
        <td>{{ c["name"] }}</td>

        {# `or` swaps in a fallback for NULL *and* for an empty string, because
           both are falsy. This is the one you want almost every time. #}
        <td>{{ c["email"] or "no email" }}</td>

        {# `is none` asks specifically about NULL, and treats "" as a real value.
           Use it when empty and missing mean different things. #}
        <td>{% if c["email"] is none %}never given{% else %}{{ c["email"] }}{% endif %}</td>

        {# The |default filter only fires on UNDEFINED (a name the view never
           passed), not on None - which is why it looks broken on NULL columns. #}
        <td>{{ c["email"]|default("unused", true) }}</td>
      </tr>
    {% endfor %}
  </tbody>
</table>
```

**What the page gets:**

```html
<table>
  <tbody>

      <tr>
        <td>Ada Lovelace</td>

        <td>ada@example.com</td>

        <td>ada@example.com</td>

        <td>ada@example.com</td>
      </tr>

      <tr>
        <td>Grace Hopper</td>

        <td>no email</td>

        <td>never given</td>

        <td>unused</td>
      </tr>

      <tr>
        <td>Alan Turing</td>

        <td>alan@example.com</td>

        <td>alan@example.com</td>

        <td>alan@example.com</td>
      </tr>

      <tr>
        <td>Katherine J.</td>

        <td>kj@example.com</td>

        <td>kj@example.com</td>

        <td>kj@example.com</td>
      </tr>

  </tbody>
</table>
```

| You write | Fires when the value is | Good for |
|---|---|---|
| `{{ x or "fallback" }}` | `None` **or** `""` **or** `0` | almost always what you want for text |
| `{% if x is none %}` | only `None` | when "blank" and "missing" differ |
| `{{ x\|default("f", true) }}` | undefined, or falsy with `true` | values the view might not pass at all |

> ⚠️ **`or` also swallows `0` and `""`.** For a number, `{{ stock or "unknown" }}` shows "unknown" when the stock really is zero. Use `{% if stock is none %}` for numbers.

> ⚠️ **`|default` does not fire on `None`.** It's for *undefined* — a name the view never passed. On a `NULL` column it does nothing, which is a very common surprise. The `true` second argument widens it to any falsy value.

---

### 6. Money

**Money is stored as whole pence and formatted on the way out.** Three places you could do the formatting, and which to pick.

**The route:**

```python
@app.route("/prices")
def money_page():
    products = rows(
        """
        SELECT name, price_pence,
               -- printf is SQLite's formatter. /100.0 (not /100) forces decimals:
               -- integer division would turn 3450 into 34 and lose the pence.
               printf('%.2f', price_pence / 100.0) AS price_text
        FROM products ORDER BY price_pence DESC LIMIT 4
        """
    )
    return render_template_string(T_MONEY, products=products)
```

**The template:**

```html
<ul>
  {% for p in products %}
    <li>
      {{ p["name"] }}:
      {{ p["price_pence"]|money }}                      {# the filter, defined once #}
      · raw: {{ p["price_pence"] }}p
      · from SQL: £{{ p["price_text"] }}                {# SQLite did the maths #}
      · 20% VAT: {{ (p["price_pence"] * 1.2)|round|int|money }}
    </li>
  {% endfor %}
</ul>
```

**What the page gets:**

```html
<ul>

    <li>
      Oak desk:
      £249.99
      · raw: 24999p
      · from SQL: £249.99
      · 20% VAT: £299.99
    </li>

    <li>
      Office chair:
      £159.99
      · raw: 15999p
      · from SQL: £159.99
      · 20% VAT: £191.99
    </li>

    <li>
      Filing cabinet:
      £89.99
      · raw: 8999p
      · from SQL: £89.99
      · 20% VAT: £107.99
    </li>

    <li>
      Floor lamp:
      £79.99
      · raw: 7999p
      · from SQL: £79.99
      · 20% VAT: £95.99
    </li>

</ul>
```

| Where | How | When to use it |
|---|---|---|
| **A filter** (`\|money`) | `f"£{pence / 100:,.2f}"` | **default** — written once, used everywhere |
| **In the SQL** | `printf('%.2f', price_pence / 100.0)` | when the SQL result goes somewhere else too, like a CSV |
| **In the template** | `{{ "%.2f"\|format(p / 100) }}` | one-off, and it clutters the markup |

> ⚠️ **`price_pence / 100` in SQL is integer division**: `3450 / 100` gives `34`, losing the pence silently. `/ 100.0` forces decimals. Python 3 doesn't have this problem (`/` is always float, `//` is the integer one).

**`{:,.2f}` decoded:** `,` = thousands separators, `.2` = exactly two decimal places, `f` = fixed-point. So `24999 / 100` → `249.99` → `£249.99`, and a million pence → `£10,000.00`.

---

### 7. Dates

**SQLite has no date type.** Dates are text in `'YYYY-MM-DD HH:MM:SS'` form, which is the format that sorts correctly as plain text — that's the whole reason for using it.

**The route:**

```python
@app.route("/orders/dates")
def dates():
    orders = rows(
        """
        SELECT id, placed_at,
               -- strftime formats a date inside SQLite itself
               strftime('%d %m %Y', placed_at) AS day,
               -- julianday() turns dates into numbers, so you can subtract them
               julianday('2026-10-07') - julianday(placed_at) AS days_ago
        FROM orders ORDER BY placed_at DESC
        """
    )
    return render_template_string(T_DATES, orders=orders)
```

**The template:**

```html
<ul>
  {% for o in orders %}
    <li>
      Order #{{ o["id"] }} —
      raw: {{ o["placed_at"] }}
      · filter: {{ o["placed_at"]|nice_date }}
      · with time: {{ o["placed_at"]|nice_date("%d/%m/%Y %H:%M") }}
      · from SQL: {{ o["day"] }}
      · {{ o["days_ago"]|int }} days ago
    </li>
  {% endfor %}
</ul>
```

**What the page gets:**

```html
<ul>

    <li>
      Order #5 —
      raw: 2026-10-06 07:55:00
      · filter: 6 Oct 2026
      · with time: 06/10/2026 07:55
      · from SQL: 06 10 2026
      · 0 days ago
    </li>

    <li>
      Order #4 —
      raw: 2026-10-05 20:41:00
      · filter: 5 Oct 2026
      · with time: 05/10/2026 20:41
      · from SQL: 05 10 2026
      · 1 days ago
    </li>

    <li>
      Order #3 —
      raw: 2026-10-04 09:15:00
      · filter: 4 Oct 2026
      · with time: 04/10/2026 09:15
      · from SQL: 04 10 2026
      · 2 days ago
    </li>

    <li>
      Order #2 —
      raw: 2026-10-03 16:20:00
      · filter: 3 Oct 2026
      · with time: 03/10/2026 16:20
      · from SQL: 03 10 2026
      · 3 days ago
    </li>

    <li>
      Order #1 —
      raw: 2026-09-28 11:02:00
      · filter: 28 Sep 2026
      · with time: 28/09/2026 11:02
      · from SQL: 28 09 2026
      · 8 days ago
    </li>

</ul>
```

| Job | Where | How |
|---|---|---|
| Show it nicely | Python filter | `datetime.strptime(...)` then `.strftime(...)` |
| Show it nicely | SQLite | `strftime('%d %m %Y', placed_at)` |
| Sort by date | SQLite | `ORDER BY placed_at` — text order *is* date order |
| "How long ago" | SQLite | `julianday('now') - julianday(placed_at)` = days, as a number |
| Today's rows | SQLite | `WHERE date(placed_at) = date('now')` |

> ⚠️ **`strftime` in SQLite has no `%b`** (short month name) — that's a Python-only code, which is why the SQL column above shows `03 10 2026` rather than `3 Oct 2026`. Month *names* are a job for the Python side.

> ⚠️ **Always store the same format.** Mix `'2026-10-03'` with `'03/10/2026'` and both sorting and parsing break. `datetime('now')` as a column default is the easy way to stay consistent.

---

### 8. 0 and 1 standing in for true/false

**SQLite has no boolean either** — you store `0` or `1`. Four ways to show one, and one way not to.

**The route:**

```python
@app.route("/customers/flags")
def flags():
    return render_template_string(T_FLAGS, customers=rows("SELECT * FROM customers ORDER BY id"))
```

**The template:**

```html
<table>
  {% for c in customers %}
    <tr>
      <td>{{ c["name"] }}</td>

      {# 1 is truthy and 0 is falsy, so a plain `if` just works. #}
      <td>{% if c["is_active"] %}Active{% else %}Closed{% endif %}</td>

      {# The same thing as a one-liner, when there is no block to write. #}
      <td>{{ "Yes" if c["is_active"] else "No" }}</td>

      {# Picking a CSS class from the value is how a flag becomes a coloured badge. #}
      <td><span class="badge {{ 'ok' if c["is_active"] else 'off' }}">
            {{ "open" if c["is_active"] else "closed" }}</span></td>

      {# ⚠️ printing it raw shows a number, which is never what you want on a page #}
      <td>{{ c["is_active"] }}</td>
    </tr>
  {% endfor %}
</table>
```

**What the page gets:**

```html
<table>

    <tr>
      <td>Ada Lovelace</td>

      <td>Active</td>

      <td>Yes</td>

      <td><span class="badge ok">
            open</span></td>

      <td>1</td>
    </tr>

    <tr>
      <td>Grace Hopper</td>

      <td>Active</td>

      <td>Yes</td>

      <td><span class="badge ok">
            open</span></td>

      <td>1</td>
    </tr>

    <tr>
      <td>Alan Turing</td>

      <td>Closed</td>

      <td>No</td>

      <td><span class="badge off">
            closed</span></td>

      <td>0</td>
    </tr>

    <tr>
      <td>Katherine J.</td>

      <td>Active</td>

      <td>Yes</td>

      <td><span class="badge ok">
            open</span></td>

      <td>1</td>
    </tr>

</table>
```

**`1` is truthy and `0` is falsy in both Python and Jinja**, so `{% if row["is_active"] %}` just works. No `== 1` needed.

The fourth column is the pattern worth stealing: choosing a **CSS class** from the data is how one number becomes a coloured badge, with the styling left in the stylesheet where it belongs.

> ⚠️ **Never print the raw flag.** The last column shows `1` and `0` on the page, which means nothing to a visitor. Always turn it into words.

---

### 9. A name from another table (JOIN)

**The data is in two tables and the page needs both.** Orders store a `customer_id`; the name lives in `customers`. A `JOIN` fetches them together, in one query.

**The route:**

```python
@app.route("/orders")
def join_name():
    orders = rows(
        """
        SELECT o.id, o.status, c.name AS customer_name, c.email
        FROM orders o
        JOIN customers c ON c.id = o.customer_id
        ORDER BY o.placed_at DESC
        """
    )
    return render_template_string(T_JOIN, orders=orders)
```

**The template:**

```html
<table>
  <thead><tr><th>Order</th><th>Customer</th><th>Email</th><th>Status</th></tr></thead>
  <tbody>
    {% for o in orders %}
      <tr>
        <td>#{{ o["id"] }}</td>
        {# "customer_name" exists only because the query renamed it with AS.
           Without AS you would have two columns called "name" and sqlite3 would
           hand you whichever came last. #}
        <td>{{ o["customer_name"] }}</td>
        <td>{{ o["email"] or "—" }}</td>
        <td>{{ o["status"] }}</td>
      </tr>
    {% endfor %}
  </tbody>
</table>
```

**What the page gets:**

```html
<table>
  <thead><tr><th>Order</th><th>Customer</th><th>Email</th><th>Status</th></tr></thead>
  <tbody>

      <tr>
        <td>#5</td>

        <td>Ada Lovelace</td>
        <td>ada@example.com</td>
        <td>new</td>
      </tr>

      <tr>
        <td>#4</td>

        <td>Katherine J.</td>
        <td>kj@example.com</td>
        <td>paid</td>
      </tr>

      <tr>
        <td>#3</td>

        <td>Grace Hopper</td>
        <td>—</td>
        <td>new</td>
      </tr>

      <tr>
        <td>#2</td>

        <td>Ada Lovelace</td>
        <td>ada@example.com</td>
        <td>paid</td>
      </tr>

      <tr>
        <td>#1</td>

        <td>Ada Lovelace</td>
        <td>ada@example.com</td>
        <td>shipped</td>
      </tr>

  </tbody>
</table>
```

**`AS customer_name` is doing essential work.** Both tables have a column called `name`, and `sqlite3` keeps only one of them in the row — whichever came last. Renaming with `AS` makes it unambiguous, and the template reads better for it.

> ⚠️ **`JOIN` drops rows that have no match; `LEFT JOIN` keeps them.** Joining orders to customers with a plain `JOIN` is right here (every order has a customer). Joining *customers to orders* that way would silently hide every customer who hasn't ordered yet. More on this in [[Flask SQLite - book tracker]].

---

### 10. Rows grouped under headings

**Rows under headings** — products by category, orders by month, tasks by status. One query, grouped in Python.

**The route:**

```python
@app.route("/catalogue")
def grouped():
    # ONE query, sorted by the thing you group by. groupby walks the list and
    # starts a new group whenever the key changes - so unsorted input gives you
    # the same heading several times. That ORDER BY is doing real work.
    products = rows("SELECT * FROM products ORDER BY category, name")
    groups = [(category, list(items)) for category, items in groupby(products, key=lambda p: p["category"])]
    return render_template_string(T_GROUPED, groups=groups)
```

**The template:**

```html
{% for category, items in groups %}
  <h2>{{ category }} <span class="muted">({{ items|length }})</span></h2>
  <ul>
    {% for p in items %}<li>{{ p["name"] }} — {{ p["price_pence"]|money }}</li>{% endfor %}
  </ul>
{% endfor %}
```

**What the page gets:**

```html
<h2>Desk gear <span class="muted">(2)</span></h2>
  <ul>
    <li>Keyboard tray — £18.99</li><li>Monitor stand — £22.50</li>
  </ul>

  <h2>Furniture <span class="muted">(3)</span></h2>
  <ul>
    <li>Filing cabinet — £89.99</li><li>Oak desk — £249.99</li><li>Office chair — £159.99</li>
  </ul>

  <h2>Lighting <span class="muted">(3)</span></h2>
  <ul>
    <li>Desk lamp — £34.50</li><li>Floor lamp — £79.99</li><li>LED bulb — £5.99</li>
  </ul>
```

**`groupby` only works on sorted input.** It walks the list and starts a new group whenever the key changes, so unsorted rows give you the same heading several times. That's what the `ORDER BY category` is for — it is not decoration.

**Why `list(items)`:** `groupby` hands back a lazy iterator that's consumed the moment the next group starts. Without `list()`, a template that loops over a group twice (or uses `|length`) finds it empty.

> **The alternative, when you need lookups rather than order:** build a dict.
> ```python
> from collections import defaultdict
> by_category = defaultdict(list)
> for p in products:
>     by_category[p["category"]].append(p)
> ```
> Then `{% for category, items in by_category.items() %}`. No sorting needed, and you can ask for one category directly.

---

### 11. The N+1 trap, and the fix

**The performance mistake everyone makes once.** You want a count beside each row, so you loop the rows and run a query for each. Five orders, six queries. Five hundred orders, five hundred and one.

**The route:**

```python
@app.route("/orders/fast")
def n_plus_one():
    # ONE query for the whole table. The obvious alternative - loop the orders
    # and run a COUNT for each - is 1 + 5 queries here, and 1 + 500 with 500
    # orders. That is the "N+1 problem", and it is the usual reason a page that
    # was fine in testing crawls once there is real data in it.
    orders = rows(
        """
        SELECT o.id,
               COUNT(i.id)                         AS item_count,
               SUM(i.quantity * i.unit_price_pence) AS total_pence
        FROM orders o
        LEFT JOIN order_items i ON i.order_id = o.id
        GROUP BY o.id
        ORDER BY o.id
        """
    )
    return render_template_string(T_NPLUS1, orders=orders)
```

**The template:**

```html
<table>
  <thead><tr><th>Order</th><th>Items</th><th>Total</th></tr></thead>
  <tbody>
    {% for o in orders %}
      <tr>
        <td>#{{ o["id"] }}</td>
        <td>{{ o["item_count"] }}</td>
        <td>{{ o["total_pence"]|money }}</td>
      </tr>
    {% endfor %}
  </tbody>
</table>
```

**What the page gets:**

```html
<table>
  <thead><tr><th>Order</th><th>Items</th><th>Total</th></tr></thead>
  <tbody>

      <tr>
        <td>#1</td>
        <td>2</td>
        <td>£318.99</td>
      </tr>

      <tr>
        <td>#2</td>
        <td>1</td>
        <td>£59.90</td>
      </tr>

      <tr>
        <td>#3</td>
        <td>3</td>
        <td>£220.47</td>
      </tr>

      <tr>
        <td>#4</td>
        <td>2</td>
        <td>£103.95</td>
      </tr>

      <tr>
        <td>#5</td>
        <td>1</td>
        <td>£34.50</td>
      </tr>

  </tbody>
</table>
```

**The slow version looks completely reasonable:**

```python
# DON'T - one extra query per row. This is the N+1 problem.
orders = rows("SELECT * FROM orders")
for o in orders:
    o["item_count"] = value("SELECT COUNT(*) FROM order_items WHERE order_id = ?", (o["id"],))
```

| Orders | Slow version | The query above |
|---|---|---|
| 5 | 6 queries | 1 |
| 500 | 501 queries | 1 |
| 5,000 | 5,001 queries | 1 |

It's fine with your five test rows and crawls with real data — which is exactly why it reaches production.

**The fix is always the same shape:** `LEFT JOIN` the child table, `GROUP BY` the parent, and let SQLite do the counting. `LEFT JOIN`, not `JOIN`, so an order with no items still appears, with a count of 0.

> **The template cannot cause this if it never queries.** That's the real reason for the rule at the top of this note.

---

### 12. Many-to-many: tags on each product

**Many-to-many: each product has several tags, each tag is on several products.** The link lives in a third table, and `GROUP_CONCAT` collapses the matches into one string per product.

**The route:**

```python
@app.route("/products/tags")
def tags_many():
    products = rows(
        """
        SELECT p.name,
               -- GROUP_CONCAT squashes many rows into one string. The separator is
               -- '|' rather than ',' so a tag containing a comma cannot break it.
               GROUP_CONCAT(t.name, '|') AS tag_list
        FROM products p
        LEFT JOIN product_tags pt ON pt.product_id = p.id
        LEFT JOIN tags t          ON t.id = pt.tag_id
        GROUP BY p.id
        ORDER BY p.name
        LIMIT 5
        """
    )
    return render_template_string(T_TAGS, products=products)
```

**The template:**

```html
<ul>
  {% for p in products %}
    <li>
      {{ p["name"] }}
      {# GROUP_CONCAT glued the tags into one string, so split it back apart.
         `if p["tag_list"]` guards against a product with no tags, where
         GROUP_CONCAT returns NULL and .split would crash. #}
      {% if p["tag_list"] %}
        {% for tag in p["tag_list"].split("|") %}<span class="badge">{{ tag }}</span>{% endfor %}
      {% else %}
        <span class="muted">no tags</span>
      {% endif %}
    </li>
  {% endfor %}
</ul>
```

**What the page gets:**

```html
<ul>

    <li>
      Desk lamp

        <span class="badge">sale</span>

    </li>

    <li>
      Filing cabinet

        <span class="badge">wooden</span><span class="badge">heavy</span>

    </li>

    <li>
      Floor lamp

        <span class="badge">energy saving</span>

    </li>

    <li>
      Keyboard tray

        <span class="muted">no tags</span>

    </li>

    <li>
      LED bulb

        <span class="badge">sale</span><span class="badge">energy saving</span>

    </li>

</ul>
```

**Why `'|'` as the separator** rather than a comma: a tag containing a comma would break the split and silently produce two tags. A character that can't appear in your data is the whole trick.

> ⚠️ **`GROUP_CONCAT` returns `NULL` when there's nothing to concatenate**, and `None.split()` raises `AttributeError`. Hence the `{% if p["tag_list"] %}` guard.

> ⚠️ **The order of concatenated values is not guaranteed.** If the tags must appear in a set order, that's a second query (`SELECT ... ORDER BY t.name`) grouped in Python, as in example 10.

**When to use which:**

| | `GROUP_CONCAT` | A second query, grouped in Python |
|---|---|---|
| Queries | 1 | 2 |
| Good for | a few short labels per row | long lists, needing order, or more than the name |
| You get | one string to split | real rows, with all their columns |

---

### 13. A parent with its children, and a total

**One parent, many children** — an order and its lines, a post and its comments. Two queries, deliberately.

**The route:**

```python
@app.route("/order/<int:order_id>")
def parent_children(order_id):
    # Two queries, not one: the order is ONE row, its items are MANY rows.
    # Joining them would repeat the order details on every item line.
    order = row(
        """
        SELECT o.*, c.name AS customer_name
        FROM orders o JOIN customers c ON c.id = o.customer_id
        WHERE o.id = ?
        """,
        (order_id,),
    )
    if order is None:
        abort(404)

    items = rows(
        """
        SELECT p.name, i.quantity, i.unit_price_pence,
               i.quantity * i.unit_price_pence AS line_pence
        FROM order_items i JOIN products p ON p.id = i.product_id
        WHERE i.order_id = ?
        ORDER BY p.name
        """,
        (order_id,),
    )
    return render_template_string(T_PARENT, order=order, items=items)
```

**The template:**

```html
<h1>Order #{{ order["id"] }}</h1>
<p>{{ order["customer_name"] }} · {{ order["placed_at"]|nice_date }} · {{ order["status"] }}</p>

<table>
  <thead><tr><th>Item</th><th>Qty</th><th>Each</th><th>Line</th></tr></thead>
  <tbody>
    {% for i in items %}
      <tr>
        <td>{{ i["name"] }}</td>
        <td>{{ i["quantity"] }}</td>
        <td>{{ i["unit_price_pence"]|money }}</td>
        <td>{{ i["line_pence"]|money }}</td>
      </tr>
    {% endfor %}
  </tbody>
  <tfoot>
    {# |sum(attribute="x") adds up one column of the rows you already have -
       no second query needed just to show a total. #}
    <tr><th colspan="3">Total</th><th>{{ items|sum(attribute="line_pence")|money }}</th></tr>
  </tfoot>
</table>
```

**What the page gets:**

```html
<h1>Order #3</h1>
<p>Grace Hopper · 4 Oct 2026 · new</p>

<table>
  <thead><tr><th>Item</th><th>Qty</th><th>Each</th><th>Line</th></tr></thead>
  <tbody>

      <tr>
        <td>Keyboard tray</td>
        <td>2</td>
        <td>£18.99</td>
        <td>£37.98</td>
      </tr>

      <tr>
        <td>Monitor stand</td>
        <td>1</td>
        <td>£22.50</td>
        <td>£22.50</td>
      </tr>

      <tr>
        <td>Office chair</td>
        <td>1</td>
        <td>£159.99</td>
        <td>£159.99</td>
      </tr>

  </tbody>
  <tfoot>

    <tr><th colspan="3">Total</th><th>£220.47</th></tr>
  </tfoot>
</table>
```

**Why not one joined query?** Joining would repeat the order's date, status and customer on every single item line, and you'd be picking the same values back out of the first row. One row for the one thing, many rows for the many things, is simpler to read and to template.

**The total is computed twice over, two different ways:**

| Where | How | Use when |
|---|---|---|
| In SQL | `i.quantity * i.unit_price_pence AS line_pence` | per-row maths — always do it here |
| In the template | `{{ items\|sum(attribute="line_pence") }}` | a total of rows you already have |

`|sum(attribute=...)` adds up one column of the rows the view handed over, so no extra query is needed just to show a total.

> ⚠️ **Store the price the customer actually paid** on the order line (`unit_price_pence`), not just a link to the product. Otherwise raising a price silently rewrites the value of every order in history.

---

### 14. Row numbers and the loop variable

**`loop` is a free variable Jinja gives you inside every `{% for %}`** — row numbers, first/last, and alternating classes, with no counter to maintain.

**The route:**

```python
@app.route("/products/numbered")
def loop_vars():
    return render_template_string(T_LOOP, products=rows("SELECT * FROM products ORDER BY price_pence LIMIT 4"))
```

**The template:**

```html
<table>
  {% for p in products %}
    {# loop.cycle alternates values - this is zebra striping with no JavaScript #}
    <tr class="{{ loop.cycle('odd', 'even') }}">
      <td>{{ loop.index }}</td>        {# 1, 2, 3...  (loop.index0 starts at 0) #}
      <td>{{ p["name"] }}</td>
      <td>
        {% if loop.first %}cheapest{% elif loop.last %}dearest{% endif %}
      </td>
      <td class="muted">{{ loop.revindex }} to go</td>
    </tr>
  {% endfor %}
</table>
<p class="muted">{{ products|length }} rows</p>
```

**What the page gets:**

```html
<table>

    <tr class="odd">
      <td>1</td>
      <td>LED bulb</td>
      <td>
        cheapest
      </td>
      <td class="muted">4 to go</td>
    </tr>

    <tr class="even">
      <td>2</td>
      <td>Keyboard tray</td>
      <td>

      </td>
      <td class="muted">3 to go</td>
    </tr>

    <tr class="odd">
      <td>3</td>
      <td>Monitor stand</td>
      <td>

      </td>
      <td class="muted">2 to go</td>
    </tr>

    <tr class="even">
      <td>4</td>
      <td>Desk lamp</td>
      <td>
        dearest
      </td>
      <td class="muted">1 to go</td>
    </tr>

</table>
<p class="muted">4 rows</p>
```

| Inside a loop | Is |
|---|---|
| `loop.index` | 1, 2, 3 … — what a human calls the row number |
| `loop.index0` | 0, 1, 2 … — what a programmer calls it |
| `loop.revindex` | how many are left, counting down |
| `loop.first` / `loop.last` | `True` on the first / last pass |
| `loop.length` | how many rows in total |
| `loop.cycle("odd", "even")` | alternates each pass — zebra stripes, free |

> ⚠️ **Row numbers are not ids.** `loop.index` is "third on this page" — on page 2 of a paginated table the first row is still `1`. For a link to a record, always use `{{ row["id"] }}`.

---

### 15. A totals row - in SQL, or in the template

**A summary table with a totals row.** Both ways of getting the totals are shown side by side, and they agree.

**The route:**

```python
@app.route("/stock")
def totals_row():
    summary = rows(
        """
        SELECT category,
               COUNT(*)                   AS products,
               SUM(stock)                 AS units,
               SUM(stock * price_pence)   AS value_pence
        FROM products GROUP BY category ORDER BY category
        """
    )
    grand = row(
        """
        SELECT COUNT(*) AS products, SUM(stock) AS units,
               SUM(stock * price_pence) AS value_pence
        FROM products
        """
    )
    return render_template_string(T_TOTALS, summary=summary, grand=grand)
```

**The template:**

```html
<table>
  <thead><tr><th>Category</th><th>Products</th><th>Stock</th><th>Stock value</th></tr></thead>
  <tbody>
    {% for r in summary %}
      <tr>
        <td>{{ r["category"] }}</td>
        <td>{{ r["products"] }}</td>
        <td>{{ r["units"] }}</td>
        <td>{{ r["value_pence"]|money }}</td>
      </tr>
    {% endfor %}
  </tbody>
  <tfoot>
    {# Adding up rows you already have, in the template. #}
    <tr>
      <th>All</th>
      <th>{{ summary|sum(attribute="products") }}</th>
      <th>{{ summary|sum(attribute="units") }}</th>
      <th>{{ summary|sum(attribute="value_pence")|money }}</th>
    </tr>
    {# The same totals, worked out by SQLite instead. Identical answer - use this
       one when the table is paginated, because the template can only add up the
       rows on the page it was given. #}
    <tr class="muted">
      <td>from SQL</td>
      <td>{{ grand["products"] }}</td>
      <td>{{ grand["units"] }}</td>
      <td>{{ grand["value_pence"]|money }}</td>
    </tr>
  </tfoot>
</table>
```

**What the page gets:**

```html
<table>
  <thead><tr><th>Category</th><th>Products</th><th>Stock</th><th>Stock value</th></tr></thead>
  <tbody>

      <tr>
        <td>Desk gear</td>
        <td>2</td>
        <td>10</td>
        <td>£217.98</td>
      </tr>

      <tr>
        <td>Furniture</td>
        <td>3</td>
        <td>4</td>
        <td>£839.96</td>
      </tr>

      <tr>
        <td>Lighting</td>
        <td>3</td>
        <td>178</td>
        <td>£2,649.99</td>
      </tr>

  </tbody>
  <tfoot>

    <tr>
      <th>All</th>
      <th>8</th>
      <th>192</th>
      <th>£3,707.93</th>
    </tr>

    <tr class="muted">
      <td>from SQL</td>
      <td>8</td>
      <td>192</td>
      <td>£3,707.93</td>
    </tr>
  </tfoot>
</table>
```

| | `{{ rows\|sum(attribute="x") }}` | A second SQL query |
|---|---|---|
| Adds up | only the rows on this page | **every** matching row |
| Cost | free, the data is already there | one more query |
| Use when | the table shows everything | the table is paginated or limited |

> ⚠️ **On a paginated table the template's total is wrong** — it can only add up the rows it was given, so "Total" would mean "total of these 20". That's the one case where you must ask SQLite.

**`SUM` ignores `NULL`s**, so a missing value doesn't poison the total (unlike adding in Python, where `None + 1` raises `TypeError`).

---

### 16. Clickable column headings that sort the table

**Clickable column headings.** Each heading is a link back to the same page with `?sort=` on it — no JavaScript at all.

**The route:**

```python
# The visitor sends a KEY; we look up OUR column. A ? parameter cannot hold a
# column name, so the name has to be pasted into the SQL - and pasting anything
# a visitor typed is how SQL injection happens. This dictionary is the fix.
SORTS = {"name": "name COLLATE NOCASE", "category": "category, name", "price": "price_pence DESC"}


@app.route("/products/sorted")
def sortable():
    sort = request.args.get("sort", "name")
    order_by = SORTS.get(sort, SORTS["name"])  # anything unrecognised falls back
    products = rows(f"SELECT * FROM products ORDER BY {order_by} LIMIT 4")
    return render_template_string(T_SORTABLE, products=products, sort=sort)
```

**The template:**

```html
<table>
  <thead>
    <tr>
      {# Each heading is a link back to this same page with ?sort=... on it.
         The arrow shows which column is in use. #}
      {% for key, label in [("name", "Product"), ("category", "Category"), ("price", "Price")] %}
        <th>
          <a href="{{ url_for('sortable', sort=key) }}">
            {{ label }}{% if sort == key %} ▲{% endif %}
          </a>
        </th>
      {% endfor %}
    </tr>
  </thead>
  <tbody>
    {% for p in products %}
      <tr><td>{{ p["name"] }}</td><td>{{ p["category"] }}</td><td>{{ p["price_pence"]|money }}</td></tr>
    {% endfor %}
  </tbody>
</table>
```

**What the page gets:**

```html
<table>
  <thead>
    <tr>

        <th>
          <a href="/products/sorted?sort=name">
            Product
          </a>
        </th>

        <th>
          <a href="/products/sorted?sort=category">
            Category
          </a>
        </th>

        <th>
          <a href="/products/sorted?sort=price">
            Price ▲
          </a>
        </th>

    </tr>
  </thead>
  <tbody>

      <tr><td>Oak desk</td><td>Furniture</td><td>£249.99</td></tr>

      <tr><td>Office chair</td><td>Furniture</td><td>£159.99</td></tr>

      <tr><td>Filing cabinet</td><td>Furniture</td><td>£89.99</td></tr>

      <tr><td>Floor lamp</td><td>Lighting</td><td>£79.99</td></tr>

  </tbody>
</table>
```

**`url_for('sortable', sort=key)` builds the link**, so the URL is never typed out by hand and keeps working if the route moves.

> ⚠️ **This is the one place SQL injection can sneak in.** A `?` parameter can only be a *value*, so `ORDER BY ?` is impossible and the column name has to go into the string. If that name came from the visitor, they'd be writing your SQL. The `SORTS` dictionary is the fix: the visitor sends a **key**, and you paste **your** value. `?sort=;DROP TABLE products` simply falls back to sorting by name.

**To add the "sort the other way" arrow**, carry the direction too: `?sort=price&dir=desc`, with a second dictionary for `ASC`/`DESC`. Never paste the direction either.

**To keep a search while sorting**, pass it through the link: `url_for('sortable', sort=key, q=q)` — see the pager in [[Flask SQLite - book tracker]].

---

### 17. Turning a number into a colour and a bar

**Turning a number into something you can see at a glance** — a word, a colour, and a bar.

**The route:**

```python
@app.route("/stock/levels")
def meters():
    return render_template_string(T_METERS, products=rows("SELECT * FROM products ORDER BY stock LIMIT 5"))
```

**The template:**

```html
<table>
  {% for p in products %}
    {% set level = "out" if p["stock"] == 0 else ("low" if p["stock"] < 5 else "ok") %}
    <tr class="row-{{ level }}">
      <td>{{ p["name"] }}</td>
      <td>{{ p["stock"] }}</td>
      <td>
        {% if level == "out" %}Out of stock
        {% elif level == "low" %}Only {{ p["stock"] }} left
        {% else %}In stock{% endif %}
      </td>
      <td>
        {# A bar chart with no library: one div, and a width worked out from the
           data. Capped at 100% so a big number cannot overflow the page. #}
        <div class="bar"><span style="width: {{ [p["stock"] * 2, 100]|min }}%"></span></div>
      </td>
    </tr>
  {% endfor %}
</table>
```

**What the page gets:**

```html
<table>

    <tr class="row-out">
      <td>Office chair</td>
      <td>0</td>
      <td>
        Out of stock

      </td>
      <td>

        <div class="bar"><span style="width: 0%"></span></div>
      </td>
    </tr>

    <tr class="row-low">
      <td>Filing cabinet</td>
      <td>1</td>
      <td>
        Only 1 left

      </td>
      <td>

        <div class="bar"><span style="width: 2%"></span></div>
      </td>
    </tr>

    <tr class="row-low">
      <td>Keyboard tray</td>
      <td>2</td>
      <td>
        Only 2 left

      </td>
      <td>

        <div class="bar"><span style="width: 4%"></span></div>
      </td>
    </tr>

    <tr class="row-low">
      <td>Oak desk</td>
      <td>3</td>
      <td>
        Only 3 left

      </td>
      <td>

        <div class="bar"><span style="width: 6%"></span></div>
      </td>
    </tr>

    <tr class="row-ok">
      <td>Monitor stand</td>
      <td>8</td>
      <td>
        In stock
      </td>
      <td>

        <div class="bar"><span style="width: 16%"></span></div>
      </td>
    </tr>

</table>
```

**`{% set %}` names a decision once.** Working out "is this low stock?" in four separate places is how three of them end up disagreeing after a change.

**The bar is one `<div>` and a `width` percentage.** No chart library, no JavaScript:

```css
.bar { background: #eee; border-radius: 4px; height: 8px; width: 120px; }
.bar span { display: block; height: 100%; border-radius: 4px; background: #2f6fde; }
```

> ⚠️ **Cap the width.** `{{ [value, 100]|min }}%` — without it, a stock of 300 becomes `width: 600%` and pushes your layout sideways.

**Colour must never be the only signal.** The row says "Out of stock" in words as well, because colour alone is invisible to a screen reader and to anyone colour-blind.

---

### 18. Search results with the match highlighted

**Search results with the matched text highlighted.** The query is a `LIKE`; the highlighting is the interesting half, because the obvious way to do it is both broken and dangerous.

**The route:**

```python
@app.route("/search")
def search_highlight():
    q = request.args.get("q", "").strip()
    products = rows("SELECT * FROM products WHERE name LIKE ? ORDER BY name", (f"%{q}%",)) if q else []
    return render_template_string(T_SEARCH, products=products, q=q)
```

**The template:**

```html
<form method="get"><input name="q" value="{{ q }}" /><button>Search</button></form>

<p class="muted">{{ products|length }} result{{ "s" if products|length != 1 else "" }} for "{{ q }}"</p>

<ul>
  {% for p in products %}
    {# The highlight filter is defined in app.py. It escapes the product name and
       adds only the <mark> tags we wrote, so a product called <script>... is
       still harmless. #}
    <li>{{ p["name"]|highlight(q) }}</li>
  {% endfor %}
</ul>
```

**What the page gets:**

```html
<form method="get"><input name="q" value="lamp" /><button>Search</button></form>

<p class="muted">2 results for "lamp"</p>

<ul>

    <li>Desk <mark>lamp</mark></li>

    <li>Floor <mark>lamp</mark></li>

</ul>
```

**The one-liner everybody tries first:**

```jinja
{{ p["name"]|e|replace(q, "<mark>" ~ q ~ "</mark>")|safe }}
```

It renders `Desk &lt;mark&gt;lamp&lt;/mark&gt;` — the tags show up as visible text. The reason: `|e` produces a `Markup` object, and `Markup.replace()` escapes whatever you pass it, *including* your `<mark>` tags. **I hit exactly this while writing the note.**

Removing the `|e` "fixes" the display and opens an XSS hole: now any HTML in the product name is live, and on a page that searches user-supplied text that's a script-injection waiting to happen.

**So it's a Python filter**, which escapes each piece of the text and adds only the tags we wrote. Tested against the nasty cases:

```python
highlight("Desk lamp", "lamp")                        # Desk <mark>lamp</mark>
highlight("LAMP post", "lamp")                        # <mark>LAMP</mark> post        (keeps the capitals)
highlight("<script>alert(1)</script> lamp", "lamp")   # &lt;script&gt;alert(1)&lt;/script&gt; <mark>lamp</mark>
highlight("a lamp here", "<img onerror=x>")           # a lamp here                   (evil search term, no match)
highlight(None, "lamp")                               # (empty, no crash)
```

> ⚠️ **`|safe` means "I promise this is safe HTML".** Only ever put it on markup **you** built. On anything a visitor typed it is a script-injection hole — the one rule that matters most in this whole note.

**The `%` wildcards go in Python, not the SQL string:** `("%" + q + "%",)` as a parameter. `LIKE '%?%'` doesn't work, because the `?` inside quotes is just a literal question mark.

---

### 19. Rows into JavaScript

**Handing rows to JavaScript** — for a chart library, a map, or anything interactive.

**The route:**

```python
@app.route("/chart")
def to_json():
    found = rows("SELECT name, stock FROM products ORDER BY stock DESC LIMIT 3")

    # A sqlite3.Row is NOT JSON. Without dict() you get
    # "Object of type Row is not JSON serializable" - the single most common
    # error when passing database rows to JavaScript.
    products = [dict(r) for r in found]

    return render_template_string(T_JSON, products=products)
```

**The template:**

```html
<div id="chart"></div>

<script>
  // |tojson writes the data straight into the page as a JavaScript value, and
  // escapes it safely. No fetch, no second request.
  const products = {{ products|tojson }};
  document.getElementById("chart").textContent =
    products.map(p => p.name + ": " + p.stock).join(", ");
</script>
```

**What the page gets:**

```html
<div id="chart"></div>

<script>
  // |tojson writes the data straight into the page as a JavaScript value, and
  // escapes it safely. No fetch, no second request.
  const products = [{"name": "LED bulb", "stock": 140}, {"name": "Desk lamp", "stock": 27}, {"name": "Floor lamp", "stock": 11}];
  document.getElementById("chart").textContent =
    products.map(p => p.name + ": " + p.stock).join(", ");
</script>
```

> ⚠️ **`dict(row)` is required.** A `sqlite3.Row` is not JSON, and passing one straight to `|tojson` gives `TypeError: Object of type Row is not JSON serializable`. This is the single most common error when wiring a database to JavaScript.

**`|tojson` is not just `json.dumps`.** It also escapes `<`, `>` and `&` into `\u003c`-style sequences, so a product named `</script>` can't break out of the script tag and run. Hand-rolling this with `json.dumps` **is** an injection hole.

| Getting data to JavaScript | How | When |
|---|---|---|
| Baked into the page | `{{ rows\|tojson }}` | the data is ready when the page loads |
| Fetched after load | `fetch("/books/api")` + a JSON route | it changes, or it's big |

The second one is [[Flask - building a JSON API]].

> ⚠️ **Only what the page may show.** `{{ user|tojson }}` on a row straight from `SELECT *` publishes the password hash and email into the page source. Select the columns you need.

---

### 20. The same markup in several places: a macro

**The same markup in several places.** A macro is a function that returns HTML: write it once, call it anywhere.

**The route:**

```python
@app.route("/highlights")
def macros():
    return render_template_string(
        T_MACRO,
        cheap=rows("SELECT * FROM products ORDER BY price_pence LIMIT 2"),
        dear=rows("SELECT * FROM products ORDER BY price_pence DESC LIMIT 2"),
    )
```

**The template:**

```html
{# A macro is a function that returns HTML. Define once, call anywhere. #}
{% macro price_tag(product) %}
  <span class="tag">
    {{ product["name"] }} — {{ product["price_pence"]|money }}
    {% if product["stock"] == 0 %}<em>(out of stock)</em>{% endif %}
  </span>
{% endmacro %}

<h2>Cheapest</h2>
{% for p in cheap %}{{ price_tag(p) }}{% endfor %}

<h2>Dearest</h2>
{% for p in dear %}{{ price_tag(p) }}{% endfor %}
```

**What the page gets:**

```html
<h2>Cheapest</h2>

  <span class="tag">
    LED bulb — £5.99

  </span>

  <span class="tag">
    Keyboard tray — £18.99

  </span>

<h2>Dearest</h2>

  <span class="tag">
    Oak desk — £249.99

  </span>

  <span class="tag">
    Office chair — £159.99
    <em>(out of stock)</em>
  </span>
```

**Macros vs includes — both reuse markup, differently:**

| | Macro | Include |
|---|---|---|
| Looks like | `{{ price_tag(p) }}` | `{% include "card.html" %}` |
| Takes arguments | ✅ yes — that's the point | ❌ no, it borrows the surrounding variables |
| Good for | a card, a badge, a form field, a table row | a header, a footer, a sidebar |

**To use a macro across several templates**, put it in its own file and import it:

```jinja
{# templates/_macros.html holds the macro, then in any template: #}
{% from "_macros.html" import price_tag %}
{{ price_tag(product) }}
```

The leading underscore is a convention meaning "a piece of a page, not a page".

---

## The filters you'll reach for

Jinja's built-ins, with what they do to real data:

| Filter | Example | Result |
|---|---|---|
| `length` | `{{ products\|length }}` | `8` |
| `sum` | `{{ items\|sum(attribute="line_pence") }}` | `31899` |
| `round` | `{{ 4.567\|round(1) }}` | `4.6` |
| `int` / `float` | `{{ "12"\|int }}` | `12` |
| `default` | `{{ missing\|default("—") }}` | `—` (undefined only) |
| `truncate` | `{{ long\|truncate(20) }}` | first 20 chars, then `...` |
| `upper` / `lower` / `title` | `{{ "oak desk"\|title }}` | `Oak Desk` |
| `trim` | `{{ "  x  "\|trim }}` | `x` |
| `replace` | `{{ name\|replace("_", " ") }}` | underscores become spaces |
| `join` | `{{ tags\|join(", ") }}` | `wooden, heavy` |
| `first` / `last` | `{{ products\|first }}` | the first row |
| `min` / `max` | `{{ [p.stock, 100]\|min }}` | the smaller one |
| `selectattr` | `{{ products\|selectattr("stock")\|list }}` | only rows where stock is truthy |
| `map` | `{{ products\|map(attribute="name")\|join(", ") }}` | just the names |
| `batch` | `{% for row in products\|batch(3) %}` | three per row — grids |
| `tojson` | `{{ data\|tojson }}` | safe JSON for `<script>` |
| `e` / `escape` | `{{ text\|e }}` | HTML turned into visible text |
| `safe` | `{{ html\|safe }}` | **only on markup you built** |
| `pprint` | `{{ row\|pprint }}` | debugging — shows the structure |

> ⚠️ **`selectattr` and `map(attribute=...)` use `.attr` access**, which `sqlite3.Row` doesn't support. For rows, filter in SQL with a `WHERE` — that's faster anyway — or convert with `dict(r)` first.

---

## Everything that goes wrong, and why

| What you see | Cause | Fix |
|---|---|---|
| `<sqlite3.Row object at 0x...>` | printed a whole row | print a column: `{{ row["name"] }}` |
| The page prints `None` | a `NULL` column, or `SUM` over no rows | `{{ x or "—" }}`, or `COALESCE(SUM(x), 0)` |
| A blank page, with a 200 OK | `row()` returned `None` and nobody checked | `if thing is None: abort(404)` |
| `UndefinedError: 'None' has no attribute 'x'` | same thing, caught later by a filter doing maths | the same `None` check |
| `<built-in method keys of sqlite3.Row ...>` on the page | a column whose name collides with a Row method | `{{ row["keys"] }}`, never `{{ row.keys }}` |
| `AttributeError: 'sqlite3.Row' object has no attribute ...` | `row.name` in **Python** (it only works in templates) | `row["name"]` |
| `'dict object' has no attribute ...` | the view never passed that name | add it to `render_template(...)` |
| `IndexError` on a column | it wasn't in the `SELECT` | add it, or use `SELECT *` |
| `jinja2.exceptions.UndefinedError` | a typo in a name | names are case-sensitive |
| `Object of type Row is not JSON serializable` | passed rows to `tojson`/`jsonify` | `[dict(r) for r in rows]` |
| `&lt;mark&gt;` showing as text | `Markup.replace()` escaped your tags | build the markup in Python (example 18) |
| Money is a penny out | stored as a float | store whole pence as `INTEGER` |
| `34` instead of `34.50` | integer division in SQL | `/ 100.0`, not `/ 100` |
| The same heading repeated | `groupby` on unsorted rows | `ORDER BY` the grouping column |
| A group is empty on second use | `groupby`'s lazy iterator | `list(items)` |
| The page is slow with real data | a query inside a loop (N+1) | one `JOIN` + `GROUP BY` (example 11) |
| Rows missing from a join | `JOIN` where you needed `LEFT JOIN` | `LEFT JOIN` |
| `AttributeError: 'NoneType' has no attribute 'split'` | `GROUP_CONCAT` returned `NULL` | guard with `{% if %}` |
| `ValueError: time data ... does not match format` | mixed date formats in the column | store one format everywhere |

---

## Debugging what the template was handed

When a page shows the wrong thing, find out which half is wrong — the data or the display:

```python
@app.route("/products")
def many_rows():
    products = rows("SELECT * FROM products ORDER BY name")

    print(len(products))                  # how many rows came back?
    print(dict(products[0]))              # what is actually in one? (a Row prints as gibberish)
    print(products[0].keys())             # which columns do I really have?

    return render_template("products.html", products=products)
```

With `--debug` on, `print()` output appears in the terminal running Flask. In the template itself:

```jinja
{{ products|length }}        {# did any rows arrive? #}
{{ products|pprint }}        {# dump the whole structure onto the page #}
{{ product|pprint }}         {# one row, readably #}
```

| Symptom | What it means |
|---|---|
| `0` rows in Python | the **query** is wrong — try it in `sqlite3 shop.db` by hand |
| Rows in Python, nothing on the page | the **template** is wrong — a typo in a name, or the loop is in the wrong place |
| Rows, but the wrong ones | the `WHERE` is wrong, or a parameter arrived as text when you expected a number |

---

## Related

[[Flask SQLite stack]] · [[Flask SQLite - notes app]] · [[Flask SQLite - book tracker]] · [[Flask - templates with Jinja]] · [[Flask - building a JSON API]] · [[Flask reference]] · [[Flask]] · [[HTML]] · [[CSS]] · [[SQL]] · [[SQL fundamentals]] · [[Python]] · [[Security in practice]]
