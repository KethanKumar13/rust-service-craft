# rust-service-craft

**Production-grade backend services in Rust: designed, built, and tested with the same discipline as real-world systems.**

Axum · Tokio · Tower · SQLx · PostgreSQL · serde · tracing · JWT

---

## About

`rust-service-craft` is an engineering workbench for building HTTP services in Rust.  
Each module isolates one concern of a production backend, implements it properly, and
documents the design decisions behind it. The modules then come together in a complete,
multi-tenant reference service.

The focus is not on "hello world" examples but on the patterns that matter in production:

- **Type-driven correctness:** newtype IDs, exhaustive error enums, compile-time-checked SQL
- **Layered architecture:** HTTP, domain logic, persistence, and shared types kept separate
- **Security by default:** cookie-based JWT auth, capability-based authorization, tenant isolation
- **Predictable performance:** streaming responses, bounded buffering, backpressure
- **Operational readiness:** structured tracing, typed configuration, graceful shutdown
- **Verified behaviour:** integration tests against a real PostgreSQL database

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

| Module | Concern | Key topics | Status |
|---|---|---|---|
| [`foundations`](docs/notes/00-rust-prerequisites.md) | Language core | Ownership, traits, generics, `Result`, serde, `Arc`, `Send`/`Sync` | 🟡 In progress |
| `async-runtime` | Concurrency | Tokio, tasks, channels, `select!`, async locks | ⬜ Planned |
| `routing` | HTTP surface | `Router`, handlers, nesting, merging, path params | ⬜ Planned |
| `extractors` | Request input | `Path`, `Query`, `Json`, custom `FromRequestParts` | ⬜ Planned |
| `errors` | Failure model | `thiserror`, `IntoResponse`, `From` conversions, consistent error bodies | ⬜ Planned |
| `state-and-config` | Shared resources | `AppState`, `Arc`, typed config from environment | ⬜ Planned |
| `persistence` | Data | SQLx, migrations, transactions, optimistic locking with `version` | ⬜ Planned |
| `middleware` | Cross-cutting concerns | Tower layers, tracing, CORS, timeouts, body limits | ⬜ Planned |
| `auth` | Identity and access | JWT in HttpOnly cookies, token revocation, capabilities, tenant scoping | ⬜ Planned |
| `realtime` | Live updates | WebSockets, in-memory presence | ⬜ Planned |
| `streaming` | Large payloads | NDJSON streaming, bounded channels, backpressure | ⬜ Planned |
| `testing` | Verification | `oneshot` handler tests, DB integration tests, `cargo nextest` | ⬜ Planned |
| **`reference-service`** | **Everything combined** | Multi-tenant REST service with all of the above | ⬜ Planned |

Each module has a README with its **goal**, **design decisions**, **how to run it**, and **trade-offs**.

---

## Repository layout

```text
rust-service-craft/
├── crates/                  # One crate per module
├── projects/
│   └── reference-service/   # Layered workspace: api / domain / db / shared
├── docs/
│   └── notes/               # Technical write-ups per module
├── .github/workflows/       # CI: fmt, clippy, tests
├── Cargo.toml               # Workspace root with shared dependency versions
└── rustfmt.toml
```

---

## Getting started

**Requirements:** Rust stable, Docker (for PostgreSQL in later modules), and `cargo-nextest`.

```bash
git clone https://github.com/KethanKumar13/rust-service-craft.git
cd rust-service-craft

cargo build --workspace
cargo nextest run --workspace        # or: cargo test --workspace
cargo run -p <module-name>
```

Quality checks (the same ones CI runs):

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
```

---

## Engineering conventions

These rules apply across the whole repository.

**Code**

- Rust 2021+, `snake_case` functions and modules, `PascalCase` types
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
- One focused branch and PR per module

---

## Current state

> Keep this section up to date. It is the hand-off point for picking up work in a new session.

- **Done:** repository set up, foundations notes (`docs/notes/00-rust-prerequisites.md`)
- **Now:** foundations exercises (ownership, enums, `Result`, traits, serde, `Arc`/`Mutex`)
- **Next:** workspace `Cargo.toml`, CI workflow, and the `async-runtime` module
- **Open questions:** none

---

## References

- [Axum documentation](https://docs.rs/axum) and the [official examples](https://github.com/tokio-rs/axum/tree/main/examples)
- [Tokio tutorial](https://tokio.rs/tokio/tutorial)
- [SQLx](https://github.com/launchbadge/sqlx)
- [The Rust Programming Language](https://doc.rust-lang.org/book/)
- *Zero To Production in Rust* by Luca Palmieri
