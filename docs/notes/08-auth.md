# Stage 8: Authentication and Authorization

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> If you come from JavaScript or TypeScript, the comparisons below map Rust concepts to familiar JS ideas (mostly Express with `jsonwebtoken`, `bcrypt`, and `cookie-parser`).

## Contents

1. [Authentication vs authorization](#1-authentication-vs-authorization)
2. [Storing passwords: Argon2](#2-storing-passwords-argon2)
3. [JWTs: what they are and what they aren't](#3-jwts-what-they-are-and-what-they-arent)
4. [Creating and verifying tokens with `jsonwebtoken`](#4-creating-and-verifying-tokens-with-jsonwebtoken)
5. [Where the token lives: HttpOnly cookies](#5-where-the-token-lives-httponly-cookies)
6. [The login and logout handlers](#6-the-login-and-logout-handlers)
7. [The `AuthUser` extractor](#7-the-authuser-extractor)
8. [Revoking tokens: the blocklist](#8-revoking-tokens-the-blocklist)
9. [Authorization: roles and capabilities](#9-authorization-roles-and-capabilities)
10. [Tenant scoping and the super-admin bypass](#10-tenant-scoping-and-the-super-admin-bypass)
11. [Security checklist](#11-security-checklist)
12. [How this looks in a production service](#12-how-this-looks-in-a-production-service)
13. [Putting it all together](#13-putting-it-all-together)
14. [Exercises](#14-exercises)
15. [Further reading](#15-further-reading)

---

## 1. Authentication vs authorization

| | Question | Failure | Status |
|---|---|---|---|
| **Authentication** (authn) | *Who are you?* | No token, bad token, expired, revoked | **401** Unauthorized |
| **Authorization** (authz) | *Are you allowed to do this?* | Missing capability | **403** Forbidden |
| **Tenant scoping** | *Is this yours?* | Resource belongs to another tenant | **404** Not Found (Stage 4) |

The flow for every protected request:

```text
cookie or Authorization header
   │
   ▼
AuthUser extractor ── verify signature, expiry, blocklist ──▶ 401 on failure
   │  user_id, tenant_id, role, capabilities
   ▼
handler / domain ── require capability ──▶ 403 on failure
   │
   ▼
db ── WHERE tenant_id = $n ──▶ 404 if it isn't yours
```

> **JS comparison:** in Express this is usually `passport` or a hand-written `authMiddleware` that sets `req.user`. Here, authentication is an **extractor** (Stage 3, section 8): a handler that declares `user: AuthUser` can't run without a verified user.

---

## 2. Storing passwords: Argon2

Never store passwords. Store a **slow, salted hash**, and compare by hashing the login attempt.

| Algorithm | Use for passwords? |
|---|---|
| Plain text, MD5, SHA-256 | **No.** SHA-256 is fast, so attackers can try billions of guesses per second |
| bcrypt | Acceptable, widely used |
| **Argon2id** | **Recommended.** Memory-hard, so GPU attacks are expensive |

```toml
argon2 = "0.5"
```

```rust
use argon2::{
    password_hash::{rand_core::OsRng, PasswordHash, PasswordHasher, PasswordVerifier, SaltString},
    Argon2,
};

pub fn hash_password(password: &str) -> Result<String, argon2::password_hash::Error> {
    let salt = SaltString::generate(&mut OsRng);           // random per password
    let hash = Argon2::default().hash_password(password.as_bytes(), &salt)?;
    Ok(hash.to_string())   // "$argon2id$v=19$m=19456,t=2,p=1$<salt>$<hash>"
}

pub fn verify_password(password: &str, stored: &str) -> bool {
    match PasswordHash::new(stored) {
        Ok(parsed) => Argon2::default().verify_password(password.as_bytes(), &parsed).is_ok(),
        Err(_) => false,
    }
}
```

- The stored string contains the **algorithm, parameters, and salt**, so one `TEXT` column is enough, and parameters can be raised later without breaking old hashes.
- Hashing deliberately takes tens of milliseconds of CPU. That's **blocking work**, so run it with `spawn_blocking` (Stage 1, section 8):

```rust
let ok = tokio::task::spawn_blocking(move || verify_password(&password, &stored_hash))
    .await
    .map_err(|e| AppError::Internal(Box::new(e)))?;
```

> **JS comparison:** like `bcrypt.hash` / `bcrypt.compare`, which run on libuv's thread pool for the same reason.

---

## 3. JWTs: what they are and what they aren't

A JSON Web Token is three base64url parts separated by dots:

```text
eyJhbGciOiJIUzI1NiJ9 . eyJzdWIiOjEsInRlbmFudF9pZCI6NDIsImV4cCI6MTc5...} . 3q2-7wX...
       header                          payload (claims)                       signature
```

- The **payload is readable by anyone.** It's encoded, not encrypted. Never put secrets or personal data in it.
- The **signature** proves the server created it and nobody changed it. With **HS256**, the server signs and verifies with the same secret (`JWT_SECRET_KEY`, at least 32 bytes, Stage 5).
- The server doesn't store sessions. It verifies the signature and expiry on every request. That's fast, but it also means **a JWT stays valid until it expires** unless you keep a blocklist (section 8).

### 3.1 The claims

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Claims {
    pub sub: i32,                    // subject: the user ID
    pub tenant_id: i32,
    pub role: Role,
    pub capabilities: Vec<String>,
    pub is_super_admin: bool,
    pub jti: String,                 // unique token ID, used for revocation
    pub iat: u64,                    // issued at (seconds since epoch)
    pub exp: u64,                    // expires at
}
```

Keep claims **small and stable**. If a capability changes, the old token still has the old list until it expires. Short lifetimes (minutes to an hour) limit that window.

---

## 4. Creating and verifying tokens with `jsonwebtoken`

```toml
jsonwebtoken = "9"
uuid = { version = "1", features = ["v4"] }
```

```rust
use jsonwebtoken::{decode, encode, get_current_timestamp, Algorithm, DecodingKey, EncodingKey, Header, Validation};

pub struct JwtKeys {
    encoding: EncodingKey,
    decoding: DecodingKey,
    ttl_secs: u64,
}

impl JwtKeys {
    pub fn new(secret: &[u8], ttl_secs: u64) -> Self {
        Self {
            encoding: EncodingKey::from_secret(secret),
            decoding: DecodingKey::from_secret(secret),
            ttl_secs,
        }
    }

    pub fn issue(&self, user: &UserRow) -> Result<(String, Claims), jsonwebtoken::errors::Error> {
        let now = get_current_timestamp();
        let claims = Claims {
            sub: user.id,
            tenant_id: user.tenant_id,
            role: user.role,
            capabilities: user.role.capabilities(),
            is_super_admin: user.is_super_admin,
            jti: uuid::Uuid::new_v4().to_string(),
            iat: now,
            exp: now + self.ttl_secs,
        };
        let token = encode(&Header::new(Algorithm::HS256), &claims, &self.encoding)?;
        Ok((token, claims))
    }

    pub fn verify(&self, token: &str) -> Result<Claims, jsonwebtoken::errors::Error> {
        let mut validation = Validation::new(Algorithm::HS256);  // only accept HS256
        validation.leeway = 30;                                  // seconds of clock skew
        decode::<Claims>(token, &self.decoding, &validation).map(|data| data.claims)
    }
}
```

`Validation` checks the signature, that the algorithm is HS256, and that `exp` is in the future. Pinning the algorithm matters: accepting whatever algorithm the token's header claims (including `none`) is a classic JWT vulnerability.

`JwtKeys` goes into `AppState` once, wrapped in `Arc` (Stage 5).

---

## 5. Where the token lives: HttpOnly cookies

Two common places for a browser to keep the token:

| | `localStorage` + `Authorization: Bearer` | **HttpOnly cookie** |
|---|---|---|
| Readable by JavaScript | Yes, so an XSS bug can steal it | **No** |
| Sent automatically | No, the frontend adds the header | Yes, by the browser |
| CSRF risk | None | Must be handled (`SameSite`, section 5.2) |
| Works for non-browser clients | Yes | Clients can send the cookie or use a bearer header |

Production services usually set an **HttpOnly cookie** for the browser and also accept `Authorization: Bearer` for scripts and service-to-service calls.

### 5.1 Setting the cookie with `axum-extra`

```toml
axum-extra = { version = "0.10", features = ["cookie"] }
time = "0.3"
```

```rust
use axum_extra::extract::cookie::{Cookie, CookieJar, SameSite};

pub const AUTH_COOKIE: &str = "access_token_cookie";

pub fn auth_cookie(token: String, ttl_secs: u64, secure: bool) -> Cookie<'static> {
    Cookie::build((AUTH_COOKIE, token))
        .http_only(true)                     // JavaScript can't read it
        .secure(secure)                      // HTTPS only (false only for local http)
        .same_site(SameSite::Lax)            // not sent on cross-site POSTs
        .path("/")
        .max_age(time::Duration::seconds(ttl_secs as i64))
        .build()
}
```

| Attribute | Purpose |
|---|---|
| `HttpOnly` | Blocks token theft through XSS |
| `Secure` | Never sent over plain HTTP |
| `SameSite=Lax` | Not sent on cross-site `POST`/`PUT`/`DELETE`, which blocks most CSRF |
| `Path=/` | Sent to every API path |
| `Max-Age` | The browser deletes it when the token expires |

### 5.2 CSRF in short

With cookies, the browser attaches the token to any request to your domain, including one triggered by another site. Defences, in order of importance:

1. `SameSite=Lax` (or `Strict`).
2. Only accept JSON bodies for state-changing requests. A plain HTML form can't send `Content-Type: application/json` cross-site without a CORS preflight, and CORS (Stage 7) only allows your frontend's origin.
3. For higher assurance, a CSRF token (double-submit cookie). Not needed for most JSON APIs with the two points above.

---

## 6. The login and logout handlers

### 6.1 Login

```rust
#[derive(Deserialize)]
pub struct LoginRequest {
    pub email: String,
    pub password: String,
}

pub async fn login(
    State(state): State<AppState>,
    jar: CookieJar,
    AppJson(body): AppJson<LoginRequest>,
) -> Result<(CookieJar, Json<MeDto>), AppError> {
    let user = db::users::get_by_email(&state.pool, &body.email).await?;

    // Always run the hash check, even for unknown emails, so response time
    // doesn't reveal which emails exist.
    let stored = user
        .as_ref()
        .map(|u| u.password_hash.clone())
        .unwrap_or_else(|| state.dummy_hash.to_string());
    let password = body.password;
    let ok = tokio::task::spawn_blocking(move || verify_password(&password, &stored))
        .await
        .map_err(|e| AppError::Internal(Box::new(e)))?;

    let user = match (user, ok) {
        (Some(user), true) => user,
        _ => return Err(AppError::Unauthorized),   // same answer for "no user" and "wrong password"
    };

    let (token, claims) = state.jwt.issue(&user).map_err(|e| AppError::Internal(Box::new(e)))?;
    let cookie = auth_cookie(token, state.config.token_ttl_secs, state.config.secure_cookies);

    Ok((jar.add(cookie), Json(MeDto::from(&claims))))
}
```

Points:

- **One generic error** for unknown email and wrong password. Different messages let attackers find valid accounts.
- **`dummy_hash`**: a real Argon2 hash made at startup, so unknown emails take as long as known ones.
- Returning `(CookieJar, Json<_>)` sets the `Set-Cookie` header and the body in one response.
- Login endpoints need **rate limiting** (per IP and per email). That's usually a middleware layer or handled at the load balancer.

### 6.2 Logout

```rust
pub async fn logout(
    State(state): State<AppState>,
    user: AuthUser,
    jar: CookieJar,
) -> Result<(CookieJar, StatusCode), AppError> {
    state.blocklist.revoke(&user.jti, user.exp)?;                 // section 8
    let jar = jar.remove(Cookie::build(AUTH_COOKIE).path("/"));   // path must match
    Ok((jar, StatusCode::NO_CONTENT))
}
```

Removing the cookie isn't enough: someone who copied the token could keep using it. Adding the `jti` to the blocklist makes the server reject it.

---

## 7. The `AuthUser` extractor

This is the real version of the Stage 3 preview:

```rust
#[derive(Debug, Clone)]
pub struct AuthUser {
    pub user_id: i32,
    pub tenant_id: TenantId,
    pub role: Role,
    pub capabilities: Vec<String>,
    pub is_super_admin: bool,
    pub jti: String,
    pub exp: u64,
}

impl FromRequestParts<AppState> for AuthUser {
    type Rejection = AppError;

    async fn from_request_parts(parts: &mut Parts, state: &AppState) -> Result<Self, Self::Rejection> {
        let token = token_from_parts(parts).ok_or(AppError::Unauthorized)?;

        let claims = state.jwt.verify(&token).map_err(|e| {
            tracing::debug!(error = %e, "rejected token");
            AppError::Unauthorized
        })?;

        if state.blocklist.is_revoked(&claims.jti)? {
            return Err(AppError::Unauthorized);
        }

        Ok(AuthUser::from(claims))
    }
}

/// Cookie first, then `Authorization: Bearer`.
fn token_from_parts(parts: &Parts) -> Option<String> {
    let jar = CookieJar::from_headers(&parts.headers);
    if let Some(cookie) = jar.get(AUTH_COOKIE) {
        return Some(cookie.value().to_owned());
    }
    parts
        .headers
        .get(header::AUTHORIZATION)
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .map(str::to_owned)
}
```

- Every failure becomes the same `401` with the generic message `authentication required`. The exact reason (expired, bad signature) goes to logs at `debug`, never to the client.
- Handlers just add the argument:

```rust
async fn list_projects(State(state): State<AppState>, user: AuthUser) -> Result<Json<Vec<ProjectDto>>, AppError> {
    let rows = db::projects::list(&state.pool, user.tenant_id, 20, 0).await?;
    Ok(Json(rows.into_iter().map(Into::into).collect()))
}
```

### 7.1 Optional authentication

For endpoints that work for both guests and users, implement `OptionalFromRequestParts` (Axum 0.8) so `Option<AuthUser>` is `None` when **no token** is sent, and still a 401 when an **invalid** token is sent. Or simply call `token_from_parts` yourself in that handler.

---

## 8. Revoking tokens: the blocklist

Stage 5 built a `Blocklist` behind `Arc<RwLock<..>>`. For tokens it must also forget entries once they expire, or it grows forever:

```rust
#[derive(Clone, Default)]
pub struct Blocklist(Arc<RwLock<HashMap<String, u64>>>);   // jti -> exp

impl Blocklist {
    pub fn revoke(&self, jti: &str, exp: u64) -> Result<(), AppError> {
        let mut map = self.0.write().map_err(|_| AppError::Internal("blocklist poisoned".into()))?;
        map.insert(jti.to_owned(), exp);
        Ok(())
    }

    pub fn is_revoked(&self, jti: &str) -> Result<bool, AppError> {
        let map = self.0.read().map_err(|_| AppError::Internal("blocklist poisoned".into()))?;
        Ok(map.contains_key(jti))
    }

    /// Drop entries whose token has expired anyway.
    pub fn purge_expired(&self, now: u64) -> Result<usize, AppError> {
        let mut map = self.0.write().map_err(|_| AppError::Internal("blocklist poisoned".into()))?;
        let before = map.len();
        map.retain(|_, exp| *exp > now);
        Ok(before - map.len())
    }
}
```

Start the cleanup task in `main` (Stage 1):

```rust
let blocklist = state.blocklist.clone();
tokio::spawn(async move {
    let mut tick = tokio::time::interval(Duration::from_secs(300));
    loop {
        tick.tick().await;
        if let Err(e) = blocklist.purge_expired(jsonwebtoken::get_current_timestamp()) {
            tracing::warn!(error = %e, "blocklist purge failed");
        }
    }
});
```

Limits of an in-memory blocklist:

- **Per process.** With several replicas, a logout on one isn't seen by the others, and a restart forgets every revocation. Production setups store revoked `jti`s in the database or Redis (with a TTL), or keep tokens short-lived and revoke **refresh tokens** instead.
- **Refresh tokens** (briefly): a short-lived access token (for example 15 minutes) plus a long-lived refresh token stored server-side. Revoking means deleting the refresh token row. That's a common next step once a single in-memory blocklist isn't enough.

---

## 9. Authorization: roles and capabilities

### 9.1 Roles map to capabilities

Check **capabilities** (what the user can do), not **roles** (who they are). Then adding a role doesn't mean editing every handler.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, sqlx::Type)]
#[serde(rename_all = "snake_case")]
#[sqlx(type_name = "user_role", rename_all = "snake_case")]
pub enum Role {
    Viewer,
    Editor,
    Admin,
}

pub mod caps {
    pub const PROJECTS_READ: &str = "projects:read";
    pub const PROJECTS_WRITE: &str = "projects:write";
    pub const USERS_MANAGE: &str = "users:manage";
}

impl Role {
    pub fn capabilities(self) -> Vec<String> {
        let list: &[&str] = match self {
            Role::Viewer => &[caps::PROJECTS_READ],
            Role::Editor => &[caps::PROJECTS_READ, caps::PROJECTS_WRITE],
            Role::Admin => &[caps::PROJECTS_READ, caps::PROJECTS_WRITE, caps::USERS_MANAGE],
        };
        list.iter().map(|c| (*c).to_owned()).collect()
    }
}
```

### 9.2 Checking a capability

```rust
impl AuthUser {
    pub fn require(&self, capability: &str) -> Result<(), AppError> {
        if self.is_super_admin || self.capabilities.iter().any(|c| c == capability) {
            Ok(())
        } else {
            Err(AppError::Forbidden)
        }
    }
}

async fn create_project(
    State(state): State<AppState>,
    user: AuthUser,
    ValidJson(body): ValidJson<CreateProject>,
) -> Result<(StatusCode, Json<ProjectDto>), AppError> {
    user.require(caps::PROJECTS_WRITE)?;           // 403 if missing
    let row = domain::projects::create(&state.pool, user.tenant_id, body).await?;
    Ok((StatusCode::CREATED, Json(row.into())))
}
```

Put the check **first** in the handler or in the domain function, before any reads or writes. The domain layer is the safest place for rules that several handlers share, because it's impossible to call the operation without going through the check.

---

## 10. Tenant scoping and the super-admin bypass

Stage 6 required every tenant-owned query to filter by `tenant_id`. Now it's clear where that value comes from:

- **`tenant_id` comes only from `AuthUser`**, which comes from a verified token.
- Never from a path parameter, query string, header, or body. A user must not be able to say "I'm tenant 7".

```rust
// RIGHT: tenant from the token
db::projects::get_by_id(&state.pool, user.tenant_id, id).await?

// WRONG: tenant from the client
db::projects::get_by_id(&state.pool, TenantId(query.tenant_id), id).await?
```

### 10.1 The super-admin bypass

Some operators must work across tenants. Make that path **explicit and narrow**:

```rust
pub fn require_super_admin(user: &AuthUser) -> Result<(), AppError> {
    if user.is_super_admin { Ok(()) } else { Err(AppError::Forbidden) }
}

async fn admin_list_tenant_projects(
    State(state): State<AppState>,
    user: AuthUser,
    Path(tenant_id): Path<TenantId>,      // allowed here only because of the check below
) -> Result<Json<Vec<ProjectDto>>, AppError> {
    require_super_admin(&user)?;
    tracing::info!(admin = user.user_id, tenant = tenant_id.0, "cross-tenant read");
    let rows = db::projects::list(&state.pool, tenant_id, 100, 0).await?;
    Ok(Json(rows.into_iter().map(Into::into).collect()))
}
```

- Cross-tenant routes live under a separate `/admin` prefix, so they're easy to find and review.
- They're **logged**, because cross-tenant access is exactly what an audit wants to see.
- A normal user calling them gets **403** (they're asking for an admin feature, not a specific resource), while a normal user reaching another tenant's project through a normal route gets **404**.

---

## 11. Security checklist

| Topic | Rule |
|---|---|
| Passwords | Argon2id, in `spawn_blocking`, never logged |
| Login errors | One generic 401, constant-time-ish with a dummy hash, rate limited |
| JWT secret | At least 32 random bytes, from env, a `Secret` type, never logged |
| JWT validation | Pin the algorithm, check `exp`, small leeway |
| Token lifetime | Short (≤ 1 hour); revocation via `jti` |
| Cookie | `HttpOnly`, `Secure` (in production), `SameSite=Lax`, `Path=/` |
| CSRF | `SameSite`, JSON-only writes, strict CORS origins with credentials |
| Claims | No secrets or personal data; they're readable |
| Rejections | 401 with a generic message; details only in `debug` logs |
| Authorization | Capabilities, checked before any work |
| Tenancy | `tenant_id` only from the token; every query scoped; cross-tenant = 404 |
| Super admin | Separate routes, explicit check, audit log |

---

## 12. How this looks in a production service

```text
crates/
├── shared/src/
│   ├── types/auth.rs       # LoginRequest, MeDto, Role, caps
│   └── config.rs           # jwt_secret: Secret, token_ttl, secure_cookies
├── domain/src/auth/
│   ├── password.rs         # hash_password, verify_password
│   ├── jwt.rs              # Claims, JwtKeys::issue / verify
│   └── authz.rs            # require(capability), require_super_admin
├── db/src/users.rs         # UserRow { password_hash, role, tenant_id, ... }
└── api/src/
    ├── extractors/auth.rs  # AuthUser: FromRequestParts<AppState>
    ├── state.rs            # AppState { pool, config, jwt: Arc<JwtKeys>, blocklist }
    └── routes/auth.rs      # POST /auth/login, POST /auth/logout, GET /auth/me
```

Conventions:

- **Authentication** happens in exactly one place: the `AuthUser` extractor.
- **Authorization** is a capability check at the start of the operation.
- **Tenancy** is enforced by repository function signatures that require a `TenantId`.
- The protected router can also carry a `route_layer` that requires `AuthUser` (Stage 7, section 10) as a safety net.

---

## 13. Putting it all together

A complete auth flow: Argon2 password hashes, login that sets an HttpOnly cookie, an `AuthUser` extractor that reads the cookie or a bearer header, capability checks, tenant-scoped data, a super-admin route, and logout with a blocklist. The "database" is an in-memory list so it runs without Postgres. It calls itself with `oneshot`, so it runs as a **local project** (the Playground may not include `argon2`, `jsonwebtoken`, and `axum-extra`).

```toml name=Cargo.toml
[package]
name = "auth-demo"
version = "0.1.0"
edition = "2021"

[dependencies]
argon2 = "0.5"
axum = "0.8"
axum-extra = { version = "0.10", features = ["cookie"] }
jsonwebtoken = "9"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "2"
time = "0.3"
tokio = { version = "1", features = ["full"] }
tower = { version = "0.5", features = ["util"] }
uuid = { version = "1", features = ["v4"] }
```

```rust name=src/main.rs
use std::collections::HashMap;
use std::sync::{Arc, RwLock};

use argon2::{
    password_hash::{rand_core::OsRng, PasswordHash, PasswordHasher, PasswordVerifier, SaltString},
    Argon2,
};
use axum::{
    body::{to_bytes, Body},
    extract::{FromRequestParts, Path, Request, State},
    http::{header, request::Parts, StatusCode},
    response::{IntoResponse, Response},
    routing::{get, post},
    Json, Router,
};
use axum_extra::extract::cookie::{Cookie, CookieJar, SameSite};
use jsonwebtoken::{decode, encode, get_current_timestamp, Algorithm, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use tower::ServiceExt;

// ================= errors =================

#[derive(Debug, thiserror::Error)]
enum AppError {
    #[error("{0} not found")]
    NotFound(String),
    #[error("authentication required")]
    Unauthorized,
    #[error("you do not have access to this resource")]
    Forbidden,
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
            AppError::NotFound(_) => (StatusCode::NOT_FOUND, "not_found"),
            AppError::Unauthorized => (StatusCode::UNAUTHORIZED, "unauthorized"),
            AppError::Forbidden => (StatusCode::FORBIDDEN, "forbidden"),
            AppError::Internal(source) => {
                eprintln!("[log] internal error: {source}");
                (StatusCode::INTERNAL_SERVER_ERROR, "internal")
            }
        };
        (status, Json(ErrorBody { error: code, message: self.to_string() })).into_response()
    }
}

fn internal<E: std::error::Error + Send + Sync + 'static>(e: E) -> AppError {
    AppError::Internal(Box::new(e))
}

// ================= passwords =================

fn hash_password(password: &str) -> Result<String, AppError> {
    let salt = SaltString::generate(&mut OsRng);
    Argon2::default()
        .hash_password(password.as_bytes(), &salt)
        .map(|h| h.to_string())
        .map_err(|e| AppError::Internal(e.to_string().into()))
}

fn verify_password(password: &str, stored: &str) -> bool {
    match PasswordHash::new(stored) {
        Ok(parsed) => Argon2::default().verify_password(password.as_bytes(), &parsed).is_ok(),
        Err(_) => false,
    }
}

// ================= users and capabilities =================

#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
#[serde(rename_all = "snake_case")]
enum Role {
    Viewer,
    Editor,
}

mod caps {
    pub const PROJECTS_READ: &str = "projects:read";
    pub const PROJECTS_WRITE: &str = "projects:write";
}

impl Role {
    fn capabilities(self) -> Vec<String> {
        let list: &[&str] = match self {
            Role::Viewer => &[caps::PROJECTS_READ],
            Role::Editor => &[caps::PROJECTS_READ, caps::PROJECTS_WRITE],
        };
        list.iter().map(|c| (*c).to_owned()).collect()
    }
}

#[derive(Clone)]
struct UserRow {
    id: i32,
    email: &'static str,
    password_hash: String,
    tenant_id: i32,
    role: Role,
    is_super_admin: bool,
}

#[derive(Clone, Serialize)]
struct Project {
    id: i32,
    tenant_id: i32,
    name: &'static str,
}

// ================= jwt =================

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Claims {
    sub: i32,
    tenant_id: i32,
    role: Role,
    capabilities: Vec<String>,
    is_super_admin: bool,
    jti: String,
    iat: u64,
    exp: u64,
}

struct JwtKeys {
    encoding: EncodingKey,
    decoding: DecodingKey,
    ttl_secs: u64,
}

impl JwtKeys {
    fn new(secret: &[u8], ttl_secs: u64) -> Self {
        Self {
            encoding: EncodingKey::from_secret(secret),
            decoding: DecodingKey::from_secret(secret),
            ttl_secs,
        }
    }

    fn issue(&self, user: &UserRow) -> Result<(String, Claims), AppError> {
        let now = get_current_timestamp();
        let claims = Claims {
            sub: user.id,
            tenant_id: user.tenant_id,
            role: user.role,
            capabilities: user.role.capabilities(),
            is_super_admin: user.is_super_admin,
            jti: uuid::Uuid::new_v4().to_string(),
            iat: now,
            exp: now + self.ttl_secs,
        };
        let token = encode(&Header::new(Algorithm::HS256), &claims, &self.encoding).map_err(internal)?;
        Ok((token, claims))
    }

    fn verify(&self, token: &str) -> Result<Claims, jsonwebtoken::errors::Error> {
        let mut validation = Validation::new(Algorithm::HS256);
        validation.leeway = 30;
        decode::<Claims>(token, &self.decoding, &validation).map(|d| d.claims)
    }
}

// ================= blocklist =================

#[derive(Clone, Default)]
struct Blocklist(Arc<RwLock<HashMap<String, u64>>>);

impl Blocklist {
    fn revoke(&self, jti: &str, exp: u64) -> Result<(), AppError> {
        let mut map = self.0.write().map_err(|_| AppError::Internal("blocklist poisoned".into()))?;
        map.insert(jti.to_owned(), exp);
        Ok(())
    }

    fn is_revoked(&self, jti: &str) -> Result<bool, AppError> {
        let map = self.0.read().map_err(|_| AppError::Internal("blocklist poisoned".into()))?;
        Ok(map.contains_key(jti))
    }
}

// ================= state =================

#[derive(Clone)]
struct AppState {
    users: Arc<Vec<UserRow>>,
    projects: Arc<Vec<Project>>,
    jwt: Arc<JwtKeys>,
    blocklist: Blocklist,
    dummy_hash: Arc<str>,
    secure_cookies: bool,
}

// ================= AuthUser extractor =================

const AUTH_COOKIE: &str = "access_token_cookie";

#[derive(Debug, Clone)]
struct AuthUser {
    user_id: i32,
    tenant_id: i32,
    capabilities: Vec<String>,
    is_super_admin: bool,
    jti: String,
    exp: u64,
}

impl AuthUser {
    fn require(&self, capability: &str) -> Result<(), AppError> {
        if self.is_super_admin || self.capabilities.iter().any(|c| c == capability) {
            Ok(())
        } else {
            Err(AppError::Forbidden)
        }
    }
}

fn token_from_parts(parts: &Parts) -> Option<String> {
    let jar = CookieJar::from_headers(&parts.headers);
    if let Some(cookie) = jar.get(AUTH_COOKIE) {
        return Some(cookie.value().to_owned());
    }
    parts
        .headers
        .get(header::AUTHORIZATION)
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .map(str::to_owned)
}

impl FromRequestParts<AppState> for AuthUser {
    type Rejection = AppError;

    async fn from_request_parts(parts: &mut Parts, state: &AppState) -> Result<Self, Self::Rejection> {
        let token = token_from_parts(parts).ok_or(AppError::Unauthorized)?;
        let claims = state.jwt.verify(&token).map_err(|_| AppError::Unauthorized)?;
        if state.blocklist.is_revoked(&claims.jti)? {
            return Err(AppError::Unauthorized);
        }
        Ok(AuthUser {
            user_id: claims.sub,
            tenant_id: claims.tenant_id,
            capabilities: claims.capabilities,
            is_super_admin: claims.is_super_admin,
            jti: claims.jti,
            exp: claims.exp,
        })
    }
}

// ================= handlers =================

#[derive(Deserialize)]
struct LoginRequest {
    email: String,
    password: String,
}

#[derive(Serialize)]
struct MeDto {
    user_id: i32,
    tenant_id: i32,
    capabilities: Vec<String>,
}

async fn login(
    State(state): State<AppState>,
    jar: CookieJar,
    Json(body): Json<LoginRequest>,
) -> Result<(CookieJar, Json<MeDto>), AppError> {
    let user = state.users.iter().find(|u| u.email == body.email).cloned();

    let stored = user
        .as_ref()
        .map(|u| u.password_hash.clone())
        .unwrap_or_else(|| state.dummy_hash.to_string());
    let password = body.password;
    let ok = tokio::task::spawn_blocking(move || verify_password(&password, &stored))
        .await
        .map_err(internal)?;

    let user = match (user, ok) {
        (Some(u), true) => u,
        _ => return Err(AppError::Unauthorized),
    };

    let (token, claims) = state.jwt.issue(&user)?;
    let cookie = Cookie::build((AUTH_COOKIE, token))
        .http_only(true)
        .secure(state.secure_cookies)
        .same_site(SameSite::Lax)
        .path("/")
        .max_age(time::Duration::seconds(state.jwt.ttl_secs as i64))
        .build();

    let me = MeDto { user_id: claims.sub, tenant_id: claims.tenant_id, capabilities: claims.capabilities };
    Ok((jar.add(cookie), Json(me)))
}

async fn logout(
    State(state): State<AppState>,
    user: AuthUser,
    jar: CookieJar,
) -> Result<(CookieJar, StatusCode), AppError> {
    state.blocklist.revoke(&user.jti, user.exp)?;
    Ok((jar.remove(Cookie::build(AUTH_COOKIE).path("/")), StatusCode::NO_CONTENT))
}

async fn me(user: AuthUser) -> Json<MeDto> {
    Json(MeDto { user_id: user.user_id, tenant_id: user.tenant_id, capabilities: user.capabilities })
}

async fn get_project(
    State(state): State<AppState>,
    user: AuthUser,
    Path(id): Path<i32>,
) -> Result<Json<Project>, AppError> {
    user.require(caps::PROJECTS_READ)?;
    state
        .projects
        .iter()
        .find(|p| p.id == id && p.tenant_id == user.tenant_id)   // tenant-scoped
        .cloned()
        .map(Json)
        .ok_or_else(|| AppError::NotFound(format!("project {id}")))
}

async fn create_project(user: AuthUser) -> Result<StatusCode, AppError> {
    user.require(caps::PROJECTS_WRITE)?;
    Ok(StatusCode::CREATED)
}

async fn admin_tenant_projects(
    State(state): State<AppState>,
    user: AuthUser,
    Path(tenant_id): Path<i32>,
) -> Result<Json<Vec<Project>>, AppError> {
    if !user.is_super_admin {
        return Err(AppError::Forbidden);
    }
    println!("[audit] user {} read tenant {tenant_id}", user.user_id);
    let list = state.projects.iter().filter(|p| p.tenant_id == tenant_id).cloned().collect();
    Ok(Json(list))
}

fn app(state: AppState) -> Router {
    Router::new()
        .route("/auth/login", post(login))
        .route("/auth/logout", post(logout))
        .route("/auth/me", get(me))
        .route("/projects", post(create_project))
        .route("/projects/{id}", get(get_project))
        .route("/admin/tenants/{tenant_id}/projects", get(admin_tenant_projects))
        .with_state(state)
}

// ================= tiny client =================

struct Reply {
    status: StatusCode,
    cookie: Option<String>,
    body: String,
}

async fn call(app: &Router, method: &str, uri: &str, cookie: Option<&str>, json: Option<&str>) -> Reply {
    let mut builder = Request::builder().method(method).uri(uri);
    if let Some(c) = cookie {
        builder = builder.header(header::COOKIE, c);
    }
    let body = match json {
        Some(j) => {
            builder = builder.header(header::CONTENT_TYPE, "application/json");
            Body::from(j.to_owned())
        }
        None => Body::empty(),
    };
    let request = builder.body(body).expect("static request is valid");

    let response = match app.clone().oneshot(request).await {
        Ok(r) => r,
        Err(never) => match never {},
    };
    let status = response.status();
    // Keep only "name=value" from Set-Cookie, like a browser would send it back
    let cookie = response
        .headers()
        .get(header::SET_COOKIE)
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.split(';').next())
        .map(str::to_owned);
    let bytes = to_bytes(response.into_body(), usize::MAX).await.unwrap_or_default();
    Reply { status, cookie, body: String::from_utf8_lossy(&bytes).into_owned() }
}

fn show(label: &str, r: &Reply) {
    println!("{label:<34} -> {} {}", r.status.as_u16(), r.body);
}

#[tokio::main]
async fn main() -> Result<(), AppError> {
    let users = vec![
        UserRow { id: 1, email: "viewer@a.test", password_hash: hash_password("viewer-pass")?, tenant_id: 1, role: Role::Viewer, is_super_admin: false },
        UserRow { id: 2, email: "editor@a.test", password_hash: hash_password("editor-pass")?, tenant_id: 1, role: Role::Editor, is_super_admin: false },
        UserRow { id: 3, email: "root@ops.test", password_hash: hash_password("root-pass")?, tenant_id: 0, role: Role::Viewer, is_super_admin: true },
    ];
    let projects = vec![
        Project { id: 1, tenant_id: 1, name: "Alpha" },
        Project { id: 2, tenant_id: 2, name: "Other tenant's project" },
    ];

    let state = AppState {
        users: Arc::new(users),
        projects: Arc::new(projects),
        jwt: Arc::new(JwtKeys::new(b"a-local-demo-secret-of-at-least-32-bytes!", 3600)),
        blocklist: Blocklist::default(),
        dummy_hash: Arc::from(hash_password("dummy-password")?),
        secure_cookies: false, // local http only; true in production
    };
    let app = app(state);

    // 1. Authentication
    show("GET  /auth/me (no cookie)", &call(&app, "GET", "/auth/me", None, None).await);
    show("POST /auth/login (wrong password)", &call(&app, "POST", "/auth/login", None,
        Some(r#"{"email":"viewer@a.test","password":"nope"}"#)).await);
    show("POST /auth/login (unknown email)", &call(&app, "POST", "/auth/login", None,
        Some(r#"{"email":"ghost@a.test","password":"nope"}"#)).await);

    let login = call(&app, "POST", "/auth/login", None,
        Some(r#"{"email":"viewer@a.test","password":"viewer-pass"}"#)).await;
    show("POST /auth/login (viewer)", &login);
    let viewer = login.cookie.clone();
    show("GET  /auth/me (viewer)", &call(&app, "GET", "/auth/me", viewer.as_deref(), None).await);

    // 2. Authorization and tenancy
    show("GET  /projects/1 (own tenant)", &call(&app, "GET", "/projects/1", viewer.as_deref(), None).await);
    show("GET  /projects/2 (other tenant)", &call(&app, "GET", "/projects/2", viewer.as_deref(), None).await);
    show("POST /projects (viewer)", &call(&app, "POST", "/projects", viewer.as_deref(), None).await);

    let editor = call(&app, "POST", "/auth/login", None,
        Some(r#"{"email":"editor@a.test","password":"editor-pass"}"#)).await.cookie;
    show("POST /projects (editor)", &call(&app, "POST", "/projects", editor.as_deref(), None).await);

    // 3. Super admin
    show("GET  /admin/.../2 (viewer)", &call(&app, "GET", "/admin/tenants/2/projects", viewer.as_deref(), None).await);
    let root = call(&app, "POST", "/auth/login", None,
        Some(r#"{"email":"root@ops.test","password":"root-pass"}"#)).await.cookie;
    show("GET  /admin/.../2 (super admin)", &call(&app, "GET", "/admin/tenants/2/projects", root.as_deref(), None).await);

    // 4. Logout revokes the token, even if the old cookie is replayed
    show("POST /auth/logout (viewer)", &call(&app, "POST", "/auth/logout", viewer.as_deref(), None).await);
    show("GET  /auth/me (old viewer cookie)", &call(&app, "GET", "/auth/me", viewer.as_deref(), None).await);

    // 5. A tampered token
    let tampered = viewer.map(|c| format!("{c}x"));
    show("GET  /auth/me (tampered token)", &call(&app, "GET", "/auth/me", tampered.as_deref(), None).await);

    Ok(())
}
```

Expected output:

```text
GET  /auth/me (no cookie)          -> 401 {"error":"unauthorized","message":"authentication required"}
POST /auth/login (wrong password)  -> 401 {"error":"unauthorized","message":"authentication required"}
POST /auth/login (unknown email)   -> 401 {"error":"unauthorized","message":"authentication required"}
POST /auth/login (viewer)          -> 200 {"user_id":1,"tenant_id":1,"capabilities":["projects:read"]}
GET  /auth/me (viewer)             -> 200 {"user_id":1,"tenant_id":1,"capabilities":["projects:read"]}
GET  /projects/1 (own tenant)      -> 200 {"id":1,"tenant_id":1,"name":"Alpha"}
GET  /projects/2 (other tenant)    -> 404 {"error":"not_found","message":"project 2 not found"}
POST /projects (viewer)            -> 403 {"error":"forbidden","message":"you do not have access to this resource"}
POST /projects (editor)            -> 201 
GET  /admin/.../2 (viewer)         -> 403 {"error":"forbidden","message":"you do not have access to this resource"}
[audit] user 3 read tenant 2
GET  /admin/.../2 (super admin)    -> 200 [{"id":2,"tenant_id":2,"name":"Other tenant's project"}]
POST /auth/logout (viewer)         -> 204 
GET  /auth/me (old viewer cookie)  -> 401 {"error":"unauthorized","message":"authentication required"}
GET  /auth/me (tampered token)     -> 401 {"error":"unauthorized","message":"authentication required"}
```

What each part shows:

| Case | Concept |
|---|---|
| Wrong password vs unknown email | Same 401, same message, both run Argon2 (dummy hash) |
| Login | `Set-Cookie: access_token_cookie=...; HttpOnly; SameSite=Lax; Path=/; Max-Age=3600` |
| `/auth/me` with the cookie | `AuthUser` reads the cookie, verifies the JWT, builds the user |
| Project 2 for a tenant-1 user | Tenant scoping → **404**, not 403 |
| Viewer creates a project | Capability missing → **403** |
| Admin route | Explicit super-admin check, audit log line |
| Old cookie after logout | Valid signature and expiry, but `jti` is on the blocklist → 401 |
| Tampered token | Signature check fails → 401 |

Running in debug mode, Argon2 is slow (a second or more per hash). Use `cargo run --release` if startup feels long.

---

## 14. Exercises

These need a local project (section 13).

- [ ] **1. Hashing:** hash the same password twice and print both hashes. Explain why they differ and how `verify_password` still works. Time one hash in debug and release builds.
- [ ] **2. Read a JWT:** log in, copy the token from the `Set-Cookie` header, and decode its payload with base64 (or paste it into a JWT debugger). Write down which claims anyone can read, and why there must be no secrets in them.
- [ ] **3. Expiry:** issue a token with `ttl_secs = 2`, wait 3 seconds plus the leeway, and confirm `/auth/me` returns 401. Then set `leeway = 0` and repeat.
- [ ] **4. Bearer fallback:** call `/auth/me` with `Authorization: Bearer <token>` instead of the cookie. Then send both a valid cookie and an invalid bearer header and decide (and document) which one should win.
- [ ] **5. Blocklist cleanup:** add `purge_expired(now)` and a background task with `tokio::time::interval`. Prove with a test that expired entries are removed and live ones stay.
- [ ] **6. Capabilities:** add an `Admin` role with `users:manage` and a `DELETE /users/{id}` route that requires it. Check that viewers and editors get 403, and that an admin can only delete users in **their own** tenant (404 otherwise).
- [ ] **7. Optional auth:** implement `OptionalFromRequestParts` for `AuthUser` so `GET /public/greeting` returns "hello guest" without a token, "hello user N" with a valid one, and 401 with an invalid one.
- [ ] **8. Explain it:** in three or four sentences, explain why the token goes in an HttpOnly cookie instead of `localStorage`, what `SameSite=Lax` protects against, and why an in-memory blocklist isn't enough for a service with several replicas.

> When you can solve exercises 1–8 without looking anything up, you're ready for **Stage 9: Realtime with WebSockets**.

---

## 15. Further reading

- [OWASP: Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html): Argon2id parameters and why
- [OWASP: Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html): generic errors, rate limiting, account enumeration
- [OWASP: CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html): `SameSite`, tokens, custom headers
- [`jsonwebtoken` docs](https://docs.rs/jsonwebtoken): `Validation`, algorithms, custom claims
- [`argon2` docs](https://docs.rs/argon2) and [`axum-extra` cookie docs](https://docs.rs/axum-extra/latest/axum_extra/extract/cookie/index.html)
- [Axum example: `jwt`](https://github.com/tokio-rs/axum/tree/main/examples/jwt): a minimal JWT extractor
- [MDN: `Set-Cookie`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie): every cookie attribute
- [RFC 8725: JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725): algorithm pinning and other pitfalls
