---
tags: [flask, sqlite, python, project, web, jinja, css, auth]
---

# Flask SQLite - notes app

**A working website where people sign up, log in, and keep their own private notes.** Every file is here, in full, and it was all run before being written down. Nothing is simplified to the point of being wrong.

Stack: [[Flask SQLite stack]] · Next project: [[Flask SQLite - book tracker]] · Hub: [[Flask]]

---

## What you're building

| Page | URL | What happens there |
|---|---|---|
| Register | `/register` | Pick a username and password, get an account |
| Log in | `/login` | Prove who you are; a cookie remembers it |
| My notes | `/` | A table of **your** notes, with search and sorting |
| New note | `/notes/new` | A form that saves a note |
| Edit note | `/notes/<id>/edit` | The same form, filled in |
| Delete | `/notes/<id>/delete` | A button that removes one |

Two people using it at once see completely separate lists — that's the part most tutorials get wrong, and it's tested at the bottom.

---

## The files

Six files and two folders. Flask finds `templates/` and `static/` **by those exact names**.

```
notes_app/
├── app.py                  everything Python: routes, queries, login
├── schema.sql              the CREATE TABLE statements
├── notes.db                created by init-db - this one file IS your data
├── templates/
│   ├── layout.html         the shell every page sits inside
│   ├── index.html          the notes table
│   ├── note_form.html      new + edit (one template does both)
│   ├── login.html
│   └── register.html
└── static/
    └── style.css           hand-written, about 100 lines
```

## Running it

```bash
python -m venv .venv
.venv\Scripts\activate          # macOS/Linux/WSL: source .venv/bin/activate
pip install flask

flask --app app init-db          # build the tables - once
flask --app app run --debug      # then open http://127.0.0.1:5000
```

---

## 1. `schema.sql` — what the data looks like

Two tables. A **user** has many **notes**; each note belongs to exactly one user, and that link is the `user_id` column.

```sql
-- The whole database, in one file. Running "flask init-db" feeds this to SQLite.
-- Writing the schema as SQL (instead of Python classes) means you can read exactly
-- what the database looks like, and run the same file in any SQLite tool.

-- DROP first so init-db can be run again on a messy database during development.
-- NEVER leave DROP lines in a file you run on real data - they delete everything.
DROP TABLE IF EXISTS notes;
DROP TABLE IF EXISTS users;

CREATE TABLE users (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,  -- SQLite fills this in for you
    username      TEXT    NOT NULL UNIQUE,            -- UNIQUE = the database itself rejects duplicates
    password_hash TEXT    NOT NULL,                   -- the SCRAMBLED password, never the real one
    created_at    TEXT    NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE notes (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id    INTEGER NOT NULL,                      -- which user owns this note
    title      TEXT    NOT NULL,
    body       TEXT    NOT NULL DEFAULT '',
    pinned     INTEGER NOT NULL DEFAULT 0,            -- SQLite has no real boolean: 0 = false, 1 = true
    created_at TEXT    NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT    NOT NULL DEFAULT (datetime('now')),

    -- A FOREIGN KEY says "user_id must be a real users.id".
    -- ON DELETE CASCADE = if the user is deleted, their notes go too.
    FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE
);

-- An INDEX makes "find this user's notes" fast once the table is big.
-- Without it SQLite reads every row in the table to find the matching ones.
CREATE INDEX idx_notes_user ON notes (user_id);
```

**Why `user_id` matters so much:** it's the difference between "a notes app" and "a notes app where everyone reads everyone else's notes". Every query later carries `WHERE user_id = ?`.

> ⚠️ **`DROP TABLE` at the top means `init-db` wipes everything.** That's what you want while building — you'll change the schema ten times. Take those lines out before the app holds anything real.

---

## 2. `app.py` — the Python

Long, but it's six small sections one after another. Each is explained below, and pasting them together in order gives you the file.

### The top: imports and configuration

```python
"""A complete Flask + SQLite notes app with user accounts, in one file.

Run it:
    pip install flask
    flask --app app init-db        <- creates notes.db from schema.sql (once)
    flask --app app run --debug    <- then open http://127.0.0.1:5000

Everything here is standard library plus Flask. No SQLAlchemy, no form library:
the point is that you can see every piece of the machinery.
"""

import hmac  # for comparing secrets safely - explained at check_csrf()
import os
import secrets  # for generating unguessable random tokens
import sqlite3
from functools import wraps  # keeps a decorated function's name - explained at login_required()

from flask import (
    Flask,
    abort,
    flash,
    g,
    redirect,
    render_template,
    request,
    session,
    url_for,
)

# Werkzeug is the library Flask is built on, so you already have it installed.
# It turns a password into a scrambled hash, and checks a password against a hash.
from werkzeug.security import check_password_hash, generate_password_hash

app = Flask(__name__)

# The SECRET_KEY signs the session cookie. Without it, a visitor could edit their
# own cookie to say "user_id: 1" and be logged in as you.
# In development a fixed string is fine; in production it MUST come from the
# environment and be random, or every logged-in session is forgeable.
app.config["SECRET_KEY"] = os.environ.get("SECRET_KEY", "dev-only-not-for-a-real-server")

# Where the database file lives. One file, next to app.py. That IS the database.
app.config["DATABASE"] = os.path.join(app.root_path, "notes.db")
```

> ⚠️ **The `SECRET_KEY` is what makes logging in trustworthy.** It signs the cookie that says who you are. If a visitor can guess it, they can write their own cookie saying they're you. A fixed `"dev-only..."` string is fine on your laptop and a disaster on a server — in production it comes from the environment and is random.

### Section 1: opening and closing the database

