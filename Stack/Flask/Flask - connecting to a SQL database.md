---
tags: [flask, sql, database, sqlalchemy, backend, guide]
---

# Flask — connecting to a SQL database

**Guide 4 of 8.** Saving data so it's still there tomorrow. What a database is, connecting Flask to one, defining tables in Python, and creating, reading, updating and deleting rows.

Hub: [[Flask]] · Previous: [[Flask - styling with Bulma]] · Next: [[Flask - forms and user input]]

---

## Why you need a database

Right now, if your app stores tasks in a Python list, they vanish every time the server restarts:

```python
tasks = []          # lives in memory - gone when you press Ctrl+C
```

**A database is a program whose whole job is storing data safely on disk**, and finding it again quickly. Your Flask app talks to it.

A **SQL** database stores data in **tables** — like spreadsheets with strict rules:

```
tasks table
┌────┬──────────────────┬───────┬─────────┐
│ id │ title            │ done  │ user_id │   <- columns: every row has the same ones
├────┼──────────────────┼───────┼─────────┤
│  1 │ Buy milk         │ false │    1    │   <- a row: one task
│  2 │ Finish homework  │ true  │    1    │
│  3 │ Call grandma     │ false │    2    │
└────┴──────────────────┴───────┴─────────┘
```

- **Every row has an `id`** — a unique number, so you can point at exactly one task. This is the **primary key**.
- **`user_id` points at a row in another table** (the users table). That's a **foreign key** — it's how tables link together.

You talk to it in a language called **SQL**: `SELECT * FROM tasks WHERE done = false`. See [[SQL fundamentals]].

---

## The piece in the middle: SQLAlchemy

Writing SQL strings by hand inside Python gets messy, and one careless string can open a security hole. **SQLAlchemy** lets you describe tables as Python classes and query them with Python:

```python
# instead of this SQL string...
"SELECT * FROM tasks WHERE done = false ORDER BY id"

# ...you write Python
db.session.scalars(db.select(Task).filter_by(done=False).order_by(Task.id))
```

It's called an **ORM** — an *Object-Relational Mapper*. It **maps** Python **objects** onto database rows. A `Task` object in Python *is* a row in the tasks table.

**Flask-SQLAlchemy** is the small add-on that plugs SQLAlchemy into Flask. Full SQLAlchemy detail: [[SQLAlchemy]].

| Which database? | When |
|---|---|
| **SQLite** | **Start here.** A single file, no server to install, built into Python |
| **PostgreSQL** | When you deploy, or have several users at once. The best general-purpose choice |
| MySQL / MariaDB | If your host provides it |
| Azure SQL / SQL Server | In a Microsoft shop — [[Azure SQL Database]] |

> **The best part: your Python code is identical for all of them.** You change one connection string to move from SQLite on your laptop to Postgres in production.

---

## Installing

```bash
uv add flask-sqlalchemy flask-migrate
```

or with pip inside an activated venv: `pip install flask-sqlalchemy flask-migrate`. Same on Windows and WSL. SQLite needs nothing extra — it's built into Python.

---

## Connecting

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):        # the parent class all your tables inherit from
    pass

db = SQLAlchemy(model_class=Base)   # the database object - you'll use `db` everywhere

app = Flask(__name__)
app.config["SQLALCHEMY_DATABASE_URI"] = "sqlite:///tasks.db"   # WHERE the database is
db.init_app(app)                    # connect the two
```

### The connection string

It tells SQLAlchemy **which kind** of database, **where** it is, and **how to log in**:

```
dialect+driver://username:password@host:port/database_name
```

| Database | Connection string |
|---|---|
| SQLite (a file) | `sqlite:///tasks.db` |
| PostgreSQL | `postgresql+psycopg://huz:secret@localhost:5432/tasks` |
| MySQL | `mysql+pymysql://huz:secret@localhost:3306/tasks` |
| Azure SQL | `mssql+pyodbc://user:pw@server.database.windows.net/db?driver=ODBC+Driver+18+for+SQL+Server` |

