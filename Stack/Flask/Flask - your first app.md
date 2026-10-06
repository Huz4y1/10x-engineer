---
tags: [flask, python, web, backend, guide]
---

# Flask — your first app

**Guide 1 of 8.** Installing Flask on Windows and WSL, the smallest app that works, and what actually happens when someone visits a page.

Hub: [[Flask]] · Next: [[Flask - templates with Jinja]] · Everything: [[Flask reference]]

---

## What Flask is, in simple words

When you type a web address into a browser, your browser sends a **request** to a computer somewhere: *"please give me the page at `/tasks`"*. That computer runs a program that works out what to send back — a **response**, usually a web page.

**Flask is a Python library for writing that program.**

You write a Python function, and you tell Flask *"when someone asks for `/tasks`, run this function and send back whatever it returns"*. That's the whole idea.

```
  Browser                         Your computer (running Flask)
  ───────                         ────────────────────────────
  "GET /tasks" ─────────────────► Flask looks up which function handles /tasks
                                  runs it
  <html>...page...</html> ◄────── sends back what the function returned
```

**Why Flask?** It's small. It gives you routing (matching URLs to functions) and templates (building HTML pages), and then gets out of your way. Everything else — databases, logins, forms — you add as you need it, one piece at a time. That makes it the best framework for **learning how the web actually works**, because nothing is hidden.

| | Flask | [[Django]] | [[FastAPI reference\|FastAPI]] |
|---|---|---|---|
| Size | Small — add pieces yourself | Big — everything included | Small |
| Best for | Websites with pages, small-to-medium apps, learning | Big sites with admin panels and many users | JSON APIs for other programs |
| Returns | HTML pages (or JSON) | HTML pages | JSON |
| Learning curve | Gentle | Steeper | Gentle, but needs type hints |

---

## Installing it

### Step 1 — Have Python

```bash
python --version          # Windows
python3 --version         # WSL / Linux
```

> **On Windows, if `python` opens the Microsoft Store**, Python isn't really installed. Get it from python.org and tick **"Add python.exe to PATH"**, or run `winget install Python.Python.3.12`.

### Step 2 — Make a project folder

| | Windows (PowerShell) | WSL |
|---|---|---|
| Go somewhere sensible | `cd ~\Documents` | `cd ~` |
| Make the folder | `mkdir tasktracker; cd tasktracker` | `mkdir tasktracker && cd tasktracker` |

> ⚠️ **In WSL, work in `~/`, never `/mnt/c/...`.** The Windows drive is dramatically slower from inside WSL, and Flask's auto-reload often doesn't notice your edits there. See [[Setting up a dev machine]].

### Step 3 — Install Flask

**With uv (recommended — same command everywhere):**

```bash
uv init                  # creates pyproject.toml and a .venv
uv add flask             # installs Flask into that .venv
```

**Without uv — a virtual environment by hand:**

| | Windows (PowerShell) | WSL / Linux |
|---|---|---|
| Create | `python -m venv .venv` | `python3 -m venv .venv` |
| Activate | `.venv\Scripts\Activate.ps1` | `source .venv/bin/activate` |
| Install | `pip install flask` | `pip install flask` |

> **A virtual environment (`.venv`) is a private box of libraries for this one project.** Without it, every project on your computer shares one set of libraries and they eventually conflict. When activated, you'll see `(.venv)` at the start of your prompt.

> ⚠️ **PowerShell may refuse to run `Activate.ps1`** with *"running scripts is disabled"*. Fix it once: `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`.

**Check it worked:**

```bash
python -c "import flask; print(flask.__version__)"
```

---

## The smallest app that works

Make a file called `app.py`:

```python
from flask import Flask          # import the Flask class

app = Flask(__name__)            # create the application. __name__ tells Flask where your files are

@app.route("/")                  # "when someone visits /  ..."
def home():                      # "...run this function..."
    return "Hello, world!"       # "...and send back this text"
```

Run it:

```bash
flask --app app run --debug
#       ^^^^^^^ which file   ^^^^^^^ restart on save + show helpful error pages
```

Open **http://127.0.0.1:5000** in your browser. You'll see *Hello, world!*

> **In WSL, the browser is on Windows** — just paste `http://127.0.0.1:5000` into it. WSL forwards the port for you. If that ever fails, run with `--host 0.0.0.0`.

**What each line did:**

| Line | Plain English |
|---|---|
| `app = Flask(__name__)` | Build the web app object. Everything else attaches to it |
| `@app.route("/")` | A **decorator** — it registers the function below as the handler for that URL |
| `def home():` | A normal Python function. Its name doesn't matter to the browser |
| `return "Hello, world!"` | Whatever you return is sent to the browser |

To stop the server: **Ctrl+C** in the terminal.

---

## Adding more pages (routes)

```python
@app.route("/")
def home():
    return "Home page"

@app.route("/about")                     # http://127.0.0.1:5000/about
def about():
    return "About page"

@app.route("/hello/<name>")              # <name> is a VARIABLE part of the URL
def hello(name):                         # Flask passes it in as an argument
    return f"Hello, {name}!"             # /hello/Huz  ->  "Hello, Huz!"

@app.route("/task/<int:task_id>")        # <int:...> only matches numbers, and converts to int
def show_task(task_id):
    return f"Task number {task_id}"      # /task/7 works, /task/abc is a 404
```