```python
# ---------------------------------------------------------------------------
# 1. The database: opening it, closing it, creating it
# ---------------------------------------------------------------------------


def get_db():
    """Return this request's database connection, opening one if needed.

    `g` is Flask's per-request scratchpad: a fresh empty object for every
    request, thrown away when the response is sent. Storing the connection
    there means one connection per request, shared by every function that
    needs it - instead of opening a new one on every query.
    """
    if "db" not in g:
        g.db = sqlite3.connect(app.config["DATABASE"])

        # By default sqlite3 hands you plain tuples: row[0], row[1]...
        # Row lets you write row["title"], which is far easier to read and
        # doesn't break when you add a column.
        g.db.row_factory = sqlite3.Row

        # SQLite ignores FOREIGN KEY rules unless you switch them on, per
        # connection, every time. Without this line ON DELETE CASCADE does nothing.
        g.db.execute("PRAGMA foreign_keys = ON")
    return g.db


@app.teardown_appcontext
def close_db(exception=None):
    """Close the connection after every request. Flask calls this for us."""
    db = g.pop("db", None)
    if db is not None:
        db.close()


def init_db():
    """Create the tables by running schema.sql."""
    db = get_db()
    with open(os.path.join(app.root_path, "schema.sql"), encoding="utf-8") as f:
        db.executescript(f.read())  # executescript runs several statements at once
    db.commit()


@app.cli.command("init-db")
def init_db_command():
    """Add "flask --app app init-db" to the command line.

    The real work lives in init_db() above, because a @cli.command can only be
    called by the flask command - the tests call init_db() directly instead.
    """
    init_db()
    print("Database ready:", app.config["DATABASE"])
```

**What `g` is for.** Opening a connection costs time. Without `g`, every function that needs the database opens its own; with it, the first one opens a connection and the rest reuse it, and Flask closes it when the response is sent.

| Line | Why you'd miss it |
|---|---|
| `row_factory = sqlite3.Row` | lets you write `row["title"]` instead of `row[3]` |
| `PRAGMA foreign_keys = ON` | without it, `ON DELETE CASCADE` quietly does nothing |
| `db.commit()` | without it, writes vanish when the request ends |

### Section 2: who's logged in

```python
# ---------------------------------------------------------------------------
# 2. Who is logged in
# ---------------------------------------------------------------------------


@app.before_request
def load_logged_in_user():
    """Run before every request: turn session["user_id"] into g.user.

    `session` is a dictionary stored in a signed cookie in the visitor's
    browser. It survives between requests, and the signature means the visitor
    can read it but cannot change it without the SECRET_KEY.

    Only the id lives in the cookie. The username is looked up fresh each
    request, so renaming or deleting a user takes effect immediately.
    """
    user_id = session.get("user_id")
    if user_id is None:
        g.user = None
    else:
        g.user = get_db().execute("SELECT * FROM users WHERE id = ?", (user_id,)).fetchone()


def login_required(view):
    """Decorator: send anonymous visitors to the login page.

    @wraps copies the original function's name onto the wrapper. Without it
    every decorated view would be called "wrapped_view", and Flask would
    refuse to start because two routes had the same endpoint name.
    """

    @wraps(view)
    def wrapped_view(**kwargs):
        if g.user is None:
            flash("Please log in first.")
            return redirect(url_for("login"))
        return view(**kwargs)

    return wrapped_view


@app.context_processor
def inject_user():
    """Make `user` available in every template without passing it every time."""
    return {"user": g.get("user")}
```

**How logging in actually works.** The session is a dictionary Flask stores in a cookie, signed with your `SECRET_KEY`. Logging in puts `user_id` in it. `@app.before_request` then runs on **every** request and turns that id back into `g.user`.

Only the id is in the cookie, never the username or anything else — so the name is always fresh, and a deleted account stops working immediately.

> **Why `@wraps(view)` is in `login_required`:** a decorator replaces your function with a new one, and the new one is called `wrapped_view`. Flask names routes after the function, so without `@wraps` the second decorated route crashes the app with "two routes named wrapped_view". One line, saves a baffling error.

### Section 3: CSRF — stopping another site posting as you

```python
# ---------------------------------------------------------------------------
# 3. CSRF protection
# ---------------------------------------------------------------------------
# The attack: you are logged in here. Another site contains a hidden form that
# POSTs to our /notes/1/delete. Your browser helpfully attaches your cookie,
# and the note is deleted - you never clicked anything on our site.
#
# The fix: put a secret random token in the session AND in a hidden field of
# every form. The other site cannot read your session, so it cannot guess the
# token, so its forged POST fails.


def csrf_token():
    """The token for this visitor, created on first use and kept in the session."""
    if "csrf_token" not in session:
        session["csrf_token"] = secrets.token_urlsafe(32)
    return session["csrf_token"]


# Registering it as a Jinja global means templates can call csrf_token() directly.
app.jinja_env.globals["csrf_token"] = csrf_token


@app.before_request
def check_csrf():
    """Reject any POST whose token is missing or wrong."""
    if request.method == "POST":
        sent = request.form.get("csrf_token", "")
        expected = session.get("csrf_token", "")
        # hmac.compare_digest compares in constant time. A normal == returns
        # faster when the first character is wrong, which leaks the answer to an
        # attacker timing the responses. Always use it on secrets.
        if not expected or not hmac.compare_digest(sent, expected):
            abort(400, "Missing or invalid CSRF token.")
```

**The attack, in plain terms.** You're logged in here. You visit some other page that contains:

```html
<form action="https://your-notes-app/notes/1/delete" method="post">
  <input type="submit" value="Click for a free prize" />
</form>
```

Your browser attaches your cookie to that POST because it always does. The note gets deleted.

**The fix:** every one of our forms carries a random token that's also in your session. The other site can't read your session, so it can't include the right token, so `check_csrf` rejects it with 400.

> **`hmac.compare_digest` instead of `==`:** `==` on strings stops at the first wrong character, so it returns *slightly* faster for a nearly-right guess. Measured over thousands of tries, that timing leaks the answer. `compare_digest` always takes the same time. Use it for any secret comparison.

### Section 4: register, log in, log out

