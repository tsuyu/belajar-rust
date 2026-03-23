# 🏗️ Cadangan Struktur Aplikasi Rust — Panduan Lengkap

> Struktur projek yang betul dari awal jimat masa kemudian.
> Panduan untuk REST API, Web App, CLI, dan Workspace.
> Termasuk Cargo.toml, konvensyen, dan anti-patterns.

---

## Prinsip Asas Struktur Rust

```
1. FLAT IS BETTER THAN NESTED
   Jangan buat hierarki modul yang terlalu dalam.
   Modul patut mencerminkan domain, bukan teknologi.

2. SEPARATE CONCERNS
   handlers/ → HTTP layer (terima request, hantar response)
   services/ → Business logic (aturan perniagaan)
   models/   → Data structures
   db/       → Database access
   config/   → Konfigurasi

3. DEPENDENCY DIRECTION
   handlers → services → db
   (lapisan atas bergantung pada lapisan bawah, bukan sebaliknya)

4. ERROR TYPES EARLY
   Definisikan AppError dari awal.
   Semua fungsi return Result<T, AppError>.
```

---

# STRUKTUR 1: REST API — Axum + SQLx 🌐

## Untuk Projek Sederhana (Satu Crate)

```
kada-api/
├── Cargo.toml
├── .env
├── .env.example
├── .gitignore
├── README.md
│
├── migrations/                    ← SQL migrations
│   ├── 001_buat_jadual_pekerja.sql
│   ├── 002_buat_jadual_kehadiran.sql
│   └── 003_tambah_indeks.sql
│
├── src/
│   ├── main.rs                    ← entry point, setup server
│   ├── lib.rs                     ← (optional) untuk testing
│   │
│   ├── config.rs                  ← baca env variables
│   ├── error.rs                   ← AppError + IntoResponse
│   ├── state.rs                   ← AppState (db pool, redis, dll)
│   │
│   ├── models/                    ← data structures
│   │   ├── mod.rs
│   │   ├── pekerja.rs
│   │   └── kehadiran.rs
│   │
│   ├── db/                        ← database queries
│   │   ├── mod.rs
│   │   ├── pekerja.rs
│   │   └── kehadiran.rs
│   │
│   ├── handlers/                  ← HTTP handlers (thin layer!)
│   │   ├── mod.rs
│   │   ├── pekerja.rs
│   │   └── kehadiran.rs
│   │
│   ├── services/                  ← business logic
│   │   ├── mod.rs
│   │   ├── auth.rs
│   │   └── kehadiran.rs
│   │
│   ├── middleware/                ← axum middleware
│   │   ├── mod.rs
│   │   ├── auth.rs
│   │   └── logging.rs
│   │
│   └── routes.rs                  ← semua routes dalam satu tempat
│
└── tests/
    ├── common/
    │   └── mod.rs                 ← test helpers
    ├── test_pekerja.rs
    └── test_kehadiran.rs
```

---

## Cargo.toml — REST API

```toml
[package]
name    = "kada-api"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "kada-api"
path = "src/main.rs"

[[bin]]
name = "migrate"
path = "src/bin/migrate.rs"

[dependencies]
# Async runtime
tokio    = { version = "1", features = ["full"] }

# Web framework
axum           = "0.7"
tower          = "0.4"
tower-http     = { version = "0.5", features = ["cors", "trace", "timeout"] }

# Database
sqlx = { version = "0.7", features = [
    "runtime-tokio-rustls",
    "mysql",           # atau "postgres"
    "macros",          # query! macro
    "chrono",          # DateTime support
    "uuid",
] }

# Cache
redis         = { version = "0.25", features = ["tokio-comp"] }
deadpool-redis = "0.15"

# Serialization
serde      = { version = "1", features = ["derive"] }
serde_json = "1"

# Authentication
jsonwebtoken = "9"
bcrypt       = "0.15"

# Validation
validator = { version = "0.18", features = ["derive"] }

# Error handling
thiserror = "1"
anyhow    = "1"

# Config
dotenvy = "0.15"
config  = "0.14"

# Logging
tracing            = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }

# Utilities
uuid    = { version = "1", features = ["v4", "serde"] }
chrono  = { version = "0.4", features = ["serde"] }

[dev-dependencies]
tokio-test  = "0.4"
axum-test   = "0.1"
sqlx        = { version = "0.7", features = ["test"] }
```

