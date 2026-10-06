Rust does JSON through `serde`, which is a general serialisation framework, plus `serde_json` for the JSON part specifically. It is the best JSON experience of any language here: you describe the shape as a struct and the compiler enforces it.

Setup

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

The `derive` feature is what gives you the `#[derive(Serialize, Deserialize)]` macros. Without it nothing works and the error is confusing.

Structs are the shape

```rust
/*
1. Deserialize = JSON -> struct, Serialize = struct -> JSON
2. field names must match the JSON keys, unless you rename them
3. from_str returns Result, so parsing failure is a value you must handle
*/

use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: u64,
    name: String,
    email: Option<String>,      // null or missing -> None
    tags: Vec<String>,
}

fn main() -> Result<(), serde_json::Error> {
    let text = r#"{"id":91,"name":"Huzayl","email":null,"tags":["rust"]}"#;

    let user: User = serde_json::from_str(text)?;
    println!("{}", user.name);          // output: Huzayl

    let back = serde_json::to_string(&user)?;
    let pretty = serde_json::to_string_pretty(&user)?;

    Ok(())
}
```

`Option<T>` is the whole story for nullable fields. If the JSON has `null`, or the key is absent, you get `None`. No unwrapping surprises later.

Renaming and defaults

Real APIs rarely use Rust naming conventions.

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase")]      // createdAt -> created_at, whole struct
struct User {
    id: u64,
    created_at: String,

    #[serde(rename = "e-mail")]         // one awkward field
    email: String,

    #[serde(default)]                   // missing -> Default::default()
    active: bool,

    #[serde(default = "default_score")] // missing -> this fn
    score: f64,

    #[serde(skip_serializing_if = "Option::is_none")]
    phone: Option<String>,              // omit from output when None

    #[serde(skip)]                      // never touch this field
    internal: String,
}

fn default_score() -> f64 { 5.0 }
```

`#[serde(default)]` is the one you will reach for constantly — it turns "the API sometimes omits this" from a runtime error into a sensible zero value.

Nested and wrapped responses

APIs usually wrap the thing you want.

```rust
use serde::Deserialize;

/*
1. one struct per level of nesting
2. this mirrors {"data":[...],"meta":{"total":91}} exactly
*/

#[derive(Debug, Deserialize)]
struct Response {
    data: Vec<User>,
    meta: Meta,
}

#[derive(Debug, Deserialize)]
struct Meta {
    total: u32,
    next_cursor: Option<String>,
}

let res: Response = serde_json::from_str(text)?;
println!("{} of {}", res.data.len(), res.meta.total);
```

Filtering, with iterators

Once parsed it is ordinary Rust — see [[Iteration]].

```rust
// filter, then collect back into a Vec
let adults: Vec<&User> = res.data
    .iter()
    .filter(|u| u.age > 30)
    .collect();

// filter + transform
let names: Vec<String> = res.data
    .iter()
    .filter(|u| u.active)
    .map(|u| u.name.clone())
    .collect();

// first match
let found = res.data.iter().find(|u| u.id == 91);

// any / all
let has_adult = res.data.iter().any(|u| u.age > 30);

// sum and count
let total: u32 = res.data.iter().map(|u| u.age).sum();
let n = res.data.iter().filter(|u| u.active).count();

// sort a copy
let mut sorted = res.data.clone();
sorted.sort_by_key(|u| u.age);
sorted.sort_by(|a, b| b.age.cmp(&a.age));            // descending
sorted.sort_by(|a, b| a.score.partial_cmp(&b.score).unwrap());  // floats

// group by, needs a HashMap
use std::collections::HashMap;

let mut by_city: HashMap<String, Vec<&User>> = HashMap::new();
for u in &res.data {
    by_city.entry(u.city.clone()).or_default().push(u);
}

// filter_map: filter and unwrap Options in one pass
let emails: Vec<&String> = res.data
    .iter()
    .filter_map(|u| u.email.as_ref())
    .collect();
```

`filter_map` with `as_ref()` is the idiomatic way to drop the `None`s and keep the values.

