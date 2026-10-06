---
tags: [flask, api, json, rest, backend, guide]
---

# Flask — building a JSON API

**Guide 7 of 8.** The backend side: returning data instead of pages, so JavaScript, a phone app or another program can use your Flask app.

Hub: [[Flask]] · Previous: [[Flask - user accounts and login]] · Next: [[Flask - project structure and blueprints]]

---

## Pages vs an API

So far Flask has sent back **HTML pages** — built for a person to look at. An **API** sends back **data** — built for a program to use.

```
Web page route:   GET /tasks      ->  <html><table><tr><td>Buy milk</td>...    (for a human)
API route:        GET /api/tasks  ->  [{"id": 1, "title": "Buy milk", "done": false}]   (for a program)
```

That data format is **JSON** — text that looks almost exactly like a Python list of dicts, and that every programming language can read.

**Why you'd want one:**
- A button that updates the page **without reloading** it (JavaScript calls the API)
- A **phone app** that shares your database ([[Expo]])
- A **dashboard** that reads your data ([[Streamlit]])
- **Another program** — a script, a scheduled job — reading or writing your data

> **The same Flask app can do both.** Your pages at `/tasks` and your API at `/api/tasks` share one database, one set of models, one login system.

---

## Returning JSON

```python
from flask import jsonify

@app.get("/api/status")                  # @app.get = a route that only accepts GET
def status():
    return {"status": "ok", "version": 1}      # returning a dict -> Flask sends JSON automatically

@app.get("/api/numbers")
def numbers():
    return jsonify([1, 2, 3])            # a LIST needs jsonify() - only dicts convert automatically
```

> **Return a dict and Flask converts it to JSON for you.** Lists need `jsonify()`.

### Turning a model into JSON

A `Task` object can't be sent as JSON directly — Flask doesn't know which fields you want to share. Give the model a method:

```python
class Task(db.Model):
    ...
    def to_dict(self) -> dict:
        return {
            "id": self.id,
            "title": self.title,
            "done": self.done,
            "created_at": self.created_at.isoformat(),    # dates must become text
        }
```

```python
from flask import jsonify
from flask_login import current_user

@app.get("/api/tasks")
@login_required
def api_list_tasks():
    tasks = db.session.scalars(db.select(Task).filter_by(user_id=current_user.id)).all()
    return jsonify([t.to_dict() for t in tasks])
```

> ⚠️ **Choose the fields deliberately.** Never return a whole model automatically — the day someone adds a `password_hash` or `is_admin` column, it quietly starts appearing in your API. `to_dict()` is a list of what you *intend* to share.

---

## A complete REST API

**REST** is just a naming convention for APIs. One URL per *thing*, and the HTTP method says what you're doing to it:

| Method | URL | Does | Returns |
|---|---|---|---|
| `GET` | `/api/tasks` | List all tasks | 200 + a list |
| `POST` | `/api/tasks` | Create a task | **201** + the new task |
| `GET` | `/api/tasks/5` | Get one task | 200 + the task, or 404 |
| `PATCH` | `/api/tasks/5` | Change some fields | 200 + the updated task |
| `DELETE` | `/api/tasks/5` | Delete it | **204**, no body |

```python
from flask_login import current_user
from flask import request, jsonify, abort

def get_own_task(task_id):
    task = db.get_or_404(Task, task_id)
    if task.user_id != current_user.id:
        abort(404)                                      # not yours = doesn't exist
    return task

# ---- LIST ----
@app.get("/api/tasks")
@login_required
def api_list():
    done = request.args.get("done")                     # optional filter: /api/tasks?done=true
    query = db.select(Task).filter_by(user_id=current_user.id)
    if done is not None:
        query = query.filter_by(done=(done.lower() == "true"))
    tasks = db.session.scalars(query.order_by(Task.id)).all()
    return jsonify([t.to_dict() for t in tasks])

# ---- CREATE ----
@app.post("/api/tasks")
@login_required
def api_create():
    data = request.get_json(silent=True) or {}          # silent=True: None instead of crashing on bad JSON
    title = str(data.get("title", "")).strip()
    if not title:
        return {"error": "title is required"}, 400      # a dict AND a status code
    if len(title) > 200:
        return {"error": "title must be 200 characters or fewer"}, 400

    task = Task(title=title, user_id=current_user.id)
    db.session.add(task)
    db.session.commit()
    return task.to_dict(), 201                          # 201 = Created

# ---- GET ONE ----
@app.get("/api/tasks/<int:task_id>")
@login_required
def api_get(task_id):
    return get_own_task(task_id).to_dict()

# ---- UPDATE ----
@app.patch("/api/tasks/<int:task_id>")
@login_required
def api_update(task_id):
    task = get_own_task(task_id)
    data = request.get_json(silent=True) or {}
    if "title" in data:                                 # only change what was sent
        title = str(data["title"]).strip()
        if not title:
            return {"error": "title cannot be empty"}, 400
        task.title = title
    if "done" in data:
        if not isinstance(data["done"], bool):
            return {"error": "done must be true or false"}, 400
        task.done = data["done"]
    db.session.commit()
    return task.to_dict()

# ---- DELETE ----
@app.delete("/api/tasks/<int:task_id>")
@login_required
def api_delete(task_id):
    db.session.delete(get_own_task(task_id))
    db.session.commit()
    return "", 204                                      # 204 = done, nothing to send back
```

