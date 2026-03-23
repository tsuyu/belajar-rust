# 🌐 Axum — Web Framework Rust yang Moden

> Panduan lengkap Axum: routing, extractors, middleware,
> state management, error handling, dan real-world patterns.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Kenapa Axum?

```
Axum = web framework dari tim Tokio
     = dibina di atas Tower (middleware ecosystem)
     = ergonomic, composable, production-ready

Keunggulan Axum:
  ✔ Extractor system — TypeSafe, ergonomic
  ✔ Tower compatibility — guna ribuan Tower middleware
  ✔ Error handling — type-safe, expressive
  ✔ Macroless routing — tiada proc macro
  ✔ Nested routing — compose router dengan mudah
  ✔ WebSocket support built-in
  ✔ Server-Sent Events support

Berbanding frameworks lain:
  Actix-web: Cepat tapi API lebih verbose, guna Actor model
  Rocket:    Ergonomic tapi guna proc macros, slower compile
  Warp:      Functional tapi API sukar untuk beginner
  Axum:      Balance antara ergonomics + composability + performance
```

---

## Peta Pembelajaran

```
Bahagian 1  → Setup & Hello World
Bahagian 2  → Routing
Bahagian 3  → Extractors — Baca Request
Bahagian 4  → Response — Hantar Balas
Bahagian 5  → State — Data Kongsi
Bahagian 6  → Error Handling
Bahagian 7  → Middleware & Layer
Bahagian 8  → Nested Router & Modular App
Bahagian 9  → WebSocket & SSE
Bahagian 10 → Mini Project: KADA REST API
```

---

# BAHAGIAN 1: Setup & Hello World 🚀

## Cargo.toml

```toml
[dependencies]
axum         = "0.7"
tokio        = { version = "1", features = ["full"] }
tower        = "0.4"
tower-http   = { version = "0.5", features = [
    "cors",
    "trace",
    "timeout",
    "compression-gzip",
    "set-header",
] }
serde        = { version = "1", features = ["derive"] }
serde_json   = "1"
tracing      = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

## Hello World Minimum

```rust
// src/main.rs
use axum::{Router, routing::get};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(halaman_utama));

    let pendengar = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    println!("Server berjalan pada http://localhost:3000");
    axum::serve(pendengar, app).await.unwrap();
}

async fn halaman_utama() -> &'static str {
    "Halo dari Axum! 🦀"
}
```

---

## Handler — Anatomi Asas

```rust
use axum::{
    Router,
    routing::{get, post, put, delete, patch},
    response::IntoResponse,
    http::StatusCode,
};

// Handler boleh return apa-apa yang impl IntoResponse
async fn handler_string()    -> &'static str { "String" }
async fn handler_owned()     -> String       { "Owned String".into() }
async fn handler_status()    -> StatusCode   { StatusCode::OK }
async fn handler_tuple()     -> (StatusCode, &'static str) {
    (StatusCode::CREATED, "Berjaya dicipta")
}
async fn handler_impl_resp() -> impl IntoResponse {
    (StatusCode::OK, "Response via impl")
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/get",    get(handler_string))
        .route("/post",   post(handler_owned))
        .route("/put",    put(handler_status))
        .route("/delete", delete(handler_tuple))
        .route("/patch",  patch(handler_impl_resp))
        // Satu route boleh handle banyak method
        .route("/multi",  get(handler_string).post(handler_owned));

    axum::serve(
        tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap(),
        app
    ).await.unwrap();
}
```

---

# BAHAGIAN 2: Routing 🗺️

## Route Patterns

```rust
use axum::{
    Router,
    routing::get,
    extract::Path,
};

async fn senarai()                   -> String { "Senarai".into() }
async fn dapatkan(Path(id): Path<u32>) -> String { format!("Item #{}", id) }
async fn versi_dua()                 -> String { "v2".into() }
async fn wildcard(Path(p): Path<String>) -> String { format!("Path: {}", p) }

#[tokio::main]
async fn main() {
    let app = Router::new()
        // ── Path biasa ──────────────────────────────────────
        .route("/pekerja",           get(senarai))
        .route("/pekerja/:id",       get(dapatkan))

        // ── Wildcard (*) — match semua ──────────────────────
        .route("/static/*laluan",    get(wildcard))

        // ── Nested routes ───────────────────────────────────
        .nest("/api/v1", Router::new()
            .route("/pekerja",    get(senarai))
            .route("/pekerja/:id", get(dapatkan))
        )
        .nest("/api/v2", Router::new()
            .route("/pekerja", get(versi_dua))
        )

        // ── Fallback ─────────────────────────────────────────
        .fallback(handler_404);

    axum::serve(
        tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap(),
        app
    ).await.unwrap();
}

async fn handler_404() -> (axum::http::StatusCode, &'static str) {
    (axum::http::StatusCode::NOT_FOUND, "Halaman tidak dijumpai")
}
```

---

## Router Modular

```rust
// src/routes/pekerja.rs
use axum::{Router, routing::{get, post, put, delete}, extract::Path};

pub fn router() -> Router {
    Router::new()
        .route("/",    get(senarai).post(buat))
        .route("/:id", get(dapatkan).put(kemaskini).delete(padam))
}

async fn senarai()                     -> String { "Senarai pekerja".into() }
async fn buat()                        -> String { "Buat pekerja".into() }
async fn dapatkan(Path(id): Path<u32>) -> String { format!("Pekerja #{}", id) }
async fn kemaskini(Path(id): Path<u32>) -> String { format!("Kemaskini #{}", id) }
async fn padam(Path(id): Path<u32>)    -> String { format!("Padam #{}", id) }

