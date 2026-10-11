# rust-service-craft

**Production-style backend services in Rust: Axum, Tokio, SQLx, middleware, auth, testing, data pipelines, deployment, and observability, built stage by stage.**

Axum · Tokio · Tower · SQLx · PostgreSQL · serde · tracing · JWT · Arrow · DataFusion · Docker · Kubernetes · OpenTelemetry

---

## About

`rust-service-craft` is an engineering workbench for building HTTP services in Rust.
Each module isolates one concern of a production backend, explains the design decisions behind it,
and ends with a complete working example and a set of exercises. The modules then come together
in a multi-tenant reference service.

Stages 14–19 go beyond the HTTP layer: columnar data processing, multi-stage stream pipelines,
background jobs across processes, shipping to containers and Kubernetes, measuring performance,
and adding AI features safely.

The focus is on the patterns that matter in production:

- **Type-driven correctness:** newtype IDs, exhaustive error enums, compile-time-checked SQL
- **Layered architecture:** HTTP, domain logic, persistence, and shared types kept separate
- **Security by default:** cookie-based JWT auth, capability-based authorization, tenant isolation
- **Predictable performance:** streaming responses, bounded buffering, backpressure
- **Data at scale:** Arrow record batches, DataFusion queries, bounded parallel pipelines
- **Reliable background work:** leases, heartbeats, fencing, retries with backoff
- **Operational readiness:** structured tracing, metrics, typed configuration, health probes, graceful shutdown
- **Verified behaviour:** handler tests, integration tests against real PostgreSQL, streaming and memory-bound tests
- **AI with guardrails:** structured LLM output, validation, rate limits, and a human in the loop

---

## Architecture

The reference service is a Cargo workspace with a strict dependency direction:

```text
           ┌────────────┐
           │    api     │  Axum router, handlers, extractors, middleware, WebSockets
           └─────┬──────┘
                 │
           ┌─────▼──────┐
           │   domain   │  Business rules, auth, validation (no HTTP, no SQL)
           └─────┬──────┘
                 │
           ┌─────▼──────┐
           │     db     │  SQLx repositories: free functions, *Row structs
           └─────┬──────┘
                 │
           ┌─────▼──────┐
           │   shared   │  Config, AppError, request/response DTOs
           └────────────┘
```

| Layer | Responsibility | Must not |
|---|---|---|
| `api` | Parse requests, call domain, map to responses | Contain business rules or raw SQL |
| `domain` | Enforce rules, orchestrate repositories | Know about HTTP types |
| `db` | Query and persist, map rows | Make authorization decisions |
| `shared` | Types used across layers | Depend on any other layer |

---

## Modules

### Core service

| # | Module | Concern | Key topics |
|---|---|---|---|
| 00 | [`foundations`](docs/notes/00-rust-prerequisites.md) | Language core | Ownership, traits, generics, `Result`, serde, `Arc`, `Send`/`Sync` |
| 01 | [`async-runtime`](docs/notes/01-async-runtime.md) | Concurrency | Tokio, tasks, channels, `select!`, async locks |
| 02 | [`routing`](docs/notes/02-routing.md) | HTTP surface | `Router`, handlers, nesting, merging, path params |
| 03 | [`extractors`](docs/notes/03-extractors.md) | Request input | `Path`, `Query`, `Json`, custom `FromRequestParts`, validation |
| 04 | [`errors`](docs/notes/04-errors.md) | Failure model | `thiserror`, `IntoResponse`, `From` conversions, consistent error bodies |
| 05 | [`state-and-config`](docs/notes/05-state-and-config.md) | Shared resources | `AppState`, `Arc`, `FromRef`, typed config from environment |
| 06 | [`persistence`](docs/notes/06-persistence.md) | Data | SQLx, migrations, transactions, optimistic locking with `version` |
| 07 | [`middleware`](docs/notes/07-middleware.md) | Cross-cutting concerns | Tower layers, tracing, CORS, timeouts, body limits |
| 08 | [`auth`](docs/notes/08-auth.md) | Identity and access | Argon2, JWT in HttpOnly cookies, revocation, capabilities, tenant scoping |
| 09 | [`realtime`](docs/notes/09-realtime.md) | Live updates | WebSockets, in-memory presence, broadcast rooms |
| 10 | [`streaming`](docs/notes/10-streaming.md) | Large payloads | NDJSON streaming, bounded channels, backpressure |
| 11 | [`testing`](docs/notes/11-testing.md) | Verification | `oneshot` handler tests, DB integration tests, `cargo nextest` |
| 12 | [**`reference-service`**](docs/notes/12-reference-service.md) | **Everything combined** | Multi-tenant REST service: design, schema, API surface, milestones |
| 13 | [`dropshot`](docs/notes/13-dropshot.md) | A second framework | Endpoints, typed responses, OpenAPI-first design, comparison with Axum |

