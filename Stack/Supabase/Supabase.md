Go to:

[https://supabase.com](https://supabase.com/)

### Steps:

1. Click **New project**
2. Choose:
    - Organization
    - Project name (e.g. `actix-backend`)
    - Database password (IMPORTANT — save it)
    - Region (closest to you)
3. Wait ~1–2 minutes

You now have:

- PostgreSQL database
- Dashboard
- API layer (don’t need this)

  

# Create a table (your real database)

Go to:

**Table Editor → Create table**

Let’s recreate your earlier example: `tasks`

## Table: `tasks`

Add columns:

|Column|Type|Settings|
|---|---|---|
|id|bigint|primary key, auto-increment|
|title|text|not null|
|completed|boolean|default false|
|created_at|timestamp|default now()|

  

# Get your database connection string

Go to:

**Settings → Database → Connection string**

You’ll see something like:

```
postgresql://postgres:[PASSWORD]@db.xxxxx.supabase.co:5432/postgres
```

This is what `sqlx` uses.

Add sqlx and dotenvy into your dependancies

```rust
sqlx = { version = "0.7", features = ["runtime-tokio", "postgres", "macros"] }
dotenvy = "0.15"
```

  

## Add environment variable

Create a `.env` file:

```
DATABASE_URL=postgresql://postgres:YOUR_PASSWORD@db.xxxxx.supabase.co:5432/postgres
```

  

## Create DB pool

in main.rs

```rust
use sqlx::postgres::PgPoolOptions;
use std::env;

#[actix_web::main]
async fn main() -> std::io::Result<()> {

    dotenvy::dotenv().ok();

    let db_url = env::var("DATABASE_URL").expect("DATABASE_URL not set");

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&db_url)
        .await
        .expect("Failed to connect to DB");

    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(pool.clone()))
            .service(create_task)
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

  

Now your handler and service functions will look a little different compared to using normal APIs

Handler function example:

```rust
 use actix_web::{post, web, HttpResponse, Responder};
use crate::services::task_service;

#[post("/tasks")]
pub async fn create_task(
    pool: web::Data<PgPool>,
    body: web::Json<CreateTask>,
) -> impl Responder {

    let data = body.into_inner();

    let result = task_service::create_task(
        &pool,
        data.title,
        data.completed
    ).await;

    match result {
        Ok(_) => HttpResponse::Created().finish(),
        Err(_) => HttpResponse::InternalServerError().finish(),
    }
}
```

  

service function example:

```rust
use sqlx::PgPool;

pub async fn create_task(
    pool: &PgPool,
    title: String,
    completed: bool,
) -> Result<(), sqlx::Error> {

    sqlx::query!(
        r#"
        INSERT INTO tasks (title, completed)
        VALUES ($1, $2)
        "#,
        title,
        completed
    )
    .execute(pool)
    .await?;

    Ok(())
}
```

  

[[sqlx]]