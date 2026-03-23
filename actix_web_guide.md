# ⚡ Actix-web — Web Framework Rust Terpantas

> Panduan lengkap Actix-web: routing, extractors, middleware,
> state management, dan cara bina REST API yang production-ready.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Kenapa Actix-web?

```
Actix-web adalah antara framework web TERPANTAS di dunia:
  ✔ Consistently top dalam TechEmpower benchmarks
  ✔ Lebih laju dari Express (Node.js), FastAPI (Python), Spring (Java)
  ✔ Type-safe extractors
  ✔ Actor model (dari crate `actix`)
  ✔ WebSocket support
  ✔ Multipart/streaming support

Perbandingan dengan Axum:
  Axum:       Dari team Tokio, API lebih baru, guna Tower middleware
  Actix-web:  Lebih lama, ekosistem lebih besar, sedikit lebih laju
  Pilih mana? Kedua-dua excellent. Axum = newer API. Actix = battle-tested.
```

---

## Setup

```toml
[dependencies]
actix-web     = "4"
actix-rt      = "2"           # tokio runtime untuk actix
tokio         = { version = "1", features = ["full"] }
serde         = { version = "1", features = ["derive"] }
serde_json    = "1"
actix-cors    = "0.7"
actix-files   = "0.6"         # serve static files
actix-session = { version = "0.9", features = ["cookie-session"] }
actix-web-httpauth = "0.8"    # JWT/Basic auth helpers

# Optional (guna dengan SQLx)
sqlx          = { version = "0.7", features = ["mysql", "runtime-tokio-rustls"] }
```

---

## Peta Pembelajaran

```
Bahagian 1  → Hello World & Routing Asas
Bahagian 2  → Extractors — Baca Request
Bahagian 3  → Response — Hantar Balas
Bahagian 4  → App State — Data Kongsi
Bahagian 5  → Middleware
Bahagian 6  → Error Handling
Bahagian 7  → Route Grouping & Scope
Bahagian 8  → File Upload & Static Files
Bahagian 9  → WebSocket
Bahagian 10 → Mini Project: KADA REST API
```

---

# BAHAGIAN 1: Hello World & Routing Asas 🚀

## Server Paling Mudah

```rust
use actix_web::{web, App, HttpServer, HttpResponse, Responder};

// Handler — fungsi yang terima request dan return response
async fn salam() -> impl Responder {
    HttpResponse::Ok().body("Halo dari Actix-web! ⚡")
}

async fn salam_nama(path: web::Path<String>) -> impl Responder {
    let nama = path.into_inner();
    format!("Halo, {}!", nama)
}

#[actix_web::main]  // macro setup Tokio runtime
async fn main() -> std::io::Result<()> {
    println!("Server pada http://localhost:8080");

    HttpServer::new(|| {
        App::new()
            .route("/",           web::get().to(salam))
            .route("/salam/{nama}", web::get().to(salam_nama))
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

---

## Semua HTTP Methods

```rust
use actix_web::{web, App, HttpServer, HttpResponse};

async fn get_handler()    -> HttpResponse { HttpResponse::Ok().body("GET") }
async fn post_handler()   -> HttpResponse { HttpResponse::Created().body("POST") }
async fn put_handler()    -> HttpResponse { HttpResponse::Ok().body("PUT") }
async fn delete_handler() -> HttpResponse { HttpResponse::NoContent().finish() }
async fn patch_handler()  -> HttpResponse { HttpResponse::Ok().body("PATCH") }

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            // Cara 1: .route() — explicit method
            .route("/api/item",     web::get().to(get_handler))
            .route("/api/item",     web::post().to(post_handler))
            .route("/api/item/{id}", web::put().to(put_handler))
            .route("/api/item/{id}", web::delete().to(delete_handler))
            .route("/api/item/{id}", web::patch().to(patch_handler))

            // Cara 2: service() dengan web::resource() — lebih clean
            .service(
                web::resource("/api/pekerja")
                    .route(web::get().to(get_handler))
                    .route(web::post().to(post_handler))
            )
            .service(
                web::resource("/api/pekerja/{id}")
                    .route(web::get().to(get_handler))
                    .route(web::put().to(put_handler))
                    .route(web::delete().to(delete_handler))
            )
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

---

## Pattern Route Lanjutan

```rust
use actix_web::{web, App, HttpServer, HttpResponse, guard};

async fn handler() -> HttpResponse { HttpResponse::Ok().finish() }

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            // Path parameter
            .route("/pekerja/{id}", web::get().to(handler))

            // Multiple parameters
            .route("/org/{org_id}/pekerja/{id}", web::get().to(handler))

            // Wildcard
            .route("/fail/{laluan:.*}", web::get().to(handler))

            // Dengan guard — syarat tambahan
            .route("/api",
                web::get()
                    .guard(guard::Header("Accept", "application/json"))
                    .to(handler)
            )

            // Custom 404
            .default_service(web::route().to(|| async {
                HttpResponse::NotFound().json(serde_json::json!({
                    "ralat": "Halaman tidak dijumpai"
                }))
            }))
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

---

## 🧠 Brain Teaser #1

Apakah perbezaan antara `web::get().to(handler)` dan `.route("/path", web::get().to(handler))`?

<details>
<summary>👀 Jawapan</summary>

```
web::get().to(handler):
  - Route factory — buat konfigurasi route
  - Belum terikat kepada path
  - Digunakan DALAM .route() atau .service()

.route("/path", web::get().to(handler)):
  - Shortcut — terus daftar route dengan path
  - Equivalent kepada:
    .service(web::resource("/path").route(web::get().to(handler)))