// src/main.rs
fn bina_app() -> Router {
    Router::new()
        .nest("/pekerja",   pekerja::router())
        .nest("/kehadiran", kehadiran::router())
}
```

---

## 🧠 Brain Teaser #1

Apakah perbezaan antara `.route()` dan `.nest()`?

<details>
<summary>👀 Jawapan</summary>

```rust
// .route() — match path TEPAT
router.route("/api/v1/pekerja", get(handler))
// Hanya match "/api/v1/pekerja" sahaja (exact match)

// .nest() — match path PREFIX dan strip prefix
router.nest("/api/v1", inner_router)
// Match "/api/v1/*" dan hantar ke inner_router
// TAPI prefix "/api/v1" di-strip dari path yang dihantar!

// Contoh:
// Request: GET /api/v1/pekerja
// Selepas .nest("/api/v1"):
// inner_router nampak: GET /pekerja
// (bukan GET /api/v1/pekerja)

// Implikasi:
// inner_router.route("/pekerja", handler)  ← BETUL
// inner_router.route("/api/v1/pekerja", handler)  ← TIDAK MATCH!

// Satu lagi perbezaan:
// .route() boleh ada multiple handlers untuk HTTP methods berbeza:
//   router.route("/x", get(a).post(b).delete(c))
//
// .nest() hanya satu router:
//   router.nest("/api", inner_router)
```
</details>

---

# BAHAGIAN 3: Extractors — Baca Request 📥

## Semua Extractor Utama

```rust
use axum::{
    extract::{Path, Query, State, Json, Form, Header, Extension},
    http::{HeaderMap, Method, Uri},
    body::Bytes,
};
use serde::Deserialize;

// ── Path Extractor ────────────────────────────────────────────
async fn satu_param(Path(id): Path<u32>) -> String {
    format!("ID: {}", id)
}

async fn banyak_param(Path((bahagian, id)): Path<(String, u32)>) -> String {
    format!("{}/{}", bahagian, id)
}

// ── Query String Extractor ────────────────────────────────────
#[derive(Deserialize)]
struct QuerySaring {
    cari:     Option<String>,
    halaman:  Option<u32>,
    saiz:     Option<u32>,
    aktif:    Option<bool>,
}

async fn dengan_query(Query(q): Query<QuerySaring>) -> String {
    format!("Cari: {:?}, Halaman: {:?}",
        q.cari, q.halaman)
}

// ── JSON Body Extractor ───────────────────────────────────────
#[derive(Deserialize)]
struct BuatPekerja {
    nama:     String,
    bahagian: String,
    gaji:     f64,
}

async fn dari_json(Json(body): Json<BuatPekerja>) -> String {
    format!("Buat: {} di {}", body.nama, body.bahagian)
}

// ── Form Data Extractor ───────────────────────────────────────
#[derive(Deserialize)]
struct FormLogin {
    username: String,
    password: String,
}

async fn dari_form(Form(data): Form<FormLogin>) -> String {
    format!("Login: {}", data.username)
}

// ── Header Extractor ──────────────────────────────────────────
async fn dari_header(
    headers: HeaderMap,
) -> String {
    let ua = headers.get("user-agent")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("unknown");
    format!("User-Agent: {}", ua)
}

// ── Raw Request Info ─────────────────────────────────────────
async fn request_info(
    method:  Method,
    uri:     Uri,
    headers: HeaderMap,
    body:    Bytes,
) -> String {
    format!("{} {} ({} bytes)", method, uri, body.len())
}

// ── Optional Extractor ───────────────────────────────────────
async fn optional_query(
    Query(q): Query<QuerySaring>
) -> String {
    // QuerySaring dengan Option fields — tidak akan panic kalau kosong
    match q.cari {
        Some(c) => format!("Cari: {}", c),
        None    => "Tiada filter".into(),
    }
}
```

---

## Custom Extractor

```rust
use axum::{
    extract::{FromRequestParts, FromRequest},
    http::{request::Parts, StatusCode, HeaderMap},
    async_trait,
};

// ── Extract Bearer Token ──────────────────────────────────────
struct BearerToken(String);

#[async_trait]
impl<S> FromRequestParts<S> for BearerToken
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> Result<Self, Self::Rejection> {
        let token = parts.headers
            .get("authorization")
            .and_then(|v| v.to_str().ok())
            .and_then(|s| s.strip_prefix("Bearer "))
            .ok_or((StatusCode::UNAUTHORIZED, "Token diperlukan"))?;

        Ok(BearerToken(token.to_string()))
    }
}

// Guna dalam handler
async fn endpoint_selamat(BearerToken(token): BearerToken) -> String {
    format!("Token: {}...", &token[..8.min(token.len())])
}

// ── Extract Claims dari JWT ───────────────────────────────────
#[derive(Clone)]
struct ClaimsUser {
    id:        u32,
    username:  String,
    peranan:   Vec<String>,
}

#[async_trait]
impl<S> FromRequestParts<S> for ClaimsUser
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(
        parts: &mut Parts,
        _state: &S,
    ) -> Result<Self, Self::Rejection> {
        // Ambil dari extension (diset oleh auth middleware)
        parts.extensions
            .get::<ClaimsUser>()
            .cloned()
            .ok_or((StatusCode::UNAUTHORIZED, "Tidak log masuk"))
    }
}