```python
# ---------------------------------------------------------------------------
# 4. Register, log in, log out
# ---------------------------------------------------------------------------


@app.route("/register", methods=["GET", "POST"])
def register():
    # One function handles both jobs: GET shows the empty form, POST receives it.
    # methods=["GET", "POST"] is required - without POST, submitting gives 405.
    if request.method == "POST":
        # request.form is a dict of the submitted fields, keyed by their name="".
        # .strip() removes stray spaces, so " bob " and "bob" are the same name.
        username = request.form.get("username", "").strip()
        password = request.form.get("password", "")

        # Validate on the SERVER. HTML's `required` attribute is a convenience
        # for honest users; anything can POST to this URL directly.
        errors = []
        if len(username) < 3:
            errors.append("Username must be at least 3 characters.")
        if len(password) < 8:
            errors.append("Password must be at least 8 characters.")

        if not errors:
            try:
                db = get_db()
                db.execute(
                    "INSERT INTO users (username, password_hash) VALUES (?, ?)",
                    # generate_password_hash turns "hunter2" into a long scrambled
                    # string. It cannot be reversed, so a stolen database does not
                    # hand over anyone's password.
                    (username, generate_password_hash(password)),
                )
                db.commit()  # nothing is saved until commit()
            except sqlite3.IntegrityError:
                # Raised by UNIQUE on users.username. Letting the database decide
                # avoids the race where two people register the same name at once.
                errors.append(f"The name {username} is already taken.")
            else:
                flash("Account created - please log in.")
                return redirect(url_for("login"))

        for error in errors:
            flash(error)

    return render_template("register.html")


@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        username = request.form.get("username", "").strip()
        password = request.form.get("password", "")
        user = get_db().execute("SELECT * FROM users WHERE username = ?", (username,)).fetchone()

        # One vague message for both "no such user" and "wrong password".
        # Saying which was wrong tells an attacker which usernames exist.
        if user is None or not check_password_hash(user["password_hash"], password):
            flash("Wrong username or password.")
        else:
            # Clear first: it throws away the old session, so a token an attacker
            # planted in the visitor's browser before login cannot be reused after.
            session.clear()
            session["user_id"] = user["id"]
            return redirect(url_for("index"))

    return render_template("login.html")


@app.route("/logout", methods=["POST"])  # POST, so a link or image cannot log you out
def logout():
    session.clear()
    flash("Logged out.")
    return redirect(url_for("login"))
```

**Why one function handles both GET and POST.** A browser asks for the form with GET, and sends it back with POST. Same URL, two jobs, so `methods=["GET", "POST"]` and an `if`. Leave `"POST"` out and submitting gives *405 Method Not Allowed*.

**What hashing is.** `generate_password_hash("hunter2")` produces something like `scrypt:32768:8:1$rLb...$9f2c...`. You cannot turn it back into `hunter2`. At login, Flask hashes what was typed and compares hashes. So a stolen database hands over nobody's password.

> ⚠️ **"Wrong username or password" is deliberately vague.** Saying *which* one was wrong tells a stranger whether an account exists, which is the first step in attacking it.

> ⚠️ **`session.clear()` before setting `user_id`.** An attacker can plant a known session on a visitor's browser *before* they log in, then reuse it afterwards. Clearing first throws it away. This is called session fixation.

### Section 5: the notes themselves

```python
# ---------------------------------------------------------------------------
# 5. The notes: list, create, edit, delete
# ---------------------------------------------------------------------------

# Columns the visitor is allowed to sort by. A column name CANNOT be a ? parameter
# (parameters are for values), so it has to be pasted into the SQL string - which
# means a visitor sending ?sort=evil would be pasting SQL into our query.
# Checking against this dictionary first is what makes that safe.
SORTS = {
    "newest": "pinned DESC, updated_at DESC",
    "oldest": "pinned DESC, updated_at ASC",
    "title": "pinned DESC, title COLLATE NOCASE ASC",  # NOCASE so "apple" and "Apple" sort together
}


@app.route("/")
@login_required
def index():
    """The notes table, with a search box and sorting."""
    # request.args is the ?query=string part of the URL (request.form is the body).
    search = request.args.get("q", "").strip()
    sort = request.args.get("sort", "newest")
    order_by = SORTS.get(sort, SORTS["newest"])  # unknown value falls back, never pasted

    # WHERE user_id = ? is what keeps users out of each other's notes.
    sql = f"SELECT * FROM notes WHERE user_id = ?"  # noqa: F541 - built up below
    params = [g.user["id"]]

    if search:
        # LIKE with % around it means "contains". The ? keeps the user's text as
        # DATA: typing ' OR 1=1 -- searches for that text instead of running it.
        sql += " AND (title LIKE ? OR body LIKE ?)"
        params += [f"%{search}%", f"%{search}%"]

    sql += f" ORDER BY {order_by}"  # safe: order_by came from SORTS, not the visitor
    notes = get_db().execute(sql, params).fetchall()

    return render_template("index.html", notes=notes, search=search, sort=sort)


def get_own_note(note_id):
    """Fetch one note, but only if the logged-in user owns it.

    This is the single most important function in the file. @login_required only
    proves SOMEONE is logged in - it does not stop user B opening /notes/3/edit
    and editing user A's note. Checking user_id in the WHERE clause does.

    404, not 403: telling a stranger "that note exists but is not yours" is
    itself a leak.
    """
    note = (
        get_db()
        .execute("SELECT * FROM notes WHERE id = ? AND user_id = ?", (note_id, g.user["id"]))
        .fetchone()
    )
    if note is None:
        abort(404)
    return note


@app.route("/notes/new", methods=["GET", "POST"])
@login_required
def create():
    if request.method == "POST":
        title = request.form.get("title", "").strip()
        body = request.form.get("body", "").strip()
        # "pinned" is a checkbox. An unticked checkbox is not submitted at all,
        # so it is absent from request.form - hence `in`, not a value check.
        pinned = 1 if "pinned" in request.form else 0

        if not title:
            flash("A note needs a title.")
            # Hand the typed text back so the visitor does not retype it.
            return render_template("note_form.html", note={"title": title, "body": body, "pinned": pinned})

        db = get_db()
        db.execute(
            "INSERT INTO notes (user_id, title, body, pinned) VALUES (?, ?, ?, ?)",
            (g.user["id"], title, body, pinned),
        )
        db.commit()
        flash("Note saved.")
        # Redirect after a successful POST, never render. Otherwise the browser
        # stays on the POSTed URL and refreshing saves the note a second time.
        return redirect(url_for("index"))

    return render_template("note_form.html", note=None)


@app.route("/notes/<int:note_id>/edit", methods=["GET", "POST"])
@login_required
def edit(note_id):
    # <int:note_id> converts the URL piece to an int and 404s on /notes/abc/edit.
    note = get_own_note(note_id)

    if request.method == "POST":
        title = request.form.get("title", "").strip()
        body = request.form.get("body", "").strip()
        pinned = 1 if "pinned" in request.form else 0

        if not title:
            flash("A note needs a title.")
        else:
            db = get_db()
            db.execute(
                # updated_at is set here because SQLite's DEFAULT only applies on INSERT.
                "UPDATE notes SET title = ?, body = ?, pinned = ?, updated_at = datetime('now')"
                " WHERE id = ? AND user_id = ?",  # the ownership check again, belt and braces
                (title, body, pinned, note_id, g.user["id"]),
            )
            db.commit()
            flash("Note updated.")
            return redirect(url_for("index"))

    return render_template("note_form.html", note=note)


@app.route("/notes/<int:note_id>/delete", methods=["POST"])  # POST only: deleting is not a GET
@login_required
def delete(note_id):
    get_own_note(note_id)  # 404s if it is not theirs, before anything is deleted
    db = get_db()
    db.execute("DELETE FROM notes WHERE id = ? AND user_id = ?", (note_id, g.user["id"]))
    db.commit()
    flash("Note deleted.")
    return redirect(url_for("index"))
```