.service(web::resource("/path")...):
  - Lebih flexible
  - Boleh tambah multiple methods untuk satu resource
  - Boleh tambah guards dan middleware pada resource
  - Recommended untuk production

Contoh equivalent:
  // Cara ringkas:
  .route("/pekerja", web::get().to(senarai))

  // Cara penuh (sama hasil):
  .service(
      web::resource("/pekerja")
          .route(web::get().to(senarai))
  )

  // Multiple methods:
  .service(
      web::resource("/pekerja")
          .route(web::get().to(senarai))
          .route(web::post().to(buat))
  )
```
</details>

---

# BAHAGIAN 2: Extractors — Baca Request 📥

## Path, Query, JSON, Form

```rust
use actix_web::{web, HttpRequest, HttpResponse};
use serde::{Serialize, Deserialize};

// ── Path Extractor ─────────────────────────────────────────────
async fn dengan_path(path: web::Path<u32>) -> HttpResponse {
    let id = path.into_inner();
    HttpResponse::Ok().body(format!("ID: {}", id))
}

// Multiple path params
async fn banyak_param(path: web::Path<(String, u32)>) -> HttpResponse {
    let (bahagian, id) = path.into_inner();
    HttpResponse::Ok().body(format!("{}/{}", bahagian, id))
}

// Menggunakan struct
#[derive(Deserialize)]
struct InfoJalur {
    id:       u32,
    bahagian: String,
}

async fn path_struct(info: web::Path<InfoJalur>) -> HttpResponse {
    HttpResponse::Ok().body(format!("{}: {}", info.bahagian, info.id))
}

// ── Query String Extractor ─────────────────────────────────────
#[derive(Deserialize)]
struct QuerySaring {
    cari:     Option<String>,
    halaman:  Option<u32>,
    saiz:     Option<u32>,
    aktif:    Option<bool>,
}

async fn dengan_query(query: web::Query<QuerySaring>) -> HttpResponse {
    let saring = query.into_inner();
    HttpResponse::Ok().json(serde_json::json!({
        "cari":    saring.cari,
        "halaman": saring.halaman.unwrap_or(1),
        "saiz":    saring.saiz.unwrap_or(20),
    }))
}

// ── JSON Body Extractor ───────────────────────────────────────
#[derive(Deserialize, Debug)]
struct BuatPekerja {
    nama:     String,
    bahagian: String,
    gaji:     f64,
}

async fn dari_json(body: web::Json<BuatPekerja>) -> HttpResponse {
    let p = body.into_inner();
    HttpResponse::Created().json(serde_json::json!({
        "mesej": format!("Pekerja {} dicipta", p.nama),
        "gaji":  p.gaji,
    }))
}

// ── Form Data Extractor ───────────────────────────────────────
#[derive(Deserialize)]
struct FormLogin {
    username: String,
    password: String,
}

async fn dari_form(form: web::Form<FormLogin>) -> HttpResponse {
    let data = form.into_inner();
    if data.username == "admin" && data.password == "rahsia" {
        HttpResponse::Ok().body("Login berjaya!")
    } else {
        HttpResponse::Unauthorized().body("Credentials tidak sah")
    }
}

// ── Headers ───────────────────────────────────────────────────
async fn dengan_header(req: HttpRequest) -> HttpResponse {
    let ua = req.headers()
        .get("User-Agent")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("Unknown");

    let auth = req.headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.strip_prefix("Bearer "));

    HttpResponse::Ok().json(serde_json::json!({
        "user_agent": ua,
        "ada_token": auth.is_some(),
    }))
}

// ── Raw Body ──────────────────────────────────────────────────
async fn raw_body(body: web::Bytes) -> HttpResponse {
    println!("Body: {} bytes", body.len());
    HttpResponse::Ok().body(format!("Terima {} bytes", body.len()))
}
```

---

## Custom Extractor

```rust
use actix_web::{web, FromRequest, HttpRequest, dev::Payload, Error};
use futures::future::{ready, Ready};
use std::future::Future;

// Custom extractor untuk ambil user dari JWT
#[derive(Debug, Clone)]
struct PenggunaSemasa {
    id:       u32,
    nama:     String,
    peranan:  Vec<String>,
}

impl FromRequest for PenggunaSemasa {
    type Error = actix_web::Error;
    type Future = Ready<Result<Self, Self::Error>>;

    fn from_request(req: &HttpRequest, _: &mut Payload) -> Self::Future {
        let token = req.headers()
            .get("Authorization")
            .and_then(|v| v.to_str().ok())
            .and_then(|s| s.strip_prefix("Bearer "));

        match token {
            Some("token-sah-contoh") => ready(Ok(PenggunaSemasa {
                id:      1,
                nama:    "Ali Ahmad".into(),
                peranan: vec!["admin".into()],
            })),
            _ => ready(Err(actix_web::error::ErrorUnauthorized("Token tidak sah"))),
        }
    }
}

// Guna dalam handler — jika token tidak sah, auto 401!
async fn profil(pengguna: PenggunaSemasa) -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({
        "id":   pengguna.id,
        "nama": pengguna.nama,
    }))
}
```

---

# BAHAGIAN 3: Response — Hantar Balas 📤

## Cara Hantar Berbagai Response

```rust
use actix_web::{HttpResponse, web, Responder};
use serde::Serialize;

