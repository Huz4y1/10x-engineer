A handler function is a function which deals with the HTTP request. It turns the HTTP request into something Rust can work with and returns that to the service function when it calls it

A service function deals with the passed down content using plain rust and does all the business logic and returns something back to the handler function so it can turn the Rust into a HTTP response so that can be sent back to the client

  

GET requets

Handler function template:

```rust
use actix_web::Responder;

#[http_method("/route")] 
pub async fn handler_name(/* 1. Extract HTTP data*/) -> impl Responder {

    // 2. Convert extracted data if needed

    // 3. Call service

    // 4. Handle success/error

    // 5. Return HTTP response
}
```

Example of a handler function:

```rust
use actix_web::{get, web, HttpResponse, Responder};

use crate::services::book_service;

#[get("/books/{isbn}")]
pub async fn get_book(path: web::Path<String>) -> impl Responder {
    // 1. Extract data from the request
    let isbn = path.into_inner();

    // 2. Call the service
    let result = book_service::get_book(isbn).await;

    // 3. Turn the service result into an HTTP response
    match result {
        Ok(book) => HttpResponse::Ok().json(book),
        Err(_) => HttpResponse::InternalServerError().finish(),
    }
}
```

  

Service function template:

```rust
pub async fn function_name(/* input*/) -> Result<OutputType, ErrorType> {

    // 1. Prepare data

    // 2. Access data source

    // 3. Parse/deserialize

    // 4. Business logic

    // 5. Transform into your model

    // 6. Return Result
}
```

An example of a service function:

```rust
pub async fn get_book(isbn: String) -> Result<Book, reqwest::Error> {
    // 1. Build URL
    let url = format!("https://example.com/api/books/{}", isbn);

    // 2. Request data
    let response = reqwest::get(&url).await?;

    // 3. Convert JSON into Rust structs
    let data: BookApiResponse = response.json().await?;

    // 4. Transform into your model
    let book = Book {
        title: data.title,
        author: data.author,
        pages: data.pages,
    };

    // 5. Return success
    Ok(book)
}
```

  

  

POST requests

  

suppose the income request contains this body:

```json
{
    "title": "Learn Rust",
    "completed": false
}
```

example handler function:

```rust
use actix_web::{post, web, HttpResponse, Responder};

use crate::models::task::CreateTask;
use crate::services::task_service;

// Register this function as the handler for POST /tasks
#[post("/tasks")]
pub async fn create_task(

    // Extract JSON from the request body and convert it into CreateTask
    body: web::Json<CreateTask>,

) -> impl Responder {

    // Remove the CreateTask struct from Actix's Json wrapper
    let task_data = body.into_inner();

    // Pass the data to the service layer
    // The handler does NOT know how to create tasks.
    let result = task_service::create_task(task_data).await;

    // Look at what the service returned
    match result {

        // Service succeeded
        Ok(task) => {

            // Return HTTP 201 Created
            // Convert the Rust struct into JSON automatically
            HttpResponse::Created().json(task)
        }

        // Service failed
        Err(_) => {

            // Return an HTTP error
            HttpResponse::InternalServerError().finish()
        }
    }
}
```

Service function example:

```rust
use crate::models::task::{CreateTask, Task};

pub async fn create_task(

    // Receive data from the handler
    task_data: CreateTask,

) -> Result<Task, String> {

    // ----------------------------
    // Business Logic Starts Here
    // ----------------------------

    // Check if the title is empty
    if task_data.title.trim().is_empty() {

        // Return an error immediately
        return Err("Title cannot be empty".to_string());
    }

    // Pretend we saved it to a database
    // Normally this is where you'd insert into PostgreSQL/MySQL/etc.

    // Build the Task we'll return
    let task = Task {

        // Pretend the database generated this ID
        id: 1,

        // Move the title from CreateTask into Task
        title: task_data.title,

        // Move the completed value
        completed: task_data.completed,
    };

    // Everything succeeded
    Ok(task)
}
```

  

## More scenarios of handler functions:

### Scenario 1: `web::Path` — "get me _this specific_ thing"

Clicking one event card to see its details. The identifier travels in the URL itself:

```rust
use actix_web::{Responder, get};

#[get("/api/ufc/events/{slug}")]
pub async fn get_event(slug: web::Path<String>) -> impl Responder {
    let slug = slug.into_inner();   // unwrap Path<String> → String

    match ufc_schedule_service::get_event_by_slug(slug).await {
        Ok(event) => HttpResponse::Ok().json(event),
        Err(_) => HttpResponse::NotFound().body("Event not found"),
    }
}
```

frontend

```tsx

const res = await fetch(`http://localhost:8080/api/ufc/events/${slug}`);
```

### Scenario 2: `web::Query` — "get me things, _filtered/configured_"

A schedule page with optional filters: `/api/ufc/events?status=scheduled&limit=5`. Query params are the natural home for _optional modifiers_ of a GET:

```rust
use actix_web::{Responder, get};
use serde::Deserialize;

#[derive(Deserialize)]
pub struct ScheduleFilters {
    status: Option<String>,   // Option = the param may be absent
    limit: Option<u32>,
}

#[get("/api/ufc/events")]
pub async fn list_events(q: web::Query<ScheduleFilters>) -> impl Responder {
    let filters = q.into_inner();
    let limit = filters.limit.unwrap_or(10);   // default when not supplied

    match ufc_schedule_service::get_schedule_filtered(filters.status, limit).await {
        Ok(events) => HttpResponse::Ok().json(events),
        Err(e) => {
            eprintln!("fetch failed: {e}");
            HttpResponse::BadGateway().body("Upstream failed")
        }
    }
}
```

frontend

```tsx
const res = await fetch('http://localhost:8080/api/ufc/events?status=scheduled&limit=5');
// or built safely from variables:
const params = new URLSearchParams({ status: 'scheduled', limit: '5' });
const res = await fetch(`http://localhost:8080/api/ufc/events?${params}`);
```

### Scenario 3: `web::Json` — "here's data, _store/process it_"

Say you add a feature: users save events to a watchlist. Creating data → POST → body:

```rust
use actix_web::{Responder, post};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]                 // incoming — parsed FROM the request
pub struct NewWatchlistItem {
    event_slug: String,
    note: Option<String>,
}

#[derive(Serialize)]                   // outgoing — sent back in the response
pub struct WatchlistItem {
    id: u32,
    event_slug: String,
    note: Option<String>,
}

#[post("/api/watchlist")]
pub async fn add_to_watchlist(body: web::Json<NewWatchlistItem>) -> impl Responder {
    let item = body.into_inner();

    match watchlist_service::add(item).await {
        Ok(created) => HttpResponse::Created().json(created),   // 201, echo the created item
        Err(_) => HttpResponse::InternalServerError().body("Could not save"),
    }
}
```

frontend

```tsx
const res = await fetch('http://localhost:8080/api/watchlist', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },   // Json<T> REQUIRES this header
  body: JSON.stringify({ event_slug: 'ufc-329', note: 'McGregor card!' }),
});
const created = await res.json();
```