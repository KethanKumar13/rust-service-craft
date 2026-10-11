# Stage 2: Routing with Axum

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> If you come from JavaScript or TypeScript, the comparisons below map Rust concepts to familiar JS ideas (mostly Express).

## Contents

1. [HTTP in five minutes](#1-http-in-five-minutes)
2. [A first server](#2-a-first-server)
3. [Handlers](#3-handlers)
4. [Routes and methods](#4-routes-and-methods)
5. [Path syntax in Axum 0.8](#5-path-syntax-in-axum-08)
6. [Organizing routes: `nest` and `merge`](#6-organizing-routes-nest-and-merge)
7. [Fallbacks and 404s](#7-fallbacks-and-404s)
8. [Calling a router without a network](#8-calling-a-router-without-a-network)
9. [How this looks in a production service](#9-how-this-looks-in-a-production-service)
10. [Putting it all together](#10-putting-it-all-together)
11. [Exercises](#11-exercises)
12. [Further reading](#12-further-reading)

---

## 1. HTTP in five minutes

Every HTTP request has the same parts:

```text
POST /api/v1/projects?draft=true HTTP/1.1      <- method, path, query string
Host: localhost:3000                           <- headers
Content-Type: application/json
Cookie: access_token_cookie=eyJ...

{"name":"Alpha"}                               <- body (optional)
```

And every response:

```text
HTTP/1.1 201 Created                           <- status code
Content-Type: application/json                 <- headers
Location: /api/v1/projects/7

{"id":7,"name":"Alpha"}                        <- body (optional)
```

### 1.1 Methods

| Method | Meaning | Has a body? | Safe to repeat? |
|---|---|---|---|
| `GET` | Read | No | Yes |
| `POST` | Create, or trigger an action | Yes | No |
| `PUT` | Replace a whole resource | Yes | Yes |
| `PATCH` | Change part of a resource | Yes | Not necessarily |
| `DELETE` | Remove | Usually not | Yes |

### 1.2 Status codes you'll use constantly

| Code | Name | Use |
|---|---|---|
| 200 | OK | Successful read or update |
| 201 | Created | `POST` created something (often with a `Location` header) |
| 204 | No Content | Success with no body (often `DELETE`) |
| 400 | Bad Request | The input is malformed or invalid |
| 401 | Unauthorized | Not logged in, or the token is invalid |
| 403 | Forbidden | Logged in, but not allowed |
| 404 | Not Found | No such resource or route |
| 405 | Method Not Allowed | The path exists, but not for this method |
| 409 | Conflict | Version mismatch (optimistic locking), duplicate |
| 422 | Unprocessable Entity | Valid JSON, wrong shape (Axum's default for `Json` errors) |
| 500 | Internal Server Error | A bug or an unexpected failure |

**Routing** is the part of a server that looks at the **method + path** and decides which function handles the request.

---

## 2. A first server

`Cargo.toml`:

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
```

```rust
use axum::{routing::get, Router};
use tokio::net::TcpListener;

async fn hello() -> &'static str {
    "Hello, world!"
}

#[tokio::main]
async fn main() -> Result<(), std::io::Error> {
    let app = Router::new().route("/", get(hello));

    let listener = TcpListener::bind("127.0.0.1:3000").await?;
    println!("listening on http://{}", listener.local_addr()?);
    axum::serve(listener, app).await
}
```

```bash
curl http://127.0.0.1:3000/        # Hello, world!
```

Three pieces:

1. **A handler** (`hello`): an `async fn` that returns something that can become a response.
2. **A `Router`**: maps paths and methods to handlers.
3. **`axum::serve`**: accepts TCP connections and runs the router for each request, each one as its own Tokio task (Stage 1).

> **JS comparison:**
> ```js
> const app = express();
> app.get("/", (req, res) => res.send("Hello, world!"));
> app.listen(3000);
> ```
> The shape is the same. The difference: in Axum the handler **returns** the response instead of calling `res.send(...)`, and the compiler checks that what you return can be turned into a response.

> **Playground note:** the Playground includes `axum` and `tokio`, but it can't open a network port. Section 8 shows how to call a router directly, without a network, which works in the Playground and in tests. The Playground's Axum version may lag behind; if `{id}` paths fail there, check its crate version.

---

## 3. Handlers

### 3.1 What makes a function a handler

A function can be used as a handler if:

- it is an `async fn` (or a closure returning a future)
- its arguments are **extractors** (0 to 16 of them, covered in Stage 3)
- its return type implements **`IntoResponse`**
- its future is **`Send`** (the Stage 1 rule: no `std::sync` lock guard held across `.await`)

Axum checks all of this through the `Handler` trait. If one rule is broken, you get a long error saying something like "the trait `Handler<_, _>` is not implemented for fn item...".

**Tip:** add `#[axum::debug_handler]` to the handler (requires `axum = { version = "0.8", features = ["macros"] }`) to get a much clearer error that points at the actual problem.

### 3.2 Return types

Many types already implement `IntoResponse`:

```rust
use axum::{
    http::{header, StatusCode},
    response::{Html, IntoResponse},
    Json,
};
use serde::Serialize;

#[derive(Serialize)]
struct Health { status: &'static str }

async fn text() -> &'static str { "plain text" }                 // 200, text/plain
async fn owned() -> String { format!("{} + {}", 1, 2) }          // 200, text/plain
async fn html() -> Html<&'static str> { Html("<h1>Hi</h1>") }    // 200, text/html
async fn json() -> Json<Health> { Json(Health { status: "ok" }) } // 200, application/json
async fn empty() -> StatusCode { StatusCode::NO_CONTENT }         // 204, no body

// A tuple: status + body
async fn created() -> (StatusCode, &'static str) {
    (StatusCode::CREATED, "created")
}

// A tuple: status + headers + body
async fn created_with_location() -> impl IntoResponse {
    (
        StatusCode::CREATED,
        [(header::LOCATION, "/api/v1/projects/7")],
        "created",
    )
}
```

### 3.3 `Result` as a return type

If both `T` and `E` implement `IntoResponse`, so does `Result<T, E>`:

```rust
async fn maybe(flag: bool) -> Result<&'static str, StatusCode> {
    if flag { Ok("yes") } else { Err(StatusCode::NOT_FOUND) }
}
```

This is how almost every real handler looks. In Stage 4, `StatusCode` gets replaced by a proper `AppError` enum.

### 3.4 `impl IntoResponse`

Use `impl IntoResponse` when the concrete type is long or doesn't matter to callers. Every return path must still produce the **same** type, so if branches return different types, call `.into_response()` on each:

```rust
use axum::response::{IntoResponse, Response};

async fn mixed(ok: bool) -> Response {
    if ok {
        Json(Health { status: "ok" }).into_response()
    } else {
        (StatusCode::SERVICE_UNAVAILABLE, "down").into_response()
    }
}
```

---

## 4. Routes and methods

### 4.1 `route(path, method_router)`

```rust
use axum::routing::{delete, get, patch, post, put};

let app = Router::new()
    .route("/projects", get(list_projects))
    .route("/projects", post(create_project));   // ERROR at startup: overlapping route
```

The **same path may only be registered once**. Attach several methods to it by **chaining**:

```rust
let app = Router::new()
    .route("/projects", get(list_projects).post(create_project))
    .route(
        "/projects/{id}",
        get(get_project).put(replace_project).patch(update_project).delete(delete_project),
    );
```

`get(...)`, `post(...)` and the others return a **`MethodRouter`**, which is a small router for one path keyed by method.

### 4.2 405 Method Not Allowed comes for free

If `/projects` has `GET` and `POST` and a client sends `DELETE /projects`, Axum answers **405** and sets an `Allow: GET,HEAD,POST` header. `HEAD` is handled automatically for `GET` routes.

> **JS comparison:** Express returns 404 in this case unless you add your own handling. Axum distinguishes "no such path" (404) from "no such method on this path" (405).

### 4.3 Route conflicts are startup panics

Registering an overlapping route **panics when the router is built**, not when a request arrives:

```rust
Router::new()
    .route("/projects/{id}", get(get_project))
    .route("/projects/{project_id}", get(other));  // panic: conflicting routes
```

This is good: a misconfigured router fails immediately at startup (and in every test that builds it), not in production at 3 a.m.

---

## 5. Path syntax in Axum 0.8

| Pattern | Matches | Doesn't match |
|---|---|---|
| `/projects` | `/projects` | `/projects/`, `/projects/1` |
| `/projects/{id}` | `/projects/1`, `/projects/abc` | `/projects`, `/projects/1/tasks` |
| `/projects/{id}/tasks/{task_id}` | `/projects/1/tasks/9` | `/projects/1/tasks` |
| `/files/{*path}` | `/files/a`, `/files/a/b/c.txt` | `/files` |

- **`{name}`** captures **one** segment.
- **`{*name}`** (wildcard) captures **the rest** of the path, slashes included. It must be the last segment.
- Static segments win over captures: `/projects/new` and `/projects/{id}` can coexist, and `/projects/new` goes to the static route.
- **Trailing slashes matter.** `/projects` and `/projects/` are different routes. Pick one style (no trailing slash is the common choice) and stick to it.

> **Version warning:** Axum **0.7 and earlier** used `/:id` and `/*path`. Axum **0.8** uses `/{id}` and `/{*path}`, and the old syntax **panics at startup** with a message telling you to migrate. Many blog posts and answers online still show the old syntax.

### 5.1 Reading path values (preview)

Captured values are read with the `Path` extractor. Stage 3 covers it properly; for now:

```rust
use axum::extract::Path;

async fn get_project(Path(id): Path<u32>) -> String {
    format!("project {id}")
}

async fn get_task(Path((project_id, task_id)): Path<(u32, u32)>) -> String {
    format!("task {task_id} in project {project_id}")
}
```

If the segment can't be parsed (`/projects/abc` for a `u32`), Axum responds with **400 Bad Request** before your handler runs.

> **JS comparison:** Express's `/projects/:id` becomes `/projects/{id}`, and `req.params.id` (always a string) becomes `Path(id): Path<u32>`, already parsed and type-checked.

---

## 6. Organizing routes: `nest` and `merge`

Real services have dozens of routes. Keep them in **one function per area** that returns a `Router`, then combine them.

### 6.1 `merge`: combine routers side by side

```rust
fn project_routes() -> Router {
    Router::new()
        .route("/projects", get(list_projects).post(create_project))
        .route("/projects/{id}", get(get_project).delete(delete_project))
}

fn user_routes() -> Router {
    Router::new()
        .route("/users", get(list_users))
        .route("/users/{id}", get(get_user))
}

let api = Router::new()
    .merge(project_routes())
    .merge(user_routes());
```

`merge` keeps paths as they are. Overlapping routes between the two routers panic, as in section 4.3.

### 6.2 `nest`: mount a router under a prefix

```rust
let app = Router::new()
    .route("/health", get(health))
    .nest("/api/v1", api);

// /api/v1/projects       -> list_projects
// /api/v1/projects/7     -> get_project
// /api/v1/users          -> list_users
// /health                -> health
```

Rules:
- The prefix must start with `/` and must not be `/` itself (that panics; use `merge` instead).
- Captures in the prefix work: `.nest("/tenants/{tenant_id}", tenant_routes)`, and handlers can extract `tenant_id`.
- A wildcard is not allowed in the prefix.

> **JS comparison:** `nest` is `app.use("/api/v1", apiRouter)`, and `merge` is like calling `app.use(router)` for several routers with no prefix.

### 6.3 Static files with `nest_service`

Routers can also contain any Tower **service**, for example a static file server from `tower-http`:

```rust
use tower_http::services::ServeDir;

let app = Router::new()
    .nest("/api/v1", api)
    .nest_service("/assets", ServeDir::new("dist/assets"));
```

Tower services and layers are covered in the middleware stage.

---

## 7. Fallbacks and 404s

With no fallback, unmatched requests get an **empty 404**. A custom fallback gives clients a consistent error body:

```rust
use axum::http::{StatusCode, Uri};
use axum::Json;
use serde_json::{json, Value};

async fn not_found(uri: Uri) -> (StatusCode, Json<Value>) {
    (
        StatusCode::NOT_FOUND,
        Json(json!({ "error": "not_found", "path": uri.path() })),
    )
}

let app = Router::new()
    .nest("/api/v1", api)
    .fallback(not_found);
```

The fallback receives any request that matched **no path**. A path that matched but used the wrong method still gets **405**, not the fallback. Use `.method_not_allowed_fallback(handler)` if you want to customize the 405 response too.

---

## 8. Calling a router without a network

A `Router` is a **Tower `Service`**: something that takes a `Request` and asynchronously returns a `Response`. That means you can call it directly, without binding a port. This is how Axum handlers are unit-tested (Stage 11), and it lets you try routing in the Playground.

`Cargo.toml` additions:

```toml
tower = { version = "0.5", features = ["util"] }   # for ServiceExt::oneshot
```

```rust
use axum::{
    body::{to_bytes, Body},
    http::{Request, StatusCode},
    routing::get,
    Router,
};
use tower::ServiceExt; // brings `.oneshot()` into scope

async fn hello() -> &'static str { "hi" }

async fn call(app: &Router, method: &str, uri: &str) -> (StatusCode, String) {
    let request = Request::builder()
        .method(method)
        .uri(uri)
        .body(Body::empty())
        .expect("static request is valid");        // OK in test helpers

    // `oneshot` consumes the service, so clone the Router (cheap: it is Arc-based inside)
    let response = match app.clone().oneshot(request).await {
        Ok(r) => r,
        Err(never) => match never {},              // a Router's error type is Infallible
    };

    let status = response.status();
    let bytes = to_bytes(response.into_body(), usize::MAX).await.unwrap_or_default();
    (status, String::from_utf8_lossy(&bytes).into_owned())
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(hello));
    println!("{:?}", call(&app, "GET", "/").await);       // (200, "hi")
    println!("{:?}", call(&app, "POST", "/").await);      // (405, "")
    println!("{:?}", call(&app, "GET", "/nope").await);   // (404, "")
}
```

`Err(never) => match never {}` is a neat trick: `Infallible` has no values, so an empty `match` proves to the compiler that this branch can never happen, without `unwrap()`.

---

## 9. How this looks in a production service

In a layered service (see the README architecture), routing lives in the **`api`** crate and follows a few conventions:

```text
crates/api/src/
├── main.rs            # build state, build router, axum::serve
├── state.rs           # AppState
└── routes/
    ├── mod.rs         # fn router(state) -> Router: nests every module
    ├── health.rs
    ├── projects.rs    # fn router() -> Router<AppState>
    └── users.rs
```

```rust
// routes/projects.rs
pub fn router() -> Router<AppState> {
    Router::new()
        .route("/projects", get(list).post(create))
        .route("/projects/{id}", get(get_one).put(update).delete(remove))
}

// routes/mod.rs
pub fn router(state: AppState) -> Router {
    let api = Router::new()
        .merge(projects::router())
        .merge(users::router());

    Router::new()
        .route("/health", get(health::check))
        .nest("/api", api)
        .fallback(not_found)
        .with_state(state)
}
```

Things to notice:

- **One `router()` function per resource.** Adding a module means adding one `.merge(...)` line.
- **`Router<AppState>`**: the type parameter means "this router still **needs** an `AppState`". `.with_state(state)` provides it and turns it into a plain `Router` (`Router<()>`), which is what `axum::serve` accepts. State is covered in Stage 5.
- **Handlers stay thin.** They parse the input, call the `domain` layer, and map the result to a response. No business rules and no SQL in the routing layer.
- **Startup builds the full router once.** Because conflicts panic at build time, a test that only builds `routes::router(test_state)` already catches routing mistakes.

---

## 10. Putting it all together

A small read-only projects API with nesting, method chaining, path parameters, a JSON fallback, and status codes. It calls itself through `oneshot`, so it runs **in the Playground** with no network.

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
    extract::Path,
    http::{header, Request, StatusCode, Uri},
    response::IntoResponse,
    routing::get,
    Json, Router,
};
use serde::Serialize;
use serde_json::{json, Value};
use tower::ServiceExt;

#[derive(Clone, Serialize)]
struct Project {
    id: u32,
    name: &'static str,
}

const PROJECTS: [Project; 2] = [
    Project { id: 1, name: "Alpha" },
    Project { id: 2, name: "Beta" },
];

// ---------- handlers ----------

async fn health() -> &'static str {
    "ok"
}

async fn list_projects() -> Json<Vec<Project>> {
    Json(PROJECTS.to_vec())
}

async fn get_project(Path(id): Path<u32>) -> Result<Json<Project>, StatusCode> {
    PROJECTS
        .iter()
        .find(|p| p.id == id)
        .cloned()
        .map(Json)
        .ok_or(StatusCode::NOT_FOUND)
}

async fn create_project() -> impl IntoResponse {
    (
        StatusCode::CREATED,
        [(header::LOCATION, "/api/v1/projects/3")],
        "created",
    )
}

async fn delete_project(Path(id): Path<u32>) -> StatusCode {
    if PROJECTS.iter().any(|p| p.id == id) {
        StatusCode::NO_CONTENT
    } else {
        StatusCode::NOT_FOUND
    }
}

async fn not_found(uri: Uri) -> (StatusCode, Json<Value>) {
    (
        StatusCode::NOT_FOUND,
        Json(json!({ "error": "not_found", "path": uri.path() })),
    )
}

// ---------- routing ----------

fn project_routes() -> Router {
    Router::new()
        .route("/projects", get(list_projects).post(create_project))
        .route("/projects/{id}", get(get_project).delete(delete_project))
}

fn app() -> Router {
    Router::new()
        .route("/health", get(health))
        .nest("/api/v1", project_routes())
        .fallback(not_found)
}

// ---------- a tiny client that calls the router directly ----------

async fn call(app: &Router, method: &str, uri: &str) -> (StatusCode, String) {
    let request = Request::builder()
        .method(method)
        .uri(uri)
        .body(Body::empty())
        .expect("static request is valid");

    let response = match app.clone().oneshot(request).await {
        Ok(r) => r,
        Err(never) => match never {},
    };

    let status = response.status();
    let bytes = to_bytes(response.into_body(), usize::MAX)
        .await
        .unwrap_or_default();
    (status, String::from_utf8_lossy(&bytes).into_owned())
}

#[tokio::main]
async fn main() {
    let app = app();

    let calls = [
        ("GET", "/health"),
        ("GET", "/api/v1/projects"),
        ("GET", "/api/v1/projects/2"),
        ("GET", "/api/v1/projects/9"),
        ("GET", "/api/v1/projects/abc"),
        ("POST", "/api/v1/projects"),
        ("DELETE", "/api/v1/projects/1"),
        ("PUT", "/api/v1/projects/1"),
        ("GET", "/api/v2/projects"),
    ];

    for (method, uri) in calls {
        let (status, body) = call(&app, method, uri).await;
        println!("{method:<6} {uri:<24} -> {} {body}", status.as_u16());
    }
}
```

Expected output (the exact text of the 400 message can differ between Axum versions):

```text
GET    /health                  -> 200 ok
GET    /api/v1/projects         -> 200 [{"id":1,"name":"Alpha"},{"id":2,"name":"Beta"}]
GET    /api/v1/projects/2       -> 200 {"id":2,"name":"Beta"}
GET    /api/v1/projects/9       -> 404 
GET    /api/v1/projects/abc     -> 400 Invalid URL: Cannot parse `abc` to a `u32`
POST   /api/v1/projects         -> 201 created
DELETE /api/v1/projects/1       -> 204 
PUT    /api/v1/projects/1       -> 405 
GET    /api/v2/projects         -> 404 {"error":"not_found","path":"/api/v2/projects"}
```

What each line shows:

| Call | Concept |
|---|---|
| `/health` | A plain route outside the nested API |
| `/api/v1/projects` | `nest` prefix + `Json` response |
| `/projects/2` vs `/projects/9` | `Result<Json<_>, StatusCode>`: `Ok` → 200, `Err` → 404 |
| `/projects/abc` | `Path<u32>` rejects bad input with 400 before the handler runs |
| `POST` | Method chaining on one path + `201` with a `Location` header |
| `DELETE` | `204 No Content` |
| `PUT` | Path exists, method doesn't → automatic **405** |
| `/api/v2/...` | No route matched → custom JSON **fallback** |

Notice the two different 404s: the one from `get_project` has an **empty** body, and the fallback has a JSON body. In Stage 4, one `AppError` type makes every error response consistent.

---

## 11. Exercises

Run them with `oneshot` (section 8) in the Playground, or locally with `axum::serve` and `curl`.

- [ ] **1. First router:** build a router with `GET /` returning `"hello"` and `GET /version` returning `Json` of `{"version":"0.1.0"}`. Call both and check the status codes and bodies.
- [ ] **2. Method chaining and 405:** add `GET /items` and `POST /items` on **one** `.route(...)` call. Send `DELETE /items` and print the status and the `Allow` response header.
- [ ] **3. Path syntax:** add `GET /users/{id}/orders/{order_id}` that returns `"order {order_id} of user {id}"` (use `Path<(u32, u32)>`). Then change the route to the old `/users/:id/...` syntax and read the startup panic message.
- [ ] **4. Wildcard:** add `GET /files/{*path}` that returns the captured path (`Path<String>`). Test `/files/a/b/c.txt` and `/files`. Explain why the second one is a 404.
- [ ] **5. Static beats capture:** register `/projects/new` and `/projects/{id}` together. Show which handler `/projects/new` and `/projects/5` reach.
- [ ] **6. Modules:** write `fn user_routes() -> Router` and `fn order_routes() -> Router`, `merge` them into one API router, and `nest` it under `/api/v1`. Verify that `/api/v1/users` works and `/users` returns 404.
- [ ] **7. Fallback:** add a fallback that returns `404` with `{"error":"not_found","method":"...","path":"..."}`. Hint: the fallback handler can take `method: axum::http::Method` and `uri: Uri` arguments.
- [ ] **8. Conflicts:** register `GET /projects` twice in two separate `.route(...)` calls, and separately `merge` two routers that both define `/health`. Note what happens and **when** it happens. Write one sentence on why failing at startup is better than failing per request.

> When you can solve exercises 1–8 without looking anything up, you're ready for **Stage 3: Extractors**.

---

## 12. Further reading

- [`axum::Router` docs](https://docs.rs/axum/latest/axum/struct.Router.html): `route`, `nest`, `merge`, `fallback`, `with_state`, and the path syntax rules
- [`axum::routing` docs](https://docs.rs/axum/latest/axum/routing/index.html): `MethodRouter` and every method function
- [`axum::response` docs](https://docs.rs/axum/latest/axum/response/index.html): everything that implements `IntoResponse`
- [Announcing Axum 0.8](https://tokio.rs/blog/2025-01-01-announcing-axum-0-8-0): the `{id}` path syntax change and other breaking changes
- [Axum examples](https://github.com/tokio-rs/axum/tree/main/examples): look at `hello-world`, `routes-and-handlers-close-together`, and `testing`
- [MDN: HTTP request methods](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods) and [HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
