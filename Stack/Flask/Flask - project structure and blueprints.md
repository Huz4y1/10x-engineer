---
tags: [flask, structure, blueprints, config, guide]
---

# Flask — project structure and blueprints

**Guide 8a of 8.** Going from one `app.py` file to a tidy project: the app factory, blueprints, config and secrets. Do this once your app has more than a handful of routes.

Hub: [[Flask]] · Previous: [[Flask - building a JSON API]] · Next: [[Flask - deploying it]]

---

## Why one file stops working

A single `app.py` is perfect for learning. Past about 200 lines it becomes a problem: routes, models, forms and config all tangled in one file, and every change means scrolling through everything.

**The fix is to split by job** — a file for models, a file for forms, and a separate file for each *area* of the site (login pages, task pages, API).

---

## The layout

```
tasktracker/
├── app/                         <- your application is now a PACKAGE (a folder with __init__.py)
│   ├── __init__.py              <- create_app(): builds and wires everything together
│   ├── models.py                <- database tables
│   ├── forms.py                 <- Flask-WTF forms
│   ├── auth.py                  <- blueprint: register / login / logout
│   ├── tasks.py                 <- blueprint: the task pages
│   ├── api.py                   <- blueprint: /api/... JSON routes
│   ├── templates/
│   │   ├── base.html
│   │   ├── _flash.html
│   │   ├── _formhelpers.html
│   │   ├── auth/                <- one folder per blueprint keeps templates findable
│   │   │   ├── login.html
│   │   │   └── register.html
│   │   └── tasks/
│   │       ├── index.html
│   │       └── edit.html
│   └── static/
│       ├── style.css
│       └── bulma.js
├── migrations/                  <- made by `flask db init`
├── instance/                    <- SQLite database lives here - NOT committed
├── tests/
├── config.py                    <- settings
├── .env                         <- secrets - NEVER committed
├── .gitignore
└── pyproject.toml
```

---

## The app factory — `create_app()`

Instead of creating `app` at the top of a file, you write a **function that builds the app**. That sounds like extra work, but it fixes two real problems:

1. **Circular imports** — `models.py` needs `db`, and the app needs the models. With a factory, the pieces exist separately and are joined at the end.
2. **Testing** — you can build a fresh app with a test database for every test.

`app/__init__.py`:

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_login import LoginManager
from flask_wtf.csrf import CSRFProtect
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

# 1. create the extensions WITHOUT an app - they're attached inside create_app()
db = SQLAlchemy(model_class=Base)
migrate = Migrate()
login_manager = LoginManager()
csrf = CSRFProtect()

login_manager.login_view = "auth.login"               # blueprint_name.function_name
login_manager.login_message_category = "is-warning"

def create_app(config_class="config.DevelopmentConfig"):
    app = Flask(__name__, instance_relative_config=True)
    app.config.from_object(config_class)              # 2. load settings

    db.init_app(app)                                  # 3. attach each extension to this app
    migrate.init_app(app, db)
    login_manager.init_app(app)
    csrf.init_app(app)

    from app import models                            # 4. import models so their tables are known
    from app.auth import bp as auth_bp                # 5. register each blueprint
    from app.tasks import bp as tasks_bp
    from app.api import bp as api_bp
    app.register_blueprint(auth_bp)
    app.register_blueprint(tasks_bp)
    app.register_blueprint(api_bp, url_prefix="/api")  # every route in api.py starts with /api

    return app
```

**Running it** — Flask finds `create_app` automatically:

```bash
flask --app app run --debug        # "app" is now the package folder, and Flask calls create_app()
```

> **Why are the imports inside the function (steps 4 and 5)?** `models.py` does `from app import db`. If `app/__init__.py` imported `models` at the top — before `db` exists — Python would hit a circular import. Importing inside `create_app()` runs after everything is defined.

---

## Blueprints — splitting routes into files

**A blueprint is a group of related routes**, in their own file, that you plug into the app. Think of each one as a mini-app: the login area, the task area, the API.

`app/auth.py`:

```python
from flask import Blueprint, render_template, redirect, url_for, flash

bp = Blueprint("auth", __name__)          # "auth" is the blueprint's NAME - used in url_for

@bp.route("/login", methods=["GET", "POST"])     # @bp.route, not @app.route
def login():
    ...

@bp.route("/register", methods=["GET", "POST"])
def register():
    ...
```

`app/tasks.py`:

```python
from flask import Blueprint
from flask_login import login_required

bp = Blueprint("tasks", __name__)

@bp.route("/")
@login_required
def index():
    ...
```

### ⚠️ `url_for` now needs the blueprint name

```python
from flask import url_for

url_for("login")              # NO LONGER WORKS - BuildError
url_for("auth.login")         # blueprint name + "." + function name
url_for("tasks.index")
url_for(".index")             # a leading dot = "in the CURRENT blueprint" - a handy shortcut
```

```html
<a href="{{ url_for('auth.login') }}">Log in</a>
```

> **This is the most common error after splitting into blueprints:** `BuildError: Could not build url for endpoint 'login'`. Every `url_for` in your Python and your templates needs the `blueprint.` prefix. `flask --app app routes` lists the exact names.

### Blueprint options

```python
from flask import Blueprint

