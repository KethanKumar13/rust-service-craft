# Stage 18: Performance and Observability

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> If you come from JavaScript or TypeScript, the comparisons below map Rust concepts to familiar JS ideas (`console.time`, `autocannon`, Chrome DevTools flame charts, and `prom-client`).

## Contents

1. [Measure first](#1-measure-first)
2. [Latency is a distribution: p50, p90, p99](#2-latency-is-a-distribution-p50-p90-p99)
3. [Micro-benchmarks with `criterion`](#3-micro-benchmarks-with-criterion)
4. [Load testing an HTTP service](#4-load-testing-an-http-service)
5. [Reading load test results honestly](#5-reading-load-test-results-honestly)
6. [CPU profiling and flamegraphs](#6-cpu-profiling-and-flamegraphs)
7. [Memory profiling](#7-memory-profiling)
8. [Async runtime insight: `tokio-console`](#8-async-runtime-insight-tokio-console)
9. [Tracing spans across async code](#9-tracing-spans-across-async-code)
10. [Metrics: counters, gauges, histograms](#10-metrics-counters-gauges-histograms)
11. [OpenTelemetry in one page](#11-opentelemetry-in-one-page)
12. [Caching](#12-caching)
13. [Common wins in Rust services](#13-common-wins-in-rust-services)
14. [How this looks in a production service](#14-how-this-looks-in-a-production-service)
15. [Putting it all together](#15-putting-it-all-together)
16. [Exercises](#16-exercises)
17. [Further reading](#17-further-reading)

---

## 1. Measure first

Rust is fast by default, which makes it easy to "optimize" the wrong thing. The working loop is always the same:

```text
 measure ──▶ find the bottleneck ──▶ change ONE thing ──▶ measure again ──▶ keep or revert
```

| Question | Tool |
|---|---|
| Is this function fast enough? | `criterion` micro-benchmark (section 3) |
| How does the service behave under load? | Load test: `oha`, `k6` (section 4) |
| Where does CPU time go? | Profiler + flamegraph (section 6) |
| Where does memory go? | Heap profiler: `dhat`, `heaptrack` (section 7) |
| Why are tasks slow or stuck? | `tokio-console` (section 8) |
| What happened in *this* slow request in production? | Tracing spans (section 9) |
| How is the service doing *right now*, overall? | Metrics (section 10) |

The first four are **performance** work: you run them on purpose. The last two are **observability**: they run all the time, so you can answer questions about production you didn't plan for.

Always benchmark **release** builds (`--release`). Debug builds can be 10–50× slower and give misleading results.

---

## 2. Latency is a distribution: p50, p90, p99

The average hides the requests users complain about:

```text
 99 requests at 10 ms + 1 request at 2,000 ms
 average = 29.9 ms        ← looks fine
 p50     = 10 ms          ← the typical request
 p99     = 2,000 ms       ← 1 in 100 users waits 2 seconds
```

| Percentile | Meaning | Use for |
|---|---|---|
| **p50** (median) | Half of requests are faster | Typical experience |
| **p90** | 9 in 10 are faster | Most users' worst case |
| **p99** | 99 in 100 are faster | Tail latency: queueing, locks, GC-like pauses, slow queries |
| **max** | Slowest seen | Outliers, often noise. Look but don't target |

Why the tail matters more than it seems: a page that makes 20 API calls hits the p99 of at least one of them about 18% of the time (1 − 0.99²⁰).

Set targets as percentiles under a stated load, for example *"p99 < 100 ms at 500 requests/second"*. A latency number without a load number means nothing.

---

## 3. Micro-benchmarks with `criterion`

`criterion` runs a function many times, handles warm-up, and reports statistics, including whether a change made it faster or slower.

```toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "serialize"
harness = false
```

```rust name=benches/serialize.rs
use std::hint::black_box;

use criterion::{criterion_group, criterion_main, BenchmarkId, Criterion, Throughput};
use serde::Serialize;

#[derive(Serialize)]
struct Item {
    id: u64,
    name: String,
    tags: Vec<String>,
}

fn items(n: u64) -> Vec<Item> {
    (0..n)
        .map(|id| Item { id, name: format!("item-{id}"), tags: vec!["a".into(), "b".into()] })
        .collect()
}

fn bench_ndjson(c: &mut Criterion) {
    let mut group = c.benchmark_group("ndjson");
    for n in [100u64, 10_000] {
        let data = items(n);
        group.throughput(Throughput::Elements(n));

        // A new Vec per record
        group.bench_with_input(BenchmarkId::new("vec_per_record", n), &data, |b, data| {
            b.iter(|| {
                let mut total = 0;
                for item in data {
                    let mut line = serde_json::to_vec(item).unwrap();
                    line.push(b'\n');
                    total += line.len();
                }
                black_box(total)
            })
        });

        // One reused buffer
        group.bench_with_input(BenchmarkId::new("reused_buffer", n), &data, |b, data| {
            let mut buf = Vec::with_capacity(4096);
            b.iter(|| {
                buf.clear();
                for item in data {
                    serde_json::to_writer(&mut buf, item).unwrap();
                    buf.push(b'\n');
                }
                black_box(buf.len())
            })
        });
    }
    group.finish();
}

criterion_group!(benches, bench_ndjson);
criterion_main!(benches);
```

```bash
cargo bench --bench serialize
cargo bench --bench serialize -- --save-baseline before
# ...change code...
cargo bench --bench serialize -- --baseline before
```

Points to know:

- **`black_box`** stops the compiler from deleting work whose result is never used. Without it, a benchmark can measure nothing.
- **Set up outside `iter`.** Building the input inside the closure benchmarks the setup too.
- **`Throughput`** makes reports show elements/second, which is easier to compare across sizes.
- **Baselines** compare a change against the previous version. Criterion reports "Performance has improved/regressed" with a confidence level.
- `unwrap()` is acceptable in benchmarks and tests: they're not runtime code (README conventions).
- For async code, enable criterion's `async_tokio` feature and use `b.to_async(&runtime).iter(...)`.

Micro-benchmarks answer "which version of this function is faster". They don't tell you whether that function matters for the whole service. That's what profiling (section 6) is for.

> **JS comparison:** `criterion` is a far more rigorous `console.time` loop, similar to `tinybench` or `benchmark.js`, with statistics and saved baselines.

---

## 4. Load testing an HTTP service

A load test sends many concurrent requests and records latency and throughput.

```bash
cargo install oha       # or use k6, wrk, hey

# 50 concurrent connections for 30 seconds
oha -z 30s -c 50 http://localhost:8080/items/1

# fixed rate: 500 requests/second, regardless of how fast responses come back
oha -z 30s -q 500 -c 100 http://localhost:8080/items/1
```

Two models:

| Model | How | Answers |
|---|---|---|
| **Closed** (`-c 50`) | 50 clients, each sends the next request when the last one returns | "What's the max throughput?" |
| **Open** (`-q 500`) | Requests arrive at a fixed rate, like real users | "What's the latency at this traffic level?" |

Open-model tests are closer to production. When the service slows down in a closed test, clients automatically send less. That hides the problem (section 5).

Good practice:

- Run the load generator on a **different machine** (or at least limit its CPU) so it doesn't compete with the service.
- **Warm up** first: the first seconds include connection setup, pool filling, and cold caches.
- Use **realistic data**: a database with 50 rows behaves very differently from one with 5 million.
- Record the **whole setup**: build profile, CPU count, pool size, data size, concurrency. A benchmark nobody can reproduce isn't evidence.

For scripted scenarios (log in, then call several endpoints, with random IDs), use `k6` (JavaScript scripts) or `goose` (Rust).

---

## 5. Reading load test results honestly

A typical `oha` summary:

```text
Summary:
  Success rate: 100.00%
  Requests/sec: 4210.55
Response time distribution:
  50.00% in 11.2 ms
  90.00% in 15.8 ms
  99.00% in 31.4 ms
Status code distribution:
  [200] 126317 responses
```

Check, in order:

1. **Success rate and status codes.** A fast `500` or `429` isn't performance, it's an error. Check these before anything else.
2. **Throughput vs target.** Did it reach the rate you asked for?
3. **p99 vs p50.** A large gap (10× or more) points to queueing: pool exhaustion, a lock, a slow dependency on some requests.
4. **Resource use during the test.** CPU near 100% means you're CPU-bound (profile it). Low CPU with high latency means you're **waiting** on something: database, locks, a too-small pool.

**Coordinated omission:** in a closed test, if the service stalls for 1 second, the clients stall with it and simply don't send the requests that would have waited. Those missing requests never appear in the percentiles, so p99 looks much better than real users would experience. Open-model tests (`-q`) avoid this.

**Find the knee:** run the open test at increasing rates (100, 200, 400, 800 rps). Latency stays flat, then rises sharply. That point is your real capacity. Plan to run well below it.

---

## 6. CPU profiling and flamegraphs

A **sampling profiler** records the call stack many times per second. A **flamegraph** shows the result: width is time spent, height is call depth.

```text
 ┌──────────────────────────────────────────────────────────────┐
 │ main                                                         │
 ├────────────────────────────────────────┬─────────────────────┤
 │ handle_request                         │ tokio::runtime …    │
 ├──────────────────────┬─────────────────┤                     │
 │ serde_json::to_vec   │ sqlx::fetch     │                     │
 ├────────┬─────────────┤                 │                     │
 │ format │ alloc       │                 │                     │
 └────────┴─────────────┴─────────────────┴─────────────────────┘
   wide boxes = where time goes; look for wide plateaus at the top
```

Tools:

| Tool | Platform | Command |
|---|---|---|
| `cargo flamegraph` | Linux (`perf`), macOS (`dtrace`) | `cargo flamegraph --bin service` |
| `samply` | Linux, macOS, Windows | `samply record ./target/release/service` (opens Firefox Profiler) |
| `perf` | Linux | `perf record -g` + `perf report` |

Release builds strip the information profilers need. Keep optimization but add debug symbols:

```toml name=Cargo.toml
[profile.profiling]
inherits = "release"
debug = true
```

```bash
cargo build --profile profiling
samply record ./target/profiling/service
# in another terminal: run the load test, then stop the service
```

What to look for:

- **Wide boxes near the top**: functions that do the work themselves. That's where optimization pays off.
- **`alloc`, `memcpy`, `clone`**: copying and allocation. Often fixable with references, `Arc`, `Bytes`, or reused buffers.
- **`serde`, `format!`**: serialization cost. Batching or reused buffers help (section 3).
- **Lock or `parking_lot` functions**: contention.

A profile of **async** code shows where the CPU was busy, not where tasks were *waiting*. Waiting on the database doesn't use CPU, so it won't appear. For that, use tracing spans (section 9) and `tokio-console` (section 8).

> **JS comparison:** the same as the Chrome DevTools Performance tab or `node --cpu-prof` flame charts.

---

## 7. Memory profiling

Stages 10, 14, and 15 designed for bounded memory. These tools **check** it.

| Tool | What it shows | How |
|---|---|---|
| `dhat` crate | Total and peak heap, allocations per call site | Swap the global allocator in a profiling build |
| `heaptrack` (Linux) | Allocation timeline, leaks, peak | `heaptrack ./target/profiling/service` |
| Process RSS | The number the OS (and Kubernetes) actually sees | `ps`, `/proc/self/status`, container metrics |

`dhat` in a test or a profiling run:

```toml
[dependencies]
dhat = { version = "0.3", optional = true }

[features]
dhat-heap = ["dep:dhat"]
```

```rust
#[cfg(feature = "dhat-heap")]
#[global_allocator]
static ALLOC: dhat::Alloc = dhat::Alloc;

fn main() {
    #[cfg(feature = "dhat-heap")]
    let _profiler = dhat::Profiler::new_heap(); // writes dhat-heap.json when dropped
    // ...
}
```

```bash
cargo run --release --features dhat-heap
# open dhat-heap.json in https://nnethercote.github.io/dh_view/dh_view.html
```

A **memory regression test**: with `dhat::HeapStats::get()` you can assert that peak heap stays under a limit while processing 10× more input. That's a direct, automated proof of the "bounded residency" rule.

The allocator matters for long-running services. glibc's `malloc` can hold on to freed memory, so RSS grows over time even without a leak. Many Rust services switch to `jemalloc` (`tikv-jemallocator`) or `mimalloc` and measure both RSS and throughput before and after.

---

## 8. Async runtime insight: `tokio-console`

`tokio-console` shows live Tokio tasks: how long each was busy, how long it waited, and how long it's been idle. It finds problems that CPU profilers can't:

| Symptom in the console | Likely cause |
|---|---|
| A task with long **busy** time per poll | Blocking code on a runtime thread (Stage 1): move it to `spawn_blocking` |
| Thousands of tasks that never finish | A task leak: `spawn` without a `JoinSet` or cancellation (Stage 15) |
| A task waiting forever | A channel nobody reads, or a deadlock |
| High **scheduled** time | The runtime is overloaded: more ready tasks than threads |

Setup (development only):

```toml
[dependencies]
console-subscriber = "0.4"
```

```toml name=.cargo/config.toml
[build]
rustflags = ["--cfg", "tokio_unstable"]
```

```rust
console_subscriber::init(); // instead of your normal tracing setup, in a dev build
```

```bash
cargo install tokio-console
cargo run            # the service
tokio-console        # in another terminal
```

It adds overhead and needs an unstable cfg flag, so keep it out of production builds.

Tokio also has a cheap, always-available warning: in a profiling or test build, `tokio::runtime::Builder` can be combined with a watchdog that logs when a poll takes longer than, say, 10 ms. A slow poll means something blocked the thread.

---

## 9. Tracing spans across async code

Stage 7 used `tracing` for logs. Its real strength is **spans**: named, timed regions with fields, nested into a tree per request:

```text
http_request{method=GET route=/items/{id} request_id=7f3a} 48ms
├── auth{user_id=12} 0.3ms
├── load_item{item_id=42} 45ms
│   ├── cache_get{hit=false} 0.1ms
│   └── db_query{table=items} 44ms      ← here's the slow part
└── serialize 0.4ms
```

Adding spans:

```rust
use tracing::{info, instrument};

#[instrument(skip(state), fields(cache_hit))]
async fn load_item(state: &AppState, item_id: i64) -> Result<Item, AppError> {
    if let Some(item) = state.cache.get(&item_id).await {
        tracing::Span::current().record("cache_hit", true);
        return Ok(item);
    }
    tracing::Span::current().record("cache_hit", false);
    let item = db::items::get_by_id(&state.pool, item_id).await?;
    info!("loaded from database");
    Ok(item)
}
```

- `#[instrument]` creates a span per call, with the function's arguments as fields.
- **`skip(...)`** large or sensitive arguments (state, pools, request bodies, passwords). Use `skip_all` and list the fields you want, which is the safer default.
- `fields(name)` declares a field to fill in later with `record`.
- `tower_http::trace::TraceLayer` (Stage 7) creates the top-level `http_request` span.

**Spans and spawned tasks.** A span follows a future through `.await` automatically, but **not** into `tokio::spawn`. A spawned task starts with no parent span unless you pass it:

```rust
use tracing::Instrument;

tokio::spawn(
    async move { run_export(state, id).await }
        .instrument(tracing::info_span!("export", export_id = %id)),
);

// or keep the current request's span as the parent:
tokio::spawn(work().in_current_span());
```

Never hold a `span.enter()` guard across an `.await`. Use `.instrument(span)` on the future instead, otherwise the span is attached to whatever else runs on that thread.

**Sampling:** recording every span for every request is expensive at high traffic. Production setups record all errors and slow requests, and a percentage of the rest (section 11).

---

## 10. Metrics: counters, gauges, histograms

Spans describe **individual requests**. Metrics describe **the whole service over time**, cheaply: a few numbers updated in memory and collected every 15–60 seconds.

| Type | Behaviour | Examples |
|---|---|---|
| **Counter** | Only goes up | Requests served, errors, rows processed, cache hits |
| **Gauge** | Goes up and down | In-flight requests, queue depth, pool connections in use |
| **Histogram** | Distribution of values in buckets | Request duration, payload size. Gives p50/p90/p99 |

The `metrics` crate is a facade (like `log` or `tracing`), with exporters for different backends. Prometheus is the common one: your service exposes `GET /metrics` as text, and Prometheus scrapes it.

```toml
metrics = "0.24"
metrics-exporter-prometheus = { version = "0.16", default-features = false }
```

```rust
use axum::{extract::{MatchedPath, Request}, middleware::Next, response::Response};
use std::time::Instant;

pub async fn track_metrics(req: Request, next: Next) -> Response {
    let route = req
        .extensions()
        .get::<MatchedPath>()
        .map(|p| p.as_str().to_owned())
        .unwrap_or_else(|| "unmatched".into());
    let method = req.method().to_string();
    let start = Instant::now();

    let response = next.run(req).await;

    let status = response.status().as_u16().to_string();
    let labels = [("method", method), ("route", route), ("status", status)];
    metrics::counter!("http_requests_total", &labels).increment(1);
    metrics::histogram!("http_request_duration_seconds", &labels).record(start.elapsed().as_secs_f64());
    response
}
```

**Labels and cardinality** are the most important rule:

| Good label | Bad label |
|---|---|
| `route="/items/{id}"` | `path="/items/123"` |
| `status="200"` | `user_id="…"` |
| `method="GET"` | `request_id="…"` |

Every distinct label combination is a separate time series stored forever. A label with unbounded values (IDs, raw paths, error messages) can exhaust the metrics backend. Use `MatchedPath`, which is the route template, never the raw URI.

**The RED method** covers most services: **R**ate (requests/sec), **E**rrors (failed requests/sec), **D**uration (latency histogram). Add a few **saturation** gauges: pool connections in use, queue depth, in-flight requests.

> **JS comparison:** the same as `prom-client` in Node: counters, gauges, histograms, and a `/metrics` endpoint.

---

## 11. OpenTelemetry in one page

**OpenTelemetry (OTel)** is the vendor-neutral standard for traces, metrics, and logs. Instead of writing to one product's API, the service sends OTel data, and you choose the backend (Jaeger, Grafana Tempo, Honeycomb, Datadog, Azure Monitor) by configuration.

```text
 service ── tracing spans ──▶ tracing-opentelemetry ──▶ OTLP exporter ──▶ collector ──▶ backend
                                                                      (batching, sampling,
                                                                       routing)
```

| Concept | Meaning |
|---|---|
| **Trace** | All spans of one request, across **all services** it touched |
| **Trace context propagation** | The `traceparent` HTTP header carries the trace ID between services, so spans from the API, a worker, and the database client line up in one view |
| **OTLP** | The OTel wire protocol |
| **Collector** | A separate process that receives, batches, samples, and forwards data |
| **Sampling** | Keep 100% of errors and slow traces, a fraction of the rest |

In Rust you keep writing `tracing` spans exactly as in section 9. The `tracing-opentelemetry` layer converts them, so the application code doesn't change when you switch backends.

For a job system (Stage 16), propagate the trace context **inside the job payload** as well as in HTTP headers. Then a trace can show "user submitted job → worker claimed it → pipeline stages", even though it crossed a queue and several processes.

---

## 12. Caching

A cache trades memory and freshness for speed. Measure first: cache only what's both **expensive** and **read much more often than it changes**.

`moka` is a fast, concurrent, bounded in-memory cache:

```toml
moka = { version = "0.12", features = ["future"] }
```

```rust
use moka::future::Cache;
use std::time::Duration;

let cache: Cache<i64, Arc<Item>> = Cache::builder()
    .max_capacity(10_000)                    // bounded: oldest/least-used entries are evicted
    .time_to_live(Duration::from_secs(60))   // freshness limit
    .build();

let item = cache
    .try_get_with(id, async { load_from_db(&pool, id).await.map(Arc::new) })
    .await
    .map_err(|e| AppError::Internal(e.to_string().into()))?;
```

`try_get_with` also solves the **thundering herd**: if 100 requests miss the same key at once, only **one** loads it, and the others wait for that result instead of sending 100 identical queries.

| Decision | Options |
|---|---|
| Bound | Always set `max_capacity` (or a size-based weigher). An unbounded cache is a memory leak with a hit rate |
| Freshness | `time_to_live` for "stale for at most N seconds is fine". Explicit `invalidate` on writes for "must be fresh" |
| Values | Store `Arc<T>` so a hit is a pointer clone, not a deep copy |
| Multi-tenant | Include `tenant_id` in the key. A shared key across tenants leaks data |
| Many replicas | Each pod has its own in-memory cache. For a shared one, use Redis, and accept the network hop |

Measure the effect with metrics: `cache_hits_total` and `cache_misses_total` counters, plus the latency histogram before and after.

---

## 13. Common wins in Rust services

Ordered roughly by how often they matter:

| Area | Problem | Fix |
|---|---|---|
| Database | N+1 queries (one query per item in a list) | One query with `IN`/`ANY($1)` or a join |
| Database | Missing index | `EXPLAIN ANALYZE`, add the index (Stage 6) |
| Database | Pool too small, requests wait for a connection | Size the pool for the load, and watch "connections in use" |
| Database | Pool too large, the database is overloaded | More connections isn't faster past a point. Benchmark it |
| Blocking | CPU work or sync I/O on a runtime thread | `spawn_blocking` (Stage 1, Stage 15) |
| Allocation | `clone()` of large values, `to_string()` in hot loops | Borrow, `Arc`, `Bytes`, `Cow` |
| Allocation | New buffer per item | Reuse buffers (section 3) |
| Serialization | Large JSON responses built in memory | Stream them (Stage 10) |
| Locks | One `Mutex` around shared state on every request | Shorter critical sections, `RwLock`, sharding, or `moka` |
| HTTP | No compression for large responses | `tower_http::compression` |
| HTTP client | A new `reqwest::Client` per call | Build one and share it: it pools connections |
| Build | Default release profile | Try `lto = "thin"` and `codegen-units = 1`, then measure |

Every one of these should be **found by measurement** (a profile, a span, a metric), not guessed.

---

## 14. How this looks in a production service

```text
crates/api/
├── src/
│   ├── telemetry.rs        # tracing subscriber: JSON logs + optional OTel layer
│   ├── metrics.rs          # Prometheus recorder, /metrics route, track_metrics middleware
│   └── cache.rs            # moka caches with capacity, TTL, tenant-scoped keys
├── benches/
│   └── serialize.rs        # criterion benchmarks for hot paths
└── tests/
    └── memory.rs           # dhat: peak heap stays bounded at 10x input
loadtest/
├── items.js                # k6 scenario
└── RESULTS.md              # setup + p50/p90/p99 at each rate, by date and commit
```

Conventions:

- **Percentiles, not averages**, with the load level stated.
- **Release builds** for every measurement. `profiling` profile with debug symbols for profilers.
- **One change at a time**, measured before and after, with criterion baselines for hot functions.
- **Spans on every handler and every spawned task**, with `skip_all` and explicit fields.
- **RED metrics per route** using `MatchedPath`, plus saturation gauges. **No unbounded labels.**
- **Every cache is bounded**, has a freshness rule, and has hit/miss counters.
- **Memory bounds are tested**, not just designed.
- `/metrics`, `/healthz`, `/readyz` are excluded from auth and request logging.
- Load test results are written down with their setup and commit, so regressions can be spotted.

---

## 15. Putting it all together

A small service with:

- `GET /items/{id}`: a "slow lookup" (20 ms, standing in for a database query), **without** a cache,
- `GET /cached/items/{id}`: the same lookup through a bounded `moka` cache,
- tracing spans on handlers and the lookup,
- RED metrics per route and a cache hit/miss counter, exposed at `GET /metrics`.

Load-test both endpoints and compare percentiles, then read the metrics.

```toml
[dependencies]
axum = "0.8"
metrics = "0.24"
metrics-exporter-prometheus = { version = "0.16", default-features = false }
moka = { version = "0.12", features = ["future"] }
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

```rust
use std::sync::Arc;
use std::time::{Duration, Instant};

use axum::{
    extract::{MatchedPath, Path, Request, State},
    http::StatusCode,
    middleware::{self, Next},
    response::{IntoResponse, Response},
    routing::get,
    Json, Router,
};
use metrics_exporter_prometheus::{PrometheusBuilder, PrometheusHandle};
use moka::future::Cache;
use serde::Serialize;
use tracing::instrument;

const LOOKUP_DELAY: Duration = Duration::from_millis(20);

#[derive(Clone, Serialize)]
struct Item {
    id: u64,
    name: String,
}

#[derive(Clone)]
struct AppState {
    cache: Cache<u64, Arc<Item>>,
    metrics: PrometheusHandle,
}

// ---------- the "slow" dependency ----------

#[instrument]
async fn slow_lookup(id: u64) -> Result<Item, String> {
    tokio::time::sleep(LOOKUP_DELAY).await; // stands in for a database query
    if id == 0 {
        return Err("item 0 does not exist".into());
    }
    Ok(Item { id, name: format!("item-{id}") })
}

// ---------- handlers ----------

#[instrument(skip_all, fields(item_id = id))]
async fn get_item(Path(id): Path<u64>) -> Response {
    match slow_lookup(id).await {
        Ok(item) => Json(item).into_response(),
        Err(_) => StatusCode::NOT_FOUND.into_response(),
    }
}

#[instrument(skip_all, fields(item_id = id, cache_hit))]
async fn get_item_cached(State(s): State<AppState>, Path(id): Path<u64>) -> Response {
    let hit = s.cache.contains_key(&id);
    tracing::Span::current().record("cache_hit", hit);
    metrics::counter!("cache_requests_total", "result" => if hit { "hit" } else { "miss" }).increment(1);

    // try_get_with: only one lookup per key even when many requests miss at once
    let result = s
        .cache
        .try_get_with(id, async move { slow_lookup(id).await.map(Arc::new) })
        .await;

    match result {
        Ok(item) => Json(item.as_ref().clone()).into_response(),
        Err(_) => StatusCode::NOT_FOUND.into_response(),
    }
}

async fn render_metrics(State(s): State<AppState>) -> String {
    s.metrics.render()
}

// ---------- RED metrics middleware ----------

async fn track_metrics(req: Request, next: Next) -> Response {
    let route = req
        .extensions()
        .get::<MatchedPath>()
        .map(|p| p.as_str().to_owned())
        .unwrap_or_else(|| "unmatched".into());
    let method = req.method().to_string();
    let start = Instant::now();

    let response = next.run(req).await;

    let labels = [
        ("method", method),
        ("route", route),
        ("status", response.status().as_u16().to_string()),
    ];
    metrics::counter!("http_requests_total", &labels).increment(1);
    metrics::histogram!("http_request_duration_seconds", &labels).record(start.elapsed().as_secs_f64());
    response
}

// ---------- main ----------

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| "info".into()),
        )
        .init();

    let metrics = PrometheusBuilder::new().install_recorder()?;

    let state = AppState {
        cache: Cache::builder()
            .max_capacity(10_000)
            .time_to_live(Duration::from_secs(60))
            .build(),
        metrics,
    };

    let app = Router::new()
        .route("/items/{id}", get(get_item))
        .route("/cached/items/{id}", get(get_item_cached))
        .route_layer(middleware::from_fn(track_metrics)) // route_layer: MatchedPath is available
        .route("/metrics", get(render_metrics))          // added after: not measured itself
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await?;
    tracing::info!("listening on 0.0.0.0:8080");
    axum::serve(listener, app).await?;
    Ok(())
}
```

Run it and load-test it:

```bash
cargo run --release

# in another terminal
oha -z 15s -q 1000 -c 100 http://localhost:8080/items/1
oha -z 15s -q 1000 -c 100 http://localhost:8080/cached/items/1

curl -s localhost:8080/metrics | grep -E 'cache_requests_total|http_requests_total'
```

Expected output, shortened. Your numbers will differ, but the shape won't:

```text
/items/1          Requests/sec ≈ 1000   p50 ≈ 21 ms   p90 ≈ 22 ms   p99 ≈ 25 ms
/cached/items/1   Requests/sec ≈ 1000   p50 ≈ 0.2 ms  p90 ≈ 0.4 ms  p99 ≈ 1 ms

cache_requests_total{result="hit"} 14999
cache_requests_total{result="miss"} 1
http_requests_total{method="GET",route="/items/{id}",status="200"} 15000
http_requests_total{method="GET",route="/cached/items/{id}",status="200"} 15000
```

What each part shows:

| Part | Concept |
|---|---|
| Fixed-rate (`-q 1000`) load test | **Open model**: latency at a stated load |
| p50/p90/p99 for both endpoints | **Percentiles** and the effect of one change |
| One `miss`, the rest `hit` | **Bounded cache** with `try_get_with` |
| `route="/items/{id}"`, not `/items/1` | **Low-cardinality labels** with `MatchedPath` |
| `#[instrument]` spans with `item_id` and `cache_hit` | **Spans with fields** for per-request detail |
| `/metrics` added after `route_layer` | Observability endpoints aren't measured themselves |

Try breaking it on purpose:

- Load-test with random IDs (k6, or a list of URLs) so most requests miss. Watch the hit ratio and p99 change. Then lower `max_capacity` to 100.
- Replace `tokio::time::sleep` in `slow_lookup` with `std::thread::sleep`. Throughput collapses and p99 explodes: blocking on a runtime thread. Find it with `tokio-console`.
- Use `req.uri().path()` instead of `MatchedPath` as the route label, load-test with random IDs, and count the lines in `/metrics`.
- Profile the cached endpoint with `samply` under load. Most of the remaining time is HTTP and JSON. Then try returning the cached `Arc<Item>` serialized once into `Bytes` and stored in the cache.

---

## 16. Exercises

- [ ] **1. Percentiles by hand:** write a function that takes a `Vec<Duration>` and returns p50, p90, p99, and max. Feed it 99 × 10 ms and 1 × 2 s, and compare the average with p99.
- [ ] **2. Criterion:** add the section 3 benchmark. Save a baseline, then try `Vec::with_capacity` with a better size estimate, and report whether criterion detects a change.
- [ ] **3. Find the knee:** load-test `/items/1` with an open model at 250, 500, 1,000, 2,000, and 4,000 rps. Plot or tabulate p50 and p99. Explain where and why latency starts rising (hint: the uncached endpoint holds each request for 20 ms).
- [ ] **4. Flamegraph:** build with the `profiling` profile, run `samply` while load-testing `/cached/items/1`, and name the three widest functions you can change.
- [ ] **5. Memory test:** with `dhat`, write a test that runs the Stage 10 export for 10,000 and 100,000 rows and asserts that peak heap differs by less than 20%.
- [ ] **6. Blocking detection:** introduce `std::thread::sleep` in a handler, find it with `tokio-console`, fix it, and record before/after p99.
- [ ] **7. Spans across tasks:** spawn a background task from a handler. Show with log output that without `.in_current_span()` its events lose the request's `item_id` field, and with it they keep it.
- [ ] **8. Saturation metrics:** add gauges for in-flight requests (increment and decrement in the middleware, including on errors) and for SQLx pool connections in use (`pool.size()` and `pool.num_idle()`) sampled every few seconds.
- [ ] **9. OpenTelemetry:** run Jaeger locally (`docker run -p 16686:16686 -p 4317:4317 jaegertracing/all-in-one`), add `tracing-opentelemetry` with an OTLP exporter, and find one request's span tree in the Jaeger UI.
- [ ] **10. Explain it:** in four or five sentences, explain why averages hide problems, what coordinated omission is, why async waiting doesn't show up in a CPU flamegraph, and why metric labels must have bounded values.

> When you can solve exercises 1–10 without looking anything up, you've finished the core path. Revisit **Stage 12: Reference Service** and apply Stages 14–18 to it.

---

## 17. Further reading

- [`criterion` user guide](https://bheisler.github.io/criterion.rs/book/)
- [The Rust Performance Book](https://nnethercote.github.io/perf-book/): profiling, allocation, build configuration
- [`oha`](https://github.com/hatoo/oha) and [k6](https://k6.io/docs/): load testing
- [Gil Tene: "How NOT to Measure Latency"](https://www.youtube.com/watch?v=lJ8ydIuPFeU): percentiles and coordinated omission
- [`cargo-flamegraph`](https://github.com/flamegraph-rs/flamegraph) and [`samply`](https://github.com/mstange/samply)
- [`dhat` crate](https://docs.rs/dhat): heap profiling and heap-usage tests
- [`tokio-console`](https://github.com/tokio-rs/console)
- [`tracing` docs: `instrument` and `Instrument`](https://docs.rs/tracing/latest/tracing/attr.instrument.html)
- [`metrics`](https://docs.rs/metrics) and [`metrics-exporter-prometheus`](https://docs.rs/metrics-exporter-prometheus)
- [Prometheus: naming and label best practices](https://prometheus.io/docs/practices/naming/)
- [OpenTelemetry Rust](https://opentelemetry.io/docs/languages/rust/) and [`tracing-opentelemetry`](https://docs.rs/tracing-opentelemetry)
- [`moka` cache](https://docs.rs/moka)
- [Tom Wilkie: "The RED Method"](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/)
