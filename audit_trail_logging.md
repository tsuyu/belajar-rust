# 📋 Strategi Audit Trail & Log dalam Rust — Panduan Lengkap

> Dari `println!` asas hingga audit trail production-grade:
> structured logging, tracing, audit database, dan compliance.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Kenapa Audit Trail & Log Penting?

```
LOG:
  "Apa yang berlaku dalam sistem?"
  - Debug bila ada masalah
  - Monitor health aplikasi
  - Trace performance bottleneck
  - Alert bila ada anomali

AUDIT TRAIL:
  "Siapa buat apa, bila, dan dari mana?"
  - Pematuhan peraturan (compliance)
  - Siasatan keselamatan
  - Forensik bila ada insiden
  - Akauntabiliti pengguna

Contoh:
  LOG:   "2024-01-15 08:02:33 INFO Request POST /kehadiran"
  AUDIT: "Ali Ahmad (KADA001) daftar masuk pada 08:02:33
          dari IP 192.168.1.5, GPS (6.0538, 102.2503),
          diluluskan sistem"

Peraturan yang mungkin memerlukan audit:
  - PDPA (Personal Data Protection Act) Malaysia
  - ISO 27001 (Information Security)
  - Audit dalaman kerajaan
  - Piawaian kewangan
```

---

## Peta Pembelajaran

```
Bahagian 1  → Log Asas dengan tracing
Bahagian 2  → Structured Logging
Bahagian 3  → Log Levels & Strategy
Bahagian 4  → Log Rotation & Storage
Bahagian 5  → Audit Trail — Reka Bentuk
Bahagian 6  → Audit Trail — Implementasi Database
Bahagian 7  → Middleware Audit untuk Axum
Bahagian 8  → Tracing Distributed (OpenTelemetry)
Bahagian 9  → Alert & Monitoring
Bahagian 10 → Mini Project: Sistem Audit KADA
```

---

# BAHAGIAN 1: Log Asas dengan tracing ✍️

## Setup tracing

```toml
[dependencies]
tracing             = "0.1"
tracing-subscriber  = { version = "0.3", features = [
    "env-filter",
    "json",        # JSON output untuk production
    "fmt",         # Pretty output untuk development
] }
tracing-appender    = "0.2"   # log ke fail
```

```rust
// src/main.rs
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

pub fn setup_logging(level: &str, dalam_json: bool) {
    let env_filter = tracing_subscriber::EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| level.into());

    if dalam_json {
        // Production — JSON format untuk log aggregator (ELK, Loki, dll)
        tracing_subscriber::registry()
            .with(env_filter)
            .with(tracing_subscriber::fmt::layer().json())
            .init();
    } else {
        // Development — pretty format yang mudah dibaca
        tracing_subscriber::registry()
            .with(env_filter)
            .with(tracing_subscriber::fmt::layer()
                .with_target(true)      // tunjuk module path
                .with_line_number(true) // tunjuk nombor baris
                .with_thread_ids(false)
                .pretty()
            )
            .init();
    }
}

// Guna dalam main():
// setup_logging("info", std::env::var("LOG_JSON").is_ok());
```

---

## Cara Guna tracing Macros

```rust
use tracing::{trace, debug, info, warn, error, instrument};

fn demo_tracing() {
    // ── Log levels (dari paling detail ke paling kritikal) ────
    trace!("Ini sangat verbose — untuk debug mendalam");
    debug!("Maklumat debug — untuk troubleshooting");
    info!("Maklumat biasa — operasi normal");
    warn!("Amaran — sesuatu yang pelik tapi tidak kritikal");
    error!("Ralat — sesuatu yang perlu perhatian segera");

    // ── Log dengan fields berstruktur ─────────────────────────
    info!(
        pekerja_id = 1001,
        nama = "Ali Ahmad",
        bahagian = "ICT",
        "Pekerja berjaya log masuk"
    );

    // ── Log dengan nilai variabel ─────────────────────────────
    let id = 42u32;
    let masa_ms = 150u64;
    info!(id, masa_ms, "Request selesai dalam {}ms", masa_ms);

    // ── Log dengan context ────────────────────────────────────
    let err = std::io::Error::new(std::io::ErrorKind::NotFound, "Fail tiada");
    error!(
        error = %err,        // Display format
        laluan = "/data/fail.txt",
        "Gagal baca fail konfigurasi"
    );

    // ── warn dengan context masalah ───────────────────────────
    let cuba_ke = 3u32;
    let max_cuba = 5u32;
    warn!(
        cuba_ke,
        max_cuba,
        "Percubaan log masuk gagal"
    );
}
```

---

## #[instrument] — Auto-trace Functions

```rust
use tracing::{info, warn, error, instrument};

// #[instrument] auto-create span dan log masuk/keluar fungsi
// PLUS semua arguments secara automatik!

#[instrument(
    name    = "proses_kehadiran",   // nama span (optional, default = fn name)
    skip(db, redis),                // skip arguments yang besar/sensitive
    fields(
        pekerja_id = %id,           // tambah field khas
        bahagian   = tracing::field::Empty  // akan diisi kemudian
    )
)]
async fn proses_kehadiran(
    db:    &sqlx::MySqlPool,
    redis: &deadpool_redis::Pool,
    id:    u32,
    data:  &DaftarMasuk,
) -> Result<RekodKehadiran, AppError> {
    // Isi field yang di-set Empty sebelum ini
    tracing::Span::current().record("bahagian", &data.bahagian.as_str());

    info!("Mula proses kehadiran");

    // Semak lokasi GPS
    let jarak = kira_jarak(data.lat, data.lon);
    if jarak > 1.0 {
        warn!(
            jarak_km = jarak,
            lat = data.lat,
            lon = data.lon,
            "Lokasi di luar kawasan"
        );
        return Err(AppError::LuarKawasan { jarak });
    }

    info!(jarak_km = jarak, "Lokasi disahkan");

    // Simpan ke database
    let rekod = simpan_rekod(db, id, data).await?;
    info!(rekod_id = rekod.id, "Rekod kehadiran disimpan");

    Ok(rekod)
}

// Output (pretty format):
// 2024-01-15T08:02:33.145Z  INFO proses_kehadiran{pekerja_id=1001 bahagian="ICT"}
//     at src/services/kehadiran.rs:45
//     Mula proses kehadiran
//
// 2024-01-15T08:02:33.147Z  INFO proses_kehadiran{pekerja_id=1001 bahagian="ICT"}
//     Lokasi disahkan jarak_km=0.34
//
// 2024-01-15T08:02:33.152Z  INFO proses_kehadiran{pekerja_id=1001 bahagian="ICT"}
//     Rekod kehadiran disimpan rekod_id=5821
```