async fn profil(pengguna: ClaimsUser) -> String {
    format!("Halo, {}!", pengguna.username)
}
```

---

# BAHAGIAN 4: Response — Hantar Balas 📤

## Semua Cara Hantar Response

```rust
use axum::{
    response::{IntoResponse, Response, Json, Html, Redirect},
    http::{StatusCode, header, HeaderValue},
};
use serde::Serialize;

// ── String / bytes ────────────────────────────────────────────
async fn teks()  -> &'static str { "Teks biasa" }
async fn owned() -> String       { "Owned string".into() }

// ── Status code ───────────────────────────────────────────────
async fn kosong()  -> StatusCode { StatusCode::NO_CONTENT }
async fn ralat()   -> StatusCode { StatusCode::INTERNAL_SERVER_ERROR }

// ── Tuple (status + body) ─────────────────────────────────────
async fn tuple_resp() -> (StatusCode, String) {
    (StatusCode::CREATED, "Berjaya!".into())
}

// ── JSON Response ─────────────────────────────────────────────
#[derive(Serialize)]
struct ApiOk<T: Serialize> {
    berjaya: bool,
    data:    T,
}

async fn json_resp() -> Json<ApiOk<Vec<&'static str>>> {
    Json(ApiOk {
        berjaya: true,
        data:    vec!["ali", "siti"],
    })
}

// ── HTML Response ─────────────────────────────────────────────
async fn html_resp() -> Html<String> {
    Html("<h1>Halo HTML!</h1>".into())
}

// ── Redirect ──────────────────────────────────────────────────
async fn redirect() -> Redirect {
    Redirect::to("/halaman-baru")
}

// ── Custom headers ────────────────────────────────────────────
async fn dengan_header() -> impl IntoResponse {
    (
        StatusCode::OK,
        [(header::CONTENT_TYPE, "application/json"),
         (header::CACHE_CONTROL, "no-cache")],
        r#"{"mesej":"ok"}"#,
    )
}

// ── Custom Response builder ───────────────────────────────────
async fn custom_resp() -> Response {
    Response::builder()
        .status(StatusCode::OK)
        .header(header::CONTENT_TYPE, "text/plain")
        .header("x-custom", "nilai")
        .body(axum::body::Body::from("Custom body"))
        .unwrap()
}

// ── File download ─────────────────────────────────────────────
async fn download() -> impl IntoResponse {
    let data = b"Kandungan fail...".to_vec();
    (
        StatusCode::OK,
        [
            (header::CONTENT_TYPE,        "application/octet-stream"),
            (header::CONTENT_DISPOSITION, "attachment; filename=\"data.txt\""),
        ],
        data,
    )
}
```

---

# BAHAGIAN 5: State — Data Kongsi 🏛️

## AppState Pattern

```rust
use axum::{Router, routing::get, extract::State};
use sqlx::MySqlPool;
use std::sync::Arc;

// ── State struct ──────────────────────────────────────────────
#[derive(Clone)]
struct AppState {
    db:     MySqlPool,
    config: Arc<Config>,
}

#[derive(Debug)]
struct Config {
    nama_app: String,
    versi:    String,
    debug:    bool,
}

// ── State dalam handler ───────────────────────────────────────
async fn dengan_state(
    State(state): State<AppState>,
) -> String {
    format!("{} v{}", state.config.nama_app, state.config.versi)
}

// ── State extraction berbeza ──────────────────────────────────

// Cara 1: Extract seluruh state
async fn handler_penuh(State(state): State<AppState>) -> String {
    state.config.nama_app.clone()
}

// Cara 2: Subset state (impl FromRef)
use axum::extract::FromRef;

#[derive(Clone)]
struct DbState(MySqlPool);

impl FromRef<AppState> for DbState {
    fn from_ref(state: &AppState) -> Self {
        DbState(state.db.clone())
    }
}

async fn handler_db(State(db): State<DbState>) -> String {
    // Hanya perlu DB, bukan seluruh AppState
    "DB handler".into()
}

// ── Setup ─────────────────────────────────────────────────────
async fn setup() {
    let state = AppState {
        db:     MySqlPool::connect("mysql://localhost/db").await.unwrap(),
        config: Arc::new(Config {
            nama_app: "KADA API".into(),
            versi:    "2.0".into(),
            debug:    false,
        }),
    };

    let app = Router::new()
        .route("/",   get(dengan_state))
        .route("/db", get(handler_db))
        .with_state(state);

    axum::serve(
        tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap(),
        app
    ).await.unwrap();
}
```

---

## Multiple State Types

```rust
use axum::{Router, routing::get, extract::State};
use std::sync::Arc;
use deadpool_redis::Pool as RedisPool;

#[derive(Clone)]
struct AppState {
    db:    Arc<sqlx::MySqlPool>,
    redis: RedisPool,
    jwt:   Arc<JwtConfig>,
}

struct JwtConfig {
    rahsia: String,
    tamat:  u64,
}

// Extrak bahagian state sahaja
use axum::extract::FromRef;

impl FromRef<AppState> for Arc<sqlx::MySqlPool> {
    fn from_ref(s: &AppState) -> Self { Arc::clone(&s.db) }
}

impl FromRef<AppState> for RedisPool {
    fn from_ref(s: &AppState) -> Self { s.redis.clone() }
}

impl FromRef<AppState> for Arc<JwtConfig> {
    fn from_ref(s: &AppState) -> Self { Arc::clone(&s.jwt) }
}

// Handler boleh minta bahagian yang perlu sahaja!
async fn hanya_db(State(db): State<Arc<sqlx::MySqlPool>>) -> String {
    "DB handler".into()
}

async fn hanya_redis(State(redis): State<RedisPool>) -> String {
    "Redis handler".into()
}

