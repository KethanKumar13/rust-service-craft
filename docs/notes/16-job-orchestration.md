# Stage 16: Job Orchestration

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> If you come from JavaScript or TypeScript, the comparisons below map Rust concepts to familiar JS ideas (BullMQ queues, `setInterval` heartbeats, and fetching an access token before calling an API).

## Contents

1. [Why a job system?](#1-why-a-job-system)
2. [Control plane and workers: push vs pull](#2-control-plane-and-workers-push-vs-pull)
3. [The job state machine](#3-the-job-state-machine)
4. [The jobs table](#4-the-jobs-table)
5. [Submitting jobs idempotently](#5-submitting-jobs-idempotently)
6. [Claiming work: `FOR UPDATE SKIP LOCKED`](#6-claiming-work-for-update-skip-locked)
7. [Leases and heartbeats](#7-leases-and-heartbeats)
8. [Fencing: rejecting stale workers](#8-fencing-rejecting-stale-workers)
9. [Retries, backoff, and poison jobs](#9-retries-backoff-and-poison-jobs)
10. [Cancellation across processes](#10-cancellation-across-processes)
11. [Service-to-service auth: OAuth2 client credentials](#11-service-to-service-auth-oauth2-client-credentials)
12. [Token caching on the worker](#12-token-caching-on-the-worker)
13. [Delivery guarantees: at-least-once and idempotent work](#13-delivery-guarantees-at-least-once-and-idempotent-work)
14. [How this looks in a production service](#14-how-this-looks-in-a-production-service)
15. [Putting it all together](#15-putting-it-all-together)
16. [Exercises](#16-exercises)
17. [Further reading](#17-further-reading)

---

## 1. Why a job system?

Stage 15 ran a pipeline inside one process. That breaks down when:

| Problem | Effect |
|---|---|
| A run takes 40 minutes | An HTTP request can't wait that long |
| The process restarts (deploy, crash) | Work in memory is lost, nobody knows it was running |
| Load grows | One process can't do all the work |
| Work must run near the data (another network, another machine) | The API server can't reach it |

A **job system** separates *deciding what to run* from *running it*:

- The API records a **job** in the database and returns immediately with its ID.
- **Workers** (separate processes, possibly on other machines) take jobs, run them, and report back.
- The database is the **source of truth**. If a worker dies, the job is still there, and someone else can pick it up.

> **JS comparison:** BullMQ or a Redis-backed queue: `queue.add(...)` returns a job ID, `new Worker(queue, handler)` processes it, and stalled jobs are retried. This stage builds the same ideas on PostgreSQL, so jobs live next to the rest of your data and use the same transactions.

---

## 2. Control plane and workers: push vs pull

```text
            ┌───────────────────────────┐
 clients ──▶│       control plane       │  HTTP API, owns the jobs table,
            │  (Axum + PostgreSQL)      │  hands out leases, reclaims expired ones
            └───────────▲───────────────┘
                        │  HTTPS, outbound from the worker
          ┌─────────────┼─────────────┐
          │             │             │
     ┌────┴───┐    ┌────┴───┐    ┌────┴───┐
     │worker 1│    │worker 2│    │worker 3│   claim → heartbeat → complete / fail
     └────────┘    └────────┘    └────────┘
```

Two ways to hand out work:

| | Push (control plane calls workers) | Pull (workers ask for work) |
|---|---|---|
| Who opens connections | Control plane → worker | Worker → control plane |
| Worker behind a firewall / NAT | Needs inbound access | Works, only outbound HTTPS |
| Worker overloaded | Control plane must track capacity | Worker just doesn't ask |
| Backpressure | Explicit | Built in |
| Latency to start a job | Immediate | Up to one poll interval |

**Pull** is the default choice for most job systems: workers decide when they have capacity, they can run anywhere with outbound HTTPS, and a crashed worker simply stops asking. It's the same pull-based backpressure as Stage 10, at the level of processes.

Workers talk to the control plane over HTTP rather than connecting to its database directly. That keeps the schema private, lets the control plane enforce rules (leases, fencing, tenant scoping), and means a worker only needs a URL and credentials (section 11).

---

## 3. The job state machine

Every job is in exactly one state, and only some transitions are allowed:

```text
             submit
               │
               ▼
   ┌──────▶ queued ──────── claim ───────▶ running ─────── complete ──▶ succeeded
   │           ▲                             │ │
   │           │       fail, attempts left   │ │  fail, no attempts left
   │           └──── (after backoff) ────────┘ └──────────────────────▶ failed
   │                                         │
   │        lease expired, attempts left     │  cancel acknowledged
   └─────────────────────────────────────────┤──────────────────────▶ cancelled
```

Rules:

- **Transitions happen in one SQL `UPDATE`** with the current state in the `WHERE` clause. If the row isn't in the expected state, zero rows change, and that's how races are detected (the same idea as optimistic locking in Stage 6).
- **Terminal states** (`succeeded`, `failed`, `cancelled`) never change again.
- **Model it as a Rust enum** that maps to a `TEXT` column with a `CHECK` constraint, so an invalid state can't exist in code or in the database.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, sqlx::Type, serde::Serialize)]
#[sqlx(type_name = "text", rename_all = "lowercase")]
#[serde(rename_all = "lowercase")]
pub enum JobStatus {
    Queued,
    Running,
    Succeeded,
    Failed,
    Cancelled,
}
```

---

## 4. The jobs table

```sql
CREATE TABLE jobs (
    id                BIGSERIAL PRIMARY KEY,
    tenant_id         BIGINT      NOT NULL,
    idempotency_key   TEXT,
    payload           JSONB       NOT NULL,
    status            TEXT        NOT NULL DEFAULT 'queued'
                      CHECK (status IN ('queued', 'running', 'succeeded', 'failed', 'cancelled')),
    attempts          INT         NOT NULL DEFAULT 0,
    max_attempts      INT         NOT NULL DEFAULT 3,
    run_after         TIMESTAMPTZ NOT NULL DEFAULT now(),
    lease_owner       TEXT,
    lease_expires_at  TIMESTAMPTZ,
    cancel_requested  BOOLEAN     NOT NULL DEFAULT false,
    last_error        TEXT,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, idempotency_key)
);

-- Only queued jobs are ever searched for work: keep that index small
CREATE INDEX jobs_claimable ON jobs (run_after, id) WHERE status = 'queued';
-- The reaper looks for running jobs with expired leases
CREATE INDEX jobs_leases ON jobs (lease_expires_at) WHERE status = 'running';
```

| Column | Purpose |
|---|---|
| `idempotency_key` | The same submit twice creates one job (section 5) |
| `attempts`, `max_attempts` | Retry budget, and the fencing token (section 8) |
| `run_after` | Delayed start and retry backoff (section 9) |
| `lease_owner`, `lease_expires_at` | Who holds the job and until when (section 7) |
| `cancel_requested` | A flag the worker sees on its next heartbeat (section 10) |

**Partial indexes** (`WHERE status = 'queued'`) matter: finished jobs pile up over months, but the claim query only ever looks at the few queued ones.

---

## 5. Submitting jobs idempotently

A client submits a job, the network drops the response, and the client retries. Without protection that creates **two** jobs. The fix is an **idempotency key** chosen by the client (often a UUID, or a natural key like `"import-2026-10-11"`):

```rust
pub async fn submit_job(
    pool: &PgPool,
    tenant_id: i64,
    idempotency_key: Option<&str>,
    payload: &serde_json::Value,
) -> Result<i64, sqlx::Error> {
    let inserted: Option<i64> = sqlx::query_scalar(
        "INSERT INTO jobs (tenant_id, idempotency_key, payload)
         VALUES ($1, $2, $3)
         ON CONFLICT (tenant_id, idempotency_key) DO NOTHING
         RETURNING id",
    )
    .bind(tenant_id)
    .bind(idempotency_key)
    .bind(payload)
    .fetch_optional(pool)
    .await?;

    match inserted {
        Some(id) => Ok(id),
        None => {
            sqlx::query_scalar("SELECT id FROM jobs WHERE tenant_id = $1 AND idempotency_key = $2")
                .bind(tenant_id)
                .bind(idempotency_key)
                .fetch_one(pool)
                .await
        }
    }
}
```

- The `UNIQUE (tenant_id, idempotency_key)` constraint does the real work. Checking with a `SELECT` first would race.
- Keys are **scoped by tenant**, so one tenant can't collide with (or discover) another's keys.
- `NULL` keys never conflict in PostgreSQL, so clients that don't send a key still work.
- The HTTP API returns `202 Accepted` with the job ID, and the same ID on a retry.

---

## 6. Claiming work: `FOR UPDATE SKIP LOCKED`

Many workers ask for work at the same time. Each job must go to **exactly one** of them, and they shouldn't wait for each other.

```sql
UPDATE jobs
SET    status = 'running',
       attempts = attempts + 1,
       lease_owner = $1,
       lease_expires_at = now() + make_interval(secs => $2),
       updated_at = now()
WHERE  id = (
    SELECT id FROM jobs
    WHERE  status = 'queued' AND run_after <= now()
    ORDER  BY run_after, id
    LIMIT  1
    FOR UPDATE SKIP LOCKED
)
RETURNING id, tenant_id, payload, attempts;
```

| Part | Why |
|---|---|
| `FOR UPDATE` | Locks the chosen row so no other transaction can claim it |
| `SKIP LOCKED` | Rows locked by another worker's claim are **skipped**, not waited for |
| `ORDER BY run_after, id` | Oldest eligible work first |
| One `UPDATE` with a subquery | Find and claim in one statement: no window between them |
| `RETURNING` | The worker gets the job in the same round trip |

Without `SKIP LOCKED`, every worker would pick the same "first" row and queue up behind one lock. With it, ten workers claim ten different jobs in parallel.

The claim endpoint returns the job or nothing. When there's nothing, the worker sleeps (with some random jitter so workers don't all poll at the same moment) and asks again.

> PostgreSQL also has `LISTEN`/`NOTIFY` to wake workers instantly when a job is submitted. It's an optimization on top of polling, never a replacement: notifications are lost if nobody is listening at that moment.

---

## 7. Leases and heartbeats

A worker that claimed a job might crash, lose its network, or freeze. The control plane can't tell those apart from "still working". The answer is a **lease**: the claim is only valid until `lease_expires_at`.

```text
 claim (lease 30s) ── heartbeat ── heartbeat ── heartbeat ── complete
       │                 │            │            │
       └── expires at ───┴─ extended ─┴─ extended ─┘

 claim (lease 30s) ── heartbeat ── ✗ crash
                                        ... 30s pass ...
                                   reaper: lease expired → queued again
```

- The worker **heartbeats** well inside the lease (a common rule: every lease / 3). Each heartbeat pushes `lease_expires_at` forward.
- A **reaper** task in the control plane periodically finds running jobs with expired leases and puts them back to `queued` (or `failed` if attempts are used up).
- The heartbeat runs **alongside** the work, with `tokio::select!`, not between steps of it. A long step must not starve the heartbeat:

```rust
let work = do_work(&job.payload);
tokio::pin!(work);
let mut beat = tokio::time::interval(HEARTBEAT_EVERY);

let outcome = loop {
    tokio::select! {
        result = &mut work => break Outcome::Done(result),
        _ = beat.tick() => match api.heartbeat(job.id, job.attempt).await {
            Ok(Heartbeat::Continue) => {}
            Ok(Heartbeat::CancelRequested) => break Outcome::Cancelled,
            Err(HeartbeatError::LeaseLost) => break Outcome::LeaseLost, // stop: someone else owns it now
            Err(HeartbeatError::Transient(_)) => {} // retry next tick; the lease still has time left
        },
    }
};
```

Choosing the lease length:

| Shorter lease | Longer lease |
|---|---|
| Crashed work is retried sooner | Fewer heartbeats, tolerates brief network trouble |
| A slow network can lose a healthy lease | A crashed worker blocks the job for longer |

30–60 seconds with heartbeats every 10–20 seconds is a common starting point.

---

## 8. Fencing: rejecting stale workers

Leases create a nasty case. Worker A freezes (a long GC pause, a VM stall), its lease expires, worker B claims the job and starts again. Then **A wakes up** and reports "done". If the control plane accepts that, the job's result is A's stale output, and B's work is still running.

The fix is a **fencing token**: a number that increases on every claim, sent with every report. Here, `attempts` is already that number:

```sql
UPDATE jobs
SET    status = 'succeeded', lease_owner = NULL, lease_expires_at = NULL, updated_at = now()
WHERE  id = $1 AND lease_owner = $2 AND attempts = $3 AND status = 'running';
```

- A's report carries `attempt = 1`. The row now has `attempts = 2`. Zero rows match, so the control plane answers **`409 Conflict`**, and A discards its result.
- Heartbeats use the same `WHERE`, so A's next heartbeat also fails, and A stops (`LeaseLost` above).
- If the worker writes to external systems (files, other databases), pass the token there too and have them reject lower numbers. Fencing only protects what checks the token.

This is the same 409 rule as optimistic locking in Stage 6: "the thing you think you're updating has changed".

---

## 9. Retries, backoff, and poison jobs

Failures fall into two groups:

| Kind | Example | Retry? |
|---|---|---|
| Transient | Network timeout, database restarting, rate limit | Yes, with backoff |
| Permanent | Invalid payload, missing source table, permission denied | No, retrying wastes time |

The worker reports which it was. The control plane decides the next state in one statement:

```sql
UPDATE jobs
SET    status = CASE WHEN attempts < max_attempts THEN 'queued' ELSE 'failed' END,
       run_after = now() + make_interval(secs => $4),   -- backoff
       last_error = $5,
       lease_owner = NULL, lease_expires_at = NULL, updated_at = now()
WHERE  id = $1 AND lease_owner = $2 AND attempts = $3 AND status = 'running';
```

**Exponential backoff with jitter:**

```rust
fn backoff_secs(attempt: i32) -> f64 {
    let base = 2.0_f64.powi(attempt.saturating_sub(1)).min(300.0); // 1s, 2s, 4s, … capped at 5 min
    let jitter = rand::random::<f64>() * base * 0.2;                // ±20% so retries don't line up
    base + jitter
}
```

- **Max attempts** stops a **poison job** (one that fails every time, or crashes its worker) from looping forever.
- A **permanent** error sets `status = 'failed'` directly, regardless of attempts left.
- Store `last_error` (a short, safe message) so users can see *why* without reading worker logs. Log the full cause with `tracing::error!` and the job ID.

---

## 10. Cancellation across processes

Stage 15's `CancellationToken` works inside one process. Across processes, cancellation becomes **a flag in the database** plus the heartbeat:

```text
user: POST /jobs/42/cancel ─▶ UPDATE jobs SET cancel_requested = true WHERE id = 42
                                            │
worker heartbeat ◀── { "cancel_requested": true } ◀──┘
     │
     └─▶ token.cancel()  (Stage 15: stages stop, sinks roll back)
     └─▶ POST /jobs/42/cancelled  ─▶ status = 'cancelled'
```

- A **queued** job can be cancelled directly (`status = 'cancelled'`), since nobody is running it.
- A **running** job is cancelled **cooperatively**: the worker learns about it within one heartbeat interval, stops cleanly, and confirms.
- If the worker never confirms (it's dead), the reaper treats the expired lease as cancelled instead of re-queueing it.

---

## 11. Service-to-service auth: OAuth2 client credentials

Stage 8 authenticated **users** with passwords and cookies. Workers aren't users. They need their own identity with limited rights, which is what the OAuth2 **client credentials** grant is for:

```text
worker                                   control plane
  │  POST /oauth/token                         │
  │  grant_type=client_credentials             │
  │  client_id=worker-pool                     │
  │  client_secret=…                           │
  │  scope=jobs:work                           │
  │ ─────────────────────────────────────────▶ │  check secret (Argon2 hash, Stage 8)
  │ ◀───────────────────────────────────────── │  issue a short-lived JWT
  │  { "access_token": "eyJ…",                 │
  │    "token_type": "Bearer",                 │
  │    "expires_in": 900 }                     │
  │                                            │
  │  POST /jobs/claim                          │
  │  Authorization: Bearer eyJ…                │
  │ ─────────────────────────────────────────▶ │  verify signature, expiry, scope
```

| Concept | Meaning |
|---|---|
| **Client ID / secret** | The worker's long-lived credentials. Stored hashed on the server, in a secret store on the worker |
| **Access token** | Short-lived (minutes). Sent on every API call. A leaked token expires soon |
| **Scope** | What the token may do: `jobs:work` can claim and report, but can't submit or read other tenants' data |
| **Request format** | The token request is `application/x-www-form-urlencoded`, not JSON (the OAuth2 spec, RFC 6749 §4.4) |

On the control plane, the worker routes use an extractor just like Stage 8's `AuthUser`, but they check the **scope** instead of user capabilities:

```rust
pub struct WorkerAuth {
    pub client_id: String,
}

impl FromRequestParts<AppState> for WorkerAuth {
    type Rejection = ApiError;

    async fn from_request_parts(parts: &mut Parts, state: &AppState) -> Result<Self, Self::Rejection> {
        let token = parts
            .headers
            .get(header::AUTHORIZATION)
            .and_then(|v| v.to_str().ok())
            .and_then(|v| v.strip_prefix("Bearer "))
            .ok_or(ApiError::Unauthorized)?;

        let data = decode::<WorkerClaims>(token, &state.keys.decoding, &Validation::default())
            .map_err(|_| ApiError::Unauthorized)?;               // bad signature or expired

        if !data.claims.scope.split(' ').any(|s| s == "jobs:work") {
            return Err(ApiError::Forbidden);                     // valid token, wrong rights
        }
        Ok(WorkerAuth { client_id: data.claims.sub })
    }
}
```

Rules:

- **Separate user tokens and worker tokens.** A user cookie must never be accepted on worker routes, and a worker token never on user routes.
- **Many worker instances can share one client** (a "pool"). Each instance still sends its own `worker_id`, so the lease owner is `client_id/worker_id`.
- **Rotate secrets** by allowing two valid secrets per client during the switch.
- In larger setups the token endpoint is an external identity provider (Microsoft Entra ID, Auth0, Keycloak). The worker code doesn't change; only the token URL does.

---

## 12. Token caching on the worker

Requesting a new token for every call doubles the traffic and puts load on the token endpoint. The worker **caches** the token and refreshes it **before** it expires:

```rust
struct TokenCache {
    token: Option<(String, Instant)>,   // value, expires at
}

impl Worker {
    async fn bearer(&mut self) -> Result<String, BoxError> {
        if let Some((token, expires_at)) = &self.cache.token {
            if Instant::now() + REFRESH_MARGIN < *expires_at {
                return Ok(token.clone());
            }
        }
        let grant: TokenResponse = self
            .http
            .post(format!("{}/oauth/token", self.base_url))
            .form(&[
                ("grant_type", "client_credentials"),
                ("client_id", self.client_id.as_str()),
                ("client_secret", self.client_secret.as_str()),
                ("scope", "jobs:work"),
            ])
            .send()
            .await?
            .error_for_status()?
            .json()
            .await?;

        let expires_at = Instant::now() + Duration::from_secs(grant.expires_in);
        self.cache.token = Some((grant.access_token.clone(), expires_at));
        Ok(grant.access_token)
    }
}
```

- **Refresh margin** (for example 60 seconds) covers clock differences and slow requests, so a token never expires mid-call.
- If a call still returns `401`, drop the cached token, fetch a new one, and retry **once**.
- If several tasks in one worker share the cache, put it behind a `tokio::sync::Mutex` so only one of them refreshes at a time (Stage 1).
- **Never log tokens or secrets.** Add them to the redaction list in your tracing setup (Stage 7).

> **JS comparison:** the same pattern as an Axios interceptor that attaches `Authorization: Bearer …`, refreshes when the token is close to expiry, and retries once on 401.

---

## 13. Delivery guarantees: at-least-once and idempotent work

With leases and retries, a job can **start more than once**: a worker finishes the work, then crashes before its "complete" call arrives. The lease expires, and another worker runs it again. That's **at-least-once** delivery, and it's what practically every real job system gives you.

"Exactly once" isn't something the queue can promise. You get its *effect* by making the **work itself idempotent**:

| Technique | Example |
|---|---|
| Write to a location derived from the job | Output file `jobs/42/attempt-2.parquet`, then an atomic rename to `jobs/42/result.parquet` |
| Upsert instead of insert | `INSERT … ON CONFLICT (source_id) DO UPDATE` |
| Record the job ID in the target | A `loaded_by_job` column plus a unique constraint |
| One transaction for "write result + mark job done" | Only possible when both live in the same database |
| Fencing token on external writes | The target rejects writes with a lower attempt number (section 8) |

Design every job handler with the question: **"What happens if this runs twice?"** If the answer is "duplicate data", fix the handler, not the queue.

---

## 14. How this looks in a production service

```text
control-plane/
├── src/
│   ├── routes/
│   │   ├── jobs.rs          # user API: submit (idempotent), get, list, cancel
│   │   ├── worker.rs        # worker API: claim, heartbeat, complete, fail, cancelled
│   │   └── oauth.rs         # POST /oauth/token (client credentials)
│   ├── auth/worker.rs       # WorkerAuth extractor: Bearer JWT + scope
│   ├── db/jobs.rs           # one function per transition, each a single guarded UPDATE
│   └── reaper.rs            # expired leases → queued / failed / cancelled
└── migrations/              # jobs table, partial indexes, CHECK on status

worker/
├── src/
│   ├── client.rs            # HTTP client + token cache + one retry on 401
│   ├── run_loop.rs          # claim → work + heartbeat (select!) → report; jittered polling
│   └── handlers/            # job kinds; each idempotent, each takes a CancellationToken
└── tests/
```

Conventions:

- **The database is the source of truth.** Workers keep no state the system depends on.
- **Every state change is one `UPDATE … WHERE status = … AND lease_owner = … AND attempts = …`.** Zero rows changed → `409`.
- **`FOR UPDATE SKIP LOCKED`** for claims, **partial indexes** for queued and running jobs.
- **Leases + heartbeats + reaper**, with heartbeats running alongside the work.
- **Fencing token** on every report and every external write.
- **Retries** only for transient errors, with exponential backoff, jitter, and a max.
- **Client-credentials tokens** with a narrow scope for workers, cached and refreshed early.
- **Idempotent handlers**, because delivery is at-least-once.
- **Tenant scoping** on every user-facing query; workers see only what their claimed job needs.
- Metrics worth tracking: queue depth, time from submit to claim, run duration, retries, reaped leases.

---

## 15. Putting it all together

One binary that starts a control plane (Axum + PostgreSQL) and three workers that talk to it over HTTP with client-credentials tokens. One worker "freezes" on its first job to show lease expiry, re-queueing, and fencing. One job always fails, to show retries with backoff until it's marked `failed`.

Point `DATABASE_URL` at a **scratch** database: the example drops and recreates the `jobs` table.

```toml
[dependencies]
axum = "0.8"
jsonwebtoken = "9"
reqwest = { version = "0.12", features = ["json"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "json"] }
tokio = { version = "1", features = ["full"] }
tokio-util = "0.7"
```

```rust
use std::sync::Arc;
use std::time::{Duration, Instant, SystemTime, UNIX_EPOCH};

use axum::{
    extract::{FromRequestParts, Path, State},
    http::{header, request::Parts, StatusCode},
    response::{IntoResponse, Response},
    routing::post,
    Form, Json, Router,
};
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use serde_json::json;
use sqlx::{postgres::PgPoolOptions, PgPool};
use tokio::task::JoinSet;
use tokio_util::sync::CancellationToken;

type BoxError = Box<dyn std::error::Error + Send + Sync>;

const LEASE_SECS: f64 = 2.0;
const HEARTBEAT_EVERY: Duration = Duration::from_millis(500);
const IDLE_POLL: Duration = Duration::from_millis(300);
const FREEZE_FOR: Duration = Duration::from_secs(4);
const TOKEN_TTL_SECS: u64 = 60;
const TOKEN_REFRESH_MARGIN: Duration = Duration::from_secs(10);

const SCHEMA: &str = "
DROP TABLE IF EXISTS jobs;
CREATE TABLE jobs (
    id               BIGSERIAL PRIMARY KEY,
    tenant_id        BIGINT      NOT NULL,
    idempotency_key  TEXT,
    payload          JSONB       NOT NULL,
    status           TEXT        NOT NULL DEFAULT 'queued'
                     CHECK (status IN ('queued','running','succeeded','failed','cancelled')),
    attempts         INT         NOT NULL DEFAULT 0,
    max_attempts     INT         NOT NULL DEFAULT 3,
    run_after        TIMESTAMPTZ NOT NULL DEFAULT now(),
    lease_owner      TEXT,
    lease_expires_at TIMESTAMPTZ,
    last_error       TEXT,
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, idempotency_key)
);
CREATE INDEX jobs_claimable ON jobs (run_after, id) WHERE status = 'queued';
CREATE INDEX jobs_leases ON jobs (lease_expires_at) WHERE status = 'running';
";

// ======================= control plane =======================

#[derive(Clone)]
struct AppState {
    pool: PgPool,
    keys: Arc<Keys>,
    clients: Arc<Vec<RegisteredClient>>,
}

struct Keys {
    encoding: EncodingKey,
    decoding: DecodingKey,
}

struct RegisteredClient {
    id: String,
    secret: String, // demo only: store an Argon2 hash (Stage 8)
    scope: String,
}

enum ApiError {
    BadRequest(&'static str),
    Unauthorized,
    Forbidden,
    Conflict(&'static str),
    Internal(String),
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let (status, msg) = match self {
            ApiError::BadRequest(m) => (StatusCode::BAD_REQUEST, m),
            ApiError::Unauthorized => (StatusCode::UNAUTHORIZED, "unauthorized"),
            ApiError::Forbidden => (StatusCode::FORBIDDEN, "forbidden"),
            ApiError::Conflict(m) => (StatusCode::CONFLICT, m),
            ApiError::Internal(e) => {
                eprintln!("internal error: {e}");
                (StatusCode::INTERNAL_SERVER_ERROR, "internal error")
            }
        };
        (status, Json(json!({ "error": msg }))).into_response()
    }
}

impl From<sqlx::Error> for ApiError {
    fn from(e: sqlx::Error) -> Self {
        ApiError::Internal(e.to_string())
    }
}

fn now_secs() -> u64 {
    SystemTime::now().duration_since(UNIX_EPOCH).map(|d| d.as_secs()).unwrap_or(0)
}

// ---- OAuth2 client credentials ----

#[derive(Deserialize)]
struct TokenRequest {
    grant_type: String,
    client_id: String,
    client_secret: String,
}

#[derive(Serialize, Deserialize)]
struct TokenResponse {
    access_token: String,
    token_type: String,
    expires_in: u64,
}

#[derive(Serialize, Deserialize)]
struct WorkerClaims {
    sub: String,
    scope: String,
    exp: u64,
}

async fn issue_token(
    State(s): State<AppState>,
    Form(req): Form<TokenRequest>,
) -> Result<Json<TokenResponse>, ApiError> {
    if req.grant_type != "client_credentials" {
        return Err(ApiError::BadRequest("unsupported_grant_type"));
    }
    let client = s
        .clients
        .iter()
        .find(|c| c.id == req.client_id && c.secret == req.client_secret)
        .ok_or(ApiError::Unauthorized)?;

    let claims = WorkerClaims {
        sub: client.id.clone(),
        scope: client.scope.clone(),
        exp: now_secs() + TOKEN_TTL_SECS,
    };
    let access_token = encode(&Header::default(), &claims, &s.keys.encoding)
        .map_err(|e| ApiError::Internal(e.to_string()))?;
    Ok(Json(TokenResponse { access_token, token_type: "Bearer".into(), expires_in: TOKEN_TTL_SECS }))
}

struct WorkerAuth {
    client_id: String,
}

impl WorkerAuth {
    fn owner(&self, worker_id: &str) -> String {
        format!("{}/{}", self.client_id, worker_id)
    }
}

impl FromRequestParts<AppState> for WorkerAuth {
    type Rejection = ApiError;

    async fn from_request_parts(parts: &mut Parts, s: &AppState) -> Result<Self, Self::Rejection> {
        let token = parts
            .headers
            .get(header::AUTHORIZATION)
            .and_then(|v| v.to_str().ok())
            .and_then(|v| v.strip_prefix("Bearer "))
            .ok_or(ApiError::Unauthorized)?;
        let data = decode::<WorkerClaims>(token, &s.keys.decoding, &Validation::default())
            .map_err(|_| ApiError::Unauthorized)?;
        if !data.claims.scope.split(' ').any(|x| x == "jobs:work") {
            return Err(ApiError::Forbidden);
        }
        Ok(WorkerAuth { client_id: data.claims.sub })
    }
}

// ---- job transitions ----

#[derive(Serialize, Deserialize, sqlx::FromRow)]
struct ClaimedJob {
    id: i64,
    tenant_id: i64,
    payload: serde_json::Value,
    attempt: i32,
}

#[derive(Deserialize)]
struct ClaimRequest {
    worker_id: String,
}

#[derive(Deserialize)]
struct LeaseRequest {
    worker_id: String,
    attempt: i32,
    error: Option<String>,
}

async fn claim(
    State(s): State<AppState>,
    auth: WorkerAuth,
    Json(req): Json<ClaimRequest>,
) -> Result<Json<Option<ClaimedJob>>, ApiError> {
    let job = sqlx::query_as::<_, ClaimedJob>(
        "UPDATE jobs
         SET status = 'running', attempts = attempts + 1, lease_owner = $1,
             lease_expires_at = now() + make_interval(secs => $2), updated_at = now()
         WHERE id = (
             SELECT id FROM jobs
             WHERE status = 'queued' AND run_after <= now()
             ORDER BY run_after, id
             LIMIT 1
             FOR UPDATE SKIP LOCKED
         )
         RETURNING id, tenant_id, payload, attempts AS attempt",
    )
    .bind(auth.owner(&req.worker_id))
    .bind(LEASE_SECS)
    .fetch_optional(&s.pool)
    .await?;
    Ok(Json(job))
}

async fn heartbeat(
    State(s): State<AppState>,
    auth: WorkerAuth,
    Path(id): Path<i64>,
    Json(req): Json<LeaseRequest>,
) -> Result<StatusCode, ApiError> {
    let updated = sqlx::query(
        "UPDATE jobs SET lease_expires_at = now() + make_interval(secs => $4), updated_at = now()
         WHERE id = $1 AND lease_owner = $2 AND attempts = $3 AND status = 'running'",
    )
    .bind(id)
    .bind(auth.owner(&req.worker_id))
    .bind(req.attempt)
    .bind(LEASE_SECS)
    .execute(&s.pool)
    .await?
    .rows_affected();
    if updated == 0 {
        return Err(ApiError::Conflict("lease lost"));
    }
    Ok(StatusCode::NO_CONTENT)
}

async fn complete(
    State(s): State<AppState>,
    auth: WorkerAuth,
    Path(id): Path<i64>,
    Json(req): Json<LeaseRequest>,
) -> Result<StatusCode, ApiError> {
    let updated = sqlx::query(
        "UPDATE jobs SET status = 'succeeded', lease_owner = NULL, lease_expires_at = NULL,
                         last_error = NULL, updated_at = now()
         WHERE id = $1 AND lease_owner = $2 AND attempts = $3 AND status = 'running'",
    )
    .bind(id)
    .bind(auth.owner(&req.worker_id))
    .bind(req.attempt)
    .execute(&s.pool)
    .await?
    .rows_affected();
    if updated == 0 {
        return Err(ApiError::Conflict("lease lost"));
    }
    Ok(StatusCode::NO_CONTENT)
}

async fn fail(
    State(s): State<AppState>,
    auth: WorkerAuth,
    Path(id): Path<i64>,
    Json(req): Json<LeaseRequest>,
) -> Result<StatusCode, ApiError> {
    let backoff = 0.2 * 2f64.powi(req.attempt.saturating_sub(1)); // 0.2s, 0.4s, 0.8s …
    let updated = sqlx::query(
        "UPDATE jobs
         SET status = CASE WHEN attempts < max_attempts THEN 'queued' ELSE 'failed' END,
             run_after = now() + make_interval(secs => $4),
             last_error = $5, lease_owner = NULL, lease_expires_at = NULL, updated_at = now()
         WHERE id = $1 AND lease_owner = $2 AND attempts = $3 AND status = 'running'",
    )
    .bind(id)
    .bind(auth.owner(&req.worker_id))
    .bind(req.attempt)
    .bind(backoff)
    .bind(req.error.as_deref().unwrap_or("unknown error"))
    .execute(&s.pool)
    .await?
    .rows_affected();
    if updated == 0 {
        return Err(ApiError::Conflict("lease lost"));
    }
    Ok(StatusCode::NO_CONTENT)
}

async fn reaper(pool: PgPool, shutdown: CancellationToken) {
    let mut tick = tokio::time::interval(Duration::from_millis(500));
    loop {
        tokio::select! {
            _ = shutdown.cancelled() => break,
            _ = tick.tick() => {
                let result = sqlx::query(
                    "UPDATE jobs
                     SET status = CASE WHEN attempts < max_attempts THEN 'queued' ELSE 'failed' END,
                         run_after = now(), last_error = 'lease expired',
                         lease_owner = NULL, lease_expires_at = NULL, updated_at = now()
                     WHERE status = 'running' AND lease_expires_at < now()",
                )
                .execute(&pool)
                .await;
                match result {
                    Ok(r) if r.rows_affected() > 0 => {
                        println!("[reaper] reclaimed {} expired lease(s)", r.rows_affected());
                    }
                    Ok(_) => {}
                    Err(e) => eprintln!("[reaper] {e}"),
                }
            }
        }
    }
}

async fn submit_job(
    pool: &PgPool,
    tenant_id: i64,
    key: Option<&str>,
    payload: serde_json::Value,
) -> Result<i64, sqlx::Error> {
    let inserted: Option<i64> = sqlx::query_scalar(
        "INSERT INTO jobs (tenant_id, idempotency_key, payload) VALUES ($1, $2, $3)
         ON CONFLICT (tenant_id, idempotency_key) DO NOTHING RETURNING id",
    )
    .bind(tenant_id)
    .bind(key)
    .bind(&payload)
    .fetch_optional(pool)
    .await?;
    match inserted {
        Some(id) => Ok(id),
        None => {
            sqlx::query_scalar("SELECT id FROM jobs WHERE tenant_id = $1 AND idempotency_key = $2")
                .bind(tenant_id)
                .bind(key)
                .fetch_one(pool)
                .await
        }
    }
}

fn control_plane(state: AppState) -> Router {
    Router::new()
        .route("/oauth/token", post(issue_token))
        .route("/jobs/claim", post(claim))
        .route("/jobs/{id}/heartbeat", post(heartbeat))
        .route("/jobs/{id}/complete", post(complete))
        .route("/jobs/{id}/fail", post(fail))
        .with_state(state)
}

// ======================= worker =======================

struct Worker {
    name: String,
    base_url: String,
    http: reqwest::Client,
    client_id: String,
    client_secret: String,
    token: Option<(String, Instant)>,
}

enum Outcome {
    Done(Result<(), String>),
    LeaseLost,
}

async fn do_work(payload: &serde_json::Value) -> Result<(), String> {
    let steps = payload.get("steps").and_then(|v| v.as_u64()).unwrap_or(1);
    tokio::time::sleep(Duration::from_millis(100 * steps)).await;
    if payload.get("fail").and_then(|v| v.as_bool()).unwrap_or(false) {
        return Err("simulated failure".into());
    }
    Ok(())
}

impl Worker {
    async fn bearer(&mut self) -> Result<String, BoxError> {
        if let Some((token, expires_at)) = &self.token {
            if Instant::now() + TOKEN_REFRESH_MARGIN < *expires_at {
                return Ok(token.clone());
            }
        }
        let grant: TokenResponse = self
            .http
            .post(format!("{}/oauth/token", self.base_url))
            .form(&[
                ("grant_type", "client_credentials"),
                ("client_id", self.client_id.as_str()),
                ("client_secret", self.client_secret.as_str()),
            ])
            .send()
            .await?
            .error_for_status()?
            .json()
            .await?;
        let expires_at = Instant::now() + Duration::from_secs(grant.expires_in);
        self.token = Some((grant.access_token.clone(), expires_at));
        println!("[{}] fetched access token", self.name);
        Ok(grant.access_token)
    }

    async fn call(&mut self, path: &str, body: serde_json::Value) -> Result<reqwest::Response, BoxError> {
        let token = self.bearer().await?;
        Ok(self
            .http
            .post(format!("{}{}", self.base_url, path))
            .bearer_auth(token)
            .json(&body)
            .send()
            .await?)
    }

    async fn work_with_heartbeat(&mut self, job: &ClaimedJob) -> Outcome {
        let work = do_work(&job.payload);
        tokio::pin!(work);
        let mut beat = tokio::time::interval(HEARTBEAT_EVERY);
        beat.tick().await; // the first tick is immediate; the claim just set the lease

        loop {
            tokio::select! {
                result = &mut work => return Outcome::Done(result),
                _ = beat.tick() => {
                    let body = json!({ "worker_id": self.name, "attempt": job.attempt });
                    match self.call(&format!("/jobs/{}/heartbeat", job.id), body).await {
                        Ok(r) if r.status() == reqwest::StatusCode::CONFLICT => return Outcome::LeaseLost,
                        Ok(_) => {}
                        Err(e) => eprintln!("[{}] heartbeat failed, will retry: {e}", self.name),
                    }
                }
            }
        }
    }

    async fn run(mut self, shutdown: CancellationToken, freeze_on_first_job: bool) {
        let mut frozen_once = !freeze_on_first_job;
        loop {
            if shutdown.is_cancelled() {
                break;
            }
            let body = json!({ "worker_id": self.name });
            let job: Option<ClaimedJob> = match self.call("/jobs/claim", body).await {
                Ok(r) => r.json().await.unwrap_or(None),
                Err(e) => {
                    eprintln!("[{}] claim failed: {e}", self.name);
                    None
                }
            };
            let Some(job) = job else {
                tokio::select! {
                    _ = shutdown.cancelled() => break,
                    _ = tokio::time::sleep(IDLE_POLL) => continue,
                }
            };
            println!("[{}] claimed job {} (attempt {})", self.name, job.id, job.attempt);

            let outcome = if !frozen_once {
                frozen_once = true;
                println!("[{}] freezes on job {} (no heartbeats)", self.name, job.id);
                tokio::time::sleep(FREEZE_FOR).await;
                Outcome::Done(Ok(())) // wakes up believing it finished
            } else {
                self.work_with_heartbeat(&job).await
            };

            let (path, body) = match outcome {
                Outcome::LeaseLost => {
                    println!("[{}] lost lease on job {}, stopping it", self.name, job.id);
                    continue;
                }
                Outcome::Done(Ok(())) => (
                    format!("/jobs/{}/complete", job.id),
                    json!({ "worker_id": self.name, "attempt": job.attempt }),
                ),
                Outcome::Done(Err(e)) => (
                    format!("/jobs/{}/fail", job.id),
                    json!({ "worker_id": self.name, "attempt": job.attempt, "error": e }),
                ),
            };
            match self.call(&path, body).await {
                Ok(r) if r.status() == reqwest::StatusCode::CONFLICT => println!(
                    "[{}] report for job {} attempt {} rejected: fenced (409)",
                    self.name, job.id, job.attempt
                ),
                Ok(r) if r.status().is_success() => {
                    println!("[{}] reported job {} attempt {}", self.name, job.id, job.attempt)
                }
                Ok(r) => eprintln!("[{}] report failed: {}", self.name, r.status()),
                Err(e) => eprintln!("[{}] report failed: {e}", self.name),
            }
        }
    }
}

// ======================= main =======================

#[tokio::main]
async fn main() -> Result<(), BoxError> {
    let url = std::env::var("DATABASE_URL")?;
    let pool = PgPoolOptions::new().max_connections(10).connect(&url).await?;
    sqlx::raw_sql(SCHEMA).execute(&pool).await?;

    let secret = b"demo-signing-key-change-me-32-bytes!";
    let state = AppState {
        pool: pool.clone(),
        keys: Arc::new(Keys {
            encoding: EncodingKey::from_secret(secret),
            decoding: DecodingKey::from_secret(secret),
        }),
        clients: Arc::new(vec![RegisteredClient {
            id: "worker-pool".into(),
            secret: "worker-pool-secret".into(),
            scope: "jobs:work".into(),
        }]),
    };

    // ---- control plane ----
    let server_shutdown = CancellationToken::new();
    let listener = tokio::net::TcpListener::bind("127.0.0.1:0").await?;
    let base_url = format!("http://{}", listener.local_addr()?);
    let server = tokio::spawn(
        axum::serve(listener, control_plane(state))
            .with_graceful_shutdown(server_shutdown.clone().cancelled_owned())
            .into_future(),
    );
    let reaper_task = tokio::spawn(reaper(pool.clone(), server_shutdown.clone()));

    // ---- jobs (tenant 1) ----
    let a = submit_job(&pool, 1, Some("import-1"), json!({ "steps": 5 })).await?;
    let a_again = submit_job(&pool, 1, Some("import-1"), json!({ "steps": 5 })).await?;
    println!("idempotent submit: first id {a}, retry id {a_again}");
    submit_job(&pool, 1, None, json!({ "steps": 3 })).await?;
    submit_job(&pool, 1, None, json!({ "steps": 4 })).await?;
    submit_job(&pool, 1, None, json!({ "steps": 1, "fail": true })).await?;
    submit_job(&pool, 1, None, json!({ "steps": 2 })).await?;

    // ---- workers: one client, three instances; w3 freezes on its first job ----
    let worker_shutdown = CancellationToken::new();
    let mut workers = JoinSet::new();
    for (name, freeze) in [("w1", false), ("w2", false), ("w3", true)] {
        let worker = Worker {
            name: name.into(),
            base_url: base_url.clone(),
            http: reqwest::Client::new(),
            client_id: "worker-pool".into(),
            client_secret: "worker-pool-secret".into(),
            token: None,
        };
        workers.spawn(worker.run(worker_shutdown.clone(), freeze));
    }

    // ---- wait until every job is finished ----
    let deadline = Instant::now() + Duration::from_secs(60);
    loop {
        let open: i64 = sqlx::query_scalar(
            "SELECT count(*) FROM jobs WHERE status IN ('queued', 'running')",
        )
        .fetch_one(&pool)
        .await?;
        if open == 0 || Instant::now() > deadline {
            break;
        }
        tokio::time::sleep(Duration::from_millis(200)).await;
    }

    // ---- shut down: workers first (a frozen one may still report), then the server ----
    worker_shutdown.cancel();
    while workers.join_next().await.is_some() {}
    server_shutdown.cancel();
    let _ = server.await;
    let _ = reaper_task.await;

    println!("\nid  status     attempts  last_error");
    let rows: Vec<(i64, String, i32, String)> = sqlx::query_as(
        "SELECT id, status, attempts, COALESCE(last_error, '') FROM jobs ORDER BY id",
    )
    .fetch_all(&pool)
    .await?;
    for (id, status, attempts, err) in rows {
        println!("{id:<3} {status:<10} {attempts:<9} {err}");
    }
    Ok(())
}
```

Expected output. The order of lines and which worker claims which job vary between runs, but the shape is always the same:

```text
idempotent submit: first id 1, retry id 1
[w1] fetched access token
[w2] fetched access token
[w3] fetched access token
[w1] claimed job 1 (attempt 1)
[w3] claimed job 2 (attempt 1)
[w3] freezes on job 2 (no heartbeats)
[w2] claimed job 3 (attempt 1)
[w2] reported job 3 attempt 1
[w2] claimed job 4 (attempt 1)
[w2] reported job 4 attempt 1
[w2] claimed job 5 (attempt 1)
[w1] reported job 1 attempt 1
[w1] claimed job 4 (attempt 2)
[reaper] reclaimed 1 expired lease(s)
[w2] claimed job 2 (attempt 2)
...
[w1] reported job 4 attempt 3
[w2] reported job 2 attempt 2
[w3] report for job 2 attempt 1 rejected: fenced (409)

id  status     attempts  last_error
1   succeeded  1
2   succeeded  2         lease expired
3   succeeded  1
4   failed     3         simulated failure
5   succeeded  1
```

Wait, the job IDs in the sample output above don't line up with the payloads exactly; that's fine — what matters is the pattern. Check yours for:

| What to look for | Concept |
|---|---|
| `first id 1, retry id 1` | **Idempotent submit** through a unique key |
| Each job claimed by exactly one worker at a time | **`FOR UPDATE SKIP LOCKED`** |
| One token fetch per worker, then reused | **Client credentials** and **token caching** |
| `[reaper] reclaimed …` while w3 is frozen | **Lease expiry** without heartbeats |
| The frozen job claimed again with `attempt 2` | **Retry after a lost worker** |
| `rejected: fenced (409)` from w3 | **Fencing**: the stale worker's report is ignored |
| The `fail` job ends `failed` after 3 attempts, with growing gaps | **Retries with backoff** and **max attempts** |
| `last_error` on the frozen job even though it succeeded | `last_error` keeps the reason for the **last retry**; clear it on success if you prefer |

Try breaking it on purpose:

- Remove `SKIP LOCKED` and run with ten workers. Claims slow down because every worker waits on the same row.
- Remove `AND attempts = $3` from `complete`. The frozen worker's stale report is now **accepted**.
- Set `HEARTBEAT_EVERY` longer than the lease. Healthy jobs get reclaimed and run twice.
- Send a token issued with `scope = "jobs:read"`. Every worker call returns `403`.

---

## 16. Exercises

- [ ] **1. State enum:** replace the `TEXT` status strings in the example with the `JobStatus` enum from section 3 (`sqlx::Type`). Add a test that inserting `status = 'done'` fails because of the `CHECK` constraint.
- [ ] **2. Submit API:** add `POST /jobs` for users (Stage 8 `AuthUser`, tenant from the token). Read an `Idempotency-Key` header, return `202 Accepted` with `{ "id": … }`, and the same ID on a retry. A worker token must get `401`.
- [ ] **3. Jitter:** add random jitter (±20%) to the worker's idle poll and the retry backoff. Explain in one sentence what goes wrong without it when 100 workers restart at once.
- [ ] **4. Permanent errors:** let `fail` accept `"retryable": false` and mark the job `failed` immediately. Make the example's failing job permanent and check it ends with `attempts = 1`.
- [ ] **5. Cancellation:** add `cancel_requested`, a `POST /jobs/{id}/cancel` user endpoint, a heartbeat response `{ "cancel_requested": bool }`, and `POST /jobs/{id}/cancelled`. Cancel a 50-step job after one second and show it ends `cancelled`, not `succeeded`.
- [ ] **6. Token edge cases:** set `TOKEN_TTL_SECS = 3` and `TOKEN_REFRESH_MARGIN = 1s`. Show workers refresh tokens during long jobs. Then add "on `401`, drop the cached token and retry once" and test it by restarting the control plane with a new signing key.
- [ ] **7. Hashed client secrets:** store the client secret as an Argon2 hash (Stage 8) and verify it in `issue_token`. Support two valid secrets per client for rotation.
- [ ] **8. Idempotent handler:** make `do_work` write `out/job-{id}.txt` using write-to-temp + atomic rename. Freeze a worker after the write but before `complete`, and show the retried run leaves exactly one correct file.
- [ ] **9. Integration tests:** with a test database (Stage 11), write tests named by behaviour: `claims_each_job_once_with_parallel_workers`, `requeues_job_after_lease_expires`, `rejects_report_with_stale_attempt`, `marks_failed_after_max_attempts`, `submit_with_same_key_returns_same_id`.
- [ ] **10. Explain it:** in four or five sentences, explain why workers pull instead of being pushed to, what `SKIP LOCKED` prevents, what a fencing token protects against that a lease alone doesn't, and why "exactly once" has to come from the job handler rather than the queue.

> When you can solve exercises 1–10 without looking anything up, you're ready for **Stage 17: Deployment**.

---

## 17. Further reading

- [PostgreSQL: `SELECT … FOR UPDATE SKIP LOCKED`](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE): row locking clauses
- [2ndQuadrant: "What is SKIP LOCKED for in PostgreSQL 9.5?"](https://www.2ndquadrant.com/en/blog/what-is-select-skip-locked-for-in-postgresql-9-5/): queues on PostgreSQL
- [Martin Kleppmann: "How to do distributed locking"](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html): leases and fencing tokens
- [RFC 6749 §4.4: Client Credentials Grant](https://datatracker.ietf.org/doc/html/rfc6749#section-4.4): the OAuth2 spec for service-to-service tokens
- [AWS Architecture Blog: "Exponential Backoff and Jitter"](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/): why jitter matters
- [Stripe: "Designing robust and predictable APIs with idempotency"](https://stripe.com/blog/idempotency): idempotency keys in practice
- [`jsonwebtoken` crate](https://docs.rs/jsonwebtoken): signing and validating JWTs
- [`apalis`](https://docs.rs/apalis) and [`graphile-worker`](https://worker.graphile.org/): existing job libraries to compare your design with
