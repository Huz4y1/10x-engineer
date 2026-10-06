---
tags: [flask, deployment, gunicorn, docker, production, guide]
---

# Flask — deploying it

**Guide 8b of 8.** Getting your app off your laptop and onto a server other people can use — safely.

Hub: [[Flask]] · Previous: [[Flask - project structure and blueprints]] · Then build: [[Flask - full project walkthrough]]

---

## Why `flask run` isn't enough

`flask run` starts Flask's **development server**. It's built for one person — you — and it says so in the terminal:

```
WARNING: This is a development server. Do not use it in a production deployment.
```

It handles requests slowly, isn't hardened against attack, and with `--debug` it will run any Python code a visitor sends it.

**In production you use a proper server program** — it runs several copies of your app at once so many visitors are served in parallel.

```
Visitors ──► [ Nginx or the cloud's load balancer ]  HTTPS, static files
                        │
                        ▼
             [ Gunicorn ]  runs 4 copies ("workers") of your Flask app
               ├── worker 1
               ├── worker 2
               ├── worker 3
               └── worker 4 ──► your database
```

---

## The production checklist

Before anyone else can reach your app:

- [ ] **Debug is OFF** — `DEBUG = False`, and never `--debug`
- [ ] **`SECRET_KEY` is long, random and not in git** — `secrets.token_hex(32)`
- [ ] **A real database** — PostgreSQL, not SQLite, if more than one person writes at once
- [ ] **Migrations applied** — `flask db upgrade`
- [ ] **HTTPS** — and `SESSION_COOKIE_SECURE = True`
- [ ] **A real server** — Gunicorn (Linux) or Waitress (Windows)
- [ ] **Secrets come from environment variables**, not files in the image
- [ ] **CSRF protection on** — `CSRFProtect`
- [ ] **Errors logged**, generic messages shown to users
- [ ] **You've restored a database backup at least once**

> ⚠️ **Debug mode on a public server is the worst possible Flask mistake.** The debug error page contains an interactive Python console. Anyone who can make your app crash can then run commands on your server — read files, steal your database, anything ([[Security in practice]]).

---

## Gunicorn — the production server (Linux, WSL, Docker)

```bash
uv add gunicorn
```

```bash
gunicorn -w 4 -b 0.0.0.0:8000 "app:create_app()"
#        ^^^^ 4 workers   ^^^^^^^^^^^^^ listen on port 8000   ^^^^^^^^^^^^^^^^^^^ call the factory
```

| Flag | Means |
|---|---|
| `-w 4` | Run 4 worker processes. A good starting point: **2 × CPU cores + 1** |
| `-b 0.0.0.0:8000` | Listen on all network interfaces, port 8000 |
| `"app:create_app()"` | Module `app`, call `create_app()` to get the Flask object |
| `--timeout 60` | Kill a worker stuck for 60 seconds |
| `--access-logfile -` | Print each request to the terminal |

> ⚠️ **Gunicorn does not run on Windows.** It relies on Unix features. Use it in **WSL** or **Docker**, or use Waitress below.

> **`0.0.0.0` vs `127.0.0.1`:** `127.0.0.1` means *only this machine can connect*. Inside Docker or on a server you need `0.0.0.0`, which means *accept connections from outside* — otherwise nothing can reach it.

## Waitress — the production server on Windows

```bash
uv add waitress
```

```bash
waitress-serve --host 0.0.0.0 --port 8000 --call app:create_app
```

Same job as Gunicorn, works natively on Windows.

---

## Production config

```python
class ProductionConfig(Config):
    DEBUG = False
    SESSION_COOKIE_SECURE = True           # cookies only over HTTPS
    REMEMBER_COOKIE_SECURE = True
    PREFERRED_URL_SCHEME = "https"
```

```python
# wsgi.py - the file your server loads
from app import create_app
app = create_app("config.ProductionConfig")
```

```bash
gunicorn -w 4 -b 0.0.0.0:8000 wsgi:app
```

### Behind a proxy

When Nginx or a cloud load balancer sits in front, Flask sees every request as coming from the proxy, over plain HTTP. This fixes it:

```python
from werkzeug.middleware.proxy_fix import ProxyFix
app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_proto=1, x_host=1)
```

> **Symptoms that you need `ProxyFix`:** `url_for(..., _external=True)` builds `http://` links on an HTTPS site, redirects bounce between http and https, or every visitor's IP address looks the same (which breaks rate limiting).

---

## Docker — the same app anywhere

A **container** packages your app, Python, and every library into one image that runs identically on your laptop and on any server. See [[Docker deep dive]].

`Dockerfile`:

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1
# PYTHONUNBUFFERED=1 makes print() and logs appear immediately in `docker logs`

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
# copy requirements FIRST - Docker caches this layer, so code changes don't reinstall everything

COPY . .

RUN useradd -m appuser
USER appuser
# don't run as root - limits the damage if the app is ever compromised

