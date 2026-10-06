  

- **Auth?** None / API key header / Bearer token / basic auth. Found in docs, or discovered via a 401 from a bare curl.
- **Envelope?** Does the response body start with `[` (bare array) or `{` (wrapper object like `{"success": true, "data": [...]}`)? This decides the parse target.
- **Naming convention?** `camelCase`, `snake_case`, or mixed. Decides whether `#[serde(rename_all)]` is needed.
- **Nullable fields?** Which fields are `null` in _some_ responses. Decides which fields become `Option<T>`.
- **What do I actually need?** List the fields the frontend will display. Everything else gets ignored — never model the whole response.

  

## Step 1 — Investigate with curl + jq (PowerShell)

Save one response, then interrogate the file (saves rate limit):

```tsx
curl.exe -s "https://api.example.com/endpoint" -H "x-api-key: KEY" -o sample.json
```

The interrogation ladder:

```bash
jq "type" sample.json                        # "array" or "object"? → envelope question
jq "keys" sample.json                        # if object: what's the wrapper key?
jq ".data[0]" sample.json                    # what does one item look like?
jq ".data[0] | keys" sample.json             # all field names
jq ".data[0] | map_values(type)" sample.json # field name → JSON type, ready to copy
jq ".data | length" sample.json              # how many items
```

### Envelope check

powershell

```powershell
jq "type" sample.json
```

**Question:** is the whole response a list or a wrapper?

**Meaning of output:** `"array"` → the body IS your data, parse straight into `Vec<Item>`. `"object"` → there's an envelope around the data; you must find and unwrap it.

**Code decision:** whether a wrapper struct exists at all, and what `.json::<T>()` targets.

**Fetch 2–3 different responses** before deciding types — one sample can't reveal nullable fields. A field that's a string today may be `null` tomorrow.

PowerShell quirks to remember:

- `curl.exe` not `curl` (PowerShell aliases `curl` to Invoke-WebRequest)
- Double-quote jq filters; escape inner quotes: `jq ".data[] | select(.city == \"Las Vegas\")"`

### Reading JSON types by eye

JSON has only six types. Quotes are the key signal:

|See|Type|Rust|TypeScript|
|---|---|---|---|
|`"text"`|string|`String`|`string`|
|`812`|number|`u32`/`i64`/`f64`|`number`|
|`true`|boolean|`bool`|`boolean`|
|`null`|null|`Option<T>`|`T \| null`|
|`[...]`|array|`Vec<T>`|`T[]`|
|`{...}`|object|struct|type|

Traps: `"812"` is a **string** (UUIDs and IDs often are). Dates are always strings — JSON has no date type. ISO dates (`2026-08-15`) sort correctly as strings.

  

## Step 2 — Write the Rust structs (the contract)

Two struct layers in the service file, both private:

```rust
use serde::Deserialize;

// Matches the wrapper (skip if response is a bare array)
#[derive(Deserialize)]
struct ApiResponse {
    data: Vec<ItemApiResponse>,   // field name = whatever `jq "keys"` showed
}

// Matches ONE item — only fields I use
#[derive(Deserialize)]
#[serde(rename_all = "camelCase")]   // if their JSON is camelCase
struct ItemApiResponse {
    id: String,                       // UUID → String, not a number
    title: String,
    starts_at: String,                // maps to "startsAt" via rename_all
    state: Option<String>,            // nullable in samples → Option
}
```

Rules:

- Declare only fields I need — serde ignores extras in the JSON
- Any field null in ANY sample → `Option<T>`
- One awkward name → `#[serde(rename = "theirName")]`; whole convention mismatch → `rename_all` on the struct
- Reserved-word field (`type`) → must rename: `#[serde(rename = "type")] type_info: ...`

Plus a clean public model in `models/` with `#[derive(Serialize)]` — the shape MY API exposes. Keep it snake_case; the frontend matches this, not the upstream API.

  

### Step 3 — The service function: three beats

```rust
pub async fn get_things() -> Result<Vec<Thing>, reqwest::Error> {
    let api_key = env::var("MY_API_KEY").expect("MY_API_KEY must be set in .env");

    // Beat 1: SEND
    let response = reqwest::Client::new()
        .get("https://api.example.com/endpoint")
        .header("x-api-key", api_key)
        .send()
        .await?
        .error_for_status()?;          // turn 4xx/5xx into Err here

    // Beat 2: PARSE (into the wrapper)
    let parsed: ApiResponse = response.json().await?;

    // Beat 3: TRANSLATE (their shape → my model)
    let things = parsed.data.into_iter()
        .map(|e| Thing { id: e.id, title: e.title, /* ... */ })
        .collect();

    Ok(things)
}
```

Gotchas that bit me:

- `reqwest::get(&url)` (one-shot) **cannot attach headers** — authenticated APIs need the `Client` builder form. This was the #1 difference from the Pokemon reference.
- `error_for_status()` matters: without it a 401 surfaces as a confusing JSON decode error instead of the real cause.
- Dynamic URL segments: `let url = format!("https://api.example.com/{sport}/events");`
- Key lives in `.env` (gitignored!), loaded by `dotenv().ok()` in main.

