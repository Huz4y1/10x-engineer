---
tags: [security, auth, secrets, deep-dive]
---

# Security in practice

The index is [[26 — SECURITY]]. This is the *doing* half — the actual code, the actual mistakes, the actual commands.

---

## The mental model

Imagine your app is a building.

- **Authentication** is the door with the ID check. *Who are you?*
- **Authorisation** is which rooms your badge opens. *What are you allowed to do?*
- **Secrets** are the keys. If you leave one taped to the door, the lock is decoration.
- **Least privilege** is only giving someone the keys to the rooms they actually need.

Almost every real breach is one of four things: **a leaked key, a missing authorisation check, unvalidated input, or an out-of-date dependency.** Nothing exotic. Get those four right and you're ahead of most production systems.

---

## 1. Secrets

### The rules

| Rule | Why |
|---|---|
| Never in source code | Git remembers forever |
| Never in a Docker image | `docker history` prints them |
| Never in a frontend bundle | It's downloaded by the user |
| Never in a log line | Logs get shipped, indexed and shared |
| Never in a URL | URLs land in server logs and browser history |

```bash
# .env  -- and .env goes in .gitignore, always
DATABASE_URL=postgres://user:pass@host/db
```

```python
import os
DB_URL = os.environ["DATABASE_URL"]                  # crashes loudly if missing - GOOD
db = os.getenv("DATABASE_URL", "sqlite:///dev.db")   # silent fallback - dangerous in prod
```

> **Prefer `os.environ[...]` over `os.getenv(..., default)` for secrets.** A missing secret should stop the app, not silently downgrade it to something insecure.

### If you commit a secret

```bash
git rm --cached .env
echo ".env" >> .gitignore
```

> ⚠️ **Deleting it in a new commit does NOT remove it.** It is still in the history, still on GitHub, still in every clone and every fork. **The only real fix is to rotate the credential** — generate a new one and revoke the old one. Do that *first*, then clean the history if you want to.

Bots scan public GitHub for cloud keys within **seconds** of a push. Assume any committed key is compromised the moment it lands.

### Better than a secret: no secret

```python
from azure.identity import DefaultAzureCredential      # Azure managed identity
cred = DefaultAzureCredential()
```

> **Managed identity / IAM roles are the best form of secret management: there is no secret.** The cloud gives your running container a short-lived token automatically. Nothing to leak, nothing to rotate. Use it wherever it exists ([[Azure fundamentals]], [[Cloud comparison dictionary]]).

---

## 2. Passwords

```python
# uv add "passlib[bcrypt]"
from passlib.context import CryptContext
pwd = CryptContext(schemes=["bcrypt"], deprecated="auto")

hashed = pwd.hash("hunter2")            # store THIS
pwd.verify("hunter2", hashed)           # True
```

**Hashing is not encryption.**

| | Encryption | Hashing |
|---|---|---|
| Reversible? | Yes, with the key | **No, ever** |
| Use for | Data you need back | Passwords |

> ⚠️ **Never use MD5 or SHA-256 for passwords.** They're designed to be *fast*, which is exactly wrong — a GPU tries billions per second. bcrypt and argon2 are deliberately slow, which is the entire point.

> **Never store a password you can read.** If a site can email you your old password, it's storing it wrong.

Salting is automatic in bcrypt — a random value mixed in so two people with the same password get different hashes.

---

## 3. Tokens and sessions

```python
# uv add pyjwt
import jwt, datetime

token = jwt.encode(
    {"sub": user_id, "exp": datetime.datetime.now(datetime.UTC) + datetime.timedelta(minutes=15)},
    SECRET_KEY, algorithm="HS256",
)

claims = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])   # raises if invalid/expired
```

> ⚠️ **A JWT is signed, not encrypted.** Anyone can paste it into a decoder and read the payload. **Never put anything secret in a JWT** — no passwords, no personal data. The signature only proves it wasn't *changed*.

> ⚠️ **Always pass `algorithms=[...]` explicitly to `decode`.** Historic libraries accepted the algorithm named *inside the token*, letting an attacker declare "none" and sign their own admin token.

