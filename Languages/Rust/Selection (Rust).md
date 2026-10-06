---
tags: [rust, control-flow, match, language]
---

# Selection (Rust)

Language: [[Rust]] · Structs and enums: [[OOP concepts]] · Functions: [[Functions (Rust)]]

```rust
let age = 20;

if age >= 18 {
    println!("Adult");
} else {
    println!("Child");
}
```

---

## `if` is an expression

The condition needs no brackets, and **the whole `if` produces a value**:

```rust
let label = if age >= 18 { "Adult" } else { "Child" };
```

> ⚠️ **Both branches must have the same type**, and if you use it as a value you need the `else`. That's why Rust has no ternary operator — `if` already is one.

> ⚠️ **The condition must be a `bool`.** `if 1 { }` doesn't compile — no truthiness. `if let Some(x) = opt` is how you test "is there a value".

```rust
if age >= 65 { "senior" }
else if age >= 18 { "adult" }
else { "child" };
```

---

## `match` — the one you'll use most

```rust
let description = match age {
    0 => "newborn",
    1..=12 => "child",
    13..=19 => "teenager",
    n if n >= 65 => "senior",        // a guard
    _ => "adult",                    // catch-all
};
```

| Pattern | Matches |
|---|---|
| `5` | That exact value |
| `1..=12` | An inclusive range |
| `"a" \| "b"` | Either |
| `n if n > 10` | With a condition attached |
| `_` | Anything (and discards it) |
| `n` | Anything, **binding it to `n`** |

> ⚠️ **`match` must be exhaustive.** Miss a case and it doesn't compile. This is the feature — add a variant to an enum and the compiler lists every `match` you now need to update.

> **`_` defeats that safety.** On an enum, prefer listing the variants so the compiler keeps telling you when something changes.

### Matching enums — the real use

```rust
match status {
    Status::Active => println!("running"),
    Status::Suspended { reason, until } => println!("{} until {}", reason, until),
    Status::Deleted(ts) => println!("gone at {}", ts),
}
```

```rust
match divide(10.0, 0.0) {
    Some(v) => println!("{}", v),
    None => println!("cannot divide by zero"),
}

match std::fs::read_to_string("config.toml") {
    Ok(text) => parse(&text),
    Err(e) if e.kind() == std::io::ErrorKind::NotFound => default_config(),
    Err(e) => return Err(e.into()),
}
```

## `if let` — match one case, ignore the rest

```rust
if let Some(name) = maybe_name {
    println!("hello {}", name);
}

if let Some(n) = maybe_n {
    println!("{}", n);
} else {
    println!("nothing");
}

// let-else: bind, or bail out. Keeps the happy path unindented.
let Some(config) = load_config() else {
    return Err("no config".into());
};
println!("{}", config.name);       // config is in scope from here
```

> **`let ... else` is the tidiest way to handle "give up if this is missing".** No nesting, and the value stays available afterwards.

## `while let`

```rust
while let Some(top) = stack.pop() {
    println!("{}", top);
}
```

## `loop` — and it returns a value

```rust
let result = loop {
    attempts += 1;
    if let Some(v) = try_connect() {
        break v;                   // break WITH a value
    }
    if attempts > 5 { break Default::default(); }
};
```

```rust
'outer: for i in 0..10 {
    for j in 0..10 {
        if grid[i][j] == target { break 'outer; }    // labelled break
    }
}
```

> **`loop` is not `while true`.** The compiler knows `loop` only exits via `break`, so it can prove things about types that `while true` cannot.

---

## Handling `Option` and `Result` without `match`

Most of the time you don't need a full `match`:

```rust
opt.unwrap_or(0)
opt.unwrap_or_else(|| expensive())
opt.unwrap_or_default()
opt.map(|x| x * 2)
opt.and_then(|x| divide(x, 2.0))
opt.filter(|x| *x > 10)
opt.is_some() / opt.is_none()

res.unwrap_or(0)
res.ok()                    // Result<T, E> -> Option<T>
res.map_err(|e| MyError::from(e))
res?                        // return the error early
```

> ⚠️ **`.unwrap()` and `.expect()` panic.** Acceptable in tests and prototypes; in a server they kill the request handler. Use `?` or an `unwrap_or` variant. `.expect("reason")` is better than `.unwrap()` because the panic message tells you which one blew up.

---

## Common mistakes

| Mistake | Result |
|---|---|
| `if (x > 5)` with brackets | Compiles, but a style warning |
| `if 1 { }` | Doesn't compile — no truthiness |
| Branches with different types | Type mismatch |
| Non-exhaustive `match` | Compile error (this is a feature) |
| `_` on an enum match | Silently ignores new variants later |
| `.unwrap()` in production | Panic on the first unexpected input |
| Deep `if let` nesting | Use `let ... else` or `?` |
| `while true` | Use `loop` |

## Related

[[Rust]] · [[Functions (Rust)]] · [[OOP concepts]] · [[Ownership and borrowing]] · [[Iteration]] · [[Types]] · [[Selection (C++)]] · [[03 — PROGRAMMING]]
