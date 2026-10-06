---
tags: [rust, crate, http, reqwest]
---

# reqwest

The HTTP **client** for Rust — for calling other people's APIs. Stack: [[Actix Web]] · Crates: [[Rust crates]] · Runtime: [[tokio]]

```toml
[dependencies]
reqwest = { version = "0.12", features = ["json"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
```

> **Actix Web *receives* requests; reqwest *sends* them.** You need both when your service calls another API.

---

## The simplest call

```rust
#[tokio::main]
async fn main() -> Result<(), reqwest::Error> {
    let body = reqwest::get("https://api.example.com/items")
        .await?
        .text()
        .await?;
    println!("{body}");
    Ok(())
}
```

> **Two `.await`s.** The first waits for the response *headers*, the second for the *body*. That surprises everyone once.

## JSON in and out

```rust
use serde::{Deserialize, Serialize};

#[derive(Deserialize, Debug)]
struct Item { id: u32, name: String, price: f64 }

#[derive(Serialize)]
struct NewItem<'a> { name: &'a str, price: f64 }

let items: Vec<Item> = reqwest::get(url).await?.json().await?;

let created: Item = client
    .post(url)
    .json(&NewItem { name: "Mug", price: 2.55 })     // sets Content-Type
    .send().await?
    .json().await?;
```

[[Serde]] does the conversion — `.json()` is just `serde_json` behind the scenes.

## Use a Client — always

```rust
use std::time::Duration;

let client = reqwest::Client::builder()
    .timeout(Duration::from_secs(10))            // <- NEVER omit
    .connect_timeout(Duration::from_secs(5))
    .user_agent("my-service/1.0")
    .build()?;

let r1 = client.get(format!("{BASE}/items")).send().await?;
let r2 = client.get(format!("{BASE}/customers")).send().await?;   // connection reused
```

> ⚠️ **`reqwest::get()` builds a brand-new client every call** — a full TCP + TLS handshake each time, 50–200 ms wasted. Build one `Client`, clone it freely (it's an `Arc` inside), reuse it everywhere.

> ⚠️ **Always set a timeout.** Without one, a hung server hangs your service forever.

## Headers, auth, query params

```rust
let r = client.get(url)
    .header("Authorization", format!("Bearer {token}"))
    .header("Accept", "application/json")
    .query(&[("category", "Kitchen"), ("weeks", "4")])
    .send().await?;

client.get(url).bearer_auth(token).send().await?;
client.get(url).basic_auth("user", Some("pass")).send().await?;
```

## Checking the response

```rust
let r = client.get(url).send().await?;

if r.status().is_success() {
    let items: Vec<Item> = r.json().await?;
} else {
    eprintln!("HTTP {}: {}", r.status(), r.text().await?);
}

let r = client.get(url).send().await?.error_for_status()?;   // turn 4xx/5xx into an Err
```

> ⚠️ **A 404 or 500 is *not* an `Err` by default.** `send()` only fails on network problems. `.json()` on an HTML error page then gives a confusing parse error. **Call `.error_for_status()`.**

## Concurrency

```rust
use futures::future::join_all;

let futures = urls.iter().map(|u| client.get(*u).send());
let responses = join_all(futures).await;        // all at once
```

```rust
// bounded concurrency
use futures::stream::{self, StreamExt};

let results: Vec<_> = stream::iter(urls)
    .map(|u| { let c = client.clone(); async move { c.get(u).send().await } })
    .buffer_unordered(10)                        // at most 10 in flight
    .collect().await;
```

## Errors and retries

```rust
match client.get(url).send().await {
    Ok(r) if r.status().is_success() => { /* ... */ }
    Ok(r)  => eprintln!("http {}", r.status()),
    Err(e) if e.is_timeout()  => eprintln!("timed out"),
    Err(e) if e.is_connect()  => eprintln!("connection failed"),
    Err(e) => eprintln!("other: {e}"),
}
```

```rust
let mut delay = Duration::from_millis(200);
for attempt in 0..3 {
    match client.get(url).send().await {
        Ok(r) if r.status().is_success() => break,
        _ if attempt < 2 => { tokio::time::sleep(delay).await; delay *= 2; }
        _ => {}
    }
}
```

> **Retry transient failures only** — timeouts, connection errors, 429, 503. Retrying a 400 just wastes time.

## Files and streaming

```rust
// download without loading it all into memory
let mut r = client.get(url).send().await?;
let mut file = tokio::fs::File::create("big.csv").await?;
while let Some(chunk) = r.chunk().await? {
    file.write_all(&chunk).await?;
}
```

```rust
// multipart upload
let form = reqwest::multipart::Form::new()
    .text("name", "readings")
    .file("file", "data.csv").await?;
client.post(url).multipart(form).send().await?;
```

## Blocking version (scripts, tests)

```toml
reqwest = { version = "0.12", features = ["blocking", "json"] }
```

```rust
let items: Vec<Item> = reqwest::blocking::get(url)?.json()?;
```

> ⚠️ **Never use the blocking client inside async code** — it blocks a tokio worker thread and can deadlock the runtime ([[tokio]]).

## Where it fits

Used by [[Scraper]] and any Actix service that calls a downstream API. The Python equivalent is [[Requests and httpx]] — same concepts, same rules about timeouts and client reuse.

## Common mistakes

| Mistake | Fix |
|---|---|
| `reqwest::get()` in a loop | Build one `Client` and reuse it |
| No timeout | `.timeout(Duration::from_secs(10))` |
| Assuming 404 is an error | `.error_for_status()` |
| Forgetting the second `.await` | Response *then* body |
| Blocking client in async | Use the async one |
| Missing the `json` feature | `features = ["json"]` |

## Related

[[Actix Web]] · [[Rust crates]] · [[tokio]] · [[Serde]] · [[Scraper]] · [[Requests and httpx]] · [[Networking reference]]
