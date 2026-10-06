---
tags: [rust, ownership, memory, language]
---

# Ownership and borrowing

The one idea that makes Rust different from every other language. Everything else follows from it.

Language: [[Rust]] · Functions: [[Functions (Rust)]] · Memory in general: [[Memory]]

---

## The problem it solves

Every program has to free the memory it allocates. There are historically two ways:

| Approach | Language | Cost |
|---|---|---|
| **You do it manually** | [[C]], [[C++]] | Forget → leak. Do it twice → crash. Use after freeing → **security hole** |
| **A garbage collector does it** | [[Python]], Java, Go | Safe, but pauses your program at unpredictable moments |

Rust takes a third route: **the compiler works out where each value dies and inserts the cleanup for you.** No leaks, no double-frees, no garbage collector, no runtime cost.

The price is that you have to write code the compiler can prove is safe. That's what fighting the borrow checker is.

---

## The three rules

1. **Every value has exactly one owner.**
2. **When the owner goes out of scope, the value is dropped.**
3. **There can be one owner at a time** — assigning or passing transfers ownership.

```rust
{
    let s = String::from("hello");   // s owns the string
    // ...
}                                    // s goes out of scope -> memory freed automatically
```

---

## Move

```rust
let a = String::from("hello");
let b = a;                  // ownership MOVES to b
println!("{}", a);          // ERROR: borrow of moved value: `a`
```

In most languages `b = a` would give you two references to one string. In Rust, `a` is now invalid — the compiler will not let you touch it.

**Why:** if both `a` and `b` were valid, both would try to free the same memory at scope end. That's a double-free. Rust removes the possibility rather than trusting you.

Same thing happens when passing to a function:

```rust
fn consume(s: String) { }

let name = String::from("huz");
consume(name);
println!("{}", name);       // ERROR: value moved into consume
```

### Copy types are the exception

```rust
let x = 5;
let y = x;
println!("{}", x);          // FINE
```

Small fixed-size values on the stack (`i32`, `f64`, `bool`, `char`, and tuples of them) implement `Copy` — they're duplicated instead of moved, because copying a few bytes is free.

> **The dividing line:** if it lives entirely on the stack, it copies. If it owns heap memory (`String`, `Vec<T>`, `Box<T>`, `HashMap`), it moves.

### Clone when you actually want two

```rust
let a = String::from("hello");
let b = a.clone();          // deep copy - now there ARE two strings
println!("{} {}", a, b);    // fine
```

> ⚠️ **`.clone()` is not free** — it allocates and copies. It's the correct escape hatch when you genuinely need two owned copies, and the wrong reflex for silencing a borrow error. If you're cloning inside a hot loop, you wanted a borrow.

---

## Borrowing

Usually you don't want to *give away* a value — you want to let something look at it.

```rust
fn length(s: &String) -> usize {      // & = borrow
    s.len()
}

let name = String::from("huz");
let n = length(&name);
println!("{} is {}", name, n);        // name is still ours
```

A borrow is a reference. It doesn't own anything, so nothing is freed when it goes out of scope.

### Mutable borrows

```rust
fn shout(s: &mut String) {
    s.push_str("!!!");
}

let mut name = String::from("huz");   // must be `mut` to be borrowed mutably
shout(&mut name);
println!("{}", name);                 // huz!!!
```

### The borrowing rules

**At any given moment, you may have either:**
- **any number of immutable borrows (`&T`)**, or
- **exactly one mutable borrow (`&mut T`)**

**— never both.**

```rust
let mut s = String::from("hello");

let r1 = &s;
let r2 = &s;          // fine - many readers
println!("{} {}", r1, r2);

let r3 = &mut s;      // fine HERE, because r1 and r2 are no longer used
r3.push_str(" world");
```

```rust
let mut s = String::from("hello");
let r1 = &s;
let r2 = &mut s;              // ERROR: cannot borrow as mutable
println!("{} {}", r1, r2);    //        while borrowed as immutable
```

> **Why this rule exists:** it makes **data races impossible at compile time**. A data race needs two accesses, at least one writing, with no synchronisation. Rust's rules forbid that shape. This is why Rust calls it *fearless concurrency* — the same rule that annoys you in single-threaded code is what makes multithreaded code safe ([[tokio]]).

> **Borrows end at their last use**, not at the closing brace. This is called non-lexical lifetimes, and it's why the first example above compiles.