---

## Fail-fail Utama

```rust
// src/config.rs
use serde::Deserialize;

#[derive(Debug, Deserialize, Clone)]
pub struct Config {
    pub host:       String,
    pub port:       u16,
    pub database_url: String,
    pub redis_url:  String,
    pub jwt_secret: String,
    pub log_level:  String,
}

impl Config {
    pub fn dari_env() -> Result<Self, config::ConfigError> {
        dotenvy::dotenv().ok();

        config::Config::builder()
            .set_default("host",      "0.0.0.0")?
            .set_default("port",      8080)?
            .set_default("log_level", "info")?
            .add_source(config::Environment::default())
            .build()?
            .try_deserialize()
    }
}
```

```rust
// src/error.rs
use axum::{http::StatusCode, response::{IntoResponse, Json}};
use serde_json::json;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("Tidak dijumpai: {0}")]
    TidakJumpai(String),

    #[error("Tidak dibenarkan")]
    TidakDibenarkan,

    #[error("Input tidak sah: {0}")]
    InputTidakSah(String),

    #[error("Konflik: {0}")]
    Konflik(String),

    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("Redis error: {0}")]
    Redis(#[from] redis::RedisError),

    #[error("Ralat dalaman: {0}")]
    Dalaman(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> axum::response::Response {
        let (status, mesej) = match &self {
            AppError::TidakJumpai(m)    => (StatusCode::NOT_FOUND, m.clone()),
            AppError::TidakDibenarkan   => (StatusCode::UNAUTHORIZED,
                                            "Tidak dibenarkan".into()),
            AppError::InputTidakSah(m)  => (StatusCode::UNPROCESSABLE_ENTITY, m.clone()),
            AppError::Konflik(m)        => (StatusCode::CONFLICT, m.clone()),
            AppError::Database(e)       => {
                tracing::error!("DB error: {}", e);
                (StatusCode::INTERNAL_SERVER_ERROR, "Ralat pangkalan data".into())
            }
            AppError::Redis(e)          => {
                tracing::error!("Redis error: {}", e);
                (StatusCode::INTERNAL_SERVER_ERROR, "Ralat cache".into())
            }
            AppError::Dalaman(m)        => {
                tracing::error!("Ralat dalaman: {}", m);
                (StatusCode::INTERNAL_SERVER_ERROR, "Ralat dalaman".into())
            }
        };

        (status, Json(json!({ "ralat": mesej, "kod": status.as_u16() }))).into_response()
    }
}

pub type AppResult<T> = Result<T, AppError>;
```

```rust
// src/state.rs
use sqlx::MySqlPool;
use deadpool_redis::Pool as RedisPool;
use crate::config::Config;

#[derive(Clone)]
pub struct AppState {
    pub db:     MySqlPool,
    pub redis:  RedisPool,
    pub config: Config,
}

impl AppState {
    pub async fn baru(config: Config) -> Result<Self, Box<dyn std::error::Error>> {
        // Database pool
        let db = sqlx::MySqlPool::connect(&config.database_url).await?;

        // Redis pool
        let redis = deadpool_redis::Config::from_url(&config.redis_url)
            .create_pool(Some(deadpool_redis::Runtime::Tokio1))?;

        Ok(AppState { db, redis, config })
    }
}
```