// ── HttpResponse builder ──────────────────────────────────────
async fn contoh_responses() -> HttpResponse {
    // Status sahaja
    HttpResponse::Ok().finish()          // 200 kosong
    HttpResponse::Created().finish()     // 201 kosong
    HttpResponse::NoContent().finish()   // 204

    // Text
    HttpResponse::Ok().body("Teks biasa")

    // JSON
    HttpResponse::Ok().json(serde_json::json!({
        "berjaya": true,
        "data":    "nilai"
    }))

    // HTML
    HttpResponse::Ok()
        .content_type("text/html")
        .body("<h1>Hello!</h1>")
}

// ── Struct yang serialize ke JSON ─────────────────────────────
#[derive(Serialize)]
struct ApiResponse<T: Serialize> {
    berjaya: bool,
    data:    Option<T>,
    mesej:   Option<String>,
}

impl<T: Serialize> ApiResponse<T> {
    fn ok(data: T) -> Self {
        ApiResponse { berjaya: true, data: Some(data), mesej: None }
    }
}

impl ApiResponse<()> {
    fn mesej(m: &str) -> Self {
        ApiResponse { berjaya: true, data: None, mesej: Some(m.into()) }
    }
    fn ralat(m: &str) -> Self {
        ApiResponse { berjaya: false, data: None, mesej: Some(m.into()) }
    }
}

#[derive(Serialize)]
struct Pekerja { id: u32, nama: String, gaji: f64 }

async fn senarai_pekerja() -> impl Responder {
    let senarai = vec![
        Pekerja { id: 1, nama: "Ali".into(), gaji: 4500.0 },
        Pekerja { id: 2, nama: "Siti".into(), gaji: 3800.0 },
    ];
    HttpResponse::Ok().json(ApiResponse::ok(senarai))
}

// ── Custom headers ────────────────────────────────────────────
async fn dengan_headers() -> HttpResponse {
    HttpResponse::Ok()
        .insert_header(("X-Custom-Header", "nilai-saya"))
        .insert_header(("Cache-Control", "no-cache"))
        .json(serde_json::json!({"mesej": "ok"}))
}

// ── Redirect ─────────────────────────────────────────────────
async fn redirect() -> HttpResponse {
    HttpResponse::Found()
        .insert_header(("Location", "/halaman-baru"))
        .finish()
}

// ── Streaming response ────────────────────────────────────────
use actix_web::body::BodyStream;
use futures::stream;
use bytes::Bytes;

async fn stream_besar() -> HttpResponse {
    let data_stream = stream::iter(vec![
        Ok::<Bytes, std::io::Error>(Bytes::from("Bahagian 1...")),
        Ok(Bytes::from("Bahagian 2...")),
        Ok(Bytes::from("Bahagian 3...")),
    ]);

    HttpResponse::Ok()
        .content_type("text/plain")
        .streaming(data_stream)
}
```

---

# BAHAGIAN 4: App State — Data Kongsi 🏛️

## Shared State dengan web::Data

```rust
use actix_web::{web, App, HttpServer, HttpResponse};
use std::sync::{Arc, Mutex};
use std::collections::HashMap;
use serde::{Serialize, Deserialize};

// App state struct
#[derive(Clone)]
struct AppState {
    nama_app:  String,
    versi:     String,
    db_pool:   Arc<sqlx::MySqlPool>, // production — guna pool
    cache:     Arc<Mutex<HashMap<u32, Pekerja>>>,
}

#[derive(Clone, Serialize, Deserialize, Debug)]
struct Pekerja {
    id:   u32,
    nama: String,
    gaji: f64,
}

// Handler menggunakan state
async fn sihat(data: web::Data<AppState>) -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({
        "app":    data.nama_app,
        "versi":  data.versi,
        "status": "OK",
    }))
}

async fn senarai(state: web::Data<AppState>) -> HttpResponse {
    let cache = state.cache.lock().unwrap();
    let senarai: Vec<&Pekerja> = cache.values().collect();
    HttpResponse::Ok().json(senarai)
}

async fn dapatkan(
    state: web::Data<AppState>,
    path:  web::Path<u32>,
) -> HttpResponse {
    let id = path.into_inner();
    let cache = state.cache.lock().unwrap();
    match cache.get(&id) {
        Some(p) => HttpResponse::Ok().json(p),
        None    => HttpResponse::NotFound().json(serde_json::json!({
            "ralat": format!("Pekerja {} tidak dijumpai", id)
        })),
    }
}

