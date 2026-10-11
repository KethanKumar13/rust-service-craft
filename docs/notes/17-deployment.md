# Stage 17: Deployment

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> If you come from JavaScript or TypeScript, the comparisons below map Rust concepts to familiar JS ideas (`node:alpine` images, `npm ci` layer caching, PM2 restarts, and GitHub Actions for Node projects).

## Contents

1. [From `cargo run` to a running service](#1-from-cargo-run-to-a-running-service)
2. [What the binary needs from its environment](#2-what-the-binary-needs-from-its-environment)
3. [Health endpoints: liveness and readiness](#3-health-endpoints-liveness-and-readiness)
4. [Handling `SIGTERM`](#4-handling-sigterm)
5. [Multi-stage Docker builds](#5-multi-stage-docker-builds)
6. [Caching dependencies with `cargo-chef`](#6-caching-dependencies-with-cargo-chef)
7. [Small, safe runtime images](#7-small-safe-runtime-images)
8. [SQLx offline mode in builds](#8-sqlx-offline-mode-in-builds)
9. [Local stack with Docker Compose](#9-local-stack-with-docker-compose)
10. [Kubernetes in one page](#10-kubernetes-in-one-page)
11. [Deployment, Service, probes, and resources](#11-deployment-service-probes-and-resources)
12. [Config and secrets](#12-config-and-secrets)
13. [Rolling updates and zero-downtime shutdown](#13-rolling-updates-and-zero-downtime-shutdown)
14. [Migrations in a multi-replica world](#14-migrations-in-a-multi-replica-world)
15. [Workers and scheduled work: Deployments, Jobs, CronJobs](#15-workers-and-scheduled-work-deployments-jobs-cronjobs)
16. [CI with GitHub Actions](#16-ci-with-github-actions)
17. [Building and publishing images](#17-building-and-publishing-images)
18. [How this looks in a production service](#18-how-this-looks-in-a-production-service)
19. [Putting it all together](#19-putting-it-all-together)
20. [Exercises](#20-exercises)
21. [Further reading](#21-further-reading)

---

## 1. From `cargo run` to a running service

So far everything ran with `cargo run` on your machine. A deployed service has different needs:

| On your laptop | In production |
|---|---|
| Debug build, compiled on the spot | Release build, compiled once, run many times |
| Config from `.env` | Config from the platform (env vars, mounted secrets) |
| You restart it when it breaks | The platform restarts it, so it must say when it's broken |
| Ctrl+C whenever | `SIGTERM` during deploys, with in-flight requests to finish |
| One copy | Several replicas, rolled one at a time |
| "Works on my machine" | Every change is built and tested the same way by CI |

The pieces in this stage:

```text
 git push ──▶ CI: fmt, clippy, nextest (with Postgres) ──▶ docker build ──▶ registry
                                                                              │
                                     Kubernetes: Deployment pulls the image ◀─┘
                                     probes ─▶ /healthz, /readyz
                                     SIGTERM ─▶ graceful shutdown
```

Rust helps a lot here. The output is **one static-ish binary** with no runtime to install, so images are small and start in milliseconds.

---

## 2. What the binary needs from its environment

A deployable service follows a few rules (often called the twelve-factor rules):

| Rule | In this repo |
|---|---|
| Config comes from the environment | Typed config from env vars (Stage 5), fail fast at startup if something is missing |
| Logs go to stdout | `tracing` with a JSON formatter in production (Stage 7); the platform collects stdout |
| Bind to `0.0.0.0` and a port from config | `127.0.0.1` inside a container is unreachable from outside it |
| No local state that matters | Files, sessions, and jobs live in the database or object storage (Stage 16) |
| Fast startup, graceful shutdown | Sections 3 and 4 |

```rust
#[derive(Debug, Clone)]
pub struct Config {
    pub bind_addr: std::net::SocketAddr, // e.g. 0.0.0.0:8080
    pub database_url: String,
    pub log_format: LogFormat,           // Pretty locally, Json in production
}
```

Validate config **before** accepting traffic. A pod that starts and then crashes on the first request is much harder to debug than one that refuses to start with `missing env var DATABASE_URL`.

---

## 3. Health endpoints: liveness and readiness

The platform asks your service two different questions:

| Probe | Question | On failure | Should check |
|---|---|---|---|
| **Liveness** `/healthz` | "Is the process stuck?" | **Restart** the container | Only that the process can answer HTTP |
| **Readiness** `/readyz` | "Can it serve traffic right now?" | **Stop sending** traffic, don't restart | Database reachable, startup finished, not shutting down |

```rust
async fn healthz() -> StatusCode {
    StatusCode::OK
}

async fn readyz(State(s): State<AppState>) -> StatusCode {
    if s.shutting_down.is_cancelled() {
        return StatusCode::SERVICE_UNAVAILABLE;          // draining: take me out of rotation
    }
    let db = tokio::time::timeout(Duration::from_secs(1), sqlx::query("SELECT 1").execute(&s.pool)).await;
    match db {
        Ok(Ok(_)) => StatusCode::OK,
        _ => StatusCode::SERVICE_UNAVAILABLE,
    }
}
```

The most common mistake: **checking the database in liveness.** If the database blips, every pod fails liveness, gets restarted at once, and the restarts make the outage worse. Dependencies belong in readiness only.

Keep both endpoints out of auth, rate limits, and request logging (Stage 7), or they'll fill your logs every few seconds.

---

## 4. Handling `SIGTERM`

When Kubernetes (or Docker) stops a container, it sends **`SIGTERM`**, waits for a grace period (30 s by default), then sends `SIGKILL`. Stage 15 handled Ctrl+C (`SIGINT`). In containers you must handle `SIGTERM` too:

```rust
use tokio_util::sync::CancellationToken;

async fn shutdown_signal(token: CancellationToken) {
    let ctrl_c = async {
        let _ = tokio::signal::ctrl_c().await;
    };

    #[cfg(unix)]
    let terminate = async {
        match tokio::signal::unix::signal(tokio::signal::unix::SignalKind::terminate()) {
            Ok(mut s) => { s.recv().await; }
            Err(e) => { tracing::error!(error = %e, "cannot listen for SIGTERM"); std::future::pending::<()>().await }
        }
    };
    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
    tracing::info!("shutdown signal received");
    token.cancel();
}
```

Then:

```rust
axum::serve(listener, app)
    .with_graceful_shutdown(token.clone().cancelled_owned())
    .await?;
```

`with_graceful_shutdown` stops accepting new connections and waits for in-flight requests. Background work (Stage 15 pipelines, Stage 16 workers) listens to the same token.

**PID 1 caveat:** the process that runs as PID 1 in a container doesn't get default signal handling. A Rust binary that installs its own handlers (as above) is fine. A shell-form `CMD my-service` starts `/bin/sh` as PID 1, which **doesn't forward** `SIGTERM`. Always use the exec form: `ENTRYPOINT ["/usr/local/bin/service"]`.

---

## 5. Multi-stage Docker builds

Compiling Rust needs the whole toolchain (over 1 GB). Running the result needs a few MB. A **multi-stage** Dockerfile builds in one image and copies only the binary into another:

```dockerfile
# ---- build ----
FROM rust:1-bookworm AS builder
WORKDIR /app
COPY . .
RUN cargo build --release --bin service

# ---- run ----
FROM debian:bookworm-slim
COPY --from=builder /app/target/release/service /usr/local/bin/service
ENTRYPOINT ["/usr/local/bin/service"]
```

This works, but every code change recompiles **every dependency**, because `COPY . .` invalidates the layer cache before `cargo build`. Section 6 fixes that.

Always add a `.dockerignore`, or the whole `target/` directory (often several GB) is sent to the build:

```text name=.dockerignore
target/
.git/
**/*.md
.env
```

> **JS comparison:** the same reason you `COPY package*.json` and `RUN npm ci` **before** `COPY . .`: dependencies change rarely, so they get their own cached layer.

---

## 6. Caching dependencies with `cargo-chef`

Cargo has no `npm ci` that installs dependencies without your code. **`cargo-chef`** creates one: it reduces the workspace to a "recipe" (manifests and lockfile only), builds dependencies from that, and caches the result as a layer.

```dockerfile
FROM lukemathwalker/cargo-chef:latest-rust-1 AS chef
WORKDIR /app

# 1. Compute the recipe: changes only when Cargo.toml / Cargo.lock change
FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

# 2. Build dependencies only: cached until the recipe changes
FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
RUN cargo chef cook --release --recipe-path recipe.json

# 3. Build the application: this is the only layer rebuilt on a code change
COPY . .
ENV SQLX_OFFLINE=true
RUN cargo build --release --locked --bin service
```

| Change | What rebuilds |
|---|---|
| Edit a `.rs` file | Step 3 only: seconds to a minute |
| Add a dependency | Steps 2 and 3 |
| Change Rust version | Everything |

`--locked` makes the build fail if `Cargo.lock` doesn't match `Cargo.toml`, so CI never silently resolves different versions than you tested.

---

## 7. Small, safe runtime images

The runtime stage only needs what the binary links against:

| Base | Size | Contains | Use when |
|---|---|---|---|
| `debian:bookworm-slim` | ~75 MB | Shell, package manager, glibc | You need to debug inside the container |
| `gcr.io/distroless/cc-debian12` | ~25 MB | glibc, libgcc, CA certs, no shell | **Default choice** for glibc builds |
| `scratch` | 0 MB | Nothing | Fully static `musl` builds, and you add CA certs yourself |
| `alpine` | ~8 MB | musl, busybox shell | musl builds that still want a shell |

```dockerfile
FROM gcr.io/distroless/cc-debian12:nonroot AS runtime
COPY --from=builder /app/target/release/service /usr/local/bin/service
USER nonroot
EXPOSE 8080
ENTRYPOINT ["/usr/local/bin/service"]
```

Notes:

- **Run as non-root.** The `:nonroot` tag gives a user with UID 65532. Kubernetes can enforce it (section 11).
- **CA certificates** are needed for any outbound HTTPS (OAuth token endpoints in Stage 16). Distroless includes them, but `scratch` does not.
- **TLS libraries:** prefer `rustls` features in `reqwest` and `sqlx` (`tls-rustls`). Then there's no OpenSSL to install or patch in the image.
- No shell means no `HEALTHCHECK CMD curl …` in Docker. Kubernetes probes use HTTP from outside, so that's fine. For Compose, see section 9.

---

## 8. SQLx offline mode in builds

`sqlx::query!` checks SQL against a live database **at compile time** (Stage 6). A Docker build has no database. SQLx's offline mode uses query metadata saved in the repository:

```bash
cargo sqlx prepare --workspace   # writes .sqlx/ with one JSON file per query
git add .sqlx
```

Then the build sets `SQLX_OFFLINE=true` (section 6). In CI, check that the saved metadata is current:

```bash
cargo sqlx prepare --workspace --check
```

If someone changes a query without re-running `prepare`, CI fails instead of the Docker build failing later, or the build passing against an outdated schema.

---

## 9. Local stack with Docker Compose

Compose runs the image you'll deploy, with its dependencies, on your machine:

```yaml name=compose.yaml
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app
    ports: ["5432:5432"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 2s
      retries: 20

  api:
    build: .
    environment:
      BIND_ADDR: 0.0.0.0:8080
      DATABASE_URL: postgres://app:app@postgres:5432/app
      LOG_FORMAT: json
    ports: ["8080:8080"]
    depends_on:
      postgres:
        condition: service_healthy
```

- Inside Compose, services reach each other by **service name** (`postgres`), not `localhost`.
- `depends_on: condition: service_healthy` waits for Postgres to accept connections, not just to start.
- Your app should **still retry** its first database connection. Order of startup isn't guaranteed in Kubernetes, so don't rely on Compose behaviour.

```bash
docker compose up --build
curl -i localhost:8080/readyz
docker compose stop api      # sends SIGTERM: check the logs for a clean shutdown
```

---

## 10. Kubernetes in one page

Kubernetes runs containers across a group of machines and keeps them in the state you describe. You write **what you want** (YAML), and controllers keep making it true.

| Object | What it is |
|---|---|
| **Pod** | One or more containers that run together. Disposable: can be killed and replaced anytime |
| **Deployment** | "Keep N identical pods of this image running", and rolls out new versions |
| **Service** | A stable name and IP that load-balances across the pods matching a label |
| **ConfigMap / Secret** | Configuration and sensitive values, injected as env vars or files |
| **Job / CronJob** | Run pods to completion, once or on a schedule |
| **Ingress / Gateway** | Routes outside HTTP traffic to Services |
| **Namespace** | A folder for objects, with its own permissions and quotas |

```text
 Ingress ──▶ Service "api" ──┬──▶ Pod api-7d9f…-abcde   (image: service:1.4.2)
   (HTTP)    (stable name)   ├──▶ Pod api-7d9f…-fghij
                             └──▶ Pod api-7d9f…-klmno
                                    ▲
                     Deployment "api" (replicas: 3) keeps these running
```

Local practice clusters: **kind** or **minikube**. Managed clusters in the cloud (AKS, EKS, GKE) use the same YAML.

> **JS comparison:** PM2 keeps Node processes alive on one machine. A Deployment does the same across many machines, and a Service is the load balancer in front of them.

---

## 11. Deployment, Service, probes, and resources

```yaml name=k8s/api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels: { app: api }
spec:
  replicas: 3
  selector:
    matchLabels: { app: api }
  strategy:
    rollingUpdate: { maxUnavailable: 0, maxSurge: 1 }   # add one new pod before removing an old one
  template:
    metadata:
      labels: { app: api }
    spec:
      terminationGracePeriodSeconds: 30
      securityContext:
        runAsNonRoot: true
        seccompProfile: { type: RuntimeDefault }
      containers:
        - name: api
          image: ghcr.io/your-github-user/rust-service-craft:1.0.0   # a fixed tag, never :latest
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef: { name: api-config }
            - secretRef: { name: api-secrets }
          readinessProbe:
            httpGet: { path: /readyz, port: 8080 }
            periodSeconds: 5
            failureThreshold: 2
          livenessProbe:
            httpGet: { path: /healthz, port: 8080 }
            periodSeconds: 10
            failureThreshold: 3
          startupProbe:                     # gives slow starts (migrations) time before liveness kicks in
            httpGet: { path: /healthz, port: 8080 }
            periodSeconds: 2
            failureThreshold: 30
          resources:
            requests: { cpu: 100m, memory: 64Mi }
            limits: { memory: 256Mi }
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          lifecycle:
            preStop:
              sleep: { seconds: 5 }        # see section 13
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector: { app: api }
  ports:
    - port: 80
      targetPort: 8080
```

**Resources:**

| Field | Meaning | If exceeded |
|---|---|---|
| `requests.cpu` / `requests.memory` | What the scheduler reserves for the pod | Used for placement only |
| `limits.memory` | Hard maximum | Container is **killed** (OOMKilled) |
| `limits.cpu` | Hard maximum | Container is **throttled** (slowed down) |

- Always set a **memory limit**. The bounded-memory work from Stages 10, 14, and 15 is what lets you pick a small one with confidence.
- Many teams **don't** set a CPU limit, because throttling causes latency spikes. Set CPU requests instead.
- Tokio uses one worker thread per visible CPU core, which may be the whole machine, not your request. Set `TOKIO_WORKER_THREADS` (or `worker_threads` in code) to match your CPU request on large nodes.
- With `readOnlyRootFilesystem: true`, give the pod an `emptyDir` volume for temp files (DataFusion spill files, Stage 14).

---

## 12. Config and secrets

```yaml name=k8s/config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  BIND_ADDR: "0.0.0.0:8080"
  LOG_FORMAT: "json"
  RUST_LOG: "info,sqlx=warn"
```

```bash
# Secrets are created from the command line or a secret manager, never committed
kubectl create secret generic api-secrets \
  --from-literal=DATABASE_URL='postgres://…' \
  --from-literal=JWT_SECRET_KEY='…'
```

| | ConfigMap | Secret |
|---|---|---|
| Content | Non-sensitive settings | Passwords, keys, tokens |
| Stored | Plain text | Base64 (encoding, **not** encryption) |
| In git | Yes | **No** |
| Protection | None needed | RBAC, encryption at rest, or an external secret store |

In real clusters, secrets usually come from a cloud secret store (Azure Key Vault, AWS Secrets Manager, HashiCorp Vault) through a CSI driver or an operator. The application still just reads environment variables or files, so its code doesn't change.

Changing a ConfigMap does **not** restart pods. Run `kubectl rollout restart deployment/api`, or add a checksum of the config as a pod annotation so changes trigger a rollout.

---

## 13. Rolling updates and zero-downtime shutdown

When you change the image tag, the Deployment replaces pods one at a time:

```text
1. start a new pod (maxSurge: 1)
2. wait until its readiness probe passes
3. pick an old pod to stop:
     a. remove it from the Service endpoints   ┐ these two happen
     b. send it SIGTERM (after preStop)        ┘ at the same time
4. repeat until all pods run the new version
```

Steps 3a and 3b race. Load balancers can keep sending requests to the old pod for a few seconds after it gets `SIGTERM`. If the app stops accepting connections immediately, those requests fail.

The fix, in order:

1. **`preStop` sleep (5 s):** the pod keeps serving normally while the endpoint removal spreads.
2. **`SIGTERM` arrives:** cancel the token. `/readyz` now returns 503, and `with_graceful_shutdown` stops accepting new connections and finishes in-flight ones.
3. **Background work** (pipelines, workers) stops at its next cancellation check.
4. **Exit** well before `terminationGracePeriodSeconds`, or `SIGKILL` cuts everything off.

Long-running work doesn't fit in 30 seconds. That's why Stage 16 uses leases. A worker killed mid-job loses its lease, and another worker picks the job up again.

```bash
kubectl set image deployment/api api=ghcr.io/your-github-user/rust-service-craft:1.0.1
kubectl rollout status deployment/api
kubectl rollout undo deployment/api         # back to the previous version
```

---

## 14. Migrations in a multi-replica world

Stage 6 ran `sqlx::migrate!()` at startup. With three replicas, three pods start at once:

| Option | How | Trade-off |
|---|---|---|
| Migrate on startup | `sqlx::migrate!().run(&pool)` in `main` | Simple. SQLx takes a PostgreSQL **advisory lock**, so only one pod migrates at a time and the others wait. Slow migrations delay startup (use `startupProbe`) |
| Separate migration Job | A Kubernetes `Job` runs `service migrate` before the rollout | Clearer failures and no app pods waiting, but more pipeline steps |

Either way, during a rolling update **old and new code run at the same time** against the **new** schema. Migrations must be backwards compatible with the previous version of the code. This approach is called expand and contract:

| Change | Safe way |
|---|---|
| Add a column | Add it nullable (or with a default) |
| Rename a column | Add the new one, write both, backfill, switch reads, then drop the old one in a **later** release |
| Drop a column | Stop using it in one release, drop it in the next |
| Add `NOT NULL` | Backfill first, then add the constraint |

---

## 15. Workers and scheduled work: Deployments, Jobs, CronJobs

Stage 16's pull-based workers map directly to Kubernetes objects:

| Work | Kubernetes object |
|---|---|
| Long-running workers that poll for jobs | **Deployment** (scale `replicas` up and down) |
| One-off task (migration, backfill) | **Job** |
| Periodic task (cleanup, nightly report) | **CronJob** |
| One pod per job, started by your control plane | **Job**, created through the Kubernetes API |

```yaml name=k8s/cleanup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-finished-jobs
spec:
  schedule: "0 3 * * *"               # 03:00 every day
  concurrencyPolicy: Forbid           # never two at once
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 600
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: cleanup
              image: ghcr.io/your-github-user/rust-service-craft:1.0.0
              args: ["cleanup", "--older-than-days", "30"]
              envFrom:
                - secretRef: { name: api-secrets }
```

One image, several **subcommands** (`serve`, `worker`, `migrate`, `cleanup`) is a common pattern. Use `clap` to parse them. The same build and the same tested binary run in every role.

---

## 16. CI with GitHub Actions

CI runs on every push and pull request. It runs the same checks you'd run locally, on a clean machine:

```yaml name=.github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

env:
  CARGO_TERM_COLOR: always
  SQLX_OFFLINE: "true"

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --all --check
      - run: cargo clippy --workspace --all-targets --locked -- -D warnings

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: app
          POSTGRES_DB: app_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U app"
          --health-interval 2s
          --health-retries 20
    env:
      DATABASE_URL: postgres://app:app@localhost:5432/app_test
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - uses: taiki-e/install-action@v2
        with:
          tool: cargo-nextest,sqlx-cli
      - run: cargo sqlx prepare --workspace --check
      - run: cargo nextest run --workspace --locked
```

| Piece | Why |
|---|---|
| `Swatinem/rust-cache` | Caches `~/.cargo` and `target/`: CI drops from many minutes to a few |
| `-D warnings` | A Clippy warning fails the build, so warnings don't pile up |
| `services: postgres` | A real database for Stage 11 integration tests |
| `sqlx prepare --check` | The `.sqlx/` metadata matches the queries (section 8) |
| `taiki-e/install-action` | Installs prebuilt `cargo-nextest` and `sqlx-cli` in seconds instead of compiling them |
| Separate `lint` and `test` jobs | They run in parallel, and failures are easier to read |

Make the workflow **required** in branch protection, so nothing merges to `main` without passing.

---

## 17. Building and publishing images

After tests pass on `main`, build the image and push it to a registry. GitHub Container Registry (GHCR) works with the built-in `GITHUB_TOKEN`:

```yaml name=.github/workflows/ci.yml
  image:
    needs: [lint, test]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:main
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

Tagging rules:

- **Deploy an immutable tag** (the commit SHA or a release version like `1.4.2`). Never deploy `:latest` or `:main`, because you can't tell which code is running, and rollbacks become guesswork.
- `cache-from/cache-to: type=gha` keeps the `cargo-chef` dependency layer between CI runs.
- `permissions:` grants only what this job needs. Other jobs don't get `packages: write`.

Deploying to a cluster from CI (`kubectl apply`, Helm, or a GitOps tool such as Argo CD or Flux watching a config repository) is the next step. The image and the YAML are the same either way.

---

## 18. How this looks in a production service

```text
.
├── Dockerfile                 # cargo-chef build, distroless nonroot runtime
├── .dockerignore
├── compose.yaml               # local Postgres + the real image
├── .sqlx/                     # SQLx offline metadata, checked in CI
├── k8s/
│   ├── api.yaml               # Deployment + Service, probes, resources, securityContext
│   ├── worker.yaml            # Deployment running `service worker`
│   ├── config.yaml            # ConfigMap (no secrets)
│   └── cleanup-cronjob.yaml
├── .github/workflows/ci.yml   # lint, test with Postgres, build and push on main
└── crates/api/src/
    ├── main.rs                # clap subcommands: serve, worker, migrate, cleanup
    ├── health.rs              # /healthz (process only), /readyz (DB + not draining)
    └── shutdown.rs            # SIGTERM + Ctrl+C → CancellationToken
```

Conventions:

- **One image, many roles** through subcommands.
- **Config from env, validated at startup.** JSON logs to stdout.
- **Liveness checks the process only.** Readiness checks dependencies and draining.
- **`SIGTERM` → token → readiness 503 → graceful shutdown**, with `preStop` sleep and a grace period longer than your slowest request.
- **Non-root, read-only filesystem, all capabilities dropped.**
- **Memory limit always.** CPU request set, CPU limit usually not.
- **Backwards-compatible migrations** (expand and contract).
- **Immutable image tags**, built only after lint and tests pass.
- **No secrets in git, images, or logs.**

---

## 19. Putting it all together

A minimal deployable service: health endpoints, `SIGTERM` handling, readiness that turns off while draining, and a simulated slow request to show graceful shutdown. Use it with the `Dockerfile`, `compose.yaml`, Kubernetes YAML, and workflow from the sections above.

```toml
[package]
name = "service"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.8"
sqlx = { version = "0.8", features = ["runtime-tokio", "tls-rustls", "postgres"] }
tokio = { version = "1", features = ["full"] }
tokio-util = "0.7"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
```

```rust
use std::net::SocketAddr;
use std::time::Duration;

use axum::{extract::State, http::StatusCode, routing::get, Router};
use sqlx::{postgres::PgPoolOptions, PgPool};
use tokio_util::sync::CancellationToken;

type BoxError = Box<dyn std::error::Error + Send + Sync>;

// ---------- config ----------

struct Config {
    bind_addr: SocketAddr,
    database_url: String,
    json_logs: bool,
}

impl Config {
    fn from_env() -> Result<Self, BoxError> {
        let bind_addr = std::env::var("BIND_ADDR")
            .unwrap_or_else(|_| "0.0.0.0:8080".into())
            .parse()
            .map_err(|e| format!("BIND_ADDR is invalid: {e}"))?;
        let database_url = std::env::var("DATABASE_URL").map_err(|_| "missing env var DATABASE_URL")?;
        let json_logs = std::env::var("LOG_FORMAT").map(|v| v == "json").unwrap_or(false);
        Ok(Self { bind_addr, database_url, json_logs })
    }
}

fn init_tracing(json: bool) {
    let filter = tracing_subscriber::EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| "info".into());
    let builder = tracing_subscriber::fmt().with_env_filter(filter);
    if json {
        builder.json().init();
    } else {
        builder.init();
    }
}

// ---------- state and handlers ----------

#[derive(Clone)]
struct AppState {
    pool: PgPool,
    draining: CancellationToken,
}

async fn healthz() -> StatusCode {
    StatusCode::OK // process is alive: no dependency checks here
}

async fn readyz(State(s): State<AppState>) -> StatusCode {
    if s.draining.is_cancelled() {
        return StatusCode::SERVICE_UNAVAILABLE;
    }
    let check = tokio::time::timeout(Duration::from_secs(1), sqlx::query("SELECT 1").execute(&s.pool)).await;
    match check {
        Ok(Ok(_)) => StatusCode::OK,
        _ => StatusCode::SERVICE_UNAVAILABLE,
    }
}

async fn slow() -> &'static str {
    tokio::time::sleep(Duration::from_secs(5)).await; // an in-flight request during shutdown
    "finished\n"
}

// ---------- shutdown ----------

async fn wait_for_signal(token: CancellationToken) {
    let ctrl_c = async {
        let _ = tokio::signal::ctrl_c().await;
    };

    #[cfg(unix)]
    let terminate = async {
        match tokio::signal::unix::signal(tokio::signal::unix::SignalKind::terminate()) {
            Ok(mut s) => {
                s.recv().await;
            }
            Err(e) => {
                tracing::error!(error = %e, "cannot listen for SIGTERM");
                std::future::pending::<()>().await;
            }
        }
    };
    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => tracing::info!("SIGINT received"),
        _ = terminate => tracing::info!("SIGTERM received"),
    }
    token.cancel();
}

// ---------- main ----------

#[tokio::main]
async fn main() -> Result<(), BoxError> {
    let config = Config::from_env()?; // fail fast before accepting traffic
    init_tracing(config.json_logs);

    // lazy pool: the process starts even if the database is briefly unavailable;
    // readiness stays 503 until it's reachable
    let pool = PgPoolOptions::new()
        .max_connections(10)
        .acquire_timeout(Duration::from_secs(2))
        .connect_lazy(&config.database_url)?;

    let draining = CancellationToken::new();
    let state = AppState { pool: pool.clone(), draining: draining.clone() };

    let app = Router::new()
        .route("/healthz", get(healthz))
        .route("/readyz", get(readyz))
        .route("/slow", get(slow))
        .with_state(state);

    tokio::spawn(wait_for_signal(draining.clone()));

    let listener = tokio::net::TcpListener::bind(config.bind_addr).await?;
    tracing::info!(addr = %config.bind_addr, "listening");

    axum::serve(listener, app)
        .with_graceful_shutdown(draining.cancelled_owned())
        .await?;

    pool.close().await;
    tracing::info!("shutdown complete");
    Ok(())
}
```

Try it with Compose:

```bash
docker compose up --build -d
curl -i localhost:8080/healthz             # 200
curl -i localhost:8080/readyz              # 200 once Postgres is up

curl localhost:8080/slow &                 # start a 5-second request
sleep 1 && docker compose stop api         # sends SIGTERM
wait                                       # the slow request still prints "finished"
docker compose logs api | tail -n 3
```

Expected log lines (JSON fields shortened):

```text
{"level":"INFO","fields":{"message":"SIGTERM received"}}
{"level":"INFO","fields":{"message":"shutdown complete"}}
```

Then on a local cluster:

```bash
kind create cluster
docker build -t service:dev .
kind load docker-image service:dev
# set image: service:dev in k8s/api.yaml, create the secret, then:
kubectl apply -f k8s/
kubectl rollout status deployment/api
kubectl port-forward svc/api 8080:80
```

What each part shows:

| Part | Concept |
|---|---|
| `Config::from_env()?` before anything else | **Fail fast** on bad config |
| `/healthz` with no DB check, `/readyz` with one | **Liveness vs readiness** |
| `connect_lazy` | Starts even if the database isn't ready yet, so readiness reports it |
| `/slow` finishes after `docker compose stop` | **Graceful shutdown** on `SIGTERM` |
| `/readyz` returns 503 while draining | **Taken out of rotation** before connections close |
| JSON logs to stdout | **Platform-friendly logging** |

Try breaking it on purpose:

- Change the Dockerfile to the shell form `ENTRYPOINT /usr/local/bin/service`. `docker compose stop` now waits 10 s and kills the process, and `/slow` fails, because the shell doesn't forward `SIGTERM`.
- Add the database check to `/healthz` and stop Postgres. In Kubernetes, every pod restarts in a loop.
- Set `limits.memory: 8Mi` and watch the pod get `OOMKilled` in `kubectl describe pod`.
- Remove `.dockerignore` and compare the "transferring context" size in the build output.

---

## 20. Exercises

- [ ] **1. Image size:** build the section 5 (single `debian:bookworm-slim`) and section 7 (distroless) images. Compare sizes with `docker images`. Then try a static `x86_64-unknown-linux-musl` build on `scratch` and note what you had to add (CA certificates).
- [ ] **2. Cache check:** with `cargo-chef`, change one line in `main.rs` and time the rebuild. Then add a dependency and time it again. Explain the difference.
- [ ] **3. Subcommands:** add `clap` with `serve`, `migrate`, and `worker` subcommands. `migrate` runs `sqlx::migrate!()` and exits. Run it as a Kubernetes `Job` before the Deployment.
- [ ] **4. Draining test:** write a test that starts the server, opens a slow request, cancels the token, and asserts: `/readyz` returns 503, the slow request completes, and new connections are refused.
- [ ] **5. Probes in action:** deploy to kind with 3 replicas. Stop Postgres and watch `kubectl get pods`: pods should become **not ready**, not restart. Start Postgres again and watch them become ready.
- [ ] **6. Zero-downtime rollout:** run `hey` or `oha` against the Service during `kubectl set image …`. Record failed requests with and without the `preStop` sleep.
- [ ] **7. Tokio and CPU requests:** set `requests.cpu: 500m` and log `std::thread::available_parallelism()` at startup. Then set `TOKIO_WORKER_THREADS=1` and compare.
- [ ] **8. CI:** add the section 16 workflow to the repo. Break `cargo fmt` in a branch and confirm the PR is blocked. Change a `query!` without running `cargo sqlx prepare` and confirm CI fails on `--check`.
- [ ] **9. Expand and contract:** plan (as migrations and code changes across three releases) renaming `projects.name` to `projects.title` with no downtime during rolling updates.
- [ ] **10. Explain it:** in four or five sentences, explain the difference between liveness and readiness, why the exec form of `ENTRYPOINT` matters, why the `preStop` sleep exists, and why you deploy an immutable image tag.

> When you can solve exercises 1–10 without looking anything up, you're ready for **Stage 18: Performance and Observability**.

---

## 21. Further reading

- [Docker: multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [`cargo-chef`](https://github.com/LukeMathWalker/cargo-chef) and [Luca Palmieri: "5x Faster Rust Docker Builds with cargo-chef"](https://www.lpalmieri.com/posts/fast-rust-docker-builds/)
- [Distroless images](https://github.com/GoogleContainerTools/distroless)
- [SQLx offline mode](https://github.com/launchbadge/sqlx/blob/main/sqlx-cli/README.md#enable-building-in-offline-mode-with-query)
- [Kubernetes: Configure liveness, readiness and startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Kubernetes: Pod lifecycle and termination](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)
- [Kubernetes: Resource management for pods](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [kind: quick start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [GitHub Actions: service containers](https://docs.github.com/en/actions/use-cases-and-examples/using-containerized-services/about-service-containers) and [publishing Docker images](https://docs.github.com/en/actions/use-cases-and-examples/publishing-packages/publishing-docker-images)
- [`Swatinem/rust-cache`](https://github.com/Swatinem/rust-cache)
- [The Twelve-Factor App](https://12factor.net/)