When you do not know the shape

`serde_json::Value` is a dynamic tree, like a JS object. Useful for exploring, or for genuinely arbitrary data.

```rust
use serde_json::Value;

let v: Value = serde_json::from_str(text)?;

// indexing returns Value::Null rather than panicking on a missing key
println!("{}", v["name"]);
println!("{}", v["address"]["city"]);
println!("{}", v["tags"][0]);

// convert out with the as_* methods, each returns Option
let name: Option<&str> = v["name"].as_str();
let age:  Option<i64>  = v["age"].as_i64();

// chained safely
let city = v.get("address")
    .and_then(|a| a.get("city"))
    .and_then(|c| c.as_str())
    .unwrap_or("unknown");

// iterate an array
if let Some(arr) = v["data"].as_array() {
    for item in arr {
        println!("{}", item["name"]);
    }
}
```

Use `Value` to explore, then write the struct. Structs give you compile-time safety; `Value` gives you none.

Building JSON

```rust
use serde_json::json;

let body = json!({
    "name": "Huzayl",
    "age": 21,
    "tags": ["rust", "hardware"],
    "address": { "city": "London" }
});

let text = body.to_string();
```

The `json!` macro takes normal Rust expressions inside, so you can interpolate variables directly.

Enums for varying shapes

This is where serde beats most languages. When a field can be one of several shapes, model it as an enum.

```rust
use serde::Deserialize;

/*
1. untagged tries each variant in order until one fits
2. good for APIs that return either a result or an error object
*/

#[derive(Debug, Deserialize)]
#[serde(untagged)]
enum ApiResult {
    Ok { data: Vec<User> },
    Err { error: String, code: u32 },
}

// or tagged, when the API sends a discriminator field
#[derive(Debug, Deserialize)]
#[serde(tag = "type", rename_all = "lowercase")]
enum Event {
    Click { x: i32, y: i32 },
    KeyPress { key: String },
}
// matches {"type":"click","x":10,"y":20}
```

Errors

`serde_json::Error` tells you the line and column, which makes debugging a large document quick.

```rust
match serde_json::from_str::<User>(text) {
    Ok(user) => println!("{:?}", user),
    Err(e) => eprintln!("failed at line {} col {}: {}", e.line(), e.column(), e),
}
```

Files and readers

```rust
use std::fs::File;
use std::io::BufReader;

// stream from a file rather than reading it all into a String first
let file = File::open("samples/users.json")?;
let reader = BufReader::new(file);
let users: Vec<User> = serde_json::from_reader(reader)?;

// write
let out = File::create("out.json")?;
serde_json::to_writer_pretty(out, &users)?;
```

JSON Lines

```rust
use std::io::{BufRead, BufReader};

let file = File::open("logs.jsonl")?;

for line in BufReader::new(file).lines() {
    let entry: LogEntry = serde_json::from_str(&line?)?;
    if entry.level == "error" {
        println!("{}", entry.msg);
    }
}
```

Constant memory regardless of file size, which is the point of JSONL.

With an HTTP client

```rust
// reqwest with the "json" feature
let users: Vec<User> = reqwest::get("https://api.example.com/users")
    .await?
    .json()
    .await?;

// posting
let res = client.post("https://api.example.com/users")
    .json(&new_user)          // sets Content-Type for you
    .send()
    .await?;
```

In [[Actix Web]] handlers, `web::Json<T>` does the same on the way in and out:

```rust
use actix_web::Responder;

async fn create(user: web::Json<NewUser>) -> impl Responder {
    HttpResponse::Ok().json(User { id: 1, name: user.name.clone() })
}
```

Big integers

Rust handles these correctly by default — `u64` is a real 64-bit integer, unlike the float everything becomes in JavaScript. One less thing to worry about, though see [[JSON]] if you are talking to a JS client that will mangle them.

Generating structs from a sample

```bash
npx quicktype -s json -o models.rs --lang rust samples/users.json
```

Then fix the optionality by hand, using what you learned from [[Investigating an API]].
