# Extra: Dropshot, a Second HTTP Framework

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> Read this after Stages 2–4. It shows the same ideas (routing, extractors, errors, state) in a framework with a very different philosophy from Axum.

## Contents

1. [Why look at another framework](#1-why-look-at-another-framework)
2. [Axum vs Dropshot at a glance](#2-axum-vs-dropshot-at-a-glance)
3. [Setup](#3-setup)
4. [Endpoints and the `#[endpoint]` macro](#4-endpoints-and-the-endpoint-macro)
5. [Context: Dropshot's version of `AppState`](#5-context-dropshots-version-of-appstate)
6. [Extractors: `Path`, `Query`, `TypedBody`](#6-extractors-path-query-typedbody)
7. [Responses](#7-responses)
8. [Errors: `HttpError`](#8-errors-httperror)
9. [No middleware: cross-cutting concerns as plain functions](#9-no-middleware-cross-cutting-concerns-as-plain-functions)
10. [OpenAPI from the code](#10-openapi-from-the-code)
11. [API traits: separating the interface from the implementation](#11-api-traits-separating-the-interface-from-the-implementation)
12. [Starting the server](#12-starting-the-server)
13. [Putting it all together](#13-putting-it-all-together)
14. [Choosing between them](#14-choosing-between-them)
15. [Exercises](#15-exercises)
16. [Further reading](#16-further-reading)

---

## 1. Why look at another framework

Production codebases don't all use Axum. **Dropshot** (from Oxide Computer) is common in services that are mainly **APIs consumed by generated clients**: control planes, internal platform services, and agent/orchestrator APIs.

Learning a second framework also shows which parts of Stages 2–11 are **Axum-specific** and which are **general backend design**. Tenant scoping, optimistic locking, bounded streaming, and typed errors carry over unchanged. Only the plumbing differs.

---

## 2. Axum vs Dropshot at a glance

| Topic | Axum | Dropshot |
|---|---|---|
| Philosophy | Flexible, composable, built on Tower | Opinionated, API-first, OpenAPI is central |
| Handlers | Any `async fn` whose arguments are extractors | `async fn` annotated with `#[endpoint { method, path }]` |
| First argument | Anything | Always `RequestContext<C>` |
| Shared state | `State<AppState>` + `with_state` | `rqctx.context()` (one context type per server) |
| Routing | `Router::route`, `nest`, `merge` | `ApiDescription::register(endpoint)` |
| Path syntax | `/projects/{id}` | `/projects/{project_id}` (same braces) |
| Path params | `Path<u32>`, tuples, or structs | `Path<Struct>` only (named fields + `JsonSchema`) |
| Body | `Json<T>` | `TypedBody<T>` (`T: Deserialize + JsonSchema`) |
| Responses | Anything `IntoResponse` | Typed: `HttpResponseOk<T>`, `HttpResponseCreated<T>`, ... |
| Errors | Your own `AppError: IntoResponse` | `HttpError` (or a custom error type in newer versions) |
| Middleware | Tower layers | **None by design**: call functions in each handler |
| Logging | `tracing` | `slog` logger on the request context |
| OpenAPI | Add-on (`utoipa`, `aide`) | **Built in**, generated from the endpoints |
| WebSockets | `WebSocketUpgrade` | `#[channel]` endpoints |

> **JS comparison:** Axum feels like Express or Fastify with types. Dropshot feels closer to a schema-first framework such as NestJS with Swagger decorators, or tRPC: the API description is the source of truth, and clients are generated from it.

---

## 3. Setup

```toml
[dependencies]
dropshot = "0.16"
schemars = { version = "0.8", features = ["chrono"] }
semver = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
slog = "2"
tokio = { version = "1", features = ["full"] }
```

`schemars` provides the `JsonSchema` derive. Every type that appears in a request or response must implement it, because Dropshot uses it to generate the OpenAPI document.

Dropshot changes its API between minor versions. Pin the version and check its changelog when upgrading.

---

## 4. Endpoints and the `#[endpoint]` macro

```rust
use dropshot::{endpoint, HttpError, HttpResponseOk, RequestContext};

#[endpoint {
    method = GET,
    path = "/health",
}]
async fn health(_rqctx: RequestContext<()>) -> Result<HttpResponseOk<String>, HttpError> {
    Ok(HttpResponseOk("ok".to_string()))
}
```

Compared with Axum:

- The **method and path live on the function**, not in a router.
- The first argument is always the **`RequestContext`**.
- The return type is always `Result<SomeHttpResponse<T>, HttpError>`, so the macro knows the success status and body schema for the OpenAPI document.
- The macro checks the signature at compile time. A wrong argument type or a missing `JsonSchema` is a compile error.

Doc comments on the endpoint become the operation's description in the OpenAPI document:

```rust
/// Fetch one project by ID.
#[endpoint { method = GET, path = "/projects/{project_id}", tags = ["projects"] }]
async fn get_project(/* ... */) -> Result<HttpResponseOk<ProjectView>, HttpError> { /* ... */ }
```

---

## 5. Context: Dropshot's version of `AppState`

A Dropshot server has **one context type** `C`. Every endpoint receives `RequestContext<C>` and reads it with `rqctx.context()`:

```rust
use std::sync::Arc;

pub struct AppContext {
    pub store: Store,              // Arc<RwLock<..>> inside, or a PgPool
    pub max_page_size: u32,
}

#[endpoint { method = GET, path = "/settings" }]
async fn settings(rqctx: RequestContext<Arc<AppContext>>) -> Result<HttpResponseOk<u32>, HttpError> {
    let ctx = rqctx.context();     // &Arc<AppContext>
    Ok(HttpResponseOk(ctx.max_page_size))
}
```

- The context must be `Send + Sync + 'static`. It's **shared**, not cloned per request, so it doesn't need `Clone` (Axum's `AppState` does).
- `Arc<AppContext>` is the common choice, so background tasks can hold a clone too.
- Stage 5 rules still apply: mutable parts go behind a lock wrapped in methods, and config is parsed once at startup.

The request context also carries per-request data:

| Field / method | Gives you |
|---|---|
| `rqctx.context()` | The shared server context |
| `rqctx.log` | An `slog::Logger` already tagged with the request ID, method, and path |
| `rqctx.request_id` | The request ID string |
| `rqctx.request` | Headers, method, URI (for auth and similar checks) |

So the request-ID and per-request logging work from Stage 7 is built in.

---

## 6. Extractors: `Path`, `Query`, `TypedBody`

```rust
use dropshot::{Path, Query, TypedBody};
use schemars::JsonSchema;
use serde::Deserialize;

#[derive(Deserialize, JsonSchema)]
struct ProjectPath {
    project_id: i32,                 // must match {project_id} in the path
}

#[derive(Deserialize, JsonSchema)]
struct ListParams {
    page: Option<u32>,
    per_page: Option<u32>,
}

#[derive(Deserialize, JsonSchema)]
struct RenameProject {
    name: String,
    version: i32,
}

#[endpoint { method = PUT, path = "/projects/{project_id}" }]
async fn rename_project(
    rqctx: RequestContext<Arc<AppContext>>,
    path: Path<ProjectPath>,
    body: TypedBody<RenameProject>,          // body extractor: last
) -> Result<HttpResponseOk<ProjectView>, HttpError> {
    let id = path.into_inner().project_id;
    let input = body.into_inner();
    /* ... */
}
```

Same ideas as Stage 3, with a few differences:

- **Path parameters must be a struct** with named fields matching the path. No `Path<i32>` or tuples. That's more verbose, but the names appear in the OpenAPI document.
- `.into_inner()` instead of destructuring (`Path(id)`).
- Invalid input is rejected with **400** and a JSON error body automatically. There's no 422/400 split as with Axum's `Json`.
- The body extractor (`TypedBody`, `UntypedBody`, `StreamingBody`) comes **last**, and there can only be one, just like Axum.
- Custom extractors are possible (`SharedExtractor` / `ExclusiveExtractor`), but the Dropshot style is to call plain functions instead (section 9).

---

## 7. Responses

Each response type fixes the **status code** and tells OpenAPI the body schema:

| Type | Status | Use |
|---|---|---|
| `HttpResponseOk<T>` | 200 | Reads and updates that return data |
| `HttpResponseCreated<T>` | 201 | `POST` that creates a resource |
| `HttpResponseAccepted<T>` | 202 | Work started asynchronously |
| `HttpResponseUpdatedNoContent` | 204 | Updates with no body |
| `HttpResponseDeleted` | 204 | Deletes |
| `HttpResponseHeaders<R, H>` | Same as `R` | Add typed headers (for example `Location`) |
| `Response<Body>` | Anything | Escape hatch for raw or streaming responses (not described in OpenAPI) |

`T` must implement `Serialize + JsonSchema`. Keep using separate view/DTO types, never `*Row` structs (Stage 6, section 8.1).

---

## 8. Errors: `HttpError`

`HttpError` already has the split that Stage 4 built by hand:

| Field | Meaning | Stage 4 equivalent |
|---|---|---|
| `status_code` | HTTP status | `AppError::status()` |
| `error_code` | Optional machine-readable code | `ErrorBody.error` |
| `external_message` | Sent to the client | The 4xx message |
| `internal_message` | Logged only | The `source` of `AppError::Internal` |

Constructors (Dropshot 0.13+; check your version):

```rust
HttpError::for_not_found(None, format!("project {id} not found"))
HttpError::for_bad_request(Some("bad_request".into()), "name must not be empty".into())
HttpError::for_client_error(Some("conflict".into()), ClientErrorStatusCode::CONFLICT, msg)
HttpError::for_internal_error(err.to_string())   // client sees a generic message
```

So the Stage 4 domain mapping becomes a `From<DomainError> for HttpError`:

```rust
impl From<DomainError> for HttpError {
    fn from(err: DomainError) -> Self {
        match err {
            DomainError::ProjectNotFound(id) => {
                HttpError::for_not_found(Some("not_found".into()), format!("project {id} not found"))
            }
            e @ DomainError::VersionMismatch { .. } => HttpError::for_client_error(
                Some("conflict".into()),
                ClientErrorStatusCode::CONFLICT,
                e.to_string(),
            ),
            e @ DomainError::EmptyName => HttpError::for_bad_request(Some("bad_request".into()), e.to_string()),
            e @ DomainError::Poisoned => HttpError::for_internal_error(e.to_string()),
        }
    }
}
```

Handlers keep using `?` exactly as in Stage 4. Newer Dropshot versions also let endpoints return **your own** error type, if it implements `HttpResponseError` and `JsonSchema`, so the error shape also appears in the OpenAPI document.

---

## 9. No middleware: cross-cutting concerns as plain functions

Dropshot deliberately has **no middleware layer**. The reasoning: middleware hides behaviour that isn't visible in the handler's signature or the API description. Instead, each handler **calls** what it needs:

```rust
#[endpoint { method = POST, path = "/projects" }]
async fn create_project(
    rqctx: RequestContext<Arc<AppContext>>,
    body: TypedBody<CreateProject>,
) -> Result<HttpResponseCreated<ProjectView>, HttpError> {
    let ctx = rqctx.context();
    let user = authenticate(&rqctx).await?;          // Stage 8: 401
    user.require(caps::PROJECTS_WRITE)?;             // 403
    let project = ctx.store.create(user.tenant_id, body.into_inner())?;
    slog::info!(rqctx.log, "project created"; "project_id" => project.id);
    Ok(HttpResponseCreated(project.into()))
}

async fn authenticate(rqctx: &RequestContext<Arc<AppContext>>) -> Result<AuthUser, HttpError> {
    let header = rqctx
        .request
        .headers()
        .get(http::header::AUTHORIZATION)
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .ok_or_else(|| HttpError::for_client_error(
            Some("unauthorized".into()),
            ClientErrorStatusCode::UNAUTHORIZED,
            "authentication required".into(),
        ))?;
    rqctx.context().verify_token(header)
}
```

How the Stage 7 concerns map:

| Concern | In Dropshot |
|---|---|
| Request IDs, per-request logging | Built in (`rqctx.request_id`, `rqctx.log`) |
| Body size limit | `ConfigDropshot { default_request_body_max_bytes, .. }` |
| Auth | A function called at the top of each handler |
| Timeouts | `tokio::time::timeout` inside the handler, or at the load balancer |
| CORS, compression | Usually a reverse proxy in front of the service |

Forgetting the `authenticate` call is the risk. Code review and tests (Stage 11, section 6.2) cover it, the same way the Axum `route_layer` safety net does.

---

## 10. OpenAPI from the code

Because every endpoint, parameter, body, and response is typed, Dropshot can produce an OpenAPI document with no extra annotations:

```rust
let mut api = ApiDescription::new();
api.register(get_project)?;
api.register(create_project)?;

api.openapi("Projects API", semver::Version::new(1, 0, 0))
    .write(&mut std::io::stdout())?;
```

Typical workflow in production codebases:

1. Generate `openapi/projects.json` from the code with a small binary or test.
2. **Commit** it, and have CI fail if the generated file differs from the committed one (like `sqlx prepare --check` in Stage 6).
3. Generate typed clients from it: [Progenitor](https://github.com/oxidecomputer/progenitor) for Rust, `openapi-typescript` for a TypeScript frontend.

A change to an endpoint then shows up as a diff in the API document during review, so accidental breaking changes are visible.

---

## 11. API traits: separating the interface from the implementation

Newer Dropshot versions can define the API as a **trait**. The trait is the contract, and an implementation provides the behaviour:

```rust
#[dropshot::api_description]
pub trait ProjectsApi {
    type Context;

    /// Fetch one project.
    #[endpoint { method = GET, path = "/projects/{project_id}" }]
    async fn get_project(
        rqctx: RequestContext<Self::Context>,
        path: Path<ProjectPath>,
    ) -> Result<HttpResponseOk<ProjectView>, HttpError>;
}

pub enum ProjectsImpl {}

impl ProjectsApi for ProjectsImpl {
    type Context = Arc<AppContext>;

    async fn get_project(
        rqctx: RequestContext<Self::Context>,
        path: Path<ProjectPath>,
    ) -> Result<HttpResponseOk<ProjectView>, HttpError> {
        /* ... */
    }
}

// Real server:
let api = projects_api_mod::api_description::<ProjectsImpl>()?;
// OpenAPI without compiling the implementation:
let stub = projects_api_mod::stub_api_description()?;
```

The macro generates the `projects_api_mod` module from the trait name. The benefit is that the API crate can be tiny, and both the OpenAPI document and the generated clients can be built without compiling the whole service.

---

## 12. Starting the server

```rust
use dropshot::{ConfigDropshot, ConfigLogging, ConfigLoggingLevel, ServerBuilder};

let log = ConfigLogging::StderrTerminal { level: ConfigLoggingLevel::Info }
    .to_logger("projects-api")
    .map_err(|e| e.to_string())?;

let config = ConfigDropshot {
    bind_address: "127.0.0.1:8080".parse()?,
    default_request_body_max_bytes: 1024 * 1024,
    ..Default::default()
};

let server = ServerBuilder::new(api, Arc::new(context), log)
    .config(config)
    .start()?;

server.await?;     // runs until shut down
```

Older versions use `HttpServerStarter::new(&config, api, ctx, &log)?.start()` instead of `ServerBuilder`. Configuration is often read from a TOML file (`ConfigDropshot` implements `Deserialize`), though the Stage 5 environment approach works too.

---

## 13. Putting it all together

The Stage 4 projects API rebuilt in Dropshot: in-memory store, tenant-free for brevity, typed path and body, version-checked rename, `From<DomainError> for HttpError`, and an OpenAPI document. Run it as a local project.

```toml name=Cargo.toml
[package]
name = "dropshot-projects"
version = "0.1.0"
edition = "2021"

[dependencies]
dropshot = "0.16"
http = "1"
schemars = "0.8"
semver = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
slog = "2"
thiserror = "2"
tokio = { version = "1", features = ["full"] }
```

```rust name=src/main.rs
use std::collections::HashMap;
use std::sync::{Arc, RwLock};

use dropshot::{
    endpoint, ApiDescription, ClientErrorStatusCode, ConfigDropshot, ConfigLogging,
    ConfigLoggingLevel, HttpError, HttpResponseCreated, HttpResponseOk, Path, RequestContext,
    ServerBuilder, TypedBody,
};
use schemars::JsonSchema;
use serde::{Deserialize, Serialize};

// ================= types =================

#[derive(Clone, Serialize, JsonSchema)]
struct ProjectView {
    id: i32,
    name: String,
    version: i32,
}

#[derive(Deserialize, JsonSchema)]
struct ProjectPath {
    project_id: i32,
}

#[derive(Deserialize, JsonSchema)]
struct CreateProject {
    name: String,
}

#[derive(Deserialize, JsonSchema)]
struct RenameProject {
    name: String,
    version: i32,
}

// ================= domain =================

#[derive(Debug, thiserror::Error)]
enum DomainError {
    #[error("project {0} not found")]
    NotFound(i32),
    #[error("version conflict: expected {expected}, found {actual}")]
    VersionMismatch { expected: i32, actual: i32 },
    #[error("name must not be empty")]
    EmptyName,
    #[error("storage lock poisoned")]
    Poisoned,
}

impl From<DomainError> for HttpError {
    fn from(err: DomainError) -> Self {
        match err {
            e @ DomainError::NotFound(_) => {
                HttpError::for_not_found(Some("not_found".into()), e.to_string())
            }
            e @ DomainError::VersionMismatch { .. } => HttpError::for_client_error(
                Some("conflict".into()),
                ClientErrorStatusCode::CONFLICT,
                e.to_string(),
            ),
            e @ DomainError::EmptyName => {
                HttpError::for_bad_request(Some("bad_request".into()), e.to_string())
            }
            e @ DomainError::Poisoned => HttpError::for_internal_error(e.to_string()),
        }
    }
}

#[derive(Default)]
struct Store {
    next_id: i32,
    projects: HashMap<i32, ProjectView>,
}

struct AppContext {
    store: RwLock<Store>,
}

impl AppContext {
    fn create(&self, name: &str) -> Result<ProjectView, DomainError> {
        let name = name.trim();
        if name.is_empty() {
            return Err(DomainError::EmptyName);
        }
        let mut store = self.store.write().map_err(|_| DomainError::Poisoned)?;
        store.next_id += 1;
        let project = ProjectView { id: store.next_id, name: name.to_owned(), version: 1 };
        store.projects.insert(project.id, project.clone());
        Ok(project)
    }

    fn get(&self, id: i32) -> Result<ProjectView, DomainError> {
        let store = self.store.read().map_err(|_| DomainError::Poisoned)?;
        store.projects.get(&id).cloned().ok_or(DomainError::NotFound(id))
    }

    fn rename(&self, id: i32, name: &str, expected: i32) -> Result<ProjectView, DomainError> {
        let name = name.trim();
        if name.is_empty() {
            return Err(DomainError::EmptyName);
        }
        let mut store = self.store.write().map_err(|_| DomainError::Poisoned)?;
        let project = store.projects.get_mut(&id).ok_or(DomainError::NotFound(id))?;
        if project.version != expected {
            return Err(DomainError::VersionMismatch { expected, actual: project.version });
        }
        project.name = name.to_owned();
        project.version += 1;
        Ok(project.clone())
    }
}

// ================= endpoints =================

/// Create a project.
#[endpoint { method = POST, path = "/projects", tags = ["projects"] }]
async fn create_project(
    rqctx: RequestContext<Arc<AppContext>>,
    body: TypedBody<CreateProject>,
) -> Result<HttpResponseCreated<ProjectView>, HttpError> {
    let project = rqctx.context().create(&body.into_inner().name)?;
    slog::info!(rqctx.log, "project created"; "project_id" => project.id);
    Ok(HttpResponseCreated(project))
}

/// Fetch one project by ID.
#[endpoint { method = GET, path = "/projects/{project_id}", tags = ["projects"] }]
async fn get_project(
    rqctx: RequestContext<Arc<AppContext>>,
    path: Path<ProjectPath>,
) -> Result<HttpResponseOk<ProjectView>, HttpError> {
    Ok(HttpResponseOk(rqctx.context().get(path.into_inner().project_id)?))
}

/// Rename a project. The request must carry the version it was based on.
#[endpoint { method = PUT, path = "/projects/{project_id}", tags = ["projects"] }]
async fn rename_project(
    rqctx: RequestContext<Arc<AppContext>>,
    path: Path<ProjectPath>,
    body: TypedBody<RenameProject>,
) -> Result<HttpResponseOk<ProjectView>, HttpError> {
    let id = path.into_inner().project_id;
    let input = body.into_inner();
    Ok(HttpResponseOk(rqctx.context().rename(id, &input.name, input.version)?))
}

fn api() -> Result<ApiDescription<Arc<AppContext>>, String> {
    let mut api = ApiDescription::new();
    api.register(create_project).map_err(|e| e.to_string())?;
    api.register(get_project).map_err(|e| e.to_string())?;
    api.register(rename_project).map_err(|e| e.to_string())?;
    Ok(api)
}

// ================= main =================

#[tokio::main]
async fn main() -> Result<(), String> {
    let api = api()?;

    if std::env::args().any(|a| a == "--openapi") {
        return api
            .openapi("Projects API", semver::Version::new(1, 0, 0))
            .write(&mut std::io::stdout())
            .map_err(|e| e.to_string());
    }

    let log = ConfigLogging::StderrTerminal { level: ConfigLoggingLevel::Info }
        .to_logger("projects-api")
        .map_err(|e| e.to_string())?;

    let config = ConfigDropshot {
        bind_address: "127.0.0.1:8080".parse().map_err(|e: std::net::AddrParseError| e.to_string())?,
        default_request_body_max_bytes: 64 * 1024,
        ..Default::default()
    };

    let context = Arc::new(AppContext { store: RwLock::new(Store::default()) });
    let server = ServerBuilder::new(api, context, log)
        .config(config)
        .start()
        .map_err(|e| e.to_string())?;

    server.await
}
```

Try it:

```bash
cargo run -- --openapi > openapi.json     # the generated API document
cargo run                                 # start the server

curl -s -X POST localhost:8080/projects -H 'content-type: application/json' -d '{"name":"Alpha"}'
# 201 {"id":1,"name":"Alpha","version":1}

curl -s localhost:8080/projects/9
# 404 {"request_id":"...","error_code":"not_found","message":"project 9 not found"}

curl -s -X PUT localhost:8080/projects/1 -H 'content-type: application/json' -d '{"name":"Beta","version":1}'
# 200 {"id":1,"name":"Beta","version":2}

curl -s -X PUT localhost:8080/projects/1 -H 'content-type: application/json' -d '{"name":"Gamma","version":1}'
# 409 {"request_id":"...","error_code":"conflict","message":"version conflict: expected 1, found 2"}

curl -s localhost:8080/projects/abc
# 400 {"request_id":"...","message":"bad parameter in URL path: unable to parse 'abc' as i32"}
```

What to notice compared with the Stage 4 Axum version:

| Thing | Axum (Stage 4) | Dropshot |
|---|---|---|
| Error body | Hand-written `ErrorBody` | Built in, with `request_id` included |
| Path `abc` | Needed `AppPath` to get JSON | JSON 400 out of the box |
| Request logging | Needed middleware (Stage 7) | Built in, with request IDs |
| API document | Not available | `--openapi` prints it |
| Domain code | — | **Unchanged**: same rules, same `DomainError`, only the `From` impl differs |

The last row is the main lesson: well-separated domain code doesn't care which HTTP framework calls it.

---

## 14. Choosing between them

| Prefer Axum when | Prefer Dropshot when |
|---|---|
| You need WebSockets, SSE, or unusual protocols with good ecosystem support | The API is consumed by generated clients |
| You want Tower middleware (CORS, compression, rate limiting) in-process | The OpenAPI document should be the reviewed contract |
| You serve a browser frontend directly | It's an internal or control-plane API behind a proxy |
| The team already knows Tower and the Tokio ecosystem | You want fewer moving parts and explicit per-handler behaviour |

Both use Tokio, serde, and the same async Rust. Moving between them mostly means changing the HTTP layer, as long as the `domain` and `db` layers stay framework-free (README architecture).

---

## 15. Exercises

- [ ] **1. First endpoint:** build `GET /health` and `GET /version` (returning `{"version":"0.1.0"}`) in Dropshot. Generate the OpenAPI document and find both operations in it.
- [ ] **2. Path structs:** add `GET /projects/{project_id}/tasks/{task_id}` with a two-field path struct. Rename one field so it no longer matches the path and read the error.
- [ ] **3. Query:** add `GET /projects?page=&per_page=` with a `Query<ListParams>` and defaults, and a check that `per_page` ≤ 100 returning 400.
- [ ] **4. Created + Location:** return `201` with a `Location` header from `create_project` using `HttpResponseHeaders`.
- [ ] **5. Auth function:** add the fake `user-<id>-tenant-<id>` bearer token (Stage 3) with an `authenticate(&rqctx)` function, make projects tenant-scoped, and prove another tenant gets 404.
- [ ] **6. OpenAPI check:** write a test that generates the OpenAPI document and compares it with a committed `openapi/projects.json`. Change an endpoint and see the test fail.
- [ ] **7. Same domain, two frameworks:** move `AppContext`'s logic into a framework-free module and call it from both the Stage 4 Axum handlers and these Dropshot endpoints. Only the error conversions should differ.
- [ ] **8. Explain it:** in three or four sentences, explain why Dropshot has no middleware, what you lose and gain from that, and why the OpenAPI document is worth committing.

---

## 16. Further reading

- [Dropshot repository and README](https://github.com/oxidecomputer/dropshot): design goals and examples
- [`dropshot` docs](https://docs.rs/dropshot): `endpoint`, extractors, `HttpError`, `ServerBuilder`, API traits
- [Dropshot examples](https://github.com/oxidecomputer/dropshot/tree/main/dropshot/examples): basic, pagination, WebSocket channels, API traits
- [Progenitor](https://github.com/oxidecomputer/progenitor): generating Rust clients from the OpenAPI document
- [`schemars` docs](https://docs.rs/schemars): `JsonSchema` derive and attributes
- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html): what the generated document describes