> ⚠️ **Where does `tasks.db` end up?** Flask-SQLAlchemy puts relative SQLite paths inside an **`instance/`** folder next to your app — so the file is `instance/tasks.db`, not in your project root. That confuses everyone who goes looking for it.

> ⚠️ **Never write the password in your code.** Load it from an environment variable (see [[Flask - project structure and blueprints]]):
> ```python
> import os
> app.config["SQLALCHEMY_DATABASE_URI"] = os.environ.get("DATABASE_URL", "sqlite:///tasks.db")
> ```

---

## Defining a table — a model

**A model is a Python class that describes one table.** Each attribute is a column.

```python
from datetime import datetime, timezone
from sqlalchemy import String, Text, ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship

class Task(db.Model):
    __tablename__ = "tasks"                                   # the table's name in the database

    id: Mapped[int] = mapped_column(primary_key=True)         # unique number, filled in automatically
    title: Mapped[str] = mapped_column(String(200))           # text, at most 200 characters
    notes: Mapped[str | None] = mapped_column(Text)           # long text; `| None` means it may be empty
    done: Mapped[bool] = mapped_column(default=False)         # starts as False
    created_at: Mapped[datetime] = mapped_column(
        default=lambda: datetime.now(timezone.utc))           # set automatically when the row is made

    def __repr__(self):                                       # what it looks like when printed
        return f"<Task {self.id}: {self.title}>"
```

**Reading that line by line:**

| Code | Means |
|---|---|
| `Mapped[int]` | This column holds a whole number |
| `Mapped[str \| None]` | This column holds text, **or nothing** (nullable) |
| `mapped_column(primary_key=True)` | The unique ID for each row |
| `String(200)` | Short text with a maximum length |
| `Text` | Long text with no limit |
| `default=False` | The value used if you don't give one |

**Column types you'll use:**

| Python type | SQL type | For |
|---|---|---|
| `int` | INTEGER | Counts, IDs |
| `str` with `String(n)` | VARCHAR(n) | Names, titles, emails |
| `str` with `Text` | TEXT | Long notes |
| `bool` | BOOLEAN | Yes/no |
| `float` | FLOAT | Measurements |
| `Decimal` with `Numeric(10, 2)` | NUMERIC | **Money — never `float`** |
| `datetime` / `date` | DATETIME / DATE | Timestamps |

> ⚠️ **Never store money as `float`.** `0.1 + 0.2` is `0.30000000000000004` in floating point, so totals drift by pennies. Use `Numeric(10, 2)` and Python's `Decimal` ([[Data modeling]]).

**Useful column options:**

```python
from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column

email: Mapped[str] = mapped_column(String(120), unique=True)   # no two rows may share it
code: Mapped[str] = mapped_column(String(10), index=True)      # make lookups on it fast
qty: Mapped[int] = mapped_column(default=1)
```

> **Why set `__tablename__` yourself?** Otherwise Flask-SQLAlchemy names the table after the class — `User` becomes `user`, and **`user` is a reserved word in PostgreSQL**. It works on SQLite and then breaks the day you switch. Plural names (`users`, `tasks`) avoid it entirely.

---

## Creating the tables

**The quick way** — fine while you're learning:

```python
with app.app_context():      # Flask needs to know which app you mean
    db.create_all()          # create every table that doesn't exist yet
```

> ⚠️ **`create_all()` only creates tables that are missing. It never changes an existing one.** Add a column to your model and it will *not* appear in the database — you'll get *no such column* errors. That's what migrations are for.

### Migrations — the proper way

A **migration** is a small script that changes the database's structure: *"add a `due_date` column to tasks"*. Flask-Migrate writes them for you by comparing your models to the database.

```python
from flask_migrate import Migrate
migrate = Migrate(app, db)
```

```bash
flask --app app db init                          # ONCE per project - creates a migrations/ folder
flask --app app db migrate -m "create tasks"     # compare models to DB, write a migration script
flask --app app db upgrade                       # APPLY it - actually change the database
```

**Every time you change a model afterwards:**

```bash
flask --app app db migrate -m "add due date to tasks"    # 1. generate
# 2. OPEN the new file in migrations/versions/ and read it
flask --app app db upgrade                               # 3. apply
```

