# Stage 9: Realtime with WebSockets

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> If you come from JavaScript or TypeScript, the comparisons below map Rust concepts to familiar JS ideas (the browser `WebSocket` API and the `ws` package).

## Contents

1. [When you need realtime](#1-when-you-need-realtime)
2. [How a WebSocket connection works](#2-how-a-websocket-connection-works)
3. [A first WebSocket handler](#3-a-first-websocket-handler)
4. [Messages and a typed protocol](#4-messages-and-a-typed-protocol)
5. [Reading and writing at the same time: `split` and `select!`](#5-reading-and-writing-at-the-same-time-split-and-select)
6. [Authenticating the upgrade](#6-authenticating-the-upgrade)
7. [Presence: who is here right now](#7-presence-who-is-here-right-now)
8. [Broadcasting to a room](#8-broadcasting-to-a-room)
9. [Cleanup, heartbeats, and limits](#9-cleanup-heartbeats-and-limits)
10. [Scaling beyond one process](#10-scaling-beyond-one-process)
11. [How this looks in a production service](#11-how-this-looks-in-a-production-service)
12. [Putting it all together](#12-putting-it-all-together)
13. [Exercises](#13-exercises)
14. [Further reading](#14-further-reading)

---

## 1. When you need realtime

Plain HTTP is request → response: the server can only answer when the client asks. Some features need the **server to push**:

| Feature | Example |
|---|---|
| Presence | "Anna and Ben are viewing this project" |
| Live updates | Another user changed a field; refresh it without reloading |
| Notifications | "Your export is ready" |
| Collaboration | Chat, cursors, comments appearing live |

| Option | Direction | Notes |
|---|---|---|
| Polling (`GET` every few seconds) | Client → server | Simple, wasteful, delayed |
| Server-Sent Events (SSE) | Server → client only | Plain HTTP, automatic reconnect; good for notifications |
| **WebSockets** | Both ways | One long-lived connection; good for presence and collaboration |

This stage uses WebSockets because presence needs both directions: clients announce themselves, and the server pushes who else is there.

---

## 2. How a WebSocket connection works

A WebSocket starts as a normal HTTP request and then **upgrades**:

```text
GET /ws/projects/7 HTTP/1.1
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Cookie: access_token_cookie=eyJ...          <- the browser sends cookies here
Origin: http://localhost:5173

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
```

After the `101`, the TCP connection carries **frames** in both directions until either side closes it:

| Frame | Purpose |
|---|---|
| Text | UTF-8 text, usually JSON |
| Binary | Raw bytes |
| Ping / Pong | Keep-alive and liveness checks |
| Close | Orderly shutdown, with an optional code and reason |

Two consequences:

- **Everything you know about the request is decided at upgrade time.** Authentication, the project ID, and the origin check all happen in the normal HTTP handler, before the `101`.
- **The connection is a long-lived task.** It holds memory and a slot in your presence map for minutes or hours, so cleanup matters (section 9).

> **JS comparison:** the browser side is `const ws = new WebSocket("ws://localhost:3000/ws/projects/7"); ws.onmessage = (e) => ...; ws.send(...)`. On the server, Node's `ws` package gives you a `connection` event; Axum gives you an `async fn` that owns the socket.

---

## 3. A first WebSocket handler

```toml
axum = { version = "0.8", features = ["ws"] }
futures = "0.3"
```

```rust
use axum::{
    extract::ws::{Message, WebSocket, WebSocketUpgrade},
    response::Response,
    routing::get,
    Router,
};

async fn ws_echo(ws: WebSocketUpgrade) -> Response {
    // 1. The HTTP part: runs before the upgrade, can reject with a normal status.
    ws.on_upgrade(handle_socket)
    // 2. Axum sends 101 and runs handle_socket in a new task.
}

async fn handle_socket(mut socket: WebSocket) {
    while let Some(Ok(msg)) = socket.recv().await {
        match msg {
            Message::Text(text) => {
                if socket.send(Message::Text(format!("echo: {text}").into())).await.is_err() {
                    break;                 // the client went away
                }
            }
            Message::Close(_) => break,
            _ => {}                        // binary, ping, pong
        }
    }
    // socket dropped here: the connection is closed
}

let app = Router::new().route("/ws/echo", get(ws_echo));
```

- `WebSocketUpgrade` is an **extractor**. It checks the upgrade headers and rejects non-WebSocket requests.
- `on_upgrade` takes a closure that receives the `WebSocket` **after** the `101` has been sent. It runs as its own Tokio task, so the Stage 1 rules apply: `'static`, `Send`, and `move` whatever it needs.
- Because the HTTP handler returns immediately, a `TimeoutLayer` (Stage 7) on the router doesn't cut long-lived sockets off. The timeout only covers the upgrade response.
- In Axum 0.8, `Message::Text` holds `Utf8Bytes` rather than `String`. Use `.into()` to build one and `.as_str()` to read it.

---

## 4. Messages and a typed protocol

Raw strings get messy quickly. Define **one enum per direction** and let serde tag them:

```rust
use serde::{Deserialize, Serialize};

/// Server -> client
#[derive(Debug, Clone, Serialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum ServerEvent {
    Snapshot { users: Vec<i32> },
    Joined { user_id: i32 },
    Left { user_id: i32 },
    Chat { user_id: i32, text: String },
    Error { message: String },
}

/// Client -> server
#[derive(Debug, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum ClientEvent {
    Chat { text: String },
}
```

On the wire:

```json
{"type":"joined","user_id":2}
{"type":"chat","text":"hello"}
```

- `#[serde(tag = "type")]` produces the `{"type": ...}` shape that frontends can `switch` on, the same idea as the stable `error` codes in Stage 4.
- Invalid client messages are answered with an `Error` event. They **don't** close the connection and don't crash the task.
- Validate client input as you would a JSON body (Stage 3): maximum text length, allowed values.

> **JS comparison:** the same as a TypeScript discriminated union: `type ServerEvent = { type: "joined"; user_id: number } | { type: "left"; user_id: number } | ...`.

---

## 5. Reading and writing at the same time: `split` and `select!`

A presence connection must do two things at once:

- **read** what this client sends
- **write** events that other connections produced

`socket.split()` (from `futures::StreamExt`) returns a separate sender and receiver, and `tokio::select!` (Stage 1) waits on whichever is ready first:

```rust
use futures::{SinkExt, StreamExt};

async fn handle_socket(socket: WebSocket, mut events: broadcast::Receiver<ServerEvent>) {
    let (mut sender, mut receiver) = socket.split();

    loop {
        tokio::select! {
            incoming = receiver.next() => match incoming {
                Some(Ok(Message::Text(text))) => { /* parse ClientEvent, act on it */ }
                Some(Ok(Message::Close(_))) | Some(Err(_)) | None => break,
                Some(Ok(_)) => {}
            },
            event = events.recv() => match event {
                Ok(ev) => {
                    let Ok(json) = serde_json::to_string(&ev) else { continue };
                    if sender.send(Message::Text(json.into())).await.is_err() {
                        break;
                    }
                }
                Err(broadcast::error::RecvError::Lagged(_)) => continue,
                Err(broadcast::error::RecvError::Closed) => break,
            },
        }
    }
}
```

- Both `receiver.next()` and `broadcast::Receiver::recv()` are **cancel-safe**: when `select!` picks the other branch, nothing is lost.
- `break` ends the loop, the socket halves are dropped, and the connection closes.
- The alternative is two spawned tasks (one reader, one writer) joined with `select!` on their handles. One loop is simpler to reason about and is enough here.

---

## 6. Authenticating the upgrade

### 6.1 Cookies work, headers don't

The browser `WebSocket` API **can't set custom headers**, so `Authorization: Bearer` isn't available from a browser. It **does** send cookies with the upgrade request. That's one more reason the Stage 8 `access_token_cookie` is useful: the same `AuthUser` extractor protects WebSockets with no extra code.

```rust
async fn ws_project(
    State(state): State<AppState>,
    user: AuthUser,                    // Stage 8: 401 before the upgrade if missing or invalid
    Path(project_id): Path<ProjectId>,
    headers: HeaderMap,
    ws: WebSocketUpgrade,
) -> Result<Response, AppError> {
    check_origin(&headers, &state.config)?;          // section 6.2
    user.require(caps::PROJECTS_READ)?;              // 403
    domain::projects::ensure_exists(&state.pool, user.tenant_id, project_id).await?; // 404

    Ok(ws.on_upgrade(move |socket| handle_socket(socket, state, user, project_id)))
}
```

All checks happen **before** `on_upgrade`, so failures are normal HTTP responses (401, 403, 404) with the usual `AppError` body.

For non-browser clients, the bearer header still works through the same extractor. If you can't use cookies, another common approach is a short-lived one-time **ticket** in the query string, issued by an authenticated `POST /ws-ticket`.

### 6.2 Check the `Origin` header

Because cookies are sent automatically, a malicious page on another site could open a WebSocket to your API **as the logged-in user**. This is cross-site WebSocket hijacking. CORS (Stage 7) **doesn't** apply to WebSockets, so you check the origin yourself:

```rust
fn check_origin(headers: &HeaderMap, config: &Config) -> Result<(), AppError> {
    match headers.get(header::ORIGIN) {
        None => Ok(()),                                     // non-browser clients send no Origin
        Some(origin) if config.allowed_origins.iter().any(|o| o.as_bytes() == origin.as_bytes()) => Ok(()),
        Some(_) => Err(AppError::Forbidden),
    }
}
```

### 6.3 Tokens expire, sockets don't

The token is checked **once**, at upgrade. A socket can stay open after the token expires or is revoked. Options, from simple to strict:

- Close sockets after the token's `exp` (start a `sleep_until` branch in the `select!` loop).
- Re-check the blocklist on a timer.
- On logout, signal that user's connections to close (a `watch` or `broadcast` channel keyed by `jti`).

---

## 7. Presence: who is here right now

Presence is **in-memory shared state** (Stage 5, section 7): it changes constantly, it doesn't need to survive a restart, and it's read often.

### 7.1 The shape

```rust
type RoomKey = (TenantId, ProjectId);       // tenant first: rooms never mix tenants

struct Room {
    members: HashMap<ConnId, UserId>,       // per connection: one user can have two tabs open
    tx: broadcast::Sender<ServerEvent>,     // section 8
}

#[derive(Clone, Default)]
pub struct Presence(Arc<RwLock<HashMap<RoomKey, Room>>>);
```

Design decisions:

- **Key rooms by tenant and project.** Even if two tenants had the same project ID, their rooms stay separate. This is the Stage 6 tenant-scoping rule, applied to memory.
- **Track connections, not users.** If Anna has two tabs open and closes one, she's still present. Store a `ConnId` (a counter from an `AtomicU64`) per connection.
- **Wrap the lock in methods** (`join`, `leave`, `snapshot`). Handlers never see the lock, so they can't hold it across an `.await`.

### 7.2 `join` and `leave`

```rust
impl Presence {
    pub fn join(&self, key: RoomKey, conn: ConnId, user: UserId) -> Result<Joined, AppError> {
        let mut rooms = self.0.write().map_err(|_| AppError::Internal("presence poisoned".into()))?;
        let room = rooms.entry(key).or_insert_with(|| Room {
            members: HashMap::new(),
            tx: broadcast::channel(64).0,
        });

        let _ = room.tx.send(ServerEvent::Joined { user_id: user.0 }); // existing members only
        let rx = room.tx.subscribe();                                  // after, so we don't see our own join
        room.members.insert(conn, user);

        let mut users: Vec<i32> = room.members.values().map(|u| u.0).collect();
        users.sort_unstable();
        users.dedup();                                                 // two tabs = one user

        Ok(Joined { snapshot: users, tx: room.tx.clone(), rx })
    }

    pub fn leave(&self, key: RoomKey, conn: ConnId) {
        let Ok(mut rooms) = self.0.write() else { return };
        let mut empty = false;
        if let Some(room) = rooms.get_mut(&key) {
            if let Some(user) = room.members.remove(&conn) {
                let _ = room.tx.send(ServerEvent::Left { user_id: user.0 });
            }
            empty = room.members.is_empty();
        }
        if empty {
            rooms.remove(&key);            // bounded memory: no empty rooms left behind
        }
    }
}
```

Everything happens under one short write lock with **no `.await`**, so `std::sync::RwLock` is correct (Stage 1, section 7). Exercise 6 refines `Left` so it's only sent when a user's **last** connection closes.

---

## 8. Broadcasting to a room

Each room has a `tokio::sync::broadcast` channel (Stage 1, section 5.3):

```text
            ┌────────────── Room (tenant 1, project 7) ──────────────┐
 Anna conn ─┤ tx.send(Chat)                                          │
            │        broadcast::Sender ──▶ rx (Anna) ──▶ Anna socket  │
            │                         └──▶ rx (Ben)  ──▶ Ben socket   │
            └─────────────────────────────────────────────────────────┘
```

- Every connection holds a `Receiver` and forwards events to its socket in the `select!` loop.
- Sending is **non-blocking**: `send` puts the event in a fixed-size ring buffer (capacity 64 here) and returns immediately. A slow client can't slow down the others.
- If a client falls more than 64 events behind, its `recv` returns **`Lagged(n)`**: it missed `n` events. Options: skip them, send a fresh `Snapshot` so the client resyncs, or close the connection. Never use an unbounded buffer per client: one stuck browser would grow memory without limit.
- `send` returns `Err` when nobody is subscribed. That's normal; ignore it.

> **JS comparison:** like an `EventEmitter` per room, except each listener has a bounded queue and is told explicitly when it fell behind.

---

## 9. Cleanup, heartbeats, and limits

### 9.1 Always leave the room: a `Drop` guard

The connection task can end in many ways: the client closes, the network drops, a send fails, or the server shuts down. Calling `leave` at the end of the function misses some of them. A **guard** whose `Drop` calls `leave` covers all of them, including cancellation (Stage 1, section 6.3):

```rust
struct PresenceGuard {
    presence: Presence,
    key: RoomKey,
    conn: ConnId,
}

impl Drop for PresenceGuard {
    fn drop(&mut self) {
        self.presence.leave(self.key, self.conn);
    }
}

// In handle_socket, right after join:
let _guard = PresenceGuard { presence: state.presence.clone(), key, conn };
```

`Drop` can't be `async`, which is fine here: `leave` only takes a `std` lock briefly.

### 9.2 Heartbeats

A client that loses its network (laptop lid closed, train tunnel) doesn't send a Close frame. Without a heartbeat, its connection and presence entry stay forever.

```rust
let mut heartbeat = tokio::time::interval(Duration::from_secs(20));
heartbeat.tick().await;                    // the first tick fires immediately; skip it

// one more branch in the select! loop:
_ = heartbeat.tick() => {
    if sender.send(Message::Ping(Bytes::new())).await.is_err() {
        break;                             // the connection is dead
    }
}
```

Browsers answer pings automatically. Axum also replies to pings it receives. For stricter liveness, record when the last `Pong` arrived and close the socket if it's older than, say, 60 seconds.

### 9.3 Limits

| Limit | How |
|---|---|
| Message size | `ws.max_message_size(64 * 1024)` before `on_upgrade` |
| Per-client queue | Bounded `broadcast` capacity; handle `Lagged` |
| Message rate | Count messages per window in the loop; close abusive connections |
| Connections per user | Count in `Presence`; reject the upgrade with 429 above a limit |

---

## 10. Scaling beyond one process

`Presence` lives in **one process's memory**. With three replicas behind a load balancer:

- Anna's socket is on replica A and Ben's is on replica B, so they don't see each other.
- A restart drops every socket. Clients must **reconnect** (with backoff) and rejoin.

Common solutions:

| Approach | Idea |
|---|---|
| Single instance for realtime | Simple; fine for internal tools with modest load |
| Sticky sessions | Route each project to the same replica; partial fix only |
| Pub/sub fan-out | Each replica publishes events to Redis pub/sub or PostgreSQL `LISTEN/NOTIFY`, and every replica forwards them to its local sockets |
| Shared presence store | Presence entries in Redis with a TTL, refreshed by heartbeats |

Start with one process and the in-memory design. It's the right default until there's a real need, and keeping the lock behind `Presence` methods means you can swap the implementation later without touching handlers.

---

## 11. How this looks in a production service

```text
crates/api/src/
├── state.rs              # AppState { pool, config, jwt, blocklist, presence: Presence, next_conn }
├── realtime/
│   ├── mod.rs            # router(): GET /ws/projects/{id}
│   ├── protocol.rs       # ServerEvent, ClientEvent (or in shared/types)
│   ├── presence.rs       # Presence, Room, join/leave/snapshot, PresenceGuard
│   └── socket.rs         # ws_project (checks), handle_socket (select! loop)
```

Conventions:

- The upgrade handler does **all** checks: `AuthUser`, origin, capability, tenant-scoped existence.
- Rooms are keyed by `(tenant_id, resource_id)`.
- Presence is behind methods; no lock is visible outside `presence.rs`.
- Every connection has a `Drop` guard, a heartbeat, a message-size limit, and a bounded event queue.
- Event types are tagged enums shared with the frontend's TypeScript types.

---

## 12. Putting it all together

A presence and chat server with tenant-scoped rooms, auth and origin checks on the upgrade, a typed protocol, a `broadcast` channel per room, a `Drop` guard, and a heartbeat. The `main` function starts the server on a random local port and connects **real WebSocket clients** with `tokio-tungstenite`, so it runs as a single local project. The token is the fake Stage 3 format to keep the example short; swap in the Stage 8 `AuthUser` for real use.

```toml name=Cargo.toml
[package]
name = "presence-demo"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = { version = "0.8", features = ["ws"] }
futures = "0.3"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "2"
tokio = { version = "1", features = ["full"] }
tokio-tungstenite = "0.26"
```

```rust name=src/main.rs
use std::collections::HashMap;
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::{Arc, RwLock};
use std::time::Duration;

use axum::{
    body::Bytes,
    extract::{
        ws::{Message, WebSocket, WebSocketUpgrade},
        FromRequestParts, Path, State,
    },
    http::{header, request::Parts, HeaderMap, StatusCode},
    response::{IntoResponse, Response},
    routing::get,
    Json, Router,
};
use futures::{stream::SplitSink, SinkExt, StreamExt};
use serde::{Deserialize, Serialize};
use tokio::sync::broadcast::{self, error::RecvError};
use tokio_tungstenite::tungstenite::{self, client::IntoClientRequest};

const ALLOWED_ORIGIN: &str = "http://localhost:5173";

// ================= errors =================

#[derive(Debug, thiserror::Error)]
enum AppError {
    #[error("authentication required")]
    Unauthorized,
    #[error("you do not have access to this resource")]
    Forbidden,
    #[error("internal server error")]
    Internal,
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
            AppError::Forbidden => (StatusCode::FORBIDDEN, "forbidden"),
            AppError::Internal => (StatusCode::INTERNAL_SERVER_ERROR, "internal"),
        };
        (status, Json(ErrorBody { error: code, message: self.to_string() })).into_response()
    }
}

// ================= protocol =================

#[derive(Debug, Clone, Serialize)]
#[serde(tag = "type", rename_all = "snake_case")]
enum ServerEvent {
    Snapshot { users: Vec<i32> },
    Joined { user_id: i32 },
    Left { user_id: i32 },
    Chat { user_id: i32, text: String },
    Error { message: String },
}

#[derive(Debug, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
enum ClientEvent {
    Chat { text: String },
}

// ================= presence =================

type RoomKey = (i32, i32); // (tenant_id, project_id)

struct Room {
    members: HashMap<u64, i32>, // conn id -> user id
    tx: broadcast::Sender<ServerEvent>,
}

struct Joined {
    snapshot: Vec<i32>,
    tx: broadcast::Sender<ServerEvent>,
    rx: broadcast::Receiver<ServerEvent>,
}

#[derive(Clone, Default)]
struct Presence(Arc<RwLock<HashMap<RoomKey, Room>>>);

impl Presence {
    fn join(&self, key: RoomKey, conn: u64, user_id: i32) -> Result<Joined, AppError> {
        let mut rooms = self.0.write().map_err(|_| AppError::Internal)?;
        let room = rooms.entry(key).or_insert_with(|| Room {
            members: HashMap::new(),
            tx: broadcast::channel(64).0,
        });

        let _ = room.tx.send(ServerEvent::Joined { user_id });
        let rx = room.tx.subscribe();
        room.members.insert(conn, user_id);

        let mut snapshot: Vec<i32> = room.members.values().copied().collect();
        snapshot.sort_unstable();
        snapshot.dedup();

        Ok(Joined { snapshot, tx: room.tx.clone(), rx })
    }

    fn leave(&self, key: RoomKey, conn: u64) {
        let Ok(mut rooms) = self.0.write() else { return };
        let mut empty = false;
        if let Some(room) = rooms.get_mut(&key) {
            if let Some(user_id) = room.members.remove(&conn) {
                let _ = room.tx.send(ServerEvent::Left { user_id });
            }
            empty = room.members.is_empty();
        }
        if empty {
            rooms.remove(&key);
        }
    }

    fn room_count(&self) -> usize {
        self.0.read().map(|r| r.len()).unwrap_or(0)
    }
}

struct PresenceGuard {
    presence: Presence,
    key: RoomKey,
    conn: u64,
}

impl Drop for PresenceGuard {
    fn drop(&mut self) {
        self.presence.leave(self.key, self.conn);
    }
}

// ================= state and auth =================

#[derive(Clone, Default)]
struct AppState {
    presence: Presence,
    next_conn: Arc<AtomicU64>,
}

#[derive(Debug, Clone)]
struct AuthUser {
    user_id: i32,
    tenant_id: i32,
}

impl FromRequestParts<AppState> for AuthUser {
    type Rejection = AppError;

    async fn from_request_parts(parts: &mut Parts, _state: &AppState) -> Result<Self, Self::Rejection> {
        // Fake token "user-<id>-tenant-<id>"; Stage 8 replaces this with JWT verification.
        let token = parts
            .headers
            .get(header::AUTHORIZATION)
            .and_then(|v| v.to_str().ok())
            .and_then(|v| v.strip_prefix("Bearer "))
            .ok_or(AppError::Unauthorized)?;
        let rest = token.strip_prefix("user-").ok_or(AppError::Unauthorized)?;
        let (user, tenant) = rest.split_once("-tenant-").ok_or(AppError::Unauthorized)?;
        Ok(AuthUser {
            user_id: user.parse().map_err(|_| AppError::Unauthorized)?,
            tenant_id: tenant.parse().map_err(|_| AppError::Unauthorized)?,
        })
    }
}

// ================= websocket =================

async fn ws_project(
    State(state): State<AppState>,
    user: AuthUser,
    Path(project_id): Path<i32>,
    headers: HeaderMap,
    ws: WebSocketUpgrade,
) -> Result<Response, AppError> {
    if let Some(origin) = headers.get(header::ORIGIN) {
        if origin.as_bytes() != ALLOWED_ORIGIN.as_bytes() {
            return Err(AppError::Forbidden);
        }
    }
    Ok(ws
        .max_message_size(64 * 1024)
        .on_upgrade(move |socket| handle_socket(socket, state, user, project_id)))
}

async fn send_event(sender: &mut SplitSink<WebSocket, Message>, event: &ServerEvent) -> bool {
    match serde_json::to_string(event) {
        Ok(json) => sender.send(Message::Text(json.into())).await.is_ok(),
        Err(_) => false,
    }
}

async fn handle_socket(socket: WebSocket, state: AppState, user: AuthUser, project_id: i32) {
    let conn = state.next_conn.fetch_add(1, Ordering::Relaxed);
    let key = (user.tenant_id, project_id);

    let Ok(Joined { snapshot, tx, mut rx }) = state.presence.join(key, conn, user.user_id) else {
        return;
    };
    let _guard = PresenceGuard { presence: state.presence.clone(), key, conn };

    let (mut sender, mut receiver) = socket.split();
    if !send_event(&mut sender, &ServerEvent::Snapshot { users: snapshot }).await {
        return;
    }

    let mut heartbeat = tokio::time::interval(Duration::from_secs(20));
    heartbeat.tick().await;

    loop {
        tokio::select! {
            incoming = receiver.next() => match incoming {
                Some(Ok(Message::Text(text))) => {
                    match serde_json::from_str::<ClientEvent>(text.as_str()) {
                        Ok(ClientEvent::Chat { text }) if !text.trim().is_empty() && text.len() <= 1000 => {
                            let _ = tx.send(ServerEvent::Chat { user_id: user.user_id, text });
                        }
                        _ => {
                            let err = ServerEvent::Error { message: "invalid message".into() };
                            if !send_event(&mut sender, &err).await {
                                break;
                            }
                        }
                    }
                }
                Some(Ok(Message::Close(_))) | Some(Err(_)) | None => break,
                Some(Ok(_)) => {}
            },
            event = rx.recv() => match event {
                Ok(ev) => {
                    if !send_event(&mut sender, &ev).await {
                        break;
                    }
                }
                Err(RecvError::Lagged(_)) => continue,
                Err(RecvError::Closed) => break,
            },
            _ = heartbeat.tick() => {
                if sender.send(Message::Ping(Bytes::new())).await.is_err() {
                    break;
                }
            }
        }
    }
    // _guard dropped here: leave() runs and "left" is broadcast
}

// ================= test clients =================

type Client = tokio_tungstenite::WebSocketStream<
    tokio_tungstenite::MaybeTlsStream<tokio::net::TcpStream>,
>;

async fn connect(
    addr: std::net::SocketAddr,
    project: i32,
    token: Option<&str>,
    origin: Option<&str>,
) -> Result<Client, String> {
    let mut req = format!("ws://{addr}/ws/projects/{project}")
        .into_client_request()
        .map_err(|e| e.to_string())?;
    if let Some(t) = token {
        let value = format!("Bearer {t}").parse().map_err(|_| "bad header".to_string())?;
        req.headers_mut().insert("authorization", value);
    }
    if let Some(o) = origin {
        let value = o.parse().map_err(|_| "bad header".to_string())?;
        req.headers_mut().insert("origin", value);
    }
    match tokio_tungstenite::connect_async(req).await {
        Ok((ws, _response)) => Ok(ws),
        Err(tungstenite::Error::Http(resp)) => {
            Err(format!("rejected with HTTP {}", resp.status().as_u16()))
        }
        Err(e) => Err(e.to_string()),
    }
}

async fn next_event(ws: &mut Client) -> String {
    loop {
        match tokio::time::timeout(Duration::from_millis(300), ws.next()).await {
            Ok(Some(Ok(tungstenite::Message::Text(t)))) => return t.to_string(),
            Ok(Some(Ok(_))) => continue, // ping/pong
            Ok(Some(Err(e))) => return format!("error: {e}"),
            Ok(None) => return "closed".into(),
            Err(_) => return "(nothing)".into(),
        }
    }
}

async fn say(ws: &mut Client, json: &str) -> Result<(), tungstenite::Error> {
    ws.send(tungstenite::Message::Text(json.into())).await
}

const ANNA: &str = "user-1-tenant-1";
const BEN: &str = "user-2-tenant-1";
const CARA: &str = "user-3-tenant-2";

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let state = AppState::default();
    let app = Router::new()
        .route("/ws/projects/{id}", get(ws_project))
        .with_state(state.clone());

    let listener = tokio::net::TcpListener::bind("127.0.0.1:0").await?;
    let addr = listener.local_addr()?;
    tokio::spawn(async move {
        let _ = axum::serve(listener, app).await;
    });

    // Rejected before the upgrade
    if let Err(e) = connect(addr, 7, None, None).await {
        println!("anonymous   -> {e}");
    }
    if let Err(e) = connect(addr, 7, Some(ANNA), Some("https://evil.example")).await {
        println!("bad origin  -> {e}");
    }

    // Presence in tenant 1, project 7
    let mut anna = connect(addr, 7, Some(ANNA), Some(ALLOWED_ORIGIN)).await?;
    println!("anna <- {}", next_event(&mut anna).await);

    let mut ben = connect(addr, 7, Some(BEN), None).await?;
    println!("ben  <- {}", next_event(&mut ben).await);
    println!("anna <- {}", next_event(&mut anna).await);

    // Same project ID, different tenant: a different room
    let mut cara = connect(addr, 7, Some(CARA), None).await?;
    println!("cara <- {}", next_event(&mut cara).await);
    println!("anna <- {}", next_event(&mut anna).await);

    // Chat is broadcast to the room
    say(&mut ben, r#"{"type":"chat","text":"hi"}"#).await?;
    println!("anna <- {}", next_event(&mut anna).await);
    println!("ben  <- {}", next_event(&mut ben).await);

    // Invalid input gets an error event, the connection stays open
    say(&mut ben, "not json").await?;
    println!("ben  <- {}", next_event(&mut ben).await);

    // Leaving: the Drop guard broadcasts "left"
    ben.close(None).await?;
    println!("anna <- {}", next_event(&mut anna).await);

    anna.close(None).await?;
    cara.close(None).await?;
    tokio::time::sleep(Duration::from_millis(200)).await;
    println!("open rooms after everyone left: {}", state.presence.room_count());
    Ok(())
}
```

Expected output:

```text
anonymous   -> rejected with HTTP 401
bad origin  -> rejected with HTTP 403
anna <- {"type":"snapshot","users":[1]}
ben  <- {"type":"snapshot","users":[1,2]}
anna <- {"type":"joined","user_id":2}
cara <- {"type":"snapshot","users":[3]}
anna <- (nothing)
anna <- {"type":"chat","user_id":2,"text":"hi"}
ben  <- {"type":"chat","user_id":2,"text":"hi"}
ben  <- {"type":"error","message":"invalid message"}
anna <- {"type":"left","user_id":2}
open rooms after everyone left: 0
```

What each part shows:

| Case | Concept |
|---|---|
| Anonymous / bad origin | All checks happen **before** the upgrade, so failures are normal HTTP 401/403 |
| Snapshots and `joined` | `join` sends the snapshot to the newcomer and `joined` to everyone else |
| Cara gets `[3]`, Anna gets nothing | Rooms are keyed by **(tenant, project)**: no cross-tenant leaks |
| Chat reaches Anna and Ben | One `broadcast` channel per room |
| `not json` | A typed protocol: invalid messages get an `error` event, not a crash |
| `left` after Ben closes | The `Drop` guard runs `leave` however the task ends |
| `open rooms: 0` | Empty rooms are removed, so memory stays bounded |

---

## 13. Exercises

These need a local project (section 12).

- [ ] **1. Echo:** build the `/ws/echo` handler from section 3. Connect from a browser console with `new WebSocket("ws://localhost:3000/ws/echo")`, send messages, and close it from both sides.
- [ ] **2. Protocol:** add a `ClientEvent::Typing { is_typing: bool }` and a matching `ServerEvent::Typing { user_id, is_typing }`. Make sure a user's typing event isn't echoed back to themselves.
- [ ] **3. Origin:** read allowed origins from config (Stage 5) as a list. Write tests for "no origin", "allowed origin", and "evil origin".
- [ ] **4. Real auth:** replace the fake `AuthUser` with the Stage 8 cookie-based extractor. Connect with `tokio-tungstenite` by adding a `Cookie` header from a login response.
- [ ] **5. Token expiry:** add a `tokio::time::sleep_until` branch to the `select!` loop that closes the socket with a Close frame when the token's `exp` is reached.
- [ ] **6. Two tabs:** open two connections for the same user. Change `leave` so `left` is only broadcast when the user's **last** connection closes, and the snapshot never lists a user twice.
- [ ] **7. Lagging clients:** set the broadcast capacity to 4, have one client stop reading, and send 20 chat messages. Handle `Lagged` by sending a fresh `Snapshot` to that client. Explain why an unbounded per-client queue would be dangerous.
- [ ] **8. Explain it:** in three or four sentences, explain why the origin check is needed even though CORS is configured, why rooms are keyed by tenant, and what changes when the service runs as several replicas.

> When you can solve exercises 1–8 without looking anything up, you're ready for **Stage 10: Streaming large data**.

---

## 14. Further reading

- [`axum::extract::ws` docs](https://docs.rs/axum/latest/axum/extract/ws/index.html): `WebSocketUpgrade`, `Message`, size limits
- [Axum examples: `websockets`](https://github.com/tokio-rs/axum/tree/main/examples/websockets) and [`chat`](https://github.com/tokio-rs/axum/tree/main/examples/chat): split sockets and broadcast rooms
- [`tokio::sync::broadcast` docs](https://docs.rs/tokio/latest/tokio/sync/broadcast/index.html): capacity and `Lagged`
- [`tokio-tungstenite` docs](https://docs.rs/tokio-tungstenite): the client used in the example
- [MDN: WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API): the browser side
- [OWASP: WebSocket Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/WebSocket_Security_Cheat_Sheet.html): origin checks, authentication, limits
- [RFC 6455](https://www.rfc-editor.org/rfc/rfc6455): the WebSocket protocol, if you want the details of frames and the handshake