async fn buat(
    state: web::Data<AppState>,
    body:  web::Json<Pekerja>,
) -> HttpResponse {
    let p = body.into_inner();
    let id = p.id;
    state.cache.lock().unwrap().insert(id, p);
    HttpResponse::Created().json(serde_json::json!({
        "mesej": format!("Pekerja {} dicipta", id)
    }))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // Inisialisasi state
    let mut cache_awal = HashMap::new();
    cache_awal.insert(1, Pekerja { id: 1, nama: "Ali".into(), gaji: 4500.0 });
    cache_awal.insert(2, Pekerja { id: 2, nama: "Siti".into(), gaji: 3800.0 });

    let state = web::Data::new(AppState {
        nama_app: "KADA API".into(),
        versi:    "1.0.0".into(),
        db_pool:  Arc::new(sqlx::MySqlPool::connect("mysql://localhost/db").await.unwrap()),
        cache:    Arc::new(Mutex::new(cache_awal)),
    });

    HttpServer::new(move || {
        App::new()
            .app_data(state.clone())  // inject state
            .route("/sihat", web::get().to(sihat))
            .route("/pekerja", web::get().to(senarai).post(buat))
            .route("/pekerja/{id}", web::get().to(dapatkan))
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

---

## State dengan RwLock (lebih cekap untuk read-heavy)

```rust
use std::sync::RwLock;

struct StateCekap {
    // RwLock: banyak pembaca ATAU satu penulis
    // Lebih cekap daripada Mutex kalau banyak read
    pekerja: RwLock<HashMap<u32, Pekerja>>,
}

async fn baca_cepat(state: web::Data<StateCekap>) -> HttpResponse {
    let pekerja = state.pekerja.read().unwrap(); // banyak boleh baca serentak!
    let senarai: Vec<&Pekerja> = pekerja.values().collect();
    HttpResponse::Ok().json(senarai)
}

async fn tulis_lambat(
    state: web::Data<StateCekap>,
    body:  web::Json<Pekerja>,
) -> HttpResponse {
    let p = body.into_inner();
    let id = p.id;
    state.pekerja.write().unwrap().insert(id, p); // satu sahaja boleh tulis
    HttpResponse::Created().json(serde_json::json!({"id": id}))
}
```

---

# BAHAGIAN 5: Middleware ⚙️

## Middleware dengan wrap_fn

```rust
use actix_web::{
    web, App, HttpServer, HttpResponse,
    dev::{ServiceRequest, ServiceResponse, Service, Transform},
    Error,
};
use actix_web::middleware::{Logger, NormalizePath};
use futures::future::{ok, Ready, LocalBoxFuture};
use std::time::Instant;

// ── Logger built-in ───────────────────────────────────────────
// actix-web ada Logger middleware yang sangat berguna!

fn bina_app() -> App<
    impl actix_web::dev::ServiceFactory<
        actix_web::dev::ServiceRequest,
        Config = (),
        Response = actix_web::dev::ServiceResponse,
        Error = actix_web::Error,
        InitError = (),
    >,
> {
    App::new()
        // Logger: format boleh customise
        .wrap(Logger::default())
        // atau custom format:
        .wrap(Logger::new("%r %s %b %T"))
        // Normalize trailing slash
        .wrap(NormalizePath::trim())
        .route("/", web::get().to(|| async { "OK" }))
}

// ── Custom middleware ─────────────────────────────────────────

pub struct MiddlewareMasa;

impl<S, B> Transform<S, ServiceRequest> for MiddlewareMasa
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response  = ServiceResponse<B>;
    type Error     = Error;
    type InitError = ();
    type Transform = MiddlewareMasaDalaman<S>;
    type Future    = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ok(MiddlewareMasaDalaman { service })
    }
}

pub struct MiddlewareMasaDalaman<S> {
    service: S,
}

impl<S, B> Service<ServiceRequest> for MiddlewareMasaDalaman<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error    = Error;
    type Future   = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    actix_web::dev::forward_ready!(service);

    fn call(&self, req: ServiceRequest) -> Self::Future {
        let mula = Instant::now();
        let kaedah = req.method().to_string();
        let uri    = req.uri().to_string();

        let fut = self.service.call(req);

        Box::pin(async move {
            let resp = fut.await?;
            println!("[{}ms] {} {}", mula.elapsed().as_millis(), kaedah, uri);
            Ok(resp)
        })
    }
}

// ── Middleware mudah dengan wrap_fn ───────────────────────────
// Cara lebih mudah untuk middleware simple!
use actix_web::dev::forward_ready;

async fn tambah_header_keselamatan(
    req: ServiceRequest,
    next: actix_web::dev::Next<impl actix_web::body::MessageBody + 'static>,
) -> Result<ServiceResponse<impl actix_web::body::MessageBody>, Error> {
    // Proses sebelum handler
    let mut resp = next.call(req).await?;
    // Proses selepas handler
    resp.headers_mut().insert(
        actix_web::http::header::HeaderName::from_static("x-frame-options"),
        actix_web::http::header::HeaderValue::from_static("DENY"),
    );
    Ok(resp)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    std::env::set_var("RUST_LOG", "actix_web=info");
    env_logger::init();

    HttpServer::new(|| {
        App::new()
            .wrap(Logger::default())         // log semua request
            .wrap(MiddlewareMasa)             // ukur masa
            .wrap_fn(|req, srv| {
                // Middleware ringkas inline
                println!("Request: {}", req.path());
                let fut = srv.call(req);
                async move {
                    let resp = fut.await?;
                    println!("Response: {}", resp.status());
                    Ok(resp)
                }
            })
            .route("/", web::get().to(|| async { "OK" }))
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

---

# BAHAGIAN 6: Error Handling 🚨

## Custom Error Types

```rust
use actix_web::{HttpResponse, ResponseError};
use thiserror::Error;
use serde::Serialize;

// Custom error type
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

    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("Ralat dalaman")]
    Dalaman(#[source] Box<dyn std::error::Error + Send + Sync>),
}

// Implement ResponseError — auto convert AppError ke HttpResponse!
impl ResponseError for AppError {
    fn error_response(&self) -> HttpResponse {
        let (status, mesej) = match self {
            AppError::TidakJumpai(m)   => (actix_web::http::StatusCode::NOT_FOUND, m.clone()),
            AppError::InputTidakSah(m) => (actix_web::http::StatusCode::UNPROCESSABLE_ENTITY, m.clone()),
            AppError::TidakDibenarkan  => (actix_web::http::StatusCode::UNAUTHORIZED, "Tidak dibenarkan".into()),
            AppError::Dilarang(m)      => (actix_web::http::StatusCode::FORBIDDEN, m.clone()),
            AppError::Database(_)      => (actix_web::http::StatusCode::INTERNAL_SERVER_ERROR, "Ralat pangkalan data".into()),
            AppError::Dalaman(_)       => (actix_web::http::StatusCode::INTERNAL_SERVER_ERROR, "Ralat dalaman".into()),
        };

        HttpResponse::build(status).json(serde_json::json!({
            "berjaya": false,
            "ralat":   mesej,
            "kod":     status.as_u16(),
        }))
    }
}

// Handler yang return Result dengan AppError
async fn dapatkan_pekerja_db(
    state: web::Data<AppState>,
    path:  web::Path<u32>,
) -> Result<HttpResponse, AppError> {
    let id = path.into_inner();

    if id == 0 {
        return Err(AppError::InputTidakSah("ID tidak boleh sifar".into()));
    }

    let cache = state.cache.lock().unwrap();
    let pekerja = cache.get(&id)
        .ok_or_else(|| AppError::TidakJumpai(format!("Pekerja {}", id)))?;

    Ok(HttpResponse::Ok().json(pekerja))
}

// Validation dengan error yang berguna
use validator::Validate;

#[derive(Deserialize, Validate)]
struct InputBaru {
    #[validate(length(min = 2, max = 100))]
    nama: String,

    #[validate(email)]
    emel: String,

    #[validate(range(min = 0.0))]
    gaji: f64,
}

async fn buat_dengan_validasi(
    body: web::Json<InputBaru>,
) -> Result<HttpResponse, AppError> {
    body.validate().map_err(|e| AppError::InputTidakSah(e.to_string()))?;

    // proses...
    Ok(HttpResponse::Created().json(serde_json::json!({"mesej": "Berjaya"})))
}
```

---

# BAHAGIAN 7: Route Grouping & Scope 📁

## Scope dan Konfigurasi Modular

```rust
use actix_web::{web, App, HttpServer, HttpResponse};

// Handler terpisah untuk setiap resource
mod handlers {
    pub mod pekerja {
        use actix_web::{web, HttpResponse};
        use serde::{Serialize, Deserialize};

        #[derive(Serialize, Deserialize, Clone)]
        pub struct Pekerja { pub id: u32, pub nama: String }

        pub async fn senarai() -> HttpResponse {
            HttpResponse::Ok().json(vec![
                Pekerja { id: 1, nama: "Ali".into() }
            ])
        }
        pub async fn dapatkan(path: web::Path<u32>) -> HttpResponse {
            HttpResponse::Ok().json(Pekerja { id: path.into_inner(), nama: "Ali".into() })
        }
        pub async fn buat(body: web::Json<Pekerja>) -> HttpResponse {
            HttpResponse::Created().json(body.into_inner())
        }
        pub async fn kemaskini(path: web::Path<u32>, body: web::Json<Pekerja>) -> HttpResponse {
            let mut p = body.into_inner();
            p.id = path.into_inner();
            HttpResponse::Ok().json(p)
        }
        pub async fn padam(path: web::Path<u32>) -> HttpResponse {
            let id = path.into_inner();
            HttpResponse::Ok().json(serde_json::json!({"dipadamkan": id}))
        }

        pub fn skop(cfg: &mut web::ServiceConfig) {
            cfg.service(
                web::resource("")
                    .route(web::get().to(senarai))
                    .route(web::post().to(buat))
            )
            .service(
                web::resource("/{id}")
                    .route(web::get().to(dapatkan))
                    .route(web::put().to(kemaskini))
                    .route(web::delete().to(padam))
            );
        }
    }

    pub mod kehadiran {
        use actix_web::{web, HttpResponse};

        pub async fn senarai() -> HttpResponse {
            HttpResponse::Ok().json(serde_json::json!({"data": []}))
        }
        pub async fn daftar_masuk() -> HttpResponse {
            HttpResponse::Created().json(serde_json::json!({"status": "masuk"}))
        }

        pub fn skop(cfg: &mut web::ServiceConfig) {
            cfg.service(
                web::resource("")
                    .route(web::get().to(senarai))
                    .route(web::post().to(daftar_masuk))
            );
        }
    }
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/sihat", web::get().to(|| async { "OK" }))

            // API v1
            .service(
                web::scope("/api/v1")
                    .configure(|cfg| {
                        cfg.service(web::scope("/pekerja").configure(handlers::pekerja::skop))
                           .service(web::scope("/kehadiran").configure(handlers::kehadiran::skop));
                    })
            )

            // API v2 (contoh versioning)
            .service(
                web::scope("/api/v2")
                    .service(web::scope("/pekerja").configure(handlers::pekerja::skop))
            )
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

---

# BAHAGIAN 8: File Upload & Static Files 📁

## Multipart & Static

```toml
actix-multipart = "0.6"
actix-files     = "0.6"
```

```rust
use actix_web::{web, App, HttpServer, HttpResponse};
use actix_multipart::Multipart;
use actix_files as fs;
use futures::StreamExt;
use std::io::Write;

// Serve static files
async fn setup_static() {
    // Serve direktori "static/" di bawah "/static/"
    // Guna dalam App::new():
    // .service(fs::Files::new("/static", "static/").show_files_listing())
    // .service(fs::Files::new("/", "frontend/dist/").index_file("index.html"))
}

// File upload
async fn muat_naik(mut payload: Multipart) -> HttpResponse {
    let mut nama_fail = String::new();
    let mut saiz_total = 0usize;

    while let Some(item) = payload.next().await {
        let mut field = match item {
            Ok(f)  => f,
            Err(e) => return HttpResponse::BadRequest().body(e.to_string()),
        };

        let content_type = field.content_disposition();
        let nama = content_type.get_filename()
            .map(String::from)
            .unwrap_or_else(|| "tidak_bernama".into());

        nama_fail = nama.clone();

        // Buat fail untuk simpan
        let laluan = format!("uploads/{}", nama);
        let mut fail = match std::fs::File::create(&laluan) {
            Ok(f)  => f,
            Err(e) => return HttpResponse::InternalServerError().body(e.to_string()),
        };

        while let Some(chunk) = field.next().await {
            let data = match chunk {
                Ok(d)  => d,
                Err(e) => return HttpResponse::BadRequest().body(e.to_string()),
            };
            saiz_total += data.len();
            if let Err(e) = fail.write_all(&data) {
                return HttpResponse::InternalServerError().body(e.to_string());
            }
        }
    }

    HttpResponse::Ok().json(serde_json::json!({
        "fail":  nama_fail,
        "saiz":  saiz_total,
        "mesej": "Muat naik berjaya"
    }))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    std::fs::create_dir_all("uploads").ok();

    HttpServer::new(|| {
        App::new()
            .route("/muat-naik", web::post().to(muat_naik))

            // Serve fail statik
            .service(fs::Files::new("/static", "static/")
                .show_files_listing()
                .use_last_modified(true)
            )

            // SPA fallback
            .service(fs::Files::new("/", "frontend/dist/")
                .index_file("index.html")
            )
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

---

# BAHAGIAN 9: WebSocket ⚡

## WebSocket dengan actix-web

```toml
actix-ws = "0.2"
```

```rust
use actix_web::{web, App, HttpServer, HttpRequest, HttpResponse};
use actix_ws::Message;

// WebSocket handler
async fn ws_handler(
    req:     HttpRequest,
    stream:  web::Payload,
) -> Result<HttpResponse, actix_web::Error> {
    let (resp, mut sesi, mut msg_strim) = actix_ws::handle(&req, stream)?;

    // Spawn task untuk tangani mesej
    actix_web::rt::spawn(async move {
        // Hantar selamat datang
        sesi.text("Selamat datang ke KADA WebSocket!").await.ok();

        // Loop terima mesej
        while let Some(Ok(msg)) = msg_strim.recv().await {
            match msg {
                Message::Text(teks) => {
                    println!("WS terima: {}", teks);
                    // Echo balik dengan prefix
                    sesi.text(format!("Echo: {}", teks)).await.ok();
                }
                Message::Binary(data) => {
                    println!("Binary: {} bytes", data.len());
                    sesi.binary(data).await.ok();
                }
                Message::Ping(data) => {
                    sesi.pong(&data).await.ok();
                }
                Message::Close(sebab) => {
                    println!("WS tutup: {:?}", sebab);
                    break;
                }
                _ => {}
            }
        }
    });

    Ok(resp)
}

// ── Broadcast WebSocket (chat room) ──────────────────────────
use std::sync::Mutex;
use actix_ws::Session;

struct BilikanChatState {
    peserta: Mutex<Vec<Session>>,
}

async fn ws_chat(
    req:     HttpRequest,
    stream:  web::Payload,
    state:   web::Data<BilikanChatState>,
) -> Result<HttpResponse, actix_web::Error> {
    let (resp, sesi, mut msg_strim) = actix_ws::handle(&req, stream)?;

    // Tambah peserta baru
    state.peserta.lock().unwrap().push(sesi);
    let idx = state.peserta.lock().unwrap().len() - 1;

    let state_klon = state.clone();

    actix_web::rt::spawn(async move {
        while let Some(Ok(msg)) = msg_strim.recv().await {
            if let Message::Text(teks) = msg {
                // Broadcast ke semua peserta
                let mut peserta = state_klon.peserta.lock().unwrap();
                let mesej = format!("Peserta {}: {}", idx + 1, teks);
                peserta.retain_mut(|s| {
                    actix_web::rt::block_on(s.text(mesej.clone())).is_ok()
                });
            }
        }
    });

    Ok(resp)
}
```

---

# BAHAGIAN 10: Mini Project — KADA REST API 🏗️

```rust
use actix_web::{
    web, App, HttpServer, HttpResponse,
    middleware::Logger,
};
use serde::{Serialize, Deserialize};
use std::sync::RwLock;
use std::collections::HashMap;
use std::sync::Arc;
use uuid::Uuid;
use chrono::Utc;
use thiserror::Error;
use validator::Validate;

// ─── Models ────────────────────────────────────────────────────

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Pekerja {
    id:          String,
    no_pekerja:  String,
    nama:        String,
    bahagian:    String,
    gaji:        f64,
    aktif:       bool,
    dicipta_pada: String,
}

#[derive(Debug, Deserialize, Validate, Clone)]
struct BuatPekerja {
    #[validate(length(min = 4, max = 20, message = "No pekerja 4-20 aksara"))]
    no_pekerja: String,

    #[validate(length(min = 2, max = 100, message = "Nama 2-100 aksara"))]
    nama: String,

    #[validate(length(min = 1, message = "Bahagian diperlukan"))]
    bahagian: String,

    #[validate(range(min = 0.0, message = "Gaji tidak boleh negatif"))]
    gaji: f64,
}

#[derive(Debug, Deserialize, Validate)]
struct KemaskiniPekerja {
    nama:     Option<String>,
    bahagian: Option<String>,
    gaji:     Option<f64>,
    aktif:    Option<bool>,
}

#[derive(Debug, Deserialize, Default)]
struct QuerySaring {
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
    fn ok(data: T) -> Self { ApiResponse { berjaya: true, data: Some(data), mesej: None } }
}

impl ApiResponse<()> {
    fn mesej(m: &str) -> Self { ApiResponse { berjaya: true, data: None, mesej: Some(m.into()) } }
    fn ralat(m: &str) -> Self { ApiResponse { berjaya: false, data: None, mesej: Some(m.into()) } }
}

// ─── Error ─────────────────────────────────────────────────────

#[derive(Debug, Error)]
enum AppError {
    #[error("Tidak dijumpai: {0}")]
    TidakJumpai(String),
    #[error("Konflik: {0}")]
    Konflik(String),
    #[error("Input tidak sah: {0}")]
    Input(String),
}

impl actix_web::ResponseError for AppError {
    fn error_response(&self) -> HttpResponse {
        let (status, mesej) = match self {
            AppError::TidakJumpai(m) => (actix_web::http::StatusCode::NOT_FOUND, m.clone()),
            AppError::Konflik(m)     => (actix_web::http::StatusCode::CONFLICT, m.clone()),
            AppError::Input(m)       => (actix_web::http::StatusCode::UNPROCESSABLE_ENTITY, m.clone()),
        };
        HttpResponse::build(status).json(ApiResponse::<()>::ralat(&mesej))
    }
}

// ─── State ─────────────────────────────────────────────────────

#[derive(Clone)]
struct AppState {
    pekerja: Arc<RwLock<HashMap<String, Pekerja>>>,
}

impl AppState {
    fn baru() -> Self {
        let mut peta = HashMap::new();
        for (no, nama, bah, gaji) in [
            ("KADA001", "Ali Ahmad",  "ICT",      4500.0),
            ("KADA002", "Siti Hawa",  "HR",        3800.0),
            ("KADA003", "Amin Razak", "ICT",       5000.0),
        ] {
            let id = Uuid::new_v4().to_string();
            peta.insert(id.clone(), Pekerja {
                id, no_pekerja: no.into(), nama: nama.into(),
                bahagian: bah.into(), gaji, aktif: true,
                dicipta_pada: Utc::now().to_rfc3339(),
            });
        }
        AppState { pekerja: Arc::new(RwLock::new(peta)) }
    }
}

// ─── Handlers ──────────────────────────────────────────────────

async fn sihat(state: web::Data<AppState>) -> HttpResponse {
    let bil = state.pekerja.read().unwrap().len();
    HttpResponse::Ok().json(serde_json::json!({
        "status":          "OK",
        "servis":          "KADA API",
        "versi":           "2.0.0",
        "jumlah_pekerja":  bil,
    }))
}

async fn senarai_pekerja(
    state:  web::Data<AppState>,
    query:  web::Query<QuerySaring>,
) -> HttpResponse {
    let saring = query.into_inner();
    let peta = state.pekerja.read().unwrap();
    let mut senarai: Vec<Pekerja> = peta.values().cloned().collect();

    if let Some(ref bah) = saring.bahagian {
        senarai.retain(|p| p.bahagian.to_lowercase() == bah.to_lowercase());
    }
    if let Some(aktif) = saring.aktif {
        senarai.retain(|p| p.aktif == aktif);
    }
    if let Some(ref cari) = saring.cari {
        let cari_l = cari.to_lowercase();
        senarai.retain(|p| p.nama.to_lowercase().contains(&cari_l)
            || p.no_pekerja.to_lowercase().contains(&cari_l));
    }

    senarai.sort_by(|a, b| a.nama.cmp(&b.nama));
    HttpResponse::Ok().json(ApiResponse::ok(senarai))
}

async fn dapatkan_pekerja(
    state: web::Data<AppState>,
    path:  web::Path<String>,
) -> Result<HttpResponse, AppError> {
    let id = path.into_inner();
    let peta = state.pekerja.read().unwrap();
    let p = peta.get(&id)
        .cloned()
        .ok_or_else(|| AppError::TidakJumpai(format!("Pekerja '{}' tidak dijumpai", id)))?;
    Ok(HttpResponse::Ok().json(ApiResponse::ok(p)))
}

async fn buat_pekerja(
    state: web::Data<AppState>,
    body:  web::Json<BuatPekerja>,
) -> Result<HttpResponse, AppError> {
    let baru = body.into_inner();

    baru.validate().map_err(|e| AppError::Input(e.to_string()))?;

    let mut peta = state.pekerja.write().unwrap();

    if peta.values().any(|p| p.no_pekerja == baru.no_pekerja) {
        return Err(AppError::Konflik(
            format!("No pekerja '{}' sudah wujud", baru.no_pekerja)
        ));
    }

    let id = Uuid::new_v4().to_string();
    let pekerja = Pekerja {
        id:          id.clone(),
        no_pekerja:  baru.no_pekerja,
        nama:        baru.nama,
        bahagian:    baru.bahagian,
        gaji:        baru.gaji,
        aktif:       true,
        dicipta_pada: Utc::now().to_rfc3339(),
    };

    peta.insert(id, pekerja.clone());
    Ok(HttpResponse::Created().json(ApiResponse::ok(pekerja)))
}

async fn kemaskini_pekerja(
    state: web::Data<AppState>,
    path:  web::Path<String>,
    body:  web::Json<KemaskiniPekerja>,
) -> Result<HttpResponse, AppError> {
    let id  = path.into_inner();
    let ubah = body.into_inner();
    let mut peta = state.pekerja.write().unwrap();

    let pekerja = peta.get_mut(&id)
        .ok_or_else(|| AppError::TidakJumpai(format!("Pekerja '{}' tidak dijumpai", id)))?;

    if let Some(nama)     = ubah.nama     { pekerja.nama     = nama; }
    if let Some(bahagian) = ubah.bahagian { pekerja.bahagian = bahagian; }
    if let Some(gaji)     = ubah.gaji     { pekerja.gaji     = gaji; }
    if let Some(aktif)    = ubah.aktif    { pekerja.aktif    = aktif; }

    Ok(HttpResponse::Ok().json(ApiResponse::ok(pekerja.clone())))
}

async fn padam_pekerja(
    state: web::Data<AppState>,
    path:  web::Path<String>,
) -> Result<HttpResponse, AppError> {
    let id = path.into_inner();
    let mut peta = state.pekerja.write().unwrap();

    peta.remove(&id)
        .ok_or_else(|| AppError::TidakJumpai(format!("Pekerja '{}' tidak dijumpai", id)))?;

    Ok(HttpResponse::Ok().json(ApiResponse::<()>::mesej("Pekerja berjaya dipadam")))
}

// ─── Main ──────────────────────────────────────────────────────

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    std::env::set_var("RUST_LOG", "actix_web=info,info");
    env_logger::init();

    let state = web::Data::new(AppState::baru());

    println!("{'═'*55}");
    println!("{:^55}", "KADA REST API — Actix-web");
    println!("{'═'*55}");
    println!("  http://localhost:8080");
    println!();
    println!("  GET    /sihat");
    println!("  GET    /api/v1/pekerja");
    println!("  POST   /api/v1/pekerja");
    println!("  GET    /api/v1/pekerja/:id");
    println!("  PUT    /api/v1/pekerja/:id");
    println!("  DELETE /api/v1/pekerja/:id");
    println!("{'═'*55}");

    HttpServer::new(move || {
        let cors = actix_cors::Cors::default()
            .allowed_origin("http://localhost:3000")
            .allowed_methods(vec!["GET", "POST", "PUT", "DELETE"])
            .allowed_headers(vec![
                actix_web::http::header::CONTENT_TYPE,
                actix_web::http::header::AUTHORIZATION,
            ])
            .max_age(3600);

        App::new()
            .app_data(state.clone())
            .app_data(web::JsonConfig::default()
                .error_handler(|err, _| {
                    let mesej = format!("JSON tidak sah: {}", err);
                    actix_web::error::InternalError::from_response(
                        err,
                        HttpResponse::BadRequest().json(ApiResponse::<()>::ralat(&mesej))
                    ).into()
                })
            )
            .wrap(Logger::new("%r %s %b %T"))
            .wrap(cors)
            .route("/sihat", web::get().to(sihat))
            .service(
                web::scope("/api/v1")
                    .service(
                        web::resource("/pekerja")
                            .route(web::get().to(senarai_pekerja))
                            .route(web::post().to(buat_pekerja))
                    )
                    .service(
                        web::resource("/pekerja/{id}")
                            .route(web::get().to(dapatkan_pekerja))
                            .route(web::put().to(kemaskini_pekerja))
                            .route(web::delete().to(padam_pekerja))
                    )
            )
            .default_service(web::route().to(|| async {
                HttpResponse::NotFound().json(ApiResponse::<()>::ralat("Endpoint tidak wujud"))
            }))
    })
    .workers(4)              // bilangan worker threads
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

---

# 📋 Rujukan Pantas — Actix-web Cheat Sheet

## Setup

```toml
actix-web  = "4"
actix-cors = "0.7"
```

## Routing

```rust
App::new()
    .route("/path",     web::get().to(handler))
    .route("/path",     web::post().to(handler))
    .route("/path/{id}", web::get().to(handler))
    .service(web::resource("/path")
        .route(web::get().to(handler))
        .route(web::post().to(handler)))
    .service(web::scope("/api")
        .service(...))
    .configure(config_fn)
    .default_service(web::route().to(not_found))
```

## Extractors

```rust
path:  web::Path<u32>          // /path/{id}
path:  web::Path<(u32, String)> // /path/{id}/{name}
query: web::Query<MyStruct>    // ?key=val
body:  web::Json<MyStruct>     // JSON body
form:  web::Form<MyStruct>     // form data
bytes: web::Bytes              // raw body
req:   HttpRequest             // full request
state: web::Data<AppState>     // shared state
```

## Response

```rust
HttpResponse::Ok().finish()
HttpResponse::Ok().body("text")
HttpResponse::Ok().json(data)
HttpResponse::Created().json(data)
HttpResponse::NotFound().json(error)
HttpResponse::Found().insert_header(("Location", "/")).finish()
```

## Actix vs Axum — Ringkas

```
Actix-web:
  ✔ Lebih laju (sedikit)
  ✔ Ekosistem lama, lebih banyak crate
  ✔ ResponseError trait untuk error handling
  ✔ actix-cors, actix-session, actix-files
  ✗ API lebih "magic" — sukar debug
  ✗ HttpServer, App, App::new() struktur unik

Axum:
  ✔ API lebih explicit, Tower ecosystem
  ✔ Lebih easy untuk reasoning
  ✔ Dari team Tokio — first-class async support
  ✗ Sedikit lebih lambat (tiada perbezaan dalam praktik)
  ✗ Lebih baru — sedikit less battle-tested
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu dengan Actix-web:

1. **Auth Middleware** — JWT verification, extract claims, role-based access
2. **Rate Limiter** — had request per IP menggunakan `actix-limitation`
3. **WebSocket Chat** — broadcast mesej ke semua klien yang connected
4. **File Server** — upload gambar, serve dengan resize on-the-fly
5. **GraphQL API** — guna `async-graphql` dengan Actix-web

---

*Actix-web = kelajuan + keselamatan + ergonomics.*
*Antara framework web terpantas di dunia — dibina dengan Rust.* 🦀