| Command | Does |
|---|---|
| `db init` | Set up the `migrations/` folder (once) |
| `db migrate -m "..."` | Detect model changes, write a script |
| `db upgrade` | Apply pending migrations |
| `db downgrade` | Undo the last migration |
| `db current` | Which migration the database is on |
| `db history` | List all migrations |

> **Always read the generated migration before running `upgrade`.** Auto-detection misses some things — renaming a column looks like *"drop the old column, add a new one"*, which **deletes that column's data**.

> **Commit the `migrations/` folder to git.** It's part of your code — it's how anyone else (and your production server) builds an identical database.

---

## CRUD — the four things you do with data

**C**reate, **R**ead, **U**pdate, **D**elete. Almost every web app is just these four, over and over.

### The session — how changes get saved

Think of `db.session` as a **shopping basket**. You put changes in, then **`commit()`** checks out and saves them all at once.

```python
db.session.add(task)        # put it in the basket
db.session.commit()         # SAVE everything in the basket to the database
db.session.rollback()       # empty the basket without saving - after an error
```

> ⚠️ **Nothing is saved until you `commit()`.** Forgetting it is the number one "my data disappeared" bug — the page looks fine, then the data is gone on refresh.

### Create — add a row

```python
task = Task(title="Buy milk")         # make a Python object (not saved yet)
db.session.add(task)                  # stage it
db.session.commit()                   # save it - the database assigns an id
print(task.id)                        # now it has one, e.g. 1

db.session.add_all([Task(title="A"), Task(title="B")])   # several at once
db.session.commit()
```

### Read — get rows back

```python
# ONE row by its id
task = db.session.get(Task, 1)                  # returns the Task, or None if not found
task = db.get_or_404(Task, 1)                   # in a route: returns the Task, or shows a 404 page

# ALL rows
tasks = db.session.scalars(db.select(Task)).all()

# rows that MATCH something
open_tasks = db.session.scalars(db.select(Task).filter_by(done=False)).all()
first = db.session.scalar(db.select(Task).filter_by(title="Buy milk"))   # one, or None

# sorted
newest = db.session.scalars(db.select(Task).order_by(Task.created_at.desc())).all()

# how many
n = db.session.scalar(db.select(db.func.count()).select_from(Task))
```

**Reading that pattern:** `db.select(Task)` builds the question, `db.session.scalars(...)` asks the database, `.all()` gives you a list.

**More ways to filter:**

```python
db.select(Task).filter_by(done=False)                    # simple equals
db.select(Task).where(Task.done == False)                # same, more flexible
db.select(Task).where(Task.title.contains("milk"))       # text contains
db.select(Task).where(Task.title.ilike("%milk%"))        # case-insensitive
db.select(Task).where(Task.id.in_([1, 2, 3]))            # one of these
db.select(Task).where(Task.notes.is_(None))              # empty
db.select(Task).where(Task.done == False, Task.user_id == 1)   # AND - both must match
db.select(Task).order_by(Task.title).limit(10)           # first 10, alphabetical
```

> **You'll see an older style in tutorials:** `Task.query.all()`, `Task.query.filter_by(done=False).first()`, `Task.query.get(1)`. It still works but is marked legacy. The `db.select(...)` style above is the current way — and it's the same as plain SQLAlchemy, so it transfers.

### Update — change a row

```python
task = db.get_or_404(Task, 1)     # 1. get it
task.title = "Buy oat milk"       # 2. change it - just set the attribute
task.done = True
db.session.commit()               # 3. save. No add() needed - it's already tracked
```

### Delete — remove a row

```python
task = db.get_or_404(Task, 1)
db.session.delete(task)
db.session.commit()
```

> ⚠️ **Deleting has no undo.** For anything important, consider a "soft delete" — a `deleted_at` column you set instead of removing the row, then filter those out when reading.

---

## Using it in Flask routes

