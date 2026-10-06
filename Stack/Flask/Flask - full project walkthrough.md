---
tags: [flask, project, walkthrough, bulma, sqlalchemy, guide]
---

# Flask — full project walkthrough

**Build a complete app, start to finish:** a task tracker with accounts, a SQL database, Bulma styling and tests. Every file, in order, with what it does.

Hub: [[Flask]] · Uses all eight guides · Everything: [[Flask reference]]

> ✅ **This code has been run, not just written.** Built with Flask 3.1, Flask-SQLAlchemy 3.1, Flask-Login 0.6 and Flask-WTF 1.3; the 8 tests at the end pass, migrations create the tables, and the server starts with the exact commands below.

---

## What you're building

- **Register, log in, log out** — with hashed passwords
- **Add, tick off, edit and delete tasks**
- **Each user sees only their own tasks** — even if they guess another task's URL
- A clean **Bulma** layout that works on a phone
- A **SQLite** database you can swap for PostgreSQL by changing one line
- **Tests** that prove all of the above

```
Browser ──► Flask routes (auth.py, tasks.py)
               │   check the form (forms.py)
               │   check who's logged in (Flask-Login)
               ▼
            models.py ──► SQLAlchemy ──► instance/tasks.db
               │
               ▼
            templates/ (Jinja + Bulma) ──► HTML back to the browser
```

---

## Step 0 — The folder

```
tasktracker/
├── app/
│   ├── __init__.py            # builds the app
│   ├── models.py              # the database tables
│   ├── forms.py               # the forms
│   ├── auth.py                # register / login / logout pages
│   ├── tasks.py               # the task pages
│   ├── templates/
│   │   ├── base.html          # the shared layout
│   │   ├── _flash.html        # one-time messages
│   │   ├── _formhelpers.html  # a macro that draws a Bulma form field
│   │   ├── auth/
│   │   │   ├── login.html
│   │   │   └── register.html
│   │   └── tasks/
│   │       ├── index.html
│   │       └── edit.html
│   └── static/
│       ├── style.css
│       └── bulma.js
├── tests/
│   ├── __init__.py            # empty - makes tests/ a package
│   └── test_app.py
├── config.py
├── .env                       # secrets - never committed
└── .gitignore
```

---

## Step 1 — Set up the project

**WSL / Linux:**

```bash
mkdir -p ~/tasktracker && cd ~/tasktracker
python3 -m venv .venv
source .venv/bin/activate
pip install flask flask-sqlalchemy flask-migrate flask-login flask-wtf email-validator python-dotenv pytest
mkdir -p app/templates/auth app/templates/tasks app/static tests
touch tests/__init__.py
```

**Windows (PowerShell):**

```powershell
mkdir ~\Documents\tasktracker; cd ~\Documents\tasktracker
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install flask flask-sqlalchemy flask-migrate flask-login flask-wtf email-validator python-dotenv pytest
mkdir app\templates\auth, app\templates\tasks, app\static, tests
New-Item tests\__init__.py
```

**With uv (either system):**

```bash
uv init tasktracker && cd tasktracker
uv add flask flask-sqlalchemy flask-migrate flask-login flask-wtf email-validator python-dotenv
uv add --dev pytest
```

| Package | Job |
|---|---|
| `flask` | The web framework |
| `flask-sqlalchemy` | Talk to the database with Python classes |
| `flask-migrate` | Change the database's structure safely |
| `flask-login` | Remember who's logged in |
| `flask-wtf` | Forms, validation and CSRF protection |
| `email-validator` | Needed by the `Email()` form check |
| `python-dotenv` | Load secrets from `.env` |
| `pytest` | Run the tests |

---

## Step 2 — Secrets: `.env` and `.gitignore`

Generate a secret key:

```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

`.env`:

```bash
SECRET_KEY=paste-the-long-random-string-here
DATABASE_URL=sqlite:///tasks.db
```

`.gitignore`:

```
.venv/
__pycache__/
instance/
.env
*.db
```

> ⚠️ **The `SECRET_KEY` signs everyone's login cookie.** Anyone who has it can log in as any user. Never commit it — [[Flask - project structure and blueprints]].

---

## Step 3 — `config.py`

```python
import os
from dotenv import load_dotenv

