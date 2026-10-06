---
tags: [flask, forms, wtforms, validation, csrf, guide]
---

# Flask — forms and user input

**Guide 5 of 8.** Getting data *from* the user: how HTML forms send data, reading it in Flask, checking it's valid, showing errors, and protecting the form from attack.

Hub: [[Flask]] · Previous: [[Flask - connecting to a SQL database]] · Next: [[Flask - user accounts and login]]

---

## How a form actually works

An HTML form is a set of boxes plus a button. When you click the button, the browser **packs up everything you typed and sends it** to the URL in `action`:

```html
<form method="post" action="/tasks">       <!-- WHERE to send it, and HOW -->
  <input name="title">                     <!-- name= is the label Flask will read it by -->
  <button>Add</button>
</form>
```

Type *Buy milk*, click Add, and the browser sends a **POST** request to `/tasks` carrying `title=Buy milk`.

```
browser                                         Flask
───────                                         ─────
[ Buy milk ] [Add]  ── POST /tasks ──────────►  request.form["title"]  ->  "Buy milk"
                       title=Buy milk
```

> ⚠️ **No `name=` attribute, no data.** An input without a `name` is simply not sent. This is the most common "my form sends nothing" bug.

---

## GET vs POST — which one?

| | GET | POST |
|---|---|---|
| Data goes in | The URL: `/search?q=milk` | The hidden body of the request |
| Visible in the address bar | Yes | No |
| Can be bookmarked / shared | Yes | No |
| Use for | **Searching, filtering** — reading | **Creating, changing, deleting** — writing |

```html
<form method="get" action="/search">     <!-- search: the result can be bookmarked -->
  <input name="q">
</form>

<form method="post" action="/tasks">     <!-- anything that CHANGES data -->
  <input name="title">
</form>
```

> **The rule: if submitting it changes something, use POST.** Never put passwords in a GET form — they'd sit in the URL, the browser history and the server logs.

---

## Reading form data in Flask

```python
from flask import request, render_template

@app.route("/tasks", methods=["GET", "POST"])     # accept both
def tasks():
    if request.method == "POST":                  # the form was submitted
        title = request.form["title"]             # CRASHES (400) if the field is missing
        title = request.form.get("title", "")     # returns "" if missing - safer
        ...
    return render_template("tasks.html")          # a normal visit: show the page
```

**Everything the request can carry:**

| Where | How you read it | Example |
|---|---|---|
| Form fields (POST) | `request.form.get("title")` | `title=Buy milk` |
| URL query string (GET) | `request.args.get("q")` | `/search?q=milk` |
| Numbers from the URL | `request.args.get("page", 1, type=int)` | `?page=3` → `3` |
| Checkboxes (several) | `request.form.getlist("tags")` | every ticked box |
| Uploaded files | `request.files.get("photo")` | an image |
| JSON (from JavaScript) | `request.get_json()` | `{"title": "..."}` |

> ⚠️ **Everything arrives as text.** `request.form.get("qty")` gives you `"5"`, a string — not the number `5`. Convert it yourself, and handle the case where it isn't a number.

**Checkboxes are awkward:**

```html
<input type="checkbox" name="done" value="yes">
```

```python
from flask import request

done = "done" in request.form       # ticked -> the key exists; unticked -> NOT SENT AT ALL
```

> An unticked checkbox sends **nothing**, not `"off"`. Test whether the key exists.

---

## Always redirect after a successful POST

```python
from flask import redirect, request, url_for

@app.post("/tasks")
def create_task():
    db.session.add(Task(title=request.form["title"]))
    db.session.commit()
    return redirect(url_for("index"))        # DON'T render_template here
```

**Why?** If you `render_template` straight after a POST, the browser's last request is still that POST. When the user presses **refresh**, the browser re-sends it — and your task gets created **twice**. (That's the *"Confirm form resubmission"* pop-up.)

Redirecting makes the browser do a fresh GET, so refreshing is harmless. This is called **Post/Redirect/Get**.

```
POST /tasks  ──►  save it  ──►  302 redirect to /  ──►  browser does GET /   (refresh-safe)
```

> **Exception:** when validation *fails*, you do `render_template` directly — so the page can show the errors and keep what the user typed.

---

## Flash messages — "Task saved!"

A **flash** message is shown once, on the next page, then disappears:

```python
from flask import flash, redirect, url_for

flash("Task added.", "is-success")          # the second argument is a category
flash("Title can't be empty.", "is-danger")
return redirect(url_for("index"))
```

```html
{% with messages = get_flashed_messages(with_categories=true) %}
  {% for category, message in messages %}
    <div class="notification {{ category }}">{{ message }}</div>
  {% endfor %}
{% endwith %}
```

> **Using Bulma class names (`is-success`, `is-danger`) as the category** means the message is coloured correctly with no extra code — see [[Flask - styling with Bulma]].

> ⚠️ **Flash needs a `SECRET_KEY`.** Messages are stored in the session cookie, which is signed with it. Without one: *"The session is unavailable because no secret key was set"*.

---

## Validating by hand

**Never trust what arrives.** Users mistype, and attackers deliberately send junk.

```python
from flask import flash, redirect, render_template, request, url_for

@app.route("/tasks/new", methods=["GET", "POST"])
def new_task():
    errors = {}
    if request.method == "POST":
        title = request.form.get("title", "").strip()             # remove stray spaces
        qty_raw = request.form.get("qty", "1")

        if not title:
            errors["title"] = "Please enter a title."
        elif len(title) > 200:
            errors["title"] = "Keep it under 200 characters."

        try:
            qty = int(qty_raw)
            if qty < 1:
                errors["qty"] = "Must be at least 1."
        except ValueError:
            errors["qty"] = "Must be a whole number."

        if not errors:                                            # all good - save and redirect
            db.session.add(Task(title=title, qty=qty))
            db.session.commit()
            flash("Task added.", "is-success")
            return redirect(url_for("index"))

    return render_template("new_task.html", errors=errors)      # show the form, with any errors
```

```html
<div class="field">
  <label class="label">Title</label>
  <div class="control">
    <input class="input {{ 'is-danger' if errors.title }}" name="title"
           value="{{ request.form.get('title', '') }}">       <!-- keep what they typed -->
  </div>
  {% if errors.title %}<p class="help is-danger">{{ errors.title }}</p>{% endif %}
</div>
```

> **Refill the fields on an error** (`value="{{ request.form.get('title', '') }}"`). Making someone retype a whole form because one box was wrong is the fastest way to annoy them.

> **The HTML `required` attribute is not validation.** It's a convenience for honest users. Anyone can bypass it — so always validate again in Python.

This works, but it gets repetitive fast. That's what Flask-WTF is for.

---

## Flask-WTF — forms done properly

**Flask-WTF** lets you describe a form once, in Python, and gives you validation, error messages and security protection for free.

```bash
uv add flask-wtf email-validator      # email-validator is needed for the Email() check
```

### 1. Describe the form

`forms.py`:

```python
from flask_wtf import FlaskForm
from wtforms import StringField, TextAreaField, IntegerField, SelectField, BooleanField, SubmitField
from wtforms.validators import DataRequired, Length, NumberRange, Email, Optional

class TaskForm(FlaskForm):
    title = StringField("Title", validators=[
        DataRequired(message="Please enter a title."),     # must not be empty
        Length(max=200),                                   # at most 200 characters
    ])
    notes = TextAreaField("Notes", validators=[Optional()])   # may be left blank
    qty = IntegerField("Quantity", default=1, validators=[NumberRange(min=1)])
    priority = SelectField("Priority", choices=[("low", "Low"), ("medium", "Medium"), ("high", "High")])
    done = BooleanField("Done")
    submit = SubmitField("Save")
```

### 2. Use it in the route

```python
from flask import flash, redirect, render_template, url_for
from forms import TaskForm

@app.route("/tasks/new", methods=["GET", "POST"])
def new_task():
    form = TaskForm()                         # reads request.form automatically on POST
    if form.validate_on_submit():             # True only if POSTed AND every check passed
        task = Task(
            title=form.title.data,            # .data holds the cleaned, converted value
            notes=form.notes.data,
            qty=form.qty.data,                # already an int - no conversion needed
        )
        db.session.add(task)
        db.session.commit()
        flash("Task added.", "is-success")
        return redirect(url_for("index"))
    return render_template("new_task.html", form=form)   # first visit OR failed validation
```

**`validate_on_submit()` does three jobs at once:** checks the request is a POST, checks the security token, and runs every validator. If any fail, it returns `False` and `form.title.errors` holds the messages.