async fn semua(State(state): State<AppState>) -> String {
    "Full state".into()
}
```

---

# BAHAGIAN 6: Error Handling ⚠️

## AppError dengan IntoResponse

```rust
use axum::{
    response::{IntoResponse, Response, Json},
    http::StatusCode,
};
use serde_json::json;
use thiserror::Error;

// ── Define error types ────────────────────────────────────────
#[derive(Debug, Error)]
pub enum AppError {
    #[error("Tidak dijumpai: {0}")]
    TidakJumpai(String),

    #[error("Input tidak sah: {0}")]
    InputTidakSah(String),

    #[error("Tidak dibenarkan")]
    TidakDibenarkan,

    #[error("Dilarang: {0}")]
    Dilarang(String),

    #[error("Konflik: {0}")]
    Konflik(String),

    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("Ralat dalaman")]
    Dalaman(#[source] Box<dyn std::error::Error + Send + Sync>),
}

// ── Implement IntoResponse ────────────────────────────────────
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, kod, mesej) = match &self {
            AppError::TidakJumpai(m)    => (StatusCode::NOT_FOUND,            404, m.clone()),
            AppError::InputTidakSah(m)  => (StatusCode::UNPROCESSABLE_ENTITY, 422, m.clone()),
            AppError::TidakDibenarkan   => (StatusCode::UNAUTHORIZED,         401, "Tidak dibenarkan".into()),
            AppError::Dilarang(m)       => (StatusCode::FORBIDDEN,            403, m.clone()),
            AppError::Konflik(m)        => (StatusCode::CONFLICT,             409, m.clone()),
            AppError::Database(e) => {
                tracing::error!("DB error: {}", e);
                (StatusCode::INTERNAL_SERVER_ERROR, 500, "Ralat pangkalan data".into())
            }
            AppError::Dalaman(e) => {
                tracing::error!("Ralat: {}", e);
                (StatusCode::INTERNAL_SERVER_ERROR, 500, "Ralat dalaman".into())
            }
        };

        (status, Json(json!({
            "berjaya": false,
            "kod":     kod,
            "ralat":   mesej,
        }))).into_response()
    }
}

pub type AppResult<T> = Result<T, AppError>;

// ── Guna dalam handler ────────────────────────────────────────
async fn dapatkan_pekerja(
    axum::extract::Path(id): axum::extract::Path<u32>,
) -> AppResult<Json<serde_json::Value>> {
    if id == 0 {
        return Err(AppError::InputTidakSah("ID mesti lebih dari 0".into()));
    }

    if id > 1000 {
        return Err(AppError::TidakJumpai(format!("Pekerja #{}", id)));
    }

    Ok(Json(json!({ "id": id, "nama": "Ali Ahmad" })))
}
```

---

## Validation Error Response

```rust
use validator::{Validate, ValidationErrors};
use axum::{response::{IntoResponse, Json}, http::StatusCode};
use serde_json::{json, Value};

fn validation_error_ke_json(errors: ValidationErrors) -> Value {
    let mut mesej_list = Vec::new();

    for (field, errors) in errors.field_errors() {
        for error in errors {
            mesej_list.push(json!({
                "field": field,
                "kod":   error.code.to_string(),
                "mesej": error.message.as_deref().unwrap_or("Tidak sah"),
            }));
        }
    }

    json!({
        "berjaya":  false,
        "kod":      422,
        "ralat":    "Input tidak sah",
        "butiran":  mesej_list,
    })
}

#[derive(serde::Deserialize, Validate)]
struct BuatPekerja {
    #[validate(length(min = 2, max = 100))]
    nama: String,

    #[validate(email)]
    emel: String,

    #[validate(range(min = 0.0))]
    gaji: f64,
}

async fn buat_pekerja(
    Json(body): Json<BuatPekerja>,
) -> impl IntoResponse {
    match body.validate() {
        Ok(_)       => (StatusCode::CREATED, Json(json!({"mesej": "Berjaya"}))).into_response(),
        Err(errors) => (StatusCode::UNPROCESSABLE_ENTITY,
                        Json(validation_error_ke_json(errors))).into_response(),
    }
}
```

---

# BAHAGIAN 7: Middleware & Layer 🔧

## Tower Middleware

```rust
use axum::{
    Router,
    middleware::{self, Next},
    extract::Request,
    response::Response,
    http::{StatusCode, header},
};
use tower::ServiceBuilder;
use tower_http::{
    cors::{CorsLayer, Any},
    trace::TraceLayer,
    timeout::TimeoutLayer,
    compression::CompressionLayer,
};
use std::time::Duration;

// ── Custom Middleware Functions ───────────────────────────────

// 1. Log setiap request
async fn log_middleware(
    req: Request,
    next: Next,
) -> Response {
    let method = req.method().clone();
    let uri    = req.uri().clone();
    let mula   = std::time::Instant::now();

    let resp = next.run(req).await;

    tracing::info!(
        method = %method,
        uri    = %uri,
        status = resp.status().as_u16(),
        masa   = mula.elapsed().as_millis(),
        "Request"
    );

    resp
}

// 2. Tambah correlation ID
async fn correlation_id(
    mut req: Request,
    next: Next,
) -> Response {
    let id = req.headers()
        .get("x-correlation-id")
        .and_then(|v| v.to_str().ok())
        .map(String::from)
        .unwrap_or_else(|| uuid::Uuid::new_v4().to_string());

    // Inject ke extension
    req.extensions_mut().insert(id.clone());

    let mut resp = next.run(req).await;

    // Tambah ke response header
    resp.headers_mut().insert(
        "x-correlation-id",
        id.parse().unwrap(),
    );

    resp
}

