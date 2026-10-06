---
tags: [rust, functions, language]
---

# Functions (Rust)

Language: [[Rust]] · Crates: [[Rust crates]] · In this stack: [[Rust for this stack]]

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

// No need for a semi colon on the last line because it is automatically returned 
```

---

## The semicolon rule

That comment is the thing to internalise, because Rust uses it everywhere.

**An expression without a semicolon is the value. With a semicolon, it's a statement and evaluates to nothing.**

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b        // returns a + b
}

fn broken(a: i32, b: i32) -> i32 {
    a + b;       // ERROR: expected i32, found ()
}
```

> `()` is called the **unit type** — Rust's "no value". The error `expected i32, found ()` almost always means a stray semicolon on your last line.

This works for `if` and `match` too, which is why Rust rarely needs a `return`:

```rust
fn category(age: u32) -> &'static str {
    if age >= 18 { "adult" } else { "child" }      // if is an EXPRESSION
}

fn describe(n: i32) -> String {
    match n {
        0 => "zero".to_string(),
        x if x < 0 => format!("negative {}", -x),
        _ => format!("positive {}", n),
    }
}
```

`return` exists for early exits:

```rust
fn find(items: &[i32], target: i32) -> Option<usize> {
    for (i, &x) in items.iter().enumerate() {
        if x == target {
            return Some(i);       // early exit needs `return`
        }
    }
    None                          // final value, no semicolon
}
```

---

## Parameters: borrow, don't take

This is the part that separates Rust from every other language you know.

```rust
fn takes_ownership(s: String) { }        // s is MOVED in - caller can't use it after
fn borrows(s: &String) { }               // caller keeps it
fn borrows_mutably(s: &mut String) { }   // caller keeps it, you can change it
```

```rust
let name = String::from("huz");
takes_ownership(name);
println!("{}", name);      // ERROR: value borrowed after move
```

```rust
let name = String::from("huz");
borrows(&name);
println!("{}", name);      // fine - it was only borrowed
```

> **Default to borrowing (`&T`).** Take ownership only when the function genuinely needs to keep or consume the value. See [[Ownership and borrowing]].

**Prefer slices to owned types in parameters** — they accept more callers:

```rust
fn greet(name: &str) { }        // accepts &String AND &str AND a literal
fn greet(name: &String) { }     // accepts only &String - unnecessarily narrow

fn total(xs: &[i32]) -> i32 { xs.iter().sum() }    // accepts &Vec<i32> and arrays
```

---

## Return types that carry failure

Rust has no exceptions. Failure is in the return type.

```rust
fn divide(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 { None } else { Some(a / b) }
}

fn parse_age(s: &str) -> Result<u32, std::num::ParseIntError> {
    let n = s.parse::<u32>()?;      // ? returns the error early
    Ok(n)
}
```

| Type | Means |
|---|---|
| `Option<T>` | Might be absent — **no nulls in Rust** |
| `Result<T, E>` | Might fail, with a reason |
| `?` | If it's `Err`/`None`, return it now; otherwise unwrap it |

> **The `?` operator is why Rust error handling is pleasant.** It replaces the nesting you'd get from checking every call, without hiding the failure the way an exception does.

> ⚠️ **`.unwrap()` panics and kills the thread.** Fine in a test or a quick script, wrong in a server. Use `?`, or `.unwrap_or(default)`, or `match`.

## Generics and traits

```rust
fn largest<T: PartialOrd>(items: &[T]) -> &T {
    let mut max = &items[0];
    for x in items { if x > max { max = x; } }
    max
}

fn print_all(items: &[impl std::fmt::Display]) {
    for x in items { println!("{}", x); }
}
```

> **The trait bound (`T: PartialOrd`) is the promise.** It says "T must be comparable", and the compiler checks every call site. This is how Rust gets generics with zero runtime cost.

## Closures

```rust
let double = |x: i32| x * 2;
let nums: Vec<i32> = (1..=5).map(double).collect();

let threshold = 10;
let above = |x: &i32| **&x > threshold;      // captures `threshold` from around it

fn apply<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 { f(x) }
```

| Trait | The closure |
|---|---|
| `Fn` | Borrows what it captures — callable many times |
| `FnMut` | Mutably borrows |
| `FnOnce` | Takes ownership — callable **once** |

## Async functions

```rust
async fn fetch(url: &str) -> Result<String, reqwest::Error> {
    let body = reqwest::get(url).await?.text().await?;
    Ok(body)
}
```

> **An `async fn` returns a future and does nothing until it's `.await`ed.** Calling it without awaiting is a no-op the compiler will warn you about. See [[reqwest]] and [[Actix Web]].

## Common mistakes

| Mistake | Error you'll see |
|---|---|
| Semicolon on the return line | `expected i32, found ()` |
| Using a value after passing it by value | `borrow of moved value` |
| `&String` parameter instead of `&str` | Callers can't pass literals |
| `.unwrap()` in production code | Panic on the first bad input |
| Forgetting `.await` | `unused implementation of Future` |
| Two mutable borrows at once | `cannot borrow as mutable more than once` |

## Related

[[Rust]] · [[Ownership and borrowing]] · [[Rust crates]] · [[Rust for this stack]] · [[Actix Web]] · [[Selection (Rust)]] · [[OOP concepts]] · [[03 — PROGRAMMING]]