| Property | Set it to | Why |
|---|---|---|
| Access token lifetime | 15 min | Limits damage if stolen |
| Refresh token | Longer, revocable, stored server-side | Can be cancelled |
| Cookie flags | `HttpOnly; Secure; SameSite=Lax` | JS can't read it, HTTPS only, blocks CSRF |

> **`HttpOnly` is the one that matters most.** It means JavaScript cannot read the cookie, so an XSS bug can't steal the session.

---

## 4. Authorisation — the check people forget

Authentication without authorisation is the most common real-world bug.

```python
from fastapi import Depends, HTTPException

# BROKEN - logged in, so you can read ANY order
@app.get("/orders/{order_id}")
async def get_order(order_id: int, user = Depends(current_user)):
    return await db.get_order(order_id)

# CORRECT
@app.get("/orders/{order_id}")
async def get_order(order_id: int, user = Depends(current_user)):
    order = await db.get_order(order_id)
    if order is None or order.user_id != user.id:
        raise HTTPException(404)          # 404, not 403 - don't confirm it exists
    return order
```

This is **IDOR** — Insecure Direct Object Reference. Change `/orders/41` to `/orders/42` and see someone else's data. It is trivially exploitable and it is everywhere.

> **Return 404, not 403, for objects the user doesn't own.** A 403 confirms the record exists, which is itself information.

> **Test it the lazy way:** log in as user A, take an ID belonging to user B, request it. If you get data back, you have this bug.

In [[Supabase]], **Row Level Security** does this check in the database so you cannot forget — see [[Supabase reference]].

---

## 5. Input validation

### SQL injection

```python
# CATASTROPHIC
cur.execute(f"SELECT * FROM users WHERE email = '{email}'")
# an email of   ' OR '1'='1     returns every user
# an email of   '; DROP TABLE users; --     does exactly what it looks like

# CORRECT - parameterised, so the driver keeps data and code separate
cur.execute("SELECT * FROM users WHERE email = %s", (email,))
```

> **The rule is absolute: never build SQL by string formatting with user input.** Not with f-strings, not with `+`, not with `.format()`. Parameterised queries, always. [[sqlx]] enforces this at compile time; [[SQLAlchemy]] and [[PostgreSQL reference]] handle it for you.

### Validate the shape, not just the type

```python
from pydantic import BaseModel, EmailStr, Field

class SignUp(BaseModel):
    email: EmailStr
    age: int = Field(ge=13, le=120)
    username: str = Field(min_length=3, max_length=32, pattern=r"^[a-zA-Z0-9_]+$")
```

> **Validate at the boundary, once, then trust it inside.** [[Pydantic]] in Python, Zod in [[TypeScript]]. Both turn "untrusted JSON" into "a typed object I can reason about".

### Command injection and path traversal

```python
from pathlib import Path
import subprocess

subprocess.run(f"convert {filename} out.png", shell=True)     # BAD: shell=True + user input
subprocess.run(["convert", filename, "out.png"])              # GOOD: list, no shell

open(f"/data/{user_path}")                                    # BAD: user_path = "../../etc/passwd"

safe = (Path("/data") / user_path).resolve()                  # GOOD
if not safe.is_relative_to(Path("/data").resolve()):
    raise ValueError("path escapes the data directory")
```

### Deserialisation

> ⚠️ **`pickle.load`, `joblib.load`, `torch.load` and `np.load(allow_pickle=True)` execute arbitrary code.** Loading an untrusted pickle is equivalent to running a stranger's script as your user. Only load artifacts **you** produced, from storage **you** control. For anything crossing a trust boundary use ONNX, safetensors, JSON or Parquet ([[Model export and serving]]).

---

## 6. Web-specific

| Attack | What it is | Defence |
|---|---|---|
| **XSS** | Attacker's JavaScript runs on your page | Escape output; React and templating engines escape by default; never render user HTML raw |
| **CSRF** | Another site makes your browser send an authenticated request | `SameSite=Lax` cookies, CSRF tokens |
| **Clickjacking** | Your site loaded in an invisible iframe | `X-Frame-Options: DENY` |
| **Open redirect** | `?next=https://evil.example` | Allowlist redirect targets |

```python
from fastapi.middleware.cors import CORSMiddleware

# CORS - the wildcard is the mistake
app.add_middleware(CORSMiddleware,
    allow_origins=["https://myapp.com"],     # a specific list, NOT ["*"]
    allow_credentials=True)
```