load_dotenv()                                        # read .env into the environment


class Config:
    SECRET_KEY = os.environ["SECRET_KEY"]            # crash loudly if it's missing
    SQLALCHEMY_DATABASE_URI = os.environ.get("DATABASE_URL", "sqlite:///tasks.db")
    SESSION_COOKIE_HTTPONLY = True
    SESSION_COOKIE_SAMESITE = "Lax"


class DevelopmentConfig(Config):
    DEBUG = True


class TestingConfig(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"   # a throwaway database for tests
    WTF_CSRF_ENABLED = False                         # tests don't send CSRF tokens


class ProductionConfig(Config):
    DEBUG = False
    SESSION_COOKIE_SECURE = True                     # cookies over HTTPS only
    REMEMBER_COOKIE_SECURE = True
```

**What it does:** one place for every setting. Development, testing and production each get a class that inherits the shared settings and changes only what differs.

---

## Step 4 — `app/__init__.py` — building the app

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_login import LoginManager
from flask_wtf.csrf import CSRFProtect
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass


db = SQLAlchemy(model_class=Base)                    # the database
migrate = Migrate()                                  # schema changes
login_manager = LoginManager()                       # who's logged in
csrf = CSRFProtect()                                 # form protection

login_manager.login_view = "auth.login"              # where to send people who aren't logged in
login_manager.login_message_category = "is-warning"  # Bulma colour for "please log in"


def create_app(config_class="config.DevelopmentConfig"):
    app = Flask(__name__)
    app.config.from_object(config_class)

    db.init_app(app)
    migrate.init_app(app, db)
    login_manager.init_app(app)
    csrf.init_app(app)

    from app import models                          # noqa: F401 - registers the tables
    from app.auth import bp as auth_bp
    from app.tasks import bp as tasks_bp
    app.register_blueprint(auth_bp)
    app.register_blueprint(tasks_bp)

    return app
```

**What it does:** creates the four extensions, then `create_app()` joins them to a Flask app and plugs in the two blueprints. The imports are *inside* the function to avoid a circular import — `models.py` needs `db`, which must exist first. See [[Flask - project structure and blueprints]].

---

## Step 5 — `app/models.py` — the database tables

```python
from datetime import datetime, timezone

from flask_login import UserMixin
from sqlalchemy import String, ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship
from werkzeug.security import generate_password_hash, check_password_hash

from app import db, login_manager


class User(UserMixin, db.Model):
    __tablename__ = "users"                          # "user" is reserved in PostgreSQL

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(120), unique=True, index=True)
    password_hash: Mapped[str] = mapped_column(String(256))
    tasks: Mapped[list["Task"]] = relationship(back_populates="owner", cascade="all, delete-orphan")

    def set_password(self, password: str) -> None:
        self.password_hash = generate_password_hash(password)

    def check_password(self, password: str) -> bool:
        return check_password_hash(self.password_hash, password)


class Task(db.Model):
    __tablename__ = "tasks"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    done: Mapped[bool] = mapped_column(default=False)
    created_at: Mapped[datetime] = mapped_column(default=lambda: datetime.now(timezone.utc))
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    owner: Mapped["User"] = relationship(back_populates="tasks")


@login_manager.user_loader
def load_user(user_id):
    return db.session.get(User, int(user_id))       # cookie id -> User object
```

**What it does:** two tables. A user has many tasks (`relationship`); each task stores its owner's id (`ForeignKey`). Passwords are only ever stored hashed. `load_user` turns the id in the login cookie back into a `User` on every request. See [[Flask - connecting to a SQL database]] and [[Flask - user accounts and login]].

---

## Step 6 — `app/forms.py`

```python
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, BooleanField, SubmitField
from wtforms.validators import DataRequired, Email, Length, EqualTo


class RegisterForm(FlaskForm):
    email = StringField("Email", validators=[DataRequired(), Email()])
    password = PasswordField("Password", validators=[DataRequired(), Length(min=8)])
    confirm = PasswordField("Confirm password", validators=[
        DataRequired(), EqualTo("password", message="Passwords must match.")])
    submit = SubmitField("Create account")


class LoginForm(FlaskForm):
    email = StringField("Email", validators=[DataRequired(), Email()])
    password = PasswordField("Password", validators=[DataRequired()])
    remember = BooleanField("Remember me")
    submit = SubmitField("Log in")


class TaskForm(FlaskForm):
    title = StringField("Task", validators=[DataRequired(), Length(max=200)])
    submit = SubmitField("Add")
```

**What it does:** each class describes one form and its rules. Flask-WTF checks them and produces the error messages. See [[Flask - forms and user input]].

---

## Step 7 — `app/auth.py` — register, log in, log out

```python
from urllib.parse import urlsplit

from flask import Blueprint, render_template, redirect, url_for, flash, request
from flask_login import login_user, logout_user, login_required, current_user

from app import db
from app.models import User
from app.forms import RegisterForm, LoginForm

bp = Blueprint("auth", __name__)


@bp.route("/register", methods=["GET", "POST"])
def register():
    if current_user.is_authenticated:
        return redirect(url_for("tasks.index"))
    form = RegisterForm()
    if form.validate_on_submit():
        email = form.email.data.strip().lower()
        if db.session.scalar(db.select(User).filter_by(email=email)):
            flash("That email is already registered.", "is-danger")
            return render_template("auth/register.html", form=form)
        user = User(email=email)
        user.set_password(form.password.data)
        db.session.add(user)
        db.session.commit()
        flash("Account created - please log in.", "is-success")
        return redirect(url_for("auth.login"))
    return render_template("auth/register.html", form=form)


@bp.route("/login", methods=["GET", "POST"])
def login():
    if current_user.is_authenticated:
        return redirect(url_for("tasks.index"))
    form = LoginForm()
    if form.validate_on_submit():
        user = db.session.scalar(db.select(User).filter_by(email=form.email.data.strip().lower()))
        if user is None or not user.check_password(form.password.data):
            flash("Wrong email or password.", "is-danger")
            return render_template("auth/login.html", form=form)
        login_user(user, remember=form.remember.data)
        next_page = request.args.get("next")
        if not next_page or urlsplit(next_page).netloc != "":   # only redirect within this site
            next_page = url_for("tasks.index")
        return redirect(next_page)
    return render_template("auth/login.html", form=form)


@bp.post("/logout")
@login_required
def logout():
    logout_user()
    flash("Logged out.", "is-info")
    return redirect(url_for("auth.login"))
```

**What it does:**

| Route | Job |
|---|---|
| `/register` | Checks the email isn't taken (lower-cased, so `A@x.com` = `a@x.com`), hashes the password, saves the user |
| `/login` | Checks the password against the hash, sets the login cookie, sends you back where you were — but only to a page on this site |
| `/logout` | POST only, so no other site can log you out with a link |

---

## Step 8 — `app/tasks.py` — the task pages

```python
from flask import Blueprint, render_template, redirect, url_for, flash, abort
from flask_login import login_required, current_user

from app import db
from app.models import Task
from app.forms import TaskForm

bp = Blueprint("tasks", __name__)


def get_own_task(task_id: int) -> Task:
    task = db.get_or_404(Task, task_id)
    if task.user_id != current_user.id:              # it exists, but it isn't yours
        abort(404)
    return task


@bp.route("/", methods=["GET", "POST"])
@login_required
def index():
    form = TaskForm()
    if form.validate_on_submit():
        db.session.add(Task(title=form.title.data.strip(), user_id=current_user.id))
        db.session.commit()
        flash("Task added.", "is-success")
        return redirect(url_for("tasks.index"))      # redirect after POST
    tasks = db.session.scalars(
        db.select(Task)
        .filter_by(user_id=current_user.id)
        .order_by(Task.done, Task.created_at.desc())  # open tasks first, newest at the top
    ).all()
    return render_template("tasks/index.html", form=form, tasks=tasks)


@bp.post("/tasks/<int:task_id>/toggle")
@login_required
def toggle(task_id):
    task = get_own_task(task_id)
    task.done = not task.done
    db.session.commit()
    return redirect(url_for("tasks.index"))


@bp.route("/tasks/<int:task_id>/edit", methods=["GET", "POST"])
@login_required
def edit(task_id):
    task = get_own_task(task_id)
    form = TaskForm(obj=task)                        # pre-fill from the task
    form.submit.label.text = "Save"
    if form.validate_on_submit():
        task.title = form.title.data.strip()
        db.session.commit()
        flash("Task updated.", "is-success")
        return redirect(url_for("tasks.index"))
    return render_template("tasks/edit.html", form=form, task=task)


@bp.post("/tasks/<int:task_id>/delete")
@login_required
def delete(task_id):
    task = get_own_task(task_id)
    db.session.delete(task)
    db.session.commit()
    flash("Task deleted.", "is-info")
    return redirect(url_for("tasks.index"))
```

**What it does:** the four CRUD actions. The line that matters most is `get_own_task()` — **every** route that touches a specific task goes through it, so no user can reach another user's task by changing the number in the URL. The owner is always set from `current_user.id`, never from the form.

> **Why toggle and delete are `POST`:** a `GET` link can be triggered by a browser prefetching pages or by another website. Changing data always goes through a form with a CSRF token.

---

## Step 9 — The templates

### `app/templates/base.html` — the shared layout

```html
<!doctype html>
<html lang="en" data-theme="light">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{% block title %}Task Tracker{% endblock %}</title>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bulma@1.0.2/css/bulma.min.css">
  <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
<body>
  <nav class="navbar is-primary" role="navigation" aria-label="main navigation">
    <div class="navbar-brand">
      <a class="navbar-item has-text-weight-bold" href="{{ url_for('tasks.index') }}">Task Tracker</a>
      <a role="button" class="navbar-burger" aria-label="menu" aria-expanded="false" data-target="main-nav">
        <span aria-hidden="true"></span><span aria-hidden="true"></span>
        <span aria-hidden="true"></span><span aria-hidden="true"></span>
      </a>
    </div>
    <div id="main-nav" class="navbar-menu">
      <div class="navbar-end">
        {% if current_user.is_authenticated %}
          <span class="navbar-item">{{ current_user.email }}</span>
          <div class="navbar-item">
            <form method="post" action="{{ url_for('auth.logout') }}">
              <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
              <button class="button is-light is-small">Log out</button>
            </form>
          </div>
        {% else %}
          <a class="navbar-item" href="{{ url_for('auth.login') }}">Log in</a>
          <a class="navbar-item" href="{{ url_for('auth.register') }}">Register</a>
        {% endif %}
      </div>
    </div>
  </nav>

  <section class="section">
    <div class="container is-max-desktop">
      {% include "_flash.html" %}
      {% block content %}{% endblock %}
    </div>
  </section>

  <script src="{{ url_for('static', filename='bulma.js') }}"></script>
</body>
</html>
```

### `app/templates/_flash.html`

```html
{% with messages = get_flashed_messages(with_categories=true) %}
  {% for category, message in messages %}
    <div class="notification {{ category }} is-light">
      <button class="delete" aria-label="close"></button>
      {{ message }}
    </div>
  {% endfor %}
{% endwith %}
```

> **The flash category *is* the Bulma class** — `flash("...", "is-success")` becomes a green box with no extra code.

### `app/templates/_formhelpers.html` — one macro for every field

```html
{% macro render_field(field, placeholder="") %}
  <div class="field">
    {% if field.type == "BooleanField" %}
      <label class="checkbox">{{ field() }} {{ field.label.text }}</label>
    {% else %}
      {{ field.label(class="label") }}
      <div class="control">
        {{ field(class="input" ~ (" is-danger" if field.errors else ""), placeholder=placeholder) }}
      </div>
    {% endif %}
    {% for error in field.errors %}
      <p class="help is-danger">{{ error }}</p>
    {% endfor %}
  </div>
{% endmacro %}
```

### `app/templates/auth/login.html`

```html
{% extends "base.html" %}
{% from "_formhelpers.html" import render_field %}
{% block title %}Log in{% endblock %}

{% block content %}
<div class="columns is-centered">
  <div class="column is-7">
    <div class="box">
      <h1 class="title">Log in</h1>
      <form method="post" novalidate>
        {{ form.hidden_tag() }}
        {{ render_field(form.email, "you@example.com") }}
        {{ render_field(form.password) }}
        {{ render_field(form.remember) }}
        {{ form.submit(class="button is-primary is-fullwidth") }}
      </form>
      <p class="has-text-centered mt-4">
        No account? <a href="{{ url_for('auth.register') }}">Register</a>
      </p>
    </div>
  </div>
</div>
{% endblock %}
```

### `app/templates/auth/register.html`

```html
{% extends "base.html" %}
{% from "_formhelpers.html" import render_field %}
{% block title %}Register{% endblock %}

{% block content %}
<div class="columns is-centered">
  <div class="column is-7">
    <div class="box">
      <h1 class="title">Create an account</h1>
      <form method="post" novalidate>
        {{ form.hidden_tag() }}
        {{ render_field(form.email, "you@example.com") }}
        {{ render_field(form.password) }}
        {{ render_field(form.confirm) }}
        {{ form.submit(class="button is-primary is-fullwidth") }}
      </form>
      <p class="has-text-centered mt-4">
        Already registered? <a href="{{ url_for('auth.login') }}">Log in</a>
      </p>
    </div>
  </div>
</div>
{% endblock %}
```

### `app/templates/tasks/index.html` — the main page

```html
{% extends "base.html" %}
{% block title %}My tasks{% endblock %}

{% block content %}
<nav class="level">
  <div class="level-left">
    <h1 class="title">My tasks</h1>
  </div>
  <div class="level-right">
    <span class="tag is-medium">{{ tasks | selectattr("done") | list | length }} / {{ tasks | length }} done</span>
  </div>
</nav>

<form method="post" class="box" novalidate>
  {{ form.hidden_tag() }}
  <div class="field has-addons">
    <div class="control is-expanded">
      {{ form.title(class="input" ~ (" is-danger" if form.title.errors else ""), placeholder="What needs doing?") }}
    </div>
    <div class="control">
      {{ form.submit(class="button is-primary") }}
    </div>
  </div>
  {% for error in form.title.errors %}
    <p class="help is-danger">{{ error }}</p>
  {% endfor %}
</form>

{% if tasks %}
<div class="table-container">
  <table class="table is-fullwidth is-hoverable">
    <tbody>
      {% for task in tasks %}
      <tr>
        <td class="is-narrow">
          <form method="post" action="{{ url_for('tasks.toggle', task_id=task.id) }}">
            <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
            <button class="button is-small {{ 'is-success' if task.done else 'is-light' }}"
                    title="{{ 'Mark as not done' if task.done else 'Mark as done' }}">
              {{ '✓' if task.done else '○' }}
            </button>
          </form>
        </td>
        <td class="{{ 'task-done' if task.done }}">{{ task.title }}</td>
        <td class="is-narrow has-text-grey is-size-7">{{ task.created_at.strftime('%d %b %Y') }}</td>
        <td class="is-narrow">
          <div class="buttons is-right">
            <a class="button is-small" href="{{ url_for('tasks.edit', task_id=task.id) }}">Edit</a>
            <form method="post" action="{{ url_for('tasks.delete', task_id=task.id) }}"
                  onsubmit="return confirm('Delete this task?')">
              <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
              <button class="button is-small is-danger is-outlined">Delete</button>
            </form>
          </div>
        </td>
      </tr>
      {% endfor %}
    </tbody>
  </table>
</div>
{% else %}
<div class="notification is-light">No tasks yet — add your first one above.</div>
{% endif %}
{% endblock %}
```

**What the pieces do:** `level` puts the title left and the counter right; `field has-addons` joins the input to its button; each row's toggle and delete are tiny POST forms carrying a CSRF token; `onsubmit="return confirm(...)"` gives a free "are you sure?" with no JavaScript file.

### `app/templates/tasks/edit.html`

```html
{% extends "base.html" %}
{% from "_formhelpers.html" import render_field %}
{% block title %}Edit task{% endblock %}

{% block content %}
<div class="columns is-centered">
  <div class="column is-8">
    <div class="box">
      <h1 class="title is-4">Edit task</h1>
      <form method="post" novalidate>
        {{ form.hidden_tag() }}
        {{ render_field(form.title) }}
        <div class="field is-grouped">
          <div class="control">{{ form.submit(class="button is-primary") }}</div>
          <div class="control"><a class="button is-light" href="{{ url_for('tasks.index') }}">Cancel</a></div>
        </div>
      </form>
    </div>
  </div>
</div>
{% endblock %}
```

---

## Step 10 — Static files

### `app/static/style.css`

```css
.task-done {
  text-decoration: line-through;
  opacity: 0.55;
}

td {
  vertical-align: middle !important;
}
```

### `app/static/bulma.js`

```javascript
// Bulma is CSS only - these few lines make the mobile menu and the notification close buttons work
document.addEventListener("DOMContentLoaded", () => {
  document.querySelectorAll(".navbar-burger").forEach(burger => {
    burger.addEventListener("click", () => {
      const menu = document.getElementById(burger.dataset.target);
      burger.classList.toggle("is-active");
      menu.classList.toggle("is-active");
      burger.setAttribute("aria-expanded", burger.classList.contains("is-active"));
    });
  });

  document.querySelectorAll(".notification .delete").forEach(btn => {
    btn.addEventListener("click", () => btn.parentNode.remove());
  });
});
```

---

## Step 11 — Create the database and run it

```bash
flask --app app db init                               # once: creates migrations/
flask --app app db migrate -m "create users and tasks"
flask --app app db upgrade                            # creates instance/tasks.db
flask --app app run --debug
```

You should see the migration report `Detected added table 'users'` and `Detected added table 'tasks'`. Open **http://127.0.0.1:5000** — you're sent to the login page. Register, log in, add a task.

> ⚠️ **`DEBUG = True` in `DevelopmentConfig` does *not* turn on the debugger or auto-reload under `flask run`.** Only the `--debug` flag does — without it the terminal says `Debug mode: off` regardless of your config. Always start development with `--debug`.

> **See every route the app has:** `flask --app app routes` — useful whenever you get a 404 or a `BuildError`.

---

## Step 12 — The tests

`tests/test_app.py`:

```python
import os

os.environ.setdefault("SECRET_KEY", "test-only-secret")

import pytest

from app import create_app, db
from app.models import Task, User


@pytest.fixture
def app():
    app = create_app("config.TestingConfig")
    with app.app_context():
        db.create_all()
        yield app
        db.drop_all()


@pytest.fixture
def client(app):
    return app.test_client()


def register(client, email="a@example.com", pw="password123"):
    return client.post("/register", data={"email": email, "password": pw, "confirm": pw},
                       follow_redirects=True)


def login(client, email="a@example.com", pw="password123"):
    return client.post("/login", data={"email": email, "password": pw}, follow_redirects=True)


def test_tasks_page_requires_login(client):
    r = client.get("/")
    assert r.status_code == 302
    assert "/login" in r.headers["Location"]


def test_register_login_add_toggle_edit_delete(client, app):
    assert b"Account created" in register(client).data
    r = login(client)
    assert b"My tasks" in r.data

    r = client.post("/", data={"title": "Buy milk"}, follow_redirects=True)
    assert b"Buy milk" in r.data and b"Task added" in r.data

    with app.app_context():
        task = db.session.scalar(db.select(Task))
        tid = task.id
        assert task.done is False

    client.post(f"/tasks/{tid}/toggle")
    with app.app_context():
        assert db.session.get(Task, tid).done is True

    r = client.post(f"/tasks/{tid}/edit", data={"title": "Buy oat milk"}, follow_redirects=True)
    assert b"Buy oat milk" in r.data

    r = client.post(f"/tasks/{tid}/delete", follow_redirects=True)
    assert b"Task deleted" in r.data
    with app.app_context():
        assert db.session.get(Task, tid) is None


def test_password_is_hashed(client, app):
    register(client)
    with app.app_context():
        user = db.session.scalar(db.select(User))
        assert user.password_hash != "password123"
        assert user.check_password("password123")


def test_wrong_password_rejected(client):
    register(client)
    r = login(client, pw="wrong-password")
    assert b"Wrong email or password" in r.data


def test_duplicate_email_rejected(client):
    register(client)
    r = register(client, email="A@Example.com")        # same email, different case
    assert b"already registered" in r.data


def test_cannot_touch_another_users_task(client, app):
    register(client, "owner@example.com")
    login(client, "owner@example.com")
    client.post("/", data={"title": "Private"})
    with app.app_context():
        tid = db.session.scalar(db.select(Task)).id
    client.post("/logout")

    register(client, "intruder@example.com")
    login(client, "intruder@example.com")
    assert client.get(f"/tasks/{tid}/edit").status_code == 404
    assert client.post(f"/tasks/{tid}/delete").status_code == 404
    assert client.post(f"/tasks/{tid}/toggle").status_code == 404
    with app.app_context():
        assert db.session.get(Task, tid) is not None      # still there


def test_open_redirect_blocked(client):
    register(client)
    r = client.post("/login?next=https://evil.example.com",
                    data={"email": "a@example.com", "password": "password123"})
    assert r.status_code == 302
    assert "evil.example.com" not in r.headers["Location"]


def test_empty_title_rejected(client):
    register(client)
    login(client)
    r = client.post("/", data={"title": ""})
    assert r.status_code == 200
    assert b"This field is required" in r.data
```

```bash
pytest -q
```

```
........                                                                 [100%]
8 passed
```

**What they prove:**

| Test | Proves |
|---|---|
| `test_tasks_page_requires_login` | Logged-out visitors are sent to the login page |
| `test_register_login_add_toggle_edit_delete` | The whole CRUD cycle works |
| `test_password_is_hashed` | The real password is never stored |
| `test_wrong_password_rejected` | A bad password doesn't get in |
| `test_duplicate_email_rejected` | `A@Example.com` and `a@example.com` are one account |
| **`test_cannot_touch_another_users_task`** | **Another user gets a 404 — can't view, edit, tick or delete your task** |
| **`test_open_redirect_blocked`** | The login page can't be used to bounce people to another site |
| `test_empty_title_rejected` | Validation errors show on the page |

> **The two security tests are the ones worth copying into every project you build.** Both bugs are invisible when you click around as a single user — they only show up when someone deliberately tries them.

---

## Step 13 — Moving to PostgreSQL

```bash
pip install "psycopg[binary]"
docker run -d --name pg -e POSTGRES_PASSWORD=secret -e POSTGRES_DB=tasks -p 5432:5432 postgres:16
```

Change one line in `.env`:

```bash
DATABASE_URL=postgresql+psycopg://postgres:secret@localhost:5432/tasks
```

```bash
flask --app app db upgrade            # build the same tables in Postgres
```

No Python changes. That's the point of the ORM — and why the tables are named `users`, not `user`.

---

## Step 14 — Where to take it next

Each of these is a small, self-contained next step:

| Add | How | Guide |
|---|---|---|
| Due dates | A `due_date: Mapped[date \| None]` column, a `DateField`, then `db migrate` + `db upgrade` | [[Flask - connecting to a SQL database]] |
| Filter tabs (All / Open / Done) | `?filter=open` read with `request.args`, Bulma `tabs` | [[Flask - styling with Bulma]] |
| Tick without reloading | A `PATCH /api/tasks/<id>` route and `fetch` | [[Flask - building a JSON API]] |
| Pagination | `db.paginate(...)` | [[Flask - connecting to a SQL database]] |
| Rate-limited login | Flask-Limiter | [[Flask - user accounts and login]] |
| Deploy it | Gunicorn + Docker Compose with Postgres | [[Flask - deploying it]] |

## Related

[[Flask]] · [[Flask reference]] · [[Flask - your first app]] · [[Flask - deploying it]] · [[Security in practice]] · [[pytest]]
