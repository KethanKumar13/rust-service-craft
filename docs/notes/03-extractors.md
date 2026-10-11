# Stage 3: Extractors

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> If you come from JavaScript or TypeScript, the comparisons below map Rust concepts to familiar JS ideas (mostly Express).

## Contents

1. [What an extractor is](#1-what-an-extractor-is)
2. [`Path`](#2-path)
3. [`Query`](#3-query)
4. [`Json`](#4-json)
5. [Headers, method, URI, and raw bodies](#5-headers-method-uri-and-raw-bodies)
6. [Argument order: the body goes last](#6-argument-order-the-body-goes-last)
7. [Optional extractors and rejections](#7-optional-extractors-and-rejections)
8. [Custom extractors with `FromRequestParts`](#8-custom-extractors-with-fromrequestparts)
9. [Validation](#9-validation)
10. [How this looks in a production service](#10-how-this-looks-in-a-production-service)
11. [Putting it all together](#11-putting-it-all-together)
12. [Exercises](#12-exercises)
13. [Further reading](#13-further-reading)

---

## 1. What an extractor is

An **extractor** is a handler argument whose type says *what to take from the request*. Axum runs every extractor **before** your handler. If one fails, the handler never runs and the client gets an error response (a **rejection**).

```rust
async fn update_project(
    Path(id): Path<u32>,                 // from the URL path
    Query(opts): Query<UpdateOptions>,   // from ?key=value
    Json(body): Json<UpdateProject>,     // from the JSON body
) -> String {
    format!("update {id} with {body:?}, opts {opts:?}")
}
```

> **JS comparison:** in Express you read `req.params.id`, `req.query.dryRun`, and `req.body` inside the handler and validate them yourself. In Axum the **types** describe the input: by the time your code runs, `id` is already a `u32` and `body` already has the right shape.

There are two kinds of extractor, and the difference matters:

| Trait | Reads | Examples | How many per handler |
|---|---|---|---|
| `FromRequestParts` | Only the **head** (method, URI, headers, extensions) | `Path`, `Query`, `HeaderMap`, `Method`, `Uri`, `State` | Any number |
| `FromRequest` | The **body** (consumes it) | `Json`, `String`, `Bytes`, `Form`, `Request` | **At most one**, and it must be **last** |

The body is a stream that can only be read once, which is why only one extractor may consume it.

---

## 2. `Path`

`Path<T>` parses the `{captures}` from the route (Stage 2).

```rust
use axum::extract::Path;
use serde::Deserialize;

// One capture
async fn get_project(Path(id): Path<u32>) -> String {
    format!("project {id}")
}

// Several captures as a tuple: in route order
async fn get_task(Path((project_id, task_id)): Path<(u32, u32)>) -> String {
    format!("task {task_id} in project {project_id}")
}

// Several captures as a struct: matched by NAME
#[derive(Deserialize)]
struct TaskPath {
    project_id: u32,
    task_id: u32,
}

async fn get_task_named(Path(p): Path<TaskPath>) -> String {
    format!("task {} in project {}", p.task_id, p.project_id)
}
```

Route: `.route("/projects/{project_id}/tasks/{task_id}", get(get_task_named))`

- `Path(id)` in the argument list is **destructuring**: `Path<u32>` is a tuple struct, and the pattern unwraps it. It's the same as writing `path: Path<u32>` and using `path.0`.
- A struct is safer than a tuple when there are several captures: names can't be swapped by accident.
- A value that doesn't parse (`/projects/abc` for `u32`) gives **400 Bad Request**.

### 2.1 Newtype IDs in paths

Following the "newtype IDs" convention, deserialize straight into the ID type:

```rust
#[derive(Debug, Clone, Copy, Deserialize)]
#[serde(transparent)]          // (de)serialize exactly like the inner i32
struct ProjectId(i32);

async fn get_project(Path(id): Path<ProjectId>) -> String {
    format!("{id:?}")
}
```

Now a `CustomerId` can never be passed where a `ProjectId` is expected, from the URL all the way down to the database layer.

---

## 3. `Query`

`Query<T>` deserializes the query string (`?page=2&per_page=20`) into a struct.

```rust
use axum::extract::Query;

#[derive(Debug, Deserialize)]
struct ListParams {
    #[serde(default = "default_page")]
    page: u32,
    #[serde(default = "default_per_page")]
    per_page: u32,
    search: Option<String>,          // optional: missing => None
}

fn default_page() -> u32 { 1 }
fn default_per_page() -> u32 { 20 }

async fn list_projects(Query(params): Query<ListParams>) -> String {
    format!("{params:?}")
}
```

| Request | Result |
|---|---|
| `/projects` | `page: 1, per_page: 20, search: None` |
| `/projects?page=3&search=alpha` | `page: 3, per_page: 20, search: Some("alpha")` |
| `/projects?page=abc` | **400 Bad Request** |

Rules of thumb:
- Every field that may be missing is either `Option<T>` or has a `#[serde(default ...)]`. Otherwise a missing key is a 400.
- Unknown keys are ignored by default. Add `#[serde(deny_unknown_fields)]` to reject them.
- Query values are always strings on the wire; serde parses them into `u32`, `bool`, enums, and so on.
- Repeated keys (`?tag=a&tag=b`) into a `Vec<String>` are **not** supported by Axum's `Query`. Use `axum_extra::extract::Query` (backed by `serde_html_form`) for that.

> **JS comparison:** `req.query.page` is a string or `undefined` and you parse it yourself. `Query<ListParams>` is parsed, defaulted, and type-checked in one step.

---

## 4. `Json`

`Json<T>` reads the whole body and deserializes it with serde. `T` must implement `Deserialize`.

```rust
use axum::{http::StatusCode, Json};
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
struct CreateProject {
    name: String,
    description: Option<String>,
    customer_id: i32,          // JSON key: "customerId"
}

#[derive(Serialize)]
#[serde(rename_all = "camelCase")]
struct ProjectDto {
    id: i32,
    name: String,
    customer_id: i32,
}

async fn create_project(Json(body): Json<CreateProject>) -> (StatusCode, Json<ProjectDto>) {
    let dto = ProjectDto { id: 7, name: body.name, customer_id: body.customer_id };
    (StatusCode::CREATED, Json(dto))
}
```

`Json` checks three things, each with its own status code:

| Problem | Example | Status |
|---|---|---|
| Missing or wrong `Content-Type` | no `Content-Type: application/json` header | **415** Unsupported Media Type |
| Not valid JSON | `{"name": ` | **400** Bad Request |
| Valid JSON, wrong shape | `{"name": 5}`, missing `customerId` | **422** Unprocessable Entity |

> **JS comparison:** `express.json()` gives you `req.body` as `any`. `Json<CreateProject>` rejects the request unless the body has exactly the shape you declared.

**Body size limit:** Axum limits bodies to **2 MB by default**. A bigger body is rejected with **413 Payload Too Large**. Change it per router with `DefaultBodyLimit` (covered with middleware in Stage 7). Large uploads should be streamed, not buffered (Stage 10).

---

## 5. Headers, method, URI, and raw bodies

```rust
use axum::{
    body::Bytes,
    http::{header, HeaderMap, Method, Uri},
};

// All headers
async fn show_agent(headers: HeaderMap) -> String {
    headers
        .get(header::USER_AGENT)
        .and_then(|v| v.to_str().ok())       // header values may not be valid UTF-8
        .unwrap_or("unknown")
        .to_string()
}

// Method and URI
async fn echo(method: Method, uri: Uri) -> String {
    format!("{method} {uri}")
}

// Raw body as text or bytes (body extractors: must be last)
async fn raw_text(body: String) -> String { format!("got {} chars", body.len()) }
async fn raw_bytes(body: Bytes) -> String { format!("got {} bytes", body.len()) }
```

Other built-in extractors you'll meet:

| Extractor | Gives you |
|---|---|
| `State<S>` | Shared application state (Stage 5) |
| `Extension<T>` | A value that middleware attached to the request (Stage 7) |
| `Form<T>` | An `application/x-www-form-urlencoded` body |
| `ConnectInfo<SocketAddr>` | The client's address (needs `into_make_service_with_connect_info`) |
| `Request` | The whole request, if you want to do everything yourself |
| `axum_extra::extract::CookieJar` | Cookies (used for the auth cookie in Stage 8) |
| `axum_extra::TypedHeader<T>` | One typed header, such as `TypedHeader<Authorization<Bearer>>` |

---

## 6. Argument order: the body goes last

Extractors run in **argument order**, left to right. Head-only extractors can appear in any order, but the body extractor must be the **last** argument:

```rust
// OK
async fn ok(Path(id): Path<u32>, headers: HeaderMap, Json(body): Json<CreateProject>) {}

// Compile error: Json consumes the body, so nothing may come after it
// async fn bad(Json(body): Json<CreateProject>, Path(id): Path<u32>) {}

// Compile error: two body extractors
// async fn bad2(body: String, Json(b): Json<CreateProject>) {}
```

The error message is the usual long "`Handler<_, _>` is not implemented". Add `#[axum::debug_handler]` and it says what's wrong: *"`Json<_>` consumes the request body and thus must be the last argument to the handler function"*.

A good habit from production code is a **fixed order**:

```text
State  →  auth user  →  Path  →  Query  →  headers  →  body
```

---

## 7. Optional extractors and rejections

### 7.1 `Option<T>`: allow it to be missing

```rust
use axum::extract::Query;

async fn search(q: Option<Query<SearchParams>>) -> String {
    match q {
        Some(Query(p)) => format!("searching for {}", p.term),
        None => "no search term".into(),
    }
}
```

In **Axum 0.8**, `Option<T>` only works for extractors that implement `OptionalFromRequestParts` / `OptionalFromRequest`. The rule is: `None` when the input is **absent**, and still a **rejection** when it is present but **invalid**. That's safer than 0.7, where a broken value silently became `None`.

### 7.2 `Result<T, T::Rejection>`: handle the error yourself

```rust
use axum::extract::rejection::JsonRejection;

async fn create(payload: Result<Json<CreateProject>, JsonRejection>) -> (StatusCode, String) {
    match payload {
        Ok(Json(body)) => (StatusCode::CREATED, format!("created {}", body.name)),
        Err(rejection) => (rejection.status(), format!("invalid body: {}", rejection.body_text())),
    }
}
```

Each extractor has its own rejection type: `PathRejection`, `QueryRejection`, `JsonRejection`, and so on. They all implement `IntoResponse`, and each has `.status()` and `.body_text()`.

### 7.3 Why you'll want consistent errors

By default, rejections return **plain-text** bodies with different wording per extractor. A real API should return the same JSON error shape everywhere. The usual fix is a **wrapper extractor** that converts the rejection into your `AppError`:

```rust
use axum::extract::{FromRequest, rejection::JsonRejection};

#[derive(FromRequest)]                       // needs axum feature "macros"
#[from_request(via(axum::Json), rejection(AppError))]
struct AppJson<T>(T);

impl From<JsonRejection> for AppError {
    fn from(r: JsonRejection) -> Self {
        AppError::BadRequest(r.body_text())
    }
}

async fn create(AppJson(body): AppJson<CreateProject>) -> Result<String, AppError> {
    Ok(format!("created {}", body.name))
}
```

`AppError` itself is built properly in Stage 4. For now, notice that **you choose** what a failed extraction looks like.

---

## 8. Custom extractors with `FromRequestParts`

You can write your own extractor for anything you need in many handlers: the current user, a tenant ID, a request ID, pagination. This is the most important pattern in this module, because **authentication is an extractor** in most Axum services.

### 8.1 A request ID extractor

```rust
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
};

struct RequestId(String);

impl<S> FromRequestParts<S> for RequestId
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        parts
            .headers
            .get("x-request-id")
            .and_then(|v| v.to_str().ok())
            .map(|s| RequestId(s.to_string()))
            .ok_or((StatusCode::BAD_REQUEST, "missing x-request-id header"))
    }
}

async fn handler(RequestId(id): RequestId) -> String {
    format!("request {id}")
}
```

How to read it:
- **`impl<S> FromRequestParts<S>`**: "this works for any state type `S`". Stage 5 shows how to require a specific `AppState` here, for example to reach the database or JWT secret.
- **`type Rejection`**: what's returned when extraction fails. Anything that implements `IntoResponse`.
- **`parts: &mut Parts`**: the request head: `parts.method`, `parts.uri`, `parts.headers`, `parts.extensions`.
- **`async fn` in a trait**: since Axum 0.8 this is plain Rust. Older code (and older tutorials) used `#[async_trait]` for this; you don't need it anymore.

### 8.2 A preview of the auth extractor

This is the shape of the `AuthUser` extractor you'll build in Stage 8. Real verification (JWT signature, expiry, blocklist) comes later:

```rust
#[derive(Debug, Clone)]
struct AuthUser {
    user_id: i32,
    tenant_id: i32,
    role: String,
}

impl<S> FromRequestParts<S> for AuthUser
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let token = parts
            .headers
            .get(header::AUTHORIZATION)
            .and_then(|v| v.to_str().ok())
            .and_then(|v| v.strip_prefix("Bearer "))
            .ok_or((StatusCode::UNAUTHORIZED, "missing bearer token"))?;

        // Stage 8: decode and verify a JWT here. For now a fake token format:
        // "user-<id>-tenant-<id>"
        decode_fake_token(token).ok_or((StatusCode::UNAUTHORIZED, "invalid token"))
    }
}
```

Every handler that takes an `AuthUser` argument is now **protected automatically**: without a valid token the handler never runs. Forgetting the argument is the only way to make a route public, which makes public routes easy to spot in review.

> **JS comparison:** in Express you'd write an `authMiddleware` that sets `req.user`, then hope every route remembered to use it and every handler handles `req.user` being `undefined`. In Axum, a handler that declares `user: AuthUser` **cannot** run without one.

### 8.3 `FromRequest` for body-based extractors

Use `FromRequest` (not `FromRequestParts`) only when your extractor needs the body. The common case is wrapping `Json` to add validation, shown next.

---

## 9. Validation

Extraction proves the input has the right **shape**. Validation proves it makes **sense**: the name isn't empty, the page size isn't 10,000.

### 9.1 A small `Validate` trait (no extra crates)

```rust
trait Validate {
    fn validate(&self) -> Result<(), String>;
}

impl Validate for CreateProject {
    fn validate(&self) -> Result<(), String> {
        let name = self.name.trim();
        if name.is_empty() {
            return Err("name must not be empty".into());
        }
        if name.len() > 100 {
            return Err("name must be at most 100 characters".into());
        }
        Ok(())
    }
}
```

### 9.2 A `ValidJson<T>` extractor: parse **and** validate

```rust
use axum::extract::{FromRequest, Request};
use serde::de::DeserializeOwned;

struct ValidJson<T>(T);

impl<S, T> FromRequest<S> for ValidJson<T>
where
    S: Send + Sync,
    T: DeserializeOwned + Validate,
{
    type Rejection = (StatusCode, String);

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|r| (r.status(), r.body_text()))?;

        value
            .validate()
            .map_err(|msg| (StatusCode::UNPROCESSABLE_ENTITY, msg))?;

        Ok(ValidJson(value))
    }
}

async fn create(ValidJson(body): ValidJson<CreateProject>) -> String {
    format!("created {}", body.name)  // body is guaranteed valid here
}
```

`DeserializeOwned` means "can be deserialized without borrowing from the input". Use it whenever the input buffer is temporary, as it is here.

### 9.3 The `validator` crate

For larger APIs, the [`validator`](https://docs.rs/validator) crate gives you derive-based rules:

```rust
use validator::Validate;

#[derive(Deserialize, Validate)]
struct CreateUser {
    #[validate(email)]
    email: String,
    #[validate(length(min = 1, max = 100))]
    name: String,
    #[validate(range(min = 1, max = 100))]
    per_page: u32,
}
```

The same `ValidJson<T>` pattern works with it: call `value.validate()` and map the errors.

### 9.4 Where validation belongs

| Check | Layer |
|---|---|
| Shape and types (`name` is a string) | Extractor (serde) |
| Simple input rules (not empty, max length, ranges) | Extractor / DTO validation in `api` |
| Business rules (name unique per tenant, project not `Live`) | `domain` |
| Constraints the database must guarantee (unique index, foreign keys) | `db` / migrations |

Validate early, but don't put business rules in extractors: they need the database and belong in the domain layer.

---

## 10. How this looks in a production service

A typical update handler in the `api` crate:

```rust
async fn update_project(
    State(state): State<AppState>,          // Stage 5
    user: AuthUser,                          // custom extractor, Stage 8
    Path(id): Path<ProjectId>,               // newtype ID
    ValidJson(body): ValidJson<UpdateProject>, // parse + validate, body last
) -> Result<Json<ProjectDto>, AppError> {    // Stage 4
    let project = domain::projects::update(&state.pool, user.tenant_id, id, body).await?;
    Ok(Json(project))
}
```

Read the signature and you know the endpoint's whole contract:

- it needs app state,
- it needs an **authenticated** user (and gets their `tenant_id` for scoping),
- it takes a typed project ID from the path,
- it takes a validated JSON body,
- it returns a project DTO or a typed error.

The body of the handler is one line: call the domain layer. All the input handling happened in the extractors.

Conventions:
- **DTOs** (`UpdateProject`, `ProjectDto`) live in the `shared` crate, under `types/`.
- **Custom extractors** (`AuthUser`, `ValidJson`, `AppJson`) live in `api/src/extractors/`.
- **Every rejection** is converted to `AppError`, so clients always get the same JSON error shape.

---

## 11. Putting it all together

A projects API using `Path` with a newtype ID, `Query` with defaults, a custom `RequestId` extractor, a fake `AuthUser`, and a `ValidJson` extractor. It calls itself with `oneshot` (Stage 2, section 8), so it runs **in the Playground**.

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
tower = { version = "0.5", features = ["util"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
use axum::{
    body::{to_bytes, Body},
    extract::{FromRequest, FromRequestParts, Path, Query, Request},
    http::{header, request::Parts, StatusCode},
    routing::{get, post},
    Json, Router,
};
use serde::{de::DeserializeOwned, Deserialize, Serialize};
use tower::ServiceExt;

// ---------- types ----------

#[derive(Debug, Clone, Copy, Deserialize, Serialize)]
#[serde(transparent)]
struct ProjectId(i32);

#[derive(Debug, Deserialize)]
struct ListParams {
    #[serde(default = "default_page")]
    page: u32,
    #[serde(default = "default_per_page")]
    per_page: u32,
}
fn default_page() -> u32 { 1 }
fn default_per_page() -> u32 { 20 }

#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase", deny_unknown_fields)]
struct CreateProject {
    name: String,
}

#[derive(Serialize)]
struct ProjectDto {
    id: ProjectId,
    name: String,
    tenant_id: i32,
}

// ---------- validation ----------

trait Validate {
    fn validate(&self) -> Result<(), String>;
}

impl Validate for CreateProject {
    fn validate(&self) -> Result<(), String> {
        if self.name.trim().is_empty() {
            Err("name must not be empty".into())
        } else {
            Ok(())
        }
    }
}

struct ValidJson<T>(T);

impl<S, T> FromRequest<S> for ValidJson<T>
where
    S: Send + Sync,
    T: DeserializeOwned + Validate,
{
    type Rejection = (StatusCode, String);

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|r| (r.status(), r.body_text()))?;
        value
            .validate()
            .map_err(|m| (StatusCode::UNPROCESSABLE_ENTITY, m))?;
        Ok(ValidJson(value))
    }
}

// ---------- custom extractors ----------

#[derive(Debug)]
struct AuthUser {
    user_id: i32,
    tenant_id: i32,
}

impl<S> FromRequestParts<S> for AuthUser
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, String);

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let unauthorized = || (StatusCode::UNAUTHORIZED, "unauthorized".to_string());

        // Fake token format for this stage: "Bearer user-<id>-tenant-<id>"
        let token = parts
            .headers
            .get(header::AUTHORIZATION)
            .and_then(|v| v.to_str().ok())
            .and_then(|v| v.strip_prefix("Bearer "))
            .ok_or_else(unauthorized)?;

        let rest = token.strip_prefix("user-").ok_or_else(unauthorized)?;
        let (user, tenant) = rest.split_once("-tenant-").ok_or_else(unauthorized)?;
        let user_id = user.parse().map_err(|_| unauthorized())?;
        let tenant_id = tenant.parse().map_err(|_| unauthorized())?;

        Ok(AuthUser { user_id, tenant_id })
    }
}

// ---------- handlers ----------

async fn list_projects(user: AuthUser, Query(p): Query<ListParams>) -> String {
    format!(
        "user {} lists tenant {} projects, page {} ({} per page)",
        user.user_id, user.tenant_id, p.page, p.per_page
    )
}

async fn get_project(user: AuthUser, Path(id): Path<ProjectId>) -> Json<ProjectDto> {
    Json(ProjectDto { id, name: "Alpha".into(), tenant_id: user.tenant_id })
}

async fn create_project(
    user: AuthUser,
    ValidJson(body): ValidJson<CreateProject>,   // body extractor last
) -> (StatusCode, Json<ProjectDto>) {
    let dto = ProjectDto {
        id: ProjectId(7),
        name: body.name.trim().to_string(),
        tenant_id: user.tenant_id,
    };
    (StatusCode::CREATED, Json(dto))
}

fn app() -> Router {
    Router::new()
        .route("/projects", get(list_projects).post(create_project))
        .route("/projects/{id}", get(get_project))
}

// ---------- tiny client ----------

const TOKEN: &str = "Bearer user-1-tenant-42";

async fn call(
    app: &Router,
    method: &str,
    uri: &str,
    auth: Option<&str>,
    json: Option<&str>,
) -> (StatusCode, String) {
    let mut builder = Request::builder().method(method).uri(uri);
    if let Some(a) = auth {
        builder = builder.header(header::AUTHORIZATION, a);
    }
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

    let cases: [(&str, &str, Option<&str>, Option<&str>); 8] = [
        ("GET", "/projects", Some(TOKEN), None),
        ("GET", "/projects?page=3", Some(TOKEN), None),
        ("GET", "/projects?page=abc", Some(TOKEN), None),
        ("GET", "/projects/5", Some(TOKEN), None),
        ("GET", "/projects/5", None, None),
        ("POST", "/projects", Some(TOKEN), Some(r#"{"name":"  Alpha  "}"#)),
        ("POST", "/projects", Some(TOKEN), Some(r#"{"name":"   "}"#)),
        ("POST", "/projects", Some(TOKEN), Some(r#"{"name":5}"#)),
    ];

    for (method, uri, auth, json) in cases {
        let (status, body) = call(&app, method, uri, auth, json).await;
        let who = if auth.is_some() { "auth" } else { "anon" };
        println!("{method:<4} {uri:<20} {who} -> {} {body}", status.as_u16());
    }
}
```

Expected output (the exact text of serde and Axum messages can differ between versions):

```text
GET  /projects            auth -> 200 user 1 lists tenant 42 projects, page 1 (20 per page)
GET  /projects?page=3     auth -> 200 user 1 lists tenant 42 projects, page 3 (20 per page)
GET  /projects?page=abc   auth -> 400 Failed to deserialize query string: page: invalid digit found in string
GET  /projects/5          auth -> 200 {"id":5,"name":"Alpha","tenant_id":42}
GET  /projects/5          anon -> 401 unauthorized
POST /projects            auth -> 201 {"id":7,"name":"Alpha","tenant_id":42}
POST /projects            auth -> 422 name must not be empty
POST /projects            auth -> 422 Failed to deserialize the JSON body into the target type: name: invalid type: integer `5`, expected a string at line 1 column 9
```

What each line shows:

| Case | Concept |
|---|---|
| `/projects` vs `?page=3` | `Query` with serde defaults |
| `?page=abc` | `Query` rejection → 400 |
| `/projects/5` with token | `Path<ProjectId>` newtype + `AuthUser` supplies `tenant_id` |
| `/projects/5` without token | Custom extractor rejects → handler never runs → 401 |
| `"  Alpha  "` | `ValidJson` passes, the handler trims |
| `"   "` | Shape OK, **validation** fails → 422 |
| `"name": 5` | **Shape** wrong → `Json` rejection → 422 |

Notice the error bodies are still plain text with different formats. Stage 4 fixes that with one `AppError` type.

---

## 12. Exercises

- [ ] **1. `Path` struct:** route `/customers/{customer_id}/projects/{project_id}` and extract it into a `#[derive(Deserialize)]` struct. Then swap the order of the struct fields and confirm it still works (matching is by name).
- [ ] **2. Newtype ID:** create `CustomerId(i32)` with `#[serde(transparent)]` and use it in `Path`. Call the route with `/customers/abc` and note the status code.
- [ ] **3. `Query` with defaults:** build `GET /search` with `q: String` (required), `page: u32` (default 1), and `sort: Option<String>`. Test with no parameters, with only `q`, and with `page=-1`. Write down each status.
- [ ] **4. `Json` status codes:** `POST` a body to a `Json<CreateProject>` handler three times to trigger **415**, **400**, and **422**. Write one line each on what caused it.
- [ ] **5. Argument order:** put `Json` before `Path` in a handler and read the compiler error. Then add `#[axum::debug_handler]` (feature `macros`) and compare the messages.
- [ ] **6. `Option` and `Result`:** write a handler that takes `Option<Query<Params>>`, and another that takes `Result<Json<CreateProject>, JsonRejection>` and returns `{"error": "..."}` with the rejection's status.
- [ ] **7. Custom extractor:** write `TenantId(i32)` that reads an `x-tenant-id` header, rejects a missing header with 400 and a non-numeric one with 400 and a different message. Use it in two handlers.
- [ ] **8. `ValidJson`:** add a `Validate` impl for `CreateCustomer { name: String, email: String }` (name 1–100 chars after trimming, email must contain `@`). Test one valid and two invalid bodies through `ValidJson`.

> When you can solve exercises 1–8 without looking anything up, you're ready for **Stage 4: Errors**.

---

## 13. Further reading

- [`axum::extract` docs](https://docs.rs/axum/latest/axum/extract/index.html): every built-in extractor, the ordering rules, and how rejections work
- [Axum example: `customize-extractor-error`](https://github.com/tokio-rs/axum/tree/main/examples/customize-extractor-error): three ways to control rejection responses
- [Axum example: `validator`](https://github.com/tokio-rs/axum/tree/main/examples/validator): a `ValidatedForm` extractor using the `validator` crate
- [`axum-extra` docs](https://docs.rs/axum-extra): `CookieJar`, `TypedHeader`, `WithRejection`, and the multi-value `Query`
- [serde attributes](https://serde.rs/attributes.html): `default`, `rename_all`, `deny_unknown_fields`, `transparent`
- [Announcing Axum 0.8](https://tokio.rs/blog/2025-01-01-announcing-axum-0-8-0): `async fn` in extractor traits and the new `Option<T>` rules