**The `SORTS` dictionary is a security measure, not tidiness.** A `?` parameter can only ever be a *value*, so `ORDER BY ?` doesn't work and the column has to go into the string. Letting the visitor choose what gets pasted in means letting them write SQL. So they send a *key* (`newest`), and we paste **our** value. Anything unrecognised falls back.

**`get_own_note` is the most important function in the file.** `@login_required` only proves *someone* is logged in. It does nothing to stop user B opening `/notes/3/edit` and editing user A's note — the URL is just a number, and guessing numbers is easy. The `AND user_id = ?` is the actual protection.

It returns **404, not 403**, because "that exists but isn't yours" is itself a leak.

> ⚠️ **Redirect after a successful POST, never render.** If you render, the browser stays on the POSTed URL, and refreshing re-submits — a second identical note, or a second payment. `redirect()` moves them to a normal GET first. (The POST-redirect-GET pattern.)

### The bottom

```python
if __name__ == "__main__":
    # Running "python app.py" works, but "flask --app app run --debug" is better:
    # it reloads on every save and shows a proper error page.
    app.run(debug=True)
```

---

## 3. The templates — HTML with holes in it

Jinja2 is HTML plus three pieces of punctuation:

| You write | It means |
|---|---|
| `{{ name }}` | print this value here |
| `{% if %}` `{% for %}` `{% endfor %}` | decide and repeat |
| `{# ... #}` | a comment, removed before sending |

### `templates/layout.html` — the shell

Every page is this file with one hole filled in, so the navbar and flash messages are written once.

```html
{# The shared shell every page sits inside. Child templates say
   {% extends "layout.html" %} and then fill in the blocks below.

   These {# ... #} comments are JINJA comments: they are stripped before the page
   is sent, so visitors never see them. Use them, not <!-- HTML comments -->, when
   writing about Jinja - Jinja still reads its own tags inside an HTML comment and
   a stray one in there is a crash, not a note to yourself. #}
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    {# Without this line a phone pretends to be a 980px-wide desktop and shrinks everything. #}
    <meta name="viewport" content="width=device-width, initial-scale=1" />

    {# A "block" is a hole a child template can fill. The text inside is the
       default, used by any page that does not fill it in. #}
    <title>{% block title %}Notes{% endblock %}</title>

    {# url_for('static', ...) builds the correct path to files in static/.
       Hard-coding "/static/style.css" breaks the day the app moves to a subfolder. #}
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}" />
  </head>
  <body>
    <header class="topbar">
      <a class="brand" href="{{ url_for('index') }}">Notes</a>

      <nav>
        {# `user` is available on every page because of @app.context_processor in app.py #}
        {% if user %}
          <span class="who">{{ user["username"] }}</span>
          {# Logging out is a form, not a link: a GET link can be triggered by any
             other site's <img> tag, which would log you out without asking. #}
          <form method="post" action="{{ url_for('logout') }}">
            <input type="hidden" name="csrf_token" value="{{ csrf_token() }}" />
            <button class="link-button" type="submit">Log out</button>
          </form>
        {% else %}
          <a href="{{ url_for('login') }}">Log in</a>
          <a href="{{ url_for('register') }}">Register</a>
        {% endif %}
      </nav>
    </header>

    <main>
      {# Messages left by flash() in the view. get_flashed_messages() both reads
         and clears them, so each message is shown exactly once. #}
      {% for message in get_flashed_messages() %}
        <p class="flash">{{ message }}</p>
      {% endfor %}

      {% block content %}{% endblock %}
    </main>
  </body>
</html>
```

> ⚠️ **Jinja reads its tags inside HTML comments too.** `<!-- {% block %} -->` is a crash, not a note to yourself — which is exactly the error I hit building this. Use Jinja's own `{# ... #}` comments: they also never reach the visitor.

### `templates/index.html` — the table