---

## 🧠 Brain Teaser #1

Apakah perbezaan antara `tracing` dan `log` crate dalam Rust?

<details>
<summary>👀 Jawapan</summary>

```
log crate (lama):
  - Simple key-value: log::info!("Mesej {}", nilai)
  - Flat — tiada konsep "span" atau hierarchy
  - Sesuai untuk aplikasi mudah
  - Banyak crate lama masih guna log

tracing crate (moden):
  - Structured: info!(field=value, "Mesej")
  - Ada SPAN — group log events dalam context
  - Causal chain — tahu request mana yang buat log
  - Async-aware — span merentasi .await dengan betul
  - Sesuai untuk web servers, microservices

Analogi:
  log     = catatan dalam buku nota (flat)
  tracing = map perjalanan (tahu dari mana, ke mana, dan laluan)

PENTING: tracing boleh forward ke log backend!
  tracing_log::LogTracer::init() // tracing events → log crate
  tracing menjadi "superset" dari log

Untuk aplikasi baru: SENTIASA guna tracing.
```
</details>

---

# BAHAGIAN 2: Structured Logging 🔧

## Format JSON untuk Production

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};
use tracing_appender::rolling;

pub fn setup_production_logging() {
    // Log ke fail dengan rolling (rotate harian)
    let log_appender = rolling::daily("/var/log/kada-api", "app.log");
    let (fail_penulis, _guard) = tracing_appender::non_blocking(log_appender);

    // Log ke stdout juga (untuk Docker/systemd)
    let (stdout_penulis, _guard2) = tracing_appender::non_blocking(std::io::stdout());

    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::from_default_env())
        // JSON ke fail
        .with(tracing_subscriber::fmt::layer()
            .json()
            .with_writer(fail_penulis)
            .with_current_span(true)   // include span info
            .with_span_list(true)      // include semua spans
        )
        // Pretty ke stdout
        .with(tracing_subscriber::fmt::layer()
            .with_writer(stdout_penulis)
            .with_ansi(false)          // tiada warna dalam fail
        )
        .init();

    // PENTING: simpan _guard! Bila _guard di-drop, flush semua log.
    // Jangan biarkan _guard di-drop sebelum program tamat!
}
```

## Contoh Output JSON

```json
{
  "timestamp": "2024-01-15T08:02:33.145Z",
  "level": "INFO",
  "target": "kada_api::services::kehadiran",
  "span": {
    "name": "proses_kehadiran",
    "pekerja_id": 1001,
    "bahagian": "ICT"
  },
  "message": "Rekod kehadiran disimpan",
  "fields": {
    "rekod_id": 5821,
    "masa_proses_ms": 7
  }
}
```

---

## Request Correlation ID

```rust
use axum::{
    middleware::Next,
    extract::Request,
    response::Response,
    http::HeaderValue,
};
use uuid::Uuid;
use tracing::Instrument;

// Setiap request dapat correlation ID unik
// Semua log dalam request guna ID yang sama
pub async fn correlation_id_middleware(
    mut req: Request,
    next: Next,
) -> Response {
    // Ambil dari header atau jana baru
    let corr_id = req.headers()
        .get("x-correlation-id")
        .and_then(|v| v.to_str().ok())
        .map(|s| s.to_string())
        .unwrap_or_else(|| Uuid::new_v4().to_string());

    // Buat span dengan correlation ID
    let span = tracing::info_span!(
        "http_request",
        correlation_id = %corr_id,
        method         = %req.method(),
        uri            = %req.uri(),
    );

    // Tambah ke response header
    let mut response = next.run(req).instrument(span).await;
    response.headers_mut().insert(
        "x-correlation-id",
        HeaderValue::from_str(&corr_id).unwrap(),
    );

    response
}
```

---

# BAHAGIAN 3: Log Levels & Strategy 📊

## Panduan Level Log

```
LEVEL       GUNA UNTUK                              CONTOH
─────────────────────────────────────────────────────────────────────
TRACE       Maklumat paling detail                  "Masuk fungsi kira_jarak(6.05, 102.25)"
            (debug mendalam, biasanya disabled)      "SQL params: [1001, '2024-01-15']"

DEBUG       Maklumat untuk troubleshooting          "Cache HIT untuk pekerja:1001"
            (diaktifkan bila debugging)              "JWT token valid, expire dalam 3560s"

INFO        Operasi normal yang penting             "Pekerja 1001 berjaya daftar masuk"
            (selalu aktif dalam production)          "Server dimulakan pada :3000"
                                                     "Backup selesai: 1500 rekod"

WARN        Situasi luar biasa tapi tidak kritikal  "Rate limit hampir penuh: 95/100"
            (perlu perhatian tapi tidak segera)      "Fail config tidak jumpa, guna default"
                                                     "Percubaan login gagal kali ke-3"

