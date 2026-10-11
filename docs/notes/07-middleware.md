# Stage 7: Middleware

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> If you come from JavaScript or TypeScript, the comparisons below map Rust concepts to familiar JS ideas (mostly Express middleware).

## Contents

1. [What middleware is for](#1-what-middleware-is-for)
2. [Tower: `Service` and `Layer`](#2-tower-service-and-layer)
3. [Adding layers in Axum](#3-adding-layers-in-axum)
4. [Layer order](#4-layer-order)
5. [`layer` vs `route_layer`](#5-layer-vs-route_layer)
6. [Writing middleware with `from_fn`](#6-writing-middleware-with-from_fn)
7. [Passing data to handlers with extensions](#7-passing-data-to-handlers-with-extensions)
8. [tower-http: ready-made layers](#8-tower-http-ready-made-layers)
9. [Logging with `tracing`](#9-logging-with-tracing)
10. [Middleware or extractor?](#10-middleware-or-extractor)
11. [How this looks in a production service](#11-how-this-looks-in-a-production-service)
12. [Putting it all together](#12-putting-it-all-together)
13. [Exercises](#13-exercises)
14. [Further reading](#14-further-reading)

---

## 1. What middleware is for

Some work applies to **many** routes, not one:

| Concern | Example |
|---|---|
| Observability | Log every request with method, path, status, and duration |
| Request IDs | Give each request an ID, return it in a header, put it in every log line |
| Safety limits | Timeouts, maximum body size, panic recovery |
| Browser rules | CORS headers |
| Performance | Response compression |
| Access | Reject requests without a valid session before any handler runs |

Writing this in every handler would be repetitive and easy to forget. **Middleware** wraps the router (or part of it) and runs around every request.

> **JS comparison:** Express middleware is `app.use((req, res, next) => { ...; next(); })`. Axum middleware has the same idea, with two differences: it **returns** the response instead of calling `res.send`, and it can run code **after** the handler by awaiting `next.run(req)` and then changing the response.

---

## 2. Tower: `Service` and `Layer`

Axum doesn't have its own middleware system. It uses **Tower**, a library shared by Axum, Hyper, Tonic (gRPC), and others. Two traits matter.

### 2.1 `Service`: request in, future of response out

Simplified:

```rust
trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn call(&mut self, req: Request) -> Self::Future;
}
```

Everything is a service: a handler, a `Router`, and a router wrapped in middleware. Stage 2 used this when calling a router with `oneshot`.

### 2.2 `Layer`: wraps one service in another

```rust
trait Layer<S> {
    type Service;
    fn layer(&self, inner: S) -> Self::Service;
}
```

A layer takes a service and returns a **new service** that does something before and/or after calling the inner one. Wrapping layers inside each other builds an onion:

```text
request ──▶ Trace ──▶ Timeout ──▶ Auth ──▶ handler
response ◀── Trace ◀── Timeout ◀── Auth ◀──┘
```

You rarely implement these traits by hand. You use ready-made layers (`tower-http`) or write a function with `axum::middleware::from_fn` (section 6).

---

## 3. Adding layers in Axum

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
tower = { version = "0.5", features = ["util"] }
tower-http = { version = "0.6", features = ["trace", "cors", "timeout", "limit", "request-id", "catch-panic", "compression-gzip"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

```rust
use std::time::Duration;
use axum::{routing::get, Router};
use tower_http::{timeout::TimeoutLayer, trace::TraceLayer};

let app = Router::new()
    .route("/projects", get(list_projects))
    .layer(TimeoutLayer::new(Duration::from_secs(10)))
    .layer(TraceLayer::new_for_http());
```

Several layers at once, with `tower::ServiceBuilder`:

```rust
use tower::ServiceBuilder;

let app = Router::new()
    .route("/projects", get(list_projects))
    .layer(
        ServiceBuilder::new()
            .layer(TraceLayer::new_for_http())
            .layer(TimeoutLayer::new(Duration::from_secs(10))),
    );
```

A layer only applies to routes added **before** it. Routes added after a `.layer(...)` call aren't wrapped.

---

## 4. Layer order

This is the most common source of middleware bugs. The two ways of adding layers run in **opposite orders**.

### 4.1 Chained `.layer()` calls: the last one runs first

```rust
Router::new()
    .route("/", get(handler))
    .layer(a)   // inner
    .layer(b)   // wraps a
    .layer(c);  // outermost: runs first
```

```text
request:  c → b → a → handler
response: handler → a → b → c
```

### 4.2 `ServiceBuilder`: top to bottom

```rust
.layer(
    ServiceBuilder::new()
        .layer(c)   // outermost: runs first
        .layer(b)
        .layer(a),  // innermost
)
```

Same order as above, but easier to read. That's why production code usually uses `ServiceBuilder` for the main stack.

### 4.3 What should go where

| Position | Layer | Why |
|---|---|---|
| Outermost | Request ID | Everything after it, including logs, can use the ID |
| | Trace | Logs every request, including ones later layers reject |
| | Catch panic | Turns a panic anywhere inside into a 500 |
| | Timeout | Measures the full time spent inside |
| | CORS | Must answer preflight `OPTIONS` before auth rejects it |
| | Body limit | Rejects huge bodies before anyone reads them |
| Innermost | Auth (`route_layer`) | Only for matched routes (section 5) |

---

## 5. `layer` vs `route_layer`

```rust
let app = Router::new()
    .route("/projects", get(list_projects))
    .route_layer(middleware::from_fn(require_auth))
    .fallback(not_found);
```

| | `.layer(...)` | `.route_layer(...)` |
|---|---|---|
| Runs for matched routes | Yes | Yes |
| Runs for unmatched paths (fallback / 404) | Yes | **No** |

Use `route_layer` for **auth**. With `layer`, a request to a path that doesn't exist gets **401** instead of **404**, which is confusing and tells attackers less than they'd like but also misleads your own clients.

### 5.1 Public and protected routes

Apply auth only to the routes that need it, then merge:

```rust
let public = Router::new()
    .route("/health", get(health))
    .route("/auth/login", post(login));

let protected = Router::new()
    .route("/projects", get(list_projects).post(create_project))
    .route_layer(middleware::from_fn_with_state(state.clone(), require_auth));

let app = Router::new()
    .merge(public)
    .merge(protected)
    .layer(TraceLayer::new_for_http())   // applies to both
    .with_state(state);
```

---

## 6. Writing middleware with `from_fn`

`axum::middleware::from_fn` turns an `async fn` into a layer. It's the easiest way to write your own middleware.

### 6.1 The shape

```rust
use axum::{extract::Request, middleware::Next, response::Response};

async fn my_middleware(req: Request, next: Next) -> Response {
    // 1. before: look at or change the request
    let response = next.run(req).await;   // 2. call everything inside
    // 3. after: look at or change the response
    response
}

let app = Router::new()
    .route("/", get(handler))
    .layer(axum::middleware::from_fn(my_middleware));
```

### 6.2 Timing every request

```rust
use std::time::Instant;

async fn timing(req: Request, next: Next) -> Response {
    let method = req.method().clone();
    let path = req.uri().path().to_owned();
    let start = Instant::now();

    let response = next.run(req).await;

    println!("{method} {path} -> {} in {:?}", response.status().as_u16(), start.elapsed());
    response
}
```

`method` and `path` are copied **before** `next.run(req)` because `req` is moved into it (Stage 0: ownership).

### 6.3 Short-circuiting: returning early

Middleware can answer without calling `next`. The return type can be anything that implements `IntoResponse`, including `Result<Response, AppError>` (Stage 4):

```rust
async fn require_json(req: Request, next: Next) -> Result<Response, AppError> {
    let is_json = req
        .headers()
        .get(header::CONTENT_TYPE)
        .and_then(|v| v.to_str().ok())
        .is_some_and(|v| v.starts_with("application/json"));

    if req.method() == Method::POST && !is_json {
        return Err(AppError::BadRequest("expected application/json".into()));
    }
    Ok(next.run(req).await)
}
```

### 6.4 Extractors in middleware

Middleware functions can take extractors **before** the `Request` argument, and `from_fn_with_state` gives them the state:

```rust
use axum::extract::State;

async fn require_api_key(
    State(state): State<AppState>,
    req: Request,
    next: Next,
) -> Result<Response, AppError> {
    let ok = req
        .headers()
        .get("x-api-key")
        .and_then(|v| v.to_str().ok())
        .is_some_and(|key| key == state.config.api_key.expose());

    if !ok {
        return Err(AppError::Unauthorized);
    }
    Ok(next.run(req).await)
}

let protected = Router::new()
    .route("/projects", get(list_projects))
    .route_layer(axum::middleware::from_fn_with_state(state.clone(), require_api_key));
```

Rules:
- The **`Request`** must be the **last** argument (like a body extractor in Stage 3) and **`Next`** comes after it.
- Only `FromRequestParts` extractors may come before it, because the body is still inside the `Request`.

### 6.5 Changing the response

```rust
async fn add_cache_header(req: Request, next: Next) -> Response {
    let mut response = next.run(req).await;
    response
        .headers_mut()
        .insert(header::CACHE_CONTROL, HeaderValue::from_static("no-store"));
    response
}
```

For simple cases, `axum::middleware::map_request` and `map_response` take a function that only changes the request or only the response.

---

## 7. Passing data to handlers with extensions

Middleware often computes something a handler needs: the request ID, the authenticated user. A request has an **extensions** map, keyed by type, for exactly this.

```rust
#[derive(Clone)]
struct RequestId(String);

async fn assign_request_id(mut req: Request, next: Next) -> Response {
    let id = RequestId(next_id());
    req.extensions_mut().insert(id.clone());     // store it for later layers and handlers

    let mut response = next.run(req).await;
    if let Ok(value) = HeaderValue::from_str(&id.0) {
        response.headers_mut().insert("x-request-id", value);
    }
    response
}

// A handler reads it with the Extension extractor
async fn handler(Extension(id): Extension<RequestId>) -> String {
    format!("request {}", id.0)
}
```

- The key is the **type**. Use a newtype (`RequestId`) rather than a plain `String`, or two middlewares inserting a `String` overwrite each other.
- The value must be `Clone + Send + Sync + 'static`.
- If no middleware inserted it, `Extension<RequestId>` fails at **runtime** with a 500. That's the trade-off from Stage 5: state is checked at compile time, extensions aren't. Use `Option<Extension<T>>` if it may be missing.

> **JS comparison:** this is `req.requestId = ...` in Express, except the type system knows what's stored, and reading something that was never set is an explicit error instead of `undefined`.

---

## 8. tower-http: ready-made layers

`tower-http` provides most of what a service needs. Each layer is behind a cargo feature (see section 3).

### 8.1 `TraceLayer`

```rust
use tower_http::trace::TraceLayer;

.layer(TraceLayer::new_for_http())
```

It opens a `tracing` span per request and logs when the response is sent and when errors happen. Section 9 shows how to see the output.

### 8.2 Request IDs

```rust
use tower_http::request_id::{MakeRequestUuid, PropagateRequestIdLayer, SetRequestIdLayer};
use axum::http::HeaderName;

let x_request_id = HeaderName::from_static("x-request-id");

ServiceBuilder::new()
    .layer(SetRequestIdLayer::new(x_request_id.clone(), MakeRequestUuid))  // set if missing
    .layer(TraceLayer::new_for_http())
    .layer(PropagateRequestIdLayer::new(x_request_id))                     // copy to the response
```

`MakeRequestUuid` needs `tower-http`'s request-id support plus `uuid`. If a client or load balancer already sent an `x-request-id`, it's kept, so one ID can follow a request through several services.

### 8.3 Timeouts

```rust
use tower_http::timeout::TimeoutLayer;

.layer(TimeoutLayer::new(Duration::from_secs(10)))
```

If the inner service takes longer, the request gets **408 Request Timeout** and the handler's future is **dropped** (cancelled, Stage 1, section 6.3). An open transaction inside it rolls back (Stage 6).

Newer tower-http versions also offer a constructor that lets you choose the status code (for example 503 or 504). Check the docs for the version you use.

### 8.4 Body size limits

```rust
use axum::extract::DefaultBodyLimit;
use tower_http::limit::RequestBodyLimitLayer;

Router::new()
    .route("/upload", post(upload))
    .layer(DefaultBodyLimit::disable())                    // turn off Axum's 2 MB default here
    .layer(RequestBodyLimitLayer::new(50 * 1024 * 1024));  // and enforce 50 MB instead
```

- `DefaultBodyLimit::max(n)` changes the limit used by Axum's body extractors (`Json`, `Bytes`, `String`).
- `RequestBodyLimitLayer` enforces a limit on the raw body for everything inside it, including streaming handlers.
- Too large → **413 Payload Too Large**.

### 8.5 CORS

Browsers block a page on `https://app.example.com` from calling `https://api.example.com` unless the API sends **CORS** headers allowing it.

```rust
use axum::http::{header, HeaderValue, Method};
use tower_http::cors::CorsLayer;

let cors = CorsLayer::new()
    .allow_origin(HeaderValue::from_static("http://localhost:5173"))
    .allow_methods([Method::GET, Method::POST, Method::PUT, Method::DELETE])
    .allow_headers([header::CONTENT_TYPE])
    .allow_credentials(true);   // needed when the browser sends cookies (Stage 8)
```

- Take the allowed origin from **config** (`CORS_ORIGIN`, Stage 5), not from code.
- With `allow_credentials(true)`, a wildcard origin (`Any`) is **not allowed**. Browsers reject it, and tower-http panics at startup if you combine them. List real origins.
- `CorsLayer::permissive()` is fine for a local experiment, never for a service that uses cookies.

> **JS comparison:** this is the `cors` package: `app.use(cors({ origin: "http://localhost:5173", credentials: true }))`.

### 8.6 Panic recovery

```rust
use tower_http::catch_panic::CatchPanicLayer;

.layer(CatchPanicLayer::new())
```

A panic in a handler normally drops the connection. With this layer the client gets a **500** instead, and the panic still shows in logs. This is a safety net, not a substitute for the no-`unwrap()` rule (Stage 4, section 9.3).

### 8.7 Compression

```rust
use tower_http::compression::CompressionLayer;

.layer(CompressionLayer::new())
```

Compresses responses when the client sends `Accept-Encoding: gzip`. JSON compresses very well, often to a fifth of its size.

---

## 9. Logging with `tracing`

`println!` was a placeholder. Production services use **`tracing`**: structured logs with fields, levels, and **spans** that group everything that happened during one request.

### 9.1 Setup in `main`

```rust
use tracing_subscriber::EnvFilter;

fn init_tracing() {
    tracing_subscriber::fmt()
        .with_env_filter(
            EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| EnvFilter::new("info,tower_http=debug")),
        )
        .init();
}
```

`RUST_LOG=debug cargo run` changes the level without recompiling. Add `.json()` (feature `json`) for JSON lines that log platforms can index.

### 9.2 Events and fields

```rust
use tracing::{debug, error, info, warn};

info!("server started");
info!(port = config.port, "listening");              // structured field
warn!(project_id = %id, "project not found");         // % = use Display
error!(error = ?err, "database query failed");        // ? = use Debug
```

> **JS comparison:** similar to `pino` or `winston` with `logger.info({ projectId }, "message")`: fields are data, not text inside the message.

### 9.3 Spans and `#[instrument]`

A span is a named scope. Every event inside it carries its fields automatically:

```rust
use tracing::instrument;

#[instrument(skip(pool), fields(tenant_id = %tenant_id.0))]
async fn rename_project(pool: &PgPool, tenant_id: TenantId, id: ProjectId, name: &str) -> Result<(), DomainError> {
    info!("renaming");      // logged with tenant_id, id, and name
    Ok(())
}
```

- `skip(pool)` leaves out arguments that are large or can't be printed.
- **Never record secrets or personal data** as fields: skip passwords, tokens, and request bodies. The `Secret` type from Stage 5 helps, because its `Debug` is redacted.

`TraceLayer` creates the outer span per request. Spans from `#[instrument]` functions nest inside it, so one request produces a tree of log lines you can filter by request ID.

### 9.4 Logging internal errors

Stage 4's `IntoResponse for AppError` used `eprintln!`. Replace it:

```rust
if let AppError::Internal(source) = &self {
    tracing::error!(error = ?source, "internal error");
}
```

Because it runs inside the request's span, the log line automatically includes the request's method, path, and ID.

---

## 10. Middleware or extractor?

Both can reject a request before the handler runs. Choose by what you need:

| Need | Use |
|---|---|
| Applies to every request, the handler doesn't need the result (logging, timeouts, CORS) | Middleware |
| Must change the **response** (headers, compression) | Middleware |
| The handler needs the result (the current user, a tenant) | **Extractor** (Stage 3) |
| Some routes in a group need it | `route_layer` on that group |
| Each handler should declare it in its signature | Extractor |

For authentication, many services use **both**: an `AuthUser` extractor that handlers declare (clear, typed, checked per handler), plus a `route_layer` on protected groups as a safety net in case someone forgets the argument. Stage 8 builds this.

---

## 11. How this looks in a production service

```text
crates/api/src/
├── main.rs              # init_tracing(), build state, routes::router(state), serve
├── middleware/
│   ├── mod.rs
│   ├── request_id.rs
│   └── auth.rs          # require_auth (route_layer), used with the AuthUser extractor
└── routes/mod.rs        # public + protected routers, the ServiceBuilder stack
```

```rust
pub fn router(state: AppState) -> Router {
    let public = Router::new()
        .route("/health", get(health::check))
        .route("/auth/login", post(auth::login));

    let protected = Router::new()
        .merge(projects::router())
        .merge(users::router())
        .route_layer(middleware::from_fn_with_state(state.clone(), auth::require_auth));

    Router::new()
        .nest("/api", public.merge(protected))
        .fallback(not_found)
        .layer(
            ServiceBuilder::new()
                .layer(SetRequestIdLayer::x_request_id(MakeRequestUuid))
                .layer(TraceLayer::new_for_http())
                .layer(PropagateRequestIdLayer::x_request_id())
                .layer(CatchPanicLayer::new())
                .layer(TimeoutLayer::new(state.config.request_timeout))
                .layer(cors_layer(&state.config))
                .layer(CompressionLayer::new()),
        )
        .with_state(state)
}
```

Conventions:
- **One `ServiceBuilder` stack** for global concerns, read top to bottom.
- **`route_layer`** for auth, applied to the protected group only.
- Timeout, CORS origins, and body limits come from **config**.
- `tracing`, not `println!`. Internal errors are logged once, in `AppError::into_response`.
- Long-running routes (WebSockets, streaming, large uploads) get their own limits, or are excluded from the global timeout (Stages 9 and 10).

---

## 12. Putting it all together

A small API with a hand-written request-ID middleware, a timing middleware, an API-key check using `from_fn_with_state` as a `route_layer`, an `Extension` read by a handler, and tower-http's timeout, body limit, and panic recovery. It calls itself with `oneshot`, so it can run **in the Playground** if the Playground includes `tower-http`; otherwise run it as a local project.

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
tower = { version = "0.5", features = ["util"] }
tower-http = { version = "0.6", features = ["timeout", "limit", "catch-panic"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "2"
```

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use std::time::{Duration, Instant};

use axum::{
    body::{to_bytes, Body},
    extract::{DefaultBodyLimit, Request, State},
    http::{header, HeaderValue, StatusCode},
    middleware::{self, Next},
    response::{IntoResponse, Response},
    routing::{get, post},
    Extension, Json, Router,
};
use serde::Serialize;
use tower::{ServiceBuilder, ServiceExt};
use tower_http::{catch_panic::CatchPanicLayer, timeout::TimeoutLayer};

// ================= errors (Stage 4, shortened) =================

#[derive(Debug, thiserror::Error)]
enum AppError {
    #[error("authentication required")]
    Unauthorized,
    #[error("route {0} not found")]
    NotFound(String),
}

#[derive(Serialize)]
struct ErrorBody {
    error: &'static str,
    message: String,
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, code) = match &self {
            AppError::Unauthorized => (StatusCode::UNAUTHORIZED, "unauthorized"),
            AppError::NotFound(_) => (StatusCode::NOT_FOUND, "not_found"),
        };
        (status, Json(ErrorBody { error: code, message: self.to_string() })).into_response()
    }
}

// ================= state =================

#[derive(Clone)]
struct AppState {
    api_key: Arc<str>,
    next_request_id: Arc<AtomicU64>,
}

// ================= middleware =================

#[derive(Clone)]
struct RequestId(String);

/// Outermost: assign an ID, make it available to handlers, return it in a header.
async fn request_id(State(state): State<AppState>, mut req: Request, next: Next) -> Response {
    let n = state.next_request_id.fetch_add(1, Ordering::Relaxed) + 1;
    let id = RequestId(format!("req-{n}"));
    req.extensions_mut().insert(id.clone());

    let mut response = next.run(req).await;
    if let Ok(value) = HeaderValue::from_str(&id.0) {
        response.headers_mut().insert("x-request-id", value);
    }
    response
}

/// Logs method, path, status, and duration. Reads the ID that request_id stored.
async fn timing(req: Request, next: Next) -> Response {
    let method = req.method().clone();
    let path = req.uri().path().to_owned();
    let id = req
        .extensions()
        .get::<RequestId>()
        .map(|r| r.0.clone())
        .unwrap_or_else(|| "-".into());
    let start = Instant::now();

    let response = next.run(req).await;

    let ms = start.elapsed().as_millis();
    let bucket = if ms < 50 { "<50ms" } else { ">=50ms" };   // stable output for the example
    println!("[log] {id} {method} {path} -> {} ({bucket})", response.status().as_u16());
    response
}

/// Only on protected routes (route_layer).
async fn require_api_key(
    State(state): State<AppState>,
    req: Request,
    next: Next,
) -> Result<Response, AppError> {
    let ok = req
        .headers()
        .get("x-api-key")
        .and_then(|v| v.to_str().ok())
        .is_some_and(|k| k == &*state.api_key);

    if !ok {
        return Err(AppError::Unauthorized);
    }
    Ok(next.run(req).await)
}

// ================= handlers =================

async fn health() -> &'static str {
    "ok"
}

async fn whoami(Extension(id): Extension<RequestId>) -> String {
    format!("you are request {}", id.0)
}

async fn slow() -> &'static str {
    tokio::time::sleep(Duration::from_secs(2)).await;
    "finished"   // never reached: the timeout fires first
}

async fn upload(body: axum::body::Bytes) -> String {
    format!("received {} bytes", body.len())
}

async fn boom() -> &'static str {
    let items: Vec<u32> = Vec::new();
    #[expect(clippy::indexing_slicing, reason = "deliberate panic to demonstrate CatchPanicLayer")]
    let _ = items[0];
    "unreachable"
}

async fn not_found(uri: axum::http::Uri) -> AppError {
    AppError::NotFound(uri.path().to_owned())
}

// ================= router =================

fn app(state: AppState) -> Router {
    let public = Router::new()
        .route("/health", get(health))
        .route("/whoami", get(whoami));

    let protected = Router::new()
        .route("/slow", get(slow))
        .route("/upload", post(upload))
        .route("/boom", get(boom))
        .route_layer(middleware::from_fn_with_state(state.clone(), require_api_key));

    Router::new()
        .merge(public)
        .merge(protected)
        .fallback(not_found)
        .layer(DefaultBodyLimit::max(1024))   // 1 KB, tiny on purpose
        .layer(
            ServiceBuilder::new()
                .layer(middleware::from_fn_with_state(state.clone(), request_id)) // outermost
                .layer(middleware::from_fn(timing))
                .layer(CatchPanicLayer::new())
                .layer(TimeoutLayer::new(Duration::from_millis(100))),            // innermost
        )
        .with_state(state)
}

// ================= tiny client =================

async fn call(
    app: &Router,
    method: &str,
    uri: &str,
    api_key: Option<&str>,
    body: Option<Vec<u8>>,
) -> (StatusCode, Option<String>, String) {
    let mut builder = Request::builder().method(method).uri(uri);
    if let Some(key) = api_key {
        builder = builder.header("x-api-key", key);
    }
    let body = match body {
        Some(bytes) => {
            builder = builder.header(header::CONTENT_TYPE, "application/octet-stream");
            Body::from(bytes)
        }
        None => Body::empty(),
    };
    let request = builder.body(body).expect("static request is valid");

    let response = match app.clone().oneshot(request).await {
        Ok(r) => r,
        Err(never) => match never {},
    };
    let status = response.status();
    let request_id = response
        .headers()
        .get("x-request-id")
        .and_then(|v| v.to_str().ok())
        .map(str::to_owned);
    let bytes = to_bytes(response.into_body(), usize::MAX).await.unwrap_or_default();
    (status, request_id, String::from_utf8_lossy(&bytes).into_owned())
}

#[tokio::main]
async fn main() {
    let state = AppState {
        api_key: Arc::from("secret-key"),
        next_request_id: Arc::new(AtomicU64::new(0)),
    };
    let app = app(state);
    let key = Some("secret-key");

    let cases: [(&str, &str, Option<&str>, Option<Vec<u8>>); 7] = [
        ("GET", "/health", None, None),
        ("GET", "/whoami", None, None),
        ("GET", "/slow", None, None),
        ("GET", "/slow", key, None),
        ("POST", "/upload", key, Some(vec![0u8; 4096])),
        ("GET", "/boom", key, None),
        ("GET", "/nope", None, None),
    ];

    for (method, uri, api_key, body) in cases {
        let (status, id, text) = call(&app, method, uri, api_key, body).await;
        let short: String = text.chars().take(60).collect();
        println!(
            "{method:<4} {uri:<8} -> {} [{}] {short}\n",
            status.as_u16(),
            id.as_deref().unwrap_or("-")
        );
    }
}
```

Expected output (bodies for 408, 413, and 500 come from Axum and tower-http, and their wording can differ between versions):

```text
[log] req-1 GET /health -> 200 (<50ms)
GET  /health   -> 200 [req-1] ok

[log] req-2 GET /whoami -> 200 (<50ms)
GET  /whoami   -> 200 [req-2] you are request req-2

[log] req-3 GET /slow -> 401 (<50ms)
GET  /slow     -> 401 [req-3] {"error":"unauthorized","message":"authentication required"}

[log] req-4 GET /slow -> 408 (>=50ms)
GET  /slow     -> 408 [req-4] 

[log] req-5 POST /upload -> 413 (<50ms)
POST /upload   -> 413 [req-5] Failed to buffer the request body: length limit exceeded

[log] req-6 GET /boom -> 500 (<50ms)
GET  /boom     -> 500 [req-6] {"error":{"code":500,"message":"Service panicked"...

[log] req-7 GET /nope -> 404 (<50ms)
GET  /nope     -> 404 [req-7] {"error":"not_found","message":"route /nope not found"}
```

What each case shows:

| Case | Concept |
|---|---|
| Every line has `[req-N]` and a `[log]` line | Outer `from_fn` layers run for **every** request, including errors and 404s |
| `/whoami` | A handler reads the `RequestId` that middleware put into the **extensions** |
| `/slow` without a key | `route_layer` auth short-circuits with `AppError` → 401; the handler never runs |
| `/slow` with a key | `TimeoutLayer` cancels the handler → 408 |
| `/upload` with 4 KB | `DefaultBodyLimit` → 413 |
| `/boom` | `CatchPanicLayer` turns a panic into a 500 instead of a dropped connection |
| `/nope` | The fallback is **not** behind `route_layer`, so it's 404, not 401 |

The panic also prints Rust's usual panic message to stderr; that's expected. Note that the 500 body from `CatchPanicLayer` doesn't use the `AppError` shape. `CatchPanicLayer::custom(handler)` lets you return your own JSON error instead (exercise 6).

---

## 13. Exercises

- [ ] **1. Order:** write two `from_fn` middlewares that print `"A before"`/`"A after"` and `"B before"`/`"B after"`. Add them with two chained `.layer()` calls, then with one `ServiceBuilder`, and predict the order each time before running it.
- [ ] **2. `layer` vs `route_layer`:** add an auth middleware with `.layer(...)` and call a path that doesn't exist. Then switch to `.route_layer(...)` and call it again. Write down both status codes.
- [ ] **3. Extensions:** write a middleware that reads an `x-tenant-id` header into a `TenantId` newtype extension, and a handler that reads it with `Extension<TenantId>`. Then remove the middleware and see what the handler returns.
- [ ] **4. Response headers:** write a middleware that adds `x-response-time-ms` to every response. Use `map_response` for a second, simpler version that adds a fixed `x-service: projects-api` header.
- [ ] **5. CORS:** add a `CorsLayer` that allows `http://localhost:5173` with credentials. Send an `OPTIONS` preflight request with `Origin` and `Access-Control-Request-Method` headers through `oneshot` and print the response headers. Then try `.allow_origin(Any)` with `.allow_credentials(true)` and read what happens.
- [ ] **6. Consistent panic errors:** use `CatchPanicLayer::custom` so a panic returns `{"error":"internal","message":"internal server error"}` like every other 500.
- [ ] **7. tracing:** replace the `println!` calls with `tracing`, add `TraceLayer::new_for_http()`, initialise `tracing_subscriber::fmt()` with `EnvFilter`, and run with `RUST_LOG=debug` and `RUST_LOG=warn`. Add `#[instrument]` to one handler and see its span nested in the request span.
- [ ] **8. Explain it:** in two or three sentences, explain why the request-ID layer must be outside the trace layer, and when you'd use an extractor instead of middleware for something that rejects requests.

> When you can solve exercises 1–8 without looking anything up, you're ready for **Stage 8: Authentication and authorization**.

---

## 14. Further reading

- [`axum::middleware` docs](https://docs.rs/axum/latest/axum/middleware/index.html): `from_fn`, `from_fn_with_state`, `map_request`, `map_response`, and ordering
- [Axum docs: "Ordering"](https://docs.rs/axum/latest/axum/middleware/index.html#ordering): the onion model with diagrams
- [`tower-http` docs](https://docs.rs/tower-http): every layer and its feature flag
- [`tracing` docs](https://docs.rs/tracing) and [`tracing-subscriber`](https://docs.rs/tracing-subscriber): spans, fields, `EnvFilter`, JSON output
- [Tokio: "Inventing the Service trait"](https://tokio.rs/blog/2021-05-14-inventing-the-service-trait): why Tower looks the way it does
- [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS): preflight requests and credentials