bp = Blueprint("api", __name__, url_prefix="/api")            # every route starts with /api
bp = Blueprint("admin", __name__, template_folder="templates") # its own templates folder
```

```python
@bp.before_request                  # runs before every route in THIS blueprint only
@login_required
def require_login():
    pass
```

> **`before_request` on a blueprint** is a neat way to protect a whole area — every route in `admin.py` requires login without decorating each one.

---

## Config — settings in one place

`config.py`, in the project root:

```python
import os
from dotenv import load_dotenv

load_dotenv()                                    # read .env into os.environ

class Config:
    SECRET_KEY = os.environ["SECRET_KEY"]        # [] - crash loudly if it's missing
    SQLALCHEMY_DATABASE_URI = os.environ.get("DATABASE_URL", "sqlite:///tasks.db")
    SESSION_COOKIE_HTTPONLY = True
    SESSION_COOKIE_SAMESITE = "Lax"
    MAX_CONTENT_LENGTH = 5 * 1024 * 1024         # 5 MB upload limit

class DevelopmentConfig(Config):
    DEBUG = True

class TestingConfig(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"   # a throwaway database in memory
    WTF_CSRF_ENABLED = False                        # tests don't carry CSRF tokens

class ProductionConfig(Config):
    DEBUG = False
    SESSION_COOKIE_SECURE = True                 # cookies over HTTPS only
    REMEMBER_COOKIE_SECURE = True
```

```bash
uv add python-dotenv
```

> **One class per environment, inheriting from `Config`.** Development turns debug on; production turns secure cookies on; testing uses a throwaway database. You pick one when you call `create_app(...)`.

### Reading config in your code

```python
from flask import current_app

current_app.config["MAX_CONTENT_LENGTH"]      # inside a route or anything called from one
```

> **`current_app`, not `app`.** With a factory there's no global `app` to import — `current_app` means "whichever app is handling this request".

---

## Secrets and `.env`

Your `SECRET_KEY` and database password must **never** be in your code or in git.

`.env` (in the project root):

```bash
SECRET_KEY=a-long-random-string-see-below
DATABASE_URL=sqlite:///tasks.db
```

**Generate a proper secret key:**

```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

`.gitignore`:

```
.venv/
__pycache__/
instance/
.env
*.db
```

> ⚠️ **What `SECRET_KEY` protects:** it signs the session cookie. **Anyone who knows it can forge a login cookie for any user**, including an admin. If it's ever committed to git, generate a new one — deleting the commit doesn't help, it's still in the history ([[Security in practice]]).

> **Commit a `.env.example`** with the variable *names* and fake values, so someone setting up the project knows what to fill in.

---

## The Flask shell

```bash
flask --app app shell           # a Python prompt with your app and context ready
```

Make your models available automatically in `create_app()`:

```python
@app.shell_context_processor
def shell_context():
    from app.models import User, Task
    return {"db": db, "User": User, "Task": Task}
```

```python
>>> u = User(email="test@example.com"); u.set_password("password123")
>>> db.session.add(u); db.session.commit()
>>> db.session.scalars(db.select(Task)).all()
```

## Custom commands

```python
import click

@app.cli.command("seed")                     # creates `flask --app app seed`
def seed():
    """Fill the database with sample data."""
    db.session.add_all([Task(title="Sample task", user_id=1)])
    db.session.commit()
    click.echo("Seeded.")
```

---

## Testing with the factory

```python
# tests/conftest.py
import pytest
from app import create_app, db

@pytest.fixture
def app():
    app = create_app("config.TestingConfig")    # a fresh app with an in-memory DB
    with app.app_context():
        db.create_all()
        yield app
        db.drop_all()

@pytest.fixture
def client(app):
    return app.test_client()                    # a fake browser
```

```python
# tests/test_pages.py
def test_login_page_loads(client):
    r = client.get("/login")
    assert r.status_code == 200
    assert b"Log in" in r.data

def test_tasks_need_login(client):
    r = client.get("/")
    assert r.status_code == 302                 # redirected to the login page
```

```bash
uv add --dev pytest
uv run pytest
```

> ⚠️ **`TestingConfig` still needs a `SECRET_KEY`** because it inherits `Config`, which reads it from the environment. Set one in `.env` or in the test config itself. See [[pytest]].

---

## Common mistakes

| Mistake | What you'll see | Fix |
|---|---|---|
| `url_for("login")` after moving to a blueprint | `BuildError` | `url_for("auth.login")` |
| Importing models at the top of `__init__.py` | `ImportError` (circular) | Import inside `create_app()` |
| Using `app.config` inside a blueprint | `NameError: app` | `current_app.config` |
| Forgot `register_blueprint` | Every route in that file is 404 | Register it in `create_app()` |
| `.env` committed to git | **Secrets leaked** | `.gitignore` it; rotate the secret |
| Short or guessable `SECRET_KEY` | **Forged logins** | `secrets.token_hex(32)` |
| Script outside a request | `Working outside of application context` | `with app.app_context():` |
| Two blueprints with the same name | `ValueError: already registered` | Unique names |

## Related

[[Flask]] · [[Flask - deploying it]] · [[Flask - full project walkthrough]] · [[Flask reference]] · [[pytest]] · [[Security in practice]] · [[Git-GitHub]]
