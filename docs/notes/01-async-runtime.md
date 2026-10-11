# Stage 1: Async Rust and Tokio

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> If you come from JavaScript or TypeScript, the comparisons below map Rust concepts to familiar JS ideas.

## Contents

1. [Why async?](#1-why-async)
2. [Futures and `async`/`await`](#2-futures-and-asyncawait)
3. [The Tokio runtime](#3-the-tokio-runtime)
4. [Tasks: `tokio::spawn`, `join!`, and `JoinSet`](#4-tasks-tokiospawn-join-and-joinset)
5. [Channels: `mpsc`, `oneshot`, `broadcast`, `watch`](#5-channels-mpsc-oneshot-broadcast-watch)
6. [`select!`, timeouts, and cancellation](#6-select-timeouts-and-cancellation)
7. [Shared state: `Mutex` vs `RwLock`, std vs Tokio](#7-shared-state-mutex-vs-rwlock-std-vs-tokio)
8. [Never block the runtime](#8-never-block-the-runtime)
9. [How this looks in Axum](#9-how-this-looks-in-axum)
10. [Putting it all together](#10-putting-it-all-together)
11. [Exercises](#11-exercises)
12. [Further reading](#12-further-reading)

---

## 1. Why async?

A backend spends most of its time **waiting**: for the database, for another HTTP service, for the client to send more bytes. While it waits, it should keep serving other requests.

| Approach | How it waits | Cost |
|---|---|---|
| One OS thread per request | The thread sleeps until the I/O is done | Each thread has its own stack (often MBs) and switching between threads is expensive. Tens of thousands of threads won't scale. |
| Async tasks | The task yields, and the thread runs another task | A task is a small state machine (often bytes or KBs). Millions are possible. |

Async Rust lets you write code that *looks* sequential but yields at every `.await`, so a few threads can serve many requests.

> **JS comparison:** Node.js does the same with its event loop and Promises. The big difference is that Tokio runs tasks on **several threads at once** (one per CPU core by default), while the Node event loop is single-threaded. That's why Rust requires `Send` for most async code: a task can move between threads.

---

## 2. Futures and `async`/`await`

### 2.1 `async fn` returns a `Future`

```rust
async fn get_number() -> u32 {
    42
}
// The real signature is roughly: fn get_number() -> impl Future<Output = u32>
```

A `Future` is a value that **will produce a result later**. You get the result with `.await`:

```rust
let n = get_number().await;   // n: u32
```

`.await` is only allowed inside an `async fn` or an `async { }` block.

### 2.2 Futures are lazy (unlike Promises)

This is the most important difference from JavaScript:

```rust
async fn say(msg: &str) {
    println!("{msg}");
}

#[tokio::main]
async fn main() {
    let fut = say("hello");  // nothing runs yet!
    println!("before");
    fut.await;               // now it runs
}
```

Output:

```text
before
hello
```

> **JS comparison:** a JS `Promise` starts running the moment you create it. A Rust `Future` does **nothing** until it is `.await`ed (or spawned). If you forget the `.await`, the code silently never runs; the compiler warns you with `unused implementors of Future must be used`.

### 2.3 `async` blocks

Like an async arrow function you call immediately:

```rust
let fut = async {
    let a = get_number().await;
    a * 2
};
let doubled = fut.await;  // 84
```

Use `async move { }` when the block must **own** the variables it uses. This is needed whenever the block is spawned as a task (section 4).

### 2.4 `.await` is a yield point

Every `.await` is a place where the task may pause and let others run. Everything between two `.await`s runs without interruption on one thread. Remember this; section 7 depends on it.

---

## 3. The Tokio runtime

Rust's standard library defines `Future` but **does not include a runtime** that runs futures. Tokio is the runtime used by Axum, SQLx, Hyper, and most of the server ecosystem.

`Cargo.toml` (the Playground already has Tokio):

```toml
tokio = { version = "1", features = ["full"] }
```

### 3.1 `#[tokio::main]`

```rust
#[tokio::main]
async fn main() {
    println!("running inside Tokio");
}
```

The macro turns `main` into roughly this:

```rust
fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .expect("failed to build runtime")   // OK at startup: nothing can run without it
        .block_on(async {
            println!("running inside Tokio");
        });
}
```

| Flavor | Threads | Use |
|---|---|---|
| `#[tokio::main]` (multi-thread) | One worker per CPU core | Servers (default) |
| `#[tokio::main(flavor = "current_thread")]` | One | Small tools, some tests |
| `#[tokio::test]` | One (by default) | Async unit tests |

### 3.2 Async versions of standard things

Inside async code, use Tokio's versions instead of the blocking std ones:

| Blocking (std) | Async (Tokio) |
|---|---|
| `std::thread::sleep` | `tokio::time::sleep(...).await` |
| `std::fs::read_to_string` | `tokio::fs::read_to_string(...).await` |
| `std::net::TcpListener` | `tokio::net::TcpListener` |
| `std::sync::mpsc` | `tokio::sync::mpsc` |

```rust
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    println!("waiting...");
    sleep(Duration::from_millis(500)).await;
    println!("done");
}
```

---

## 4. Tasks: `tokio::spawn`, `join!`, and `JoinSet`

### 4.1 `tokio::spawn`: run in the background

A **task** is a lightweight green thread managed by Tokio. `tokio::spawn` starts one immediately and returns a `JoinHandle`:

```rust
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        sleep(Duration::from_millis(100)).await;
        42
    });

    println!("task started, doing other work...");

    match handle.await {
        Ok(n) => println!("task returned {n}"),
        Err(e) => println!("task failed: {e}"),   // a JoinError: the task panicked or was aborted
    }
}
```

- `handle.await` returns `Result<T, JoinError>`.
- `handle.abort()` cancels the task.
- If you drop the handle, the task **keeps running** (it is detached).

> **JS comparison:** `tokio::spawn(fut)` is like starting a Promise without awaiting it, and `handle.await` is like awaiting it later.

### 4.2 The `'static + Send` requirement

A spawned task may outlive the function that spawned it and may run on any thread, so it must:

- **own** its data (`'static`): use `async move` and clone what you need
- be **`Send`**: everything held across an `.await` must be safe to move between threads

```rust
let name = String::from("worker-1");

// tokio::spawn(async { println!("{name}") });  // ERROR: may outlive `name`
tokio::spawn(async move { println!("{name}") }); // OK: the task owns `name`
```

Sharing with several tasks means one `Arc` clone per task:

```rust
use std::sync::Arc;

let config = Arc::new(String::from("db=postgres://..."));
for i in 0..3 {
    let config = Arc::clone(&config);
    tokio::spawn(async move {
        println!("task {i} sees {config}");
    });
}
```

### 4.3 `join!`: run several futures concurrently and wait for all

```rust
use std::time::{Duration, Instant};
use tokio::time::sleep;

async fn fetch_user() -> &'static str {
    sleep(Duration::from_millis(200)).await;
    "Anna"
}

async fn fetch_orders() -> u32 {
    sleep(Duration::from_millis(300)).await;
    7
}

#[tokio::main]
async fn main() {
    let start = Instant::now();
    let (user, orders) = tokio::join!(fetch_user(), fetch_orders());
    println!("{user} has {orders} orders, took {:?}", start.elapsed()); // ~300 ms, not 500 ms
}
```

`tokio::try_join!` does the same for futures that return `Result` and stops at the **first error**.

> **JS comparison:** `join!` is `Promise.all`, and `try_join!` is `Promise.all` that rejects on the first failure.

**`join!` vs `spawn`:** `join!` runs the futures concurrently **inside the current task** (on one thread). `spawn` creates independent tasks that can run **in parallel** on different threads. Use `join!` for a few I/O calls within one request, and `spawn` for background work.

### 4.4 `JoinSet`: a dynamic group of tasks

When the number of tasks is only known at runtime:

```rust
use tokio::task::JoinSet;

#[tokio::main]
async fn main() {
    let mut set = JoinSet::new();
    for id in 1..=5 {
        set.spawn(async move { id * 10 });
    }

    let mut total = 0;
    while let Some(res) = set.join_next().await {   // results arrive in completion order
        match res {
            Ok(v) => total += v,
            Err(e) => println!("task failed: {e}"),
        }
    }
    println!("total = {total}"); // 150
}
```

Dropping a `JoinSet` aborts all of its tasks, which makes it a good tool for structured cleanup.

---

## 5. Channels: `mpsc`, `oneshot`, `broadcast`, `watch`

Instead of sharing memory behind a lock, tasks can **send messages** to each other.

| Channel | Shape | Typical use |
|---|---|---|
| `mpsc` | many senders → one receiver | Job queues, event pipelines |
| `oneshot` | one value, once | Reply to a single request |
| `broadcast` | many senders → every receiver gets each message | Fan-out events, chat |
| `watch` | many receivers see only the **latest** value | Config reload, shutdown signals |

### 5.1 `mpsc`: bounded queues

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<String>(32);  // capacity 32: a BOUNDED buffer

    for i in 0..3 {
        let tx = tx.clone();
        tokio::spawn(async move {
            if tx.send(format!("hello from {i}")).await.is_err() {
                println!("receiver dropped");
            }
        });
    }
    drop(tx); // drop the original sender, otherwise rx.recv() never returns None

    while let Some(msg) = rx.recv().await {
        println!("{msg}");
    }
    println!("all senders gone, channel closed");
}
```

Key points:

- `send(...).await` **waits when the buffer is full**. This is **backpressure**: a fast producer is slowed down to the consumer's speed instead of eating all memory.
- `recv()` returns `None` once **every** sender is dropped. Forgetting `drop(tx)` is the most common reason a loop hangs forever.
- `mpsc::unbounded_channel()` exists, but production code should prefer bounded channels (the "bounded buffers only" convention).

### 5.2 `oneshot`: request/response

Often paired with `mpsc` to get an answer back:

```rust
use tokio::sync::{mpsc, oneshot};

enum Command {
    Get { reply: oneshot::Sender<u32> },
    Increment,
}

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<Command>(16);

    // The "actor": the only task that owns `count`, so no lock is needed
    tokio::spawn(async move {
        let mut count = 0;
        while let Some(cmd) = rx.recv().await {
            match cmd {
                Command::Increment => count += 1,
                Command::Get { reply } => {
                    let _ = reply.send(count); // ignore the error if the asker gave up
                }
            }
        }
    });

    for _ in 0..3 {
        let _ = tx.send(Command::Increment).await;
    }

    let (reply_tx, reply_rx) = oneshot::channel();
    let _ = tx.send(Command::Get { reply: reply_tx }).await;
    match reply_rx.await {
        Ok(n) => println!("count = {n}"),   // 3
        Err(_) => println!("actor stopped"),
    }
}
```

This is the **actor pattern**: one task owns the state and everyone else talks to it through messages.

### 5.3 `broadcast` and `watch` (short version)

```rust
use tokio::sync::{broadcast, watch};

// broadcast: every subscriber gets every message (slow receivers can miss messages: RecvError::Lagged)
let (tx, mut rx1) = broadcast::channel::<String>(16);
let mut rx2 = tx.subscribe();

// watch: receivers see the latest value; perfect for "shutting down = true"
let (shutdown_tx, mut shutdown_rx) = watch::channel(false);
// later: shutdown_tx.send(true)
// a task:  shutdown_rx.changed().await
```

> **JS comparison:** `mpsc` is like an async queue, `oneshot` is a single Promise resolved by someone else, `broadcast` is an `EventEmitter`, and `watch` is like an observable that always holds its current value.

---

## 6. `select!`, timeouts, and cancellation

### 6.1 `select!`: wait for the first of several futures

```rust
use std::time::Duration;
use tokio::sync::mpsc;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    let (_tx, mut rx) = mpsc::channel::<String>(8);  // nobody sends anything

    tokio::select! {
        msg = rx.recv() => println!("got {msg:?}"),
        _ = sleep(Duration::from_millis(200)) => println!("timed out waiting for a message"),
    }
}
```

- Whichever branch finishes first runs its handler.
- The other branches are **dropped**, which cancels them.
- Add `biased;` as the first line to check branches **in order** instead of randomly (useful when one branch should take priority).

> **JS comparison:** `select!` is `Promise.race`, but the losers are actually **cancelled**. In JS, the losing promises keep running in the background.

### 6.2 `timeout`

The common case is wrapping a single future:

```rust
use std::time::Duration;
use tokio::time::{sleep, timeout};

async fn slow_query() -> u32 {
    sleep(Duration::from_secs(5)).await;
    1
}

#[tokio::main]
async fn main() {
    match timeout(Duration::from_millis(100), slow_query()).await {
        Ok(v) => println!("result {v}"),
        Err(_) => println!("query took too long"),  // Err(Elapsed)
    }
}
```

### 6.3 Cancellation = dropping the future

In Rust, a future is cancelled by **dropping** it. It stops at its current `.await` and never continues. This happens with `select!` losers, `timeout`, `handle.abort()`, and when a client disconnects in Axum (the handler's future is dropped).

Consequence: **any `.await` can be the last line that runs.** Don't leave half-finished state between two `.await`s. For example, don't write to the database in two separate steps without a transaction.

> **JS comparison:** this replaces `AbortController`. Cancellation is automatic, so you don't have to pass a signal around.

---

## 7. Shared state: `Mutex` vs `RwLock`, std vs Tokio

### 7.1 Which lock?

| Lock | Allows | Use when |
|---|---|---|
| `std::sync::Mutex` | One accessor at a time | Short critical sections, **no `.await` inside** (most common) |
| `std::sync::RwLock` | Many readers **or** one writer | Read-heavy data such as caches and presence maps |
| `tokio::sync::Mutex` | One at a time, and the lock can be held across `.await` | You truly must hold it while awaiting (rare) |
| `tokio::sync::RwLock` | Many readers or one writer, `.await`-safe | Same as above, read-heavy |

The Tokio docs recommend the **std** mutex by default because it is faster. Choose the Tokio one only when you need to hold the lock across an `.await`.

### 7.2 The golden rule: never hold a `std::sync` lock across `.await`

```rust
use std::sync::{Arc, Mutex};

async fn save() {}

async fn bad(counter: Arc<Mutex<u32>>) {
    let mut guard = counter.lock().unwrap();
    *guard += 1;
    save().await;          // the guard is still alive here!
}
```

Two problems:

1. `MutexGuard` is **not `Send`**, so `tokio::spawn(bad(counter))` **does not compile**: "future cannot be sent between threads safely".
2. Even where it compiles, other tasks block a whole worker thread while waiting for the lock, which can deadlock the runtime.

**Fix:** lock, update, drop the guard, then await.

```rust
async fn good(counter: Arc<Mutex<u32>>) {
    {
        let mut guard = counter.lock().unwrap();  // fine in examples, see the note below
        *guard += 1;
    } // guard dropped here
    save().await;
}
```

Or copy the value out in one expression:

```rust
let current = *counter.lock().unwrap();
save_value(current).await;
```

> **Note on `.unwrap()` here:** `lock()` fails only if another thread panicked while holding the lock ("poisoned"). Runtime code should map the error, for example `counter.lock().map_err(|_| AppError::Internal)?`, following the convention of no `unwrap()` in runtime code.

### 7.3 Tokio locks: when you really must await while locked

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

async fn write_log(line: &str) { let _ = line; }

async fn append(log: Arc<Mutex<Vec<String>>>, line: String) {
    let mut guard = log.lock().await;  // async lock, no poisoning, no unwrap
    write_log(&line).await;            // allowed: the Tokio guard is Send
    guard.push(line);
}
```

### 7.4 Locks vs channels

| Prefer | When |
|---|---|
| `Arc<Mutex<T>>` / `Arc<RwLock<T>>` | Small shared data, short updates (counters, caches, presence maps) |
| Channels and an actor | Complex state, ordered operations, or work that involves `.await` |

---

## 8. Never block the runtime

Tokio has only a few worker threads. If one of them runs **blocking** code, every task scheduled on that thread freezes.

Blocking means:
- `std::thread::sleep`
- synchronous file or network I/O (`std::fs`, a blocking HTTP client)
- CPU-heavy work that takes more than about 10–100 µs without an `.await`: password hashing (argon2, bcrypt), compression, large JSON or image processing

```rust
// BAD: freezes a worker thread for one second
async fn bad() {
    std::thread::sleep(std::time::Duration::from_secs(1));
}

// GOOD: yields, so the thread serves other tasks meanwhile
async fn good() {
    tokio::time::sleep(std::time::Duration::from_secs(1)).await;
}
```

### 8.1 `spawn_blocking` for CPU-heavy or blocking work

```rust
use tokio::task;

fn expensive_hash(input: String) -> u64 {
    // pretend this is argon2
    input.bytes().fold(0u64, |acc, b| acc.wrapping_mul(31).wrapping_add(b as u64))
}

#[tokio::main]
async fn main() {
    let password = String::from("secret");
    let result = task::spawn_blocking(move || expensive_hash(password)).await;
    match result {
        Ok(hash) => println!("hash = {hash}"),
        Err(e) => println!("blocking task failed: {e}"),
    }
}
```

`spawn_blocking` runs the closure on a **separate thread pool** made for blocking work, and you `.await` the result.

> **JS comparison:** it's like moving heavy work into a Worker thread so the event loop stays responsive. In Node, blocking freezes **everything**. In Tokio, it freezes one worker thread and every task waiting on it.

---

## 9. How this looks in Axum

Every Axum handler is an `async fn`. Each request runs as its own Tokio task, so everything in this module applies directly.

```rust
use std::time::Duration;
use axum::{extract::State, routing::get, Json, Router};
use tokio::net::TcpListener;

#[derive(Clone)]
struct AppState {
    jobs: tokio::sync::mpsc::Sender<String>,   // queue to a background worker
}

async fn dashboard() -> Json<(String, u32)> {
    // Two independent I/O calls in parallel: join!
    let (user, orders) = tokio::join!(load_user(), count_orders());
    Json((user, orders))
}

async fn enqueue(State(state): State<AppState>) -> &'static str {
    // A bounded queue: if the worker is overloaded, this await applies backpressure
    match state.jobs.send("rebuild-report".into()).await {
        Ok(()) => "queued",
        Err(_) => "worker unavailable",
    }
}

async fn load_user() -> String { "Anna".into() }
async fn count_orders() -> u32 { 7 }

async fn shutdown_signal() {
    // Resolves on Ctrl+C; Axum then stops accepting and lets in-flight requests finish
    let _ = tokio::signal::ctrl_c().await;
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let (tx, mut rx) = tokio::sync::mpsc::channel::<String>(100);

    // A background worker task living for the whole program
    tokio::spawn(async move {
        while let Some(job) = rx.recv().await {
            tokio::time::sleep(Duration::from_millis(50)).await;
            println!("processed {job}");
        }
    });

    let app = Router::new()
        .route("/dashboard", get(dashboard))
        .route("/jobs", axum::routing::post(enqueue))
        .with_state(AppState { jobs: tx });

    let listener = TcpListener::bind("0.0.0.0:3000").await?;
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await?;
    Ok(())
}
```

What to notice:

| Concept | Where |
|---|---|
| `#[tokio::main]` | Starts the runtime that `axum::serve` runs on |
| `tokio::join!` | Concurrent I/O inside one handler |
| `tokio::spawn` + `mpsc` | A background worker fed by handlers |
| Bounded channel | Backpressure when the worker falls behind |
| Cancellation | If the client disconnects, the handler future is dropped at its next `.await` |
| `Send` | Handlers must return `Send` futures, so the "no std lock across `.await`" rule applies to every handler |

---

## 10. Putting it all together

A small job-processing service: a producer submits jobs into a **bounded queue**, a **worker task** processes them with a **per-job timeout**, updates **shared stats** under a short-lived lock, and stops on a **shutdown signal** after draining the queue. It runs as-is in the Rust Playground.

```rust
use std::sync::{Arc, Mutex};
use std::time::Duration;
use tokio::sync::{mpsc, watch};
use tokio::time::{sleep, timeout};

#[derive(Debug)]
struct Job {
    id: u32,
    cost_ms: u64,
}

#[derive(Debug, Default)]
struct Stats {
    completed: u32,
    timed_out: u32,
}

#[derive(Clone)]
struct AppState {
    jobs: mpsc::Sender<Job>,
    stats: Arc<Mutex<Stats>>,
}

const JOB_TIMEOUT: Duration = Duration::from_millis(100);

async fn process(job: &Job) {
    // Stands in for real I/O: a DB query, an HTTP call...
    sleep(Duration::from_millis(job.cost_ms)).await;
}

fn record(stats: &Mutex<Stats>, ok: bool) {
    // Lock, update, drop. Never held across an .await.
    if let Ok(mut s) = stats.lock() {
        if ok {
            s.completed += 1;
        } else {
            s.timed_out += 1;
        }
    }
}

async fn worker(
    mut rx: mpsc::Receiver<Job>,
    stats: Arc<Mutex<Stats>>,
    mut shutdown: watch::Receiver<bool>,
) {
    loop {
        tokio::select! {
            biased; // check the queue first, so queued jobs are drained before shutting down

            maybe_job = rx.recv() => {
                let Some(job) = maybe_job else {
                    println!("worker: queue closed");
                    break;
                };
                match timeout(JOB_TIMEOUT, process(&job)).await {
                    Ok(()) => {
                        println!("job {} done ({} ms)", job.id, job.cost_ms);
                        record(&stats, true);
                    }
                    Err(_) => {
                        println!("job {} timed out after {} ms", job.id, JOB_TIMEOUT.as_millis());
                        record(&stats, false);
                    }
                }
            }

            _ = shutdown.changed() => {
                println!("worker: shutdown signal, stopping");
                break;
            }
        }
    }
}

async fn submit(state: &AppState, job: Job) -> Result<(), String> {
    state
        .jobs
        .send(job)
        .await
        .map_err(|e| format!("queue closed, job {} rejected", e.0.id))
}

#[tokio::main]
async fn main() {
    let (tx, rx) = mpsc::channel(8);                    // bounded queue
    let (shutdown_tx, shutdown_rx) = watch::channel(false);

    let state = AppState {
        jobs: tx,
        stats: Arc::new(Mutex::new(Stats::default())),
    };

    let worker_handle = tokio::spawn(worker(rx, Arc::clone(&state.stats), shutdown_rx));

    // Job 3 is too slow and will hit the timeout
    for id in 1..=5 {
        let cost_ms = if id == 3 { 500 } else { 15 * u64::from(id) };
        if let Err(e) = submit(&state, Job { id, cost_ms }).await {
            println!("submit error: {e}");
        }
    }

    if shutdown_tx.send(true).is_err() {
        println!("worker already stopped");
    }

    if let Err(e) = worker_handle.await {
        println!("worker task failed: {e}");
    }

    match state.stats.lock() {
        Ok(s) => println!("final stats: {:?}", *s),
        Err(_) => println!("stats lock poisoned"),
    }
}
```

Expected output:

```text
job 1 done (15 ms)
job 2 done (30 ms)
job 3 timed out after 100 ms
job 4 done (60 ms)
job 5 done (75 ms)
worker: shutdown signal, stopping
final stats: Stats { completed: 4, timed_out: 1 }
```

**How this maps to Axum:** `AppState` is the router state, `submit` plays the role of a `POST /jobs` handler, `worker` is a background task started in `main`, the `watch` channel is the same idea as `with_graceful_shutdown`, and the `std::sync::Mutex` follows the "lock briefly, never across `.await`" rule that every handler must follow.

---

## 11. Exercises

All of these run in the [Rust Playground](https://play.rust-lang.org/) (Tokio is available there).

- [ ] **1. Lazy futures:** create two futures with `async` blocks that print `"A"` and `"B"`, print `"start"`, then `.await` them in the order B, A. Predict the output before running it.
- [ ] **2. `join!` vs sequential:** write `async fn work(ms: u64) -> u64` that sleeps `ms` and returns it. Time three calls (100, 200, 300 ms) run sequentially, then with `tokio::join!`, using `std::time::Instant`. Explain the difference (~600 ms vs ~300 ms).
- [ ] **3. `JoinSet`:** spawn 10 tasks where task `i` sleeps `(10 - i) * 10` ms and returns `i`. Collect the results with `join_next()` and print them in **completion order**. Then sum them (`45`).
- [ ] **4. Bounded `mpsc`:** 3 producer tasks each send 5 numbers into a channel with capacity **2**. One consumer receives until the channel closes and prints the count (`15`). Make sure the program ends: drop the original sender.
- [ ] **5. Actor with `oneshot`:** build an actor that owns a `HashMap<String, u32>` and handles `Set { key, value }` and `Get { key, reply: oneshot::Sender<Option<u32>> }`. Set two keys, then get one existing and one missing key.
- [ ] **6. `select!` with a deadline:** wait for a message from a channel that receives a value after 300 ms, with `select!` against a 200 ms `sleep`. Print `"timeout"`. Then change the sleep to 400 ms and print the message instead.
- [ ] **7. Fix the lock:** this does not compile. Fix it **without** switching to `tokio::sync::Mutex`, then fix it a second way **with** `tokio::sync::Mutex`.
  ```rust
  use std::sync::{Arc, Mutex};
  async fn notify() {}
  async fn bump(c: Arc<Mutex<u32>>) {
      let mut g = c.lock().unwrap();
      *g += 1;
      notify().await;
  }
  #[tokio::main]
  async fn main() {
      let c = Arc::new(Mutex::new(0));
      tokio::spawn(bump(c.clone())).await.unwrap();
      println!("{}", *c.lock().unwrap());
  }
  ```
- [ ] **8. Don't block:** spawn a "ticker" task that prints `tick` every 50 ms, 10 times. In `main`, compute the sum of `0..200_000_000u64` first **directly** in async code, then with `spawn_blocking`, using the multi-thread runtime (`#[tokio::main(flavor = "multi_thread", worker_threads = 1)]`). Observe that the ticker stalls in the first version and keeps ticking in the second.

> When you can solve exercises 1–8 without looking anything up, you're ready for **Stage 2: Routing with Axum**.

---

## 12. Further reading

- [Tokio tutorial](https://tokio.rs/tokio/tutorial): especially *Spawning*, *Shared state*, *Channels*, and *Select*
- [*Asynchronous Programming in Rust*](https://rust-lang.github.io/async-book/): how futures work under the hood
- [`tokio::sync` docs](https://docs.rs/tokio/latest/tokio/sync/index.html): every channel and lock, with guidance on which one to choose
- [Tokio: "Async in depth"](https://tokio.rs/tokio/tutorial/async): wakers, polling, and why futures are lazy
- [Alice Ryhl: "Async: What is blocking?"](https://ryhl.io/blog/async-what-is-blocking/): the best explanation of "never block the runtime"
- [Alice Ryhl: "Actors with Tokio"](https://ryhl.io/blog/actors-with-tokio/): the `mpsc` + `oneshot` actor pattern in depth