// 3. Auth middleware
async fn semak_auth(
    State(state): State<AppState>,
    mut req: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let token = req.headers()
        .get(header::AUTHORIZATION)
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.strip_prefix("Bearer "))
        .ok_or(StatusCode::UNAUTHORIZED)?;

    let claims = state.jwt.verify(token)
        .map_err(|_| StatusCode::UNAUTHORIZED)?;

    req.extensions_mut().insert(claims);
    Ok(next.run(req).await)
}

// ── Security headers ──────────────────────────────────────────
async fn security_headers(req: Request, next: Next) -> Response {
    let mut resp = next.run(req).await;
    let h = resp.headers_mut();

    h.insert("x-frame-options",         "DENY".parse().unwrap());
    h.insert("x-content-type-options",  "nosniff".parse().unwrap());
    h.insert("x-xss-protection",        "1; mode=block".parse().unwrap());
    h.insert("referrer-policy",         "strict-origin-when-cross-origin".parse().unwrap());
    h.remove("server");

    resp
}

// ── Susun semua middleware ────────────────────────────────────
fn bina_app(state: AppState) -> Router {
    // Routes yang perlu auth
    let route_lindung = Router::new()
        .route("/profil", get(profil_handler))
        .layer(middleware::from_fn_with_state(
            state.clone(),
            semak_auth,
        ));

    Router::new()
        .route("/sihat", get(sihat_handler))
        .merge(route_lindung)
        // Tower middleware layers
        .layer(
            ServiceBuilder::new()
                // Order PENTING! Pertama = paling luar
                .layer(middleware::from_fn(security_headers))
                .layer(middleware::from_fn(correlation_id))
                .layer(middleware::from_fn(log_middleware))
                .layer(TraceLayer::new_for_http())
                .layer(TimeoutLayer::new(Duration::from_secs(30)))
                .layer(CompressionLayer::new())
                .layer(
                    CorsLayer::new()
                        .allow_origin(Any)
                        .allow_methods(Any)
                        .allow_headers(Any)
                )
        )
        .with_state(state)
}

async fn sihat_handler()  -> &'static str { "OK" }
async fn profil_handler() -> &'static str { "Profil" }
```

---

## 🧠 Brain Teaser #2

Mengapa order layer dalam Axum penting, dan apakah yang berlaku bila kita tukar ordernya?

<details>
<summary>👀 Jawapan</summary>

```
Layer dalam Axum dibaca dari LUAR ke DALAM.
Layer yang ditambah KEMUDIAN = lapisan LUAR.
Layer yang ditambah DULU = lapisan DALAM.

Router::new()
    .route("/", get(handler))
    .layer(LayerA)   // ← lapisan dalam (dekat dengan handler)
    .layer(LayerB)   // ← lapisan tengah
    .layer(LayerC);  // ← lapisan paling luar

Request flow:  LayerC → LayerB → LayerA → Handler
Response flow: Handler → LayerA → LayerB → LayerC

IMPLIKASI PRAKTIKAL:

// ❌ SALAH — Auth check selepas logging
// Kalau auth gagal, logging tidak nampak "auth failed"
router
    .layer(semak_auth)    // dalam — auth dulu
    .layer(log);          // luar — log kemudian

// ✔ BETUL — Log semua request, kemudian auth check
router
    .layer(log)           // dalam — log dulu
    .layer(semak_auth);   // luar... tapi ini salah juga!

// SEBENARNYA betul:
router
    .layer(semak_auth)    // lapisan dalam — check auth
    .layer(log);          // lapisan luar — log semua request

// Kenapa? Kerana request masuk DARI LUAR:
// Request → log (catat request) → semak_auth (check) → handler

// Untuk CORS — mesti paling luar (paling kemudian):
router
    .layer(auth)
    .layer(log)
    .layer(cors);  // CORS check berlaku SEBELUM apa-apa
```
</details>

---

# BAHAGIAN 8: Nested Router & Modular App 🏗️

## Struktur App Modular

```rust
// src/handlers/pekerja.rs
use axum::{Router, routing::{get, post, put, delete}, extract::{Path, State, Json, Query}};
use serde::{Serialize, Deserialize};
use crate::{state::AppState, error::AppResult};

pub fn router() -> Router<AppState> {
    Router::new()
        .route("/",    get(senarai).post(buat))
        .route("/:id", get(dapatkan).put(kemaskini).delete(padam))
}

#[derive(Serialize, Deserialize)]
pub struct Pekerja {
    pub id:   u32,
    pub nama: String,
}

#[derive(Deserialize)]
pub struct BuatPekerja { pub nama: String }

#[derive(Deserialize)]
pub struct QueryFilter { pub aktif: Option<bool> }

pub async fn senarai(
    State(_state): State<AppState>,
    Query(q): Query<QueryFilter>,
) -> AppResult<Json<Vec<Pekerja>>> {
    Ok(Json(vec![Pekerja { id: 1, nama: "Ali".into() }]))
}

pub async fn buat(
    State(_state): State<AppState>,
    Json(body): Json<BuatPekerja>,
) -> AppResult<Json<Pekerja>> {
    Ok(Json(Pekerja { id: 999, nama: body.nama }))
}

pub async fn dapatkan(
    State(_state): State<AppState>,
    Path(id): Path<u32>,
) -> AppResult<Json<Pekerja>> {
    Ok(Json(Pekerja { id, nama: "Ali".into() }))
}