### 3. Draw it in the template

```html
<form method="post" novalidate>
  {{ form.hidden_tag() }}                          <!-- the security token - REQUIRED -->

  <div class="field">
    {{ form.title.label(class="label") }}          <!-- <label>Title</label> -->
    <div class="control">
      {{ form.title(class="input") }}              <!-- <input name="title" ...> -->
    </div>
    {% for error in form.title.errors %}
      <p class="help is-danger">{{ error }}</p>
    {% endfor %}
  </div>

  {{ form.submit(class="button is-primary") }}
</form>
```

> **`novalidate`** turns off the browser's own pop-up checks, so your Flask error messages show consistently instead of fighting the browser's.

### 4. One macro for every field

Writing that `field / label / control / errors` block for every input is tedious. Write it once:

`templates/_formhelpers.html`:

```html
{% macro render_field(field, placeholder="") %}
  <div class="field">
    {{ field.label(class="label") }}
    <div class="control">
      {% if field.type == "TextAreaField" %}
        {{ field(class="textarea" ~ (" is-danger" if field.errors else ""), placeholder=placeholder) }}
      {% elif field.type == "SelectField" %}
        <div class="select">{{ field() }}</div>
      {% elif field.type == "BooleanField" %}
        <label class="checkbox">{{ field() }} {{ field.label.text }}</label>
      {% else %}
        {{ field(class="input" ~ (" is-danger" if field.errors else ""), placeholder=placeholder) }}
      {% endif %}
    </div>
    {% for error in field.errors %}
      <p class="help is-danger">{{ error }}</p>
    {% endfor %}
  </div>
{% endmacro %}
```

> **`~` joins strings in Jinja.** `"input" ~ " is-danger"` becomes `"input is-danger"` — red border when there's an error.

Now any form is a few lines:

```html
{% extends "base.html" %}
{% from "_formhelpers.html" import render_field %}

{% block content %}
<div class="columns is-centered">
  <div class="column is-6">
    <form method="post" class="box" novalidate>
      {{ form.hidden_tag() }}
      {{ render_field(form.title, "What needs doing?") }}
      {{ render_field(form.notes) }}
      {{ render_field(form.qty) }}
      {{ render_field(form.priority) }}
      {{ render_field(form.done) }}
      {{ form.submit(class="button is-primary is-fullwidth") }}
    </form>
  </div>
</div>
{% endblock %}
```

### Editing — filling a form from the database

```python
from flask import flash, redirect, render_template, url_for

@app.route("/tasks/<int:task_id>/edit", methods=["GET", "POST"])
def edit_task(task_id):
    task = db.get_or_404(Task, task_id)
    form = TaskForm(obj=task)                  # pre-fill every field from the Task object
    if form.validate_on_submit():
        form.populate_obj(task)                # copy every field back onto the Task
        db.session.commit()
        flash("Saved.", "is-success")
        return redirect(url_for("index"))
    return render_template("edit_task.html", form=form, task=task)
```

> **`obj=task` fills the form; `populate_obj(task)` saves it back.** Two lines for a whole edit page.

> ⚠️ **`populate_obj` copies *every* field in the form onto the object.** Never put a field like `user_id` or `is_admin` in a form you use it with — a user could post a different value and take over someone else's record, or make themselves an admin.

### Custom validation

```python
from flask_wtf import FlaskForm
from wtforms import StringField
from wtforms.validators import ValidationError, DataRequired, Email

class RegisterForm(FlaskForm):
    email = StringField("Email", validators=[DataRequired(), Email()])

    def validate_email(self, field):              # validate_<fieldname> runs automatically
        if db.session.scalar(db.select(User).filter_by(email=field.data.lower())):
            raise ValidationError("That email is already registered.")
```

### Validators you'll use

| Validator | Checks |
|---|---|
| `DataRequired()` | Not empty |
| `Optional()` | May be blank — skip other checks if it is |
| `Length(min=, max=)` | Text length |
| `NumberRange(min=, max=)` | A number's range |
| `Email()` | Looks like an email — needs `email-validator` |
| `EqualTo("password")` | Matches another field — "confirm password" |
| `Regexp(r"...")` | Matches a pattern |
| `URL()` | Looks like a web address |
| `AnyOf([...])` | One of a fixed list |

