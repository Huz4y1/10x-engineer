---
tags: [flask, auth, login, security, flask-login, guide]
---

# Flask — user accounts and login

**Guide 6 of 8.** Letting people register, log in and log out, storing passwords safely, and making sure each user only ever sees their own data.

Hub: [[Flask]] · Previous: [[Flask - forms and user input]] · Next: [[Flask - building a JSON API]]

---

## How logging in actually works

The web has no memory. Every request arrives as if from a stranger — the server doesn't automatically know that the person asking for `/tasks` is the same person who just logged in.

**The fix is a cookie.** When you log in, the server gives your browser a small signed note that says *"this is user 7"*. The browser sends that note back with every request after, and the server reads it.

```
1. POST /login  email + password  ─────►  check the password is right
                                   ◄─────  "OK. Here's a cookie: session = user 7 (signed)"
2. GET /tasks   + cookie           ─────►  read the cookie -> "ah, user 7" -> show THEIR tasks
3. POST /logout                    ─────►  delete the cookie
```

**Signed** means the server stamps the cookie with your `SECRET_KEY`. If anyone edits it to say *"user 8"*, the stamp no longer matches and Flask throws it away.

> ⚠️ **Flask's session cookie is signed, not encrypted.** Anyone can *read* what's inside it — they just can't *change* it. Never put secrets in the session; a user id is fine, a password is not.

**Flask-Login** handles all of this for you: remembering who's logged in, protecting pages, and giving you `current_user` everywhere.

```bash
uv add flask-login flask-wtf email-validator
```

---

## Passwords — the one thing you must get right

> ⚠️ **Never store a password. Store a *hash* of it.**

A **hash** is a one-way scramble: `hunter2` → `scrypt:32768:8:1$Xy...long gibberish`. You can turn a password into a hash, but you can **never** turn the hash back into the password.

To check a login, you hash what they typed and compare the two hashes:

```python
from werkzeug.security import generate_password_hash, check_password_hash
# werkzeug comes with Flask - nothing extra to install

stored = generate_password_hash("hunter2")      # save THIS in the database
check_password_hash(stored, "hunter2")          # True
check_password_hash(stored, "wrong")            # False
```

**Why it matters:** if someone ever steals your database, they get a list of hashes, not passwords. And since people reuse passwords, you've protected their email and bank accounts too.

| ❌ Never | ✅ Always |
|---|---|
| Store the password as typed | `generate_password_hash()` |
| MD5 or plain SHA-256 | Werkzeug's default (scrypt), bcrypt or argon2 — deliberately slow |
| Email someone their password | A password-reset link |
| Log the password, even by accident | Log the email, never the password |

> **Slow is the point.** Fast hashes like MD5 let an attacker with a graphics card try billions of guesses per second. The hashes above are designed to be slow, so guessing is impractical ([[Security in practice]]).

---

## The user model

```python
from flask_login import UserMixin
from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column, relationship
from werkzeug.security import generate_password_hash, check_password_hash

class User(UserMixin, db.Model):                  # UserMixin adds what Flask-Login needs
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(120), unique=True, index=True)
    password_hash: Mapped[str] = mapped_column(String(256))     # the HASH, never the password
    tasks: Mapped[list["Task"]] = relationship(back_populates="owner", cascade="all, delete-orphan")

    def set_password(self, password: str) -> None:
        self.password_hash = generate_password_hash(password)

    def check_password(self, password: str) -> bool:
        return check_password_hash(self.password_hash, password)
```

**`UserMixin` gives your class four things Flask-Login asks about:**

| Property | Means |
|---|---|
| `is_authenticated` | Is this a real logged-in user? |
| `is_active` | Is the account enabled? |
| `is_anonymous` | Is this a visitor who isn't logged in? |
| `get_id()` | The id to store in the cookie |

---

## Setting up Flask-Login

```python
from flask_login import LoginManager

login_manager = LoginManager()
login_manager.init_app(app)
login_manager.login_view = "login"                   # where to send people who aren't logged in
login_manager.login_message_category = "is-warning"  # Bulma colour for "please log in"

@login_manager.user_loader                           # "given an id from the cookie, find the user"
def load_user(user_id):
    return db.session.get(User, int(user_id))        # runs on EVERY request from a logged-in user
```

> **`user_loader` is the bridge.** The cookie only holds an id. On every request Flask-Login calls this function to turn that id back into a `User` object — which becomes `current_user`.

---