```html
{# extends must be the first line: it says "start from layout.html, fill in its blocks" #}
{% extends "layout.html" %}

{% block title %}My notes{% endblock %}

{% block content %}
  <div class="page-head">
    <h1>My notes</h1>
    <a class="button" href="{{ url_for('create') }}">New note</a>
  </div>

  {# A search form uses GET, not POST: the search belongs in the URL so the
     visitor can bookmark it, share it, and refresh without a warning box. #}
  <form class="toolbar" method="get" action="{{ url_for('index') }}">
    <input type="search" name="q" value="{{ search }}" placeholder="Search title or body" />

    <select name="sort">
      {# `selected` keeps the dropdown on the choice already in use after reload #}
      <option value="newest" {% if sort == 'newest' %}selected{% endif %}>Newest first</option>
      <option value="oldest" {% if sort == 'oldest' %}selected{% endif %}>Oldest first</option>
      <option value="title"  {% if sort == 'title'  %}selected{% endif %}>By title</option>
    </select>

    <button type="submit">Go</button>
    {% if search %}<a class="clear" href="{{ url_for('index') }}">Clear</a>{% endif %}
  </form>

  {# An empty list is a normal state, not an error - always give it a message,
     otherwise a new user just sees a blank page and assumes it is broken. #}
  {% if not notes %}
    <p class="empty">
      {% if search %}
        Nothing matches "{{ search }}".
      {% else %}
        No notes yet. Make your first one.
      {% endif %}
    </p>
  {% else %}
    <table>
      <thead>
        <tr>
          <th>Title</th>
          <th>Updated</th>
          <th class="right">Actions</th>
        </tr>
      </thead>
      <tbody>
        {% for note in notes %}
          <tr>
            <td>
              {% if note["pinned"] %}<span class="pin" title="Pinned">PIN</span>{% endif %}
              <a href="{{ url_for('edit', note_id=note['id']) }}">{{ note["title"] }}</a>

              {# Jinja escapes every {{ }} by default, so a note titled
                 <script>alert(1)</script> is DISPLAYED, not run. This is the one
                 protection you get for free - and the reason never to undo it
                 with |safe on text a user typed. #}
              {% if note["body"] %}
                <span class="preview">{{ note["body"][:60] }}{% if note["body"]|length > 60 %}…{% endif %}</span>
              {% endif %}
            </td>
            <td class="muted">{{ note["updated_at"] }}</td>
            <td class="right">
              <a href="{{ url_for('edit', note_id=note['id']) }}">Edit</a>

              {# Deleting changes data, so it is a POST form with a token - not a link.
                 onsubmit is a plain-JavaScript confirm box: the only JS in this app. #}
              <form method="post"
                    action="{{ url_for('delete', note_id=note['id']) }}"
                    onsubmit="return confirm('Delete this note?')">
                <input type="hidden" name="csrf_token" value="{{ csrf_token() }}" />
                <button class="danger" type="submit">Delete</button>
              </form>
            </td>
          </tr>
        {% endfor %}
      </tbody>
    </table>

    {# |length is a Jinja filter: it calls len() for you. #}
    <p class="muted">{{ notes|length }} note{{ "s" if notes|length != 1 else "" }}.</p>
  {% endif %}
{% endblock %}
```

**Why the search form is GET and the delete form is POST.** A GET puts everything in the URL, so a search can be bookmarked, shared and refreshed. A POST has a body and can change things — which is why anything that changes data must be one. A delete link would be followed by any browser that pre-loads links.

> ⚠️ **`{{ }}` escapes HTML for you.** A note titled `<script>alert(1)</script>` is *shown*, not run, because Jinja turns `<` into `&lt;`. That's the one security win you get free — and the reason never to put `|safe` on text a user typed. Tested below.

### `templates/note_form.html` — new and edit in one file

```html
{% extends "layout.html" %}

{# One template for both creating and editing. `note` is None when creating,
   and a database row when editing - so every spot that differs asks about it. #}
{% block title %}{{ "Edit note" if note else "New note" }}{% endblock %}

{% block content %}
  <h1>{{ "Edit note" if note else "New note" }}</h1>

  {# No action="" means "post back to the URL I am already on", which is exactly
     what we want: /notes/new posts to /notes/new, /notes/4/edit posts to itself. #}
  <form method="post" class="stack">
    <input type="hidden" name="csrf_token" value="{{ csrf_token() }}" />

    <label for="title">Title</label>
    {# `value` is what refills the box: the existing note, or the rejected text
       handed back after a failed save, or empty. `or ''` turns a missing value
       into an empty string instead of printing the word "None". #}
    <input id="title" name="title" maxlength="200" required
           value="{{ note["title"] if note else "" }}" />

    <label for="body">Body</label>
    {# A textarea has no value attribute - its content goes BETWEEN the tags,
       and the tags must touch the text or you save leading spaces and newlines. #}
    <textarea id="body" name="body" rows="10">{{ note["body"] if note else "" }}</textarea>

    <label class="check">
      {# A ticked checkbox submits pinned=on; an unticked one submits nothing at
         all, which is why app.py tests `"pinned" in request.form`. #}
      <input type="checkbox" name="pinned" {% if note and note["pinned"] %}checked{% endif %} />
      Pin to the top
    </label>

    <div class="row">
      <button type="submit">{{ "Save changes" if note else "Create note" }}</button>
      <a class="clear" href="{{ url_for('index') }}">Cancel</a>
    </div>
  </form>
{% endblock %}
```

**One template, two jobs.** `note` is `None` when creating and a row when editing, so every spot that differs asks `if note`. Two near-identical templates would drift apart the first time you changed one.

### `templates/login.html` and `templates/register.html`

```html
{% extends "layout.html" %}

{% block title %}Log in{% endblock %}

{% block content %}
  <h1>Log in</h1>

  <form method="post" class="stack narrow">
    <input type="hidden" name="csrf_token" value="{{ csrf_token() }}" />

    <label for="username">Username</label>
    {# autofocus puts the cursor here on load. autocomplete tells the browser's
       password manager what this field is, so it offers to fill it in. #}
    <input id="username" name="username" autocomplete="username" autofocus required />

    <label for="password">Password</label>
    {# type="password" hides the characters. It does NOT encrypt anything:
       without HTTPS the password still crosses the network readable. #}
    <input id="password" name="password" type="password" autocomplete="current-password" required />

    <button type="submit">Log in</button>
  </form>

  <p class="muted">No account? <a href="{{ url_for('register') }}">Register</a>.</p>
{% endblock %}
```

