# Stage 11: Testing

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> If you come from JavaScript or TypeScript, the comparisons below map Rust concepts to familiar JS ideas (Jest/Vitest and Supertest).

## Contents

1. [What to test in a backend service](#1-what-to-test-in-a-backend-service)
2. [Rust test basics](#2-rust-test-basics)
3. [Async tests with `#[tokio::test]`](#3-async-tests-with-tokiotest)
4. [Structuring the app so it can be tested](#4-structuring-the-app-so-it-can-be-tested)
5. [Handler tests with `oneshot`](#5-handler-tests-with-oneshot)
6. [Testing errors, auth, and tenant isolation](#6-testing-errors-auth-and-tenant-isolation)
7. [Database integration tests](#7-database-integration-tests)
8. [`cargo nextest`](#8-cargo-nextest)
9. [Controlling time](#9-controlling-time)
10. [Testing streaming and WebSockets](#10-testing-streaming-and-websockets)
11. [Test naming and conventions](#11-test-naming-and-conventions)
12. [Running tests in CI](#12-running-tests-in-ci)
13. [Putting it all together](#13-putting-it-all-together)
14. [Exercises](#14-exercises)
15. [Further reading](#15-further-reading)

---

## 1. What to test in a backend service

| Layer | Test type | Speed | What it proves |
|---|---|---|---|
| Pure logic (`domain`, validation, config parsing) | Unit tests | Microseconds | Rules are correct for every edge case |
| HTTP surface (`api`) | Handler tests with `oneshot` | Milliseconds | Routing, extractors, status codes, JSON shapes, auth |
| Persistence (`db`) | Integration tests against real PostgreSQL | 10–100 ms | SQL is valid, constraints work, tenant scoping, optimistic locking |
| Whole service | A few end-to-end tests over a real socket | Slower | Startup, middleware stack, WebSockets |

Most tests should be in the first two rows. Database tests catch bugs that nothing else can, so every repository function needs some. End-to-end tests are few and focused.

> **JS comparison:** unit tests ≈ Jest/Vitest tests of plain functions; `oneshot` handler tests ≈ Supertest's `request(app).get("/...")`, but without opening a port; database tests ≈ tests against a Docker Postgres with a reset between tests.

---

## 2. Rust test basics

### 2.1 Unit tests live next to the code

```rust
// src/domain/projects.rs
pub fn validate_name(name: &str) -> Result<&str, String> {
    let name = name.trim();
    if name.is_empty() {
        return Err("name must not be empty".into());
    }
    if name.len() > 100 {
        return Err("name must be at most 100 characters".into());
    }
    Ok(name)
}

#[cfg(test)]                 // compiled only for `cargo test`
mod tests {
    use super::*;            // the private items of the parent module are visible

    #[test]
    fn trims_surrounding_whitespace() {
        assert_eq!(validate_name("  Alpha  "), Ok("Alpha"));
    }

    #[test]
    fn rejects_blank_name() {
        assert!(validate_name("   ").is_err());
    }

    #[test]
    fn rejects_name_over_100_chars() {
        let long = "x".repeat(101);
        assert_eq!(validate_name(&long), Err("name must be at most 100 characters".into()));
    }
}
```

| Macro | Checks |
|---|---|
| `assert!(cond)` | `cond` is true |
| `assert_eq!(a, b)` / `assert_ne!` | Equality; prints both values on failure (needs `Debug` + `PartialEq`) |
| `assert!(matches!(value, Pattern))` | `value` matches an enum pattern |

Tests can also return `Result` so you can use `?`:

```rust
#[test]
fn parses_port() -> Result<(), ConfigError> {
    let config = Config::from_lookup(lookup(&[("API_KEY", "0123456789abcdef"), ("PORT", "9000")]))?;
    assert_eq!(config.port, 9000);
    Ok(())
}
```

### 2.2 Integration tests live in `tests/`

```text
crates/api/
├── src/lib.rs          # pub fn app(state) -> Router
└── tests/
    ├── common/mod.rs   # shared helpers (not a test binary itself)
    ├── projects.rs     # one test binary
    └── auth.rs         # another test binary
```

Each file in `tests/` is compiled as a **separate crate** that can only use the **public** API, the same as a real user of the library. That's why the app should be a library (section 4).

### 2.3 `unwrap` in tests

The convention bans `unwrap()`/`expect()` in **runtime** code. Tests are different: a panic is exactly how a test fails. Allow it for tests only with `clippy.toml`:

```toml name=clippy.toml
allow-unwrap-in-tests = true
allow-expect-in-tests = true
```

Prefer `expect("what was expected")` over `unwrap()`, so the failure message explains what went wrong.

---

## 3. Async tests with `#[tokio::test]`

```rust
#[tokio::test]
async fn fetches_project() {
    let result = load_project(1).await;
    assert!(result.is_ok());
}
```

- `#[tokio::test]` builds a **single-threaded** runtime per test (the `current_thread` flavor).
- Use `#[tokio::test(flavor = "multi_thread", worker_threads = 2)]` when the test needs real parallelism, for example to show that blocking code freezes other tasks (Stage 1).
- Each test has its own runtime, so a test can't leak tasks into another.

---

## 4. Structuring the app so it can be tested

The key decision: **`main` does almost nothing**. Everything that builds the app lives in a library function that tests can call.

```rust
// src/lib.rs
pub fn app(state: AppState) -> Router {
    routes::router(state)       // routes, middleware, fallback, with_state
}

// src/main.rs
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let config = Config::from_env()?;
    let state = AppState::build(&config).await?;     // pool, migrations, jwt keys
    let listener = tokio::net::TcpListener::bind(("0.0.0.0", config.port)).await?;
    axum::serve(listener, my_service::app(state)).await?;
    Ok(())
}
```

Then a test builds its own `AppState`:

- a **test database pool** (section 7) or an in-memory store
- a **config** from `Config::from_lookup` over a `HashMap` (Stage 5), never from real environment variables
- **known secrets**, so tests can issue valid tokens (section 6.2)

Because state is explicit (Stage 5, section 3.3), many tests can each build their own app **in parallel** without sharing globals.

---

## 5. Handler tests with `oneshot`

Stage 2 introduced calling a router without a network. In tests, wrap it in a small helper so every test reads like a sentence.

### 5.1 A test client

```rust
// tests/common/mod.rs
use axum::{
    body::{to_bytes, Body},
    http::{header, Method, Request, StatusCode},
    Router,
};
use serde_json::Value;
use tower::ServiceExt;

pub struct TestResponse {
    pub status: StatusCode,
    pub headers: axum::http::HeaderMap,
    pub body: Vec<u8>,
}

impl TestResponse {
    pub fn json(&self) -> Value {
        serde_json::from_slice(&self.body).expect("response body is JSON")
    }
}

pub struct TestApp {
    router: Router,
    token: Option<String>,
}

impl TestApp {
    pub fn new(router: Router) -> Self {
        Self { router, token: None }
    }

    pub fn as_user(mut self, token: &str) -> Self {
        self.token = Some(token.to_owned());
        self
    }

    pub async fn request(&self, method: Method, uri: &str, json: Option<Value>) -> TestResponse {
        let mut builder = Request::builder().method(method).uri(uri);
        if let Some(t) = &self.token {
            builder = builder.header(header::AUTHORIZATION, format!("Bearer {t}"));
        }
        let body = match json {
            Some(v) => {
                builder = builder.header(header::CONTENT_TYPE, "application/json");
                Body::from(v.to_string())
            }
            None => Body::empty(),
        };

        let response = self
            .router
            .clone()
            .oneshot(builder.body(body).expect("valid request"))
            .await
            .expect("router is infallible");

        let status = response.status();
        let headers = response.headers().clone();
        let body = to_bytes(response.into_body(), usize::MAX).await.expect("readable body").to_vec();
        TestResponse { status, headers, body }
    }

    pub async fn get(&self, uri: &str) -> TestResponse {
        self.request(Method::GET, uri, None).await
    }

    pub async fn post(&self, uri: &str, json: Value) -> TestResponse {
        self.request(Method::POST, uri, Some(json)).await
    }

    pub async fn put(&self, uri: &str, json: Value) -> TestResponse {
        self.request(Method::PUT, uri, Some(json)).await
    }
}
```

### 5.2 A handler test

```rust
// tests/projects.rs
mod common;

use axum::http::StatusCode;
use common::TestApp;
use serde_json::json;

#[tokio::test]
async fn creates_project_and_returns_201() {
    let app = TestApp::new(my_service::app(test_state())).as_user("user-1-tenant-1");

    let res = app.post("/projects", json!({ "name": "Alpha" })).await;

    assert_eq!(res.status, StatusCode::CREATED);
    assert_eq!(res.json()["name"], "Alpha");
    assert_eq!(res.json()["version"], 1);
}
```

Use `serde_json::json!` for request bodies and compare responses as `serde_json::Value`. That checks the real wire format, including field names and `rename_all`. Deserializing into your own DTO in the test would hide a renamed field.

---

## 6. Testing errors, auth, and tenant isolation

These tests protect the parts that matter most, and they're cheap with `oneshot`.

### 6.1 Every `AppError` variant has a test

```rust
#[tokio::test]
async fn returns_404_with_error_body_for_unknown_project() {
    let app = TestApp::new(my_service::app(test_state())).as_user("user-1-tenant-1");

    let res = app.get("/projects/999").await;

    assert_eq!(res.status, StatusCode::NOT_FOUND);
    assert_eq!(res.json(), json!({ "error": "not_found", "message": "project 999 not found" }));
}

#[tokio::test]
async fn returns_409_on_stale_version() {
    let app = TestApp::new(my_service::app(test_state())).as_user("user-1-tenant-1");
    let id = app.post("/projects", json!({ "name": "Alpha" })).await.json()["id"].clone();

    let first = app.put(&format!("/projects/{id}"), json!({ "name": "B", "version": 1 })).await;
    let second = app.put(&format!("/projects/{id}"), json!({ "name": "C", "version": 1 })).await;

    assert_eq!(first.status, StatusCode::OK);
    assert_eq!(second.status, StatusCode::CONFLICT);
    assert_eq!(second.json()["error"], "conflict");
}
```

Also assert that **500s never leak details**: the body must be exactly the generic message (Stage 4, section 9).

### 6.2 Authentication

With real JWTs (Stage 8), tests need valid tokens. Because the test builds `AppState` with a **known secret**, a helper can issue tokens with the same `JwtKeys`:

```rust
pub fn token_for(keys: &JwtKeys, user_id: i32, tenant_id: i32, role: Role) -> String {
    let user = UserRow { id: user_id, tenant_id, role, is_super_admin: false, /* ... */ };
    keys.issue(&user).expect("token issued").0
}
```

Cover each way authentication can fail:

| Test | Expected |
|---|---|
| `rejects_request_without_token` | 401 |
| `rejects_tampered_token` | 401 |
| `rejects_expired_token` (issue with `ttl = 0`, leeway 0) | 401 |
| `rejects_revoked_token_after_logout` | 401 |
| `rejects_viewer_creating_project` | 403 |

### 6.3 Tenant isolation

This is the most important test category in a multi-tenant service. For every resource, prove that another tenant **can't read, update, or delete** it:

```rust
#[tokio::test]
async fn hides_projects_from_other_tenants() {
    let state = test_state();
    let tenant_1 = TestApp::new(my_service::app(state.clone())).as_user("user-1-tenant-1");
    let tenant_2 = TestApp::new(my_service::app(state)).as_user("user-2-tenant-2");

    let id = tenant_1.post("/projects", json!({ "name": "Secret" })).await.json()["id"].clone();

    assert_eq!(tenant_2.get(&format!("/projects/{id}")).await.status, StatusCode::NOT_FOUND);
    assert_eq!(
        tenant_2.put(&format!("/projects/{id}"), json!({ "name": "Hacked", "version": 1 })).await.status,
        StatusCode::NOT_FOUND
    );
    // and the data didn't change:
    assert_eq!(tenant_1.get(&format!("/projects/{id}")).await.json()["name"], "Secret");
}
```

Note the last assertion: checking the status code isn't enough. A buggy `UPDATE` without `tenant_id` could change the row and **then** return 404.

---

## 7. Database integration tests

Mocking the database hides exactly the bugs these tests should find: SQL typos, missing `tenant_id` filters, constraint behaviour, and the version check. Test against **real PostgreSQL**.

### 7.1 A test database

```bash
docker compose up -d postgres
export DATABASE_URL=postgres://app:app@localhost:5432/app_test
sqlx database create
```

Keep it separate from the development database: tests create and delete data freely.

### 7.2 `#[sqlx::test]`: a fresh database per test

```rust
use sqlx::PgPool;

#[sqlx::test(migrations = "./migrations")]
async fn insert_then_get_round_trips(pool: PgPool) -> sqlx::Result<()> {
    let tenant = TenantId(1);
    let row = db::projects::insert(&pool, tenant, "Alpha").await?;

    let loaded = db::projects::get_by_id(&pool, tenant, row.id).await?;

    assert_eq!(loaded.map(|r| r.name), Some("Alpha".to_owned()));
    Ok(())
}
```

`#[sqlx::test]`:

- connects using `DATABASE_URL`
- creates a **new, empty database for each test**, runs the migrations, and passes in a pool
- deletes the database afterwards (failed tests' databases are kept for debugging)

So tests can run **in parallel** without seeing each other's rows. That's the main reason to prefer it over a shared database.

Fixtures load SQL before the test:

```rust
#[sqlx::test(migrations = "./migrations", fixtures("tenants", "projects"))]
async fn lists_only_own_tenant(pool: PgPool) -> sqlx::Result<()> { /* ... */ Ok(()) }
// loads tests/fixtures/tenants.sql and tests/fixtures/projects.sql
```

### 7.3 What every repository needs a test for

| Behaviour | Example test name |
|---|---|
| Round trip | `insert_then_get_round_trips` |
| Tenant scoping on read, update, delete | `get_by_id_returns_none_for_other_tenant` |
| Optimistic locking | `update_with_stale_version_returns_none` |
| Constraints | `insert_duplicate_name_is_unique_violation` |
| Transactions roll back | `failed_second_insert_rolls_back_first` |
| Pagination | `list_respects_limit_and_order` |

### 7.4 When one shared database is required

Some setups can't create databases per test (limited permissions, a managed test server). Then all API tests share **one** database and must not run concurrently. With nextest, put them in a **serial test group** (section 8.2) instead of passing `--test-threads=1` by hand. That way unit tests and other binaries still run in parallel.

---

## 8. `cargo nextest`

```bash
cargo install cargo-nextest --locked
cargo nextest run --workspace
cargo nextest run -p my-api                 # one crate
cargo nextest run -E 'test(stale_version)'  # filter by test name
```

Why the conventions use nextest instead of `cargo test`:

| | `cargo test` | `cargo nextest` |
|---|---|---|
| Isolation | Tests in one binary share a process | **Each test is its own process** |
| Output | Interleaved | Clean per-test output, failures summarized at the end |
| Speed | Parallel within a binary | Parallel across all binaries |
| Slow / flaky tests | No help | Timeouts, retries, "slow test" warnings |
| Grouping | `--test-threads` for everything | **Test groups** with their own limits |

One gap: nextest doesn't run doctests. Run `cargo test --doc` separately if you have them.

### 8.1 Configuration

```toml name=.config/nextest.toml
[profile.default]
slow-timeout = { period = "30s", terminate-after = 4 }   # kill tests stuck for 2 minutes

[profile.ci]
retries = 0
fail-fast = false        # report all failures, not just the first
```

### 8.2 Serial test groups

```toml name=.config/nextest.toml
[test-groups]
api-integration-db = { max-threads = 1 }

[[profile.default.overrides]]
filter = 'package(my-api) and kind(test)'   # integration test binaries of my-api
test-group = 'api-integration-db'
```

Tests in the `api-integration-db` group run one at a time. Every other test still runs fully in parallel.

---

## 9. Controlling time

Tests that `sleep` for real are slow and flaky. Tokio can **pause time** (feature `test-util`, in `dev-dependencies`):

```toml
[dev-dependencies]
tokio = { version = "1", features = ["full", "test-util"] }
```

```rust
use std::time::Duration;

#[tokio::test(start_paused = true)]
async fn purges_expired_blocklist_entries_every_five_minutes() {
    let blocklist = Blocklist::default();
    blocklist.revoke("old", now_secs() - 1).expect("revoke");
    spawn_purge_task(blocklist.clone(), Duration::from_secs(300));

    tokio::time::advance(Duration::from_secs(301)).await;   // instant, no real waiting
    tokio::task::yield_now().await;                          // let the purge task run

    assert!(!blocklist.is_revoked("old").expect("read"));
}
```

- With time paused, `sleep`, `interval`, and `timeout` jump forward **instantly** once all tasks are idle.
- Use this for timeouts (Stage 7), heartbeats (Stage 9), and cleanup tasks (Stage 8).
- Paused time only affects **Tokio's** clock. `SystemTime::now()` and JWT `exp` checks still use the real clock. Inject a clock (a function or trait that returns "now") where that matters.

---

## 10. Testing streaming and WebSockets

### 10.1 Streaming: prove the properties, not just the output

Stage 10, section 11 listed three properties. As tests:

```rust
#[tokio::test]
async fn export_emits_first_line_before_source_is_exhausted() {
    let (metrics, mut body) = open_export(1_000_000).await;

    let first = body.next().await.expect("a chunk").expect("no error");

    assert!(first.starts_with(br#"{"id":1"#));
    assert_eq!(metrics.finished.load(Ordering::Relaxed), 0);
}

#[tokio::test]
async fn export_keeps_resident_rows_bounded_as_input_grows() {
    let small = max_rows_ahead(1_000).await;
    let large = max_rows_ahead(200_000).await;

    assert!(small <= CHANNEL_CAPACITY as u64 + 2);
    assert!(large <= CHANNEL_CAPACITY as u64 + 2);   // doesn't grow with the input
}

#[tokio::test]
async fn export_stops_producer_when_client_disconnects() {
    let (metrics, mut body) = open_export(1_000_000).await;
    for _ in 0..3 {
        body.next().await;
    }
    drop(body);
    tokio::time::sleep(Duration::from_millis(50)).await;

    assert_eq!(metrics.finished.load(Ordering::Relaxed), 0);
    assert!(metrics.stopped_at.load(Ordering::Relaxed) < 100);
}
```

A test that only checks "the export contains 1,000 lines" would pass for a version that collects everything into a `Vec` first.

### 10.2 WebSockets: a real socket

WebSockets need a real connection. Start the server on port 0 (a random free port) inside the test, as in Stage 9, section 12:

```rust
async fn spawn_server(state: AppState) -> std::net::SocketAddr {
    let listener = tokio::net::TcpListener::bind("127.0.0.1:0").await.expect("bind");
    let addr = listener.local_addr().expect("addr");
    tokio::spawn(async move {
        axum::serve(listener, my_service::app(state)).await.expect("server");
    });
    addr
}

#[tokio::test]
async fn second_user_sees_first_user_join() {
    let addr = spawn_server(test_state()).await;
    let mut anna = connect(addr, 7, ANNA).await;
    let _ = next_event(&mut anna).await;             // snapshot

    let _ben = connect(addr, 7, BEN).await;

    assert_eq!(next_event(&mut anna).await, r#"{"type":"joined","user_id":2}"#);
}
```

Always read with a **timeout** (as `next_event` does), so a missing message fails the test instead of hanging it forever.

---

## 11. Test naming and conventions

**Name tests by behaviour**, so the list of test names reads like a specification:

| Good | Bad |
|---|---|
| `returns_409_on_stale_version` | `test_update` |
| `rejects_expired_token` | `auth_test_3` |
| `hides_projects_from_other_tenants` | `test_get_project_2` |
| `stream_chunk_empty_stream` | `it_works` |

Other conventions:

- **Arrange, act, assert**, separated by blank lines.
- **One behaviour per test.** Several assertions are fine if they describe the same behaviour.
- **No shared mutable state** between tests: each builds its own `AppState` or gets its own database.
- **No real network calls** to other services. Inject a client trait or a test server.
- **No sleeping for synchronization.** Use channels, `yield_now`, timeouts, or paused time.
- **A bug fix starts with a failing test** that reproduces it.

---

## 12. Running tests in CI

```yaml name=.github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: app
          POSTGRES_DB: app_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U app"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
    env:
      DATABASE_URL: postgres://app:app@localhost:5432/app_test
      SQLX_OFFLINE: "true"
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt
      - uses: Swatinem/rust-cache@v2
      - uses: taiki-e/install-action@v2
        with:
          tool: cargo-nextest,sqlx-cli
      - run: cargo fmt --all -- --check
      - run: cargo clippy --workspace --all-targets -- -D warnings
      - run: cargo sqlx prepare --check --workspace
      - run: cargo nextest run --workspace --profile ci
```

What each step guards:

| Step | Catches |
|---|---|
| `fmt --check` | Unformatted code |
| `clippy -D warnings` | Lints, including `unwrap_used` in runtime code |
| `sqlx prepare --check` | Queries changed without regenerating `.sqlx/` (Stage 6, section 6.1) |
| `nextest --profile ci` | Every unit, handler, and database test |

---

## 13. Putting it all together

A small library crate with an in-memory projects API, plus an integration test file that covers success, validation, every error status, authentication, tenant isolation, and optimistic locking. Run it as a local project with `cargo nextest run` (or `cargo test`).

```text
projects-tests/
├── Cargo.toml
├── clippy.toml
├── src/lib.rs
└── tests/api.rs
```

```toml name=Cargo.toml
[package]
name = "projects-tests"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "2"
tokio = { version = "1", features = ["full"] }

[dev-dependencies]
tower = { version = "0.5", features = ["util"] }
tokio = { version = "1", features = ["full", "test-util"] }
```

```toml name=clippy.toml
allow-unwrap-in-tests = true
allow-expect-in-tests = true
```

```rust name=src/lib.rs
use std::collections::HashMap;
use std::sync::{Arc, RwLock};

use axum::{
    extract::{rejection::JsonRejection, FromRequest, FromRequestParts, Path, Request, State},
    http::{header, request::Parts, StatusCode},
    response::{IntoResponse, Response},
    routing::{get, post},
    Json, Router,
};
use serde::{de::DeserializeOwned, Deserialize, Serialize};

// ================= errors =================

#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("{0} not found")]
    NotFound(String),
    #[error("{0}")]
    BadRequest(String),
    #[error("authentication required")]
    Unauthorized,
    #[error("{0}")]
    Conflict(String),
    #[error("internal server error")]
    Internal,
}

#[derive(Serialize)]
struct ErrorBody {
    error: &'static str,
    message: String,
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, code) = match &self {
            AppError::NotFound(_) => (StatusCode::NOT_FOUND, "not_found"),
            AppError::BadRequest(_) => (StatusCode::BAD_REQUEST, "bad_request"),
            AppError::Unauthorized => (StatusCode::UNAUTHORIZED, "unauthorized"),
            AppError::Conflict(_) => (StatusCode::CONFLICT, "conflict"),
            AppError::Internal => (StatusCode::INTERNAL_SERVER_ERROR, "internal"),
        };
        (status, Json(ErrorBody { error: code, message: self.to_string() })).into_response()
    }
}

impl From<JsonRejection> for AppError {
    fn from(r: JsonRejection) -> Self {
        AppError::BadRequest(r.body_text())
    }
}

pub struct AppJson<T>(pub T);

impl<S, T> FromRequest<S> for AppJson<T>
where
    S: Send + Sync,
    T: DeserializeOwned,
{
    type Rejection = AppError;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(v) = Json::<T>::from_request(req, state).await?;
        Ok(AppJson(v))
    }
}

// ================= auth (fake token: "user-<id>-tenant-<id>") =================

#[derive(Debug, Clone, Copy)]
pub struct AuthUser {
    pub user_id: i32,
    pub tenant_id: i32,
}

impl<S: Send + Sync> FromRequestParts<S> for AuthUser {
    type Rejection = AppError;

    async fn from_request_parts(parts: &mut Parts, _: &S) -> Result<Self, Self::Rejection> {
        let token = parts
            .headers
            .get(header::AUTHORIZATION)
            .and_then(|v| v.to_str().ok())
            .and_then(|v| v.strip_prefix("Bearer "))
            .ok_or(AppError::Unauthorized)?;
        let rest = token.strip_prefix("user-").ok_or(AppError::Unauthorized)?;
        let (u, t) = rest.split_once("-tenant-").ok_or(AppError::Unauthorized)?;
        Ok(AuthUser {
            user_id: u.parse().map_err(|_| AppError::Unauthorized)?,
            tenant_id: t.parse().map_err(|_| AppError::Unauthorized)?,
        })
    }
}

// ================= store =================

#[derive(Debug, Clone, Serialize)]
pub struct Project {
    pub id: i32,
    #[serde(skip)]
    pub tenant_id: i32,
    pub name: String,
    pub version: i32,
}

#[derive(Default)]
struct Inner {
    next_id: i32,
    projects: HashMap<i32, Project>,
}

#[derive(Clone, Default)]
pub struct AppState(Arc<RwLock<Inner>>);

fn validate_name(name: &str) -> Result<String, AppError> {
    let name = name.trim();
    if name.is_empty() {
        return Err(AppError::BadRequest("name must not be empty".into()));
    }
    if name.len() > 100 {
        return Err(AppError::BadRequest("name must be at most 100 characters".into()));
    }
    Ok(name.to_owned())
}

// ================= handlers =================

#[derive(Deserialize)]
pub struct CreateProject {
    pub name: String,
}

#[derive(Deserialize)]
pub struct RenameProject {
    pub name: String,
    pub version: i32,
}

async fn create(
    State(state): State<AppState>,
    user: AuthUser,
    AppJson(body): AppJson<CreateProject>,
) -> Result<(StatusCode, Json<Project>), AppError> {
    let name = validate_name(&body.name)?;
    let mut inner = state.0.write().map_err(|_| AppError::Internal)?;
    if inner.projects.values().any(|p| p.tenant_id == user.tenant_id && p.name == name) {
        return Err(AppError::Conflict("a project with this name already exists".into()));
    }
    inner.next_id += 1;
    let project = Project { id: inner.next_id, tenant_id: user.tenant_id, name, version: 1 };
    inner.projects.insert(project.id, project.clone());
    Ok((StatusCode::CREATED, Json(project)))
}

async fn get_one(
    State(state): State<AppState>,
    user: AuthUser,
    Path(id): Path<i32>,
) -> Result<Json<Project>, AppError> {
    let inner = state.0.read().map_err(|_| AppError::Internal)?;
    inner
        .projects
        .get(&id)
        .filter(|p| p.tenant_id == user.tenant_id)
        .cloned()
        .map(Json)
        .ok_or_else(|| AppError::NotFound(format!("project {id}")))
}

async fn rename(
    State(state): State<AppState>,
    user: AuthUser,
    Path(id): Path<i32>,
    AppJson(body): AppJson<RenameProject>,
) -> Result<Json<Project>, AppError> {
    let name = validate_name(&body.name)?;
    let mut inner = state.0.write().map_err(|_| AppError::Internal)?;
    let project = inner
        .projects
        .get_mut(&id)
        .filter(|p| p.tenant_id == user.tenant_id)
        .ok_or_else(|| AppError::NotFound(format!("project {id}")))?;
    if project.version != body.version {
        return Err(AppError::Conflict(format!(
            "version conflict: expected {}, found {}",
            body.version, project.version
        )));
    }
    project.name = name;
    project.version += 1;
    Ok(Json(project.clone()))
}

pub fn app(state: AppState) -> Router {
    Router::new()
        .route("/projects", post(create))
        .route("/projects/{id}", get(get_one).put(rename))
        .with_state(state)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn validate_name_trims_whitespace() {
        assert_eq!(validate_name("  Alpha  ").ok(), Some("Alpha".to_owned()));
    }

    #[test]
    fn validate_name_rejects_blank() {
        assert!(matches!(validate_name("   "), Err(AppError::BadRequest(_))));
    }

    #[test]
    fn validate_name_rejects_101_chars() {
        assert!(validate_name(&"x".repeat(101)).is_err());
    }
}
```

```rust name=tests/api.rs
use axum::{
    body::{to_bytes, Body},
    http::{header, Method, Request, StatusCode},
    Router,
};
use projects_tests::{app, AppState};
use serde_json::{json, Value};
use tower::ServiceExt;

// ================= test client =================

struct TestResponse {
    status: StatusCode,
    body: Value,
}

struct TestApp {
    router: Router,
    token: Option<&'static str>,
}

impl TestApp {
    fn new(state: AppState) -> Self {
        Self { router: app(state), token: None }
    }

    fn as_user(&self, token: &'static str) -> Self {
        Self { router: self.router.clone(), token: Some(token) }
    }

    async fn send(&self, method: Method, uri: &str, raw_body: Option<&str>) -> TestResponse {
        let mut builder = Request::builder().method(method).uri(uri);
        if let Some(t) = self.token {
            builder = builder.header(header::AUTHORIZATION, format!("Bearer {t}"));
        }
        let body = match raw_body {
            Some(b) => {
                builder = builder.header(header::CONTENT_TYPE, "application/json");
                Body::from(b.to_owned())
            }
            None => Body::empty(),
        };
        let response = self
            .router
            .clone()
            .oneshot(builder.body(body).expect("valid request"))
            .await
            .expect("infallible");
        let status = response.status();
        let bytes = to_bytes(response.into_body(), usize::MAX).await.expect("body");
        let body = serde_json::from_slice(&bytes).unwrap_or(Value::Null);
        TestResponse { status, body }
    }

    async fn get(&self, uri: &str) -> TestResponse {
        self.send(Method::GET, uri, None).await
    }

    async fn post(&self, uri: &str, body: Value) -> TestResponse {
        self.send(Method::POST, uri, Some(&body.to_string())).await
    }

    async fn put(&self, uri: &str, body: Value) -> TestResponse {
        self.send(Method::PUT, uri, Some(&body.to_string())).await
    }

    async fn create_project(&self, name: &str) -> i64 {
        let res = self.post("/projects", json!({ "name": name })).await;
        assert_eq!(res.status, StatusCode::CREATED, "setup: create {name}");
        res.body["id"].as_i64().expect("id")
    }
}

const ANNA: &str = "user-1-tenant-1";
const BEN: &str = "user-2-tenant-2";

// ================= success paths =================

#[tokio::test]
async fn creates_project_with_version_1() {
    let app = TestApp::new(AppState::default()).as_user(ANNA);

    let res = app.post("/projects", json!({ "name": "  Alpha  " })).await;

    assert_eq!(res.status, StatusCode::CREATED);
    assert_eq!(res.body["name"], "Alpha");
    assert_eq!(res.body["version"], 1);
    assert!(res.body.get("tenant_id").is_none(), "tenant_id must not be exposed");
}

#[tokio::test]
async fn rename_increments_version() {
    let app = TestApp::new(AppState::default()).as_user(ANNA);
    let id = app.create_project("Alpha").await;

    let res = app.put(&format!("/projects/{id}"), json!({ "name": "Beta", "version": 1 })).await;

    assert_eq!(res.status, StatusCode::OK);
    assert_eq!(res.body, json!({ "id": id, "name": "Beta", "version": 2 }));
}

// ================= errors =================

#[tokio::test]
async fn returns_404_with_error_body_for_unknown_project() {
    let app = TestApp::new(AppState::default()).as_user(ANNA);

    let res = app.get("/projects/999").await;

    assert_eq!(res.status, StatusCode::NOT_FOUND);
    assert_eq!(res.body, json!({ "error": "not_found", "message": "project 999 not found" }));
}

#[tokio::test]
async fn returns_400_for_blank_name() {
    let app = TestApp::new(AppState::default()).as_user(ANNA);

    let res = app.post("/projects", json!({ "name": "   " })).await;

    assert_eq!(res.status, StatusCode::BAD_REQUEST);
    assert_eq!(res.body["message"], "name must not be empty");
}

#[tokio::test]
async fn returns_400_with_json_error_body_for_wrong_shape() {
    let app = TestApp::new(AppState::default()).as_user(ANNA);

    let res = app.post("/projects", json!({ "name": 5 })).await;

    assert_eq!(res.status, StatusCode::BAD_REQUEST);
    assert_eq!(res.body["error"], "bad_request");
}

#[tokio::test]
async fn returns_409_for_duplicate_name_in_same_tenant() {
    let app = TestApp::new(AppState::default()).as_user(ANNA);
    app.create_project("Alpha").await;

    let res = app.post("/projects", json!({ "name": "Alpha" })).await;

    assert_eq!(res.status, StatusCode::CONFLICT);
}

#[tokio::test]
async fn returns_409_on_stale_version() {
    let app = TestApp::new(AppState::default()).as_user(ANNA);
    let id = app.create_project("Alpha").await;
    app.put(&format!("/projects/{id}"), json!({ "name": "B", "version": 1 })).await;

    let res = app.put(&format!("/projects/{id}"), json!({ "name": "C", "version": 1 })).await;

    assert_eq!(res.status, StatusCode::CONFLICT);
    assert_eq!(res.body["message"], "version conflict: expected 1, found 2");
}

// ================= auth =================

#[tokio::test]
async fn rejects_request_without_token() {
    let app = TestApp::new(AppState::default());

    let res = app.get("/projects/1").await;

    assert_eq!(res.status, StatusCode::UNAUTHORIZED);
    assert_eq!(res.body["error"], "unauthorized");
}

#[tokio::test]
async fn rejects_malformed_token() {
    let app = TestApp::new(AppState::default()).as_user("not-a-token");

    let res = app.get("/projects/1").await;

    assert_eq!(res.status, StatusCode::UNAUTHORIZED);
}

// ================= tenant isolation =================

#[tokio::test]
async fn hides_projects_from_other_tenants() {
    let shared = TestApp::new(AppState::default());
    let anna = shared.as_user(ANNA);
    let ben = shared.as_user(BEN);
    let id = anna.create_project("Secret").await;

    let read = ben.get(&format!("/projects/{id}")).await;
    let write = ben.put(&format!("/projects/{id}"), json!({ "name": "Hacked", "version": 1 })).await;

    assert_eq!(read.status, StatusCode::NOT_FOUND);
    assert_eq!(write.status, StatusCode::NOT_FOUND);
    assert_eq!(anna.get(&format!("/projects/{id}")).await.body["name"], "Secret");
}

#[tokio::test]
async fn allows_same_name_in_different_tenants() {
    let shared = TestApp::new(AppState::default());

    shared.as_user(ANNA).create_project("Alpha").await;
    let res = shared.as_user(BEN).post("/projects", json!({ "name": "Alpha" })).await;

    assert_eq!(res.status, StatusCode::CREATED);
}

// ================= time (paused clock) =================

#[tokio::test(start_paused = true)]
async fn timeout_fires_without_real_waiting() {
    let started = std::time::Instant::now();

    let result = tokio::time::timeout(
        std::time::Duration::from_secs(30),
        tokio::time::sleep(std::time::Duration::from_secs(60)),
    )
    .await;

    assert!(result.is_err(), "the 30 s timeout should fire first");
    assert!(started.elapsed() < std::time::Duration::from_secs(1), "no real waiting");
}
```

Run it:

```bash
cargo nextest run
```

Expected output (order and timings vary):

```text
    Starting 15 tests across 2 binaries
        PASS [   0.004s] projects-tests tests::validate_name_rejects_101_chars
        PASS [   0.004s] projects-tests tests::validate_name_rejects_blank
        PASS [   0.004s] projects-tests tests::validate_name_trims_whitespace
        PASS [   0.008s] projects-tests::api allows_same_name_in_different_tenants
        PASS [   0.008s] projects-tests::api creates_project_with_version_1
        PASS [   0.008s] projects-tests::api hides_projects_from_other_tenants
        PASS [   0.007s] projects-tests::api rejects_malformed_token
        PASS [   0.007s] projects-tests::api rejects_request_without_token
        PASS [   0.008s] projects-tests::api rename_increments_version
        PASS [   0.008s] projects-tests::api returns_400_for_blank_name
        PASS [   0.008s] projects-tests::api returns_400_with_json_error_body_for_wrong_shape
        PASS [   0.008s] projects-tests::api returns_404_with_error_body_for_unknown_project
        PASS [   0.008s] projects-tests::api returns_409_for_duplicate_name_in_same_tenant
        PASS [   0.008s] projects-tests::api returns_409_on_stale_version
        PASS [   0.006s] projects-tests::api timeout_fires_without_real_waiting
------------
     Summary [   0.050s] 15 tests run: 15 passed, 0 skipped
```

What the suite covers:

| Group | Tests |
|---|---|
| Unit | Validation rules, without HTTP |
| Success | Status, JSON shape, `tenant_id` not exposed, version increments |
| Errors | 400 (validation and shape), 404, 409 (duplicate and stale version), all with the JSON error body |
| Auth | Missing and malformed tokens → 401 |
| Tenant isolation | Another tenant can't read **or change** data; names are unique only within a tenant |
| Time | A 60-second scenario runs instantly with a paused clock |

Try breaking the code on purpose: remove `.filter(|p| p.tenant_id == user.tenant_id)` from `rename`, and `hides_projects_from_other_tenants` fails on the final assertion, even though `read` still returns 404.

---

## 14. Exercises

- [ ] **1. Unit tests:** write tests for `Config::from_lookup` (Stage 5): a missing required key, an invalid number, a default value, and a too-short secret. Use `HashMap`-backed lookups, never real environment variables.
- [ ] **2. Error coverage:** add a `Forbidden` variant and a route that returns it, and write one test per `AppError` variant that checks status **and** the exact JSON body.
- [ ] **3. Real JWTs:** with the Stage 8 code, write a `token_for(user_id, tenant_id, role)` helper using a test `JwtKeys`. Add `rejects_expired_token`, `rejects_tampered_token`, and `rejects_revoked_token_after_logout`.
- [ ] **4. Database tests:** with the Stage 6 project, write `#[sqlx::test]` tests for `get_by_id_returns_none_for_other_tenant`, `update_with_stale_version_returns_none`, and `insert_duplicate_name_is_unique_violation`. Run them with nextest and confirm they run in parallel.
- [ ] **5. Serial group:** set up `.config/nextest.toml` with an `api-integration-db` group of `max-threads = 1` for the integration test binary. Use `cargo nextest list` and `cargo nextest run` to confirm which tests are in the group.
- [ ] **6. Streaming properties:** turn the three checks from the Stage 10 example into `#[tokio::test]` functions. Then switch the channel to `unbounded_channel` and confirm the bounded-residency test fails.
- [ ] **7. Paused time:** write `#[tokio::test(start_paused = true)]` tests for the blocklist purge task (Stage 8) and the WebSocket heartbeat (Stage 9), using `tokio::time::advance`.
- [ ] **8. Explain it:** in three or four sentences, explain why database tests use a real PostgreSQL instead of mocks, why the tenant-isolation test checks the data after the 404, and why a streaming test must check more than the final output.

> When you can solve exercises 1–8 without looking anything up, you're ready for the **reference service**: one multi-tenant service split into `api` / `domain` / `db` / `shared`, using every stage.

---

## 15. Further reading

- [*The Rust Programming Language*, chapter 11](https://doc.rust-lang.org/book/ch11-00-testing.html): unit tests, integration tests, and test organization
- [`cargo-nextest` docs](https://nexte.st/): test groups, profiles, filtersets, CI setup
- [`#[sqlx::test]` docs](https://docs.rs/sqlx/latest/sqlx/attr.test.html): per-test databases, migrations, fixtures
- [Axum example: `testing`](https://github.com/tokio-rs/axum/tree/main/examples/testing): `oneshot` tests and tests over a real socket
- [`tokio::time::pause`](https://docs.rs/tokio/latest/tokio/time/fn.pause.html): paused time and `advance`
- [Clippy configuration](https://doc.rust-lang.org/clippy/lint_configuration.html): `allow-unwrap-in-tests` and related settings
- *Zero To Production in Rust*, the chapters on testing: black-box API tests and per-test databases