> ⚠️ **`allow_origins=["*"]` together with `allow_credentials=True` is invalid and browsers reject it** — for good reason. If you're tempted by the wildcard, what you actually want is a specific list.

> **CORS is a *browser* protection, not a server one.** It stops other websites calling your API from a user's browser. It does nothing against `curl`. **CORS is not authorisation.**

### Rate limiting

```python
# uv add slowapi
@limiter.limit("5/minute")
@app.post("/login")
async def login(...): ...
```

> **Rate-limit login, signup, password reset, and anything that sends an email or costs money** (LLM endpoints especially — [[Large language models]]). Without it, password guessing is free and unlimited.

### Never leak the stack trace

```python
from fastapi.responses import JSONResponse

@app.exception_handler(Exception)
async def handler(request, exc):
    logger.exception("unhandled")                            # full detail in the LOG
    return JSONResponse({"detail": "Internal error"}, 500)   # nothing in the RESPONSE
```

> A stack trace tells an attacker your framework, your versions, your file paths and sometimes your query. Log it; don't return it ([[FastAPI fundamentals]]).

---

## 7. Containers and Kubernetes

```dockerfile
FROM python:3.12-slim
RUN useradd -m -u 1000 appuser
USER appuser                       # not root
```

```bash
docker history myimage             # secrets baked into layers show up here
docker scout cves myimage          # scan for known vulnerabilities
trivy image myimage
```

> ⚠️ **A Kubernetes Secret is base64, not encryption.** Anyone who can read the secret object can decode it in one command, and anyone with `get secrets` in that namespace can read every secret in it. Enable encryption at rest, restrict with RBAC, or use an external secret store ([[17 — KUBERNETES]]).

> **Never put a secret in a Dockerfile `ENV` or `ARG`.** Both persist in the image layers. Use build secrets or runtime environment injection ([[Docker deep dive]]).

---

## 8. Dependencies

```bash
uv pip audit          # Python
npm audit             # Node
cargo audit           # Rust
```

> **Most vulnerabilities you ship are in code you didn't write.** Pin versions with a lockfile, commit the lockfile, run an audit in [[CI-CD pipelines]], and update on a schedule rather than in a panic.

> **Check a package name before you install it.** Typosquatting — a package named one character away from the real one — is a real and active attack.

---

## 9. The pre-deploy checklist

- [ ] No secrets in git, in the image, or in the frontend bundle
- [ ] HTTPS only; `Secure; HttpOnly; SameSite` on cookies
- [ ] Passwords hashed with bcrypt or argon2
- [ ] **Every endpoint checks ownership, not just login**
- [ ] All input validated at the boundary ([[Pydantic]] / Zod)
- [ ] All SQL parameterised
- [ ] CORS is a specific list, not `*`
- [ ] Rate limits on auth and on anything expensive
- [ ] Errors logged, generic message returned
- [ ] Container runs as non-root
- [ ] Dependency audit passing in CI
- [ ] RLS enabled on every [[Supabase]] table
- [ ] Backups exist **and you have restored one**

> **A backup you've never restored is a hope, not a backup.** Restore one, on purpose, before you need to.

---

## Common mistakes

| Mistake | Consequence |
|---|---|
| Secret committed, then "deleted" | Still compromised — **rotate it** |
| Admin / service key in the frontend | Total database access |
| Login check but no ownership check | **IDOR** — everyone reads everyone's data |
| f-string SQL | SQL injection |
| MD5/SHA for passwords | Cracked in minutes |
| Secret data inside a JWT | Readable by anyone |
| `allow_origins=["*"]` | Any site can call your API from a browser |
| Stack trace in the response | Free reconnaissance |
| `pickle.load` on an uploaded file | **Remote code execution** |
| Container running as root | A container escape becomes host root |
| No rate limit on `/login` | Unlimited free password guessing |

## Related

[[26 — SECURITY]] · [[FastAPI fundamentals]] · [[Supabase reference]] · [[sqlx]] · [[Pydantic]] · [[Docker deep dive]] · [[17 — KUBERNETES]] · [[CI-CD pipelines]] · [[Azure fundamentals]] · [[Networking reference]] · [[Model export and serving]]