```html
{% extends "layout.html" %}

{% block title %}Register{% endblock %}

{% block content %}
  <h1>Create an account</h1>

  <form method="post" class="stack narrow">
    <input type="hidden" name="csrf_token" value="{{ csrf_token() }}" />

    <label for="username">Username</label>
    {# minlength here matches the check in app.py. The browser's version is a
       courtesy that saves a round trip; the server's version is the real rule,
       because anything can POST straight to this URL. #}
    <input id="username" name="username" minlength="3" maxlength="50"
           autocomplete="username" autofocus required />

    <label for="password">Password</label>
    <input id="password" name="password" type="password" minlength="8"
           autocomplete="new-password" required />
    <p class="hint">At least 8 characters.</p>

    <button type="submit">Create account</button>
  </form>

  <p class="muted">Already have one? <a href="{{ url_for('login') }}">Log in</a>.</p>
{% endblock %}
```

---

## 4. `static/style.css` — about 100 lines, written by hand

No framework. Files in `static/` are sent to the browser exactly as they are.

```css
/* Hand-written CSS. No framework, no build step - the browser reads this file
   exactly as it is. Roughly 100 lines is enough for a real, tidy app. */

/* Custom properties ("CSS variables"). Declaring the colours once here means
   changing the look is one edit, not forty. Use them as var(--ink). */
:root {
  --bg: #f6f7f9;
  --card: #ffffff;
  --ink: #1b1f24;
  --muted: #697280;
  --line: #dfe3e8;
  --accent: #2f6fde;
  --danger: #c0392b;
  --radius: 8px;
}

/* The browser's own styles differ between browsers, so start by flattening
   the few that matter instead of pulling in a whole reset file. */
* {
  box-sizing: border-box; /* padding counts INSIDE a width, which is what you expect */
}

body {
  margin: 0;
  /* system-ui = whatever font this device already uses. No download, no delay. */
  font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
  line-height: 1.5;
  color: var(--ink);
  background: var(--bg);
}

/* ---------- layout ---------- */

.topbar {
  display: flex; /* lays children in a row */
  align-items: center; /* ...vertically centred */
  gap: 1rem;
  padding: 0.75rem 1rem;
  background: var(--card);
  border-bottom: 1px solid var(--line);
}

.brand {
  font-weight: 700;
  font-size: 1.1rem;
  color: var(--ink);
  text-decoration: none;
}

.topbar nav {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-left: auto; /* the trick that pushes the nav to the right edge */
}

main {
  /* max-width stops text stretching to 3000px on a wide monitor;
     margin: auto centres what is left. */
  max-width: 820px;
  margin: 0 auto;
  padding: 1.5rem 1rem 4rem;
}

.page-head,
.toolbar,
.row {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  flex-wrap: wrap; /* on a narrow phone the items drop onto the next line */
}

.page-head {
  justify-content: space-between;
  margin-bottom: 1rem;
}

.toolbar {
  margin-bottom: 1rem;
}

/* ---------- forms ---------- */

/* .stack is a column of label-then-field pairs with even spacing. */
.stack {
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
  background: var(--card);
  border: 1px solid var(--line);
  border-radius: var(--radius);
  padding: 1rem;
}

.narrow {
  max-width: 24rem;
}

label {
  font-weight: 600;
  font-size: 0.9rem;
}

input,
textarea,
select,
button {
  font: inherit; /* form controls ignore the body font unless told to inherit it */
}

input,
textarea,
select {
  padding: 0.5rem;
  border: 1px solid var(--line);
  border-radius: 6px;
  background: #fff;
}

/* :focus-visible is the keyboard-only focus ring. Never remove it with
   outline: none - keyboard users then cannot see where they are. */
input:focus-visible,
textarea:focus-visible,
select:focus-visible,
button:focus-visible {
  outline: 2px solid var(--accent);
  outline-offset: 1px;
}

textarea {
  resize: vertical; /* dragging sideways would break the layout */
}

.check {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  font-weight: 400;
}

.hint {
  margin: 0;
  font-size: 0.85rem;
  color: var(--muted);
}

/* ---------- buttons ---------- */

button,
.button {
  padding: 0.5rem 0.9rem;
  border: 0;
  border-radius: 6px;
  background: var(--accent);
  color: #fff;
  font-weight: 600;
  text-decoration: none;
  cursor: pointer;
}

button:hover,
.button:hover {
  filter: brightness(0.93); /* one line instead of a second colour variable */
}

button.danger {
  background: var(--danger);
}

/* A submit button that has to look like a link, for the log-out form. */
.link-button {
  background: none;
  border: 0;
  padding: 0;
  color: var(--accent);
  font-weight: 400;
  text-decoration: underline;
  cursor: pointer;
}

/* ---------- table ---------- */

table {
  width: 100%;
  border-collapse: collapse; /* single thin lines instead of doubled-up borders */
  background: var(--card);
  border: 1px solid var(--line);
  border-radius: var(--radius);
  overflow: hidden; /* makes the rounded corners actually clip the rows */
}

th,
td {
  text-align: left;
  padding: 0.6rem 0.75rem;
  border-bottom: 1px solid var(--line);
  vertical-align: top;
}

th {
  font-size: 0.8rem;
  text-transform: uppercase;
  letter-spacing: 0.03em;
  color: var(--muted);
  background: #fbfcfd;
}

tbody tr:last-child td {
  border-bottom: 0;
}

tbody tr:hover {
  background: #f9fbff;
}

.right {
  text-align: right;
}

/* The two little forms in the Actions column must sit beside the Edit link. */
td form {
  display: inline;
}

/* ---------- small pieces ---------- */

.muted {
  color: var(--muted);
  font-size: 0.9rem;
}

.who {
  color: var(--muted);
  font-size: 0.9rem;
}

.preview {
  display: block;
  color: var(--muted);
  font-size: 0.85rem;
}

.pin {
  font-size: 0.7rem;
  font-weight: 700;
  color: var(--accent);
  border: 1px solid var(--accent);
  border-radius: 4px;
  padding: 0 0.25rem;
  margin-right: 0.25rem;
}

.flash {
  margin: 0 0 1rem;
  padding: 0.6rem 0.8rem;
  background: #fff8e1;
  border: 1px solid #f0d58c;
  border-radius: var(--radius);
}

.empty {
  padding: 2rem;
  text-align: center;
  color: var(--muted);
  background: var(--card);
  border: 1px dashed var(--line);
  border-radius: var(--radius);
}

/* ---------- dark mode ---------- */
/* Redefining the variables is the whole job: every rule above already uses them. */
@media (prefers-color-scheme: dark) {
  :root {
    --bg: #15181c;
    --card: #1d2126;
    --ink: #e8eaed;
    --muted: #9aa4b2;
    --line: #2d333b;
    --accent: #6ea8fe;
  }

  input,
  textarea,
  select {
    background: #14171a;
    color: var(--ink);
  }

  th {
    background: #21262c;
  }

  tbody tr:hover {
    background: #22272e;
  }

  .flash {
    background: #2a2416;
    border-color: #5c4d20;
  }
}
```