```rust
// src/models/pekerja.rs
use serde::{Serialize, Deserialize};
use sqlx::FromRow;
use uuid::Uuid;
use chrono::NaiveDateTime;
use validator::Validate;

// ── DB Model (dari database) ──────────────────────────────────
#[derive(Debug, Clone, Serialize, FromRow)]
pub struct Pekerja {
    pub id:         u32,
    pub uuid:       String,
    pub no_pekerja: String,
    pub nama:       String,
    pub bahagian:   String,
    pub gaji:       f64,
    pub aktif:      bool,
    pub dibuat_pada: NaiveDateTime,
}

// ── Request DTO (dari klien) ──────────────────────────────────
#[derive(Debug, Deserialize, Validate)]
pub struct BuatPekerja {
    #[validate(length(min = 1, max = 20, message = "No pekerja 1-20 aksara"))]
    pub no_pekerja: String,

    #[validate(length(min = 2, max = 100, message = "Nama 2-100 aksara"))]
    pub nama:       String,

    #[validate(length(min = 1, message = "Bahagian diperlukan"))]
    pub bahagian:   String,

    #[validate(range(min = 0.0, message = "Gaji tidak boleh negatif"))]
    pub gaji:       f64,
}

// ── Response DTO (ke klien) ───────────────────────────────────
#[derive(Debug, Serialize)]
pub struct PekerjaResponse {
    pub id:         u32,
    pub no_pekerja: String,
    pub nama:       String,
    pub bahagian:   String,
    pub gaji:       f64,
    pub aktif:      bool,
}

impl From<Pekerja> for PekerjaResponse {
    fn from(p: Pekerja) -> Self {
        PekerjaResponse {
            id:         p.id,
            no_pekerja: p.no_pekerja,
            nama:       p.nama,
            bahagian:   p.bahagian,
            gaji:       p.gaji,
            aktif:      p.aktif,
        }
    }
}

// ── Generic API Response ──────────────────────────────────────
#[derive(Debug, Serialize)]
pub struct ApiResponse<T: Serialize> {
    pub berjaya: bool,
    pub data:    Option<T>,
    pub mesej:   Option<String>,
}

impl<T: Serialize> ApiResponse<T> {
    pub fn ok(data: T) -> Self {
        ApiResponse { berjaya: true, data: Some(data), mesej: None }
    }
}

impl ApiResponse<()> {
    pub fn mesej(mesej: &str) -> Self {
        ApiResponse { berjaya: true, data: None, mesej: Some(mesej.into()) }
    }
}
```

```rust
// src/db/pekerja.rs
use sqlx::MySqlPool;
use crate::{error::AppResult, models::pekerja::*};

pub async fn senarai(
    db: &MySqlPool,
    bahagian: Option<&str>,
    had: u32,
    offset: u32,
) -> AppResult<Vec<Pekerja>> {
    let pekerja = if let Some(b) = bahagian {
        sqlx::query_as!(
            Pekerja,
            "SELECT * FROM pekerja WHERE bahagian = ? AND aktif = 1 LIMIT ? OFFSET ?",
            b, had, offset
        )
        .fetch_all(db)
        .await?
    } else {
        sqlx::query_as!(
            Pekerja,
            "SELECT * FROM pekerja WHERE aktif = 1 LIMIT ? OFFSET ?",
            had, offset
        )
        .fetch_all(db)
        .await?
    };

    Ok(pekerja)
}

pub async fn dapatkan_satu(db: &MySqlPool, id: u32) -> AppResult<Pekerja> {
    sqlx::query_as!(
        Pekerja,
        "SELECT * FROM pekerja WHERE id = ?",
        id
    )
    .fetch_optional(db)
    .await?
    .ok_or_else(|| crate::error::AppError::TidakJumpai(
        format!("Pekerja ID {} tidak dijumpai", id)
    ))
}

pub async fn simpan(db: &MySqlPool, data: &BuatPekerja) -> AppResult<u32> {
    // Semak duplikat
    let ada: bool = sqlx::query_scalar!(
        "SELECT EXISTS(SELECT 1 FROM pekerja WHERE no_pekerja = ?)",
        data.no_pekerja
    )
    .fetch_one(db)
    .await?
    .map(|v| v > 0)
    .unwrap_or(false);

    if ada {
        return Err(crate::error::AppError::Konflik(
            format!("No pekerja '{}' sudah wujud", data.no_pekerja)
        ));
    }

    let id = sqlx::query!(
        "INSERT INTO pekerja (no_pekerja, nama, bahagian, gaji) VALUES (?, ?, ?, ?)",
        data.no_pekerja, data.nama, data.bahagian, data.gaji
    )
    .execute(db)
    .await?
    .last_insert_id();

    Ok(id as u32)
}
```