### Status codes that matter for APIs

| Code | Means | Return it when |
|---|---|---|
| **200** OK | Worked | Normal success |
| **201** Created | Made something new | After a successful POST |
| **204** No Content | Worked, nothing to say | After a DELETE |
| **400** Bad Request | Your input is wrong | Missing or invalid fields |
| **401** Unauthorized | Who are you? | Not logged in |
| **403** Forbidden | I know you, and no | Logged in, not allowed |
| **404** Not Found | Doesn't exist | Wrong id — or not yours |
| **409** Conflict | Clashes with existing data | Duplicate email |
| **500** Server Error | Your code crashed | Never on purpose |

> **Status codes matter more in an API than on a web page.** A person reads the error message; a program usually only checks the number. Returning `200` with `{"error": ...}` means the caller thinks it worked.

---

## Errors as JSON, not HTML

By default, a 404 or 500 in Flask returns an **HTML** error page — useless to JavaScript, which expects JSON.

```python
from flask import request
from werkzeug.exceptions import HTTPException

@app.errorhandler(HTTPException)
def handle_http_error(e):
    if request.path.startswith("/api/"):                       # only for API routes
        return {"error": e.name, "detail": e.description}, e.code
    return e                                                   # normal pages keep HTML errors

@app.errorhandler(Exception)
def handle_crash(e):
    app.logger.exception("unhandled error")                    # full detail in YOUR logs
    if request.path.startswith("/api/"):
        return {"error": "Internal server error"}, 500         # nothing internal in the RESPONSE
    raise e
```

> ⚠️ **Never send the real error message or stack trace back to the caller.** It tells an attacker your file paths, library versions and sometimes your SQL. Log it; return something generic ([[Security in practice]]).

### Login for API routes

`@login_required` normally **redirects** to the login page — which, for a JavaScript caller, means it gets back an HTML page with a 200 status. Make API routes answer with a clean 401 instead:

```python
from flask import redirect, request, url_for

@login_manager.unauthorized_handler
def unauthorized():
    if request.path.startswith("/api/"):
        return {"error": "login required"}, 401
    return redirect(url_for("login", next=request.path))
```

---

## Calling your API from JavaScript

The classic reason to build an API inside a Flask site: ticking a task **without reloading the page**.

```html
<input type="checkbox" class="task-toggle" data-id="{{ task.id }}" {{ 'checked' if task.done }}>
<meta name="csrf-token" content="{{ csrf_token() }}">      <!-- put this in <head> -->
```

```javascript
const csrf = document.querySelector('meta[name="csrf-token"]').content;

document.querySelectorAll(".task-toggle").forEach(box => {
  box.addEventListener("change", async () => {
    const res = await fetch(`/api/tasks/${box.dataset.id}`, {
      method: "PATCH",
      headers: {
        "Content-Type": "application/json",     // tell Flask the body is JSON
        "X-CSRFToken": csrf,                    // Flask-WTF checks this header on API calls
      },
      body: JSON.stringify({ done: box.checked }),
    });
    if (!res.ok) {                              // fetch does NOT throw on 400/500 - check yourself
      alert("Couldn't save that change");
      box.checked = !box.checked;               // put it back
    }
  });
});
```

> ⚠️ **`fetch` does not fail on a 404 or 500** — it only fails if the network is down. Always check `res.ok` ([[Fetch and APIs (JavaScript)]]).

> ⚠️ **If you've turned on `CSRFProtect`, API calls from your pages must send the token** in an `X-CSRFToken` header, as above — otherwise you get a 400 *CSRF token missing*. The browser sends your login cookie automatically, so without CSRF protection another site could make the same call.

> ⚠️ **Without `Content-Type: application/json`, `request.get_json()` returns `None`** and your route sees no data.