ERROR       Ralat yang memerlukan perhatian         "Gagal sambung database"
            (perlu tindakan segera)                  "JWT token tidak sah"
                                                     "Fail simpan rekod kehadiran"
```

---

## Strategi Logging Mengikut Lapisan

```rust
// ── Handler (HTTP layer) ──────────────────────────────────────
// Log: request masuk, response status, masa
// JANGAN log: request body (mungkin ada kata laluan!)

#[instrument(skip_all, fields(
    method = %req.method(),
    uri    = %req.uri(),
    status = tracing::field::Empty,
    masa   = tracing::field::Empty,
))]
async fn log_request(req: Request, next: Next) -> Response {
    let mula = std::time::Instant::now();
    let resp = next.run(req).await;
    let masa_ms = mula.elapsed().as_millis();

    let status = resp.status().as_u16();
    let span = tracing::Span::current();
    span.record("status", status);
    span.record("masa", masa_ms);

    if status >= 500 {
        error!("Request selesai dengan server error");
    } else if status >= 400 {
        warn!("Request selesai dengan client error");
    } else {
        info!("Request selesai");
    }

    resp
}

// ── Service (Business Logic) ──────────────────────────────────
// Log: operasi penting, keputusan perniagaan
// Log: warn bila ada situasi luar biasa

async fn service_log_masuk(
    username: &str,
    // JANGAN log kata laluan!
) -> Result<Token, LoginError> {
    debug!("Cuba log masuk untuk '{}'", username);

    // Semak pengguna
    let pengguna = db::cari_pengguna(username).await?;

    // Log cuba yang gagal
    if !sahkan_kata_laluan(&pengguna.hash) {
        warn!(
            username,
            cuba_ke = pengguna.cuba_ke + 1,
            ip = tracing::field::Empty, // akan diisi dari context
            "Kata laluan salah"
        );
        return Err(LoginError::KataLaluanSalah);
    }

    info!(
        pengguna_id = pengguna.id,
        username,
        "Log masuk berjaya"
    );

    Ok(jana_token(&pengguna))
}

// ── Database Layer ────────────────────────────────────────────
// Log: masa query yang lambat
// Log DEBUG: semua queries (hanya bila debugging!)

async fn query_dengan_slow_log<T>(
    nama_query: &str,
    query_future: impl std::future::Future<Output = Result<T, sqlx::Error>>,
) -> Result<T, sqlx::Error> {
    let mula = std::time::Instant::now();
    let hasil = query_future.await;
    let masa_ms = mula.elapsed().as_millis();

    if masa_ms > 100 {
        warn!(
            query = nama_query,
            masa_ms,
            "SLOW QUERY terkesan"
        );
    } else {
        debug!(query = nama_query, masa_ms, "Query selesai");
    }

    hasil
}
```

---

## Apa yang TIDAK Patut Di-log

```rust
// ❌ JANGAN LOG MAKLUMAT SENSITIF!

// Kata laluan
error!("Login gagal untuk password: {}", password); // JANGAN!

// Token
info!("JWT: {}", jwt_token); // JANGAN!

// Nombor kad kredit
info!("Bayaran: {}", nombor_kad); // JANGAN!

// IC / nombor pengenalan penuh
info!("IC: {}", ic_penuh); // JANGAN!

// ✔ CARA BETUL:
error!("Login gagal untuk pengguna: {}", username); // OK
info!("JWT dijanakan untuk user_id: {}", user_id);  // OK
info!("Bayaran: ****{}", &nombor_kad[12..]);         // Mask sebahagian
info!("IC: {}**-**-{}", &ic[..6], &ic[10..]);        // Mask tengah

// ── Juga elak log: ────────────────────────────────────────────
// - Data peribadi yang tidak perlu (alamat rumah, dll)
// - Query params yang mengandungi sensitive data
// - Request body penuh (guna field selective)
// - Stack trace penuh kepada pengguna akhir
```

---

# BAHAGIAN 4: Log Rotation & Storage 💾

## Log ke Fail dengan Rotation

```rust
use tracing_appender::rolling::{RollingFileAppender, Rotation};
use tracing_appender::non_blocking;

pub fn setup_fail_logging() -> Vec<tracing_appender::non_blocking::WorkerGuard> {
    let mut guards = Vec::new();

    // ── Log harian (rotate setiap tengah malam) ───────────────
    let harian = RollingFileAppender::new(
        Rotation::DAILY,
        "/var/log/kada-api",
        "app.log",
    );
    // Fail: app.log.2024-01-15, app.log.2024-01-16, dll

    // ── Log per jam (untuk traffic tinggi) ────────────────────
    let per_jam = RollingFileAppender::new(
        Rotation::HOURLY,
        "/var/log/kada-api",
        "error.log",
    );

    // ── Non-blocking: log tidak block async tasks ─────────────
    let (penulis_harian, guard1) = non_blocking(harian);
    let (penulis_jam, guard2)    = non_blocking(per_jam);

    guards.push(guard1);
    guards.push(guard2);

    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::from_default_env())
        .with(tracing_subscriber::fmt::layer()
            .json()
            .with_writer(penulis_harian)
        )
        .with(tracing_subscriber::fmt::layer()
            .json()
            .with_writer(penulis_jam)
            .with_filter(
                // Hanya ERROR ke fail error.log
                tracing_subscriber::filter::LevelFilter::ERROR
            )
        )
        .init();

    guards // PENTING: simpan guards! Kalau di-drop, flush tidak berlaku.
}
```

---

## systemd / Docker Log Configuration

```
# /etc/systemd/system/kada-api.service
[Service]
# Tulis log ke systemd journal
StandardOutput=journal
StandardError=journal
SyslogIdentifier=kada-api

# Env variable untuk log level
Environment=RUST_LOG=info
Environment=LOG_JSON=true