---

## CSRF — what that hidden token is for

**The attack:** you're logged in to your task app. You visit a nasty website, which contains a hidden form that submits to *your* app's `/tasks/5/delete`. Your browser helpfully sends your login cookie along — and your task is deleted, without you clicking anything in your app.

That's **CSRF** — *Cross-Site Request Forgery*.

**The defence:** every form on your site includes a secret random token that only your site knows. The nasty site can't read it, so its forged form can't include it, and Flask rejects the request.

```html
{{ form.hidden_tag() }}      <!-- adds <input type="hidden" name="csrf_token" value="..."> -->
```

**Protect every form, including ones that aren't FlaskForms** — turn it on for the whole app:

```python
from flask_wtf.csrf import CSRFProtect
csrf = CSRFProtect(app)       # now EVERY POST must carry a valid token
```

Then plain HTML forms (a delete button, a logout button) add the token like this:

```html
<form method="post" action="{{ url_for('delete_task', task_id=task.id) }}">
  <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
  <button class="button is-danger is-small">Delete</button>
</form>
```

> ⚠️ **"The CSRF token is missing" means you forgot the token in that form.** Add `{{ form.hidden_tag() }}` or the `csrf_token()` hidden input. Don't disable CSRF to make the error go away.

> **CSRF needs a `SECRET_KEY`** — the token is signed with it.

---

## Uploading files

```html
<form method="post" enctype="multipart/form-data">    <!-- REQUIRED for file uploads -->
  <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
  <div class="file">
    <label class="file-label">
      <input class="file-input" type="file" name="photo" accept="image/*">
      <span class="file-cta"><span class="file-label">Choose a file…</span></span>
    </label>
  </div>
  <button class="button is-primary">Upload</button>
</form>
```

```python
from flask import flash, redirect, request, url_for
import os
from werkzeug.utils import secure_filename

app.config["MAX_CONTENT_LENGTH"] = 5 * 1024 * 1024        # reject anything over 5 MB
ALLOWED = {"png", "jpg", "jpeg", "gif"}

@app.post("/upload")
def upload():
    file = request.files.get("photo")
    if not file or file.filename == "":
        flash("No file chosen.", "is-danger")
        return redirect(url_for("index"))
    ext = file.filename.rsplit(".", 1)[-1].lower()
    if ext not in ALLOWED:
        flash("Images only.", "is-danger")
        return redirect(url_for("index"))
    name = secure_filename(file.filename)                  # strips ../ and other tricks
    file.save(os.path.join(app.instance_path, "uploads", name))
    flash("Uploaded.", "is-success")
    return redirect(url_for("index"))
```

> ⚠️ **Without `enctype="multipart/form-data"`, `request.files` is empty.** The browser sends only the filename, not the file.

> ⚠️ **Always `secure_filename()`.** A filename of `../../app.py` would otherwise overwrite your own code. And never trust the extension alone for anything security-critical — it's just part of the name.

---

## Common mistakes

| Mistake | What you'll see | Fix |
|---|---|---|
| Input has no `name=` | The field never arrives | Add `name="..."` |
| Route lacks `methods=["POST"]` | **405 Method Not Allowed** | Add it |
| `request.form["x"]` on a missing field | **400 Bad Request** | `request.form.get("x")` |
| `render_template` after a successful POST | Duplicate rows on refresh | `redirect(...)` |
| Number compared as text | `"10" < "9"` is `True` | `int(...)` or `IntegerField` |
| Unticked checkbox read as `"off"` | Always seems ticked | Check `"done" in request.form` |
| No `SECRET_KEY` | Session / flash / CSRF errors | Set one — see guide 8 |
| No CSRF token in a form | *"The CSRF token is missing"* | `{{ form.hidden_tag() }}` |
| `user_id` in a form with `populate_obj` | **Users can edit others' records** | Set it in the route, never from the form |
| No `enctype` on an upload form | Empty `request.files` | `enctype="multipart/form-data"` |
| Trusting HTML `required` | Bypassed easily | Validate in Python too |

## Related

[[Flask]] · [[Flask - connecting to a SQL database]] · [[Flask - user accounts and login]] · [[Flask - styling with Bulma]] · [[Flask reference]] · [[HTML]] · [[Security in practice]] · [[Pydantic]]
