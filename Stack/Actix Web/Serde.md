---
tags: [rust, crate, serde, serialisation, json]
---

# Serde

Rust's serialisation framework. Stack: [[Actix Web]] · Crates: [[Rust crates]]

```rust
use serde::Deserialize;

#[derive(Deserialize)]
```

From data format to Rust Struct

---

## The two traits

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
struct Reading {
    device_id: String,
    temp_c: f64,
    tags: Vec<String>,
}
```

| Trait | Direction |
|---|---|
| **`Deserialize`** | Data format → Rust struct (parsing input) |
| **`Serialize`** | Rust struct → data format (producing output) |

> **`#[derive(...)]` writes the code for you at compile time.** No reflection, no runtime cost — it's as fast as hand-written parsing.

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

## JSON

```rust
let json = serde_json::to_string(&reading)?;
let json = serde_json::to_string_pretty(&reading)?;
let reading: Reading = serde_json::from_str(&json)?;
let reading: Reading = serde_json::from_slice(&bytes)?;

// untyped, when you don't know the shape
let v: serde_json::Value = serde_json::from_str(&json)?;
let temp = v["temp_c"].as_f64().unwrap_or(0.0);
```

## Field attributes — where the real work happens

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]          // deviceId in JSON, device_id in Rust
struct Reading {
    device_id: String,

    #[serde(rename = "temperature")]        // different name for one field
    temp_c: f64,

    #[serde(default)]                        // missing -> Default::default()
    tags: Vec<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    note: Option<String>,                    // omit from output when None

    #[serde(skip)]
    internal: bool,                          // never serialised or read

    #[serde(alias = "ts", alias = "timestamp")]
    time: String,                            // accept several input names
}
```

> **`rename_all = "camelCase"` is the one you'll use most** — JavaScript APIs send `deviceId`, Rust wants `device_id`, and this bridges them with no manual mapping.

> **`#[serde(default)]` makes a field optional on input.** Without it, missing JSON fields are a hard error.

## Strict input

```rust
use serde::Deserialize;

#[derive(Deserialize)]
#[serde(deny_unknown_fields)]               // reject anything unexpected
struct Config { host: String, port: u16 }
```

> **Use `deny_unknown_fields` on config files.** It turns a typo like `prot = 8080` into an error instead of a silently ignored setting.

## Enums

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
enum Event {
    Reading { device_id: String, temp: f64 },
    Heartbeat { device_id: String },
}
// {"type":"reading","device_id":"1","temp":42.1}
```

## Other formats

Serde is format-agnostic — the same derives work everywhere:

| Crate | Format |
|---|---|
| `serde_json` | JSON |
| `toml` | TOML (config files) |
| `serde_yaml` | YAML |
| `csv` | CSV |
| `bincode` | Compact binary |
| `rmp-serde` | MessagePack |

```rust
let cfg: Config = toml::from_str(&std::fs::read_to_string("config.toml")?)?;
```

## With Actix Web

```rust
use actix_web::{Responder, post};

#[post("/readings")]
async fn create(body: web::Json<Reading>) -> impl Responder {
    HttpResponse::Ok().json(&*body)         // Serialize -> JSON response
}
```

> **`web::Json<T>` deserialises and validates the body automatically**, returning 400 on malformed input. It's Actix's equivalent of [[Pydantic]] in [[FastAPI reference]] — though Serde checks *shape and types*, not business rules like ranges.

## Custom conversion

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct Row {
    #[serde(deserialize_with = "parse_date")]
    date: chrono::NaiveDate,
}

fn parse_date<'de, D>(d: D) -> Result<chrono::NaiveDate, D::Error>
where D: serde::Deserializer<'de> {
    let s = String::deserialize(d)?;
    chrono::NaiveDate::parse_from_str(&s, "%Y-%m-%d").map_err(serde::de::Error::custom)
}
```

## Common mistakes

| Mistake | Fix |
|---|---|
| `derive` not found | Add `features = ["derive"]` |
| Missing field errors | `#[serde(default)]` or `Option<T>` |
| camelCase JSON not mapping | `#[serde(rename_all = "camelCase")]` |
| `null` failing to parse | Use `Option<T>` |
| Config typos silently ignored | `#[serde(deny_unknown_fields)]` |
| Numbers overflowing | Match the type — `i64` not `i32` |

## Related

[[Actix Web]] · [[Rust crates]] · [[reqwest]] · [[sqlx]] · [[JSON]] · [[Pydantic]] · [[Rust]]