# Limitkan saiz log (optional)
LogRateLimitIntervalSec=30
LogRateLimitBurst=1000
```

```yaml
# docker-compose.yml
services:
  kada-api:
    image: kada-api:latest
    environment:
      - RUST_LOG=info
      - LOG_JSON=true
    logging:
      driver: "json-file"
      options:
        max-size: "100m"    # max 100MB per fail
        max-file: "5"       # simpan 5 fail (500MB total)
```

---

# BAHAGIAN 5: Audit Trail — Reka Bentuk 🗄️

## Apa yang Perlu Diaudit?

```
KATEGORI AUDIT:

1. AUTENTIKASI
   ✔ Log masuk berjaya
   ✔ Log masuk gagal (termasuk IP)
   ✔ Log keluar
   ✔ Reset kata laluan
   ✔ Tukar kata laluan

2. AKSES DATA
   ✔ Lihat rekod sensitif (profil pekerja, gaji)
   ✔ Export data
   ✔ Carian dengan kriteria sensitif

3. PENGUBAHAN DATA (CRUD)
   ✔ CREATE — buat rekod baru
   ✔ UPDATE — ubah rekod (simpan nilai LAMA dan BARU)
   ✔ DELETE — padam rekod
   ✔ RESTORE — pulihkan rekod

4. PENTADBIRAN
   ✔ Buat/ubah/padam akaun pengguna
   ✔ Tukar peranan/kebenaran
   ✔ Konfigurasi sistem

5. KESELAMATAN
   ✔ Percubaan akses tidak dibenarkan
   ✔ Bypass keselamatan
   ✔ Anomali (login dari lokasi luar biasa)
```

---

## Skema Database Audit Trail

```sql
-- migrations/001_buat_jadual_audit.sql

-- Jadual utama audit trail
CREATE TABLE audit_trail (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid            VARCHAR(36)  NOT NULL UNIQUE DEFAULT (UUID()),

    -- Siapa
    pengguna_id     INT UNSIGNED NULL,          -- NULL = sistem/bot
    no_pekerja      VARCHAR(20)  NULL,
    nama_pengguna   VARCHAR(100) NULL,

    -- Apa
    tindakan        ENUM(
        'LOGIN_BERJAYA', 'LOGIN_GAGAL', 'LOGOUT',
        'BUAT', 'BACA', 'KEMASKINI', 'PADAM',
        'EKSPORT', 'IMPORT',
        'TUKAR_KATA_LALUAN', 'RESET_KATA_LALUAN',
        'TUKAR_PERANAN', 'LAIN'
    ) NOT NULL,

    modul           VARCHAR(50)  NOT NULL,      -- 'pekerja', 'kehadiran', dll
    id_rekod        VARCHAR(100) NULL,           -- ID rekod yang diubah
    nama_rekod      VARCHAR(200) NULL,           -- Nama/keterangan rekod

    -- Perubahan (untuk UPDATE)
    nilai_lama      JSON         NULL,           -- data sebelum ubah
    nilai_baru      JSON         NULL,           -- data selepas ubah

    -- Konteks teknikal
    ip_address      VARCHAR(45)  NULL,           -- IPv4 atau IPv6
    user_agent      VARCHAR(500) NULL,
    correlation_id  VARCHAR(36)  NULL,           -- untuk trace request
    endpoint        VARCHAR(200) NULL,           -- /api/v1/pekerja/:id
    kaedah_http     VARCHAR(10)  NULL,           -- GET, POST, PUT, DELETE

    -- Status
    berjaya         BOOLEAN      NOT NULL DEFAULT TRUE,
    mesej_ralat     TEXT         NULL,

    -- Masa
    dibuat_pada     DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,

    -- Indeks
    INDEX idx_pengguna (pengguna_id),
    INDEX idx_tindakan (tindakan),
    INDEX idx_modul (modul),
    INDEX idx_masa (dibuat_pada),
    INDEX idx_correlation (correlation_id),
    INDEX idx_ip (ip_address)

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Jadual ringkasan (untuk dashboard)
CREATE TABLE audit_ringkasan_harian (
    id              INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    tarikh          DATE         NOT NULL,
    tindakan        VARCHAR(50)  NOT NULL,
    modul           VARCHAR(50)  NOT NULL,
    kiraan          INT UNSIGNED NOT NULL DEFAULT 0,
    kiraan_gagal    INT UNSIGNED NOT NULL DEFAULT 0,

    UNIQUE KEY uk_ringkasan (tarikh, tindakan, modul)
) ENGINE=InnoDB;
```

---

# BAHAGIAN 6: Audit Trail — Implementasi 💻

## Audit Writer

```rust
// src/audit/mod.rs
pub mod writer;
pub mod model;
pub use writer::AuditWriter;
pub use model::*;
```

```rust
// src/audit/model.rs
use serde::{Serialize, Deserialize};
use chrono::NaiveDateTime;

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "VARCHAR", rename_all = "SCREAMING_SNAKE_CASE")]
pub enum TindakanAudit {
    LoginBerjaya,
    LoginGagal,
    Logout,
    Buat,
    Baca,
    Kemaskini,
    Padam,
    Eksport,
    Import,
    TukarKataLaluan,
    ResetKataLaluan,
    TukarPeranan,
    Lain,
}

#[derive(Debug, Clone)]
pub struct RekorAudit {
    pub pengguna_id:  Option<u32>,
    pub no_pekerja:   Option<String>,
    pub nama_pengguna: Option<String>,
    pub tindakan:     TindakanAudit,
    pub modul:        String,
    pub id_rekod:     Option<String>,
    pub nama_rekod:   Option<String>,
    pub nilai_lama:   Option<serde_json::Value>,
    pub nilai_baru:   Option<serde_json::Value>,
    pub ip_address:   Option<String>,
    pub user_agent:   Option<String>,
    pub correlation_id: Option<String>,
    pub endpoint:     Option<String>,
    pub kaedah_http:  Option<String>,
    pub berjaya:      bool,
    pub mesej_ralat:  Option<String>,
}