| Converter | Matches | Example |
|---|---|---|
| `<name>` or `<string:name>` | Any text without a slash | `/hello/Huz` |
| `<int:id>` | Whole numbers | `/task/7` |
| `<float:x>` | Decimals | `/price/9.99` |
| `<path:p>` | Text *including* slashes | `/files/a/b/c.txt` |

> **Use `<int:...>` for IDs.** You get a real integer, and anything that isn't a number becomes a 404 automatically instead of crashing your code.

---

## What a request actually is

Every request has a **method** — the *kind* of thing the browser wants to do:

| Method | Meaning | When the browser uses it |
|---|---|---|
| **GET** | "Give me something" | Typing a URL, clicking a link |
| **POST** | "Here's some data, do something with it" | Submitting a form |
| PUT / PATCH | "Update this" | JavaScript / API calls |
| DELETE | "Remove this" | JavaScript / API calls |

By default a Flask route only accepts **GET**. To accept form submissions too:

```python
from flask import request

@app.route("/contact", methods=["GET", "POST"])   # accept both
def contact():
    if request.method == "POST":                  # the form was submitted
        return f"Thanks, {request.form['name']}"  # read a field from the form
    return "Show the empty form here"             # a normal visit
```

> **Forms get their own guide** — [[Flask - forms and user input]]. For now just know: **GET = asking, POST = sending**.

---

## Status codes — how the server says how it went

Every response carries a three-digit number:

| Code | Means | You'll see it when |
|---|---|---|
| **200** | OK | Everything worked |
| **302** | Go somewhere else | After a `redirect()` |
| **404** | Not found | The URL has no route, or the item doesn't exist |
| **405** | Method not allowed | You POSTed to a GET-only route |
| **500** | Server error | Your Python code crashed |

```python
from flask import abort

@app.route("/task/<int:task_id>")
def show_task(task_id):
    if task_id > 100:
        abort(404)                  # stop here and send a 404 page
    return f"Task {task_id}"
```

---

## Debug mode — your best friend, and a real danger

`--debug` does two things:

1. **Auto-reload** — save a file and the server restarts with your changes
2. **The debugger** — when your code crashes, the browser shows the exact line and the error, instead of a blank *Internal Server Error*

> ⚠️ **NEVER run debug mode on a server anyone else can reach.** The debug error page includes an interactive Python console. Anyone who can trigger an error can then run any code they like on your machine. It is strictly for your own laptop ([[Flask - deploying it]], [[Security in practice]]).

---

## The two ways to run it

```bash
flask --app app run --debug        # the Flask command - recommended
```

or add this to the bottom of `app.py` and run `python app.py`:

```python
if __name__ == "__main__":         # only true when you run THIS file directly
    app.run(debug=True)
```

Both work. The `flask` command is the standard, and it's what the rest of these guides use.

**Useful flags:**

```bash
flask --app app run --debug --port 8000        # a different port
flask --app app run --debug --host 0.0.0.0     # reachable from other machines / WSL->Windows
flask --app app routes                         # list every URL your app has
```

> **`flask --app app routes` is great for debugging a 404.** It prints every route Flask knows about. If yours isn't in the list, the decorator or the file isn't being loaded.

---

## Returning different things

```python
from flask import redirect, url_for, jsonify

@app.route("/old-page")
def old_page():
    return redirect(url_for("home"))          # send the browser to another route

@app.route("/api/status")
def status():
    return jsonify({"status": "ok"})          # JSON, for JavaScript or other programs

@app.route("/teapot")
def teapot():
    return "I'm a teapot", 418                # text AND a custom status code
```

### `url_for` — never type a URL by hand

```python
from flask import url_for

url_for("home")                        # -> "/"
url_for("hello", name="Huz")           # -> "/hello/Huz"
url_for("show_task", task_id=7)        # -> "/task/7"
```

`url_for` takes the **function name**, not the URL. If you later change `@app.route("/about")` to `@app.route("/about-us")`, every `url_for("about")` updates automatically. Hand-typed links would all break.

---

## Common mistakes

| Mistake | What you'll see | Fix |
|---|---|---|
| Forgot to activate the venv | `No module named flask` | Activate it, or use `uv run flask ...` |
| Two functions with the same name | `AssertionError: View function mapping is overwriting` | Every route function needs a unique name |
| Submitted a form to a GET-only route | **405 Method Not Allowed** | Add `methods=["GET", "POST"]` |
| Edits don't show up | Old page keeps appearing | Run with `--debug`; in WSL, move out of `/mnt/c/` |
| Port already in use | `Address already in use` | Another server is still running — Ctrl+C it, or use `--port 5001` |
| Named the file `flask.py` | `cannot import name 'Flask'` | Your file hides the real library — rename it |
| Debug mode on a public server | **Anyone can run code on your machine** | Only ever on your laptop |

## Related

[[Flask]] · [[Flask - templates with Jinja]] · [[Flask reference]] · [[Python]] · [[Setting up a dev machine]] · [[Networking reference]]
