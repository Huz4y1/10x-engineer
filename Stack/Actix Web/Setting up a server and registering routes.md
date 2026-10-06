```rust
use actix_web::{get, App, HttpServer, Responder};

#[get("/")]
async fn hello() -> impl Responder {
    "Hello, World!"
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(hello)
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

## What each part of this code does

```rust
use actix_web::{get, App, HttpServer, Responder};
```

Imports everything you need:

- `get` → create GET routes
- `App` → register routes and middleware
- `HttpServer` → create the web server
- `Responder` → something that can be turned into an HTTP response

  

```rust
use actix_web::get;

#[get("/")]
```

This registers a route and calls the handler function below when a user hits the URL. Usually the route along with the handler function should be in the handlers folder just modularity

  

```rust
.service(hello)
```

This registers the hello handler to the server. Without this line that endpoint doesn’t exist on the server