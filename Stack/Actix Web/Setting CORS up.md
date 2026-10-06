  

```rust
use actix_cors::Cors;
use actix_web::{App, HttpServer, web, HttpResponse, get};

#[get("/api/test")]
async fn test() -> HttpResponse {
    HttpResponse::Ok().body("OK")
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        let cors = Cors::default()
            .allow_any_origin()   // 👈 allows all frontend requests (DEV ONLY)
            .allow_any_method()
            .allow_any_header();

        App::new()
            .wrap(cors)
            .service(test)
    })
    .bind("0.0.0.0:3000")?
    .run()
    .await
}
```

  

```rust
.allow_any_origin()  → allows Expo / browser to call API
.allow_any_method()  → GET, POST, PUT, DELETE
.allow_any_header()  → allows JSON headers
```