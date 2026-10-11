# Stage 4: Errors

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> If you come from JavaScript or TypeScript, the comparisons below map Rust concepts to familiar JS ideas (mostly Express).

## Contents

1. [The goal: one error type, one error shape](#1-the-goal-one-error-type-one-error-shape)
2. [Rust error basics, revisited](#2-rust-error-basics-revisited)
3. [`thiserror`: defining error types](#3-thiserror-defining-error-types)
4. [`anyhow`: errors at the edges](#4-anyhow-errors-at-the-edges)
5. [Designing `AppError`](#5-designing-apperror)
6. [`IntoResponse` for `AppError`](#6-intoresponse-for-apperror)
7. [`From` conversions and `?`](#7-from-conversions-and-)
8. [Turning extractor rejections into `AppError`](#8-turning-extractor-rejections-into-apperror)
9. [Internal errors: log everything, leak nothing](#9-internal-errors-log-everything-leak-nothing)
10. [How this looks in a production service](#10-how-this-looks-in-a-production-service)
11. [Putting it all together](#11-putting-it-all-together)
12. [Exercises](#12-exercises)
13. [Further reading](#13-further-reading)

---

## 1. The goal: one error type, one error shape

At the end of Stage 3, the API returned errors in several formats:

| Source | Status | Body |
|---|---|---|
| `Path` rejection | 400 | plain text from Axum |
| `Json` rejection | 415 / 400 / 422 | plain text from Axum |
| Custom extractor | 401 | plain text we wrote |
| Handler `Err(StatusCode::NOT_FOUND)` | 404 | empty |
| Fallback | 404 | JSON |

A client (for example a React frontend) can't handle that reliably. The goal of this stage:

- **One** error enum, `AppError`, used by every handler and extractor.
- **One** JSON shape for every error:

  ```json
  { "error": "not_found", "message": "project 9 not found" }
  ```

- A stable machine-readable `error` code for the frontend, a human-readable `message` for people.
- **No internal details** (SQL, stack traces, file paths) ever sent to the client.

> **JS comparison:** in Express this is the error-handling middleware `app.use((err, req, res, next) => ...)` at the end of the chain. In Axum there's no "catch-all" step; instead every handler returns `Result<T, AppError>`, and `AppError` knows how to turn itself into a response.

---

## 2. Rust error basics, revisited

Stage 0 covered `Result`, `?`, `map_err`, and `From`. This stage adds the standard **`Error` trait**:

```rust
pub trait Error: Debug + Display {
    fn source(&self) -> Option<&(dyn Error + 'static)> { None }
}
```

An error type should:

| Implement | Why |
|---|---|
| `Debug` | For logs and `{:?}` (usually `#[derive(Debug)]`) |
| `Display` | The human-readable message, `{}` |
| `Error` | So it works with `?`, `Box<dyn Error>`, `anyhow`, and logging |
| `source()` | Points at the **underlying** error, forming a chain |

Writing this by hand is tedious:

```rust
#[derive(Debug)]
enum ParseConfigError {
    Missing(String),
    InvalidPort(std::num::ParseIntError),
}

impl std::fmt::Display for ParseConfigError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::Missing(key) => write!(f, "missing config key {key}"),
            Self::InvalidPort(_) => write!(f, "invalid port"),
        }
    }
}

impl std::error::Error for ParseConfigError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            Self::InvalidPort(e) => Some(e),
            Self::Missing(_) => None,
        }
    }
}

impl From<std::num::ParseIntError> for ParseConfigError {
    fn from(e: std::num::ParseIntError) -> Self {
        Self::InvalidPort(e)
    }
}
```

That's 30 lines of boilerplate for two variants. `thiserror` generates all of it.

---

## 3. `thiserror`: defining error types

```toml
thiserror = "2"
```

The same error as above:

```rust
#[derive(Debug, thiserror::Error)]
enum ParseConfigError {
    #[error("missing config key {0}")]
    Missing(String),

    #[error("invalid port")]
    InvalidPort(#[from] std::num::ParseIntError),
}
```

What the attributes do:

| Attribute | Generates |
|---|---|
| `#[derive(thiserror::Error)]` | `impl std::error::Error` |
| `#[error("...")]` | `impl Display`. Use `{0}` for tuple fields, `{name}` for named fields |
| `#[from]` | `impl From<Inner> for YourError` **and** `source()` returns the inner error |
| `#[source]` | Only `source()`, no `From` |
| `#[error(transparent)]` | Forward `Display` and `source()` to the inner error |

```rust
fn read_port(raw: &str) -> Result<u16, ParseConfigError> {
    let port: u16 = raw.parse()?;   // ParseIntError -> ParseConfigError via #[from]
    Ok(port)
}
```

> **JS comparison:** like writing `class InvalidPortError extends Error { constructor(cause) { super("invalid port", { cause }) } }`, except the compiler forces callers to handle every variant.

`thiserror` adds nothing at runtime. It only writes the code you would have written by hand.

---

## 4. `anyhow`: errors at the edges

```toml
anyhow = "1"
```

`anyhow::Error` is **one type that can hold any error**, with optional context:

```rust
use anyhow::Context;

fn load_config() -> anyhow::Result<u16> {
    let raw = std::env::var("PORT").context("PORT is not set")?;
    let port = raw.parse::<u16>().with_context(|| format!("PORT={raw} is not a number"))?;
    Ok(port)
}

fn main() -> anyhow::Result<()> {
    let port = load_config()?;
    println!("port {port}");
    Ok(())
}
```

If `main` returns an error, the program prints the whole chain:

```text
Error: PORT=abc is not a number

Caused by:
    invalid digit found in string
```

### 4.1 `thiserror` or `anyhow`?

| | `thiserror` | `anyhow` |
|---|---|---|
| What it is | A derive that builds **your own** enum | One opaque error type |
| Caller can `match` on the cause | Yes | Not easily |
| Best for | Libraries, domain logic, `AppError` | Binaries: `main`, startup, scripts |
| Convention in this repo | `domain`, `db`, `api` | `main` only |

Rule of thumb: **if a caller needs to react differently to different failures, use an enum (`thiserror`).** An HTTP handler must choose between 404, 409, and 500, so `AppError` is an enum. At startup, any failure means "print it and exit", so `anyhow` is fine there.

---

## 5. Designing `AppError`

The conventions list six variants. Each one maps to exactly one status code:

```rust
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("{0} not found")]
    NotFound(String),

    #[error("{0}")]
    BadRequest(String),

    #[error("authentication required")]
    Unauthorized,

    #[error("you do not have access to this resource")]
    Forbidden,

    #[error("{0}")]
    Conflict(String),

    #[error("internal server error")]
    Internal(#[source] Box<dyn std::error::Error + Send + Sync>),
}
```

| Variant | Status | Typical cause |
|---|---|---|
| `NotFound` | 404 | Row missing, **or** it belongs to another tenant |
| `BadRequest` | 400 | Invalid input, failed validation |
| `Unauthorized` | 401 | No token, bad token, expired, revoked |
| `Forbidden` | 403 | Logged in but missing the capability |
| `Conflict` | 409 | Stale `version` (optimistic locking), unique constraint |
| `Internal` | 500 | Database down, a bug, anything unexpected |

Design decisions worth noticing:

- **Messages for 4xx are written for the client.** `NotFound("project 9")` displays as `project 9 not found`.
- **`Internal` keeps the real error as its `source`**, but its `Display` is always the generic `internal server error`. The cause is for logs, not for clients (section 9).
- **`Box<dyn Error + Send + Sync>`** can hold *any* error type. `Send + Sync` is required because Axum futures run on multiple threads (Stage 1).
- **Cross-tenant access returns 404, not 403.** Saying "forbidden" would confirm that the resource exists in someone else's tenant.
- **Keep the list short.** Add a variant only when the client must react differently, for example `TooManyRequests` (429) when rate limiting arrives.

---

## 6. `IntoResponse` for `AppError`

Stage 0 showed the idea. Here is the full version:

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde::Serialize;

#[derive(Serialize)]
struct ErrorBody {
    error: &'static str,
    message: String,
}

impl AppError {
    fn status(&self) -> StatusCode {
        match self {
            Self::NotFound(_) => StatusCode::NOT_FOUND,
            Self::BadRequest(_) => StatusCode::BAD_REQUEST,
            Self::Unauthorized => StatusCode::UNAUTHORIZED,
            Self::Forbidden => StatusCode::FORBIDDEN,
            Self::Conflict(_) => StatusCode::CONFLICT,
            Self::Internal(_) => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }

    fn code(&self) -> &'static str {
        match self {
            Self::NotFound(_) => "not_found",
            Self::BadRequest(_) => "bad_request",
            Self::Unauthorized => "unauthorized",
            Self::Forbidden => "forbidden",
            Self::Conflict(_) => "conflict",
            Self::Internal(_) => "internal",
        }
    }
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        if let Self::Internal(source) = &self {
            // Stage 7 replaces this with tracing::error!
            eprintln!("internal error: {source}");
        }

        let body = ErrorBody {
            error: self.code(),
            message: self.to_string(),   // Display from #[error(...)]
        };
        (self.status(), Json(body)).into_response()
    }
}
```

Now any handler can return `Result<T, AppError>`:

```rust
async fn get_project(Path(id): Path<i32>) -> Result<Json<Project>, AppError> {
    find(id).map(Json).ok_or_else(|| AppError::NotFound(format!("project {id}")))
}
```

Because every `match` is exhaustive, adding a variant later (say `TooManyRequests`) causes compile errors in `status()` and `code()` until you decide its status code and code string. Nothing can be forgotten.

---

## 7. `From` conversions and `?`

`?` calls `From::from` on the error. So if lower layers have their own error types, implement `From<LowerError> for AppError` once, and every handler can just use `?`.

### 7.1 A domain error

The `domain` layer must not know about HTTP (README architecture), so it gets its own enum:

```rust
#[derive(Debug, thiserror::Error)]
pub enum DomainError {
    #[error("project {0} not found")]
    ProjectNotFound(i32),

    #[error("version conflict: expected {expected}, found {actual}")]
    VersionMismatch { expected: i32, actual: i32 },

    #[error("live projects cannot be edited")]
    ProjectIsLive,

    #[error("storage failure")]
    Storage(#[source] Box<dyn std::error::Error + Send + Sync>),
}
```

### 7.2 The mapping, in one place

```rust
impl From<DomainError> for AppError {
    fn from(err: DomainError) -> Self {
        match err {
            DomainError::ProjectNotFound(id) => AppError::NotFound(format!("project {id}")),
            e @ DomainError::VersionMismatch { .. } => AppError::Conflict(e.to_string()),
            e @ DomainError::ProjectIsLive => AppError::Conflict(e.to_string()),
            e @ DomainError::Storage(_) => AppError::Internal(Box::new(e)),
        }
    }
}
```

- `e @ Pattern` binds the whole value while matching a pattern, so you can reuse its `Display` text.
- `Storage` is boxed **including** its source, so the logs still see the full chain.

Now handlers stay one line:

```rust
async fn update(/* extractors */) -> Result<Json<Project>, AppError> {
    let project = domain::update_project(id, body)?;   // DomainError -> AppError
    Ok(Json(project))
}
```

### 7.3 Mapping database errors (preview of Stage 6)

```rust
impl From<sqlx::Error> for AppError {
    fn from(err: sqlx::Error) -> Self {
        match &err {
            sqlx::Error::RowNotFound => AppError::NotFound("record".into()),
            sqlx::Error::Database(db) if db.is_unique_violation() => {
                AppError::Conflict("a record with these values already exists".into())
            }
            _ => AppError::Internal(Box::new(err)),
        }
    }
}
```

Unknown database errors become `Internal`: logged in full, sent to the client as a generic 500.

> **Avoid a blanket `impl<E: Error> From<E> for AppError`.** It would turn *every* error into a 500 silently, including ones that should be 404 or 409. Explicit `From` impls make you decide.

---

## 8. Turning extractor rejections into `AppError`

Rejections from `Path`, `Query`, and `Json` still produce Axum's plain-text bodies. Fix it with `From` impls plus small wrapper extractors.

### 8.1 `From` for each rejection

```rust
use axum::extract::rejection::{JsonRejection, PathRejection, QueryRejection};

impl From<JsonRejection> for AppError {
    fn from(r: JsonRejection) -> Self { AppError::BadRequest(r.body_text()) }
}
impl From<PathRejection> for AppError {
    fn from(r: PathRejection) -> Self { AppError::BadRequest(r.body_text()) }
}
impl From<QueryRejection> for AppError {
    fn from(r: QueryRejection) -> Self { AppError::BadRequest(r.body_text()) }
}
```

This maps every input problem to **400**. If your frontend needs to tell "malformed JSON" (400) apart from "wrong shape" (422), add a `Validation(String)` variant with status 422 and match on the rejection type.

### 8.2 Wrapper extractors

```rust
use axum::extract::{FromRequest, FromRequestParts, Path, Request};
use axum::http::request::Parts;
use serde::de::DeserializeOwned;

pub struct AppJson<T>(pub T);

impl<S, T> FromRequest<S> for AppJson<T>
where
    S: Send + Sync,
    T: DeserializeOwned,
{
    type Rejection = AppError;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(value) = Json::<T>::from_request(req, state).await?;  // JsonRejection -> AppError
        Ok(AppJson(value))
    }
}

pub struct AppPath<T>(pub T);

impl<S, T> FromRequestParts<S> for AppPath<T>
where
    S: Send + Sync,
    T: DeserializeOwned + Send,
{
    type Rejection = AppError;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let Path(value) = Path::<T>::from_request_parts(parts, state).await?;
        Ok(AppPath(value))
    }
}
```

Handlers use `AppJson` and `AppPath` instead of `Json` and `Path` for **input**. For **output**, plain `Json` is still fine.

The `ValidJson<T>` from Stage 3 gets the same treatment: set `type Rejection = AppError` and return `AppError::BadRequest(msg)` when validation fails.

### 8.3 The fallback and custom extractors

Same idea: return `AppError` from them.

```rust
async fn not_found(uri: Uri) -> AppError {
    AppError::NotFound(format!("route {}", uri.path()))
}

impl<S: Send + Sync> FromRequestParts<S> for AuthUser {
    type Rejection = AppError;          // missing or bad token -> AppError::Unauthorized
    // ...
}
```

After this, **every** error the API returns has the same JSON shape.

---

## 9. Internal errors: log everything, leak nothing

A 500 means "our fault". Two audiences need different information:

| Audience | Needs | Gets |
|---|---|---|
| Client | To know it failed, and maybe to retry | `{"error":"internal","message":"internal server error"}` |
| You (logs) | The real cause, the full chain, which request | `internal error: storage failure: connection refused` |

### 9.1 Don't leak

```rust
// BAD: sends SQL details, table names, maybe hostnames to the client
AppError::BadRequest(sqlx_err.to_string())

// GOOD: keep it as the source of Internal; the client sees a generic message
AppError::Internal(Box::new(sqlx_err))
```

### 9.2 Log the whole chain

`Display` of the top error often isn't enough. Walk `source()`:

```rust
fn error_chain(err: &dyn std::error::Error) -> String {
    let mut text = err.to_string();
    let mut current = err.source();
    while let Some(cause) = current {
        text.push_str(": ");
        text.push_str(&cause.to_string());
        current = cause.source();
    }
    text
}
```

In Stage 7 this goes through `tracing::error!`, and the log line also carries the request ID and route, so you can find the request that failed.

### 9.3 Panics

A panic inside a handler aborts that request's task. The client sees a dropped connection, not a JSON error. Two defences:

1. **Don't panic in runtime code.** No `unwrap()` / `expect()`, enforced with `clippy::unwrap_used` and `clippy::expect_used`.
2. **Safety net:** `tower_http::catch_panic::CatchPanicLayer` turns a panic into a 500 response (Stage 7).

---

## 10. How this looks in a production service

```text
crates/
├── shared/src/error.rs      # AppError, ErrorBody, IntoResponse, From<sqlx::Error>, From<rejections>
├── domain/src/error.rs      # DomainError (no HTTP types)
├── db/src/...               # returns sqlx::Error or a small DbError
└── api/src/
    ├── extractors/          # AppJson, AppPath, AppQuery, ValidJson, AuthUser (all reject with AppError)
    └── routes/              # handlers: Result<_, AppError>
```

The error flow for one request:

```text
sqlx::Error ──From──▶ DomainError::Storage ──From──▶ AppError::Internal ──IntoResponse──▶ 500 + JSON
                      DomainError::VersionMismatch ─▶ AppError::Conflict ───────────────▶ 409 + JSON
JsonRejection ─────────────────────────────────────▶ AppError::BadRequest ──────────────▶ 400 + JSON
```

Rules that keep it working:

- Handlers return `Result<_, AppError>` and use `?`. They almost never build an error by hand.
- **Each `From` impl is the single place** where a lower-level error gets its status code.
- 4xx messages are safe to show to users. 5xx messages are always generic.
- Frontend code switches on `error` (`"conflict"` → "someone else changed this, reload"), never on `message`.

---

## 11. Putting it all together

A projects API with an in-memory store, a domain-style error, optimistic locking with `version`, wrapper extractors, and one JSON error shape for everything. It uses `with_state`, which Stage 5 explains in detail. It calls itself with `oneshot`, so it runs **in the Playground**.

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
tower = { version = "0.5", features = ["util"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "2"
```

```rust
use std::collections::HashMap;
use std::sync::{Arc, RwLock};

use axum::{
    body::{to_bytes, Body},
    extract::{
        rejection::{JsonRejection, PathRejection},
        FromRequest, FromRequestParts, Path, Request, State,
    },
    http::{header, request::Parts, StatusCode, Uri},
    response::{IntoResponse, Response},
    routing::get,
    Json, Router,
};
use serde::{de::DeserializeOwned, Deserialize, Serialize};
use tower::ServiceExt;

// ================= AppError =================

#[derive(Debug, thiserror::Error)]
enum AppError {
    #[error("{0} not found")]
    NotFound(String),
    #[error("{0}")]
    BadRequest(String),
    #[error("{0}")]
    Conflict(String),
    #[error("internal server error")]
    Internal(#[source] Box<dyn std::error::Error + Send + Sync>),
}

#[derive(Serialize)]
struct ErrorBody {
    error: &'static str,
    message: String,
}

impl AppError {
    fn status(&self) -> StatusCode {
        match self {
            Self::NotFound(_) => StatusCode::NOT_FOUND,
            Self::BadRequest(_) => StatusCode::BAD_REQUEST,
            Self::Conflict(_) => StatusCode::CONFLICT,
            Self::Internal(_) => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }

    fn code(&self) -> &'static str {
        match self {
            Self::NotFound(_) => "not_found",
            Self::BadRequest(_) => "bad_request",
            Self::Conflict(_) => "conflict",
            Self::Internal(_) => "internal",
        }
    }
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        if let Self::Internal(source) = &self {
            eprintln!("[log] internal error: {source}");
        }
        let body = ErrorBody { error: self.code(), message: self.to_string() };
        (self.status(), Json(body)).into_response()
    }
}

impl From<JsonRejection> for AppError {
    fn from(r: JsonRejection) -> Self {
        AppError::BadRequest(r.body_text())
    }
}

impl From<PathRejection> for AppError {
    fn from(r: PathRejection) -> Self {
        AppError::BadRequest(r.body_text())
    }
}

// ================= wrapper extractors =================

struct AppJson<T>(T);

impl<S, T> FromRequest<S> for AppJson<T>
where
    S: Send + Sync,
    T: DeserializeOwned,
{
    type Rejection = AppError;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(value) = Json::<T>::from_request(req, state).await?;
        Ok(AppJson(value))
    }
}

struct AppPath<T>(T);

impl<S, T> FromRequestParts<S> for AppPath<T>
where
    S: Send + Sync,
    T: DeserializeOwned + Send,
{
    type Rejection = AppError;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let Path(value) = Path::<T>::from_request_parts(parts, state).await?;
        Ok(AppPath(value))
    }
}

// ================= "domain" layer =================

#[derive(Debug, Clone, Serialize)]
struct Project {
    id: i32,
    name: String,
    version: i32,
}

#[derive(Debug, thiserror::Error)]
enum DomainError {
    #[error("project {0} not found")]
    ProjectNotFound(i32),
    #[error("version conflict: expected {expected}, found {actual}")]
    VersionMismatch { expected: i32, actual: i32 },
    #[error("name must not be empty")]
    EmptyName,
    #[error("storage lock poisoned")]
    Poisoned,
}

impl From<DomainError> for AppError {
    fn from(err: DomainError) -> Self {
        match err {
            DomainError::ProjectNotFound(id) => AppError::NotFound(format!("project {id}")),
            e @ DomainError::VersionMismatch { .. } => AppError::Conflict(e.to_string()),
            e @ DomainError::EmptyName => AppError::BadRequest(e.to_string()),
            e @ DomainError::Poisoned => AppError::Internal(Box::new(e)),
        }
    }
}

#[derive(Clone, Default)]
struct Store {
    projects: Arc<RwLock<HashMap<i32, Project>>>,
}

impl Store {
    fn seed() -> Self {
        let store = Store::default();
        if let Ok(mut map) = store.projects.write() {
            map.insert(1, Project { id: 1, name: "Alpha".into(), version: 1 });
        }
        store
    }

    fn get(&self, id: i32) -> Result<Project, DomainError> {
        let map = self.projects.read().map_err(|_| DomainError::Poisoned)?;
        map.get(&id).cloned().ok_or(DomainError::ProjectNotFound(id))
    }

    fn rename(&self, id: i32, name: &str, expected: i32) -> Result<Project, DomainError> {
        let name = name.trim();
        if name.is_empty() {
            return Err(DomainError::EmptyName);
        }
        let mut map = self.projects.write().map_err(|_| DomainError::Poisoned)?;
        let project = map.get_mut(&id).ok_or(DomainError::ProjectNotFound(id))?;
        if project.version != expected {
            return Err(DomainError::VersionMismatch { expected, actual: project.version });
        }
        project.name = name.to_string();
        project.version += 1;
        Ok(project.clone())
    }
}

// ================= handlers =================

#[derive(Deserialize)]
struct RenameProject {
    name: String,
    version: i32,
}

async fn get_project(
    State(store): State<Store>,
    AppPath(id): AppPath<i32>,
) -> Result<Json<Project>, AppError> {
    Ok(Json(store.get(id)?))
}

async fn rename_project(
    State(store): State<Store>,
    AppPath(id): AppPath<i32>,
    AppJson(body): AppJson<RenameProject>,
) -> Result<Json<Project>, AppError> {
    let project = store.rename(id, &body.name, body.version)?;
    Ok(Json(project))
}

async fn report() -> Result<Json<Project>, AppError> {
    // Simulates an unexpected failure deep in a lower layer
    Err(AppError::Internal("connection refused (db.internal:5432)".into()))
}

async fn not_found(uri: Uri) -> AppError {
    AppError::NotFound(format!("route {}", uri.path()))
}

fn app() -> Router {
    Router::new()
        .route("/projects/{id}", get(get_project).put(rename_project))
        .route("/report", get(report))
        .fallback(not_found)
        .with_state(Store::seed())
}

// ================= tiny client =================

async fn call(app: &Router, method: &str, uri: &str, json: Option<&str>) -> (StatusCode, String) {
    let mut builder = Request::builder().method(method).uri(uri);
    let body = match json {
        Some(j) => {
            builder = builder.header(header::CONTENT_TYPE, "application/json");
            Body::from(j.to_string())
        }
        None => Body::empty(),
    };
    let request = builder.body(body).expect("static request is valid");

    let response = match app.clone().oneshot(request).await {
        Ok(r) => r,
        Err(never) => match never {},
    };
    let status = response.status();
    let bytes = to_bytes(response.into_body(), usize::MAX).await.unwrap_or_default();
    (status, String::from_utf8_lossy(&bytes).into_owned())
}

#[tokio::main]
async fn main() {
    let app = app();

    let cases: [(&str, &str, Option<&str>); 9] = [
        ("GET", "/projects/1", None),
        ("GET", "/projects/9", None),
        ("GET", "/projects/abc", None),
        ("PUT", "/projects/1", Some(r#"{"name":"Alpha v2","version":1}"#)),
        ("PUT", "/projects/1", Some(r#"{"name":"Alpha v3","version":1}"#)),
        ("PUT", "/projects/1", Some(r#"{"name":"   ","version":2}"#)),
        ("PUT", "/projects/1", Some(r#"{"name":5}"#)),
        ("GET", "/report", None),
        ("GET", "/nope", None),
    ];

    for (method, uri, json) in cases {
        let (status, body) = call(&app, method, uri, json).await;
        println!("{method:<3} {uri:<14} -> {} {body}", status.as_u16());
    }
}
```

Expected output (the text of Axum and serde messages can differ between versions):

```text
GET /projects/1    -> 200 {"id":1,"name":"Alpha","version":1}
GET /projects/9    -> 404 {"error":"not_found","message":"project 9 not found"}
GET /projects/abc  -> 400 {"error":"bad_request","message":"Invalid URL: Cannot parse `abc` to a `i32`"}
PUT /projects/1    -> 200 {"id":1,"name":"Alpha v2","version":2}
PUT /projects/1    -> 409 {"error":"conflict","message":"version conflict: expected 1, found 2"}
PUT /projects/1    -> 400 {"error":"bad_request","message":"name must not be empty"}
PUT /projects/1    -> 400 {"error":"bad_request","message":"Failed to deserialize the JSON body into the target type: name: invalid type: integer `5`, expected a string at line 1 column 9"}
[log] internal error: connection refused (db.internal:5432)
GET /report        -> 500 {"error":"internal","message":"internal server error"}
GET /nope          -> 404 {"error":"not_found","message":"route /nope not found"}
```

What each line shows:

| Case | Concept |
|---|---|
| `/projects/9` | `DomainError::ProjectNotFound` → `From` → `AppError::NotFound` → 404 JSON |
| `/projects/abc` | `PathRejection` → `AppPath` → `AppError::BadRequest` |
| Second `PUT` with `version: 1` | **Optimistic locking**: stale version → 409 |
| `"name": "   "` | A domain rule → 400 |
| `"name": 5` | `JsonRejection` → `AppJson` → the same JSON shape |
| `/report` | `Internal`: the cause is **logged**, the client sees a generic message |
| `/nope` | The fallback returns `AppError` too |

Every error now has the same shape, and the handlers contain no error-formatting code.

---

## 12. Exercises

- [ ] **1. By hand, then `thiserror`:** write `ConfigError { Missing(String), InvalidPort(ParseIntError) }` with manual `Display`, `Error`, and `From` impls. Then rewrite it with `thiserror` and compare the line counts.
- [ ] **2. `anyhow` in `main`:** write a `main` that returns `anyhow::Result<()>`, reads a `PORT` variable, and adds `.context(...)` at each step. Run it with `PORT` unset and with `PORT=abc`, and read the "Caused by" chain.
- [ ] **3. Complete `AppError`:** add `Unauthorized` and `Forbidden` to the section 11 example, with `status()` and `code()`. Notice how the compiler points at every `match` you need to update.
- [ ] **4. Domain mapping:** add `DomainError::ProjectIsLive` and make `rename` return it when the name is `"LIVE"`. Map it to a 409 with its own message.
- [ ] **5. `AppQuery`:** write an `AppQuery<T>` wrapper like `AppPath`, with `From<QueryRejection> for AppError`. Use it in a `GET /projects?min_version=...` handler and send `min_version=abc`.
- [ ] **6. 400 vs 422:** add a `Validation(String)` variant with status 422. Make `AppJson` return 422 for `JsonRejection::JsonDataError` and 400 for everything else (`match` on the rejection).
- [ ] **7. Error chain:** wrap an `std::io::Error` inside a `DomainError::Storage(#[source] ...)` inside `AppError::Internal`, and log it with the `error_chain` function from section 9.2. Confirm the client still sees only `internal server error`.
- [ ] **8. Explain it:** in two or three sentences, explain why cross-tenant access should return 404 instead of 403, and why a blanket `impl<E: Error> From<E> for AppError` is a bad idea.

> When you can solve exercises 1–8 without looking anything up, you're ready for **Stage 5: State and configuration**.

---

## 13. Further reading

- [*The Rust Programming Language*, chapter 9](https://doc.rust-lang.org/book/ch09-00-error-handling.html): `panic!`, `Result`, and when to use which
- [`thiserror` docs](https://docs.rs/thiserror): every attribute, with examples
- [`anyhow` docs](https://docs.rs/anyhow): `Context`, `bail!`, `ensure!`, and how the chain is printed
- [`axum::response::IntoResponse`](https://docs.rs/axum/latest/axum/response/trait.IntoResponse.html): implementing it for your own types
- [Axum example: `error-handling`](https://github.com/tokio-rs/axum/tree/main/examples/error-handling): an `AppError` with `From` conversions and logging
- [Luca Palmieri: "Error Handling In Rust - A Deep Dive"](https://www.lpalmieri.com/posts/error-handling-rust/): the chapter from *Zero To Production* on errors for operators vs errors for users