## The forms

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
```

> **`PasswordField` renders `type="password"`** — the dots instead of letters — and never refills itself after an error, so the password isn't sent back into the page.

---

## Register

```python
from flask import flash, redirect, render_template, url_for
from flask_login import login_user, logout_user, login_required, current_user

@app.route("/register", methods=["GET", "POST"])
def register():
    if current_user.is_authenticated:                     # already logged in - nothing to do
        return redirect(url_for("index"))

    form = RegisterForm()
    if form.validate_on_submit():
        email = form.email.data.strip().lower()           # "Huz@Mail.com " and "huz@mail.com" are the same person

        if db.session.scalar(db.select(User).filter_by(email=email)):
            flash("That email is already registered.", "is-danger")
            return render_template("register.html", form=form)

        user = User(email=email)
        user.set_password(form.password.data)             # hash it
        db.session.add(user)
        db.session.commit()

        flash("Account created — please log in.", "is-success")
        return redirect(url_for("login"))

    return render_template("register.html", form=form)
```

> **Always lower-case and trim emails before storing and before looking them up.** Otherwise `Huz@mail.com` and `huz@mail.com` become two separate accounts.

## Log in

```python
from flask import flash, redirect, render_template, request, url_for
from flask_login import current_user, login_user
from urllib.parse import urlsplit

@app.route("/login", methods=["GET", "POST"])
def login():
    if current_user.is_authenticated:
        return redirect(url_for("index"))

    form = LoginForm()
    if form.validate_on_submit():
        user = db.session.scalar(db.select(User).filter_by(email=form.email.data.strip().lower()))

        if user is None or not user.check_password(form.password.data):
            flash("Wrong email or password.", "is-danger")      # deliberately vague - see below
            return render_template("login.html", form=form)

        login_user(user, remember=form.remember.data)           # sets the session cookie

        next_page = request.args.get("next")                    # where they were trying to go
        if not next_page or urlsplit(next_page).netloc != "":   # only allow pages on THIS site
            next_page = url_for("index")
        return redirect(next_page)

    return render_template("login.html", form=form)
```

> **Say *"wrong email or password"*, not *"no account with that email"*.** The specific version tells an attacker which emails have accounts on your site.

> ⚠️ **Check the `next` parameter before redirecting to it.** When Flask-Login bounces someone to the login page, it adds `?next=/tasks` so you can send them back afterwards. An attacker can craft `?next=https://evil-site.com` and use your login page to redirect people to a fake copy. The `netloc` check only allows paths on your own site — this is called an **open redirect**.

## Log out

```python
from flask import flash, redirect, url_for
from flask_login import logout_user

@app.post("/logout")                   # POST, not GET
@login_required
def logout():
    logout_user()                      # delete the session cookie
    flash("Logged out.", "is-info")
    return redirect(url_for("login"))
```

```html
<!-- in the navbar -->
<form method="post" action="{{ url_for('logout') }}">
  <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
  <button class="button is-light is-small">Log out</button>
</form>
```

> **Why POST for logout?** A GET logout link can be triggered by any other site — an `<img src="yoursite.com/logout">` logs your users out without them clicking anything. Annoying rather than dangerous, but POST costs nothing.

---

## Protecting pages

```python
from flask import render_template
from flask_login import current_user

@app.route("/tasks")
@login_required                     # not logged in? -> redirected to login_view, with ?next=/tasks
def index():
    tasks = db.session.scalars(
        db.select(Task).filter_by(user_id=current_user.id)   # ONLY this user's tasks
    ).all()
    return render_template("index.html", tasks=tasks)
```

> ⚠️ **Decorator order matters.** `@app.route` must be on top, `@login_required` underneath. The other way round, the route is registered *before* it's protected — and the page is wide open.

### `current_user` — who's asking

Available in every route and every template:

```python
from flask_login import current_user

current_user.is_authenticated        # True if logged in
current_user.id
current_user.email
```

```html
{% if current_user.is_authenticated %}
  <span class="navbar-item">{{ current_user.email }}</span>
{% else %}
  <a class="navbar-item" href="{{ url_for('login') }}">Log in</a>
{% endif %}
```

---

## ⚠️ The check everyone forgets: is this *theirs*?

`@login_required` checks that *someone* is logged in. **It does not check that the thing they're asking for belongs to them.**

```python
# BROKEN - any logged-in user can edit ANY task by changing the number in the URL
@app.route("/tasks/<int:task_id>/edit", methods=["GET", "POST"])
@login_required
def edit_task(task_id):
    task = db.get_or_404(Task, task_id)
    ...
```