Dependencies: `reqwest = { version = "0.12", features = ["json"] }`, `serde = { version = "1", features = ["derive"] }`.

  

## Step 4 — Handler + wiring

Handler = thin: extract → call service → translate Result to HTTP:

```rust
use actix_web::{Responder, get};

#[get("/api/ufc/schedule")]           // lowercase, /api prefix, case-sensitive!
pub async fn get_schedule_handler() -> impl Responder {
    match ufc_schedule_service::get_schedule().await {
        Ok(s) => HttpResponse::Ok().json(s),
        Err(e) => {
            eprintln!("fetch failed: {e}");
            if e.is_decode() {
                HttpResponse::InternalServerError().body("Response shape mismatch")
            } else {
                HttpResponse::BadGateway().body("Upstream API unreachable")
            }
        }
    }
}
```

Module-tree checklist (source of "unresolved import" / "unlinked file" errors):

- `main.rs`: `mod handlers; mod services; mod models;`
- Each folder has `mod.rs` with `pub mod file_name;` — **must match the filename exactly** (watch for typos: shedule vs schedule cost me an hour)
- Functions crossing module boundaries need `pub`
- Import the _module_ in the handler (`use crate::services::x_service;`) to avoid name collisions with the handler function
- Main imports handlers only — never reaches down into services
- Register with `.service(handler_name)` in main

`mod` builds the tree, `use` navigates it.

  

## Step 5 — Test the backend ALONE before touching the frontend

The sanity ladder — each rung isolates a layer:

[http://localhost:8080/api/test](http://localhost:8080/api/test) → server up? right process?  
[http://localhost:8080/api/ufc/schedule](http://localhost:8080/api/ufc/schedule) → full chain works?

- 404 → route string mismatch, unregistered handler, or **STALE BINARY** (see below)
- "error decoding response body" → struct doesn't match reality; back to jq (Step 1)
- 401/403 in the logs → key problem
- My own error message in browser → chain connected, service errored; **read the terminal** — `eprintln!` has the real cause

**⚠ Rust has NO hot reload.** Every code change: Ctrl+C, `cargo run` again. (Django/Next habits will betray you.) Fix: `cargo install cargo-watch`, then `cargo watch -x run`.

**⚠ Windows:** error 4551 "Application Control policy has blocked this file" = Smart App Control blocking the freshly-built exe. Windows Security → App & browser control → turn it off (one-way switch; normal for dev machines).

  

## Step 6 — Next.js page

TS type matches **my** backend's output (snake_case), not the upstream API:

```tsx
type Schedule = {
  id: string;
  title: string;
  starts_at: string;
  event_date_label: string;
  image_url: string;
  // ...
};

export default async function SchedulePage() {
  const res = await fetch('http://localhost:8080/api/ufc/schedule', {
    cache: 'no-store',              // schedules change; don't serve stale
  });
  if (!res.ok) throw new Error(`Request failed: ${res.status}`);

  const events: Schedule[] = await res.json();

  return (
    <main className="schedule-page">
      <h1>Upcoming events</h1>
      {events.length === 0 && <p>No upcoming events.</p>}
      <div className="schedule-grid">
        {events.map((event) => (
          <div className="schedule-card" key={event.id}>
            <img src={event.image_url} alt={event.title} className="schedule-img" />
            <div className="schedule-info">
              <h3>{event.title}</h3>
              <p>{event.event_date_label}</p>
              <p>{event.venue} — {event.city}</p>
            </div>
          </div>
        ))}
      </div>
    </main>
  );
}
```

Reminders:

- Server component fetch runs server-to-server → no CORS involved, no loading spinner, data in the HTML. CORS only matters for `'use client'` fetches.
- `fetch` does NOT throw on 404/500 — always check `res.ok`
- `.map()` needs `key={unique_id}` on the outermost element
- **Don't name a type** `**Event**` — it shadows the DOM's built-in Event type and produces bizarre errors silently
- `{}` in JSX = any JS expression; `{cond && <X/>}` = render-if; ternary = if/else
- Base URL → `NEXT_PUBLIC_API_URL` in `.env.local` before deploying

  

## Step 7 — Styling the cards

In `globals.css` (imported once in `app/layout.tsx`; `className` not `class`):

```css
.schedule-page { max-width: 1100px; margin: 0 auto; padding: 32px 16px; }

.schedule-grid {
  display: flex;
  flex-wrap: wrap;              /* wrap to rows, don't squish */
  gap: 24px;                    /* spacing between cards, no margins needed */
  justify-content: center;
}

.schedule-card {
  width: 320px;
  border: 1px solid #ddd;
  border-radius: 12px;
  overflow: hidden;             /* clip image to rounded corners */
}

.schedule-img {
  width: 100%;
  height: 180px;
  object-fit: cover;            /* crop, don't stretch varied aspect ratios */
}

.schedule-info { padding: 16px; }
```

Flexbox memory hooks:

- `justify-content` = main axis (horizontal in a row) · `align-items` = cross axis
- Vertical centering needs the container to have height (`min-height: 100vh`) or nothing moves
- Axes swap in `flex-direction: column`