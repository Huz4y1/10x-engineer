A crate is your rust project, When you run cargo new project, project is now a crate and everything inside of project belongs to the crate called project

When you write

```rust
crate::services::berry_service
```

you're saying:

"Start at the root of **my project**, then go into `services`, then `berry_service`."

  

## what is a module

A module is a way to organise your code

for example:

```rust
services/
    berry_service.rs
    pokemon_service.rs
```

That directory called services is a module

  

## Why do we need `mod.rs`

suppose you have:

```rust
services/
    berry_service.rs
    pokemon_service.rs
```

Rust doesn’t automatically expose these files

Which means you have to create a [mod.rs](http://mod.rs) file inside that folder

inside that [mod.rs](http://mod.rs) file you write

```rust
pub mod berry_service;
pub mod pokemon_service;
```

You’re telling rust that these files belong to the service module

  

## What does “use” do?

Suppose you have:

```rust
crate::services::berry_service::get_berry
```

You have to write that line everytime

but when you write

```rust
use crate::services::berry_service::get_berry
```

Now you only need to write

```rust
berry_service::get_berry(...)
```

its exactly like imports in python

```python
from services import berry_service
```