**The three ideas that do most of the work:**

| Idea | What it buys you |
|---|---|
| `--variables` on `:root` | the colours are declared once; dark mode just redeclares them |
| `display: flex` | lays things in a row with even gaps, and wraps on a phone |
| `max-width` on `main` | text stops stretching to 3000px on a wide monitor |

> ⚠️ **Never `outline: none` on a focused field.** It removes the ring that shows keyboard users where they are. If you dislike the default, replace it — `:focus-visible` above does.

---

## 5. Proving it works

This is the part worth copying into every project you build. `app.test_client()` is a fake browser built into Flask: it sends requests straight into the app and keeps cookies, so it can log in and stay logged in — no browser, no clicking.

```python
"""Proves the app works, without a browser. Run: python test_app.py

app.test_client() is a fake browser built into Flask: it sends requests straight
into the app and keeps cookies between calls, so it can log in and stay logged in.
"""

import os
import re
import tempfile

os.environ["SECRET_KEY"] = "test-key"
import app as appmod  # noqa: E402

# Point the app at a throwaway database file so tests never touch notes.db.
fd, path = tempfile.mkstemp(suffix=".db")
os.close(fd)
appmod.app.config.update(DATABASE=path, TESTING=True)

with appmod.app.app_context():
    appmod.init_db()  # the same thing "flask init-db" does

client = appmod.app.test_client()
client2 = appmod.app.test_client()  # a second, separate "browser" = a second person


def token(client, url="/login"):
    """Scrape the CSRF token out of a page, the way a real browser would use it."""
    html = client.get(url).get_data(as_text=True)
    return re.search(r'name="csrf_token" value="([^"]+)"', html).group(1)


def check(label, got, want):
    assert got == want, f"{label}: got {got!r}, wanted {want!r}"
    print(f"  ok  {label}: {got}")


print("auth")
check("anonymous / redirects to login", client.get("/").status_code, 302)
check("register page loads", client.get("/register").status_code, 200)
check(
    "short password rejected",
    b"at least 8" in client.post(
        "/register", data={"username": "zoe", "password": "short", "csrf_token": token(client, "/register")}
    ).data.lower(),
    True,
)
check(
    "register works",
    client.post(
        "/register",
        data={"username": "zoe", "password": "correct-horse", "csrf_token": token(client, "/register")},
    ).status_code,
    302,
)
check(
    "duplicate username refused",
    b"already taken" in client.post(
        "/register",
        data={"username": "zoe", "password": "correct-horse", "csrf_token": token(client, "/register")},
        follow_redirects=True,
    ).data,
    True,
)
check(
    "wrong password refused",
    b"Wrong username or password" in client.post(
        "/login", data={"username": "zoe", "password": "nope", "csrf_token": token(client)}, follow_redirects=True
    ).data,
    True,
)
check(
    "login works",
    client.post(
        "/login", data={"username": "zoe", "password": "correct-horse", "csrf_token": token(client)}
    ).status_code,
    302,
)
check("logged-in / loads", client.get("/").status_code, 200)
check("empty state shown", b"No notes yet" in client.get("/").data, True)

print("csrf")
check(
    "POST with no token is refused",
    client.post("/notes/new", data={"title": "sneaky"}).status_code,
    400,
)
check(
    "POST with a wrong token is refused",
    client.post("/notes/new", data={"title": "sneaky", "csrf_token": "made-up"}).status_code,
    400,
)

print("notes")
t = token(client, "/notes/new")
check(
    "create note",
    client.post("/notes/new", data={"title": "Buy milk", "body": "two litres", "csrf_token": t}).status_code,
    302,
)
client.post("/notes/new", data={"title": "Apple pie recipe", "body": "butter, flour", "csrf_token": t})
client.post("/notes/new", data={"title": "Pinned thing", "pinned": "on", "csrf_token": t})
page = client.get("/").get_data(as_text=True)
check("note appears in the table", "Buy milk" in page, True)
check("3 notes listed", page.count("</tr>") - 1, 3)  # minus the header row
check("pinned note sorts first", page.index("Pinned thing") < page.index("Buy milk"), True)

check("search finds one", b"Apple pie" in client.get("/?q=apple").data, True)
check("search excludes others", b"Buy milk" in client.get("/?q=apple").data, False)
check("search matches the body too", b"Buy milk" in client.get("/?q=litres").data, True)
check("no match says so", b"Nothing matches" in client.get("/?q=zzzz").data, True)
check("sort by title works", client.get("/?sort=title").status_code, 200)
check("a made-up sort does not crash", client.get("/?sort=DROP TABLE notes").status_code, 200)

check(
    "edit note",
    client.post(
        "/notes/1/edit", data={"title": "Buy oat milk", "body": "two litres", "csrf_token": t}
    ).status_code,
    302,
)
check("edit took effect", b"Buy oat milk" in client.get("/").data, True)

print("injection and escaping")
client.post("/notes/new", data={"title": "<script>alert(1)</script>", "csrf_token": t})
check("script tag is escaped, not run", b"&lt;script&gt;" in client.get("/").data, True)
client.get("/?q=' OR 1=1 --")  # a classic SQL injection attempt
check("injection in search is harmless", client.get("/?q=' OR 1=1 --").status_code, 200)
check("and matches nothing", b"Nothing matches" in client.get("/?q=' OR 1=1 --").data, True)

print("one user cannot touch another's notes")
client2.post(
    "/register",
    data={"username": "mallory", "password": "correct-horse", "csrf_token": token(client2, "/register")},
)
client2.post("/login", data={"username": "mallory", "password": "correct-horse", "csrf_token": token(client2)})
t2 = token(client2, "/notes/new")
check("mallory sees no notes", b"No notes yet" in client2.get("/").data, True)
check("cannot open zoe's note", client2.get("/notes/1/edit").status_code, 404)
check(
    "cannot edit zoe's note",
    client2.post("/notes/1/edit", data={"title": "hacked", "csrf_token": t2}).status_code,
    404,
)
check("cannot delete zoe's note", client2.post("/notes/1/delete", data={"csrf_token": t2}).status_code, 404)
check("zoe's note is untouched", b"Buy oat milk" in client.get("/").data, True)

print("delete and log out")
check("delete own note", client.post("/notes/1/delete", data={"csrf_token": t}).status_code, 302)
check("it is gone", b"Buy oat milk" in client.get("/").data, False)
check("log out", client.post("/logout", data={"csrf_token": t}).status_code, 302)
check("and / is protected again", client.get("/").status_code, 302)

os.unlink(path)
print("\nALL CHECKS PASSED")
```

