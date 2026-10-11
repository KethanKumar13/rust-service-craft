# Reference Service: Multi-Tenant Projects API

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> This is the capstone. It combines every earlier stage into one service and is built in milestones, each a focused branch and PR.

## Contents

1. [Goal and scope](#1-goal-and-scope)
2. [The domain](#2-the-domain)
3. [Architecture and workspace layout](#3-architecture-and-workspace-layout)
4. [Crate responsibilities and dependency rules](#4-crate-responsibilities-and-dependency-rules)
5. [Database schema](#5-database-schema)
6. [API surface](#6-api-surface)
7. [Cross-cutting design](#7-cross-cutting-design)
8. [Key code skeletons](#8-key-code-skeletons)
9. [Testing plan](#9-testing-plan)
10. [Tooling and CI](#10-tooling-and-ci)
11. [Milestones](#11-milestones)
12. [Definition of done](#12-definition-of-done)
13. [Possible extensions](#13-possible-extensions)
14. [Stage map](#14-stage-map)

---

## 1. Goal and scope

Build a small but complete HTTP service that a team could run in production:

- **Multi-tenant:** many organizations share one database; each sees only its own data.
- **Authenticated:** password login, JWT in an HttpOnly cookie, revocation on logout.
- **Authorized:** roles map to capabilities; a narrow super-admin path for cross-tenant operations.
- **Safe to change concurrently:** optimistic locking on every editable resource.
- **Observable:** structured tracing with request IDs.
- **Realtime:** presence on a project page over WebSockets.
- **Scalable for large data:** NDJSON export streamed with bounded memory.
- **Verified:** handler, database, and streaming tests run in CI.

**Out of scope** (on purpose): a frontend, email, file storage, multi-region deployment, and an ORM. Section 13 lists optional extensions.

---

## 2. The domain

| Entity | Belongs to | Key fields |
|---|---|---|
| `Tenant` | — | `id`, `name` |
| `User` | Tenant | `email`, `password_hash`, `role`, `is_super_admin` |
| `Project` | Tenant | `name` (unique per tenant), `description`, `status`, `version` |
| `Task` | Project (and Tenant) | `title`, `done`, `version` |

Rules:

| Rule | Where it's enforced |
|---|---|
| Project names are unique within a tenant | `UNIQUE (tenant_id, name)` → 409 |
| `Live` projects can't be edited | `domain` → 409 |
| Status only moves forward: `draft → in_progress → completed → live` | `domain` → 400 |
| Every update sends the `version` it read | `db` `WHERE version = $n` → 409 |
| Tasks can only be created in a project of the same tenant | `domain` check + scoped query → 404 |
| Viewers read; editors write; admins also manage users | capabilities → 403 |

---

## 3. Architecture and workspace layout

```text
rust-service-craft/
├── Cargo.toml                         # workspace root
├── rust-toolchain.toml
├── clippy.toml
├── rustfmt.toml
├── .config/nextest.toml
├── .github/workflows/ci.yml
├── docker-compose.yml                 # postgres for dev and tests
├── docs/notes/                        # stages 00–12
└── projects/reference-service/
    ├── .env.example
    ├── migrations/                    # NNNN_name.up.sql / .down.sql
    ├── .sqlx/                         # offline query data (committed)
    └── crates/
        ├── shared/                    # config, AppError, DTOs, IDs
        ├── db/                        # *Row structs, repository functions
        ├── domain/                    # rules, auth (password, jwt, authz)
        └── api/                       # Axum: routes, extractors, middleware, realtime, main
            ├── src/
            │   ├── main.rs
            │   ├── lib.rs             # pub fn app(state) -> Router
            │   ├── state.rs
            │   ├── extractors/        # AuthUser, AppJson, AppPath, ValidJson
            │   ├── middleware/
            │   ├── realtime/          # presence, protocol, socket
            │   └── routes/            # auth, projects, tasks, users, admin, export, health
            └── tests/                 # oneshot + websocket + streaming tests
```

The dependency direction (README architecture):

```text
api ──▶ domain ──▶ db ──▶ shared
 └──────────────────────────▲
```

`api` may use `shared` directly (DTOs, `AppError`). Nothing depends on `api`.

---

## 4. Crate responsibilities and dependency rules

| Crate | Owns | Depends on | Must not |
|---|---|---|---|
| `shared` | `Config`, `Secret`, `AppError` + `IntoResponse`, DTOs, `TenantId`/`ProjectId`/`UserId`, `Role`, `caps` | `serde`, `thiserror`, `axum` (for `IntoResponse` only) | Depend on any other crate here |
| `db` | `*Row` structs, free functions per table, `sqlx` mapping | `shared`, `sqlx` | Make authorization decisions; return DTOs |
| `domain` | Business rules, `DomainError`, password hashing, JWT issue/verify, capability checks, 404-vs-409 decisions | `shared`, `db` | Import HTTP types (`axum`, `StatusCode`) |
| `api` | Router, handlers, extractors, middleware, WebSockets, streaming, `main` | all of the above | Contain SQL or business rules |

Two conversions keep it clean:

```text
sqlx::Error ──From──▶ DomainError ──From──▶ AppError ──IntoResponse──▶ HTTP
ProjectRow  ──From──▶ ProjectDto  ──Json──▶ client
```

---

## 5. Database schema

```sql name=projects/reference-service/migrations/0001_init.up.sql
CREATE TYPE user_role AS ENUM ('viewer', 'editor', 'admin');
CREATE TYPE project_status AS ENUM ('draft', 'in_progress', 'completed', 'live');

CREATE TABLE tenants (
    id         SERIAL PRIMARY KEY,
    name       TEXT NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id             SERIAL PRIMARY KEY,
    tenant_id      INTEGER NOT NULL REFERENCES tenants (id),
    email          TEXT NOT NULL UNIQUE,
    password_hash  TEXT NOT NULL,
    role           user_role NOT NULL DEFAULT 'viewer',
    is_super_admin BOOLEAN NOT NULL DEFAULT false,
    version        INTEGER NOT NULL DEFAULT 1,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX users_tenant_id_idx ON users (tenant_id);

CREATE TABLE projects (
    id          SERIAL PRIMARY KEY,
    tenant_id   INTEGER NOT NULL REFERENCES tenants (id),
    name        TEXT NOT NULL CHECK (length(trim(name)) BETWEEN 1 AND 100),
    description TEXT,
    status      project_status NOT NULL DEFAULT 'draft',
    version     INTEGER NOT NULL DEFAULT 1,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT projects_name_unique_per_tenant UNIQUE (tenant_id, name)
);
CREATE INDEX projects_tenant_id_idx ON projects (tenant_id);

CREATE TABLE tasks (
    id         SERIAL PRIMARY KEY,
    tenant_id  INTEGER NOT NULL REFERENCES tenants (id),
    project_id INTEGER NOT NULL REFERENCES projects (id) ON DELETE CASCADE,
    title      TEXT NOT NULL CHECK (length(trim(title)) BETWEEN 1 AND 200),
    done       BOOLEAN NOT NULL DEFAULT false,
    version    INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX tasks_tenant_project_idx ON tasks (tenant_id, project_id);

CREATE TABLE audit_log (
    id         BIGSERIAL PRIMARY KEY,
    tenant_id  INTEGER NOT NULL REFERENCES tenants (id),
    user_id    INTEGER NOT NULL REFERENCES users (id),
    action     TEXT NOT NULL,
    entity     TEXT NOT NULL,
    entity_id  INTEGER NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX audit_log_tenant_idx ON audit_log (tenant_id, created_at);
```

Design notes:

- `tasks` carries its own `tenant_id`, so every task query can be scoped without a join.
- `audit_log` rows are written **in the same transaction** as the change they describe (Stage 6, section 10).
- Seeding (a demo tenant and an admin user) is a startup step driven by config, not a migration, so passwords never live in SQL files.

---

## 6. API surface

All routes are under `/api`. Errors always use `{"error": "<code>", "message": "<text>"}`.

| Method | Path | Auth | Capability | Success | Notable errors |
|---|---|---|---|---|---|
| `GET` | `/health` | — | — | 200 | 503 if DB unreachable |
| `POST` | `/auth/login` | — | — | 200 + `Set-Cookie` | 401 (generic) |
| `POST` | `/auth/logout` | ✅ | — | 204 + cookie removed | 401 |
| `GET` | `/auth/me` | ✅ | — | 200 | 401 |
| `GET` | `/projects?page=&per_page=` | ✅ | `projects:read` | 200 | 400 (page size > max) |
| `POST` | `/projects` | ✅ | `projects:write` | 201 + `Location` | 400, 409 (duplicate) |
| `GET` | `/projects/{id}` | ✅ | `projects:read` | 200 | 404 |
| `PUT` | `/projects/{id}` | ✅ | `projects:write` | 200 | 400, 404, 409 (version, live) |
| `DELETE` | `/projects/{id}` | ✅ | `projects:write` | 204 | 404 |
| `GET` | `/projects/{id}/tasks` | ✅ | `projects:read` | 200 | 404 |
| `POST` | `/projects/{id}/tasks` | ✅ | `projects:write` | 201 | 400, 404 |
| `PUT` | `/tasks/{id}` | ✅ | `projects:write` | 200 | 404, 409 |
| `GET` | `/projects/export` | ✅ | `projects:read` | 200 NDJSON stream | 401, 403, 429 (too many exports) |
| `GET` | `/users` | ✅ | `users:manage` | 200 | 403 |
| `POST` | `/users` | ✅ | `users:manage` | 201 | 400, 409 |
| `GET` | `/admin/tenants/{id}/projects` | ✅ | super admin | 200 + audit log | 403 |
| `GET` (WS) | `/ws/projects/{id}` | ✅ cookie | `projects:read` | 101 | 401, 403 (origin), 404 |

---

## 7. Cross-cutting design

### 7.1 Configuration (Stage 5)

| Variable | Required | Default | Validation |
|---|---|---|---|
| `DATABASE_URL` | ✅ | — | starts with `postgres://` |
| `JWT_SECRET_KEY` | ✅ | — | ≥ 32 bytes |
| `PORT` | | `8000` | `u16` |
| `TOKEN_TTL_MINUTES` | | `60` | 1–1440 |
| `CORS_ORIGINS` | | empty | comma-separated list |
| `SECURE_COOKIES` | | `true` | bool |
| `REQUEST_TIMEOUT_SECS` | | `15` | 1–300 |
| `MAX_PAGE_SIZE` | | `100` | 1–500 |
| `MAX_CONCURRENT_EXPORTS` | | `2` | 1–20 |
| `SEED_ADMIN_EMAIL` / `SEED_ADMIN_PASSWORD` | | — | both or neither |

Parsed once with `Config::from_lookup`; secrets wrapped in `Secret`.

### 7.2 `AppState`

```rust
#[derive(Clone)]
pub struct AppState {
    pub pool: PgPool,
    pub config: Arc<Config>,
    pub jwt: Arc<JwtKeys>,
    pub blocklist: Blocklist,
    pub presence: Presence,
    pub exports: Arc<Semaphore>,      // MAX_CONCURRENT_EXPORTS
    pub dummy_hash: Arc<str>,         // constant-time-ish login
}
```

### 7.3 Middleware stack (Stage 7), outermost first

```text
SetRequestId → Trace → PropagateRequestId → CatchPanic (AppError JSON)
  → CORS → Compression
     ├── normal routes: Timeout(REQUEST_TIMEOUT_SECS) + DefaultBodyLimit(1 MB)
     └── export + websocket routes: no global timeout, own limits
```

Auth is an `AuthUser` extractor on every protected handler, plus a `route_layer` on the protected router as a safety net.

### 7.4 Error model (Stage 4)

`AppError { NotFound, BadRequest, Unauthorized, Forbidden, Conflict, TooManyRequests, Internal }`. Every rejection (`Json`, `Path`, `Query`, auth, panic, fallback) produces the same JSON shape. `Internal` is logged with its full chain and the request ID; the client sees a generic message.

### 7.5 Realtime (Stage 9)

Rooms keyed by `(TenantId, ProjectId)`, `broadcast` capacity 64, `Drop` guard per connection, 20 s heartbeat, 64 KB message limit, origin check from `CORS_ORIGINS`, socket closed at token `exp`. Project updates (`PUT /projects/{id}`) also publish a `ProjectUpdated { version }` event to the room so open pages can refresh.

### 7.6 Streaming export (Stage 10)

Checks first → acquire an export permit (`try_acquire` → 429) → producer task with `fetch` → bounded `mpsc(16)` → `ready_chunks(256)` → NDJSON body with a final `{"type":"end","rows":N}` or `{"type":"error",...}` line. The permit is moved into the producer task so it's released when the export ends or the client disconnects.

---

## 8. Key code skeletons

### 8.1 Workspace root

```toml name=Cargo.toml
[workspace]
resolver = "2"
members = ["projects/reference-service/crates/*"]

[workspace.package]
edition = "2021"
version = "0.1.0"

[workspace.dependencies]
anyhow = "1"
argon2 = "0.5"
axum = { version = "0.8", features = ["ws", "macros"] }
axum-extra = { version = "0.10", features = ["cookie"] }
dotenvy = "0.15"
futures = "0.3"
jsonwebtoken = "9"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.8", features = ["runtime-tokio", "tls-rustls", "postgres", "macros", "migrate", "chrono"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "2"
time = "0.3"
tokio = { version = "1", features = ["full"] }
tower = { version = "0.5", features = ["util"] }
tower-http = { version = "0.6", features = ["trace", "cors", "timeout", "limit", "request-id", "catch-panic", "compression-gzip", "util"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
uuid = { version = "1", features = ["v4"] }

[workspace.lints.clippy]
all = { level = "warn", priority = -1 }
unwrap_used = "deny"
expect_used = "deny"
panic = "deny"
indexing_slicing = "warn"
```

Each crate opts in with:

```toml
[lints]
workspace = true
```

### 8.2 `api/src/lib.rs`

```rust
pub mod extractors;
pub mod middleware;
pub mod realtime;
pub mod routes;
pub mod state;

pub use state::AppState;

/// Builds the full router. `main` and every test call this.
pub fn app(state: AppState) -> axum::Router {
    routes::router(state)
}
```

### 8.3 `api/src/main.rs`

```rust
use anyhow::Context;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let _ = dotenvy::dotenv();
    api::middleware::init_tracing();

    let config = shared::config::Config::from_env().context("invalid configuration")?;
    let state = api::AppState::build(config.clone()).await.context("startup failed")?;
    // build(): connect pool, run migrations, seed admin, start blocklist purge task

    let listener = tokio::net::TcpListener::bind(("0.0.0.0", config.port)).await?;
    tracing::info!(port = config.port, "listening");
    axum::serve(listener, api::app(state))
        .with_graceful_shutdown(api::middleware::shutdown_signal())
        .await?;
    Ok(())
}
```

### 8.4 `api/src/routes/mod.rs`

```rust
pub fn router(state: AppState) -> Router {
    let public = Router::new()
        .route("/health", get(health::check))
        .route("/auth/login", post(auth::login));

    let protected = Router::new()
        .merge(auth::protected_routes())
        .merge(projects::router())
        .merge(tasks::router())
        .merge(users::router())
        .merge(admin::router())
        .layer(DefaultBodyLimit::max(1024 * 1024))
        .layer(TimeoutLayer::new(state.config.request_timeout))
        .route_layer(middleware::from_fn_with_state(state.clone(), crate::middleware::require_auth));

    let long_running = Router::new()
        .merge(export::router())
        .merge(crate::realtime::router())
        .route_layer(middleware::from_fn_with_state(state.clone(), crate::middleware::require_auth));

    Router::new()
        .nest("/api", public.merge(protected).merge(long_running))
        .fallback(not_found)
        .layer(crate::middleware::global_stack(&state.config))
        .with_state(state)
}
```

### 8.5 A handler, end to end

```rust
pub async fn update(
    State(state): State<AppState>,
    user: AuthUser,
    AppPath(id): AppPath<ProjectId>,
    ValidJson(body): ValidJson<UpdateProject>,
) -> Result<Json<ProjectDto>, AppError> {
    user.require(caps::PROJECTS_WRITE)?;
    let row = domain::projects::update(&state.pool, user.tenant_id, user.user_id, id, body).await?;
    state.presence.publish((user.tenant_id, id), ServerEvent::ProjectUpdated { version: row.version });
    Ok(Json(row.into()))
}
```

---

## 9. Testing plan

| Suite | Location | Kind | Examples |
|---|---|---|---|
| Domain unit | `domain/src/**` | `#[test]` | `status_cannot_move_backwards`, `rejects_edit_of_live_project` |
| Config unit | `shared/src/config.rs` | `#[test]` | `rejects_short_jwt_secret`, `parses_cors_origin_list` |
| Repository | `db/tests/` | `#[sqlx::test]` | `get_by_id_returns_none_for_other_tenant`, `update_with_stale_version_returns_none` |
| HTTP | `api/tests/http_*.rs` | `oneshot` | `returns_409_on_stale_version`, `rejects_viewer_creating_project` |
| Tenant isolation | `api/tests/tenancy.rs` | `oneshot` + DB | One test per resource × (read, update, delete) |
| Auth | `api/tests/auth.rs` | `oneshot` + DB | `rejects_expired_token`, `rejects_revoked_token_after_logout`, `login_error_is_generic` |
| Streaming | `api/tests/export.rs` | body stream | `export_emits_first_line_before_source_is_exhausted`, `export_keeps_resident_rows_bounded_as_input_grows`, `export_stops_producer_when_client_disconnects`, `export_returns_429_over_limit` |
| Realtime | `api/tests/realtime.rs` | real socket | `second_user_sees_first_user_join`, `rooms_do_not_cross_tenants`, `rejects_foreign_origin` |
| Time | various | `start_paused` | `purges_expired_blocklist_entries`, `closes_socket_at_token_expiry` |

Shared test helpers in `api/tests/common/`: `TestApp`, `token_for(...)`, `seed_tenant(...)`, `spawn_server(...)`.

API tests that share a database run in the `api-integration-db` nextest group with `max-threads = 1`. Everything else runs in parallel.

---

## 10. Tooling and CI

| File | Purpose |
|---|---|
| `rust-toolchain.toml` | Pin `stable` with `clippy` and `rustfmt` |
| `rustfmt.toml` | `max_width = 100`, `imports_granularity = "Crate"` (nightly-only options left out) |
| `clippy.toml` | `allow-unwrap-in-tests = true`, `allow-expect-in-tests = true` |
| `.config/nextest.toml` | `ci` profile, slow-test timeout, `api-integration-db` group |
| `docker-compose.yml` | `postgres:16` for development and tests |
| `.github/workflows/ci.yml` | fmt check, clippy `-D warnings`, `sqlx prepare --check`, nextest (Stage 11, section 12) |
| `.env.example` | Every variable from section 7.1 with fake values |

---

## 11. Milestones

Each milestone is one branch and one PR, merged when its checks pass. Commit messages follow Conventional Commits.

- [ ] **M0: Workspace skeleton.** Root `Cargo.toml`, the four empty crates, toolchain files, `.gitignore`, CI running fmt + clippy + nextest on an empty test. Branch `chore/workspace-setup`.
- [ ] **M1: Shared foundations.** `Config` + `Secret` + tests, `AppError` + `IntoResponse` + tests, ID newtypes, `Role` and `caps`. Branch `feat/shared-foundations`.
- [ ] **M2: Schema and repositories.** Migrations, `db::tenants/users/projects/tasks/audit`, `#[sqlx::test]` suite, `.sqlx/` committed. Branch `feat/db-repositories`.
- [ ] **M3: Domain rules.** `DomainError`, project and task rules, 404-vs-409 logic, audit in transactions, unit tests. Branch `feat/domain-rules`.
- [ ] **M4: HTTP skeleton.** `AppState::build`, router, `AppJson`/`AppPath`/`ValidJson`, health and project/task CRUD with a temporary test-only auth, `oneshot` tests. Branch `feat/api-crud`.
- [ ] **M5: Authentication.** Argon2, JWT cookie login/logout/me, `AuthUser`, blocklist with purge task, admin seeding, auth tests. Branch `feat/auth`.
- [ ] **M6: Authorization and tenancy.** Capability checks everywhere, users management, super-admin routes with audit, full tenant-isolation suite. Branch `feat/authz-tenancy`.
- [ ] **M7: Middleware and observability.** Global stack, tracing setup, request IDs in logs, consistent panic body, CORS from config. Branch `feat/middleware`.
- [ ] **M8: Streaming export.** Export endpoint with permits, end/error markers, the three streaming property tests and the 429 test. Branch `feat/export-streaming`.
- [ ] **M9: Realtime presence.** WebSocket route, presence, `ProjectUpdated` events, heartbeat, expiry close, realtime tests. Branch `feat/realtime-presence`.
- [ ] **M10: Polish.** README "Getting started" with real commands, CI badge, architecture diagram check, a short operations section (config table, health check, logs). Branch `docs/reference-service`.

---

## 12. Definition of done

The reference service is complete when:

- [ ] `docker compose up -d postgres && cargo run -p api` starts the service with only `.env` values.
- [ ] `cargo nextest run --workspace` passes locally and in CI.
- [ ] `cargo clippy --workspace --all-targets -- -D warnings` is clean, with no `#[allow]` (only `#[expect(..., reason = "...")]`).
- [ ] No `unwrap()` / `expect()` / `panic!` in runtime code.
- [ ] Every tenant-owned query takes a `TenantId`, and each resource has read/update/delete isolation tests.
- [ ] Every editable resource has a stale-version test returning 409.
- [ ] Every `AppError` variant has a test checking status and JSON body.
- [ ] The export passes the early-output, bounded-residency, and cancellation tests.
- [ ] Secrets never appear in logs (checked with a test that formats `Config` with `{:?}`).
- [ ] The README describes how to run, test, and configure the service.

---

## 13. Possible extensions

| Extension | Builds on |
|---|---|
| Refresh tokens stored in the database; revocation across replicas | Stage 8 |
| Rate limiting on `/auth/login` (per IP and per email) | Stages 7, 8 |
| OpenAPI document with `utoipa` and a Swagger UI route | Stages 2, 3 |
| PostgreSQL `LISTEN/NOTIFY` fan-out so presence works across replicas | Stages 6, 9 |
| CSV import with a streaming upload and per-line validation report | Stages 3, 10 |
| Keyset pagination for project lists | Stage 6 |
| Dockerfile (multi-stage, distroless runtime) and a `compose` file for the full stack | — |
| Metrics endpoint (request counts, latencies, pool usage) | Stage 7 |
| A second HTTP framework comparison: the health and projects routes rewritten in Dropshot | Stages 2–4 |

---

## 14. Stage map

| Stage | Used in the reference service as |
|---|---|
| 00 foundations | Newtype IDs, enums with `match`, `Result` + `?`, `Arc` |
| 01 async-runtime | Producer tasks, `select!` loops, purge task, `spawn_blocking` for Argon2 |
| 02 routing | `Router<AppState>` per module, `nest`/`merge`, fallback |
| 03 extractors | `AppJson`, `AppPath`, `ValidJson`, `AuthUser` |
| 04 errors | `AppError`, `DomainError`, `From` chain, generic 500s |
| 05 state-and-config | `AppState`, `Config::from_lookup`, `Secret` |
| 06 persistence | Migrations, repositories, transactions, optimistic locking, tenant scoping |
| 07 middleware | Global Tower stack, tracing, request IDs, timeouts, CORS |
| 08 auth | Argon2, JWT cookie, blocklist, capabilities, super admin |
| 09 realtime | Presence rooms, broadcast, heartbeat |
| 10 streaming | NDJSON export with bounded channel and permits |
| 11 testing | `oneshot`, `#[sqlx::test]`, nextest groups, paused time, streaming property tests |
