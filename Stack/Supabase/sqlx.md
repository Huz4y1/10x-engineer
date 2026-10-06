---
tags: [rust, crate, sql, database, sqlx]
---

# sqlx

Async SQL for Rust, with **queries checked at compile time**. Stack: [[Supabase]] · Crates: [[Rust crates]]

```toml
[dependencies]
sqlx = { version = "0.8", features = ["runtime-tokio", "tls-rustls", "postgres", "macros", "chrono", "uuid"] }
```

---

## The idea

sqlx is **not an ORM**. You write SQL. Its trick is that the `query!` macro connects to your real database **at compile time** and verifies the SQL and the types.

```rust
let rows = sqlx::query!("SELECT id, name FROM customers WHERE country = $1", "UK")
    .fetch_all(&pool).await?;
```

> **Typo a column name and it fails to compile.** Not at runtime, not in production — at `cargo build`. That's the whole reason to choose sqlx over a query builder.

## Connecting

```rust
use sqlx::postgres::PgPoolOptions;

let pool = PgPoolOptions::new()
    .max_connections(5)
    .acquire_timeout(Duration::from_secs(5))
    .connect(&std::env::var("DATABASE_URL")?)
    .await?;
```

> **One pool for the whole application**, created at startup and shared (it's cheap to clone). Same rule as [[SQLAlchemy]] in Python — a pool per request exhausts the database.

For [[Supabase]] the URL is on the project's Database settings page:
```
postgres://postgres:[PASSWORD]@db.[PROJECT].supabase.co:5432/postgres
```

## Compile-time checking setup

The macros need a database to check against:

```bash
echo 'DATABASE_URL=postgres://user:pw@localhost/mydb' > .env
cargo install sqlx-cli
cargo sqlx prepare        # caches the schema into .sqlx/ so CI can build offline
```

> ⚠️ **Commit the `.sqlx/` directory.** Without it, CI and Docker builds fail because they can't reach your database ([[CI-CD pipelines]]).

## Querying

```rust
// one row, into a struct - checked
let c = sqlx::query!("SELECT id, name FROM customers WHERE id = $1", id)
    .fetch_one(&pool).await?;
println!("{}", c.name);          // typed

// into YOUR struct
#[derive(sqlx::FromRow)]
struct Customer { id: i32, name: String, country: Option<String> }

let c = sqlx::query_as!(Customer, "SELECT id, name, country FROM customers WHERE id = $1", id)
    .fetch_one(&pool).await?;

let all: Vec<Customer> = sqlx::query_as!(Customer, "SELECT id, name, country FROM customers")
    .fetch_all(&pool).await?;

let maybe = sqlx::query!("SELECT * FROM customers WHERE id = $1", id)
    .fetch_optional(&pool).await?;      // None instead of an error
```

| Method | Returns |
|---|---|
| `fetch_one` | One row, **errors** if none |
| `fetch_optional` | `Option<Row>` |
| `fetch_all` | `Vec<Row>` |
| `fetch` | A stream — for large results |
| `execute` | Just the row count |

> **`Option<String>` for nullable columns.** sqlx makes nullability part of the type, so the compiler forces you to handle it ([[Rust]]).

## Writing

```rust
sqlx::query!("INSERT INTO readings (device_id, temp) VALUES ($1, $2)", id, temp)
    .execute(&pool).await?;

let rec = sqlx::query!(
    "INSERT INTO customers (name) VALUES ($1) RETURNING id",
    name
).fetch_one(&pool).await?;
println!("new id {}", rec.id);
```

> **`$1`, `$2` are Postgres placeholders** (MySQL uses `?`). They're parameters, so **SQL injection is impossible** — you can't build a query by string concatenation with the macro even if you wanted to ([[26 — SECURITY]]).

## Transactions

```rust
let mut tx = pool.begin().await?;

sqlx::query!("UPDATE accounts SET balance = balance - $1 WHERE id = $2", amt, from)
    .execute(&mut *tx).await?;
sqlx::query!("UPDATE accounts SET balance = balance + $1 WHERE id = $2", amt, to)
    .execute(&mut *tx).await?;

tx.commit().await?;      // dropping without commit = automatic ROLLBACK
```

## Dynamic queries

The macro needs a literal string, so for dynamic SQL use the non-macro form (no compile-time checking):

```rust
let mut q = String::from("SELECT * FROM customers WHERE 1=1");
if country.is_some() { q.push_str(" AND country = $1"); }

let rows = sqlx::query_as::<_, Customer>(&q)
    .bind(country)
    .fetch_all(&pool).await?;
```

> **Still use `.bind()`, never `format!` values into the SQL.** Only the *structure* should be dynamic.

## Migrations

```bash
sqlx migrate add create_customers      # creates migrations/<timestamp>_create_customers.sql
sqlx migrate run
sqlx migrate revert
```

```rust
sqlx::migrate!("./migrations").run(&pool).await?;   // run at startup
```

## With Actix Web

```rust
let pool = PgPoolOptions::new().max_connections(5).connect(&url).await?;

HttpServer::new(move || {
    App::new()
        .app_data(web::Data::new(pool.clone()))     // cheap - it's an Arc
        .service(get_customer)
})
.bind(("0.0.0.0", 8080))?.run().await
```

```rust
use actix_web::{Responder, get};

#[get("/customers/{id}")]
async fn get_customer(pool: web::Data<PgPool>, id: web::Path<i32>) -> impl Responder {
    match sqlx::query_as!(Customer, "SELECT id, name, country FROM customers WHERE id = $1", *id)
        .fetch_optional(pool.get_ref()).await
    {
        Ok(Some(c)) => HttpResponse::Ok().json(c),
        Ok(None)    => HttpResponse::NotFound().finish(),
        Err(e)      => { eprintln!("{e}"); HttpResponse::InternalServerError().finish() }
    }
}
```

## sqlx vs the alternatives

| | sqlx | Diesel | SeaORM |
|---|---|---|---|
| Style | **Write SQL** | Query builder DSL | ORM |
| Compile-time checked | ✅ Against a real DB | ✅ Against its schema | Partly |
| Async | ✅ | Partly | ✅ |
| Learning curve | Low if you know SQL | Steeper | Medium |

> **Choose sqlx if you already know SQL** ([[SQL fundamentals]]) — you keep full control and gain compile-time safety. Diesel and SeaORM hide the SQL behind a DSL.

## Common mistakes

| Mistake | Fix |
|---|---|
| Build fails in CI/Docker | Commit `.sqlx/` after `cargo sqlx prepare` |
| Macro can't find `DATABASE_URL` | Put it in `.env` |
| Pool per request | One pool at startup, clone it |
| Nullable column into `String` | Use `Option<String>` |
| Transaction silently rolled back | You forgot `tx.commit()` |
| `fetch_one` errors on no rows | `fetch_optional` |

## Related

[[Supabase]] · [[Rust crates]] · [[tokio]] · [[Actix Web]] · [[SQL fundamentals]] · [[PostgreSQL reference]] · [[SQLAlchemy]]