```rust
// src/handlers/pekerja.rs
use axum::{
    extract::{State, Path, Query, Json},
    http::StatusCode,
    response::IntoResponse,
};
use serde::Deserialize;
use validator::Validate;

use crate::{
    state::AppState,
    error::{AppError, AppResult},
    models::pekerja::*,
    db,
};

#[derive(Debug, Deserialize)]
pub struct QueryParam {
    pub bahagian: Option<String>,
    pub halaman:  Option<u32>,
    pub saiz:     Option<u32>,
}

pub async fn senarai(
    State(state): State<AppState>,
    Query(q): Query<QueryParam>,
) -> AppResult<impl IntoResponse> {
    let halaman = q.halaman.unwrap_or(1).max(1);
    let saiz    = q.saiz.unwrap_or(20).min(100);
    let offset  = (halaman - 1) * saiz;

    let pekerja = db::pekerja::senarai(
        &state.db,
        q.bahagian.as_deref(),
        saiz,
        offset,
    ).await?;

    let response: Vec<PekerjaResponse> = pekerja.into_iter()
        .map(PekerjaResponse::from)
        .collect();

    Ok((StatusCode::OK, Json(ApiResponse::ok(response))))
}

pub async fn dapatkan(
    State(state): State<AppState>,
    Path(id): Path<u32>,
) -> AppResult<impl IntoResponse> {
    let pekerja = db::pekerja::dapatkan_satu(&state.db, id).await?;
    Ok((StatusCode::OK, Json(ApiResponse::ok(PekerjaResponse::from(pekerja)))))
}

pub async fn buat(
    State(state): State<AppState>,
    Json(body): Json<BuatPekerja>,
) -> AppResult<impl IntoResponse> {
    // Validate input
    body.validate()
        .map_err(|e| AppError::InputTidakSah(e.to_string()))?;

    let id = db::pekerja::simpan(&state.db, &body).await?;
    let pekerja = db::pekerja::dapatkan_satu(&state.db, id).await?;

    Ok((StatusCode::CREATED, Json(ApiResponse::ok(PekerjaResponse::from(pekerja)))))
}
```

```rust
// src/routes.rs
use axum::{Router, routing::{get, post, put, delete}};
use crate::{handlers, state::AppState, middleware};

pub fn bina_router(state: AppState) -> Router {
    let laluan_awam = Router::new()
        .route("/sihat", get(handlers::am::sihat));

    let laluan_api = Router::new()
        .route("/pekerja",      get(handlers::pekerja::senarai)
                                .post(handlers::pekerja::buat))
        .route("/pekerja/:id",  get(handlers::pekerja::dapatkan)
                                .put(handlers::pekerja::kemaskini)
                                .delete(handlers::pekerja::padam))
        .route("/kehadiran",    get(handlers::kehadiran::senarai)
                                .post(handlers::kehadiran::daftar))
        .layer(axum::middleware::from_fn_with_state(
            state.clone(),
            middleware::auth::semak_jwt,
        ));

    Router::new()
        .merge(laluan_awam)
        .nest("/api/v1", laluan_api)
        .with_state(state)
}
```

```rust
// src/main.rs
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

mod config;
mod error;
mod state;
mod models;
mod db;
mod handlers;
mod services;
mod middleware;
mod routes;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Config
    let config = config::Config::dari_env()?;

    // Logging
    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| "info".into()))
        .with(tracing_subscriber::fmt::layer())
        .init();

    tracing::info!("Memulakan {} v{}", env!("CARGO_PKG_NAME"), env!("CARGO_PKG_VERSION"));

    // State
    let state = state::AppState::baru(config.clone()).await?;

    // Router
    let app = routes::bina_router(state)
        .layer(tower_http::cors::CorsLayer::permissive())
        .layer(tower_http::trace::TraceLayer::new_for_http());

    let alamat = format!("{}:{}", config.host, config.port);
    let pendengar = tokio::net::TcpListener::bind(&alamat).await?;

    tracing::info!("Server berjalan pada http://{}", alamat);
    axum::serve(pendengar, app).await?;

    Ok(())
}
```

---

# STRUKTUR 2: Web App — Axum + Askama + SQLx 🖥️

