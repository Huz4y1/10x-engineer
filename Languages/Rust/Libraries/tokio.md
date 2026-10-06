---
tags: [rust, crate, async, tokio]
---

# tokio

The async runtime almost every Rust network program is built on.

Index: [[Rust crates]] · Language: [[Rust]] · In this stack: [[Rust for this stack]]

### Installing

Rust crates are added with `cargo`, not pip. **Identical on Windows, WSL and Linux.**

```bash
cargo add tokio --features full        # everything - start here
cargo add tokio --features rt-multi-thread,macros,net,time,sync   # trimmed, for release
```

Or by hand in `Cargo.toml`:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```bash
cargo build                            # downloads and compiles it
cargo tree | grep tokio                # check which version you actually got
```

**Getting Rust itself:**

| | Windows | WSL / Linux |
|---|---|---|
| Install | `winget install Rustlang.Rustup` | `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \| sh` |
| Also needed | **Visual Studio Build Tools** (C++ workload) | `sudo apt install build-essential pkg-config libssl-dev` |
| Check | `rustc --version && cargo --version` | same |

> ⚠️ **On Windows, Rust needs the MSVC linker.** Without Visual Studio Build Tools you get *link.exe not found*. `rustup` offers to install them — say yes.

> ⚠️ **`--features full` pulls in everything.** Fine while learning; trim to what you use before shipping, since it affects compile time and binary size.

> **Build in WSL, not `/mnt/c/`.** Cargo writes thousands of files to `target/`, and doing that across the filesystem boundary is dramatically slower ([[Setting up a dev machine]]).

---

## Why it exists

Rust's `async`/`await` is syntax only — **the language ships no runtime**. Something has to actually poll the futures and schedule the work. That's tokio.

```rust
#[tokio::main]
async fn main() {
    let body = reqwest::get("https://example.com").await.unwrap().text().await.unwrap();
    println!("{body}");
}
```

> **`#[tokio::main]` wraps `main` in a runtime.** Without a runtime, an `async fn` does nothing at all — futures in Rust are lazy and only run when polled.

## Core concepts

| Term | Meaning |
|---|---|
| **Runtime** | The scheduler that drives futures |
| **Task** | A unit of async work — like a very cheap thread |
| **`spawn`** | Run a task concurrently |
| **`.await`** | Yield until this future is ready |
| **`select!`** | Race several futures, take whichever finishes first |
| **`join!`** | Wait for all of them |

## Spawning and concurrency

```rust
let handle = tokio::spawn(async {
    expensive().await
});
let result = handle.await.unwrap();

// run concurrently, wait for all
let (a, b) = tokio::join!(fetch_a(), fetch_b());

// race - first to finish wins
tokio::select! {
    r = fetch() => println!("got {r:?}"),
    _ = tokio::time::sleep(Duration::from_secs(5)) => println!("timeout"),
}
```

```rust
// many tasks
let mut set = tokio::task::JoinSet::new();
for url in urls { set.spawn(fetch(url)); }
while let Some(res) = set.join_next().await { /* ... */ }
```

## Blocking work

```rust
// ✗ blocks the whole runtime thread
std::thread::sleep(Duration::from_secs(1));

// ✓
tokio::time::sleep(Duration::from_secs(1)).await;

// CPU-heavy or blocking I/O -> move it off the async threads
let result = tokio::task::spawn_blocking(|| heavy_computation()).await.unwrap();
```

> **Same rule as Python's async** ([[Async (Python)]]): a blocking call inside async code freezes everything. `spawn_blocking` is tokio's `run_in_threadpool`.

## Channels

```rust
use tokio::sync::{mpsc, oneshot, broadcast, Mutex, RwLock};

let (tx, mut rx) = mpsc::channel::<Job>(100);       // many senders, one receiver
tx.send(job).await?;
while let Some(job) = rx.recv().await { handle(job).await; }

let (tx, rx) = oneshot::channel();                  // exactly one message
let (tx, _rx) = broadcast::channel(16);             // many receivers, all get it
```

> **Prefer channels over shared mutable state.** "Share memory by communicating" — it sidesteps most locking bugs, and the borrow checker makes the alternative awkward on purpose ([[Rust for this stack]]).

```rust
use std::sync::{Arc, Mutex};

let shared = Arc::new(Mutex::new(0));               // when you must share
{
    let mut guard = shared.lock().await;
    *guard += 1;
}                                                    // lock released here
```

> **Use `tokio::sync::Mutex` in async code**, not `std::sync::Mutex` — the std one blocks the runtime thread while waiting.

## Time

```rust
tokio::time::sleep(Duration::from_millis(500)).await;
tokio::time::timeout(Duration::from_secs(5), fetch()).await??;

let mut ticker = tokio::time::interval(Duration::from_secs(60));
loop {
    ticker.tick().await;
    do_periodic_work().await;
}
```

## I/O

```rust
use tokio::fs;
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

let content = fs::read_to_string("f.txt").await?;

let listener = TcpListener::bind("0.0.0.0:8080").await?;
loop {
    let (mut socket, _) = listener.accept().await?;
    tokio::spawn(async move {
        let mut buf = [0u8; 1024];
        let n = socket.read(&mut buf).await.unwrap();
        socket.write_all(&buf[..n]).await.unwrap();
    });
}
```

## Graceful shutdown

```rust
tokio::select! {
    _ = server => {},
    _ = tokio::signal::ctrl_c() => println!("shutting down"),
}
```

## What builds on tokio

| Crate | For | Note |
|---|---|---|
| **actix-web** / axum | Web servers | [[Actix Web]] |
| **reqwest** | HTTP client | [[reqwest]] |
| **sqlx** | Async SQL | [[sqlx]] |
| tracing | Structured logging | [[Rust crates]] |

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Nothing happens | No runtime, or future never awaited | `#[tokio::main]`; `.await` it |
| Everything freezes | Blocking call in async | `spawn_blocking` / async equivalent |
| "cannot be sent between threads safely" | Non-`Send` type held across `.await` | Drop it before awaiting; use `Arc` |
| Deadlock with a Mutex | Used `std::sync::Mutex` | Use `tokio::sync::Mutex` |
| Task silently disappears | `JoinHandle` dropped or panicked | `.await` the handle and check the result |

## Related

[[Rust crates]] · [[Rust]] · [[Rust for this stack]] · [[Actix Web]] · [[reqwest]] · [[Async (Python)]]