EXPOSE 8000
CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:8000", "wsgi:app"]
```

```bash
uv export --no-hashes --format requirements-txt > requirements.txt    # uv -> requirements.txt
```

`.dockerignore`:

```
.venv
__pycache__
instance
.env
*.db
.git
```

> ⚠️ **`.env` must be in `.dockerignore`.** Otherwise your secrets are baked into the image, and anyone who gets the image gets the secrets.

**Build and run:**

```bash
docker build -t tasktracker .
docker run -p 8000:8000 --env-file .env tasktracker    # secrets passed in at RUN time
```

### With a Postgres database — Docker Compose

`docker-compose.yml`:

```yaml
services:
  web:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    environment:
      DATABASE_URL: postgresql+psycopg://tasks:tasks@db:5432/tasks   # "db" = the service name below
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: tasks
      POSTGRES_PASSWORD: tasks
      POSTGRES_DB: tasks
    volumes:
      - pgdata:/var/lib/postgresql/data     # keep the data when the container restarts
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U tasks"]
      interval: 5s
      retries: 10

volumes:
  pgdata:
```

```bash
docker compose up -d --build                              # start both
docker compose exec web flask --app app db upgrade        # create the tables
docker compose logs -f web                                # watch the logs
docker compose down                                       # stop (data kept in the volume)
```

> ⚠️ **Inside Docker, the database host is the service name — `db` — not `localhost`.** `localhost` inside the web container means *the web container itself*, where there's no database. This is the single most common Docker + Flask error.

---

## Where to host it

| Option | Good for | Notes |
|---|---|---|
| **Render / Railway / Fly.io** | First deployment | Push to git, it builds and runs. Managed Postgres available |
| **PythonAnywhere** | Simplest possible | Made for Flask/Django, free tier |
| **Azure App Service / Container Apps** | Your Azure stack | [[Pipeline setup - Azure]] |
| **AWS App Runner / GCP Cloud Run** | Container-based | Scale to zero |
| **A VPS** (a rented Linux server) | Full control, cheapest at scale | You manage Nginx, HTTPS and updates yourself |

> **For your first deployment, use a platform (Render, Railway, Fly).** They handle HTTPS, restarts and logs. Running your own server is a good thing to learn, but it's a separate project.

**Most platforms need three things from you:**

1. A **start command** — `gunicorn -w 4 -b 0.0.0.0:$PORT wsgi:app` (`$PORT` is set by the platform)
2. **Environment variables** — `SECRET_KEY`, `DATABASE_URL`
3. A **release command** to run migrations — `flask --app app db upgrade`

---

## Static files in production

Flask *can* serve your CSS and images, but it's slow at it. Two fixes:

```bash
uv add whitenoise
```

```python
from whitenoise import WhiteNoise
app.wsgi_app = WhiteNoise(app.wsgi_app, root="app/static/", prefix="static/")
```

> **WhiteNoise is the easy option** — static files are served efficiently by your app itself, with no Nginx to configure. On a VPS, letting Nginx serve `/static/` directly is the traditional alternative.

---

## Logging

```python
from flask_login import current_user
import logging

logging.basicConfig(level=logging.INFO,
                    format="%(asctime)s %(levelname)s %(name)s: %(message)s")

app.logger.info("task created id=%s user=%s", task.id, current_user.id)
app.logger.exception("failed to save task")     # inside an except - includes the traceback
```

> **Log to stdout, not to a file.** Docker and every hosting platform collect stdout automatically. Log files inside a container disappear when it restarts. See [[Observability]].

> ⚠️ **Never log passwords, full card numbers or session cookies** — logs get copied, shared and kept for a long time.

---

## Common mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| **Debug mode in production** | **Anyone can run code on your server** | `DEBUG = False` |
| `flask run` in production | Slow, fragile | Gunicorn or Waitress |
| Gunicorn on Windows | Won't start | WSL, Docker, or Waitress |
| Binding to `127.0.0.1` in Docker | Nothing can connect | `0.0.0.0` |
| `localhost` as the DB host in Compose | Connection refused | The service name, `db` |
| `.env` copied into the image | **Secrets leaked** | `.dockerignore` it, pass at runtime |
| SQLite with several workers | `database is locked` errors | PostgreSQL |
| Forgot to run migrations | `relation "tasks" does not exist` | `flask db upgrade` on every deploy |
| No `ProxyFix` behind a proxy | http/https redirect loops | Add `ProxyFix` |
| Logs written to a file in a container | Lost on restart | Log to stdout |
| No `PYTHONUNBUFFERED=1` | `docker logs` shows nothing | Set it in the Dockerfile |

## Related

[[Flask]] · [[Flask - project structure and blueprints]] · [[Flask - full project walkthrough]] · [[Flask reference]] · [[Docker deep dive]] · [[Deployment patterns]] · [[PostgreSQL reference]] · [[Security in practice]] · [[Observability]]
