# Stage 5: State and Configuration

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> If you come from JavaScript or TypeScript, the comparisons below map Rust concepts to familiar JS ideas (mostly Express and Node).

## Contents

1. [The problem: sharing things between requests](#1-the-problem-sharing-things-between-requests)
2. [`AppState` and the `State` extractor](#2-appstate-and-the-state-extractor)
3. [Why `Clone`, and what `Arc` is for](#3-why-clone-and-what-arc-is-for)
4. [`Router<S>` and `with_state`](#4-routers-and-with_state)
5. [Substates with `FromRef`](#5-substates-with-fromref)
6. [Using state inside custom extractors](#6-using-state-inside-custom-extractors)
7. [Mutable shared state](#7-mutable-shared-state)
8. [Typed configuration from the environment](#8-typed-configuration-from-the-environment)
9. [Secrets](#9-secrets)
10. [Startup: building everything in `main`](#10-startup-building-everything-in-main)
11. [How this looks in a production service](#11-how-this-looks-in-a-production-service)
12. [Putting it all together](#12-putting-it-all-together)
13. [Exercises](#13-exercises)
14. [Further reading](#14-further-reading)

---

## 1. The problem: sharing things between requests

Handlers need things that live longer than one request:

| Thing | Example | Created |
|---|---|---|
| Database pool | `sqlx::PgPool` | Once at startup |
| Configuration | JWT secret, token lifetime, feature flags | Once at startup |
| HTTP clients | `reqwest::Client` for a downstream service | Once at startup |
| In-memory data | Presence map, token blocklist, cache | Once, then changed at runtime |

Each request runs in its own Tokio task, possibly on a different thread (Stage 1). So the shared data has to be **owned by many tasks at once** and **safe to use from several threads**.

> **JS comparison:** in Node you'd put `const pool = new Pool()` at the top of a module and import it everywhere, or attach it to `app.locals`. A module-level global works because Node is single-threaded. Rust can do globals too (section 3.3), but the idiomatic way is to **pass state explicitly** so every handler declares what it uses, and tests can swap it out.

---

## 2. `AppState` and the `State` extractor

```rust
use std::sync::Arc;
use axum::{extract::State, routing::get, Router};

#[derive(Clone)]
struct AppState {
    app_name: Arc<str>,
    max_page_size: u32,
}

async fn info(State(state): State<AppState>) -> String {
    format!("{} (max page size {})", state.app_name, state.max_page_size)
}

fn app() -> Router {
    let state = AppState { app_name: Arc::from("projects-api"), max_page_size: 100 };

    Router::new()
        .route("/info", get(info))
        .with_state(state)
}
```

Three pieces:

1. **A struct** holding everything that's shared. `#[derive(Clone)]` is required.
2. **`.with_state(state)`** gives the router its state.
3. **`State<AppState>`** in a handler argument extracts it. It's a `FromRequestParts` extractor, so it can be in any position before the body; by convention it goes **first**.

Every request gets its own **clone** of `AppState`. Section 3 explains why that is cheap.

> **Compile-time check:** if a handler asks for `State<AppState>` and the router was given a different state type, or none, the code doesn't compile. In Express, forgetting to set `app.locals.db` fails at runtime with `undefined`.

---

## 3. Why `Clone`, and what `Arc` is for

### 3.1 Axum clones the state per request

Axum needs an **owned** `AppState` for each request (handlers take owned arguments, and futures must be `'static`, Stage 1). So it calls `.clone()`. The rule:

> **Cloning `AppState` must be cheap.** It should copy a few pointers, never real data.

| Field type | What `.clone()` does | Cost |
|---|---|---|
| `u32`, `bool` | Copies the value | Tiny |
| `Arc<T>` | Increments a counter | Tiny |
| `PgPool`, `reqwest::Client` | Already an `Arc` inside | Tiny |
| `String`, `Vec<T>`, `HashMap<K, V>` | **Copies everything** | Grows with the data |

So any field that holds real data is wrapped in `Arc`:

```rust
#[derive(Clone)]
struct AppState {
    pool: sqlx::PgPool,                        // cheap: Arc inside
    config: Arc<Config>,                       // cheap: one counter increment
    blocklist: Arc<RwLock<HashSet<String>>>,   // cheap, and mutable (section 7)
}
```

`Arc<str>` instead of `String` for a read-only string follows the same idea: one allocation shared by all clones.

### 3.2 One `Arc` around everything?

A common alternative is to put the whole state behind one `Arc`:

```rust
struct Inner {
    pool: PgPool,
    config: Config,
}

#[derive(Clone)]
struct AppState(Arc<Inner>);
```

Both work. Per-field `Arc`s make it easy to split the state into substates (section 5). One `Arc` keeps `Inner` simple and makes every clone a single counter increment. Pick one style per project.

### 3.3 Why not a global?

```rust
use std::sync::OnceLock;

static CONFIG: OnceLock<Config> = OnceLock::new();
```

`OnceLock` (or `LazyLock`) is fine for truly global, never-changing things like a compiled regex. For the database pool and config, explicit state is better:

- Handlers **declare** their dependencies in their signature.
- Tests build an `AppState` with a test database and different settings, **in parallel**, without fighting over one global.
- Nothing is initialized implicitly on first use, so startup failures happen at startup.

---

## 4. `Router<S>` and `with_state`

Stage 2 showed `Router<AppState>` in the production layout. Here is what the type parameter means.

`Router<S>` reads as **"a router that still needs a state of type `S`"**.

```rust
fn project_routes() -> Router<AppState> {      // still missing AppState
    Router::new()
        .route("/projects", get(list_projects))
}

fn app(state: AppState) -> Router {             // Router<()>: needs nothing
    Router::new()
        .nest("/api", project_routes())
        .with_state(state)                      // provides AppState
}
```

- Routers that need the same state can be **`nest`ed and `merge`d** before the state is provided.
- `.with_state(state)` turns `Router<AppState>` into `Router<()>` (written just `Router`).
- `axum::serve` only accepts a router whose state is fully provided. If you forget `.with_state`, you get a compile error at the `serve` call.

### 4.1 Common mistake: `with_state` too early

```rust
// Problem: this router no longer has AppState in its type...
fn project_routes(state: AppState) -> Router {
    Router::new().route("/projects", get(list_projects)).with_state(state)
}
```

It compiles, but now every module gets its own copy of the state passed separately, and routers can't be combined generically. The convention is: **sub-routers return `Router<AppState>`, and `with_state` is called once, at the top.**

### 4.2 `State` vs `Extension`

Older code often uses `Extension<T>` and `.layer(Extension(value))` for state. It still works, but a missing extension is a **runtime 500**, while a missing `State` is a **compile error**. Use `State` for application state; `Extension` is for values that **middleware** attaches per request (Stage 7).

---

## 5. Substates with `FromRef`

Some handlers or extractors only need part of the state, such as the config. `FromRef` lets them extract just that part:

```rust
use axum::extract::FromRef;

#[derive(Clone)]
struct AppState {
    config: Arc<Config>,
    store: Store,
}

impl FromRef<AppState> for Arc<Config> {
    fn from_ref(state: &AppState) -> Self {
        Arc::clone(&state.config)
    }
}

// This handler can't touch the store; it only sees the config
async fn settings(State(config): State<Arc<Config>>) -> String {
    format!("token lifetime: {} min", config.token_ttl_minutes)
}
```

With the `macros` feature, a derive writes these impls for every field:

```rust
#[derive(Clone, FromRef)]
struct AppState {
    config: Arc<Config>,
    store: Store,
}
```

Each field type must be unique in the struct for this to work (two `Arc<Config>` fields would be ambiguous).

Substates are mostly useful for **reusable extractors**: an extractor that needs only `Arc<Config>` can work in any app whose state provides one (section 6).

---

## 6. Using state inside custom extractors

In Stage 3, `AuthUser` used a fake token because it couldn't reach the JWT secret. Extractors get the state as the second argument of `from_request_parts`. There are two ways to use it.

### 6.1 Tied to one state type (simplest)

```rust
impl FromRequestParts<AppState> for AuthUser {
    type Rejection = AppError;

    async fn from_request_parts(parts: &mut Parts, state: &AppState) -> Result<Self, Self::Rejection> {
        let token = bearer_token(parts).ok_or(AppError::Unauthorized)?;
        // Stage 8: verify the JWT with state.config.jwt_secret
        verify(token, &state.config.jwt_secret).ok_or(AppError::Unauthorized)
    }
}
```

### 6.2 Generic over any state that contains what it needs

```rust
impl<S> FromRequestParts<S> for AuthUser
where
    S: Send + Sync,
    Arc<Config>: FromRef<S>,              // "S can give me an Arc<Config>"
{
    type Rejection = AppError;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let config = Arc::<Config>::from_ref(state);
        let token = bearer_token(parts).ok_or(AppError::Unauthorized)?;
        verify(token, &config.jwt_secret).ok_or(AppError::Unauthorized)
    }
}
```

The generic version can be moved into a shared crate and reused by several services. For a single service, the first version is perfectly fine.

---

## 7. Mutable shared state

`Arc` gives shared **read-only** access. To change data at runtime, add a lock (Stage 0 and Stage 1):

```rust
use std::collections::HashSet;
use std::sync::{Arc, RwLock};

#[derive(Clone, Default)]
struct Blocklist(Arc<RwLock<HashSet<String>>>);

impl Blocklist {
    fn revoke(&self, jti: &str) -> Result<(), AppError> {
        let mut set = self.0.write().map_err(|_| AppError::Internal("blocklist poisoned".into()))?;
        set.insert(jti.to_string());
        Ok(())
    }

    fn is_revoked(&self, jti: &str) -> Result<bool, AppError> {
        let set = self.0.read().map_err(|_| AppError::Internal("blocklist poisoned".into()))?;
        Ok(set.contains(jti))
    }
}
```

Rules:

- **Wrap the lock in a small type with methods.** Handlers call `blocklist.revoke(jti)` and never see the lock, so they can't hold it across an `.await` by accident.
- `std::sync::RwLock` for short, read-heavy access. `tokio::sync` locks only when you must `.await` while holding the lock (Stage 1, section 7).
- **In-memory state is per process.** If the service runs as three replicas, each has its own blocklist and presence map. That's acceptable for presence and caches, but data that must be shared across replicas belongs in the database or something like Redis.
- **Memory must stay bounded.** A map that only grows (for example a blocklist that never removes expired tokens) is a memory leak. Plan for cleanup: expiry timestamps plus a periodic background task (Stage 1, `tokio::spawn`).

---

## 8. Typed configuration from the environment

### 8.1 Why the environment

Production services read configuration from **environment variables** (the [twelve-factor](https://12factor.net/config) approach): the same build runs locally, in CI, and in production, with only the environment changing. Containers and Kubernetes set environment variables natively.

> **JS comparison:** `process.env.PORT` is always `string | undefined`, and you parse it wherever you use it. In Rust you parse **all** of it **once** at startup into a typed struct. If something is missing or invalid, the service refuses to start, with a clear message.

### 8.2 A `Config` struct

```rust
use std::time::Duration;

#[derive(Debug, Clone)]
pub struct Config {
    pub port: u16,
    pub database_url: Secret,          // section 9
    pub jwt_secret: Secret,
    pub token_ttl: Duration,
    pub cors_origin: Option<String>,
}

#[derive(Debug, thiserror::Error)]
pub enum ConfigError {
    #[error("missing required environment variable {0}")]
    Missing(&'static str),

    #[error("environment variable {key} is invalid: {reason}")]
    Invalid { key: &'static str, reason: String },
}
```

### 8.3 Parse from a lookup function, not from `std::env` directly

Reading `std::env::var` inside the parser makes it hard to test: tests would have to set process-wide environment variables, which affects other tests running in parallel (and `set_var` is `unsafe` in Rust 2024). Instead, parse from **any function that looks up a key**:

```rust
impl Config {
    /// Production entry point.
    pub fn from_env() -> Result<Self, ConfigError> {
        Self::from_lookup(|key| std::env::var(key).ok())
    }

    /// Testable: pass any lookup function, for example one backed by a HashMap.
    pub fn from_lookup(get: impl Fn(&str) -> Option<String>) -> Result<Self, ConfigError> {
        let required = |key: &'static str| get(key).ok_or(ConfigError::Missing(key));

        let port = match get("PORT") {
            Some(raw) => raw.parse::<u16>().map_err(|e| ConfigError::Invalid {
                key: "PORT",
                reason: e.to_string(),
            })?,
            None => 8000,                                   // a sensible default
        };

        let jwt_secret = required("JWT_SECRET_KEY")?;
        if jwt_secret.len() < 32 {
            return Err(ConfigError::Invalid {
                key: "JWT_SECRET_KEY",
                reason: "must be at least 32 bytes".into(),
            });
        }

        let ttl_minutes = match get("TOKEN_TTL_MINUTES") {
            Some(raw) => raw.parse::<u64>().map_err(|e| ConfigError::Invalid {
                key: "TOKEN_TTL_MINUTES",
                reason: e.to_string(),
            })?,
            None => 60,
        };

        Ok(Config {
            port,
            database_url: Secret::new(required("DATABASE_URL")?),
            jwt_secret: Secret::new(jwt_secret),
            token_ttl: Duration::from_secs(ttl_minutes * 60),
            cors_origin: get("CORS_ORIGIN"),
        })
    }
}
```

What this gives you:

- **Fail fast:** a missing `DATABASE_URL` stops startup with `missing required environment variable DATABASE_URL`, not a confusing error on the first request.
- **Validate, don't just parse:** the 32-byte minimum for the JWT secret is checked once, here.
- **Defaults are visible** in one place.
- **Typed values:** the rest of the code sees `Duration`, not `"60"`.

### 8.4 `.env` files for local development

```toml
dotenvy = "0.15"
```

```rust
fn main() {
    // Load .env if it exists; real environment variables take precedence.
    let _ = dotenvy::dotenv();
    // ...
}
```

- Commit a **`.env.example`** with every key and fake values. **Never commit `.env`** (it's in `.gitignore`).
- Load `.env` only in `main`, never inside library code.

### 8.5 Crates that do this for you

- [`envy`](https://docs.rs/envy): deserializes environment variables into a `#[derive(Deserialize)]` struct.
- [`config`](https://docs.rs/config): layers files, environment variables, and defaults.

They save code for large configs. A hand-written `from_lookup` is easier to understand, easier to test, and enough for most services.

---

## 9. Secrets

`#[derive(Debug)]` on `Config` means `println!("{config:?}")` or a `tracing` log line would print the database password and JWT secret. Prevent it with a newtype that hides its value:

```rust
#[derive(Clone)]
pub struct Secret(String);

impl Secret {
    pub fn new(value: String) -> Self {
        Self(value)
    }

    /// The only way to read the value. Easy to search for in code review.
    pub fn expose(&self) -> &str {
        &self.0
    }
}

impl std::fmt::Debug for Secret {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.write_str("Secret([REDACTED])")
    }
}
```

```rust
println!("{config:?}");
// Config { port: 8000, database_url: Secret([REDACTED]), jwt_secret: Secret([REDACTED]), ... }
```

- `Secret` deliberately has **no `Display`**, so it can't end up in a `format!` by accident.
- Reading it requires `.expose()`, which makes every use visible.
- The [`secrecy`](https://docs.rs/secrecy) crate provides a ready-made `SecretString` that also zeroes the memory when dropped.

---

## 10. Startup: building everything in `main`

`main` is the one place that uses `anyhow` (Stage 4). It turns configuration into a running server:

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let _ = dotenvy::dotenv();

    let config = Config::from_env().context("invalid configuration")?;

    let pool = PgPoolOptions::new()                        // Stage 6
        .max_connections(10)
        .connect(config.database_url.expose())
        .await
        .context("failed to connect to the database")?;

    let state = AppState {
        pool,
        config: Arc::new(config.clone()),
        blocklist: Blocklist::default(),
    };

    let app = routes::router(state);

    let addr = std::net::SocketAddr::from(([0, 0, 0, 0], config.port));
    let listener = tokio::net::TcpListener::bind(addr).await?;
    axum::serve(listener, app).with_graceful_shutdown(shutdown_signal()).await?;
    Ok(())
}
```

The order matters: **config → connections → state → router → serve**. Each step can fail with a clear message, and nothing serves traffic until everything is ready.

---

## 11. How this looks in a production service

```text
crates/
├── shared/src/config.rs   # Config, ConfigError, Secret, from_env / from_lookup
└── api/src/
    ├── main.rs            # dotenvy, Config::from_env, pool, AppState, serve
    ├── state.rs           # AppState (+ FromRef impls), Blocklist, PresenceMap
    └── routes/            # fn router() -> Router<AppState>, with_state once at the top
```

```rust
// api/src/state.rs
#[derive(Clone)]
pub struct AppState {
    pub pool: PgPool,                     // the single database pool
    pub config: Arc<Config>,              // read-only, shared
    pub blocklist: Blocklist,             // Arc<RwLock<HashSet<..>>> inside
    pub presence: PresenceMap,            // Arc<RwLock<HashMap<..>>> inside (Stage 9)
}
```

Conventions:

- **One `AppState`** for the whole service, cheap to clone.
- **Config is parsed once**, validated, and never read from `std::env` anywhere else.
- **Secrets** are wrapped and never logged.
- **Tests** build their own `AppState` with a test database and `Config::from_lookup` over a `HashMap` (Stage 11).

---

## 12. Putting it all together

A small service with a typed `Config` parsed from a lookup function, a redacted `Secret`, an `AppState` with read-only config and mutable in-memory data, a `FromRef` substate, and an extractor that reads state. It uses the `AppError` pattern from Stage 4 (shortened) and calls itself with `oneshot`, so it runs **in the Playground**.

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
use std::time::Duration;

use axum::{
    body::{to_bytes, Body},
    extract::{FromRef, FromRequestParts, Request, State},
    http::{header, request::Parts, StatusCode},
    response::{IntoResponse, Response},
    routing::{get, post},
    Json, Router,
};
use serde::Serialize;
use tower::ServiceExt;

// ================= config =================

#[derive(Clone)]
struct Secret(String);

impl Secret {
    fn expose(&self) -> &str {
        &self.0
    }
}

impl std::fmt::Debug for Secret {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.write_str("Secret([REDACTED])")
    }
}

#[derive(Debug, Clone)]
struct Config {
    port: u16,
    api_key: Secret,
    max_page_size: u32,
    token_ttl: Duration,
}

#[derive(Debug, thiserror::Error)]
enum ConfigError {
    #[error("missing required environment variable {0}")]
    Missing(&'static str),
    #[error("environment variable {key} is invalid: {reason}")]
    Invalid { key: &'static str, reason: String },
}

fn parse_or<T: std::str::FromStr>(
    get: &impl Fn(&str) -> Option<String>,
    key: &'static str,
    default: T,
) -> Result<T, ConfigError>
where
    T::Err: std::fmt::Display,
{
    match get(key) {
        Some(raw) => raw
            .parse::<T>()
            .map_err(|e| ConfigError::Invalid { key, reason: e.to_string() }),
        None => Ok(default),
    }
}

impl Config {
    fn from_lookup(get: impl Fn(&str) -> Option<String>) -> Result<Self, ConfigError> {
        let api_key = get("API_KEY").ok_or(ConfigError::Missing("API_KEY"))?;
        if api_key.len() < 16 {
            return Err(ConfigError::Invalid {
                key: "API_KEY",
                reason: "must be at least 16 characters".into(),
            });
        }

        let max_page_size = parse_or(&get, "MAX_PAGE_SIZE", 50u32)?;
        if max_page_size == 0 || max_page_size > 500 {
            return Err(ConfigError::Invalid {
                key: "MAX_PAGE_SIZE",
                reason: "must be between 1 and 500".into(),
            });
        }

        Ok(Config {
            port: parse_or(&get, "PORT", 8000u16)?,
            api_key: Secret(api_key),
            max_page_size,
            token_ttl: Duration::from_secs(parse_or(&get, "TOKEN_TTL_MINUTES", 60u64)? * 60),
        })
    }
}

/// Builds a lookup function from a fixed list, standing in for std::env::var.
fn lookup(pairs: &[(&str, &str)]) -> impl Fn(&str) -> Option<String> {
    let map: HashMap<String, String> = pairs
        .iter()
        .map(|(k, v)| (k.to_string(), v.to_string()))
        .collect();
    move |key| map.get(key).cloned()
}

// ================= errors (Stage 4, shortened) =================

#[derive(Debug, thiserror::Error)]
enum AppError {
    #[error("authentication required")]
    Unauthorized,
    #[error("internal server error")]
    Internal(#[source] Box<dyn std::error::Error + Send + Sync>),
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
            AppError::Internal(source) => {
                eprintln!("[log] internal error: {source}");
                (StatusCode::INTERNAL_SERVER_ERROR, "internal")
            }
        };
        (status, Json(ErrorBody { error: code, message: self.to_string() })).into_response()
    }
}

// ================= state =================

#[derive(Clone, Default)]
struct HitCounter(Arc<RwLock<HashMap<String, u64>>>);

impl HitCounter {
    fn hit(&self, path: &str) -> Result<u64, AppError> {
        let mut map = self
            .0
            .write()
            .map_err(|_| AppError::Internal("hit counter poisoned".into()))?;
        let count = map.entry(path.to_string()).or_insert(0);
        *count += 1;
        Ok(*count)
    }

    fn snapshot(&self) -> Result<HashMap<String, u64>, AppError> {
        let map = self
            .0
            .read()
            .map_err(|_| AppError::Internal("hit counter poisoned".into()))?;
        Ok(map.clone())  // small map: copying a snapshot out is fine here
    }
}

#[derive(Clone)]
struct AppState {
    config: Arc<Config>,
    hits: HitCounter,
}

impl FromRef<AppState> for Arc<Config> {
    fn from_ref(state: &AppState) -> Self {
        Arc::clone(&state.config)
    }
}

// ================= extractor that uses state =================

struct ApiClient;

impl<S> FromRequestParts<S> for ApiClient
where
    S: Send + Sync,
    Arc<Config>: FromRef<S>,
{
    type Rejection = AppError;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let config = Arc::<Config>::from_ref(state);
        let provided = parts
            .headers
            .get("x-api-key")
            .and_then(|v| v.to_str().ok())
            .ok_or(AppError::Unauthorized)?;

        if provided == config.api_key.expose() {
            Ok(ApiClient)
        } else {
            Err(AppError::Unauthorized)
        }
    }
}

// ================= handlers =================

#[derive(Serialize)]
struct SettingsDto {
    max_page_size: u32,
    token_ttl_minutes: u64,
}

// Only needs the config: extracts the substate
async fn settings(State(config): State<Arc<Config>>) -> Json<SettingsDto> {
    Json(SettingsDto {
        max_page_size: config.max_page_size,
        token_ttl_minutes: config.token_ttl.as_secs() / 60,
    })
}

// Needs the full state and a valid API key
async fn ping(State(state): State<AppState>, _client: ApiClient) -> Result<String, AppError> {
    let count = state.hits.hit("/ping")?;
    Ok(format!("pong #{count}"))
}

async fn stats(
    State(state): State<AppState>,
    _client: ApiClient,
) -> Result<Json<HashMap<String, u64>>, AppError> {
    Ok(Json(state.hits.snapshot()?))
}

fn api_routes() -> Router<AppState> {
    Router::new()
        .route("/settings", get(settings))
        .route("/ping", post(ping))
        .route("/stats", get(stats))
}

fn app(state: AppState) -> Router {
    Router::new().nest("/api", api_routes()).with_state(state)
}

// ================= tiny client =================

async fn call(app: &Router, method: &str, uri: &str, api_key: Option<&str>) -> (StatusCode, String) {
    let mut builder = Request::builder().method(method).uri(uri);
    if let Some(key) = api_key {
        builder = builder.header("x-api-key", key);
    }
    let request = builder
        .header(header::ACCEPT, "application/json")
        .body(Body::empty())
        .expect("static request is valid");

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
    // 1. Configuration: two bad environments, then a good one
    for env in [
        lookup(&[("PORT", "8080")]),
        lookup(&[("API_KEY", "0123456789abcdef"), ("MAX_PAGE_SIZE", "abc")]),
    ] {
        match Config::from_lookup(env) {
            Ok(_) => println!("config ok"),
            Err(e) => println!("config error: {e}"),
        }
    }

    let config = match Config::from_lookup(lookup(&[
        ("API_KEY", "0123456789abcdef"),
        ("MAX_PAGE_SIZE", "100"),
    ])) {
        Ok(c) => c,
        Err(e) => {
            println!("config error: {e}");
            return;
        }
    };
    println!("loaded: {config:?}");

    // 2. State and router
    let state = AppState { config: Arc::new(config), hits: HitCounter::default() };
    let app = app(state);

    // 3. Requests
    let key = Some("0123456789abcdef");
    let cases: [(&str, &str, Option<&str>); 6] = [
        ("GET", "/api/settings", None),
        ("POST", "/api/ping", None),
        ("POST", "/api/ping", Some("wrong-key")),
        ("POST", "/api/ping", key),
        ("POST", "/api/ping", key),
        ("GET", "/api/stats", key),
    ];

    for (method, uri, api_key) in cases {
        let (status, body) = call(&app, method, uri, api_key).await;
        let who = match api_key {
            None => "no key",
            Some(k) if k.len() == 16 => "valid key",
            Some(_) => "bad key",
        };
        println!("{method:<4} {uri:<14} {who:<9} -> {} {body}", status.as_u16());
    }
}
```

Expected output:

```text
config error: missing required environment variable API_KEY
config error: environment variable MAX_PAGE_SIZE is invalid: invalid digit found in string
loaded: Config { port: 8000, api_key: Secret([REDACTED]), max_page_size: 100, token_ttl: 3600s }
GET  /api/settings  no key    -> 200 {"max_page_size":100,"token_ttl_minutes":60}
POST /api/ping      no key    -> 401 {"error":"unauthorized","message":"authentication required"}
POST /api/ping      bad key   -> 401 {"error":"unauthorized","message":"authentication required"}
POST /api/ping      valid key -> 200 pong #1
POST /api/ping      valid key -> 200 pong #2
GET  /api/stats     valid key -> 200 {"/ping":2}
```

What each part shows:

| Part | Concept |
|---|---|
| The first two config lines | Fail fast: a missing key and an invalid value are reported at startup, with the key name |
| `loaded: ...` | `Debug` prints the config, but the `Secret` is redacted |
| `/api/settings` | `State<Arc<Config>>`: a **substate** via `FromRef` |
| `/api/ping` without or with a bad key | `ApiClient` reads the expected key **from state** inside an extractor |
| `pong #1`, `pong #2` | Mutable shared state behind `Arc<RwLock<..>>`, wrapped in `HitCounter` |
| `/api/stats` | Every request saw the **same** counter, even though each got its own clone of `AppState` |

A note on the comparison `provided == config.api_key.expose()`: comparing secrets with `==` can leak timing information. Real auth code uses a constant-time comparison (for example the `subtle` crate) or, better, signed tokens (Stage 8).

---

## 13. Exercises

- [ ] **1. Cheap clones:** create an `AppState` with a `Vec<String>` of 1,000,000 items, then change the field to `Arc<Vec<String>>`. Time 10,000 `.clone()` calls of each version with `std::time::Instant` and explain the difference.
- [ ] **2. `Router<S>`:** write `fn user_routes() -> Router<AppState>` and `fn admin_routes() -> Router<AppState>`, `nest` them under `/users` and `/admin`, and call `with_state` once. Then remove the `with_state` call and read the compiler error at `oneshot` or `serve`.
- [ ] **3. `FromRef`:** add a `feature_flags: Arc<FeatureFlags>` field to `AppState`, implement `FromRef` for it, and write a handler that takes only `State<Arc<FeatureFlags>>`.
- [ ] **4. Config validation:** add `DATABASE_URL` (required, must start with `postgres://`) and `LOG_LEVEL` (optional, one of `debug`, `info`, `warn`, `error`, default `info`) to `Config::from_lookup`. Test one good and two bad environments.
- [ ] **5. `Secret`:** add `database_url: Secret` to `Config`, print the config with `{:?}`, and confirm the URL isn't shown. Then try to `println!("{}", config.database_url)` and read the compiler error.
- [ ] **6. Mutable state:** build a `Blocklist` with `revoke(jti)` and `is_revoked(jti)`, add it to `AppState`, and write `POST /logout/{jti}` and `GET /check/{jti}` handlers. Make sure no handler sees the lock directly.
- [ ] **7. Bounded memory:** change `Blocklist` to store `(jti, expires_at: Instant)`, and spawn a background task in `main` that removes expired entries every few seconds (`tokio::time::interval`). Explain why the plain `HashSet` version would leak memory in a long-running service.
- [ ] **8. Explain it:** in two or three sentences, explain why explicit `State` is easier to test than a global `OnceLock<PgPool>`, and why in-memory state is a problem when a service runs as several replicas.

> When you can solve exercises 1–8 without looking anything up, you're ready for **Stage 6: Persistence with SQLx**.

---

## 14. Further reading

- [`axum::extract::State` docs](https://docs.rs/axum/latest/axum/extract/struct.State.html): `with_state`, substates, and `FromRef`
- [Axum docs: "Sharing state with handlers"](https://docs.rs/axum/latest/axum/index.html#sharing-state-with-handlers): `State` vs `Extension` vs closures
- [The Twelve-Factor App: Config](https://12factor.net/config): why configuration belongs in the environment
- [`dotenvy` docs](https://docs.rs/dotenvy): loading `.env` files for local development
- [`secrecy` docs](https://docs.rs/secrecy): `SecretString` with redacted `Debug` and zeroing on drop
- *Zero To Production in Rust*, the chapter on configuration: layered config and per-environment settings