// Builder pattern untuk mudah buat rekod audit
pub struct AuditBuilder {
    rekod: RekorAudit,
}

impl AuditBuilder {
    pub fn baru(tindakan: TindakanAudit, modul: &str) -> Self {
        AuditBuilder {
            rekod: RekorAudit {
                pengguna_id:   None,
                no_pekerja:    None,
                nama_pengguna: None,
                tindakan,
                modul:         modul.into(),
                id_rekod:      None,
                nama_rekod:    None,
                nilai_lama:    None,
                nilai_baru:    None,
                ip_address:    None,
                user_agent:    None,
                correlation_id: None,
                endpoint:      None,
                kaedah_http:   None,
                berjaya:       true,
                mesej_ralat:   None,
            }
        }
    }

    pub fn pengguna(mut self, id: u32, no: &str, nama: &str) -> Self {
        self.rekod.pengguna_id   = Some(id);
        self.rekod.no_pekerja    = Some(no.into());
        self.rekod.nama_pengguna = Some(nama.into());
        self
    }

    pub fn rekod(mut self, id: impl ToString, nama: &str) -> Self {
        self.rekod.id_rekod  = Some(id.to_string());
        self.rekod.nama_rekod = Some(nama.into());
        self
    }

    pub fn perubahan<T: Serialize>(mut self, lama: &T, baru: &T) -> Self {
        self.rekod.nilai_lama = serde_json::to_value(lama).ok();
        self.rekod.nilai_baru = serde_json::to_value(baru).ok();
        self
    }

    pub fn konteks(mut self, ip: &str, ua: &str, corr_id: &str) -> Self {
        self.rekod.ip_address    = Some(ip.into());
        self.rekod.user_agent    = Some(ua.into());
        self.rekod.correlation_id = Some(corr_id.into());
        self
    }

    pub fn http(mut self, endpoint: &str, kaedah: &str) -> Self {
        self.rekod.endpoint    = Some(endpoint.into());
        self.rekod.kaedah_http = Some(kaedah.into());
        self
    }

    pub fn gagal(mut self, ralat: &str) -> Self {
        self.rekod.berjaya     = false;
        self.rekod.mesej_ralat = Some(ralat.into());
        self
    }

    pub fn bina(self) -> RekorAudit {
        self.rekod
    }
}
```

```rust
// src/audit/writer.rs
use sqlx::MySqlPool;
use super::model::RekorAudit;
use tracing::{info, warn, error};
use tokio::sync::mpsc;

// AuditWriter guna channel untuk async, non-blocking writes
#[derive(Clone)]
pub struct AuditWriter {
    pengirim: mpsc::UnboundedSender<RekorAudit>,
}

impl AuditWriter {
    pub fn baru(db: MySqlPool) -> Self {
        let (tx, mut rx) = mpsc::unbounded_channel::<RekorAudit>();

        // Background task untuk tulis audit ke DB
        tokio::spawn(async move {
            while let Some(rekod) = rx.recv().await {
                if let Err(e) = tulis_ke_db(&db, &rekod).await {
                    // Log ralat tapi JANGAN panic — audit tidak patut crash app!
                    error!(
                        error = %e,
                        tindakan = ?rekod.tindakan,
                        "GAGAL tulis audit trail ke database!"
                    );
                }
            }
        });

        AuditWriter { pengirim: tx }
    }

    // Non-blocking! Hantar ke channel dan teruskan
    pub fn log(&self, rekod: RekorAudit) {
        // Log ke tracing juga (untuk real-time monitoring)
        match rekod.berjaya {
            true  => info!(
                tindakan    = ?rekod.tindakan,
                modul       = rekod.modul,
                pengguna    = rekod.nama_pengguna.as_deref().unwrap_or("-"),
                id_rekod    = rekod.id_rekod.as_deref().unwrap_or("-"),
                ip          = rekod.ip_address.as_deref().unwrap_or("-"),
                "[AUDIT] Tindakan berjaya"
            ),
            false => warn!(
                tindakan    = ?rekod.tindakan,
                modul       = rekod.modul,
                pengguna    = rekod.nama_pengguna.as_deref().unwrap_or("-"),
                ralat       = rekod.mesej_ralat.as_deref().unwrap_or("-"),
                ip          = rekod.ip_address.as_deref().unwrap_or("-"),
                "[AUDIT] Tindakan GAGAL"
            ),
        }

        // Hantar ke channel untuk tulis ke DB
        if let Err(e) = self.pengirim.send(rekod) {
            error!("Gagal hantar ke audit channel: {}", e);
        }
    }
}

async fn tulis_ke_db(
    db:    &MySqlPool,
    rekod: &RekorAudit,
) -> Result<(), sqlx::Error> {
    sqlx::query!(
        r#"INSERT INTO audit_trail (
            pengguna_id, no_pekerja, nama_pengguna,
            tindakan, modul, id_rekod, nama_rekod,
            nilai_lama, nilai_baru,
            ip_address, user_agent, correlation_id,
            endpoint, kaedah_http,
            berjaya, mesej_ralat
        ) VALUES (
            ?, ?, ?,
            ?, ?, ?, ?,
            ?, ?,
            ?, ?, ?,
            ?, ?,
            ?, ?
        )"#,
        rekod.pengguna_id,
        rekod.no_pekerja,
        rekod.nama_pengguna,
        format!("{:?}", rekod.tindakan).to_uppercase(),
        rekod.modul,
        rekod.id_rekod,
        rekod.nama_rekod,
        rekod.nilai_lama.as_ref().map(|v| v.to_string()),
        rekod.nilai_baru.as_ref().map(|v| v.to_string()),
        rekod.ip_address,
        rekod.user_agent,
        rekod.correlation_id,
        rekod.endpoint,
        rekod.kaedah_http,
        rekod.berjaya,
        rekod.mesej_ralat,
    )
    .execute(db)
    .await?;

    Ok(())
}
```

---

# BAHAGIAN 7: Middleware Audit untuk Axum 🔒

## Audit Middleware

```rust
// src/middleware/audit.rs
use axum::{
    middleware::Next,
    extract::{Request, State},
    response::Response,
};
use std::time::Instant;
use crate::{
    state::AppState,
    audit::{AuditBuilder, TindakanAudit},
};