User A logs in, visits `/tasks/41/edit`, changes it to `/tasks/42/edit` — and edits user B's task.

**Fix it once, with a helper, and use it everywhere:**

```python
from flask_login import current_user
from flask import abort

def get_own_task(task_id: int) -> Task:
    task = db.get_or_404(Task, task_id)
    if task.user_id != current_user.id:
        abort(404)            # 404, not 403 - don't even confirm the task exists
    return task

@app.route("/tasks/<int:task_id>/edit", methods=["GET", "POST"])
@login_required
def edit_task(task_id):
    task = get_own_task(task_id)          # safe
    ...
```

> **This bug has a name — IDOR — and it's one of the most common real-world security holes.** Test for it the lazy way: log in as one user, copy a task URL, log in as another user in a private window, paste it. If you can see it, you have the bug ([[Security in practice]]).

**When creating a record, set the owner in Python — never from the form:**

```python
from flask_login import current_user

db.session.add(Task(title=form.title.data, user_id=current_user.id))   # from the session, trusted
```

---

## The templates

`templates/login.html`:

```html
{% extends "base.html" %}
{% from "_formhelpers.html" import render_field %}
{% block title %}Log in{% endblock %}

{% block content %}
<div class="columns is-centered">
  <div class="column is-5">
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
        No account? <a href="{{ url_for('register') }}">Register</a>
      </p>
    </div>
  </div>
</div>
{% endblock %}
```

`register.html` is the same with the register form's fields. The `render_field` macro is in [[Flask - forms and user input]].

---

## "Remember me" and how long sessions last

```python
from datetime import timedelta

app.config["REMEMBER_COOKIE_DURATION"] = timedelta(days=14)   # "remember me" lasts 2 weeks
app.config["SESSION_COOKIE_HTTPONLY"] = True     # JavaScript can't read the cookie (default)
app.config["SESSION_COOKIE_SAMESITE"] = "Lax"    # blocks most cross-site request tricks
app.config["SESSION_COOKIE_SECURE"] = True       # HTTPS only - turn ON in production
app.config["REMEMBER_COOKIE_SECURE"] = True
```

> ⚠️ **`SESSION_COOKIE_SECURE = True` breaks login on `http://localhost`** because the cookie is only sent over HTTPS. Set it in your production config only ([[Flask - project structure and blueprints]]).

---

## Admin-only pages

```python
from flask import abort
from flask_login import current_user
from functools import wraps

def admin_required(view):
    @wraps(view)
    def wrapped(*args, **kwargs):
        if not current_user.is_authenticated or not current_user.is_admin:
            abort(404)
        return view(*args, **kwargs)
    return wrapped

@app.route("/admin")
@login_required
@admin_required
def admin_dashboard():
    ...
```

Add an `is_admin: Mapped[bool] = mapped_column(default=False)` column to `User`, and set it by hand in the database or a Flask shell — **never** from a form.

---

## Stopping password guessing

Without a limit, someone can try thousands of passwords a minute against your login form.

```bash
uv add flask-limiter
```

```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(get_remote_address, app=app)

@app.route("/login", methods=["GET", "POST"])
@limiter.limit("5 per minute")          # 5 attempts a minute per IP address
def login():
    ...
```

---

## Common mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Storing the password as typed | **Every password leaked if the DB is** | `generate_password_hash` |
| `@login_required` above `@app.route` | **Page isn't protected** | `@app.route` on top |
| Login check but no ownership check | **IDOR — users see each other's data** | `get_own_task()` helper |
| `user_id` taken from the form | Users can create records as someone else | Use `current_user.id` |
| Redirecting to `next` unchecked | **Open redirect** to a fake site | Check `netloc` |
| "No account with that email" | Reveals which emails are registered | "Wrong email or password" |
| Emails not lower-cased | Duplicate accounts | `.strip().lower()` |
| `SESSION_COOKIE_SECURE` on localhost | Login silently doesn't stick | Production config only |
| Secret data in the session | Readable by the user | Store only the user id |
| Hardcoded `SECRET_KEY` in git | Anyone can forge a login cookie | Environment variable |
| No rate limit on login | Unlimited password guessing | Flask-Limiter |

## Related

[[Flask]] · [[Flask - forms and user input]] · [[Flask - connecting to a SQL database]] · [[Flask - building a JSON API]] · [[Flask reference]] · [[Security in practice]] · [[user registration - login system]]
