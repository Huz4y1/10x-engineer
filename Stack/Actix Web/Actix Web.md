Actix web is our backend for full stack apps and web apps. Actix is here to wait for HTTP requests, run some rust code and send back a response

```
Browser
   ↓
Next.js
   ↓
HTTP Request
   ↓
Actix Web
   ↓
Supabase
   ↓
PostgreSQL
```

dependencies needed for the backend

```tsx
[dependencies]
actix-web = "4"
serde = { version = "1", features = ["derive"] }
serde_json = "1"

dotenvy = "0.15"

actix-cors = "0.7"

log = "0.4"
env_logger = "0.11"
```

[[simple API project]]

[[Setting up a server and registering routes]]

[[Handler + service logic]]

[[Supabase]]

[[Setting CORS up]]

[[reqwest]]

[[Serde]]
---

[[Rust for this stack]] — serving an ONNX model from Actix, and when it beats FastAPI

---

**In this vault:** [[Rust for this stack]] · [[15 — FASTAPI]] - the Python alternative and when each wins.
