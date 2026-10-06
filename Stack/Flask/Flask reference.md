---
tags: [flask, reference, cheatsheet, python, web]
---

# Flask reference

**Every Flask thing on one page.** The guides explain; this is where you look things up.

Hub: [[Flask]] · Build it: [[Flask - full project walkthrough]]

---

## I want to…

| I want to… | Go to |
|---|---|
| **Install Flask and run a first app** | [[#Install and run]] |
| Add a page | [[#Routes]] |
| Put a number or name in the URL | [[#URL variables]] |
| Accept a form submission | [[#HTTP methods]] |
| **Read what the user typed** | [[#The request object]] |
| Send the user to another page | [[#Redirects and url_for]] |
| Return JSON | [[#Responses]] |
| Show a 404 or other error | [[#Errors]] |
| **Render an HTML page** | [[#Templates]] |
| Loop, if, or share a layout in HTML | [[#Jinja cheat sheet]] |
| **Style it with Bulma** | [[#Bulma cheat sheet]] |
| Show "Task saved!" once | [[#Flash messages]] |
| Remember something between pages | [[#Sessions and cookies]] |
| **Connect to a database** | [[#Database setup]] |
| Define a table | [[#Models]] |
| **Create, read, update, delete rows** | [[#CRUD]] |
| Change the table structure | [[#Migrations]] |
| Build a form with validation | [[#Flask-WTF forms]] |
| **Add login** | [[#Flask-Login]] |
| Stop users seeing each other's data | [[#Ownership checks]] |
| Split the app into files | [[#Blueprints and the app factory]] |
| Store secrets and settings | [[#Config]] |
| Run code before every request | [[#Request hooks]] |
| Write tests | [[#Testing]] |
| **Deploy it** | [[#Production]] |
| Every command | [[#Command-line cheat sheet]] |
| Fix an error | [[#Error messages and what they mean]] |

---

## Install and run

```bash
uv add flask                                  # or: pip install flask (in a venv)
```

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello"
```

```bash
flask --app app run --debug                   # http://127.0.0.1:5000
```

| Package | `uv add ...` | For |
|---|---|---|
| Flask | `flask` | The framework |
| Flask-SQLAlchemy | `flask-sqlalchemy` | Database |
| Flask-Migrate | `flask-migrate` | Schema changes |
| Flask-Login | `flask-login` | Logins |
| Flask-WTF | `flask-wtf email-validator` | Forms + CSRF |
| python-dotenv | `python-dotenv` | `.env` files |
| Gunicorn / Waitress | `gunicorn` / `waitress` | Production server (Linux / Windows) |
| Postgres driver | `"psycopg[binary]"` | PostgreSQL |

Guide: [[Flask - your first app]]

---

## Routes

```python
@app.route("/")                               # GET only, by default
@app.route("/about")
@app.route("/tasks", methods=["GET", "POST"])  # GET and POST
@app.get("/items")                            # shortcut: GET only
@app.post("/items")                           # shortcut: POST only
@app.put("/items/<int:id>")
@app.patch("/items/<int:id>")
@app.delete("/items/<int:id>")
```

> ⚠️ **Every route function needs a unique name** — two functions called `index` give *"View function mapping is overwriting an existing endpoint"*.

### URL variables

```python
@app.route("/user/<name>")                    # text, no slashes
@app.route("/task/<int:task_id>")             # whole number - non-numbers become 404
@app.route("/price/<float:value>")
@app.route("/files/<path:filepath>")          # text including slashes
```

### HTTP methods

| Method | Use | Browser sends it when |
|---|---|---|
| GET | Read | Visiting a URL, clicking a link |
| POST | Create / change / delete | Submitting a form |
| PUT / PATCH | Update | JavaScript |
| DELETE | Remove | JavaScript |

```python
from flask import request

@app.route("/contact", methods=["GET", "POST"])
def contact():
    if request.method == "POST":
        ...
```

---

## The request object

```python
from flask import request
```

| You want | Code |
|---|---|
| A form field | `request.form.get("title", "")` |
| A form field, crash if missing | `request.form["title"]` |
| Several checkboxes | `request.form.getlist("tags")` |
| URL query `?page=2` | `request.args.get("page", 1, type=int)` |
| JSON body | `request.get_json(silent=True)` |
| An uploaded file | `request.files.get("photo")` |
| A header | `request.headers.get("X-API-Key")` |
| A cookie | `request.cookies.get("theme")` |
| The method | `request.method` |
| The path | `request.path` |
| The full URL | `request.url` |
| Which route handled it | `request.endpoint` |
| The visitor's IP | `request.remote_addr` |

> ⚠️ **Everything from a form or URL is a string.** `"5"`, not `5`. Use `type=int` with `request.args.get`, or convert and handle `ValueError`.

Guide: [[Flask - forms and user input]]

---

## Redirects and url_for

```python
from flask import redirect, url_for

return redirect(url_for("home"))                          # to a route, by FUNCTION name
return redirect(url_for("show_task", task_id=7))          # with a URL variable
return redirect(url_for("search", q="milk"))              # extra args -> ?q=milk
return redirect(url_for("auth.login"))                    # in a blueprint: "blueprint.function"
url_for("static", filename="style.css")                   # a static file
url_for("home", _external=True)                           # full URL with https://...
```

> **Always redirect after a successful POST** — otherwise refreshing the page submits the form again.

---

## Responses

```python
return "text"                                  # 200, text/html
return "Created", 201                          # custom status
return {"id": 1, "title": "Milk"}              # a dict -> JSON automatically
return jsonify([1, 2, 3])                      # a list needs jsonify
return {"error": "not found"}, 404             # JSON + status
return "", 204                                 # no content
return render_template("page.html", x=1)       # a template
return redirect(url_for("home"))               # a redirect
return send_file("report.pdf", as_attachment=True)   # download a file

from flask import make_response, jsonify, redirect, render_template, send_file, url_for
resp = make_response(render_template("page.html"))
resp.headers["Cache-Control"] = "no-store"
resp.set_cookie("theme", "dark", max_age=60*60*24*30, httponly=True, samesite="Lax")
return resp
```

Guide: [[Flask - building a JSON API]]

---

## Errors

```python
from flask import abort, render_template

abort(404)                                     # stop and return a 404
abort(403)
abort(400, description="Title is required")

@app.errorhandler(404)                         # a custom 404 page
def not_found(e):
    return render_template("404.html"), 404

@app.errorhandler(500)
def server_error(e):
    return render_template("500.html"), 500
```

| Code | Means |
|---|---|
| 200 | OK |
| 201 | Created |
| 204 | No content |
| 302 | Redirect |
| 400 | Bad input |
| 401 | Not logged in |
| 403 | Logged in, not allowed |
| 404 | Not found |
| 405 | Wrong HTTP method |
| 409 | Conflict (duplicate) |
| 413 | Upload too large |
| 500 | Your code crashed |

---

## Templates

```
app.py
templates/        <- must be named exactly this
static/           <- CSS, JS, images
```

```python
from flask import render_template
return render_template("tasks/index.html", tasks=tasks, title="My tasks")
```

### Jinja cheat sheet

```html
{{ value }}                            <!-- print (auto-escaped - safe) -->
{{ user.email }} {{ d["key"] }}        <!-- attribute / key -->
{# a comment #}

{% if x %}...{% elif y %}...{% else %}...{% endif %}
{% for t in tasks %}...{% else %}empty...{% endfor %}
{{ loop.index }} {{ loop.first }} {{ loop.last }}

{% extends "base.html" %}              <!-- line 1 of a child template -->
{% block content %}...{% endblock %}   <!-- fill a gap in the base -->
{{ super() }}                          <!-- keep the base's block content -->
{% include "_flash.html" %}            <!-- paste in a partial -->

{% macro field(f) %}...{% endmacro %}  <!-- define a reusable bit -->
{% from "_macros.html" import field %}  <!-- import it -->

{% set total = tasks | length %}
{{ 'a' ~ 'b' }}                        <!-- join strings: "ab" -->
{{ 'done' if t.done else 'open' }}     <!-- inline if -->
```

| Filter | Does |
|---|---|
| `\| length` | Count |
| `\| upper` `\| lower` `\| title` | Case |
| `\| default("x")` | Fallback |
| `\| round(2)` | Round |
| `\| truncate(50)` | Shorten |
| `\| join(", ")` | List to text |
| `\| selectattr("done")` | Keep items where `.done` is true |
| `\| safe` | ⚠️ Turn off escaping — never on user input |

**Available in every template:** `request`, `session`, `g`, `config`, `url_for()`, `get_flashed_messages()`, `csrf_token()` (with CSRFProtect), `current_user` (with Flask-Login).

Guide: [[Flask - templates with Jinja]]

---

## Bulma cheat sheet

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bulma@1.0.2/css/bulma.min.css">
<html lang="en" data-theme="light">     <!-- stop Bulma 1.x following dark mode -->
```

| Want | Classes |
|---|---|
| Page wrapper | `section` > `container` |
| Side by side | `columns` > `column` (`is-half`, `is-one-third`, `is-4`) |
| Centre a box | `columns is-centered` > `column is-5` |
| Grid from a loop | `columns is-multiline` |
| Heading | `title` / `subtitle` (`is-1`…`is-6`) |
| Button | `button is-primary` · `is-danger` · `is-light` · `is-outlined` · `is-small` · `is-fullwidth` |
| Row of buttons | `buttons` |
| Form field | `field` > `label` + `control` > `input` / `textarea`, then `help` |
| Dropdown | `div.select` > `select` |
| Error on a field | `input is-danger` + `p.help.is-danger` |
| Input + button joined | `field has-addons` > `control is-expanded` |
| Panel | `box` · `card` (`card-header`, `card-content`, `card-footer`) |
| Table | `table is-fullwidth is-striped is-hoverable`, wrapped in `table-container` |
| Label | `tag is-success` |
| Message box | `notification is-success` · `message` |
| Title left, button right | `level` > `level-left` + `level-right` |
| Big banner | `hero is-primary` > `hero-body` |
| Spacing | `mt-4` `mb-2` `p-5` `px-3` (0–6) |
| Text | `has-text-centered` `has-text-weight-bold` `has-text-grey` `is-size-7` |

**Colours:** `is-primary` · `is-link` · `is-info` · `is-success` · `is-warning` · `is-danger` · `is-dark` · `is-light`

> ⚠️ **Bulma has no JavaScript.** The mobile menu, modals, dropdowns and notification close buttons need a few lines of your own — in [[Flask - styling with Bulma]].

---

## Flash messages

```python
from flask import flash
flash("Task saved.", "is-success")             # category = a Bulma colour class
```

```html
{% with messages = get_flashed_messages(with_categories=true) %}
  {% for category, message in messages %}
    <div class="notification {{ category }}">{{ message }}</div>
  {% endfor %}
{% endwith %}
```

Needs a `SECRET_KEY`.

---

## Sessions and cookies

```python
from flask import session

session["theme"] = "dark"                      # store
theme = session.get("theme", "light")          # read
session.pop("theme", None)                     # remove
session.clear()                                # remove everything
```

> ⚠️ **The session is a signed cookie — signed, not encrypted.** The user can read it but not change it. Never put secrets in it. Needs a `SECRET_KEY`.

---

## Database setup

```python
from flask_sqlalchemy import SQLAlchemy
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

db = SQLAlchemy(model_class=Base)
app.config["SQLALCHEMY_DATABASE_URI"] = "sqlite:///tasks.db"    # -> instance/tasks.db
db.init_app(app)
```

| Database | URI |
|---|---|
| SQLite | `sqlite:///tasks.db` |
| PostgreSQL | `postgresql+psycopg://user:pw@localhost:5432/db` |
| MySQL | `mysql+pymysql://user:pw@localhost:3306/db` |
| In-memory (tests) | `sqlite:///:memory:` |

Guide: [[Flask - connecting to a SQL database]]

## Models

```python
from decimal import Decimal
from sqlalchemy import String, Text, Numeric, ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship

class Task(db.Model):
    __tablename__ = "tasks"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    notes: Mapped[str | None] = mapped_column(Text)            # nullable
    price: Mapped[Decimal] = mapped_column(Numeric(10, 2))     # money
    done: Mapped[bool] = mapped_column(default=False)
    email: Mapped[str] = mapped_column(String(120), unique=True, index=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    owner: Mapped["User"] = relationship(back_populates="tasks")
```

## CRUD

```python
# CREATE
db.session.add(Task(title="Milk", user_id=1)); db.session.commit()

# READ
db.session.get(Task, 1)                                              # by id, or None
db.get_or_404(Task, 1)                                               # by id, or 404
db.session.scalars(db.select(Task)).all()                            # all
db.session.scalars(db.select(Task).filter_by(done=False)).all()      # filtered
db.session.scalar(db.select(Task).filter_by(title="Milk"))           # one, or None
db.session.scalars(db.select(Task).order_by(Task.id.desc()).limit(10)).all()
db.session.scalar(db.select(db.func.count()).select_from(Task))      # count
db.paginate(db.select(Task), page=1, per_page=20)                    # a page

# UPDATE
task = db.get_or_404(Task, 1); task.title = "Oat milk"; db.session.commit()

# DELETE
db.session.delete(task); db.session.commit()

# after an error
db.session.rollback()
```

| Filter | Code |
|---|---|
| Equals | `.filter_by(done=False)` or `.where(Task.done == False)` |
| Contains | `.where(Task.title.contains("milk"))` |
| Case-insensitive | `.where(Task.title.ilike("%milk%"))` |
| In a list | `.where(Task.id.in_([1, 2]))` |
| Is empty | `.where(Task.notes.is_(None))` |
| AND | `.where(a, b)` |
| OR | `.where(db.or_(a, b))` |
| Load related rows (avoid N+1) | `.options(selectinload(Task.owner))` |

> ⚠️ **Raw SQL: `text("... WHERE id = :id"), {"id": x}` — never f-strings.**

## Migrations

```bash
flask --app app db init                        # once
flask --app app db migrate -m "describe it"    # after every model change
flask --app app db upgrade                     # apply
flask --app app db downgrade                   # undo one
flask --app app db current                     # where am I
```

> **Read each generated migration before `upgrade`.** A renamed column is detected as drop + add — which deletes the data.

---

## Flask-WTF forms

```python
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, TextAreaField, IntegerField, SelectField, BooleanField, DateField, SubmitField
from wtforms.validators import DataRequired, Optional, Length, Email, EqualTo, NumberRange

class TaskForm(FlaskForm):
    title = StringField("Title", validators=[DataRequired(), Length(max=200)])
    submit = SubmitField("Save")
```

```python
form = TaskForm()                              # new
form = TaskForm(obj=task)                      # pre-filled for editing
if form.validate_on_submit():                  # POST + CSRF + validators all OK
    form.title.data                            # the cleaned value
    form.populate_obj(task)                    # copy all fields onto the object
```

```html
<form method="post" novalidate>
  {{ form.hidden_tag() }}                      <!-- CSRF token - required -->
  {{ form.title.label(class="label") }}
  {{ form.title(class="input") }}
  {% for e in form.title.errors %}<p class="help is-danger">{{ e }}</p>{% endfor %}
  {{ form.submit(class="button is-primary") }}
</form>
```

```python
from flask_wtf.csrf import CSRFProtect
CSRFProtect(app)                               # protect EVERY POST, not just FlaskForms
```

```html
<input type="hidden" name="csrf_token" value="{{ csrf_token() }}">   <!-- in plain forms -->
```

---

## Flask-Login

```python
from flask_login import LoginManager, UserMixin, login_user, logout_user, login_required, current_user

login_manager = LoginManager(app)
login_manager.login_view = "auth.login"

class User(UserMixin, db.Model): ...

@login_manager.user_loader
def load_user(user_id):
    return db.session.get(User, int(user_id))
```

```python
from flask_login import current_user, login_user, logout_user

login_user(user, remember=True)                # log in
logout_user()                                  # log out
current_user.is_authenticated                  # logged in?
current_user.id                                # who

@app.route("/private")                         # route FIRST
@login_required                                # then this
def private(): ...
```

```python
from werkzeug.security import generate_password_hash, check_password_hash
generate_password_hash("pw")                   # store this
check_password_hash(stored_hash, "pw")         # True / False
```

Guide: [[Flask - user accounts and login]]

### Ownership checks

```python
from flask import abort
from flask_login import current_user

def get_own_task(task_id):
    task = db.get_or_404(Task, task_id)
    if task.user_id != current_user.id:
        abort(404)
    return task
```

> ⚠️ **`@login_required` checks *someone* is logged in, not that the item is *theirs*.** Use a helper like this on every route that takes an id.

---

## Blueprints and the app factory

```python
# app/tasks.py
from flask import Blueprint
bp = Blueprint("tasks", __name__, url_prefix="/tasks")

@bp.route("/")
def index(): ...
```

```python
# app/__init__.py
from flask import Flask

def create_app(config="config.DevelopmentConfig"):
    app = Flask(__name__)
    app.config.from_object(config)
    db.init_app(app)
    from app.tasks import bp
    app.register_blueprint(bp)
    return app
```

```python
url_for("tasks.index")                         # blueprint.function
url_for(".index")                              # current blueprint
from flask import current_app, url_for
current_app.config["SECRET_KEY"]               # instead of app.config
```

Guide: [[Flask - project structure and blueprints]]

## Config

```python
import os

app.config["SECRET_KEY"] = os.environ["SECRET_KEY"]
app.config.from_object("config.ProductionConfig")
app.config["MAX_CONTENT_LENGTH"] = 5 * 1024 * 1024
```

| Setting | For |
|---|---|
| `SECRET_KEY` | Signs sessions, flash, CSRF — **required** |
| `SQLALCHEMY_DATABASE_URI` | Database location |
| `SESSION_COOKIE_SECURE` | HTTPS-only cookies (production) |
| `SESSION_COOKIE_HTTPONLY` | JS can't read the cookie |
| `SESSION_COOKIE_SAMESITE` | `"Lax"` — blocks most cross-site tricks |
| `REMEMBER_COOKIE_DURATION` | How long "remember me" lasts |
| `MAX_CONTENT_LENGTH` | Upload size limit |
| `WTF_CSRF_ENABLED` | Turn CSRF off — **tests only** |

```bash
python -c "import secrets; print(secrets.token_hex(32))"     # a proper SECRET_KEY
```

## Request hooks

```python
@app.before_request
def before(): ...                              # runs before every request

@app.after_request
def after(response):
    response.headers["X-Frame-Options"] = "DENY"
    return response                            # must return the response

@app.context_processor
def inject():
    return {"app_name": "Task Tracker"}        # available in every template
```

`g` holds data for the current request only: `from flask import g; g.start = time.time()`.

---

## Testing

```python
import pytest

@pytest.fixture
def client():
    app = create_app("config.TestingConfig")
    with app.app_context():
        db.create_all()
        yield app.test_client()
        db.drop_all()

def test_home(client):
    r = client.get("/")
    assert r.status_code == 302

def test_add(client):
    r = client.post("/", data={"title": "Milk"}, follow_redirects=True)
    r = client.post("/api/tasks", json={"title": "Milk"})
```

Full working example: [[Flask - full project walkthrough]]

---

## Production

```bash
gunicorn -w 4 -b 0.0.0.0:8000 "app:create_app()"            # Linux / WSL / Docker
waitress-serve --host 0.0.0.0 --port 8000 --call app:create_app   # Windows
```

- [ ] Debug **off** — never `--debug` on a server
- [ ] `SECRET_KEY` random, from the environment
- [ ] PostgreSQL if several users write at once
- [ ] `flask db upgrade` on every deploy
- [ ] HTTPS + `SESSION_COOKIE_SECURE = True`
- [ ] `ProxyFix` behind a proxy
- [ ] Logs to stdout

Guide: [[Flask - deploying it]]

---

## Command-line cheat sheet

```bash
flask --app app run --debug                 # develop
flask --app app run --host 0.0.0.0 --port 8000
flask --app app routes                      # list every URL
flask --app app shell                       # Python prompt with the app loaded
flask --app app db init | migrate -m "x" | upgrade | downgrade
```

---

## Error messages and what they mean

| Error | Cause | Fix |
|---|---|---|
| `No module named flask` | venv not active | Activate it / `uv run` |
| `TemplateNotFound` | Folder not named `templates` | Rename it |
| `BuildError: Could not build url for endpoint 'x'` | Wrong name in `url_for` | Use the function name; `blueprint.name` in blueprints |
| `405 Method Not Allowed` | POST to a GET-only route | `methods=["GET", "POST"]` |
| `400 Bad Request` | `request.form["x"]` missing, or no CSRF token | `.get()` / add the token |
| `The CSRF token is missing` | Form without a token | `{{ form.hidden_tag() }}` |
| `The session is unavailable because no secret key was set` | No `SECRET_KEY` | Set one |
| `Working outside of application context` | DB use outside a request | `with app.app_context():` |
| `no such table` / `no such column` | Tables not built / not migrated | `flask db upgrade` |
| `PendingRollbackError` | An earlier DB error | `db.session.rollback()` |
| `View function mapping is overwriting` | Two routes, same function name | Rename one |
| `Address already in use` | Server already running | Stop it / `--port 5001` |
| `Debug mode: off` despite `DEBUG=True` | Config doesn't control `flask run` | `--debug` |
| `database is locked` | SQLite under load | PostgreSQL |

## Related

[[Flask]] · [[Flask - full project walkthrough]] · [[Django reference]] · [[FastAPI reference]] · [[SQLAlchemy]] · [[Security in practice]] · [[Python]]
