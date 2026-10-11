# Stage 6: Persistence with SQLx

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> If you come from JavaScript or TypeScript, the comparisons below map Rust concepts to familiar JS ideas (mostly `pg`, Knex, and Prisma).

## Contents

1. [Why SQLx](#1-why-sqlx)
2. [Setup: PostgreSQL and the crate](#2-setup-postgresql-and-the-crate)
3. [The connection pool](#3-the-connection-pool)
4. [Migrations](#4-migrations)
5. [Queries: `query`, `query_as`, and the fetch methods](#5-queries-query-query_as-and-the-fetch-methods)
6. [Compile-time checked queries: `query!` and `query_as!`](#6-compile-time-checked-queries-query-and-query_as)
7. [Mapping types: rows, newtypes, enums, NULL](#7-mapping-types-rows-newtypes-enums-null)
8. [The repository pattern](#8-the-repository-pattern)
9. [Tenant scoping](#9-tenant-scoping)
10. [Transactions](#10-transactions)
11. [Optimistic locking with `version`](#11-optimistic-locking-with-version)
12. [Errors from the database](#12-errors-from-the-database)
13. [Pagination and large results](#13-pagination-and-large-results)
14. [How this looks in a production service](#14-how-this-looks-in-a-production-service)
15. [Putting it all together](#15-putting-it-all-together)
16. [Exercises](#16-exercises)
17. [Further reading](#17-further-reading)

---

## 1. Why SQLx

There are three common ways to talk to a SQL database from Rust:

| Crate | Style | You write |
|---|---|---|
| **SQLx** | Async SQL toolkit | Plain SQL. Results are mapped to structs, and the SQL can be checked against the real schema **at compile time** |
| Diesel | ORM / query builder (sync, with an async add-on) | Rust code that generates SQL |
| SeaORM | Async ORM built on SQLx | Entities and a query builder |

SQLx is the most common choice with Axum. You write **real SQL**, so you keep full control (CTEs, `RETURNING`, `ON CONFLICT`, JSONB, window functions), and the `query!` macros catch typos in column names and type mismatches **before** the program runs.

> **JS comparison:** SQLx is close to `pg` (node-postgres): `pool.query("SELECT ... WHERE id = $1", [id])`. The difference is that SQLx can check at compile time that the query is valid and that the result matches your struct, which Prisma does with a generated client and plain `pg` doesn't do at all.

SQLx is **not** an ORM: no lazy loading, no automatic relations, no hidden queries. Every query in the code is a query that runs.

---

## 2. Setup: PostgreSQL and the crate

### 2.1 PostgreSQL with Docker

```yaml name=docker-compose.yml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

```bash
docker compose up -d postgres
```

Connection string: `postgres://app:app@localhost:5432/app`, set as `DATABASE_URL` in `.env` (Stage 5).

### 2.2 The crate

```toml
[dependencies]
sqlx = { version = "0.8", features = [
    "runtime-tokio",     # use Tokio (Stage 1)
    "tls-rustls",        # TLS for real databases
    "postgres",          # the driver
    "macros",            # query!, query_as!, FromRow derive
    "migrate",           # sqlx::migrate!
    "chrono",            # DateTime<Utc> <-> timestamptz
] }
chrono = { version = "0.4", features = ["serde"] }
```

### 2.3 The CLI

```bash
cargo install sqlx-cli --no-default-features --features rustls,postgres
sqlx database create        # creates the database from DATABASE_URL
sqlx migrate add -r create_projects
sqlx migrate run
```

> **Playground note:** the Playground has no database, so the examples in this stage need a local project. The *shapes* (structs, `FromRow`, error mapping) still compile there, but nothing can connect.

---

## 3. The connection pool

Opening a PostgreSQL connection is slow (TCP, TLS, authentication). A **pool** keeps a set of open connections and lends them to queries.

```rust
use std::time::Duration;
use sqlx::postgres::PgPoolOptions;

let pool = PgPoolOptions::new()
    .max_connections(10)
    .acquire_timeout(Duration::from_secs(5))   // fail instead of waiting forever
    .connect(config.database_url.expose())
    .await?;
```

- `PgPool` is **cheap to clone**: it's an `Arc` inside. That's why it goes straight into `AppState` (Stage 5).
- Every query **borrows** a connection and returns it when done.
- **`max_connections` is a hard limit.** If all connections are busy, the next query waits up to `acquire_timeout`, then fails. That's a form of backpressure: the database is protected from too many concurrent queries.
- PostgreSQL itself has a connection limit (often 100). The sum over all replicas of a service must stay below it: 5 replicas × 10 connections = 50.

> **JS comparison:** the same as `new Pool({ max: 10 })` in `pg`.

---

## 4. Migrations

A **migration** is a versioned SQL file that changes the schema. Migrations are applied in order and recorded in a `_sqlx_migrations` table, so each one runs exactly once per database.

```bash
sqlx migrate add -r create_projects
# migrations/20261011120000_create_projects.up.sql
# migrations/20261011120000_create_projects.down.sql
```

```sql name=migrations/20261011120000_create_projects.up.sql
CREATE TABLE tenants (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TYPE project_status AS ENUM ('draft', 'in_progress', 'completed', 'live');

CREATE TABLE projects (
    id          SERIAL PRIMARY KEY,
    tenant_id   INTEGER NOT NULL REFERENCES tenants (id),
    name        TEXT NOT NULL,
    description TEXT,
    status      project_status NOT NULL DEFAULT 'draft',
    version     INTEGER NOT NULL DEFAULT 1,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT projects_name_unique_per_tenant UNIQUE (tenant_id, name)
);

CREATE INDEX projects_tenant_id_idx ON projects (tenant_id);
```

```sql name=migrations/20261011120000_create_projects.down.sql
DROP TABLE projects;
DROP TYPE project_status;
DROP TABLE tenants;
```

Running them at startup:

```rust
sqlx::migrate!("./migrations").run(&pool).await?;
```

`migrate!` **embeds** the SQL files into the binary at compile time, so the deployed service carries its own migrations.

Rules:

- **Never edit a migration that has already run** anywhere shared. SQLx stores a checksum and refuses to start if a file changed. Add a new migration instead.
- Schema rules belong in the database too: `NOT NULL`, `UNIQUE`, `REFERENCES`, `CHECK`. Application validation (Stage 3) gives good error messages, and constraints guarantee the data stays correct even if there's a bug.
- Every `tenant_id` column gets an **index**, because every query filters on it (section 9).

---

## 5. Queries: `query`, `query_as`, and the fetch methods

### 5.1 Runtime-checked queries

```rust
use sqlx::FromRow;

#[derive(Debug, FromRow)]
pub struct ProjectRow {
    pub id: i32,
    pub tenant_id: i32,
    pub name: String,
    pub description: Option<String>,
    pub version: i32,
}

let row: Option<ProjectRow> = sqlx::query_as::<_, ProjectRow>(
    "SELECT id, tenant_id, name, description, version
     FROM projects
     WHERE id = $1 AND tenant_id = $2",
)
.bind(project_id)       // $1
.bind(tenant_id)        // $2
.fetch_optional(&pool)
.await?;
```

- **`$1`, `$2`** are PostgreSQL placeholders. Values are sent **separately** from the SQL text, so they can't cause SQL injection.
- `.bind(...)` calls must be in the same order as the placeholders.
- `#[derive(FromRow)]` maps columns to fields **by name**.

**Never build SQL with `format!`** from user input:

```rust
// SQL INJECTION: never do this
let sql = format!("SELECT * FROM projects WHERE name = '{name}'");

// Safe
sqlx::query("SELECT id FROM projects WHERE name = $1").bind(name)
```

### 5.2 The fetch methods

| Method | Returns | Use when |
|---|---|---|
| `fetch_one` | `T` (error `RowNotFound` if none) | Exactly one row must exist |
| `fetch_optional` | `Option<T>` | Zero or one row (lookups by ID) |
| `fetch_all` | `Vec<T>` | A **bounded** number of rows (always with `LIMIT`) |
| `fetch` | `Stream<Item = Result<T>>` | Many rows, processed one at a time (section 13) |
| `execute` | `PgQueryResult` (`.rows_affected()`) | `INSERT` / `UPDATE` / `DELETE` without `RETURNING` |

Prefer `fetch_optional` + `ok_or(...)` over `fetch_one` for lookups. Then "not found" is a value you handle, not an error you have to recognize later.

### 5.3 `RETURNING`

PostgreSQL can return the written row, so one round trip both inserts and reads:

```rust
let row = sqlx::query_as::<_, ProjectRow>(
    "INSERT INTO projects (tenant_id, name, description)
     VALUES ($1, $2, $3)
     RETURNING id, tenant_id, name, description, version",
)
.bind(tenant_id)
.bind(&input.name)
.bind(&input.description)
.fetch_one(&pool)
.await?;
```

---

## 6. Compile-time checked queries: `query!` and `query_as!`

The macro versions connect to the database **while compiling** and check the query against the real schema:

```rust
let row = sqlx::query_as!(
    ProjectRow,
    "SELECT id, tenant_id, name, description, version
     FROM projects
     WHERE id = $1 AND tenant_id = $2",
    project_id,
    tenant_id,
)
.fetch_optional(&pool)
.await?;
```

At compile time, SQLx checks:

- the SQL is valid (typo in `SELEC` → compile error)
- every column and table exists (`nmae` → compile error)
- parameter types match (`$1` is `INTEGER`, you passed a `String` → compile error)
- the result types match the struct fields (a nullable column into a non-`Option` field → compile error)

### 6.1 Offline mode

Compiling with the macros requires a database. That's a problem in CI and for other developers. The solution is **offline mode**:

```bash
cargo sqlx prepare --workspace     # writes query metadata to .sqlx/
```

- Commit the **`.sqlx/`** directory.
- With `SQLX_OFFLINE=true`, the macros use those files instead of a live database.
- In CI, run `cargo sqlx prepare --check --workspace` to fail the build when a query changed but `.sqlx/` wasn't regenerated.

### 6.2 Nullability overrides

SQLx infers nullability from the schema, but it can't always know (for example with `LEFT JOIN` or expressions). Override it in the column alias:

```rust
sqlx::query!(
    r#"SELECT count(*) AS "count!" FROM projects WHERE tenant_id = $1"#,  // "!" = never NULL
    tenant_id,
)
```

| Alias suffix | Meaning |
|---|---|
| `"col!"` | Force non-null |
| `"col?"` | Force nullable (`Option<T>`) |
| `"col: Type"` | Use this Rust type (for example a newtype or enum) |

### 6.3 Which one to use?

| | `query_as` (function) | `query_as!` (macro) |
|---|---|---|
| Checked | At runtime | At compile time |
| Needs a DB to compile | No | Yes, or `.sqlx/` offline data |
| Dynamic SQL (optional filters) | Yes | No: the SQL must be a string literal |
| Default in production code | For dynamic queries | **Yes** |

---

## 7. Mapping types: rows, newtypes, enums, NULL

### 7.1 Type mapping

| PostgreSQL | Rust |
|---|---|
| `INTEGER` / `SERIAL` | `i32` |
| `BIGINT` / `BIGSERIAL` | `i64` |
| `TEXT` / `VARCHAR` | `String` |
| `BOOLEAN` | `bool` |
| `TIMESTAMPTZ` | `chrono::DateTime<Utc>` (feature `chrono`) |
| `UUID` | `uuid::Uuid` (feature `uuid`) |
| `JSONB` | `serde_json::Value` or `sqlx::types::Json<T>` |
| Any nullable column | `Option<T>` |

Note: PostgreSQL has **no unsigned integers**, so `u32` doesn't map to anything. Use `i32` / `i64` in rows.

### 7.2 Newtype IDs

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, sqlx::Type, serde::Serialize, serde::Deserialize)]
#[sqlx(transparent)]      // store as the inner i32
#[serde(transparent)]     // serialize as the inner i32
pub struct ProjectId(pub i32);

#[derive(Debug, Clone, Copy, PartialEq, Eq, sqlx::Type, serde::Serialize, serde::Deserialize)]
#[sqlx(transparent)]
#[serde(transparent)]
pub struct TenantId(pub i32);
```

Now `ProjectId` flows from `Path<ProjectId>` (Stage 3) all the way to `.bind(id)`, and swapping a `TenantId` and a `ProjectId` is a compile error everywhere.

### 7.3 Enums

```rust
#[derive(Debug, Clone, Copy, sqlx::Type, serde::Serialize, serde::Deserialize)]
#[sqlx(type_name = "project_status", rename_all = "snake_case")]
#[serde(rename_all = "snake_case")]
pub enum ProjectStatus {
    Draft,
    InProgress,
    Completed,
    Live,
}
```

`type_name` must match the `CREATE TYPE` name from the migration. With the macros, use an alias override: `status AS "status: ProjectStatus"`.

---

## 8. The repository pattern

The `db` crate contains **one module per table or entity**, each with plain **free functions** that take the pool (or a transaction) as their first argument.

```rust
// db/src/projects.rs
use sqlx::PgPool;

#[derive(Debug, sqlx::FromRow)]
pub struct ProjectRow { /* ... */ }

pub async fn get_by_id(
    pool: &PgPool,
    tenant_id: TenantId,
    id: ProjectId,
) -> Result<Option<ProjectRow>, sqlx::Error> { /* ... */ }

pub async fn list(
    pool: &PgPool,
    tenant_id: TenantId,
    limit: i64,
    offset: i64,
) -> Result<Vec<ProjectRow>, sqlx::Error> { /* ... */ }

pub async fn insert(
    pool: &PgPool,
    tenant_id: TenantId,
    name: &str,
    description: Option<&str>,
) -> Result<ProjectRow, sqlx::Error> { /* ... */ }

pub async fn update(
    pool: &PgPool,
    tenant_id: TenantId,
    id: ProjectId,
    name: &str,
    expected_version: i32,
) -> Result<Option<ProjectRow>, sqlx::Error> { /* ... */ }
```

Why free functions instead of a `ProjectRepository` struct with methods:

- No struct to construct or pass around: the pool is the only dependency.
- Easy to call with a pool **or** a transaction (section 10).
- Easy to read: `db::projects::get_by_id(&pool, tenant, id)` says exactly what happens.

### 8.1 `*Row` vs DTO

```text
db:      ProjectRow   ── mirrors the table, derives FromRow
             │
             │  From<ProjectRow> for ProjectDto   (in domain or api)
             ▼
shared:  ProjectDto   ── what the API returns, derives Serialize
```

`*Row` structs **never leave the `db` layer** (README conventions). Why keep two types:

- The table can change (new column, renamed column) without changing the API.
- Some columns must never be sent to clients (`password_hash`, internal flags).
- The API shape can differ from the table (camelCase names, computed fields, nested objects).

```rust
impl From<ProjectRow> for ProjectDto {
    fn from(row: ProjectRow) -> Self {
        ProjectDto { id: row.id, name: row.name, version: row.version /* ... */ }
    }
}
```

---

## 9. Tenant scoping

In a multi-tenant service, many customers share the same tables. A bug that forgets one `WHERE tenant_id = ...` can show one customer's data to another. That is the most serious kind of bug a multi-tenant service can have.

The rules:

1. **Every** query on a tenant-owned table filters by `tenant_id`, including `UPDATE` and `DELETE`.
2. `tenant_id` comes from the **authenticated user** (Stage 8), never from the request body or query string.
3. Repository functions **require** a `TenantId` parameter, so it's impossible to call them without one.
4. A row from another tenant looks exactly like a missing row: **404** (Stage 4).

```rust
// WRONG: any tenant can read any project by guessing IDs
"SELECT ... FROM projects WHERE id = $1"

// RIGHT
"SELECT ... FROM projects WHERE id = $1 AND tenant_id = $2"

// WRONG: tenant check on read, then an unscoped write
"UPDATE projects SET name = $1 WHERE id = $2"

// RIGHT
"UPDATE projects SET name = $1 WHERE id = $2 AND tenant_id = $3"
```

Making `TenantId` a newtype (section 7.2) means it can't be mixed up with a `ProjectId` by accident. Admin features that really must cross tenants (a super-admin view) should use **separate, clearly named** functions such as `list_all_tenants_admin`, and the authorization check for them happens in the domain layer.

> PostgreSQL also offers **row-level security** (RLS) policies as a second line of defence. It's worth knowing about, but this repository uses explicit `tenant_id` filters, which are visible in every query.

---

## 10. Transactions

A **transaction** groups several statements so they all succeed or all fail.

```rust
let mut tx = pool.begin().await?;

let project = sqlx::query_as::<_, ProjectRow>(
    "INSERT INTO projects (tenant_id, name) VALUES ($1, $2)
     RETURNING id, tenant_id, name, description, version",
)
.bind(tenant_id)
.bind(name)
.fetch_one(&mut *tx)        // run on the transaction, not the pool
.await?;

sqlx::query("INSERT INTO audit_log (tenant_id, action, project_id) VALUES ($1, 'create', $2)")
    .bind(tenant_id)
    .bind(project.id)
    .execute(&mut *tx)
    .await?;

tx.commit().await?;
```

- `pool.begin()` borrows a connection and starts the transaction.
- `&mut *tx` passes the transaction's connection to a query.
- `tx.commit()` makes the changes permanent.
- **If `tx` is dropped without `commit`, it rolls back.** So any `?` that returns early undoes everything automatically. The same is true if the request is cancelled (Stage 1, section 6.3): the future is dropped, the transaction is dropped, and the changes are rolled back.

### 10.1 Repository functions that accept a pool or a transaction

Use the `PgExecutor` trait so the same function works in both places:

```rust
use sqlx::PgExecutor;

pub async fn insert_audit<'e>(
    exec: impl PgExecutor<'e>,
    tenant_id: TenantId,
    action: &str,
) -> Result<(), sqlx::Error> {
    sqlx::query("INSERT INTO audit_log (tenant_id, action) VALUES ($1, $2)")
        .bind(tenant_id)
        .bind(action)
        .execute(exec)
        .await?;
    Ok(())
}

// Both work:
insert_audit(&pool, tenant, "login").await?;
insert_audit(&mut *tx, tenant, "create").await?;
```

An executor is used up by one query. For a function that runs **several** queries in the same transaction, take `&mut PgConnection` (or `&mut Transaction<'_, Postgres>`) instead.

### 10.2 Keep transactions short

A transaction holds a pool connection and often row locks. **Never** call a slow external service (HTTP, email) while a transaction is open. Gather the data first, open the transaction, write, commit, then call out.

---

## 11. Optimistic locking with `version`

Two users open the same project. Both edit it. The second save silently overwrites the first one's changes: a **lost update**.

**Optimistic locking** prevents that without holding locks while users think:

1. Every row has a `version` column.
2. Clients read the row **with** its version.
3. An update sends back the version it was based on.
4. The `UPDATE` only matches if the version is unchanged, and increments it.
5. Zero rows updated means someone else changed it first → **409 Conflict** (Stage 4).

```rust
pub async fn update_name(
    pool: &PgPool,
    tenant_id: TenantId,
    id: ProjectId,
    name: &str,
    expected_version: i32,
) -> Result<Option<ProjectRow>, sqlx::Error> {
    sqlx::query_as::<_, ProjectRow>(
        "UPDATE projects
         SET name = $1, version = version + 1, updated_at = now()
         WHERE id = $2 AND tenant_id = $3 AND version = $4
         RETURNING id, tenant_id, name, description, version",
    )
    .bind(name)
    .bind(id)
    .bind(tenant_id)
    .bind(expected_version)
    .fetch_optional(pool)
    .await
}
```

`None` can mean two things: the project doesn't exist (404), or the version is stale (409). The domain layer decides which:

```rust
pub async fn rename(
    pool: &PgPool,
    tenant_id: TenantId,
    id: ProjectId,
    name: &str,
    expected_version: i32,
) -> Result<ProjectRow, DomainError> {
    if let Some(row) = db::projects::update_name(pool, tenant_id, id, name, expected_version).await? {
        return Ok(row);
    }
    match db::projects::get_by_id(pool, tenant_id, id).await? {
        None => Err(DomainError::ProjectNotFound(id)),
        Some(current) => Err(DomainError::VersionMismatch {
            expected: expected_version,
            actual: current.version,
        }),
    }
}
```

The check and the write are **one atomic statement** (`WHERE ... AND version = $4`), so there's no race between "check" and "update".

> **JS comparison:** Sequelize (`version: true`) and Mongoose (`__v`) offer the same pattern. Here it's explicit SQL: one extra `WHERE` condition.

| | Optimistic locking | Pessimistic locking (`SELECT ... FOR UPDATE`) |
|---|---|---|
| Holds locks | No | Yes, until the transaction ends |
| Conflict | Detected on save → 409, the user reloads | The second writer waits |
| Fits | Web UIs where people edit for minutes | Short server-side read-modify-write in one transaction |

---

## 12. Errors from the database

Stage 4 previewed the mapping. With repository functions returning `sqlx::Error`, the conversions are:

```rust
impl From<sqlx::Error> for DomainError {
    fn from(err: sqlx::Error) -> Self {
        match &err {
            sqlx::Error::Database(db) if db.is_unique_violation() => {
                DomainError::Duplicate(db.constraint().unwrap_or("unique").to_string())
            }
            sqlx::Error::Database(db) if db.is_foreign_key_violation() => {
                DomainError::InvalidReference
            }
            _ => DomainError::Storage(Box::new(err)),
        }
    }
}
```

Then `DomainError → AppError` (Stage 4, section 7.2) turns `Duplicate` into **409**, `InvalidReference` into **400**, and `Storage` into a logged **500**.

| Database error | PostgreSQL code | Status |
|---|---|---|
| Unique violation | `23505` | 409 Conflict |
| Foreign key violation | `23503` | 400 Bad Request |
| Check constraint violation | `23514` | 400 Bad Request |
| Connection or pool timeout | (not a DB error) | 500, or 503 if you want clients to retry |
| Anything else | | 500, logged in full |

Using `db.constraint()` lets you return a precise message such as "a project with this name already exists" for the `projects_name_unique_per_tenant` constraint, instead of a generic conflict.

---

## 13. Pagination and large results

### 13.1 Always `LIMIT` list queries

`fetch_all` loads every row into a `Vec`. A list endpoint without a limit works with 50 rows in development and runs out of memory with 5 million in production.

```rust
pub async fn list(
    pool: &PgPool,
    tenant_id: TenantId,
    limit: i64,
    offset: i64,
) -> Result<Vec<ProjectRow>, sqlx::Error> {
    sqlx::query_as::<_, ProjectRow>(
        "SELECT id, tenant_id, name, description, version
         FROM projects
         WHERE tenant_id = $1
         ORDER BY id
         LIMIT $2 OFFSET $3",
    )
    .bind(tenant_id)
    .bind(limit)
    .bind(offset)
    .fetch_all(pool)
    .await
}
```

- Cap the page size from config (`max_page_size`, Stage 5) in the API layer.
- Always `ORDER BY` a unique column; without it, the order between pages isn't defined.
- **Keyset pagination** (`WHERE id > $last_id ORDER BY id LIMIT $n`) stays fast for deep pages, where `OFFSET 100000` gets slow.

### 13.2 Streaming with `fetch`

For exports and other "all rows" work, `fetch` returns a stream that yields rows **as they arrive**:

```rust
use futures::TryStreamExt;   // futures = "0.3"

let mut rows = sqlx::query_as::<_, ProjectRow>(
    "SELECT id, tenant_id, name, description, version FROM projects WHERE tenant_id = $1",
)
.bind(tenant_id)
.fetch(&pool);

while let Some(row) = rows.try_next().await? {
    // handle one row at a time: memory stays flat
}
```

Stage 10 turns this into a streaming HTTP response (NDJSON) with backpressure. Collecting a large stream into a `Vec` defeats its purpose.

### 13.3 Avoid N+1 queries

```rust
// N+1: one query for the projects, then one per project
for p in &projects {
    let tasks = db::tasks::list_for_project(pool, tenant, p.id).await?;
}

// One query: fetch all the tasks for these projects at once
"SELECT ... FROM tasks WHERE tenant_id = $1 AND project_id = ANY($2)"
// .bind(tenant).bind(&project_ids)   // Vec<i32> binds to an INTEGER[]
```

---

## 14. How this looks in a production service

```text
backend/
├── migrations/                 # NNNN_name.up.sql / .down.sql, run on startup
├── .sqlx/                      # offline query metadata (committed)
└── crates/
    ├── shared/src/types/       # ProjectDto, CreateProject, UpdateProject, ProjectId, TenantId
    ├── db/src/
    │   ├── lib.rs              # pub mod projects; pub mod users; ...
    │   └── projects.rs         # ProjectRow + get_by_id / list / insert / update / delete
    ├── domain/src/projects.rs  # rules: version conflict vs not found, status checks
    └── api/src/routes/projects.rs
```

The request path:

```text
handler (api)
  │  AuthUser gives tenant_id; Path<ProjectId>; ValidJson<UpdateProject>
  ▼
domain::projects::update(&state.pool, tenant_id, id, input)
  │  business rules, decides 404 vs 409
  ▼
db::projects::update(pool, tenant_id, id, name, version)
  │  one SQL statement, scoped by tenant_id and version
  ▼
ProjectRow ──From──▶ ProjectDto ──Json──▶ client
```

Conventions:

- `query_as!` with offline data by default; runtime `query_as` only for dynamic SQL.
- Every repo function takes `TenantId` for tenant-owned tables.
- Every updatable table has `version`, and updates check it.
- Every list query has `LIMIT` and `ORDER BY`.
- Migrations run on startup and are never edited after merge.

---

## 15. Putting it all together

A complete local project: Postgres in Docker, a migration, a `db` module with tenant-scoped repository functions, a domain function that tells 404 from 409, and Axum handlers using the `AppError` pattern. It uses the runtime `query_as` function so it compiles without a database; the macros work the same way once `DATABASE_URL` is set.

```bash
cargo new projects-db && cd projects-db
docker compose up -d postgres          # the compose file from section 2.1
echo 'DATABASE_URL=postgres://app:app@localhost:5432/app' > .env
mkdir migrations
```

```toml name=Cargo.toml
[package]
name = "projects-db"
version = "0.1.0"
edition = "2021"

[dependencies]
anyhow = "1"
axum = "0.8"
dotenvy = "0.15"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.8", features = ["runtime-tokio", "tls-rustls", "postgres", "macros", "migrate"] }
thiserror = "2"
tokio = { version = "1", features = ["full"] }
```

```sql name=migrations/0001_create_projects.sql
CREATE TABLE projects (
    id        SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL,
    name      TEXT NOT NULL,
    version   INTEGER NOT NULL DEFAULT 1,
    CONSTRAINT projects_name_unique_per_tenant UNIQUE (tenant_id, name)
);

CREATE INDEX projects_tenant_id_idx ON projects (tenant_id);
```

```rust name=src/main.rs
use axum::{
    extract::{Path, Query, State},
    http::{HeaderMap, StatusCode},
    response::{IntoResponse, Response},
    routing::get,
    Json, Router,
};
use serde::{Deserialize, Serialize};
use sqlx::{postgres::PgPoolOptions, PgPool};

// ================= shared: ids and DTOs =================

#[derive(Debug, Clone, Copy, PartialEq, Eq, sqlx::Type, Serialize, Deserialize)]
#[sqlx(transparent)]
#[serde(transparent)]
struct ProjectId(i32);

#[derive(Debug, Clone, Copy, PartialEq, Eq, sqlx::Type)]
#[sqlx(transparent)]
struct TenantId(i32);

#[derive(Serialize)]
struct ProjectDto {
    id: ProjectId,
    name: String,
    version: i32,
}

#[derive(Deserialize)]
struct CreateProject {
    name: String,
}

#[derive(Deserialize)]
struct RenameProject {
    name: String,
    version: i32,
}

#[derive(Deserialize)]
struct ListParams {
    #[serde(default)]
    page: i64,
}

// ================= db =================

mod db {
    use super::{ProjectId, TenantId};
    use sqlx::PgPool;

    #[derive(Debug, sqlx::FromRow)]
    pub struct ProjectRow {
        pub id: ProjectId,
        pub tenant_id: TenantId,
        pub name: String,
        pub version: i32,
    }

    const COLUMNS: &str = "id, tenant_id, name, version";

    pub async fn get_by_id(
        pool: &PgPool,
        tenant_id: TenantId,
        id: ProjectId,
    ) -> Result<Option<ProjectRow>, sqlx::Error> {
        sqlx::query_as::<_, ProjectRow>(&format!(
            "SELECT {COLUMNS} FROM projects WHERE id = $1 AND tenant_id = $2"
        ))
        .bind(id)
        .bind(tenant_id)
        .fetch_optional(pool)
        .await
    }

    pub async fn list(
        pool: &PgPool,
        tenant_id: TenantId,
        limit: i64,
        offset: i64,
    ) -> Result<Vec<ProjectRow>, sqlx::Error> {
        sqlx::query_as::<_, ProjectRow>(&format!(
            "SELECT {COLUMNS} FROM projects WHERE tenant_id = $1
             ORDER BY id LIMIT $2 OFFSET $3"
        ))
        .bind(tenant_id)
        .bind(limit)
        .bind(offset)
        .fetch_all(pool)
        .await
    }

    pub async fn insert(
        pool: &PgPool,
        tenant_id: TenantId,
        name: &str,
    ) -> Result<ProjectRow, sqlx::Error> {
        sqlx::query_as::<_, ProjectRow>(&format!(
            "INSERT INTO projects (tenant_id, name) VALUES ($1, $2) RETURNING {COLUMNS}"
        ))
        .bind(tenant_id)
        .bind(name)
        .fetch_one(pool)
        .await
    }

    pub async fn update_name(
        pool: &PgPool,
        tenant_id: TenantId,
        id: ProjectId,
        name: &str,
        expected_version: i32,
    ) -> Result<Option<ProjectRow>, sqlx::Error> {
        sqlx::query_as::<_, ProjectRow>(&format!(
            "UPDATE projects SET name = $1, version = version + 1
             WHERE id = $2 AND tenant_id = $3 AND version = $4
             RETURNING {COLUMNS}"
        ))
        .bind(name)
        .bind(id)
        .bind(tenant_id)
        .bind(expected_version)
        .fetch_optional(pool)
        .await
    }
}

// Note: `format!` here only inserts a constant column list, never user input.
// User values always go through `.bind(...)`.

// ================= domain =================

#[derive(Debug, thiserror::Error)]
enum DomainError {
    #[error("project {0} not found")]
    NotFound(i32),
    #[error("version conflict: expected {expected}, found {actual}")]
    VersionMismatch { expected: i32, actual: i32 },
    #[error("a project with this name already exists")]
    DuplicateName,
    #[error("name must not be empty")]
    EmptyName,
    #[error("storage failure")]
    Storage(#[source] sqlx::Error),
}

impl From<sqlx::Error> for DomainError {
    fn from(err: sqlx::Error) -> Self {
        match &err {
            sqlx::Error::Database(db) if db.is_unique_violation() => DomainError::DuplicateName,
            _ => DomainError::Storage(err),
        }
    }
}

async fn rename_project(
    pool: &PgPool,
    tenant_id: TenantId,
    id: ProjectId,
    name: &str,
    expected_version: i32,
) -> Result<db::ProjectRow, DomainError> {
    let name = name.trim();
    if name.is_empty() {
        return Err(DomainError::EmptyName);
    }
    if let Some(row) = db::update_name(pool, tenant_id, id, name, expected_version).await? {
        return Ok(row);
    }
    match db::get_by_id(pool, tenant_id, id).await? {
        None => Err(DomainError::NotFound(id.0)),
        Some(current) => Err(DomainError::VersionMismatch {
            expected: expected_version,
            actual: current.version,
        }),
    }
}

// ================= api: errors =================

#[derive(Debug, thiserror::Error)]
enum AppError {
    #[error("{0}")]
    NotFound(String),
    #[error("{0}")]
    BadRequest(String),
    #[error("authentication required")]
    Unauthorized,
    #[error("{0}")]
    Conflict(String),
    #[error("internal server error")]
    Internal(#[source] Box<dyn std::error::Error + Send + Sync>),
}

impl From<DomainError> for AppError {
    fn from(err: DomainError) -> Self {
        match err {
            e @ DomainError::NotFound(_) => AppError::NotFound(e.to_string()),
            e @ (DomainError::VersionMismatch { .. } | DomainError::DuplicateName) => {
                AppError::Conflict(e.to_string())
            }
            e @ DomainError::EmptyName => AppError::BadRequest(e.to_string()),
            e @ DomainError::Storage(_) => AppError::Internal(Box::new(e)),
        }
    }
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
            AppError::Internal(source) => {
                eprintln!("[log] internal error: {source:?}");
                (StatusCode::INTERNAL_SERVER_ERROR, "internal")
            }
        };
        (status, Json(ErrorBody { error: code, message: self.to_string() })).into_response()
    }
}

// ================= api: handlers =================

const PAGE_SIZE: i64 = 20;

/// Placeholder until Stage 8: the tenant comes from an `x-tenant-id` header.
/// In a real service it comes from the verified JWT, never from client input.
fn tenant_from(headers: &HeaderMap) -> Result<TenantId, AppError> {
    headers
        .get("x-tenant-id")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.parse::<i32>().ok())
        .map(TenantId)
        .ok_or(AppError::Unauthorized)
}

impl From<db::ProjectRow> for ProjectDto {
    fn from(row: db::ProjectRow) -> Self {
        ProjectDto { id: row.id, name: row.name, version: row.version }
    }
}

async fn list(
    State(pool): State<PgPool>,
    headers: HeaderMap,
    Query(params): Query<ListParams>,
) -> Result<Json<Vec<ProjectDto>>, AppError> {
    let tenant = tenant_from(&headers)?;
    let offset = params.page.max(0) * PAGE_SIZE;
    let rows = db::list(&pool, tenant, PAGE_SIZE, offset)
        .await
        .map_err(DomainError::from)?;
    Ok(Json(rows.into_iter().map(ProjectDto::from).collect()))
}

async fn create(
    State(pool): State<PgPool>,
    headers: HeaderMap,
    Json(body): Json<CreateProject>,
) -> Result<(StatusCode, Json<ProjectDto>), AppError> {
    let tenant = tenant_from(&headers)?;
    let name = body.name.trim();
    if name.is_empty() {
        return Err(DomainError::EmptyName.into());
    }
    let row = db::insert(&pool, tenant, name).await.map_err(DomainError::from)?;
    Ok((StatusCode::CREATED, Json(row.into())))
}

async fn get_one(
    State(pool): State<PgPool>,
    headers: HeaderMap,
    Path(id): Path<ProjectId>,
) -> Result<Json<ProjectDto>, AppError> {
    let tenant = tenant_from(&headers)?;
    let row = db::get_by_id(&pool, tenant, id)
        .await
        .map_err(DomainError::from)?
        .ok_or(DomainError::NotFound(id.0))?;
    Ok(Json(row.into()))
}

async fn rename(
    State(pool): State<PgPool>,
    headers: HeaderMap,
    Path(id): Path<ProjectId>,
    Json(body): Json<RenameProject>,
) -> Result<Json<ProjectDto>, AppError> {
    let tenant = tenant_from(&headers)?;
    let row = rename_project(&pool, tenant, id, &body.name, body.version).await?;
    Ok(Json(row.into()))
}

// ================= main =================

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let _ = dotenvy::dotenv();
    let database_url = std::env::var("DATABASE_URL")?;

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await?;

    sqlx::migrate!("./migrations").run(&pool).await?;

    let app = Router::new()
        .route("/projects", get(list).post(create))
        .route("/projects/{id}", get(get_one).put(rename))
        .with_state(pool);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await?;
    println!("listening on http://127.0.0.1:3000");
    axum::serve(listener, app).await?;
    Ok(())
}
```

Try it:

```bash
cargo run

# tenant 1 creates a project
curl -s -X POST localhost:3000/projects -H 'x-tenant-id: 1' \
  -H 'content-type: application/json' -d '{"name":"Alpha"}'
# 201 {"id":1,"name":"Alpha","version":1}

# same name again in tenant 1 -> unique constraint
curl -s -X POST localhost:3000/projects -H 'x-tenant-id: 1' \
  -H 'content-type: application/json' -d '{"name":"Alpha"}'
# 409 {"error":"conflict","message":"a project with this name already exists"}

# tenant 2 can't see tenant 1's project
curl -s localhost:3000/projects/1 -H 'x-tenant-id: 2'
# 404 {"error":"not_found","message":"project 1 not found"}

# rename with the current version
curl -s -X PUT localhost:3000/projects/1 -H 'x-tenant-id: 1' \
  -H 'content-type: application/json' -d '{"name":"Alpha v2","version":1}'
# 200 {"id":1,"name":"Alpha v2","version":2}

# rename again with the stale version
curl -s -X PUT localhost:3000/projects/1 -H 'x-tenant-id: 1' \
  -H 'content-type: application/json' -d '{"name":"Alpha v3","version":1}'
# 409 {"error":"conflict","message":"version conflict: expected 1, found 2"}
```

(Add `-w ' %{http_code}\n'` to each `curl` to print the status code.)

What each call shows:

| Call | Concept |
|---|---|
| Create | `INSERT ... RETURNING`, newtype IDs through SQLx and serde |
| Duplicate name | `UNIQUE (tenant_id, name)` → `is_unique_violation()` → 409 |
| Tenant 2 reads project 1 | Tenant scoping: another tenant's row is a **404** |
| Rename with version 1 | Optimistic locking: version checked and incremented in one statement |
| Rename with stale version | `None` from the update, then the domain tells 409 from 404 |

---

## 16. Exercises

These need a local project with Postgres (section 2).

- [ ] **1. Migration:** add a reversible migration for a `tasks` table (`id`, `tenant_id`, `project_id` referencing `projects`, `title`, `done BOOLEAN NOT NULL DEFAULT false`, `version`). Run it, then run `sqlx migrate revert` and run it again.
- [ ] **2. Fetch methods:** write `count_projects(pool, tenant) -> Result<i64, sqlx::Error>` using `query_scalar`. Then write `get_by_name` using `fetch_optional`, and explain why `fetch_one` is a worse choice for it.
- [ ] **3. Macros:** rewrite `get_by_id` with `query_as!`. Misspell a column name and read the compile error. Then run `cargo sqlx prepare` and build with `SQLX_OFFLINE=true` with the database stopped.
- [ ] **4. Enum mapping:** add a `project_status` Postgres enum and a matching `ProjectStatus` Rust enum with `sqlx::Type`. Return it in `ProjectDto` as `"in_progress"` style JSON.
- [ ] **5. Tenant scoping review:** write `delete(pool, tenant, id) -> Result<bool, sqlx::Error>` using `rows_affected()`. Prove with `curl` that tenant 2 can't delete tenant 1's project and gets a 404.
- [ ] **6. Transaction:** write `create_with_first_task` that inserts a project and a task in one transaction. Make the task insert fail on purpose (for example a too-long title with a `CHECK` constraint) and confirm the project wasn't created either.
- [ ] **7. Optimistic locking:** add `PUT /tasks/{id}` with version checking. Open two terminals, send two updates based on the same version, and confirm the second gets 409.
- [ ] **8. N+1:** write `GET /projects-with-tasks` twice: once with one query per project, once with a single `WHERE project_id = ANY($2)` query grouped in Rust. Explain why the second scales better.

> When you can solve exercises 1–8 without looking anything up, you're ready for **Stage 7: Middleware**.

---

## 17. Further reading

- [SQLx README](https://github.com/launchbadge/sqlx) and [docs.rs/sqlx](https://docs.rs/sqlx): features, the macros, offline mode
- [SQLx CLI](https://github.com/launchbadge/sqlx/tree/main/sqlx-cli): `migrate`, `prepare`, `database`
- [Axum example: `sqlx-postgres`](https://github.com/tokio-rs/axum/tree/main/examples/sqlx-postgres): a pool in state, a custom extractor for a connection
- [PostgreSQL docs: constraints](https://www.postgresql.org/docs/current/ddl-constraints.html) and [error codes](https://www.postgresql.org/docs/current/errcodes-appendix.html)
- [Use The Index, Luke: pagination](https://use-the-index-luke.com/no-offset): why keyset pagination beats `OFFSET`
- *Zero To Production in Rust*, the chapters on the database and transactions