pub async fn audit_middleware(
    State(state): State<AppState>,
    req: Request,
    next: Next,
) -> Response {
    let kaedah  = req.method().to_string();
    let uri     = req.uri().to_string();
    let mula    = Instant::now();

    // Ambil info dari headers
    let ip = req.headers()
        .get("x-forwarded-for")
        .or_else(|| req.headers().get("x-real-ip"))
        .and_then(|v| v.to_str().ok())
        .unwrap_or("unknown")
        .to_string();

    let ua = req.headers()
        .get("user-agent")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("unknown")
        .to_string();

    let corr_id = req.headers()
        .get("x-correlation-id")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("unknown")
        .to_string();

    // Ambil info pengguna dari extension (diset oleh auth middleware)
    let pengguna = req.extensions().get::<InfoPengguna>().cloned();

    // Jalankan request
    let resp = next.run(req).await;
    let masa_ms = mula.elapsed().as_millis();
    let status  = resp.status().as_u16();

    // Tentukan sama ada perlu audit berdasarkan method dan status
    let perlu_audit = matches!(kaedah.as_str(), "POST" | "PUT" | "PATCH" | "DELETE")
        || status == 401
        || status == 403;

    if perlu_audit {
        let tindakan = tentukan_tindakan(&kaedah, status);

        let mut builder = AuditBuilder::baru(tindakan, &extract_modul(&uri))
            .http(&uri, &kaedah)
            .konteks(&ip, &ua, &corr_id);

        if let Some(p) = &pengguna {
            builder = builder.pengguna(p.id, &p.no_pekerja, &p.nama);
        }

        if status >= 400 {
            builder = builder.gagal(&format!("HTTP {}", status));
        }

        state.audit.log(builder.bina());
    }

    resp
}

fn tentukan_tindakan(kaedah: &str, status: u16) -> TindakanAudit {
    if status == 401 { return TindakanAudit::LoginGagal; }
    match kaedah {
        "POST"   => TindakanAudit::Buat,
        "PUT"
        | "PATCH" => TindakanAudit::Kemaskini,
        "DELETE" => TindakanAudit::Padam,
        "GET"    => TindakanAudit::Baca,
        _        => TindakanAudit::Lain,
    }
}

fn extract_modul(uri: &str) -> String {
    uri.split('/')
        .filter(|s| !s.is_empty() && *s != "api" && !s.starts_with('v'))
        .next()
        .unwrap_or("am")
        .to_string()
}

#[derive(Clone)]
pub struct InfoPengguna {
    pub id:         u32,
    pub no_pekerja: String,
    pub nama:       String,
}
```

---

## Guna Audit dalam Services

```rust
// src/services/pekerja.rs
use crate::audit::{AuditBuilder, AuditWriter, TindakanAudit};
use serde_json::json;

pub async fn kemaskini_pekerja(
    db:    &sqlx::MySqlPool,
    audit: &AuditWriter,
    id:    u32,
    data:  &KemaskiniPekerja,
    oleh:  &InfoPengguna,       // siapa yang kemaskini
    ip:    &str,
) -> AppResult<Pekerja> {
    // Ambil data LAMA dulu
    let lama = db::pekerja::dapatkan_satu(db, id).await?;

    // Buat perubahan
    db::pekerja::kemaskini(db, id, data).await?;

    // Ambil data BARU
    let baru = db::pekerja::dapatkan_satu(db, id).await?;

    // Log audit — simpan nilai lama DAN baru
    audit.log(
        AuditBuilder::baru(TindakanAudit::Kemaskini, "pekerja")
            .pengguna(oleh.id, &oleh.no_pekerja, &oleh.nama)
            .rekod(id, &lama.nama)
            .perubahan(&json!({
                "nama":     lama.nama,
                "bahagian": lama.bahagian,
                "gaji":     lama.gaji,
                "aktif":    lama.aktif,
            }), &json!({
                "nama":     baru.nama,
                "bahagian": baru.bahagian,
                "gaji":     baru.gaji,
                "aktif":    baru.aktif,
            }))
            .konteks(ip, "system", "")
            .bina()
    );

    info!(
        pekerja_id  = id,
        dikemaskini_oleh = oleh.no_pekerja,
        "Pekerja dikemaskini"
    );

    Ok(baru)
}
```

---

## 🧠 Brain Teaser #2

Mengapa audit trail perlu ditulis ke STORAGE YANG BERBEZA dari log aplikasi biasa?

<details>
<summary>👀 Jawapan</summary>

```
LOG APLIKASI:
  - Boleh dipadam/rotate (hanya simpan N hari)
  - Format boleh berubah mengikut keperluan
  - Tujuan: debug dan monitoring
  - Penyimpan: fail log, Loki, ELK
  - Retention: 7-90 hari biasanya cukup

AUDIT TRAIL:
  - TIDAK BOLEH dipadam atau diubah (tamper-proof!)
  - Format mesti konsisten (untuk siasatan)
  - Tujuan: compliance, forensik, akauntabiliti
  - Penyimpan: database dengan backup, atau immutable storage
  - Retention: 1-7 TAHUN (bergantung peraturan)