pub async fn kemaskini(
    State(_state): State<AppState>,
    Path(id): Path<u32>,
    Json(body): Json<BuatPekerja>,
) -> AppResult<Json<Pekerja>> {
    Ok(Json(Pekerja { id, nama: body.nama }))
}

pub async fn padam(
    State(_state): State<AppState>,
    Path(id): Path<u32>,
) -> AppResult<axum::http::StatusCode> {
    Ok(axum::http::StatusCode::NO_CONTENT)
}
```

```rust
// src/routes.rs
use axum::Router;
use crate::state::AppState;

pub fn bina_router(state: AppState) -> Router {
    Router::new()
        .route("/sihat", axum::routing::get(sihat))
        .nest("/api/v1", api_v1())
        .with_state(state)
}

fn api_v1() -> Router<AppState> {
    Router::new()
        .nest("/pekerja",   crate::handlers::pekerja::router())
        .nest("/kehadiran", crate::handlers::kehadiran::router())
        .nest("/auth",      crate::handlers::auth::router())
}

async fn sihat() -> &'static str { "OK" }
```

---

# BAHAGIAN 9: WebSocket & SSE 🔄

## WebSocket Handler

```rust
use axum::{
    Router,
    routing::get,
    extract::{
        ws::{WebSocket, WebSocketUpgrade, Message},
        State,
    },
    response::IntoResponse,
};
use futures::{StreamExt, SinkExt};

async fn ws_upgrade(
    ws: WebSocketUpgrade,
    State(state): State<AppState>,
) -> impl IntoResponse {
    ws.on_upgrade(|soket| kendalikan_ws(soket, state))
}

async fn kendalikan_ws(mut soket: WebSocket, _state: AppState) {
    // Hantar selamat datang
    soket.send(Message::Text("Sambungan berjaya!".into()))
        .await
        .ok();

    // Loop terima-balas
    while let Some(msg) = soket.recv().await {
        match msg {
            Ok(Message::Text(teks)) => {
                tracing::info!("WS terima: {}", teks);
                // Echo balik
                if soket.send(Message::Text(
                    format!("Echo: {}", teks)
                )).await.is_err() {
                    break; // klien putus
                }
            }
            Ok(Message::Binary(data)) => {
                soket.send(Message::Binary(data)).await.ok();
            }
            Ok(Message::Close(_)) | Err(_) => break,
            _ => {}
        }
    }

    tracing::info!("WS klien putus");
}

// ── WebSocket dengan broadcast (chat room) ────────────────────
use tokio::sync::broadcast;
use std::sync::Arc;

struct ChatState {
    tx: broadcast::Sender<String>,
}

async fn ws_chat(
    ws: WebSocketUpgrade,
    State(state): State<Arc<ChatState>>,
) -> impl IntoResponse {
    ws.on_upgrade(|soket| bilik_chat(soket, state))
}

async fn bilik_chat(soket: WebSocket, state: Arc<ChatState>) {
    let (mut pengirim, mut penerima) = soket.split();
    let mut rx = state.tx.subscribe();

    // Task: broadcast → klien
    let mut send_task = tokio::spawn(async move {
        while let Ok(mesej) = rx.recv().await {
            if pengirim.send(Message::Text(mesej)).await.is_err() {
                break;
            }
        }
    });

    // Task: klien → broadcast
    let tx = state.tx.clone();
    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(Message::Text(teks))) = penerima.next().await {
            tx.send(teks).ok();
        }
    });

    tokio::select! {
        _ = &mut send_task  => recv_task.abort(),
        _ = &mut recv_task  => send_task.abort(),
    }
}
```

---

## Server-Sent Events (SSE)

```rust
use axum::{
    response::sse::{Event, Sse},
    routing::get,
};
use futures::stream::{self, Stream};
use std::{convert::Infallible, time::Duration};
use tokio_stream::StreamExt as _;