### The real output

```
auth
  ok  anonymous / redirects to login: 302
  ok  register page loads: 200
  ok  short password rejected: True
  ok  register works: 302
  ok  duplicate username refused: True
  ok  wrong password refused: True
  ok  login works: 302
  ok  logged-in / loads: 200
  ok  empty state shown: True
csrf
  ok  POST with no token is refused: 400
  ok  POST with a wrong token is refused: 400
notes
  ok  create note: 302
  ok  note appears in the table: True
  ok  3 notes listed: 3
  ok  pinned note sorts first: True
  ok  search finds one: True
  ok  search excludes others: False
  ok  search matches the body too: True
  ok  no match says so: True
  ok  sort by title works: 200
  ok  a made-up sort does not crash: 200
  ok  edit note: 302
  ok  edit took effect: True
injection and escaping
  ok  script tag is escaped, not run: True
  ok  injection in search is harmless: 200
  ok  and matches nothing: True
one user cannot touch another's notes
  ok  mallory sees no notes: True
  ok  cannot open zoe's note: 404
  ok  cannot edit zoe's note: 404
  ok  cannot delete zoe's note: 404
  ok  zoe's note is untouched: True
delete and log out
  ok  delete own note: 302
  ok  it is gone: False
  ok  log out: 302
  ok  and / is protected again: 302

ALL CHECKS PASSED
```

**What those last four groups actually prove:**

| Group | Why it matters |
|---|---|
| **csrf** | another site's forged POST is rejected |
| **injection and escaping** | `' OR 1=1 --` in the search box does nothing; `<script>` is displayed, not run |
| **one user cannot touch another's** | mallory gets 404 on zoe's note — by id, in three different ways |
| **delete and log out** | the session really ends, and `/` is protected again |

> **Run the tests after every change.** They take about a second, and they catch the two bugs you cannot see by clicking around: a missing ownership check, and a missing CSRF token.

---

## Where it breaks, and what to do

| Message | Cause | Fix |
|---|---|---|
| `no such table: notes` | never ran init-db, or the DB file is elsewhere | `flask --app app init-db` |
| `405 Method Not Allowed` | form POSTs to a route that only allows GET | add `methods=["GET", "POST"]` |
| `400 Bad Request` on a form | missing `csrf_token` hidden input | add it to that form |
| `TemplateNotFound: index.html` | file isn't in `templates/`, or it's misspelled | names are case-sensitive |
| `jinja2.exceptions.UndefinedError: 'note' is undefined` | the view didn't pass it | add it to `render_template(...)` |
| `RuntimeError: ... SECRET_KEY` | `session` used with no key set | set `app.config["SECRET_KEY"]` |
| Changes vanish after a refresh | missing `db.commit()` | commit after every write |
| `sqlite3.ProgrammingError: parameters are of unsupported type` | passed `(7)` instead of `(7,)` | the comma makes it a tuple |
| Page shows `&lt;b&gt;hi&lt;/b&gt;` | you wanted HTML through | only ever on text **you** wrote: `{{ x|safe }}` |
| `database is locked` | two writes at once | fine here; at scale → [[PostgreSQL]] |

---

## Make it yours

Each of these is a small change to code that's already here:

1. **Tags on notes** — a `tags` table plus `note_tags`, joining three tables ([[Flask SQLite - book tracker]] shows joins)
2. **"Change my password"** — a form that checks the old one with `check_password_hash`, then saves a new hash
3. **Pagination** — the notes table with 500 rows in it ([[Flask SQLite - book tracker]] has it)
4. **A JSON endpoint** — `/api/notes` returning `jsonify([dict(r) for r in rows])`
5. **Deploy it** — [[Flask - deploying it]]: a real server, a real `SECRET_KEY`, debug off

---

## Related

[[Flask SQLite stack]] · [[Flask SQLite - showing data on the page]] · [[Flask SQLite - book tracker]] · [[Flask]] · [[Flask reference]] · [[Flask - forms and user input]] · [[Flask - user accounts and login]] · [[Flask - templates with Jinja]] · [[Flask - deploying it]] · [[HTML]] · [[Forms (HTML)]] · [[CSS]] · [[SQL]] · [[Python]] · [[pytest]] · [[Security in practice]]