```
kada-web/
├── Cargo.toml
├── .env
├── .gitignore
│
├── migrations/
│   └── 001_init.sql
│
├── static/                        ← fail statik
│   ├── css/
│   │   └── app.css
│   ├── js/
│   │   └── app.js
│   └── images/
│
├── templates/                     ← Askama templates
│   ├── asas.html                  ← base template
│   ├── _macros.html               ← reusable macros
│   ├── halaman_utama.html
│   ├── auth/
│   │   ├── log_masuk.html
│   │   └── daftar.html
│   └── pekerja/
│       ├── senarai.html
│       ├── profil.html
│       └── borang.html
│
└── src/
    ├── main.rs
    ├── config.rs
    ├── error.rs
    ├── state.rs
    ├── filters.rs                 ← custom Askama filters
    │
    ├── models/
    │   ├── mod.rs
    │   └── pekerja.rs
    │
    ├── db/
    │   ├── mod.rs
    │   └── pekerja.rs
    │
    ├── handlers/
    │   ├── mod.rs
    │   ├── auth.rs                ← login, logout, register
    │   ├── pekerja.rs
    │   └── api/                   ← JSON API endpoints
    │       └── mod.rs
    │
    ├── services/
    │   ├── mod.rs
    │   ├── auth.rs
    │   └── sesi.rs
    │
    └── routes.rs
```

---

# STRUKTUR 3: CLI Tool 💻

```
kada-cli/
├── Cargo.toml
├── .env.example
├── README.md
│
└── src/
    ├── main.rs                    ← entry point, setup clap
    │
    ├── cli/                       ← CLI argument definitions
    │   ├── mod.rs
    │   └── args.rs
    │
    ├── commands/                  ← satu fail per subcommand
    │   ├── mod.rs
    │   ├── import.rs              ← `kada-cli import pekerja.csv`
    │   ├── eksport.rs             ← `kada-cli eksport --format json`
    │   ├── laporan.rs             ← `kada-cli laporan gaji --bulan 01`
    │   └── sync.rs                ← `kada-cli sync --server api.kada.gov.my`
    │
    ├── config.rs
    ├── error.rs
    │
    └── utils/
        ├── mod.rs
        ├── format.rs              ← table formatting, progress bar
        └── fail.rs                ← file helpers
```

```toml
# Cargo.toml — CLI Tool
[package]
name    = "kada-cli"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "kada"
path = "src/main.rs"

[dependencies]
# CLI parsing
clap = { version = "4", features = ["derive", "env"] }

# Output formatting
tabled     = "0.15"        # buat jadual
indicatif  = "0.17"        # progress bar
console    = "0.15"        # terminal colors
dialoguer  = "0.11"        # interactive prompts

# HTTP client (untuk sync dengan API)
reqwest = { version = "0.11", features = ["json", "blocking"] }

# CSV
csv   = "1"
serde = { version = "1", features = ["derive"] }

# Async runtime
tokio = { version = "1", features = ["full"] }

# Error handling
thiserror = "1"
anyhow    = "1"

# Config
dotenvy = "0.15"
```

```rust
// src/cli/args.rs
use clap::{Parser, Subcommand};

#[derive(Parser, Debug)]
#[command(
    name    = "kada",
    version = env!("CARGO_PKG_VERSION"),
    about   = "KADA Mobile CLI Tool",
    long_about = None,
)]
pub struct Cli {
    /// URL API server
    #[arg(long, env = "KADA_API_URL", default_value = "http://localhost:3000")]
    pub api_url: String,

    /// Token autentikasi
    #[arg(long, env = "KADA_TOKEN")]
    pub token: Option<String>,

    /// Tahap verbose (guna -v, -vv, -vvv)
    #[arg(short, long, action = clap::ArgAction::Count)]
    pub verbose: u8,

    #[command(subcommand)]
    pub arahan: Arahan,
}

#[derive(Subcommand, Debug)]
pub enum Arahan {
    /// Import data pekerja dari CSV
    Import {
        /// Laluan fail CSV
        fail: std::path::PathBuf,

        /// Tindihan rekod yang ada
        #[arg(long)]
        tindih: bool,
    },

    /// Eksport data ke CSV atau JSON
    Eksport {
        /// Format output (csv atau json)
        #[arg(short, long, default_value = "csv")]
        format: String,

        /// Laluan output
        #[arg(short, long)]
        output: Option<std::path::PathBuf>,
    },

    /// Jana laporan gaji
    Laporan {
        #[command(subcommand)]
        jenis: JenisLaporan,
    },

    /// Sync dengan server
    Sync {
        /// Arah sync (push atau pull)
        #[arg(default_value = "pull")]
        arah: String,
    },
}

#[derive(Subcommand, Debug)]
pub enum JenisLaporan {
    /// Laporan gaji bulanan
    Gaji {
        #[arg(short, long)]
        bulan: u8,
        #[arg(short, long)]
        tahun: u32,
    },
    /// Laporan kehadiran
    Kehadiran {
        #[arg(short, long)]
        bulan: u8,
    },
}
```