async fn sse_handler() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    // Jana events setiap saat
    let strim = stream::repeat_with(|| {
        let masa = chrono::Utc::now().to_rfc3339();
        Event::default()
            .data(format!(r#"{{"masa":"{}"}}"#, masa))
            .event("kemas_kini")
    })
    .map(Ok)
    .throttle(Duration::from_secs(1));

    Sse::new(strim).keep_alive(
        axum::response::sse::KeepAlive::new()
            .interval(Duration::from_secs(30))
            .text("ping"),
    )
}
```

---

# BAHAGIAN 10: Mini Project — KADA REST API 🏗️

```rust
// Cargo.toml
// axum = "0.7"
// tokio = { version = "1", features = ["full"] }
// serde = { version = "1", features = ["derive"] }
// serde_json = "1"
// tower-http = { version = "0.5", features = ["cors", "trace"] }
// tracing-subscriber = { version = "0.3", features = ["env-filter"] }
// thiserror = "1"
// uuid = { version = "1", features = ["v4"] }

use axum::{
    Router,
    routing::{get, post, put, delete},
    extract::{State, Path, Json, Query},
    http::StatusCode,
    middleware::{self, Next},
    extract::Request,
    response::{IntoResponse, Response},
};
use serde::{Serialize, Deserialize};
use std::sync::Arc;
use tokio::sync::RwLock;
use std::collections::HashMap;
use uuid::Uuid;
use thiserror::Error;

// ─── Models ───────────────────────────────────────────────────

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Pekerja {
    id:          String,
    no_pekerja:  String,
    nama:        String,
    bahagian:    String,
    gaji:        f64,
    aktif:       bool,
    dibuat_pada: String,
}

#[derive(Debug, Deserialize, Clone)]
struct BuatPekerja {
    no_pekerja: String,
    nama:       String,
    bahagian:   String,
    gaji:       f64,
}

#[derive(Debug, Deserialize)]
struct KemaskiniPekerja {
    nama:     Option<String>,
    bahagian: Option<String>,
    gaji:     Option<f64>,
    aktif:    Option<bool>,
}

#[derive(Debug, Deserialize)]
struct QueryFilter {
    bahagian: Option<String>,
    aktif:    Option<bool>,
    cari:     Option<String>,
}

// ─── Response Types ────────────────────────────────────────────

#[derive(Serialize)]
struct ApiResponse<T: Serialize> {
    berjaya: bool,
    data:    Option<T>,
    mesej:   Option<String>,
}

impl<T: Serialize> ApiResponse<T> {
    fn ok(data: T)       -> Self { ApiResponse { berjaya: true, data: Some(data), mesej: None } }
    fn mesej(m: &str)    -> ApiResponse<()> { ApiResponse { berjaya: true, data: None, mesej: Some(m.into()) } }
    fn ralat(m: &str)    -> ApiResponse<()> { ApiResponse { berjaya: false, data: None, mesej: Some(m.into()) } }
}

// ─── Error ────────────────────────────────────────────────────

#[derive(Debug, Error)]
enum AppError {
    #[error("Tidak dijumpai: {0}")]
    TidakJumpai(String),
    #[error("Konflik: {0}")]
    Konflik(String),
    #[error("Input tidak sah: {0}")]
    Input(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, mesej) = match &self {
            AppError::TidakJumpai(m) => (StatusCode::NOT_FOUND,            m.clone()),
            AppError::Konflik(m)     => (StatusCode::CONFLICT,             m.clone()),
            AppError::Input(m)       => (StatusCode::UNPROCESSABLE_ENTITY, m.clone()),
        };
        (status, Json(ApiResponse::<()>::ralat(&mesej))).into_response()
    }
}

type AppResult<T> = Result<Json<ApiResponse<T>>, AppError>;

// ─── State ─────────────────────────────────────────────────────

#[derive(Clone)]
struct AppState {
    pekerja: Arc<RwLock<HashMap<String, Pekerja>>>,
}

impl AppState {
    fn baru() -> Self {
        let mut peta = HashMap::new();

        // Data awal
        let p1 = Pekerja {
            id: Uuid::new_v4().to_string(), no_pekerja: "KADA001".into(),
            nama: "Ali Ahmad".into(), bahagian: "ICT".into(),
            gaji: 4500.0, aktif: true, dibuat_pada: sekarang(),
        };
        let p2 = Pekerja {
            id: Uuid::new_v4().to_string(), no_pekerja: "KADA002".into(),
            nama: "Siti Hawa".into(), bahagian: "HR".into(),
            gaji: 3800.0, aktif: true, dibuat_pada: sekarang(),
        };

        peta.insert(p1.id.clone(), p1);
        peta.insert(p2.id.clone(), p2);

        AppState { pekerja: Arc::new(RwLock::new(peta)) }
    }
}

fn sekarang() -> String {
    chrono::Utc::now().format("%Y-%m-%dT%H:%M:%SZ").to_string()
}

// ─── Middleware ────────────────────────────────────────────────

async fn log_request(req: Request, next: Next) -> Response {
    let method = req.method().to_string();
    let uri    = req.uri().to_string();
    let mula   = std::time::Instant::now();
    let resp   = next.run(req).await;

    println!("[{}] {} {} — {}ms",
        resp.status().as_u16(), method, uri,
        mula.elapsed().as_millis());

    resp
}

// ─── Handlers ─────────────────────────────────────────────────

async fn sihat() -> Json<serde_json::Value> {
    Json(serde_json::json!({
        "status": "OK",
        "servis": "KADA API",
        "versi":  "2.0"
    }))
}

async fn senarai_pekerja(
    State(state): State<AppState>,
    Query(q): Query<QueryFilter>,
) -> Json<ApiResponse<Vec<Pekerja>>> {
    let peta = state.pekerja.read().await;
    let mut senarai: Vec<Pekerja> = peta.values().cloned().collect();

    // Filter
    if let Some(ref bah) = q.bahagian {
        senarai.retain(|p| &p.bahagian == bah);
    }
    if let Some(aktif) = q.aktif {
        senarai.retain(|p| p.aktif == aktif);
    }
    if let Some(ref cari) = q.cari {
        let cari_l = cari.to_lowercase();
        senarai.retain(|p| p.nama.to_lowercase().contains(&cari_l));
    }

    senarai.sort_by(|a, b| a.nama.cmp(&b.nama));
    Json(ApiResponse::ok(senarai))
}

async fn dapatkan_pekerja(
    State(state): State<AppState>,
    Path(id): Path<String>,
) -> AppResult<Pekerja> {
    let peta = state.pekerja.read().await;
    let p = peta.get(&id)
        .cloned()
        .ok_or_else(|| AppError::TidakJumpai(format!("Pekerja '{}' tidak dijumpai", id)))?;
    Ok(Json(ApiResponse::ok(p)))
}