MASALAH kalau simpan di tempat sama:
  ✗ Log rotate → audit trail hilang!
  ✗ Admin boleh delete log → evidence hilang
  ✗ Format log berubah → audit lama tidak boleh diparse
  ✗ Log mungkin tidak ada nilai lama/baru untuk changes
  ✗ Log tidak ada indexing yang baik untuk carian

PENYELESAIAN:
  Audit Trail → Database (MySQL/PostgreSQL)
              → dengan backup automatik
              → dengan indeks untuk carian
              → dengan access control (admin pun tidak boleh delete!)
  
  Log Aplikasi → fail/Loki/ELK
               → boleh rotate, boleh padam
               → untuk debug dan monitoring

Bonus: Audit trail dalam DB boleh:
  ✔ Join dengan data lain (siapa buat apa)
  ✔ Report dan analytics
  ✔ Alert berdasarkan pattern
  ✔ Carian berstruktur (WHERE tindakan = 'DELETE')
```
</details>

---

# BAHAGIAN 8: Distributed Tracing 🌐

## OpenTelemetry Integration

```toml
[dependencies]
opentelemetry         = "0.22"
opentelemetry-otlp    = "0.15"
tracing-opentelemetry = "0.23"
opentelemetry_sdk     = "0.22"
```

```rust
use opentelemetry::global;
use opentelemetry_otlp::WithExportConfig;
use tracing_opentelemetry::OpenTelemetryLayer;

pub fn setup_otel_tracing(
    servis_nama: &str,
    otlp_endpoint: &str,
) -> Result<(), Box<dyn std::error::Error>> {
    // Setup OTLP exporter (ke Jaeger, Tempo, OTEL Collector)
    let tracer = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(
            opentelemetry_otlp::new_exporter()
                .tonic()
                .with_endpoint(otlp_endpoint),
        )
        .with_trace_config(
            opentelemetry_sdk::trace::config()
                .with_resource(opentelemetry_sdk::Resource::new(vec![
                    opentelemetry::KeyValue::new("service.name", servis_nama.to_string()),
                    opentelemetry::KeyValue::new("service.version", env!("CARGO_PKG_VERSION")),
                ]))
        )
        .install_batch(opentelemetry_sdk::runtime::Tokio)?;

    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::from_default_env())
        .with(tracing_subscriber::fmt::layer())
        .with(OpenTelemetryLayer::new(tracer))
        .init();

    Ok(())
}

// Setiap request auto-create trace yang boleh diikut
// merentasi pelbagai services!
```

---

# BAHAGIAN 9: Alert & Monitoring 🚨

## Pattern Alert Berdasarkan Log

```rust
// src/monitoring/alert.rs

use std::sync::atomic::{AtomicU32, Ordering};
use std::sync::Arc;
use tokio::time::{interval, Duration};
use tracing::{warn, error};

pub struct KiraLoginGagal {
    kiraan:    Arc<AtomicU32>,
    had:       u32,
    tempoh_s:  u64,
}

impl KiraLoginGagal {
    pub fn baru(had: u32, tempoh_s: u64) -> Self {
        let kiraan = Arc::new(AtomicU32::new(0));
        let kiraan_klon = Arc::clone(&kiraan);

        // Reset kiraan setiap tempoh
        tokio::spawn(async move {
            let mut selang = interval(Duration::from_secs(tempoh_s));
            loop {
                selang.tick().await;
                let lama = kiraan_klon.swap(0, Ordering::Relaxed);
                if lama > 0 {
                    warn!("[MONITOR] Reset kiraan login gagal: {} dalam {}s", lama, tempoh_s);
                }
            }
        });

        KiraLoginGagal { kiraan, had, tempoh_s }
    }

    pub fn rekod_gagal(&self, ip: &str, username: &str) -> bool {
        let kiraan = self.kiraan.fetch_add(1, Ordering::Relaxed) + 1;

        if kiraan >= self.had {
            // ALERT! Mungkin brute force attack
            error!(
                ip,
                username,
                kiraan,
                had = self.had,
                "[SECURITY ALERT] Terlalu banyak login gagal - kemungkinan brute force!"
            );
            return true; // trigger alert
        }

        warn!(ip, username, kiraan, "Login gagal");
        false
    }
}

// Kirim alert ke Slack/Email
async fn kirim_alert(mesej: &str, tahap: &str) {
    if let Ok(webhook) = std::env::var("SLACK_WEBHOOK") {
        let _ = reqwest::Client::new()
            .post(&webhook)
            .json(&serde_json::json!({
                "text": format!("[{}] {}", tahap, mesej)
            }))
            .send()
            .await;
    }
}
```

---

# BAHAGIAN 10: Mini Project — Sistem Audit KADA 🏗️

```rust
// Contoh penggunaan penuh dalam handler
// src/handlers/pekerja.rs

use axum::{
    extract::{State, Path, Json, Extension},
    http::StatusCode,
    response::IntoResponse,
};

use crate::{
    state::AppState,
    error::{AppError, AppResult},
    models::pekerja::*,
    services,
    audit::{AuditBuilder, TindakanAudit},
    middleware::audit::InfoPengguna,
};

pub async fn kemaskini_pekerja(
    State(state): State<AppState>,
    Extension(pengguna): Extension<InfoPengguna>,    // dari auth middleware
    Extension(ip): Extension<String>,                 // dari IP middleware
    Extension(corr_id): Extension<String>,            // dari correlation middleware
    Path(id): Path<u32>,
    Json(body): Json<KemaskiniPekerja>,
) -> AppResult<impl IntoResponse> {
    // Validate
    body.validate().map_err(|e| AppError::InputTidakSah(e.to_string()))?;

    // Service akan handle audit juga
    let pekerja = services::pekerja::kemaskini(
        &state.db,
        &state.audit,
        id,
        &body,
        &pengguna,
        &ip,
    ).await?;

    Ok((StatusCode::OK, Json(ApiResponse::ok(PekerjaResponse::from(pekerja)))))
}