---

# STRUKTUR 4: Workspace Besar 🏢

## Untuk Aplikasi Production

```
kada-platform/
├── Cargo.toml                     ← Workspace root
├── Cargo.lock
├── .gitignore
├── README.md
├── docker-compose.yml
│
├── crates/
│   │
│   ├── kada-core/                 ← Shared types, utilities, traits
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── models/
│   │       ├── error.rs
│   │       └── utils/
│   │
│   ├── kada-db/                   ← Database layer
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── migrations/
│   │       └── repositories/
│   │
│   ├── kada-api/                  ← REST API server
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs
│   │       ├── handlers/
│   │       └── middleware/
│   │
│   ├── kada-web/                  ← Web portal (HTML)
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs
│   │       └── handlers/
│   │
│   ├── kada-worker/               ← Background jobs
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs
│   │       └── jobs/
│   │
│   └── kada-cli/                  ← CLI tool
│       ├── Cargo.toml
│       └── src/
│           └── main.rs
│
└── tools/                         ← Development scripts
    ├── setup.sh
    └── seed.sql
```

## Workspace Cargo.toml

```toml
# Cargo.toml (workspace root)
[workspace]
members = [
    "crates/kada-core",
    "crates/kada-db",
    "crates/kada-api",
    "crates/kada-web",
    "crates/kada-worker",
    "crates/kada-cli",
]
resolver = "2"

# Shared dependencies — guna workspace = true dalam member crates
[workspace.dependencies]
# Core
tokio     = { version = "1", features = ["full"] }
serde     = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror  = "1"
anyhow     = "1"
tracing    = "0.1"

# Database
sqlx = { version = "0.7", features = [
    "runtime-tokio-rustls", "mysql", "macros", "chrono", "uuid"
] }

# Web
axum       = "0.7"
tower-http = { version = "0.5", features = ["cors", "trace"] }

# Cache
redis         = { version = "0.25", features = ["tokio-comp"] }
deadpool-redis = "0.15"

# Common utilities
uuid    = { version = "1", features = ["v4", "serde"] }
chrono  = { version = "0.4", features = ["serde"] }
dotenvy = "0.15"
validator = { version = "0.18", features = ["derive"] }

# Internal crates
kada-core = { path = "crates/kada-core" }
kada-db   = { path = "crates/kada-db" }
```

---

# KONVENSYEN & AMALAN TERBAIK 📋

## Penamaan

```
Fail/modul  → snake_case          (pekerja.rs, auth_service.rs)
Struct      → PascalCase          (Pekerja, AppState, JwtClaims)
Enum        → PascalCase          (AppError, StatusPekerja)
Fungsi      → snake_case          (dapatkan_pekerja, buat_token_jwt)
Constants   → SCREAMING_SNAKE     (MAX_LOGIN_CUBA, JWT_EXPIRY_SAAT)
Trait       → PascalCase          (PekerjaRepository, Searchable)
```

## Konvensyen Fail .env

```bash
# .env.example — commit ini!
DATABASE_URL=mysql://pengguna:katalaluan@localhost/nama_db
REDIS_URL=redis://127.0.0.1:6379
JWT_SECRET=ganti_dengan_secret_yang_kuat
PORT=3000
HOST=0.0.0.0
LOG_LEVEL=debug

# .env — JANGAN commit! (dalam .gitignore)
DATABASE_URL=mysql://root:rahsia@localhost/kada_db
JWT_SECRET=super_secret_key_production_abc123
```

## Layout Modul — Cara yang Betul

```rust
// ✔ CARA BETUL: mod.rs sebagai re-exporter sahaja
// src/models/mod.rs
pub mod pekerja;
pub mod kehadiran;
pub use pekerja::Pekerja;         // re-export untuk kemudahan
pub use kehadiran::Kehadiran;

// ✔ CARA BETUL: satu fail = satu tanggungjawab
// src/db/pekerja.rs — hanya query database pekerja

// ❌ ELAK: god module — satu fail semua benda
// src/everything.rs  ← JANGAN!

// ❌ ELAK: nested terlalu dalam
// src/api/v1/handlers/pekerja/crud/update.rs ← TERLALU DALAM!
```

