# Stage 10: Streaming Large Data

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> If you come from JavaScript or TypeScript, the comparisons below map Rust concepts to familiar JS ideas (Node streams and `fetch` response readers).

## Contents

1. [Why stream?](#1-why-stream)
2. [The `Stream` trait](#2-the-stream-trait)
3. [Streaming a response with `Body::from_stream`](#3-streaming-a-response-with-bodyfrom_stream)
4. [NDJSON: a format made for streaming](#4-ndjson-a-format-made-for-streaming)
5. [Backpressure and bounded channels](#5-backpressure-and-bounded-channels)
6. [Streaming rows from the database](#6-streaming-rows-from-the-database)
7. [Errors after the response has started](#7-errors-after-the-response-has-started)
8. [Cancellation: when the client goes away](#8-cancellation-when-the-client-goes-away)
9. [Streaming request bodies (uploads)](#9-streaming-request-bodies-uploads)
10. [Batching without losing the bound](#10-batching-without-losing-the-bound)
11. [Proving it streams: tests that matter](#11-proving-it-streams-tests-that-matter)
12. [How this looks in a production service](#12-how-this-looks-in-a-production-service)
13. [Putting it all together](#13-putting-it-all-together)
14. [Exercises](#14-exercises)
15. [Further reading](#15-further-reading)

---

## 1. Why stream?

So far every handler built its whole response in memory and then sent it:

```rust
let rows = db::projects::list_all(&pool, tenant).await?;   // Vec with every row
Ok(Json(rows))                                             // one big JSON string
```

With 50 rows that's fine. With 5 million rows (an export, a report, a data migration):

| Problem | Effect |
|---|---|
| Memory | The `Vec` **and** the serialized JSON are both fully in RAM: gigabytes per request |
| Latency | The client sees nothing until the last row is read and serialized |
| Concurrency | Three simultaneous exports can exhaust the server's memory |
| Wasted work | If the client gives up, the server still finishes building everything |

**Streaming** sends data **as it's produced**: read a few rows, write them out, read the next few. Memory stays flat no matter how big the result is, and the first bytes arrive almost immediately.

The rule from the README conventions:

> Stream large results instead of collecting them into memory. Bounded channels and buffers only.

And the stricter version, which this stage is about: **returning a stream isn't enough if building it first drains or collects its input.** Every step must produce output incrementally, with a bounded amount of data in flight.

> **JS comparison:** this is the difference between `res.json(await getAllRows())` and `dbCursor.pipe(transform).pipe(res)` with Node streams. Node streams handle backpressure through `highWaterMark` and `pipe`. In Rust you get the same with `Stream`s and bounded channels.

---

## 2. The `Stream` trait

A `Future` produces **one** value later. A `Stream` produces **many** values over time, the async version of an `Iterator`:

| | One value | Many values |
|---|---|---|
| Sync | `T` | `Iterator<Item = T>` |
| Async | `Future<Output = T>` | `Stream<Item = T>` |

```toml
futures = "0.3"
```

```rust
use futures::{stream, StreamExt};

#[tokio::main]
async fn main() {
    let mut numbers = stream::iter(1..=5)
        .map(|n| n * 10)               // like Iterator::map, but lazy and async
        .filter(|n| futures::future::ready(*n != 30));

    while let Some(n) = numbers.next().await {
        println!("{n}");               // 10, 20, 40, 50
    }
}
```

Key points:

- `StreamExt` (from `futures`) adds `map`, `filter`, `then`, `take`, `chunks`, `ready_chunks`, `next`, and more.
- Like futures, streams are **lazy**: nothing happens until someone polls them. The consumer pulls items one at a time, and that pull is what makes backpressure work.
- `stream.collect::<Vec<_>>()` exists, and it's exactly the thing to **avoid** for large data, because it drains the whole stream into memory.

### 2.1 Building streams

| Source | How |
|---|---|
| An iterator | `stream::iter(iter)` |
| A loop with state | `stream::unfold(state, \|state\| async { ... Some((item, state)) })` |
| A bounded channel | `stream::unfold(rx, \|mut rx\| async { rx.recv().await.map(\|x\| (x, rx)) })` or `tokio_stream::wrappers::ReceiverStream` |
| Database rows | `sqlx::query_as(...).fetch(&pool)` (section 6) |
| A request body | `body.into_data_stream()` (section 9) |

---

## 3. Streaming a response with `Body::from_stream`

Axum turns any stream of byte chunks into a response body:

```rust
use axum::{body::{Body, Bytes}, http::header, response::IntoResponse};
use futures::stream;

async fn count() -> impl IntoResponse {
    let chunks = stream::iter(1..=5).map(|n| Ok::<_, std::convert::Infallible>(Bytes::from(format!("{n}\n"))));
    ([(header::CONTENT_TYPE, "text/plain")], Body::from_stream(chunks))
}
```

- The stream's items must be `Result<impl Into<Bytes>, impl Into<BoxError>>`. An `Err` item aborts the response (section 7).
- The status code and headers are sent **first**. The body follows chunk by chunk, using HTTP/1.1 chunked transfer encoding (or HTTP/2 data frames).
- Hyper (Axum's HTTP layer) only polls the stream for the next chunk when it has room to write. That's the network end of backpressure: a slow client means the stream is polled slowly.

`Bytes` is a cheap, reference-counted byte buffer. Cloning it doesn't copy the data, and Hyper can write it to the socket without another copy.

---

## 4. NDJSON: a format made for streaming

A normal JSON array can't be parsed until the closing `]` arrives, and producing one correctly while streaming means tracking commas. **NDJSON** (newline-delimited JSON, also called JSON Lines) is one JSON object per line:

```text
{"id":1,"name":"Alpha"}
{"id":2,"name":"Beta"}
{"id":3,"name":"Gamma"}
```

| | JSON array | NDJSON |
|---|---|---|
| Write incrementally | Awkward (commas, brackets) | One `serialize + "\n"` per record |
| Read incrementally | Needs a streaming JSON parser | Split on `\n`, parse each line |
| Content type | `application/json` | `application/x-ndjson` |
| Partial results usable | No | Yes, every complete line is valid |

```rust
fn to_ndjson_line<T: serde::Serialize>(record: &T) -> Result<Bytes, serde_json::Error> {
    let mut line = serde_json::to_vec(record)?;
    line.push(b'\n');
    Ok(Bytes::from(line))
}
```

CSV works the same way for spreadsheet exports, one row per line, with the `csv` crate writing into a small buffer per record.

> **JS comparison:** reading NDJSON in the browser:
> ```js
> const reader = res.body.pipeThrough(new TextDecoderStream()).getReader();
> let buf = "";
> for (;;) {
>   const { value, done } = await reader.read();
>   if (done) break;
>   buf += value;
>   const lines = buf.split("\n");
>   buf = lines.pop();                       // keep the incomplete last line
>   for (const line of lines) if (line) handle(JSON.parse(line));
> }
> ```

---

## 5. Backpressure and bounded channels

**Backpressure** means a fast producer is slowed down to the speed of a slow consumer, instead of piling data up in memory.

```text
 database ──▶ producer task ──▶ [ bounded channel, capacity 8 ] ──▶ response stream ──▶ socket ──▶ client
   fast            fast          full? then send().await waits       polled only when      slow
                                                                    the socket has room
```

The chain only stays bounded if **every** link waits for the next one:

| Link | Bounded how |
|---|---|
| Socket → response stream | Hyper polls only when it can write |
| Response stream → channel | `rx.recv()` only runs when the stream is polled |
| Channel → producer | `tx.send(x).await` waits while the channel is full |
| Producer → database | The producer only asks for the next row after the send completes |

At any moment, at most about **capacity + a couple of items** exist in memory per request, whether the export has 100 rows or 100 million.

What breaks the bound:

| Mistake | Why it's unbounded |
|---|---|
| `mpsc::unbounded_channel()` | The producer never waits |
| `stream.collect::<Vec<_>>().await` then streaming the `Vec` | Everything was materialized first |
| `fetch_all` then `stream::iter(rows)` | Same thing, hidden in the database call |
| A `Vec` buffer that grows until "done" | Its size depends on the input |
| `tokio::spawn` per row | Spawned tasks run regardless of the consumer |

The pattern is the same as the Stage 1 job queue: bounded `mpsc` plus `send().await`. Here the consumer happens to be an HTTP response.

---

## 6. Streaming rows from the database

SQLx's `fetch` returns a stream that reads rows from PostgreSQL as they arrive (Stage 6, section 13.2):

```rust
let rows = sqlx::query_as::<_, ProjectRow>("SELECT ... FROM projects WHERE tenant_id = $1")
    .bind(tenant_id)
    .fetch(&pool);   // impl Stream<Item = Result<ProjectRow, sqlx::Error>> + '_
```

You might try to map it and hand it straight to `Body::from_stream`. It doesn't compile, because the stream **borrows** `pool` (and the SQL string), while a response body must be `'static`: it outlives the handler.

The standard fix is a **producer task** that owns its own pool clone and feeds a **bounded channel**:

```rust
async fn export_projects(
    State(state): State<AppState>,
    user: AuthUser,
) -> Result<Response, AppError> {
    user.require(caps::PROJECTS_READ)?;                       // checks happen BEFORE streaming

    let (tx, rx) = mpsc::channel::<Result<Bytes, AppStreamError>>(16);
    let pool = state.pool.clone();                            // cheap: Arc inside
    let tenant_id = user.tenant_id;

    tokio::spawn(async move {
        let mut rows = sqlx::query_as::<_, ProjectRow>(
            "SELECT id, tenant_id, name, description, version
             FROM projects WHERE tenant_id = $1 ORDER BY id",
        )
        .bind(tenant_id)
        .fetch(&pool);

        while let Some(row) = rows.next().await {
            let item = row
                .map_err(AppStreamError::from)
                .and_then(|r| to_ndjson_line(&ProjectDto::from(r)).map_err(AppStreamError::from));
            let failed = item.is_err();
            if tx.send(item).await.is_err() {
                return;                                       // client went away (section 8)
            }
            if failed {
                return;                                       // stop after reporting an error
            }
        }
    });

    let body = stream::unfold(rx, |mut rx| async move { rx.recv().await.map(|item| (item, rx)) });
    Ok(([(header::CONTENT_TYPE, "application/x-ndjson")], Body::from_stream(body)).into_response())
}
```

Notes:

- **Do auth, capability, and validation checks before** creating the stream. Once the `200` is sent, you can't change it to a 403.
- The query is still **tenant-scoped** (Stage 6, section 9). Streaming doesn't change any of the safety rules.
- A streaming query holds a pool connection **for the whole export**. Limit concurrent exports (for example with a `tokio::sync::Semaphore` in state) so long exports can't starve normal requests of connections.
- Exclude export routes from the global `TimeoutLayer` (Stage 7), or give them a much longer timeout, so long-but-healthy exports aren't cut off.

---

## 7. Errors after the response has started

Before the first byte, errors are normal `AppError` responses. After the `200` and the first chunks are sent, **the status code can't change**. Two options:

| Option | How | Client sees |
|---|---|---|
| Abort | The stream yields `Err(e)` | The connection is cut, and the response is visibly truncated (chunked encoding never finishes) |
| In-band error | Write a final line like `{"type":"error","message":"export failed"}` and end normally | A complete response whose last line says it failed |

Production APIs often do both: write an error line for clients that understand it, then end the stream. Either way:

- **Log** the real cause with `tracing::error!` (Stage 7). The client gets a generic message, as with any 500 (Stage 4, section 9).
- Document that clients must treat a missing final `{"type":"end","rows":N}` line (or a truncated body) as failure. An explicit end marker with a row count makes truncation easy to detect.

---

## 8. Cancellation: when the client goes away

If the user closes the browser tab, Hyper notices the closed connection and **drops** the response body (Stage 1, section 6.3). With the producer-and-channel pattern, that cancellation travels backwards:

```text
client disconnects
   → Hyper drops the body stream
   → the stream drops rx (the channel receiver)
   → the producer's next tx.send(...).await returns Err
   → the producer returns, dropping the SQLx stream
   → the pool connection goes back to the pool
```

So **always check the result of `send`** and stop when it fails. A producer that ignores send errors (`let _ = tx.send(x).await;`) keeps reading the whole table for nobody.

If the producer holds a transaction, dropping it rolls the transaction back (Stage 6, section 10), so a cancelled export never leaves partial writes.

---

## 9. Streaming request bodies (uploads)

Stage 3's `Json`, `String`, and `Bytes` extractors **buffer the whole body**. For large uploads, take the body as a stream instead:

```rust
use axum::body::Body;
use futures::TryStreamExt;
use tokio::io::AsyncWriteExt;

async fn upload(body: Body) -> Result<String, AppError> {
    let mut stream = body.into_data_stream();
    let mut file = tokio::fs::File::create("/tmp/upload.bin")
        .await
        .map_err(|e| AppError::Internal(Box::new(e)))?;
    let mut total: u64 = 0;

    while let Some(chunk) = stream
        .try_next()
        .await
        .map_err(|e| AppError::BadRequest(e.to_string()))?
    {
        total += chunk.len() as u64;
        file.write_all(&chunk).await.map_err(|e| AppError::Internal(Box::new(e)))?;
    }
    file.flush().await.map_err(|e| AppError::Internal(Box::new(e)))?;
    Ok(format!("stored {total} bytes"))
}
```

- Only one chunk is in memory at a time. The disk write's `.await` provides backpressure all the way back to the client's TCP connection.
- Axum's 2 MB `DefaultBodyLimit` applies to buffering extractors. For upload routes, disable it and set an explicit limit with `RequestBodyLimitLayer` (Stage 7, section 8.4), which also applies to streamed bodies.
- To process records from an uploaded NDJSON or CSV file, split the chunks into lines as they arrive (exercise 6). Don't buffer the whole file and then split it.
- For browser file uploads with `multipart/form-data`, Axum's `Multipart` extractor (feature `multipart`) gives you each field as a stream of chunks.

---

## 10. Batching without losing the bound

Sending one tiny chunk per row costs overhead per chunk (and per database round trip, if you also write rows somewhere). **Batching** helps, as long as the batch size is **fixed**:

```rust
let batched = rows
    .ready_chunks(256)                          // up to 256 already-available items, never more
    .map(|batch| {
        let mut buf = Vec::with_capacity(batch.len() * 64);
        for item in batch {
            let row = item?;
            serde_json::to_writer(&mut buf, &ProjectDto::from(row))?;
            buf.push(b'\n');
        }
        Ok::<_, AppStreamError>(Bytes::from(buf))
    });
```

- `ready_chunks(n)` groups items that are **already** available, up to `n`, without waiting for more. It never delays the first output to fill a batch.
- `chunks(n)` waits until `n` items arrive (or the stream ends), which delays output for slow sources.
- A byte-size limit (flush when the buffer exceeds, say, 64 KB) is often better than a row count, because row sizes vary.

The bound is now "capacity × batch size" instead of "capacity". It's larger, but still **constant**: it doesn't grow with the input.

---

## 11. Proving it streams: tests that matter

"It returns a stream" doesn't prove it streams. A function can return `stream::iter(vec)` after collecting everything. Tests should show two properties:

1. **Output is observable before input EOF.** Feed the endpoint a source that is much larger than the buffer (or one that never ends), read the first chunk, and assert the producer has **not** finished. If the first chunk only arrives after everything was produced, the code materializes its input.
2. **Resident data stays bounded as input grows.** Count items "produced but not yet consumed" (an atomic counter in the producer and one in the consumer), track the maximum while reading slowly, and assert it stays below a constant such as `capacity + batch_size`. Run with 1,000 and 1,000,000 items: the maximum must not change.

A third useful test: **cancellation stops the producer.** Drop the response body after a few chunks, and assert the producer stopped long before the end of its input.

Section 13 does all three. Stage 11 turns them into proper `#[tokio::test]` functions.

---

## 12. How this looks in a production service

```text
crates/
├── db/src/projects.rs          # fn stream_for_export(pool, tenant) -> impl Stream<..> + '_
├── domain/src/export.rs        # producer task: rows -> DTOs -> bounded channel
└── api/src/
    ├── streaming.rs            # ndjson_body(rx) -> Body, AppStreamError, end/error marker lines
    └── routes/exports.rs       # GET /projects/export: checks first, then stream
```

Conventions:

- **All checks before the first byte:** auth, capability, tenant, validation.
- **One producer task + bounded channel** per streamed response, with send errors stopping the producer.
- **NDJSON** (or CSV) for large results, with an explicit end marker.
- Every buffer has a **fixed** maximum: channel capacity, batch size, chunk size.
- Long-running routes have their own timeout and concurrency limits, and their own body-size limits for uploads.
- Tests prove early output, bounded residency, and cancellation, not just correct final output.

---

## 13. Putting it all together

An NDJSON export endpoint with a producer task, a bounded channel, and a response stream, plus three checks: early output, bounded residency, and cancellation. The "database" is a generated sequence so it runs without Postgres, and the client reads the body chunk by chunk through `oneshot`, so it runs **in the Playground**.

```toml
[dependencies]
axum = "0.8"
futures = "0.3"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
tower = { version = "0.5", features = ["util"] }
```

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use std::time::Duration;

use axum::{
    body::{Body, Bytes},
    extract::{Query, Request, State},
    http::header,
    response::{IntoResponse, Response},
    routing::get,
    Router,
};
use futures::{stream, StreamExt};
use serde::{Deserialize, Serialize};
use tokio::sync::mpsc;
use tower::ServiceExt;

const CHANNEL_CAPACITY: usize = 8;

// ================= metrics shared with the test client =================

#[derive(Default)]
struct Metrics {
    produced: AtomicU64,    // rows the producer has handed to the channel
    finished: AtomicU64,    // 1 once the producer reached the end of its input
    stopped_at: AtomicU64,  // row number where the producer noticed the client left
}

#[derive(Clone)]
struct AppState {
    metrics: Arc<Metrics>,
}

// ================= the endpoint =================

#[derive(Deserialize)]
struct ExportParams {
    rows: u64,
}

#[derive(Serialize)]
struct Record {
    id: u64,
    name: String,
}

fn to_ndjson_line<T: Serialize>(record: &T) -> Result<Bytes, serde_json::Error> {
    let mut line = serde_json::to_vec(record)?;
    line.push(b'\n');
    Ok(Bytes::from(line))
}

async fn export(State(state): State<AppState>, Query(params): Query<ExportParams>) -> Response {
    let (tx, rx) = mpsc::channel::<Record>(CHANNEL_CAPACITY);
    let metrics = Arc::clone(&state.metrics);

    // Producer: stands in for a SQLx `fetch` stream
    tokio::spawn(async move {
        for id in 1..=params.rows {
            let record = Record { id, name: format!("item-{id}") };
            if tx.send(record).await.is_err() {
                metrics.stopped_at.store(id, Ordering::Relaxed);
                return; // client went away: stop reading the source
            }
            metrics.produced.fetch_add(1, Ordering::Relaxed);
        }
        metrics.finished.store(1, Ordering::Relaxed);
    });

    // Response stream: pulls from the channel only when Hyper (or our test client) asks
    let body = stream::unfold(rx, |mut rx| async move {
        let record = rx.recv().await?;
        Some((to_ndjson_line(&record), rx))
    });

    ([(header::CONTENT_TYPE, "application/x-ndjson")], Body::from_stream(body)).into_response()
}

fn app(state: AppState) -> Router {
    Router::new().route("/export", get(export)).with_state(state)
}

// ================= checks =================

async fn open_export(rows: u64) -> (Arc<Metrics>, impl futures::Stream<Item = Result<Bytes, axum::Error>>) {
    let metrics = Arc::new(Metrics::default());
    let app = app(AppState { metrics: Arc::clone(&metrics) });
    let request = Request::builder()
        .uri(format!("/export?rows={rows}"))
        .body(Body::empty())
        .expect("static request is valid");
    let response = match app.oneshot(request).await {
        Ok(r) => r,
        Err(never) => match never {},
    };
    (metrics, response.into_body().into_data_stream())
}

/// Reads the whole export slowly and reports how far the producer ever got ahead.
async fn read_all_slowly(rows: u64) {
    let (metrics, mut body) = open_export(rows).await;

    let mut received: u64 = 0;
    let mut max_ahead: u64 = 0;
    let mut first_line = String::new();

    while let Some(chunk) = body.next().await {
        let Ok(chunk) = chunk else {
            println!("stream error");
            return;
        };
        if first_line.is_empty() {
            first_line = String::from_utf8_lossy(&chunk).trim_end().to_owned();
        }
        received += chunk.iter().filter(|b| **b == b'\n').count() as u64;

        let produced = metrics.produced.load(Ordering::Relaxed);
        max_ahead = max_ahead.max(produced.saturating_sub(received));

        tokio::task::yield_now().await; // a slow-ish client
    }

    println!(
        "rows={rows:<7} received={received:<7} first={first_line} max_ahead={max_ahead} (capacity {CHANNEL_CAPACITY})"
    );
}

#[tokio::main]
async fn main() {
    // 1. Early output: the first line arrives long before the producer finishes
    let (metrics, mut body) = open_export(1_000_000).await;
    if let Some(Ok(first)) = body.next().await {
        println!(
            "first chunk: {} | producer finished yet? {} | produced so far: <= {}",
            String::from_utf8_lossy(&first).trim_end(),
            metrics.finished.load(Ordering::Relaxed) == 1,
            CHANNEL_CAPACITY + 2,
        );
    }
    drop(body);

    // 2. Bounded residency: the same small maximum for small and large inputs
    read_all_slowly(1_000).await;
    read_all_slowly(100_000).await;

    // 3. Cancellation: drop the body after 3 chunks; the producer must stop early
    let (metrics, mut body) = open_export(1_000_000).await;
    for _ in 0..3 {
        let _ = body.next().await;
    }
    drop(body);
    tokio::time::sleep(Duration::from_millis(50)).await;
    let stopped = metrics.stopped_at.load(Ordering::Relaxed);
    println!(
        "cancelled after 3 chunks: producer stopped at row {stopped} of 1000000 (finished: {})",
        metrics.finished.load(Ordering::Relaxed) == 1
    );
}
```

Expected output (exact `max_ahead` and `stopped at` values vary slightly between runs because of scheduling, but they stay close to the channel capacity):

```text
first chunk: {"id":1,"name":"item-1"} | producer finished yet? false | produced so far: <= 10
rows=1000    received=1000    first={"id":1,"name":"item-1"} max_ahead=8 (capacity 8)
rows=100000  received=100000  first={"id":1,"name":"item-1"} max_ahead=8 (capacity 8)
cancelled after 3 chunks: producer stopped at row 12 of 1000000 (finished: false)
```

What each check shows:

| Check | Concept |
|---|---|
| First chunk before the producer finished | **Early output**: nothing waits for the end of the input |
| `max_ahead` is the same for 1,000 and 100,000 rows | **Bounded residency**: memory depends on the channel capacity, not the input size |
| Producer stopped near row 12 | **Cancellation**: dropping the body closes the channel and the producer stops |

Try breaking it on purpose: replace the channel with `mpsc::unbounded_channel()` and watch `max_ahead` grow with the input. Or collect all records into a `Vec` before building the stream, and watch the first chunk only arrive after `finished` is true.

---

## 14. Exercises

- [ ] **1. Streams:** use `stream::unfold` to build a stream of the first 20 Fibonacci numbers, then use `.map`, `.filter`, and `.take` to print only the even ones. Don't call `collect`.
- [ ] **2. NDJSON endpoint:** write `GET /numbers?count=N` that streams `{"n":1}` … `{"n":N}` as NDJSON with `Body::from_stream`. Read it with `curl -N` (no buffering) on a local server and watch lines appear.
- [ ] **3. Break the bound:** in the section 13 example, switch to `unbounded_channel` and record `max_ahead` for 1,000 and 100,000 rows. Then switch back and explain the difference in one sentence.
- [ ] **4. End marker and errors:** make the producer fail at row 500 of 1,000. Emit an in-band `{"type":"error","message":"export failed"}` line and log the real cause. On success, emit a final `{"type":"end","rows":N}` line.
- [ ] **5. Database export:** with the Stage 6 project, write `GET /projects/export` that streams tenant-scoped rows from `fetch` through a producer task and a bounded channel. Insert 100,000 rows with `generate_series` and check memory with your OS task manager while exporting.
- [ ] **6. Streaming upload:** write `POST /import` that reads an NDJSON body with `into_data_stream`, splits it into lines across chunk boundaries, and counts valid and invalid lines **without** buffering the whole body. Test it with a line that is split across two chunks.
- [ ] **7. Batching:** add `ready_chunks(64)` to the response stream so each chunk holds up to 64 lines. Update the `max_ahead` check and explain the new bound.
- [ ] **8. Explain it:** in three or four sentences, explain why returning a `Stream` isn't enough to prove a handler streams, what the two properties a streaming test must show are, and why auth checks must run before the first byte.

> When you can solve exercises 1–8 without looking anything up, you're ready for **Stage 11: Testing**.

---

## 15. Further reading

- [`futures::StreamExt` docs](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html): every stream combinator, including `ready_chunks`
- [Tokio tutorial: Streams](https://tokio.rs/tokio/tutorial/streams): streams, `tokio-stream`, and `async-stream`
- [`axum::body::Body` docs](https://docs.rs/axum/latest/axum/body/struct.Body.html): `from_stream` and `into_data_stream`
- [Axum example: `stream-to-file`](https://github.com/tokio-rs/axum/tree/main/examples/stream-to-file): streaming uploads to disk
- [SQLx `fetch`](https://docs.rs/sqlx/latest/sqlx/query/struct.QueryAs.html#method.fetch): streaming rows from the database
- [JSON Lines](https://jsonlines.org/): the NDJSON format
- [Alice Ryhl: "Actors with Tokio"](https://ryhl.io/blog/actors-with-tokio/): bounded channels and why unbounded ones are risky
