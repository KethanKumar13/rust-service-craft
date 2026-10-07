# Stage 0: Rust Prerequisites for Axum

> Part of **rust-service-craft**: notes on building production-style backend services in Rust.
> If you come from JavaScript or TypeScript, the comparisons below map Rust concepts to familiar JS ideas.

## Contents

1. [Ownership, borrowing, and lifetimes](#1-ownership-borrowing-and-lifetimes)
2. [`struct`, `enum`, `impl`, and `match`](#2-struct-enum-impl-and-match)
3. [`Option<T>`, `Result<T, E>`, and `?`](#3-optiont-resultt-e-and-)
4. [Traits and generics](#4-traits-and-generics-the-key-topic-for-axum)
5. [Derive macros and serde](#5-derive-macros-and-serde)
6. [Smart pointers and sharing](#6-smart-pointers-and-sharing-box-rc-arc-mutex)
7. [Putting it all together](#7-putting-it-all-together)
8. [Exercises](#8-exercises)
9. [Further reading](#9-further-reading)

---

## 1. Ownership, borrowing, and lifetimes

### 1.1 The problem Rust solves

In JavaScript, a garbage collector frees memory for you. In C, you free it yourself and can get it wrong. Rust has neither. It uses **ownership rules** that the compiler checks before the program ever runs.

### 1.2 The three ownership rules

1. Every value has exactly **one owner**, which is a variable.
2. There can be only one owner at a time.
3. When the owner goes out of scope, the value is **dropped** (freed).

```rust
fn main() {
    let name = String::from("Anna"); // `name` owns the String
    println!("{name}");
} // `name` goes out of scope here, so the String is freed automatically
```

### 1.3 Move

Assigning a heap value to another variable **moves** ownership. The old variable can no longer be used.

```rust
let a = String::from("hello");
let b = a;              // ownership moves from a to b
// println!("{a}");     // ERROR: value borrowed after move
println!("{b}");        // OK
```

Passing a value to a function moves it too:

```rust
fn consume(s: String) {
    println!("{s}");
} // s is dropped here

let msg = String::from("hi");
consume(msg);
// println!("{msg}"); // ERROR: msg was moved into consume()
```

> **JS comparison:** in JS, `let b = a` makes both variables point to the same object. In Rust, `a` becomes unusable after the move.

### 1.4 Copy types

Small, simple types such as `i32`, `u32`, `f64`, `bool`, `char`, and tuples of these are **copied** instead of moved:

```rust
let x = 5;
let y = x;     // a copy, so both are usable
println!("{x} {y}");
```

### 1.5 Clone

To get a real, independent copy of a heap value, call `.clone()` explicitly:

```rust
let a = String::from("hello");
let b = a.clone();  // a deep copy, so both are usable
println!("{a} {b}");
```

### 1.6 Borrowing with references (`&`)

Usually you only want to *use* a value, not own it. To do that, you **borrow** it with `&`:

```rust
fn print_len(s: &String) {      // borrows, does not own
    println!("{}", s.len());
}

let msg = String::from("hello");
print_len(&msg);                // lend it
println!("{msg}");              // still usable: we only lent it
```

### 1.7 Mutable borrowing (`&mut`)

```rust
fn add_world(s: &mut String) {
    s.push_str(" world");
}

let mut msg = String::from("hello");  // the variable must be `mut`
add_world(&mut msg);
println!("{msg}"); // "hello world"
```

### 1.8 The borrowing rule (the most important rule)

At any moment you can have **either**:
- **many** immutable borrows (`&T`), **or**
- **exactly one** mutable borrow (`&mut T`)

but never both at the same time.

```rust
let mut s = String::from("x");
let r1 = &s;
let r2 = &s;          // OK: many readers
// let r3 = &mut s;   // ERROR: cannot borrow as mutable while it is borrowed as immutable
println!("{r1} {r2}");
let r3 = &mut s;      // OK now: r1 and r2 are no longer used
r3.push('y');
```

This rule is why safe Rust has no data races.

### 1.9 `String` vs `&str`

| Type | Meaning | Example |
|---|---|---|
| `String` | Owned, growable text on the heap | `String::from("hi")`, `"hi".to_string()` |
| `&str` | A borrowed view of text | `"hi"` (literal), `&my_string` |

Rule of thumb: **function parameters take `&str`, struct fields hold `String`.**

```rust
fn greet(name: &str) -> String {
    format!("Hello, {name}")
}

let owned = String::from("Anna");
greet(&owned);   // a &String converts to &str automatically
greet("Bob");    // a literal is already &str
```

The same idea applies to `Vec<T>` (owned) and `&[T]` (a borrowed slice).

### 1.10 Lifetimes (basic level)

A lifetime tells the compiler how long a reference is valid. Most of the time Rust **infers** it. You only write one when the compiler can't work it out:

```rust
// Which input does the returned reference come from? Rust needs to know.
fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() > b.len() { a } else { b }
}
```

`'a` means "the returned reference is valid only as long as the *shorter-lived* of the two inputs."

Lifetimes also prevent dangling references:

```rust
// fn bad() -> &String {
//     let s = String::from("x");
//     &s          // ERROR: s is dropped at the end of the function
// }
fn good() -> String {
    String::from("x")   // return the owned value instead
}
```

**What you need for Axum:** very little. Handlers take **owned** values (`String`, `Json<T>`), and Axum requires `'static` data in many places. You'll mostly see `&'static str`, meaning text that lives for the whole program, such as a string literal.

---

## 2. `struct`, `enum`, `impl`, and `match`

### 2.1 Struct

A struct is like a JS object with a fixed shape:

```rust
struct User {
    id: i32,
    name: String,
    active: bool,
}

let u = User { id: 1, name: "Anna".to_string(), active: true };
println!("{}", u.name);

let mut u2 = User { id: 2, name: "Bob".into(), active: false };
u2.active = true;   // the whole binding must be `mut` to change a field
```

Struct update syntax, similar to the JS spread operator:

```rust
let u3 = User { id: 3, ..u2 };  // takes the remaining fields from u2 (and moves `name` out of u2)
```

**Newtypes** (tuple structs) make IDs type-safe. Production codebases use this pattern a lot:

```rust
struct ProjectId(i32);
struct CustomerId(i32);

fn load_project(id: ProjectId) { /* ... */ }

load_project(ProjectId(5));
// load_project(CustomerId(5)); // ERROR: wrong type, so the IDs cannot be mixed up
```

### 2.2 `impl`: methods on a type

```rust
impl User {
    // An associated function (like a static method), called as User::new(...)
    fn new(id: i32, name: &str) -> Self {
        Self { id, name: name.to_string(), active: true }
    }

    // &self: read-only access
    fn display_name(&self) -> String {
        format!("#{} {}", self.id, self.name)
    }

    // &mut self: can modify
    fn deactivate(&mut self) {
        self.active = false;
    }

    // self: consumes the value
    fn into_name(self) -> String {
        self.name
    }
}

let mut u = User::new(1, "Anna");
println!("{}", u.display_name());
u.deactivate();
let name = u.into_name();  // u cannot be used after this
```

### 2.3 Enum

Rust enums are much more powerful than JS or TS enums, because each variant can **carry data**:

```rust
enum ProjectStatus {
    Draft,
    InProgress,
    Completed,
    Live,
}

enum AppError {
    NotFound,
    BadRequest(String),                      // carries a message
    Conflict { expected: i32, actual: i32 }, // carries named fields
}
```

### 2.4 `match`

`match` is like `switch`, but the compiler checks that it is **exhaustive**: you must handle every case.

```rust
fn status_code(err: &AppError) -> u16 {
    match err {
        AppError::NotFound => 404,
        AppError::BadRequest(msg) => {
            println!("bad request: {msg}");
            400
        }
        AppError::Conflict { expected, actual } => {
            println!("version {expected} vs {actual}");
            409
        }
    }
}
```

If you later add `AppError::Forbidden`, every `match` that doesn't handle it becomes a compile error. That is a strong safety net.

Other `match` patterns:

```rust
let n = 7;
match n {
    0 => println!("zero"),
    1..=5 => println!("small"),
    x if x % 2 == 0 => println!("even"),
    _ => println!("other"),   // `_` matches anything (the default case)
}
```

`if let` is a shortcut when you care about only one case:

```rust
if let AppError::BadRequest(msg) = &err {
    println!("{msg}");
}
```

---

## 3. `Option<T>`, `Result<T, E>`, and `?`

Rust has **no `null`** and **no exceptions**. These two enums replace them.

### 3.1 `Option<T>`: a value may be missing

```rust
enum Option<T> {   // already built in
    Some(T),
    None,
}
```

```rust
fn find_user(id: i32) -> Option<String> {
    if id == 1 { Some("Anna".into()) } else { None }
}

match find_user(1) {
    Some(name) => println!("found {name}"),
    None => println!("not found"),
}

// Common helpers
let name = find_user(2).unwrap_or("guest".to_string());
let len  = find_user(1).map(|n| n.len());   // Option<usize>
```

> **JS comparison:** `Option<String>` is like `string | null`, but the compiler forces you to handle the `null` case.

### 3.2 `Result<T, E>`: an operation may fail

```rust
enum Result<T, E> {   // already built in
    Ok(T),
    Err(E),
}
```

```rust
fn parse_age(input: &str) -> Result<u32, String> {
    match input.parse::<u32>() {
        Ok(n) => Ok(n),
        Err(_) => Err(format!("'{input}' is not a number")),
    }
}

match parse_age("42") {
    Ok(age) => println!("age {age}"),
    Err(e) => println!("error: {e}"),
}
```

### 3.3 The `?` operator

`?` means "if this is an `Err` (or `None`), **return it right now**; otherwise unwrap the value."

```rust
fn total_age(a: &str, b: &str) -> Result<u32, String> {
    let x = parse_age(a)?;   // on Err, return early
    let y = parse_age(b)?;
    Ok(x + y)
}
```

Without `?`, the same function would look like this:

```rust
let x = match parse_age(a) {
    Ok(v) => v,
    Err(e) => return Err(e),
};
```

> **JS comparison:** `?` behaves like an automatic `throw` that travels up only one function, and the function's return type tells everyone it can fail.

> **Note:** `?` only works inside a function that itself returns `Result` (or `Option`). It won't compile in a plain `fn main()` unless `main` returns `Result`.

### 3.4 Converting between `Option` and `Result`

```rust
fn get_user(id: i32) -> Result<String, String> {
    find_user(id).ok_or_else(|| format!("user {id} not found"))
}
```

`ok_or_else` is very common in real backend code.

### 3.5 Converting error types with `map_err`

```rust
fn load(id: &str) -> Result<u32, AppError> {
    let n = id.parse::<u32>()
        .map_err(|_| AppError::BadRequest("invalid id".into()))?;
    Ok(n)
}
```

### 3.6 `unwrap` and `expect`

```rust
let n = "5".parse::<i32>().unwrap();          // panics (crashes) on Err
let n = "5".parse::<i32>().expect("bad num"); // panics with a message
```

Use them only in tests and quick experiments. **Production services usually forbid them in runtime code**, often enforced with clippy lints (`clippy::unwrap_used`, `clippy::expect_used`). Use `?`, `ok_or_else`, or `map_err` instead.

### 3.7 How this looks in Axum

```rust
async fn get_project(Path(id): Path<i32>) -> Result<Json<Project>, AppError> {
    let project = db_find(id).await?.ok_or(AppError::NotFound)?;
    Ok(Json(project))
}
```

Almost every handler follows this pattern.

---

## 4. Traits and generics (the key topic for Axum)

### 4.1 Traits: shared behaviour

A trait is like a TypeScript `interface`, but you can implement it for **any** type, including types you didn't write.

```rust
trait Describe {
    fn describe(&self) -> String;

    // A default method: implementors get it for free
    fn shout(&self) -> String {
        self.describe().to_uppercase()
    }
}

struct Project { name: String }
struct Customer { id: i32 }

impl Describe for Project {
    fn describe(&self) -> String { format!("Project {}", self.name) }
}

impl Describe for Customer {
    fn describe(&self) -> String { format!("Customer #{}", self.id) }
}
```

### 4.2 Generics with trait bounds

```rust
// Works for ANY type T that implements Describe
fn print_it<T: Describe>(item: &T) {
    println!("{}", item.describe());
}

// The same thing written two other ways
fn print_it2(item: &impl Describe) { /* ... */ }
fn print_it3<T>(item: &T) where T: Describe { /* ... */ }
```

Multiple bounds:

```rust
fn log<T: Describe + Clone + Send>(item: T) { /* ... */ }
```

### 4.3 Generic structs

```rust
struct Wrapper<T> {
    value: T,
}

let a = Wrapper { value: 5 };          // Wrapper<i32>
let b = Wrapper { value: "hi" };       // Wrapper<&str>
```

`Option<T>`, `Result<T, E>`, `Vec<T>`, and Axum's `Json<T>` are all generic types like this.

### 4.4 Returning `impl Trait`

```rust
fn make() -> impl Describe {
    Project { name: "X".into() }
}
```

This means "I return *some* type that implements `Describe`." Axum handlers often return `impl IntoResponse`.

### 4.5 Standard traits you'll meet constantly

| Trait | Meaning |
|---|---|
| `Debug` | Printable with `{:?}` |
| `Display` | Printable with `{}` |
| `Clone` | Can be duplicated with `.clone()` |
| `Copy` | Duplicated implicitly (small types) |
| `PartialEq` / `Eq` | Comparable with `==` |
| `Default` | Has a default value, `T::default()` |
| `From` / `Into` | Converts between types |
| `Send` | Safe to move to another thread |
| `Sync` | Safe to share between threads |

### 4.6 `From` and `Into`

```rust
impl From<std::num::ParseIntError> for AppError {
    fn from(_: std::num::ParseIntError) -> Self {
        AppError::BadRequest("not a number".into())
    }
}

fn parse(s: &str) -> Result<i32, AppError> {
    let n: i32 = s.parse()?;  // `?` calls From automatically to convert the error
    Ok(n)
}
```

This is how library errors such as `sqlx::Error` get converted into your `AppError` behind the scenes.

### 4.7 Why this matters for Axum

Axum is essentially a set of traits:

| Axum trait | Meaning |
|---|---|
| `IntoResponse` | "This type can become an HTTP response" |
| `FromRequest` / `FromRequestParts` | "This type can be extracted from a request" |
| `Handler` | "This async function can handle requests" |

Strings, `Json<T>`, `StatusCode`, and tuples all implement `IntoResponse`, which is why any of them can be returned from a handler:

```rust
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        match self {
            AppError::NotFound => (StatusCode::NOT_FOUND, "not found").into_response(),
            // ...
        }
    }
}
```

When you see a long, confusing Axum compile error, it almost always means **"your type does not implement the trait Axum expects."**

---

## 5. Derive macros and serde

### 5.1 `#[derive]`

`#[derive]` makes the compiler implement common traits for you:

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point { x: i32, y: i32 }

let p = Point { x: 1, y: 2 };
println!("{:?}", p);        // Debug:  Point { x: 1, y: 2 }
println!("{:#?}", p);       // pretty-printed Debug
let q = p.clone();          // Clone
assert_eq!(p, q);           // PartialEq
```

### 5.2 serde: converting between Rust and JSON

`Cargo.toml`:

```toml
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Project {
    id: i32,
    name: String,
    description: Option<String>,  // a missing or null JSON value becomes None
}

// Rust -> JSON (Serialize)
let p = Project { id: 1, name: "Alpha".into(), description: None };
let json = serde_json::to_string(&p)?;
// {"id":1,"name":"Alpha","description":null}

// JSON -> Rust (Deserialize)
let input = r#"{"id":2,"name":"Beta"}"#;
let p2: Project = serde_json::from_str(input)?;
```

> **JS comparison:** `Serialize` is `JSON.stringify` and `Deserialize` is `JSON.parse`, except that deserializing **validates the shape**. A missing required field or a wrong type produces an error.

### 5.3 Useful serde attributes

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]          // first_name <-> "firstName"
struct UserDto {
    first_name: String,

    #[serde(default)]                       // missing field -> Default value
    is_active: bool,

    #[serde(skip_serializing_if = "Option::is_none")]  // omit the key if None
    email: Option<String>,

    #[serde(rename = "type")]               // `type` is a Rust keyword
    kind: String,
}

#[derive(Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]         // InProgress <-> "in_progress"
enum ProjectStatus { Draft, InProgress, Completed, Live }
```

### 5.4 How this looks in Axum

```rust
#[derive(Deserialize)]
struct CreateProject { name: String }       // the request body

#[derive(Serialize)]
struct ProjectDto { id: i32, name: String } // the response body

async fn create(Json(body): Json<CreateProject>) -> Json<ProjectDto> {
    Json(ProjectDto { id: 1, name: body.name })
}
```

`Json<T>` requires `T: Deserialize` when it's used as input and `T: Serialize` when it's used as output. In larger projects, these DTOs usually live in a shared `types` module or crate.

---

## 6. Smart pointers and sharing: `Box`, `Rc`, `Arc`, `Mutex`

### 6.1 The problem

A web server handles many requests **at the same time, on different threads**. All of them need the same database pool and config. Ownership rules say a value has only one owner, so how can many threads share it?

### 6.2 `Arc<T>`: Atomic Reference Counted

`Arc` lets **many owners share read-only data** safely across threads. Cloning an `Arc` does **not** copy the data. It only increments a counter.

```rust
use std::sync::Arc;
use std::thread;

let config = Arc::new(String::from("db=postgres://..."));

let mut handles = vec![];
for i in 0..3 {
    let cfg = Arc::clone(&config);   // a cheap clone: counter + 1
    handles.push(thread::spawn(move || {
        println!("thread {i} sees {cfg}");
    }));
}
for h in handles { h.join().unwrap(); }
// The data is freed when the last Arc is dropped
```

`Rc<T>` does the same thing but is **single-threaded only**. It is not `Send`, so you won't use it in Axum.

### 6.3 `Arc<Mutex<T>>`: shared and **mutable**

`Arc` alone gives read-only access. To change shared data, wrap it in a lock:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));

let c = Arc::clone(&counter);
thread::spawn(move || {
    let mut n = c.lock().unwrap();   // only one thread at a time gets in
    *n += 1;
}).join().unwrap();

println!("{}", *counter.lock().unwrap()); // 1
```

| Type | Use |
|---|---|
| `Mutex<T>` | One reader *or* writer at a time |
| `RwLock<T>` | Many readers *or* one writer |
| `tokio::sync::Mutex` | Use when the lock must be held across an `.await` |

A common pattern for in-memory shared state is `Arc<RwLock<HashMap<K, V>>>`. **Golden rule:** never hold a `std::sync` lock across an `.await`.

### 6.4 `move` closures

`move` makes a closure take ownership of the variables it uses. You need it whenever you send data into another thread or async task:

```rust
let name = String::from("x");
let f = move || println!("{name}");  // name is moved into the closure
```

### 6.5 `Send` and `Sync`

- `Send`: the value can be **moved** to another thread.
- `Sync`: the value can be **shared** (`&T`) between threads.

`Arc<T>` is `Send + Sync` if `T` is too. `Rc<T>` is neither. Axum requires state and futures to be `Send`, so an error like "`Rc<...>` cannot be sent between threads safely" means you used a type that isn't thread-safe.

### 6.6 How this looks in Axum

```rust
#[derive(Clone)]                       // Axum clones the state for every request
struct AppState {
    pool: sqlx::PgPool,                // already an Arc internally, so cheap to clone
    config: Arc<Config>,               // wrapped so the clone is cheap
    cache: Arc<RwLock<HashMap<i32, String>>>, // shared and mutable
}
```

---

## 7. Putting it all together

This program uses every topic above. It needs no Axum, only:

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, RwLock};

#[derive(Debug)]
enum AppError {
    NotFound(i32),
    BadRequest(String),
}

impl std::fmt::Display for AppError {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        match self {
            AppError::NotFound(id) => write!(f, "project {id} not found"),
            AppError::BadRequest(m) => write!(f, "bad request: {m}"),
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Project {
    id: i32,
    name: String,
}

#[derive(Clone)]
struct AppState {
    projects: Arc<RwLock<HashMap<i32, Project>>>,
}

impl AppState {
    fn new() -> Self {
        Self { projects: Arc::new(RwLock::new(HashMap::new())) }
    }

    fn create(&self, json: &str) -> Result<Project, AppError> {
        let p: Project = serde_json::from_str(json)
            .map_err(|e| AppError::BadRequest(e.to_string()))?;
        let mut map = self.projects.write()
            .map_err(|_| AppError::BadRequest("lock poisoned".into()))?;
        map.insert(p.id, p.clone());
        Ok(p)
    }

    fn get(&self, id: i32) -> Result<Project, AppError> {
        let map = self.projects.read()
            .map_err(|_| AppError::BadRequest("lock poisoned".into()))?;
        map.get(&id).cloned().ok_or(AppError::NotFound(id))
    }
}

fn main() {
    let state = AppState::new();

    match state.create(r#"{"id":1,"name":"Alpha"}"#) {
        Ok(p) => println!("created {:?}", p),
        Err(e) => println!("error: {e}"),
    }

    match state.get(1) {
        Ok(p) => println!("json: {}", serde_json::to_string(&p).unwrap_or_default()),
        Err(e) => println!("error: {e}"),
    }

    if let Err(e) = state.get(99) {
        println!("error: {e}");
    }

    if let Err(e) = state.create(r#"{"id":"oops"}"#) {
        println!("error: {e}");
    }
}
```

Expected output (the exact line and column in serde's message may differ slightly):

```text
created Project { id: 1, name: "Alpha" }
json: {"id":1,"name":"Alpha"}
error: project 99 not found
error: bad request: invalid type: string "oops", expected i32 at line 1 column 12
```

**How this maps to Axum:** `AppState` is the shared state, `create` and `get` play the role of handlers, `AppError` is the error type, and the serde structs are the DTOs. In Stage 3, `main` is replaced with a `Router`.

---

## 8. Exercises

- [ ] **1. Ownership:** write `fn first_word(s: &str) -> &str` that returns the text before the first space.
- [ ] **2. Struct and impl:** create `Customer { id, name, email: Option<String> }` with `new()` and `has_email(&self) -> bool`.
- [ ] **3. Enum and match:** create `ProjectStatus` with 4 variants and `fn can_edit(&self) -> bool`, which is false only for `Live`.
- [ ] **4. Result and `?`:** write `fn parse_ids(input: &str) -> Result<Vec<i32>, String>` that turns `"1,2,3"` into `vec![1, 2, 3]` and returns an error on bad input.
- [ ] **5. Trait:** define `trait Validate { fn validate(&self) -> Result<(), String>; }` and implement it for `Customer` (name must not be empty).
- [ ] **6. serde:** deserialize `{"firstName":"Anna","status":"in_progress"}` into a struct using `rename_all`.
- [ ] **7. Arc:** spawn 5 threads that each add 10 to a shared `Arc<Mutex<i32>>`, then print the result (`50`).

> When you can solve exercises 1–7 without looking anything up, you're ready for **Stage 1: Async Rust and Tokio**.

---

## 9. Further reading

- [*The Rust Programming Language*](https://doc.rust-lang.org/book/): chapters 4 (ownership), 5 (structs), 6 (enums), 9 (errors), 10 (generics, traits, lifetimes), 15 (smart pointers), and 16 (concurrency)
- [*Rust by Example*](https://doc.rust-lang.org/rust-by-example/): short runnable snippets
- [Rustlings](https://github.com/rust-lang/rustlings): small exercises covering exactly these topics
- [serde.rs](https://serde.rs/): the full list of serde attributes