### Data, operations, and AI

| # | Module | Concern | Key topics |
|---|---|---|---|
| 14 | [`arrow-datafusion`](docs/notes/14-arrow-datafusion.md) | Columnar data | Arrow arrays, `RecordBatch`, kernels, DataFusion SQL, plans, `execute_stream`, memory limits, UDFs, Parquet |
| 15 | [`stream-pipelines`](docs/notes/15-stream-pipelines.md) | Multi-stage data flow | `Stream`/`Pin`, `async-stream`, `buffered`, `spawn_blocking`, `JoinSet`, `Semaphore`, `CancellationToken`, side outputs, pipeline tests |
| 16 | [`job-orchestration`](docs/notes/16-job-orchestration.md) | Background work across processes | Job state machine, `SKIP LOCKED` claims, leases, heartbeats, fencing, retries with backoff, OAuth2 client credentials |
| 17 | [`deployment`](docs/notes/17-deployment.md) | Shipping | Multi-stage Docker, `cargo-chef`, distroless, health probes, `SIGTERM`, Kubernetes Deployments/Jobs, rolling updates, GitHub Actions CI |
| 18 | [`performance-observability`](docs/notes/18-performance-observability.md) | Measuring and seeing | Percentiles, `criterion`, load testing, flamegraphs, `dhat`, `tokio-console`, tracing spans, Prometheus metrics, OpenTelemetry, `moka` caching |
| 19 | [`llm-and-ml`](docs/notes/19-llm-and-ml.md) | AI features | `reqwest` LLM calls, structured output, validation, retries with backoff, rate limiting, features, embeddings, logistic regression |

Each module covers the concepts, shows how they look in a production service,
and ends with a complete runnable example and exercises.

---

## Engineering conventions

These rules apply across the whole repository.

**Code**

- Rust 2021, `snake_case` functions and modules, `PascalCase` types
- No `unwrap()` / `expect()` in runtime code: use `?`, `ok_or_else`, `map_err`
- `thiserror` for library and domain errors, `anyhow` only at binary entry points
- Prefer `#[expect(lint, reason = "...")]` over a broad `#[allow]`
- Newtype IDs (`ProjectId(i32)`) instead of raw integers
- Never hold a `std::sync` lock across `.await`

**API design**

- One `AppError` enum implementing `IntoResponse` (`NotFound`, `BadRequest`, `Unauthorized`, `Forbidden`, `Conflict`, `Internal`)
- DTOs live in `shared`; database rows (`*Row`) never leave `db`
- `409 Conflict` for optimistic-locking version mismatches and stale job leases
- Every tenant-owned query is filtered by `tenant_id`

**Performance**

- Stream large results instead of collecting them into memory
- Bounded channels and buffers only; every pipeline stage has a fixed limit on work in progress
- Clone `Arc` handles, not the data they point to
- Measure before optimizing: percentiles under a stated load, release builds only

**Background work**

- Every job state change is one guarded `UPDATE`; zero rows changed means someone else owns it
- Leases, heartbeats, and fencing tokens for anything that runs outside the request
- Job handlers are idempotent, because delivery is at-least-once

**Operations**

- Config from the environment, validated at startup; JSON logs to stdout
- Liveness checks the process only; readiness checks dependencies and draining
- `SIGTERM` triggers graceful shutdown; images run as non-root with immutable tags
- Metric labels have bounded values (route templates, never raw paths or IDs)

**External AI calls**

- Timeouts, retries with backoff and jitter, and a shared rate limiter on every call
- Structured output, validated against known values before use
- Send metadata, not customer data; a human confirms suggestions

**Testing**

- Tests are named by behaviour, e.g. `rejects_expired_token`, `returns_409_on_stale_version`
- Integration tests use a dedicated test database
- Streaming code is tested for early output, bounded residency, and cancellation
- `cargo nextest` is the test runner

**Git**

- Conventional Commits: `feat(auth): add capability check`, `test(db): cover version conflict`
- One focused branch and PR per change

---

## References

- [Axum documentation](https://docs.rs/axum) and the [official examples](https://github.com/tokio-rs/axum/tree/main/examples)
- [Tokio tutorial](https://tokio.rs/tokio/tutorial)
- [SQLx](https://github.com/launchbadge/sqlx)
- [The Rust Programming Language](https://doc.rust-lang.org/book/)
- [Apache Arrow](https://arrow.apache.org/docs/format/Columnar.html) and [DataFusion](https://datafusion.apache.org/)
- [Kubernetes documentation](https://kubernetes.io/docs/home/)
- [The Rust Performance Book](https://nnethercote.github.io/perf-book/)
- [OpenTelemetry for Rust](https://opentelemetry.io/docs/languages/rust/)