---

## Calling your API from Python

```python
import httpx

s = httpx.Client(base_url="http://127.0.0.1:5000")

r = s.get("/api/tasks")
r.raise_for_status()                     # turn 4xx/5xx into an exception
for task in r.json():
    print(task["title"])

r = s.post("/api/tasks", json={"title": "Written from Python"})    # json= sets the header for you
print(r.status_code, r.json())
```

See [[Requests and httpx]]. For a script calling your API, logging in via a cookie is awkward — that's when you'd add **API keys** or **tokens** instead (below).

---

## Testing an API by hand

```bash
curl http://127.0.0.1:5000/api/status

curl -X POST http://127.0.0.1:5000/api/tasks \
     -H "Content-Type: application/json" \
     -d '{"title": "from curl"}'
```

> **On Windows PowerShell, `curl` is an alias for something else** and the quoting differs. Use `curl.exe`, or `Invoke-RestMethod`:
> ```powershell
> Invoke-RestMethod http://127.0.0.1:5000/api/status
> ```

Or use a GUI — **Bruno**, **Insomnia**, **Postman**, or the **REST Client** extension in VS Code.

---

## API keys — for programs instead of people

Browsers log in with a cookie. A script or another server usually sends a **key** in a header instead:

```python
from flask import request
import os, hmac
from functools import wraps

API_KEY = os.environ["API_KEY"]

def require_api_key(view):
    @wraps(view)
    def wrapped(*args, **kwargs):
        sent = request.headers.get("X-API-Key", "")
        if not hmac.compare_digest(sent, API_KEY):     # constant-time compare - see below
            return {"error": "invalid API key"}, 401
        return view(*args, **kwargs)
    return wrapped

@app.get("/api/export")
@require_api_key
def export():
    ...
```

> **`hmac.compare_digest` instead of `==`.** A normal `==` stops at the first wrong character, and an attacker can measure that tiny time difference to guess a key one character at a time.

> **`@csrf.exempt` on key-authenticated routes.** CSRF protects cookie logins; a request carrying an API key isn't vulnerable to it, and other programs have no CSRF token to send.

---

## CORS — when another website calls your API

If your API is called from a page on a **different** address (a React app on `localhost:3000` calling Flask on `localhost:5000`), the browser blocks it unless Flask says it's allowed:

```bash
uv add flask-cors
```

```python
from flask_cors import CORS
CORS(app, resources={r"/api/*": {"origins": ["http://localhost:3000"]}})   # a SPECIFIC list
```

> **CORS only matters for other *websites* calling your API from a browser.** Pages served by your own Flask app, Python scripts, curl and phone apps don't need it.

> ⚠️ **Don't use `origins="*"` together with cookie logins.** That lets any site on the internet make logged-in requests as your users.

---

## Flask API or FastAPI?

| Choose | When |
|---|---|
| **A Flask API** | You already have a Flask site and want some JSON routes for its own pages or a small app |
| **[[FastAPI reference\|FastAPI]]** | The API *is* the product — automatic validation, automatic docs at `/docs`, async, typed |

> **Pairing Flask pages with a separate FastAPI service is common**, but don't do it for a small project. One Flask app serving both pages and JSON is simpler to run, deploy and debug.

---

## Common mistakes

| Mistake | What you'll see | Fix |
|---|---|---|
| Returning a list without `jsonify` | `TypeError` | `jsonify([...])` |
| Returning a model object | `TypeError: not JSON serializable` | `task.to_dict()` |
| Returning a `datetime` | Same error | `.isoformat()` |
| Returning every model field | Leaks new sensitive columns | A deliberate `to_dict()` |
| No `Content-Type` from JavaScript | `get_json()` is `None` | Set the header |
| 200 status with an error body | Caller thinks it worked | Return 400/404/... |
| API returns the login page | HTML where JSON was expected | `unauthorized_handler` returning 401 |
| JS `fetch` + CSRFProtect | **400 CSRF token missing** | Send `X-CSRFToken` |
| Not checking `res.ok` in JS | Silent failures | `if (!res.ok)` |
| Stack traces in responses | Information leak | Generic message, log the detail |
| No ownership check | **IDOR** | `get_own_task()` |
| `origins="*"` with cookies | Any site can act as your users | A specific list |

## Related

[[Flask]] · [[Flask - user accounts and login]] · [[Flask - project structure and blueprints]] · [[Flask reference]] · [[FastAPI reference]] · [[Fetch and APIs (JavaScript)]] · [[Requests and httpx]] · [[Networking reference]] · [[Security in practice]]
