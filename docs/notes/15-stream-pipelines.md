# Stage 15: Stream Pipelines

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> If you come from JavaScript or TypeScript, the comparisons below map Rust concepts to familiar JS ideas (`Promise.all`, `p-limit`, `AbortController`, and Node stream `pipeline`).

## Contents

1. [From one stream to a pipeline](#1-from-one-stream-to-a-pipeline)
2. [What a `Stream` really is: `poll_next` and `Pin`](#2-what-a-stream-really-is-poll_next-and-pin)
3. [Returning streams: `impl Stream`, `BoxStream`, and lifetimes](#3-returning-streams-impl-stream-boxstream-and-lifetimes)
4. [Writing stages with `async-stream`](#4-writing-stages-with-async-stream)
5. [Bounded parallelism: `buffered` and `buffer_unordered`](#5-bounded-parallelism-buffered-and-buffer_unordered)
6. [CPU-heavy work: `spawn_blocking` inside a stream](#6-cpu-heavy-work-spawn_blocking-inside-a-stream)
7. [Task groups: `JoinSet`](#7-task-groups-joinset)
8. [Limiting shared resources: `Semaphore`](#8-limiting-shared-resources-semaphore)
9. [Cancellation: `CancellationToken`](#9-cancellation-cancellationtoken)
10. [Graceful shutdown](#10-graceful-shutdown)
11. [Branching and side outputs](#11-branching-and-side-outputs)
12. [Errors in a pipeline](#12-errors-in-a-pipeline)
13. [Testing pipelines: early output, bounded residency, cancellation](#13-testing-pipelines-early-output-bounded-residency-cancellation)
14. [How this looks in a production service](#14-how-this-looks-in-a-production-service)
15. [Putting it all together](#15-putting-it-all-together)
16. [Exercises](#16-exercises)
17. [Further reading](#17-further-reading)

---

## 1. From one stream to a pipeline

Stage 10 streamed one source to one client. Stage 14 streamed query results batch by batch. Real data work chains several steps:

```text
 source ──▶ parse ──▶ validate ──┬──▶ main sink
                                 └──▶ rejects sink (side output)
```

Each arrow is a place where data can pile up. A pipeline stays bounded only if **every** stage:

| Rule | Why |
|---|---|
| Pulls input only when it can push output | Otherwise items queue inside the stage |
| Has a fixed limit on work in progress | Parallelism without a limit is an unbounded buffer |
| Stops when its consumer is gone | Otherwise it keeps working for nobody |
| Reports errors without dropping the bound | A failed item mustn't stall or flood other branches |

This stage covers the tools for each rule: stream combinators, `JoinSet`, `Semaphore`, `CancellationToken`, and tests that prove the rules hold.

> **JS comparison:** Node's `stream.pipeline(source, transform, sink)` wires backpressure and error handling across stages. In Rust you build the same thing from streams and bounded channels, and the compiler checks the types between stages.

---

## 2. What a `Stream` really is: `poll_next` and `Pin`

Stage 10 used streams through `StreamExt`. Underneath, the trait is small:

```rust
pub trait Stream {
    type Item;
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>>;
}
```

| `poll_next` returns | Meaning |
|---|---|
| `Poll::Ready(Some(item))` | Here is the next item |
| `Poll::Ready(None)` | The stream is finished |
| `Poll::Pending` | Nothing yet; I've stored `cx.waker()` and will wake you |

Two consequences:

- **Pull-based:** a stream does nothing unless polled. That's why backpressure works without extra effort: a slow consumer simply polls less often.
- **`Pin`:** async blocks and many streams hold references into themselves across `.await` points. `Pin` guarantees they won't be moved in memory after polling starts. In practice you meet it as `Box::pin(stream)` or `tokio::pin!(stream)` when a method needs `Unpin`:

```rust
let stream = make_stream();
tokio::pin!(stream);                 // pins on the stack
while let Some(x) = stream.next().await { /* ... */ }
```

You rarely implement `poll_next` by hand. Section 4 shows the easier way.

---

## 3. Returning streams: `impl Stream`, `BoxStream`, and lifetimes

Stages are functions that take a stream and return a stream:

```rust
use futures::{Stream, StreamExt};

fn only_valid(input: impl Stream<Item = Record>) -> impl Stream<Item = Record> {
    input.filter(|r| futures::future::ready(r.is_valid()))
}
```

When the concrete type differs between branches, or you store stages in a struct, **box** it:

```rust
use futures::stream::BoxStream;

fn source(kind: SourceKind) -> BoxStream<'static, Result<Line, PipelineError>> {
    match kind {
        SourceKind::File(path) => file_lines(path).boxed(),
        SourceKind::Memory(v) => futures::stream::iter(v.into_iter().map(Ok)).boxed(),
    }
}
```

`BoxStream<'a, T>` is `Pin<Box<dyn Stream<Item = T> + Send + 'a>>`.

**The `'static` question.** A stream that is spawned into a task, or used as a response body, must be `Send + 'static`: it can't borrow anything from the function that created it. The fix is always the same:

| Borrowed | Make it owned |
|---|---|
| `&PgPool` | `pool.clone()` (an `Arc` inside) |
| `&Config` | `Arc<Config>` |
| `&str` | `String` or `Arc<str>` |
| `&[T]` for large data | Don't. Stream it instead |

The compiler error usually says "borrowed value does not live long enough" or "`T` cannot be sent between threads safely". Read it as: this stage still points at something it doesn't own.

---

## 4. Writing stages with `async-stream`

Combinators get awkward when a stage has state, early exits, or several `await`s. The `async-stream` crate lets you write a stream as a normal async loop:

```toml
async-stream = "0.3"
```

```rust
use async_stream::try_stream;
use futures::{Stream, StreamExt};

/// Groups lines into chunks of at most `max` without waiting for the input to end.
fn chunk_lines(
    input: impl Stream<Item = Result<String, PipelineError>>,
    max: usize,
) -> impl Stream<Item = Result<Vec<String>, PipelineError>> {
    try_stream! {
        let mut buf = Vec::with_capacity(max);
        tokio::pin!(input);
        while let Some(line) = input.next().await {
            buf.push(line?);                   // `?` ends the stream with that error
            if buf.len() == max {
                yield std::mem::replace(&mut buf, Vec::with_capacity(max));
            }
        }
        if !buf.is_empty() {
            yield buf;                          // flush the last partial chunk
        }
    }
}
```

- `yield` suspends until the consumer asks for the next item. The stage can never run ahead of its consumer.
- `try_stream!` allows `?`. Use `stream!` for infallible stages.
- State (`buf`) is ordinary local variables, and its size is fixed by `max`.

---

## 5. Bounded parallelism: `buffered` and `buffer_unordered`

A stream of **futures** can run several of them at once with a fixed limit:

```rust
use futures::StreamExt;

let enriched = records
    .map(|r| async move { lookup_customer(r).await })   // stream of futures
    .buffered(8);                                        // at most 8 running at once
```

| Combinator | Concurrency | Output order |
|---|---|---|
| `.then(f)` | 1 | Input order |
| `.buffered(n)` | Up to `n` | Input order |
| `.buffer_unordered(n)` | Up to `n` | Completion order |

Points to know:

- The limit is also a **memory bound**: at most `n` items are in progress inside the stage.
- `buffered` keeps order, so one slow item holds back the ones behind it (head-of-line blocking). Use `buffer_unordered` when order doesn't matter.
- These futures run **inside the polling task**, concurrently, not on other threads. That's fine for I/O. For CPU work, see section 6.
- Never write `records.map(|r| tokio::spawn(work(r)))` without a limit. Each spawn starts immediately and nothing slows the source down.

> **JS comparison:** `buffered(8)` is `p-limit(8)` plus `Promise.all` that keeps order, but lazy: new work starts only when a slot frees up **and** the consumer wants more.

---

## 6. CPU-heavy work: `spawn_blocking` inside a stream

Parsing, compression, hashing, or heavy validation blocks the thread. On a Tokio worker that stalls every other task on that thread (Stage 1). Move it to the blocking pool, and keep the limit:

```rust
let parsed = lines
    .map(|line| tokio::task::spawn_blocking(move || parse(line)))  // JoinHandle is a future
    .buffered(num_cpus);                                            // bounded and ordered
```

Each item is `Result<T, JoinError>`. A `JoinError` means the closure panicked or was cancelled. Treat it as a pipeline failure, not as bad data.

| Work type | Tool |
|---|---|
| Waiting on I/O (HTTP, DB) | Plain `async`, `buffered(n)` |
| Short CPU work per item | `spawn_blocking` + `buffered(n)` |
| Heavy CPU over a whole batch | `spawn_blocking` per batch, or `rayon` inside one `spawn_blocking` |
| Data already in Arrow | Let DataFusion's partitions do it (Stage 14) |

Make the unit of work big enough to pay for the thread hop. Spawning per *line* is fine for an example. Production code usually spawns per *chunk* (section 4's `chunk_lines`).

---

## 7. Task groups: `JoinSet`

A pipeline is often several long-running tasks (one per stage). `JoinSet` owns a group of tasks and gives their results back as they finish:

```rust
use tokio::task::JoinSet;

let mut set = JoinSet::new();
set.spawn(run_source(tx));
set.spawn(run_transform(rx, out_tx));
set.spawn(run_sink(out_rx));

while let Some(result) = set.join_next().await {
    match result {
        Ok(Ok(())) => {}
        Ok(Err(e)) => { set.abort_all(); return Err(e); }    // a stage failed
        Err(join_err) => { set.abort_all(); return Err(join_err.into()); } // a stage panicked
    }
}
```

| | `tokio::spawn` alone | `JoinSet` |
|---|---|---|
| Who owns the task | Nobody (detached unless you keep the handle) | The set |
| Dropping the owner | Task keeps running | **All tasks are aborted** |
| Waiting for "whichever finishes first" | Manual `select!` | `join_next()` |
| Stopping everything | Abort each handle | `abort_all()` |

Dropping a `JoinSet` aborts its tasks, so a pipeline owned by a request or a run can't leak background work.

---

## 8. Limiting shared resources: `Semaphore`

`buffered(n)` limits one stage. A `Semaphore` limits something **shared across** stages, requests, or runs: database connections, calls to an external API, or the number of pipelines running at once.

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;

#[derive(Clone)]
struct AppState {
    pipeline_slots: Arc<Semaphore>,   // e.g. Semaphore::new(2)
}

async fn start_run(state: AppState) -> Result<(), AppError> {
    let permit = state
        .pipeline_slots
        .clone()
        .try_acquire_owned()
        .map_err(|_| AppError::Conflict("too many runs in progress".into()))?;

    tokio::spawn(async move {
        let _permit = permit;         // released when the task ends, even on error or panic
        run_pipeline().await
    });
    Ok(())
}
```

| Method | Behaviour when no permit is free |
|---|---|
| `acquire().await` / `acquire_owned().await` | Waits |
| `try_acquire()` / `try_acquire_owned()` | Fails immediately |

Use `_owned` variants when the permit must move into a spawned task. The permit is released on `Drop`, so it's never leaked by an early `return` or `?`.

---

## 9. Cancellation: `CancellationToken`

Stage 10 relied on "consumer dropped, so `send` fails". That covers one direction. A pipeline also needs **explicit** cancellation: a user presses Stop, a timeout fires, or another stage fails.

```toml
tokio-util = "0.7"
```

```rust
use tokio_util::sync::CancellationToken;

let token = CancellationToken::new();
let child = token.child_token();           // cancelled when the parent is, can't cancel the parent

tokio::spawn(async move {
    loop {
        tokio::select! {
            _ = child.cancelled() => break,              // stop promptly
            item = rx.recv() => match item {
                Some(item) => process(item).await,
                None => break,                           // upstream finished
            },
        }
    }
});

token.cancel();                            // every clone and child sees it
```

Rules:

- **Every loop that can wait** selects on `cancelled()` together with its `recv` or `send`. A stage that only checks `is_cancelled()` between items can still sit forever in a `send().await`.
- Pass **child tokens** down. A sub-pipeline can be cancelled alone without stopping its parent.
- Cancellation is **cooperative**. It doesn't kill a running `spawn_blocking` closure, so keep blocking units small or check a flag inside them.
- `JoinSet::abort_all()` is the forceful backup. It stops tasks at their next `.await`, without letting them clean up.

> **JS comparison:** `CancellationToken` is `AbortController`. `cancelled()` is `signal.addEventListener("abort", ...)`, and `child_token()` is like `AbortSignal.any([parent])`.

---

## 10. Graceful shutdown

When the process gets SIGTERM (Kubernetes rolling update, Ctrl+C), running pipelines should stop **cleanly**: stop reading new input, finish or roll back what's in flight, and close files properly (a Parquet file without its footer is unreadable, Stage 14).

```rust
let shutdown = CancellationToken::new();

// Signal listener
{
    let shutdown = shutdown.clone();
    tokio::spawn(async move {
        let _ = tokio::signal::ctrl_c().await;
        shutdown.cancel();
    });
}

// Each run gets a child token
let run_token = shutdown.child_token();
let result = run_pipeline(run_token).await;

// Give in-flight work a deadline, then stop forcefully
match tokio::time::timeout(Duration::from_secs(30), set.join_all_tasks()).await {
    Ok(_) => tracing::info!("pipelines stopped cleanly"),
    Err(_) => { set.abort_all(); tracing::warn!("forced shutdown after 30s"); }
}
```

(`join_all_tasks` stands for "loop over `join_next()` until it returns `None`".)

The order matters:

1. **Stop the source.** No new input enters.
2. **Drain or drop** what's in the channels, depending on whether partial output is acceptable.
3. **Finalize sinks.** Flush buffers, write footers, commit or roll back transactions.
4. **Deadline**, then `abort_all()`.

Axum's `serve(...).with_graceful_shutdown(shutdown.clone().cancelled_owned())` uses the same token, so HTTP and pipelines stop together.

---

## 11. Branching and side outputs

A validation stage often splits its input: valid records go on, invalid ones go to a **side output** (a rejects file, an error table, a report).

```text
                         ┌─▶ [valid channel, cap 8]  ─▶ main sink
 parsed ──▶ router ──────┤
                         └─▶ [reject channel, cap 8] ─▶ rejects sink
```

```rust
while let Some(item) = parsed.next().await {
    let delivered = match item {
        Ok(record) => valid_tx.send(record).await.is_ok(),
        Err(reject) => reject_tx.send(reject).await.is_ok(),
    };
    if !delivered {
        break;      // a sink went away: stop the whole branch point
    }
}
```

What keeps this bounded and deadlock-free:

| Rule | Why |
|---|---|
| Every branch has a **bounded** channel | One slow sink can't make the other branch's data pile up |
| Every branch has a **running consumer** | If nobody reads the reject channel, the router blocks after 8 rejects, and so does the main path |
| The router waits on `send().await` | The slowest branch sets the pace. That's backpressure, not a bug |
| A closed branch stops the router | Otherwise the router silently drops records for that branch |

**Broadcasting** (every item to every branch) is the same pattern with one `send` per branch. If items are large, send an `Arc<T>` (or an Arrow `RecordBatch`, which is already reference-counted) so branches share **one** value instead of each getting a copy.

Avoid `tokio::sync::broadcast` for data. It has a fixed buffer, but a slow receiver **loses** messages (`RecvError::Lagged`) instead of slowing the sender. That's fine for notifications, wrong for records.

---

## 12. Errors in a pipeline

Decide per error type whether it's **data** or **failure**:

| Kind | Example | Handling |
|---|---|---|
| Bad record | Unparseable line, failed validation rule | Route to the side output, keep going |
| Stage failure | Database down, disk full, panic in a task | Cancel the token, stop all stages, report failure |
| Too many bad records | 40% of input rejected | Treat as failure once a threshold is passed |

`try_` combinators stop at the first `Err`:

```rust
use futures::TryStreamExt;

let total = lines
    .map_ok(|l| l.len())          // only touches Ok values
    .try_fold(0, |acc, n| async move { Ok(acc + n) })
    .await?;                      // first Err ends the whole stream
```

Rules:

- **Bad data is an `Ok` item for the router** (for example `Result<Record, Reject>` inside the stream). Only real failures use the outer `Err`.
- When one stage fails, **cancel the token** so the others stop, then report the **first** error. Later errors are usually just "channel closed" side effects.
- Log the cause with `tracing::error!` and include counts (rows read, written, rejected) in the final report, so a failed run is diagnosable.

---

## 13. Testing pipelines: early output, bounded residency, cancellation

Stage 10, section 11 named the three properties. For pipelines, test them **per stage** and for the **whole chain**:

| Property | How to measure |
|---|---|
| Output before input EOF | Record whether the source had finished when the sink saw its first item |
| Bounded residency | Atomic counter: +1 when the source produces, −1 when a sink consumes. Track the maximum and compare it across small and large inputs |
| Cancellation | Cancel after N items. Assert the source stopped near N, not at the end |
| Ordering (if promised) | Sink checks that IDs increase |

Use Tokio's paused clock for stages with timers or timeouts, so tests are fast and deterministic:

```toml
[dev-dependencies]
tokio = { version = "1", features = ["full", "test-util"] }
```

```rust
#[tokio::test(start_paused = true)]
async fn batches_flush_after_timeout_without_waiting_for_more_input() {
    // time only moves when every task is idle, so a 5s timeout passes instantly
}
```

The key assertion for bounded residency isn't a specific number. It's that the maximum **doesn't grow** when the input grows by 10× or 100×.

---

## 14. How this looks in a production service

```text
crates/
├── pipeline/src/
│   ├── stages/
│   │   ├── source.rs       # produces owned items into a bounded channel; selects on cancel
│   │   ├── parse.rs        # spawn_blocking per chunk + buffered(n)
│   │   ├── validate.rs     # router: valid / reject channels
│   │   └── sink.rs         # writes batch by batch, finalizes on success, rolls back on cancel
│   ├── run.rs              # wires stages, owns the JoinSet and CancellationToken
│   └── report.rs           # rows read / written / rejected, first error
└── pipeline/tests/
    └── pipeline_streaming.rs   # early output, bounded residency at 10x input, cancellation, order
```

Conventions:

- **One `JoinSet` and one `CancellationToken` per run.** Dropping the run aborts everything.
- **Every stage has a fixed limit**: channel capacity, `buffered(n)`, or chunk size. Write the overall bound down.
- **`Semaphore` in `AppState`** limits concurrent runs and shared external calls.
- **Every waiting point selects on cancellation.**
- **Bad records are data, failures are errors**, with a reject threshold.
- **Side outputs have their own bounded channel and a consumer that always runs.**
- **Large values are shared** (`Arc`, `Bytes`, `RecordBatch`), never copied per branch.
- **Graceful shutdown**: stop the source, finalize sinks, then a deadline and `abort_all()`.

---

## 15. Putting it all together

A four-stage pipeline: **source → parallel parse → router → valid sink + rejects sink**. It runs on two input sizes and then once with cancellation, and prints the checks from section 13.

```toml
[dependencies]
futures = "0.3"
tokio = { version = "1", features = ["full"] }
tokio-stream = "0.1"
tokio-util = "0.7"
```

```rust
use std::collections::HashMap;
use std::sync::atomic::{AtomicBool, AtomicI64, AtomicU64, Ordering::Relaxed};
use std::sync::Arc;

use futures::StreamExt;
use tokio::sync::mpsc;
use tokio::task::JoinSet;
use tokio_stream::wrappers::ReceiverStream;
use tokio_util::sync::CancellationToken;

const CHANNEL_CAPACITY: usize = 4;
const PARALLELISM: usize = 4;

// ================= data =================

#[derive(Debug)]
struct Record {
    id: u64,
}

#[derive(Debug)]
struct Reject {
    line: String,
    reason: String,
}

fn parse(line: String) -> Result<Record, Reject> {
    let mut parts = line.split(',');
    let id: Option<u64> = parts.next().and_then(|s| s.parse().ok());
    let amount: Option<u64> = parts.next().and_then(|s| s.parse().ok());
    match (id, amount) {
        (Some(id), Some(_amount)) => Ok(Record { id }),
        _ => Err(Reject { reason: "amount is not a number".into(), line }),
    }
}

// ================= metrics =================

#[derive(Default)]
struct Metrics {
    source_done: AtomicBool,
    first_before_eof: AtomicBool,
    in_flight: AtomicI64,      // produced by the source, not yet consumed by a sink
    max_in_flight: AtomicI64,
    out_of_order: AtomicU64,
}

impl Metrics {
    fn enter(&self) {
        let now = self.in_flight.fetch_add(1, Relaxed) + 1;
        self.max_in_flight.fetch_max(now, Relaxed);
    }
    fn leave(&self) {
        self.in_flight.fetch_sub(1, Relaxed);
    }
}

// ================= the pipeline =================

struct Report {
    sent: u64,
    valid: u64,
    rejected: u64,
    source_finished: bool,
    first_before_eof: bool,
    out_of_order: u64,
    max_in_flight: i64,
}

async fn run(rows: u64, cancel_after: Option<u64>) -> Report {
    let metrics = Arc::new(Metrics::default());
    let token = CancellationToken::new();

    let (raw_tx, raw_rx) = mpsc::channel::<String>(CHANNEL_CAPACITY);
    let (valid_tx, mut valid_rx) = mpsc::channel::<Record>(CHANNEL_CAPACITY);
    let (reject_tx, mut reject_rx) = mpsc::channel::<Reject>(CHANNEL_CAPACITY);

    let mut set: JoinSet<(&'static str, u64)> = JoinSet::new();

    // ---- stage 1: source (every 10th line is bad) ----
    {
        let metrics = Arc::clone(&metrics);
        let token = token.child_token();
        set.spawn(async move {
            let mut sent = 0;
            for id in 1..=rows {
                let line = if id % 10 == 0 { format!("{id},oops") } else { format!("{id},{}", id % 100) };
                metrics.enter();
                tokio::select! {
                    _ = token.cancelled() => { metrics.leave(); break; }
                    res = raw_tx.send(line) => {
                        if res.is_err() { metrics.leave(); break; }
                    }
                }
                sent += 1;
            }
            if sent == rows {
                metrics.source_done.store(true, Relaxed);
            }
            ("source", sent)
        });
    }

    // ---- stage 2 + 3: parallel parse, then route to valid / rejects ----
    {
        let token = token.child_token();
        set.spawn(async move {
            let mut parsed = ReceiverStream::new(raw_rx)
                .map(|line| tokio::task::spawn_blocking(move || parse(line)))
                .buffered(PARALLELISM); // bounded, keeps input order

            let mut routed = 0;
            while let Some(joined) = parsed.next().await {
                let Ok(result) = joined else {
                    token.cancel(); // a parse task panicked: stage failure
                    break;
                };
                let delivered = tokio::select! {
                    _ = token.cancelled() => false,
                    ok = async {
                        match result {
                            Ok(record) => valid_tx.send(record).await.is_ok(),
                            Err(reject) => reject_tx.send(reject).await.is_ok(),
                        }
                    } => ok,
                };
                if !delivered {
                    break;
                }
                routed += 1;
            }
            ("router", routed)
        });
    }

    // ---- stage 4a: valid sink (slow-ish) ----
    {
        let metrics = Arc::clone(&metrics);
        let token = token.clone();
        set.spawn(async move {
            let mut count = 0;
            let mut last_id = 0;
            while let Some(record) = valid_rx.recv().await {
                metrics.leave();
                if count == 0 {
                    metrics.first_before_eof.store(!metrics.source_done.load(Relaxed), Relaxed);
                }
                if record.id <= last_id {
                    metrics.out_of_order.fetch_add(1, Relaxed);
                }
                last_id = record.id;
                count += 1;
                if Some(count) == cancel_after {
                    token.cancel(); // e.g. the user pressed Stop
                    break;
                }
                tokio::task::yield_now().await;
            }
            ("valid", count)
        });
    }

    // ---- stage 4b: rejects sink (side output) ----
    {
        let metrics = Arc::clone(&metrics);
        set.spawn(async move {
            let mut count = 0;
            while let Some(reject) = reject_rx.recv().await {
                metrics.leave();
                if count == 0 {
                    println!("  first reject: {:?} ({})", reject.line, reject.reason);
                }
                count += 1;
            }
            ("rejects", count)
        });
    }

    // ---- wait for every stage ----
    let mut counts: HashMap<&'static str, u64> = HashMap::new();
    while let Some(result) = set.join_next().await {
        match result {
            Ok((stage, n)) => {
                counts.insert(stage, n);
            }
            Err(e) => {
                eprintln!("stage failed: {e}");
                token.cancel();
            }
        }
    }

    Report {
        sent: counts.get("source").copied().unwrap_or(0),
        valid: counts.get("valid").copied().unwrap_or(0),
        rejected: counts.get("rejects").copied().unwrap_or(0),
        source_finished: metrics.source_done.load(Relaxed),
        first_before_eof: metrics.first_before_eof.load(Relaxed),
        out_of_order: metrics.out_of_order.load(Relaxed),
        max_in_flight: metrics.max_in_flight.load(Relaxed),
    }
}

#[tokio::main]
async fn main() {
    // 1 + 2. early output and bounded residency at two input sizes
    for rows in [10_000, 200_000] {
        let r = run(rows, None).await;
        println!(
            "rows={rows:<7} valid={:<7} rejected={:<6} first_before_eof={} out_of_order={} max_in_flight={}",
            r.valid, r.rejected, r.first_before_eof, r.out_of_order, r.max_in_flight
        );
    }

    // 3. cancellation: stop after 50 valid records
    let r = run(200_000, Some(50)).await;
    println!(
        "cancelled after 50 valid: source sent {} of 200000 rows (finished: {})",
        r.sent, r.source_finished
    );
}
```

Expected output. Exact `max_in_flight` and `sent` values vary between runs because of scheduling, but they stay small and **don't grow with the input**:

```text
  first reject: "10,oops" (amount is not a number)
rows=10000   valid=9000    rejected=1000   first_before_eof=true out_of_order=0 max_in_flight=18
  first reject: "10,oops" (amount is not a number)
rows=200000  valid=180000  rejected=20000  first_before_eof=true out_of_order=0 max_in_flight=18
  first reject: "10,oops" (amount is not a number)
cancelled after 50 valid: source sent 72 of 200000 rows (finished: false)
```

What each check shows:

| Check | Concept |
|---|---|
| `first_before_eof=true` | **Early output** through four stages, not just one |
| Same `max_in_flight` for 10,000 and 200,000 rows | **Bounded residency**: about 3 channels × capacity + `PARALLELISM` + a few in hand |
| `out_of_order=0` | `buffered` keeps **order** while parsing in parallel |
| Rejects counted separately, main path still completes | **Side output** with its own bounded channel and consumer |
| Source stopped near row 72 | **Cancellation** through `CancellationToken` and closed channels |
| `JoinSet` returns every stage's count | **Task group**: no detached work left behind |

Try breaking it on purpose:

- Replace `buffered` with `buffer_unordered` and watch `out_of_order` become non-zero.
- Remove the rejects sink task (keep `reject_rx` alive but unread). The pipeline **hangs** after 4 rejects: a side output without a consumer blocks the main path.
- Make the raw channel `unbounded_channel()` and watch `max_in_flight` grow with `rows`.
- Remove the `token.cancelled()` branch from the source's `select!`, and cancel from the valid sink. The source still stops, because the channels close, but only after the router notices. Compare `sent`.

---

## 16. Exercises

- [ ] **1. `try_stream!` stage:** write `chunk_lines` from section 4 and a check that the first chunk is yielded before the input stream ends (use a source that never ends and `take(1)` on the output).
- [ ] **2. `buffered` vs `buffer_unordered`:** map 20 items to futures that sleep a random 0–50 ms. Print output order and total time for `.then`, `.buffered(5)`, and `.buffer_unordered(5)`. Explain the differences in two sentences.
- [ ] **3. Per-chunk CPU work:** change the section 15 parse stage to parse **chunks of 256 lines** in one `spawn_blocking` call. Keep ordering and the bound. Compare run time with the per-line version.
- [ ] **4. Shared limit:** add `Arc<Semaphore>` with 2 permits to an Axum app with `POST /runs` that starts the section 15 pipeline in the background. A third concurrent request must return `409 Conflict`. The permit must be released when the run ends, including on cancel.
- [ ] **5. Stop endpoint:** keep a `HashMap<RunId, CancellationToken>` in state (behind a `tokio::sync::Mutex`, Stage 1). `POST /runs/{id}/stop` cancels it. Prove with a test that the source stops shortly after.
- [ ] **6. Graceful shutdown:** wire `tokio::signal::ctrl_c()` to a root token, give each run a child token, and make the sink write a final `{"type":"end","status":"cancelled","rows":N}` line on cancel. Add a 10-second deadline, then `abort_all()`.
- [ ] **7. Reject threshold:** fail the run when more than 20% of the first 1,000 records are rejected. Report the first error and the counts. All stages must stop.
- [ ] **8. Broadcast branch:** add a second consumer that receives **every** valid record (for example, a counter per `id % 10`). Send `Arc<Record>` so both branches share one value. Show that `max_in_flight` is still constant.
- [ ] **9. Real tests:** turn the section 15 checks into `#[tokio::test]` functions in `tests/pipeline_streaming.rs`: early output, bounded residency at 10× input, cancellation, order, and "side output without consumer is detected by a timeout" (use `start_paused = true` and `tokio::time::timeout`).
- [ ] **10. Explain it:** in four or five sentences, explain why `buffered(n)` is both a concurrency limit and a memory limit, why every side output needs a running consumer, the difference between dropping a channel and cancelling a token, and why `broadcast` is wrong for data.

> When you can solve exercises 1–10 without looking anything up, you're ready for **Stage 16: Job Orchestration**.

---

## 17. Further reading

- [`futures::StreamExt`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html) and [`TryStreamExt`](https://docs.rs/futures/latest/futures/stream/trait.TryStreamExt.html): `buffered`, `buffer_unordered`, `try_fold`, and more
- [`async-stream`](https://docs.rs/async-stream): `stream!` and `try_stream!`
- [Tokio: `JoinSet`](https://docs.rs/tokio/latest/tokio/task/struct.JoinSet.html) and [`Semaphore`](https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html)
- [`tokio_util::sync::CancellationToken`](https://docs.rs/tokio-util/latest/tokio_util/sync/struct.CancellationToken.html)
- [Tokio tutorial: Graceful shutdown](https://tokio.rs/tokio/topics/shutdown)
- [Tokio: testing with paused time](https://docs.rs/tokio/latest/tokio/time/fn.pause.html)
- [Without boats: "Pin"](https://without.boats/blog/pin/): why `Pin` exists and how to think about it
- [Alice Ryhl: "Async: What is blocking?"](https://ryhl.io/blog/async-what-is-blocking/): when to use `spawn_blocking`