```python
from flask import render_template, request, redirect, url_for, flash

@app.route("/")
def index():
    tasks = db.session.scalars(db.select(Task).order_by(Task.created_at.desc())).all()
    return render_template("index.html", tasks=tasks)      # hand the list to the template

@app.post("/tasks")                                         # @app.post = route that only accepts POST
def create_task():
    title = request.form.get("title", "").strip()
    if not title:
        flash("Title can't be empty.", "is-danger")
        return redirect(url_for("index"))
    db.session.add(Task(title=title))
    db.session.commit()
    flash("Task added.", "is-success")
    return redirect(url_for("index"))                       # redirect after POST - see guide 5

@app.post("/tasks/<int:task_id>/toggle")
def toggle_task(task_id):
    task = db.get_or_404(Task, task_id)
    task.done = not task.done                               # flip it
    db.session.commit()
    return redirect(url_for("index"))

@app.post("/tasks/<int:task_id>/delete")
def delete_task(task_id):
    task = db.get_or_404(Task, task_id)
    db.session.delete(task)
    db.session.commit()
    return redirect(url_for("index"))
```

> **Deleting and changing data always goes through `POST`, never a plain link.** A `GET` link can be triggered by a browser prefetching pages, a search-engine crawler, or someone embedding it in an image on another site. `POST` from a form can't be.

---

## Relationships — linking tables together

A user **has many** tasks; each task **belongs to** one user.

```python
from sqlalchemy import ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column, relationship

class User(db.Model):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(120), unique=True)

    tasks: Mapped[list["Task"]] = relationship(            # a Python list of this user's tasks
        back_populates="owner",
        cascade="all, delete-orphan")                       # delete the user -> delete their tasks

class Task(db.Model):
    __tablename__ = "tasks"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))   # the actual link, stored in the DB
    owner: Mapped["User"] = relationship(back_populates="tasks")   # the Python shortcut to the user
```

**Using it:**

```python
user = db.session.get(User, 1)
for task in user.tasks:                 # every task belonging to this user
    print(task.title)

task = db.session.get(Task, 5)
print(task.owner.email)                 # the user who owns it

db.session.add(Task(title="New", user_id=user.id))    # attach a task to a user
db.session.commit()
```

| Piece | What it is |
|---|---|
| `ForeignKey("users.id")` | The real column in the database — `user_id` stores the user's id |
| `relationship(...)` | A Python convenience, **not** a column — lets you write `user.tasks` |
| `back_populates` | Links the two sides, so `task.owner` and `user.tasks` stay in sync |
| `cascade="all, delete-orphan"` | Deleting a user deletes their tasks too |

### ⚠️ The N+1 problem

```python
tasks = db.session.scalars(db.select(Task)).all()      # 1 query
for t in tasks:
    print(t.owner.email)                               # +1 query PER TASK -> 101 queries for 100 tasks
```

Fine with 10 rows, painfully slow with 10,000. Load the related rows in the same query:

```python
from sqlalchemy.orm import selectinload
tasks = db.session.scalars(db.select(Task).options(selectinload(Task.owner))).all()   # 2 queries total
```

---

## Only showing a user their own data

```python
from flask import abort, render_template
from flask_login import current_user

@app.route("/tasks/<int:task_id>")
@login_required
def show_task(task_id):
    task = db.get_or_404(Task, task_id)
    if task.user_id != current_user.id:      # it exists, but it isn't YOURS
        abort(404)                           # 404, not 403 - don't confirm it exists
    return render_template("task.html", task=task)
```

> ⚠️ **This check is the most commonly forgotten line in web development.** Without it, a logged-in user changes `/tasks/41` to `/tasks/42` in the address bar and reads someone else's data. It's called **IDOR** — see [[Security in practice]] and [[Flask - user accounts and login]].

---

## Pagination — one page at a time

```python
from flask import render_template, request

@app.route("/")
def index():
    page_num = request.args.get("page", 1, type=int)             # ?page=2 in the URL
    page = db.paginate(db.select(Task).order_by(Task.id.desc()),
                       page=page_num, per_page=20)               # 20 per page
    return render_template("index.html", page=page)
```

```html
{% for task in page.items %}<li>{{ task.title }}</li>{% endfor %}
```

