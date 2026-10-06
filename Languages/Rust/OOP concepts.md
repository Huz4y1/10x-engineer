---
tags: [rust, structs, traits, oop, language]
---

# OOP concepts

Language: [[Rust]] · Ownership: [[Ownership and borrowing]] · Functions: [[Functions (Rust)]]

```rust
let user = User {
    name: String::from("Bob"),
    age: 25,
};

println!("{}", user.name);
```

---

## Rust is not object-oriented, and that's the point

There are **no classes and no inheritance** in Rust. Instead there are three pieces:

| Piece | Job |
|---|---|
| **`struct`** | Holds the data |
| **`impl`** | Holds the methods |
| **`trait`** | Defines shared behaviour |

Where a class fuses all three, Rust keeps them separate. That's what lets you add methods to types you didn't write.

---

## Structs

```rust
struct User {
    name: String,
    age: u32,
    email: Option<String>,      // Option instead of null
}

let user = User {
    name: String::from("Bob"),
    age: 25,
    email: None,
};
```

```rust
struct Point(f64, f64);            // tuple struct - fields by position
let p = Point(1.0, 2.0);
println!("{}", p.0);

struct Marker;                     // unit struct - no data, used as a type-level tag
```

> **Fields are private outside their module by default.** Add `pub` to expose them ([[Crates, Modules and modularisation]]).

```rust
let older = User { age: 26, ..user };     // struct update syntax - takes the rest from `user`
```

> ⚠️ **`..user` *moves* any non-`Copy` field.** After that line `user.name` is gone. This surprises people ([[Ownership and borrowing]]).

## Methods — `impl`

```rust
impl User {
    // associated function - no self. This is how you write a constructor.
    fn new(name: &str, age: u32) -> Self {
        Self { name: name.to_string(), age, email: None }
    }

    fn is_adult(&self) -> bool { self.age >= 18 }        // borrows
    fn birthday(&mut self) { self.age += 1; }            // borrows mutably
    fn into_name(self) -> String { self.name }           // CONSUMES - user is gone after
}

let mut user = User::new("Bob", 25);     // :: for associated functions
user.birthday();                          // .  for methods
println!("{}", user.is_adult());
```

| First parameter | Means |
|---|---|
| `&self` | Read the value — **the default** |
| `&mut self` | Modify it |
| `self` | Consume it; the caller can't use it afterwards |
| *(none)* | Associated function — called as `Type::name()` |

> **Rust has no constructors.** `new` is a convention, not a keyword. You can have `from_json`, `with_capacity`, `default` — as many as make sense.

---

## Traits — shared behaviour

A trait is a set of methods a type promises to provide. This is Rust's interface.

```rust
trait Describe {
    fn describe(&self) -> String;

    fn shout(&self) -> String {            // a DEFAULT implementation
        self.describe().to_uppercase()
    }
}

impl Describe for User {
    fn describe(&self) -> String {
        format!("{} ({})", self.name, self.age)
    }
}

println!("{}", user.shout());
```

> **You can implement your own trait for types you didn't write** — `impl Describe for String` is legal. This is how Rust replaces inheritance, and it's more flexible: behaviour is added from outside rather than baked into a hierarchy.

### Traits worth knowing

```rust
#[derive(Debug, Clone, PartialEq, Default)]
struct User { name: String, age: u32 }

println!("{:?}", user);        // Debug
println!("{:#?}", user);       // Debug, pretty-printed
```

| Trait | Gives you |
|---|---|
| `Debug` | `{:?}` printing — **derive it on everything** |
| `Clone` | `.clone()` |
| `Copy` | Implicit copying (small stack types only) |
| `PartialEq` / `Eq` | `==` |
| `PartialOrd` / `Ord` | `<`, sorting |
| `Default` | `Type::default()` |
| `Hash` | Use as a `HashMap` key |
| `Display` | `{}` printing — **write by hand**, it's the user-facing format |
| `From` / `Into` | Conversions |
| `Iterator` | Works in a `for` loop |
| `Serialize` / `Deserialize` | JSON — see [[Serde]] |

```rust
use std::fmt;
impl fmt::Display for User {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{} ({})", self.name, self.age)
    }
}
```

> **`Debug` is for you, `Display` is for users.** That's why `Debug` can be derived and `Display` can't — only you know how it should read.

### Traits as parameters

```rust
fn print_it(item: &impl Describe) { println!("{}", item.describe()); }
fn print_it<T: Describe>(item: &T) { }                    // same thing
fn print_it<T>(item: &T) where T: Describe + Clone { }    // multiple bounds

let items: Vec<Box<dyn Describe>> = vec![Box::new(user), Box::new(other)];
```

| Form | How it works |
|---|---|
| `impl Trait` / `<T: Trait>` | **Static dispatch** — resolved at compile time, zero cost |
| `Box<dyn Trait>` | **Dynamic dispatch** — decided at runtime, small cost, allows mixed types in one collection |

> **Prefer generics; reach for `dyn` when you need a collection of different types behind one trait.**

---

## Enums — the feature you'll miss elsewhere

```rust
enum Status {
    Active,
    Suspended { reason: String, until: u64 },    // variants can CARRY data
    Deleted(u64),
}

match user.status {
    Status::Active => println!("fine"),
    Status::Suspended { reason, .. } => println!("suspended: {}", reason),
    Status::Deleted(ts) => println!("deleted at {}", ts),
}
```

> **`match` must be exhaustive.** Add a variant to the enum and every `match` that doesn't handle it fails to compile — the compiler shows you every place that needs updating. This is one of Rust's best features ([[Selection (Rust)]]).

`Option` and `Result` are just enums:

```rust
enum Option<T> { Some(T), None }
enum Result<T, E> { Ok(T), Err(E) }
```

> **This is why Rust has no null.** "Might be missing" is encoded in the type, so the compiler forces you to handle the missing case.

---

## Composition instead of inheritance

```rust
struct Engine { hours: f64 }
struct Aircraft { engine: Engine, tail: String }   // HAS-A, not IS-A

impl Aircraft {
    fn hours(&self) -> f64 { self.engine.hours }   // delegate explicitly
}
```

> **Rust deliberately has no inheritance.** Shared behaviour goes in a trait; shared data goes in a nested struct. Deep class hierarchies are a common source of hard-to-follow code, and Rust removes the option.

## Common mistakes

| Mistake | Fix |
|---|---|
| Looking for `class` | `struct` + `impl` |
| Looking for inheritance | Traits for behaviour, composition for data |
| `self` where you meant `&self` | Method consumes the value |
| Forgetting `#[derive(Debug)]` | Can't `{:?}` print it |
| `{}` on a type without `Display` | Use `{:?}`, or implement `Display` |
| `..other` moving fields unexpectedly | Derive `Clone`, or move the field last |
| `Box<dyn Trait>` everywhere | Generics are faster; use `dyn` only for mixed collections |
| Non-exhaustive `match` | The compiler tells you exactly what's missing |

## Related

[[Rust]] · [[Ownership and borrowing]] · [[Functions (Rust)]] · [[Types]] · [[Selection (Rust)]] · [[Crates, Modules and modularisation]] · [[Serde]] · [[Rust for this stack]] · [[C++]] · [[Classes]]