pub async fn padam_pekerja(
    State(state): State<AppState>,
    Extension(pengguna): Extension<InfoPengguna>,
    Extension(ip): Extension<String>,
    Path(id): Path<u32>,
) -> AppResult<impl IntoResponse> {
    // Ambil rekod sebelum padam (untuk audit)
    let pekerja = db::pekerja::dapatkan_satu(&state.db, id).await?;

    // Padam (soft delete — hanya set aktif=false)
    db::pekerja::padam(&state.db, id).await?;

    // Log audit dengan nilai yang dipadam
    state.audit.log(
        AuditBuilder::baru(TindakanAudit::Padam, "pekerja")
            .pengguna(pengguna.id, &pengguna.no_pekerja, &pengguna.nama)
            .rekod(id, &pekerja.nama)
            .perubahan(
                &serde_json::json!({
                    "no_pekerja": pekerja.no_pekerja,
                    "nama":       pekerja.nama,
                    "bahagian":   pekerja.bahagian,
                    "aktif":      true,
                }),
                &serde_json::json!({ "aktif": false })
            )
            .konteks(&ip, "", "")
            .bina()
    );

    Ok((StatusCode::OK, Json(ApiResponse::mesej("Pekerja dipadam"))))
}

// Query audit trail
pub async fn lihat_audit(
    State(state): State<AppState>,
    Query(q): Query<QueryAudit>,
) -> AppResult<impl IntoResponse> {
    let senarai = sqlx::query!(
        r#"SELECT
            a.id, a.tindakan, a.modul, a.nama_pengguna,
            a.nama_rekod, a.ip_address, a.berjaya,
            a.dibuat_pada
        FROM audit_trail a
        WHERE
            (? IS NULL OR a.modul = ?)
            AND (? IS NULL OR a.tindakan = ?)
            AND (? IS NULL OR a.pengguna_id = ?)
            AND a.dibuat_pada >= COALESCE(?, '1970-01-01')
        ORDER BY a.dibuat_pada DESC
        LIMIT 100"#,
        q.modul, q.modul,
        q.tindakan, q.tindakan,
        q.pengguna_id, q.pengguna_id,
        q.dari_tarikh,
    )
    .fetch_all(&state.db)
    .await?;

    Ok((StatusCode::OK, Json(ApiResponse::ok(senarai))))
}
```

---

# 📋 Rujukan Pantas — Audit Trail & Log Cheat Sheet

## Setup Logging

```toml
tracing            = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
tracing-appender   = "0.2"
```

```rust
// Development
tracing_subscriber::fmt().with_env_filter("debug").init();

// Production (JSON ke fail)
let (penulis, _guard) = tracing_appender::non_blocking(
    tracing_appender::rolling::daily("/var/log", "app.log")
);
tracing_subscriber::fmt().json().with_writer(penulis).init();

// Env variable:
// RUST_LOG=info,kada_api=debug
```

## Cara Log

```rust
trace!("paling detail");
debug!("troubleshooting");
info!("operasi normal");
warn!("luar biasa tapi ok");
error!("perlu perhatian");

// Dengan fields
info!(field = value, "Mesej");
warn!(user_id, ip, cuba_ke, "Login gagal");
error!(error = %err, "Gagal buat sesuatu");

// Auto-trace function
#[instrument(skip(password), fields(user = %username))]
async fn login(username: &str, password: &str) -> Result<...> { ... }
```

## Audit Trail Database

```sql
-- Skema minimum audit_trail
CREATE TABLE audit_trail (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    pengguna_id INT UNSIGNED,
    tindakan    VARCHAR(50) NOT NULL,
    modul       VARCHAR(50) NOT NULL,
    id_rekod    VARCHAR(100),
    nilai_lama  JSON,
    nilai_baru  JSON,
    ip_address  VARCHAR(45),
    berjaya     BOOLEAN NOT NULL DEFAULT TRUE,
    dibuat_pada DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX (pengguna_id), INDEX (tindakan), INDEX (dibuat_pada)
);
```

## Golden Rules

```
LOG:
  ✔ Guna structured fields (bukan string interpolation)
  ✔ Log correlation ID setiap request
  ✔ Log masa untuk semua operations
  ✔ Guna level yang betul (DEBUG/INFO/WARN/ERROR)
  ✗ JANGAN log kata laluan, token, IC penuh, kad kredit

AUDIT:
  ✔ Simpan nilai LAMA dan BARU untuk setiap update
  ✔ Simpan siapa, apa, bila, dari mana
  ✔ Non-blocking (jangan lambatkan request!)
  ✔ Immutable — audit tidak boleh diubah/padam
  ✔ Backup berasingan dari data utama
  ✗ JANGAN simpan kata laluan dalam audit
```

---

## 🏆 Cabaran Akhir

Cuba implement salah satu:

1. **Audit Diff Viewer** — UI yang tunjukkan perbezaan nilai lama vs baru dalam format yang cantik
2. **Login Anomaly Detector** — detect login dari lokasi/masa yang luar biasa
3. **Audit Report Generator** — jana laporan PDF/Excel dari audit trail untuk compliance
4. **Real-time Audit Dashboard** — WebSocket yang push audit events ke dashboard admin
5. **Log Tainting** — trace data sensitif melalui sistem dan pastikan tidak di-log

---

*Audit trail yang baik adalah bukti bahawa sistem anda boleh dipercayai.*
*Log yang baik adalah pembantu terbaik bila ada masalah.*
*Kedua-duanya adalah keperluan, bukan pilihan.* 🦀