`page` gives you `.items`, `.has_next`, `.has_prev`, `.next_num`, `.prev_num`, `.pages` and `.iter_pages()` — the Bulma pagination bar that uses them is in [[Flask - styling with Bulma]].

---

## Raw SQL, when you need it

Sometimes a query is simpler in plain SQL:

```python
from flask_login import current_user
from sqlalchemy import text

rows = db.session.execute(
    text("SELECT title, created_at FROM tasks WHERE user_id = :uid AND done = :done"),
    {"uid": current_user.id, "done": False},     # values passed SEPARATELY
).all()

for row in rows:
    print(row.title, row.created_at)
```

> ⚠️ **Always use `:name` placeholders. Never build SQL with f-strings.**
> ```python
> from sqlalchemy import text
>
> text(f"SELECT * FROM tasks WHERE title = '{title}'")    # SQL INJECTION - never
> text("SELECT * FROM tasks WHERE title = :t"), {"t": title}   # safe
> ```
> If `title` is `' OR '1'='1`, the f-string version returns every row in the table. The placeholder version treats it as ordinary text ([[Security in practice]]).

---

## Moving from SQLite to PostgreSQL

1. Install the driver: `uv add "psycopg[binary]"`
2. Run Postgres — the easiest way is Docker:
   ```bash
   docker run -d --name pg -e POSTGRES_PASSWORD=secret -e POSTGRES_DB=tasks -p 5432:5432 postgres:16
   ```
3. Change the connection string:
   ```bash
   DATABASE_URL=postgresql+psycopg://postgres:secret@localhost:5432/tasks
   ```
4. Build the tables: `flask --app app db upgrade`

**That's the whole move** — no model or route changes. See [[PostgreSQL reference]] for the database itself.

> **Use `psycopg[binary]`, not plain `psycopg`**, especially on Windows — the plain version tries to compile against a Postgres installation you probably don't have.

---

## Looking inside the database

```bash
flask --app app shell                   # a Python prompt with your app loaded
```

```python
>>> from app import db, Task
>>> db.session.scalars(db.select(Task)).all()
[<Task 1: Buy milk>, <Task 2: Finish homework>]
```

```bash
sqlite3 instance/tasks.db               # SQLite's own tool (sudo apt install sqlite3 on WSL)
sqlite> .tables                         # list tables
sqlite> .schema tasks                   # show a table's columns
sqlite> SELECT * FROM tasks;            # plain SQL
sqlite> .quit
```

> **On Windows**, *DB Browser for SQLite* (free, `winget install DBBrowserForSQLite.DBBrowserForSQLite`) lets you open the `.db` file and click around the tables.

---

## Common mistakes

| Mistake | What you'll see | Fix |
|---|---|---|
| Forgot `db.session.commit()` | Data vanishes on refresh | Always commit |
| Added a column, didn't migrate | `no such column` / `UndefinedColumn` | `db migrate` then `db upgrade` |
| Used `create_all()` after changing a model | Same | It never alters tables — use migrations |
| Can't find `tasks.db` | It's not in the project folder | Look in `instance/` |
| Error, then every query fails | `PendingRollbackError` | `db.session.rollback()` after an exception |
| Table named `user` on Postgres | Confusing SQL errors | `__tablename__ = "users"` |
| Loop that touches `task.owner` | Very slow pages | `selectinload(...)` |
| f-string in raw SQL | **SQL injection** | `:name` placeholders |
| No ownership check | **Users see each other's data** | Compare `user_id` to `current_user.id` |
| Money as `float` | Totals off by pennies | `Numeric(10, 2)` |
| Delete via a GET link | Data deleted by crawlers/prefetch | `POST` forms only |
| `Working outside of application context` | Error in a script | Wrap in `with app.app_context():` |

## Related

[[Flask]] · [[Flask - forms and user input]] · [[Flask - user accounts and login]] · [[Flask reference]] · [[SQLAlchemy]] · [[SQL fundamentals]] · [[PostgreSQL reference]] · [[Data modeling]] · [[Security in practice]] · [[Docker deep dive]]
