# rust-service-craft

**Production-style backend services in Rust: Axum, Tokio, SQLx, middleware, auth, and testing, built stage by stage.**

Axum · Tokio · Tower · SQLx · PostgreSQL · serde · tracing · JWT

---

## About

`rust-service-craft` is an engineering workbench for building HTTP services in Rust.
Each module isolates one concern of a production backend, explains the design decisions behind it,
and ends with a complete working example and a set of exercises. The modules then come together
in a multi-tenant reference service.

The focus is on the patterns that matter in production:

- **Type-driven correctness:** newtype IDs, exhaustive error enums, compile-time-checked SQL
- **Layered architecture:** HTTP, domain logic, persistence, and shared types kept separate
- **Security by default:** cookie-based JWT auth, capability-based authorization, tenant isolation
- **Predictable performance:** streaming responses, bounded buffering, backpressure
- **Operational readiness:** structured tracing, typed configuration, graceful shutdown
- **Verified behaviour:** handler tests and integration tests against a real PostgreSQL database

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
- `409 Conflict` for optimistic-locking version mismatches
- Every tenant-owned query is filtered by `tenant_id`

**Performance**

- Stream large results instead of collecting them into memory
- Bounded channels and buffers only
- Clone `Arc` handles, not the data they point to

**Testing**

- Tests are named by behaviour, e.g. `rejects_expired_token`, `returns_409_on_stale_version`
- Integration tests use a dedicated test database
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
