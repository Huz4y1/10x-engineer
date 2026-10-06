- A user can hit a url endpoint
- that endpoint will be display the pokemon’s data using pokeapi

We first need to think about the architecture of this project of how it will all work together

  

1. User hits URL
2. The main server will have that URL registered
3. The URL will call a handler function, this handler function will take the data from the URL and extract it, then pass it down to the service function
4. The service function will call the pokeAPI and pass in the data as an argument, the pokeAPI will return JSON, the service then turns that JSON into a rust struct
5. Service returns the struct to the handler function, handler function then converts that rust struct into a HTTP response
6. Response is sent to the browser

  

Setup the server app

```rust
use actix_web::{get, App, HttpServer, Responder};

mod handlers;
mod services;
mod models;

use handlers::pokemon_handler::get_pokemon;

#[get("/")]
async fn hello() -> impl Responder {
    "Hello from Actix!"
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(hello)
            .service(get_pokemon)
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

  

define the data that will be returned to the user

```rust
use serde::Serialize;

#[derive(Serialize)]
pub struct Pokemon {
    pub name: String,
    pub height: i32,
    pub weight: i32,
    pub types: Vec<String>,
}
```

  

create the service function, this is the brains and business logic this is where all the work happens

```rust
use reqwest;
use crate::models::pokemon::Pokemon;
use serde::Deserialize;

#[derive(Deserialize)]
struct PokeApiResponse {
    name: String,
    height: i32,
    weight: i32,
    types: Vec<PokeType>,
}

#[derive(Deserialize)]
struct PokeType {
    #[serde(rename = "type")]
    type_info: TypeInfo,
}

#[derive(Deserialize)]
struct TypeInfo {
    name: String,
}

pub async fn get_pokemon(name: String) -> Result<Pokemon, reqwest::Error> {
    let url = format!("https://pokeapi.co/api/v2/pokemon/{}", name);

    let response = reqwest::get(&url).await?;

    let data: PokeApiResponse = response.json().await?;

    let pokemon = Pokemon {
        name: data.name,
        height: data.height,
        weight: data.weight,
        types: data.types.into_iter().map(|t| t.type_info.name).collect(),
    };

    Ok(pokemon)
}
```

- `reqwest` → makes HTTP requests (like `fetch` in JS)
- `Pokemon` → your clean output model
- `Deserialize` → allows JSON → Rust struct conversion

  

struct PokeApiResponse = temporary struct that matches the pokeAPI’s data when it comes

PokeAPI will send data like

```rust
{
  "name": "pikachu",
  "height": 4,
  "weight": 60,
  "types": [...]
}
```

We need to create a struct to parse that JSON into a struct

The poke type struct is nested it shows data like

```rust
"types": [
  {
    "type": {
      "name": "electric"
    }
  }
]
```

- `type` is a reserved keyword in Rust → renamed to `type_info`
- `serde(rename = "type")` maps JSON field `"type"` → Rust field `type_info`

  

Now lets look at the main service function

```rust
pub async fn get_pokemon(name: String) -> Result<Pokemon, reqwest::Error>
```

### What this means:

- input: `"pikachu"`
- output: either:
    - `Pokemon` (success)
    - or error from HTTP request

  

```rust
let url = format!("https://pokeapi.co/api/v2/pokemon/{}", name);
```

format! allows you to push variables into the created string in 1 line without needing to make a completely new variable with the new url

If:

```
name = "pikachu"
```

Then:

```
https://pokeapi.co/api/v2/pokemon/pikachu
```

  

Now need to call the external API which is the PokeAPI

```rust
let response = reqwest::get(&url).await?;
```

### What happens here:

- sends HTTP GET request to the defined URL
- waits asynchronously
- `?` means: if error → return early

  

time to parse JSON

```rust
letdata:PokeApiResponse=response.json().await?;
```

This means:  
“Take the HTTP response body (JSON) and convert it into a Rust struct called `PokeApiResponse`.”

```rust
response.json()
```

Reads the HTTP body

Tries to parse JSON

Converts it into Rust types using serde

So now:

data.name = "pikachu"  
data.height = 4  
data.types = [...]

  

transform the data

```rust
let pokemon = Pokemon {
    name: data.name,
    height: data.height,
    weight: data.weight,
    types: data.types.into_iter().map(|t| t.type_info.name).collect(),
};

Ok(pokemon)
```

  

now need to create the handler function, this extracts data from the URL and passes it to the service function so it can work with it

```rust
use actix_web::{get, web, HttpResponse, Responder};

use crate::services::pokemon_service;

#[get("/pokemon/{name}")]
pub async fn get_pokemon(path: web::Path<String>) -> impl Responder {
    let name = path.into_inner();

    match pokemon_service::get_pokemon(name).await {
        Ok(pokemon) => HttpResponse::Ok().json(pokemon),
        Err(_) => HttpResponse::InternalServerError().finish(),
    }
}
```

  

defining routes

```rust
use actix_web::get;

#[get("/pokemon/{name}")]
```

Extract URL parameter

```rust
pub async fn get_pokemon(path: web::Path<String>)
```

This means

path = "pikachu”

Now we need to convert the path

```rust
let name = path.into_inner();
```

This allows name = "pikachu”

Call service

```rust
pokemon_service::get_pokemon(name).await
```

Now we need to handle the result of what service gives to us back

```rust
match pokemon_service::get_pokemon(name).await {
	Ok(pokemon) => HttpResponse::Ok().json(pokemon),
	Err(_) => HttpResponse::InternalServerError().finish(),
}
```

the first result means

- status: 200
- converts struct → JSON automatically

And the second result means

- status: 500
- no body