---

## Slices

A slice borrows *part* of something.

```rust
let s = String::from("hello world");
let hello = &s[0..5];        // &str - a borrowed view, no copy
let all = &s[..];

let v = vec![1, 2, 3, 4, 5];
let middle = &v[1..4];       // &[i32]
```

| Owned | Borrowed view |
|---|---|
| `String` | `&str` |
| `Vec<T>` | `&[T]` |

> **Take `&str` and `&[T]` as function parameters, return `String` and `Vec<T>`.** Borrowed types accept more callers; owned types are what you hand back. This single habit removes most beginner borrow errors ([[Functions (Rust)]]).

---

## Lifetimes

A lifetime is the compiler's name for *how long a reference is valid*. Usually it infers them and you never write one.

```rust
// The compiler needs to know: does the result borrow from x or y?
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

`'a` says: *the returned reference lives as long as the shorter of the two inputs.*

```rust
fn dangle() -> &String {          // ERROR: missing lifetime specifier
    let s = String::from("hi");
    &s                            // s dies here - this reference would dangle
}
```

> **This error is the borrow checker doing its job.** In C++ this compiles and returns a pointer to freed memory — a use-after-free bug that might work for months and then corrupt something. Rust simply refuses.

> **You rarely write lifetimes.** They appear on structs holding references, and on functions returning a reference derived from multiple inputs. If you're writing them everywhere, you probably want owned values instead.

---

## When the rules genuinely aren't enough

```rust
use std::rc::Rc;                        // many owners, single-threaded
use std::sync::{Arc, Mutex};            // many owners, across threads
use std::cell::RefCell;                 // mutate through a shared reference

let shared = Arc::new(Mutex::new(vec![1, 2, 3]));
let clone = Arc::clone(&shared);
std::thread::spawn(move || {
    clone.lock().unwrap().push(4);
});
```

| Type | For |
|---|---|
| `Rc<T>` | Shared ownership, one thread — a graph or tree |
| `Arc<T>` | Shared ownership across threads (atomic, slightly slower) |
| `RefCell<T>` | Borrow rules checked **at runtime** instead of compile time |
| `Mutex<T>` / `RwLock<T>` | Shared mutable state across threads |

> ⚠️ **`RefCell` moves the check to runtime — it panics instead of failing to compile.** You've traded a compiler error for a crash. Use it deliberately, not to escape a borrow error you don't understand.

> **`Arc<Mutex<T>>` is the standard shared-state pattern in an [[Actix Web]] handler.** Keep the lock held for as few lines as possible — holding it across an `.await` will deadlock your server.

---

## How to actually get unstuck

When the borrow checker rejects your code, in order:

1. **Read the error properly.** Rust's messages name the exact line where the conflicting borrow started and where it's still in use.
2. **Can you borrow instead of move?** Add `&`.
3. **Can you shorten the borrow?** Do the read, finish with it, then take the mutable borrow.
4. **Can you restructure?** Two mutable borrows of one struct often means the function should take the two fields, not the whole struct.
5. **Do you genuinely need two owners?** `Rc`/`Arc`.
6. **Only then, `.clone()`.** It's a legitimate answer — just not the first one.

> **The borrow checker is not being difficult for its own sake.** Every error it gives you is a bug that would have compiled in C++ and shown up in production six months later. Once the ownership model clicks, you stop fighting it and start using it as a design tool.

---

## Common mistakes

| Mistake | Error |
|---|---|
| Using a value after passing it by value | `borrow of moved value` |
| `&mut` on a non-`mut` binding | `cannot borrow as mutable` |
| Immutable and mutable borrow overlapping | `cannot borrow ... as mutable` |
| Returning a reference to a local | `missing lifetime specifier` |
| `&String` / `&Vec<T>` parameters | Unnecessarily narrow — use `&str` / `&[T]` |
| `.clone()` everywhere to silence errors | Works, but slow and hides the design problem |
| Holding a `Mutex` guard across `.await` | Deadlock |
| `RefCell` used to dodge the compiler | Runtime panic instead of a compile error |

## Related

[[Rust]] · [[Functions (Rust)]] · [[Types]] · [[Variables (Rust)]] · [[OOP concepts]] · [[Rust for this stack]] · [[tokio]] · [[Actix Web]] · [[Memory]] · [[C++]] · [[33 — SYSTEMS PERFORMANCE]]