async fn buat_pekerja(
    State(state): State<AppState>,
    Json(body): Json<BuatPekerja>,
) -> AppResult<Pekerja> {
    // Validate
    if body.nama.trim().is_empty() {
        return Err(AppError::Input("Nama tidak boleh kosong".into()));
    }

    let mut peta = state.pekerja.write().await;

    // Semak duplikat
    if peta.values().any(|p| p.no_pekerja == body.no_pekerja) {
        return Err(AppError::Konflik(format!(
            "No pekerja '{}' sudah wujud", body.no_pekerja
        )));
    }

    let pekerja = Pekerja {
        id:          Uuid::new_v4().to_string(),
        no_pekerja:  body.no_pekerja,
        nama:        body.nama,
        bahagian:    body.bahagian,
        gaji:        body.gaji,
        aktif:       true,
        dibuat_pada: sekarang(),
    };

    let id = pekerja.id.clone();
    peta.insert(id, pekerja.clone());

    Ok(Json(ApiResponse::ok(pekerja)))
}

async fn kemaskini_pekerja(
    State(state): State<AppState>,
    Path(id): Path<String>,
    Json(body): Json<KemaskiniPekerja>,
) -> AppResult<Pekerja> {
    let mut peta = state.pekerja.write().await;
    let pekerja = peta.get_mut(&id)
        .ok_or_else(|| AppError::TidakJumpai(format!("Pekerja '{}' tidak dijumpai", id)))?;

    if let Some(nama)     = body.nama     { pekerja.nama     = nama; }
    if let Some(bahagian) = body.bahagian { pekerja.bahagian = bahagian; }
    if let Some(gaji)     = body.gaji     { pekerja.gaji     = gaji; }
    if let Some(aktif)    = body.aktif    { pekerja.aktif    = aktif; }

    Ok(Json(ApiResponse::ok(pekerja.clone())))
}

async fn padam_pekerja(
    State(state): State<AppState>,
    Path(id): Path<String>,
) -> Result<(StatusCode, Json<ApiResponse<()>>), AppError> {
    let mut peta = state.pekerja.write().await;

    if peta.remove(&id).is_none() {
        return Err(AppError::TidakJumpai(format!("Pekerja '{}' tidak dijumpai", id)));
    }

    Ok((StatusCode::OK, Json(ApiResponse::mesej("Pekerja berjaya dipadam"))))
}

async fn handler_404() -> (StatusCode, Json<ApiResponse<()>>) {
    (StatusCode::NOT_FOUND, Json(ApiResponse::ralat("Endpoint tidak wujud")))
}

// ─── Main ──────────────────────────────────────────────────────

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_env_filter("info")
        .init();

    let state = AppState::baru();

    let app = Router::new()
        .route("/sihat",           get(sihat))
        .route("/api/v1/pekerja",  get(senarai_pekerja).post(buat_pekerja))
        .route("/api/v1/pekerja/:id",
               get(dapatkan_pekerja)
               .put(kemaskini_pekerja)
               .delete(padam_pekerja))
        .fallback(handler_404)
        .layer(middleware::from_fn(log_request))
        .with_state(state);

    let pendengar = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await.unwrap();

    println!("{'═'*50}");
    println!("  KADA API Server");
    println!("  http://localhost:3000");
    println!("{'═'*50}");
    println!("  GET  /sihat");
    println!("  GET  /api/v1/pekerja?bahagian=ICT&aktif=true");
    println!("  POST /api/v1/pekerja");
    println!("  GET  /api/v1/pekerja/:id");
    println!("  PUT  /api/v1/pekerja/:id");
    println!("  DEL  /api/v1/pekerja/:id");
    println!("{'═'*50}");

    axum::serve(pendengar, app).await.unwrap();
}
```

---

# 📋 Rujukan Pantas — Axum Cheat Sheet

## Routing

```rust
Router::new()
    .route("/path",     get(h).post(h).put(h).delete(h))
    .route("/:id",      get(h))              // path param
    .route("/*laluan",  get(h))              // wildcard
    .nest("/api", inner_router)              // prefix + strip
    .merge(other_router)                     // merge tanpa prefix
    .fallback(handler_404)                   // 404 handler
    .with_state(state)                       // inject state
    .layer(middleware)                       // tower middleware
```

## Extractors

```rust
Path(id): Path<u32>                          // /path/:id
Path((a, b)): Path<(String, u32)>           // /path/:a/:b
Query(q): Query<MyStruct>                    // ?key=val
Json(body): Json<MyStruct>                  // JSON body
Form(data): Form<MyStruct>                  // form data
State(state): State<AppState>               // shared state
Extension(ext): Extension<T>                // request extension
HeaderMap                                    // all headers
Method, Uri, Request                         // raw request
Bytes                                        // raw body
```

## Response

```rust
"string"                                     // text/plain
StatusCode::OK                               // status sahaja
(StatusCode, body)                           // status + body
Json(data)                                   // JSON response
Html("<h1>Hi</h1>")                         // HTML response
Redirect::to("/new")                         // redirect
impl IntoResponse                            // custom
```

## Middleware Order

```
DALAM → LUAR (layer yang kemudian = lebih luar)
Request masuk: Luar → Dalam → Handler
Response keluar: Handler → Dalam → Luar

.layer(A)   // dalam
.layer(B)   // tengah
.layer(C)   // luar — diproses PERTAMA untuk request
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu dengan Axum:

1. **Full CRUD API** — dengan SQLx, validation, pagination
2. **Auth System** — JWT, refresh token, logout
3. **File Upload Server** — multipart form, save ke disk
4. **Rate-Limited API** — dengan governor, per-IP limits
5. **Real-time Dashboard** — SSE + REST API dalam satu app

---

*Axum = kelajuan + keselamatan + ergonomics.*
*Tower ecosystem memberi ribuan middleware sedia pakai.*
*Type-safe extractors memastikan bug di-tangkap sebelum production.* 🦀