## Error Handling Pattern

```rust
// ✔ Guna thiserror untuk library/crate
#[derive(Debug, thiserror::Error)]
pub enum AppError { ... }

// ✔ Guna ? untuk propagate
pub async fn dapatkan(db: &Pool, id: u32) -> AppResult<Pekerja> {
    sqlx::query_as!(...).fetch_optional(db).await?
        .ok_or_else(|| AppError::TidakJumpai(format!("ID {}", id)))
}

// ✔ Guna anyhow untuk binary/main
fn main() -> anyhow::Result<()> {
    let config = Config::dari_env()?;
    Ok(())
}

// ❌ JANGAN unwrap() dalam production code
let data = query.fetch_one(&db).unwrap(); // ← BAHAYA!
```

## Testing Structure

```rust
// src/db/pekerja.rs — unit test dalam fail yang sama
#[cfg(test)]
mod tests {
    use super::*;
    use sqlx::MySqlPool;

    #[sqlx::test]                    // auto-manage test DB!
    async fn test_simpan_pekerja(pool: MySqlPool) {
        let data = BuatPekerja { ... };
        let id = simpan(&pool, &data).await.unwrap();
        assert!(id > 0);
    }
}

// tests/integration/test_pekerja.rs — integration test
#[tokio::test]
async fn test_api_pekerja() {
    let app = bina_test_app().await;
    let response = app.get("/api/v1/pekerja").send().await;
    assert_eq!(response.status(), 200);
}
```

---

# .gitignore Standard untuk Rust

```gitignore
# Build
/target

# Environment
.env
*.env
!.env.example

# IDE
.idea/
.vscode/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Database
*.sqlite
*.db

# Certificates
*.pem
*.key
*.crt

# Temporary
tmp/
temp/
```

---

# 📋 Rujukan Pantas — Checklist Projek Baru

## Langkah Bermula

```bash
# 1. Buat projek
cargo new nama-projek
# atau workspace:
mkdir nama-projek && cd nama-projek

# 2. Setup .gitignore
echo "/target\n.env" > .gitignore

# 3. Buat .env.example
touch .env.example .env

# 4. Tambah dependencies asas
cargo add tokio --features full
cargo add serde --features derive
cargo add thiserror
cargo add tracing tracing-subscriber --features env-filter

# 5. Untuk web API:
cargo add axum
cargo add sqlx --features "runtime-tokio-rustls mysql macros"
cargo add tower-http --features "cors trace"

# 6. Buat struktur folder
mkdir -p src/{models,db,handlers,services,middleware}

# 7. Buat fail asas
touch src/{config,error,state,routes}.rs

# 8. Jalankan
cargo run
```

## Checklist Sebelum Production

```
Keselamatan:
  ✔ JWT secret dalam environment variable (bukan hardcode)
  ✔ SQL query guna parameterized (bukan string interpolation)
  ✔ Input validation dengan crate validator
  ✔ Rate limiting untuk endpoint sensitif
  ✔ CORS dikonfigurasi dengan betul

Performance:
  ✔ Connection pooling untuk database (sqlx pool)
  ✔ Connection pooling untuk Redis
  ✔ Gunakan &str / &[T] bukannya String/Vec bila boleh
  ✔ Logging tidak block async tasks

Error Handling:
  ✔ Tiada .unwrap() dalam business logic
  ✔ Error disertai konteks (with_context)
  ✔ Log error level yang betul (error!/warn!/info!)
  ✔ Response error tidak dedah internal detail

Testing:
  ✔ Unit tests untuk business logic (services/)
  ✔ Integration tests untuk endpoints
  ✔ Test untuk happy path DAN error path

Deployment:
  ✔ cargo build --release
  ✔ Dockerfile atau binary distribution
  ✔ Health check endpoint /sihat
  ✔ Graceful shutdown
```

---

*Struktur yang baik dari awal = kod yang mudah dijaga.*
*Ikut konvensyen Rust, separation of concerns, dan error-first design.* 🦀
