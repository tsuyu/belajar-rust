# 🔒 Strategi Web Security dalam Rust — Panduan Lengkap

> Dari JWT authentication hingga rate limiting, CSRF protection,
> input validation, dan security headers.
> Defense-in-depth: setiap lapisan melindungi lapisan lain.

---

## Kenapa Security dalam Rust Lebih Baik?

```
Bahasa lain — punca kelemahan biasa:
  C/C++:  Buffer overflow, use-after-free, null pointer
  PHP:    SQL injection (tiada parameterized query enforcement)
  Java:   Deserialization attacks, XXE injection
  Node:   Prototype pollution, ReDoS

Rust — protection built-in:
  ✔ Buffer overflow → MUSTAHIL (bounds checking)
  ✔ Use-after-free → MUSTAHIL (ownership system)
  ✔ Null pointer → MUSTAHIL (Option<T> bukan null)
  ✔ Data race → MUSTAHIL (borrow checker)
  ✔ Integer overflow → Compile/runtime error

TAPI:
  Rust tidak melindungi dari:
  ✗ Logic errors (autorization bypass, IDOR)
  ✗ SQL injection (kalau guna raw query)
  ✗ XSS (kalau tidak escape HTML)
  ✗ CSRF (perlu implement sendiri)
  ✗ Weak cryptography (salah pilih algorithm)
  → Kita perlu address semua ini!
```

---

## Peta Pembelajaran

```
Bahagian 1  → Authentication — JWT & Session
Bahagian 2  → Password Hashing yang Betul
Bahagian 3  → Authorization — RBAC
Bahagian 4  → Input Validation & Sanitization
Bahagian 5  → SQL Injection Prevention
Bahagian 6  → XSS Prevention
Bahagian 7  → CSRF Protection
Bahagian 8  → Rate Limiting & Brute Force
Bahagian 9  → Security Headers
Bahagian 10 → Mini Project: Secure KADA API
```

---

# BAHAGIAN 1: Authentication — JWT & Session 🔐

## Cargo.toml untuk Security

```toml
[dependencies]
# JWT
jsonwebtoken = "9"

# Password hashing
argon2     = "0.5"     # lebih baik dari bcrypt untuk password
bcrypt     = "0.15"    # alternatif (lebih mudah digunakan)

# Cryptography
rand       = "0.8"     # RNG yang selamat
sha2       = "0.10"    # SHA-256/512
hmac       = "0.12"    # HMAC
hex        = "0.4"     # encode ke hex

# Session
tower-sessions = "0.12"

# Validation
validator = { version = "0.18", features = ["derive"] }

# Rate limiting
governor = "0.6"

# Security headers
tower-http = { version = "0.5", features = ["set-header"] }

# UUID
uuid = { version = "1", features = ["v4"] }

# Time
chrono = { version = "0.4", features = ["serde"] }
```

---

## JWT Authentication

```rust
// src/auth/jwt.rs
use jsonwebtoken::{
    encode, decode, Header, Algorithm, Validation,
    EncodingKey, DecodingKey, errors::Error as JwtError,
};
use serde::{Serialize, Deserialize};
use chrono::{Utc, Duration};

// Claims yang akan disimpan dalam token
#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Claims {
    pub sub:        String,          // subject (user ID)
    pub no_pekerja: String,
    pub nama:       String,
    pub peranan:    Vec<String>,     // ["admin", "pengurus"]
    pub jti:        String,          // JWT ID (untuk revocation)
    pub iat:        i64,             // issued at
    pub exp:        i64,             // expiry
}

pub struct JwtKonfig {
    pub rahsia:     String,
    pub tamat_saat: i64,
}

impl JwtKonfig {
    pub fn jana_token(&self, claims: &Claims) -> Result<String, JwtError> {
        encode(
            &Header::new(Algorithm::HS256),
            claims,
            &EncodingKey::from_secret(self.rahsia.as_bytes()),
        )
    }

    pub fn sahkan_token(&self, token: &str) -> Result<Claims, JwtError> {
        let mut pengesahan = Validation::new(Algorithm::HS256);
        pengesahan.validate_exp = true;   // semak expiry
        pengesahan.leeway = 0;            // tiada leeway

        let data = decode::<Claims>(
            token,
            &DecodingKey::from_secret(self.rahsia.as_bytes()),
            &pengesahan,
        )?;

        Ok(data.claims)
    }
}

// Buat claims untuk pengguna
pub fn buat_claims(
    pengguna: &Pengguna,
    peranan: Vec<String>,
    tamat_saat: i64,
) -> Claims {
    let sekarang = Utc::now().timestamp();
    Claims {
        sub:        pengguna.id.to_string(),
        no_pekerja: pengguna.no_pekerja.clone(),
        nama:       pengguna.nama.clone(),
        peranan,
        jti:        uuid::Uuid::new_v4().to_string(), // unique per token
        iat:        sekarang,
        exp:        sekarang + tamat_saat,
    }
}
```

---

## JWT Middleware untuk Axum

```rust
// src/middleware/auth.rs
use axum::{
    middleware::Next,
    extract::{Request, State},
    response::Response,
    http::{StatusCode, header::AUTHORIZATION},
};
use crate::{state::AppState, error::AppError, auth::jwt::Claims};

pub async fn semak_jwt(
    State(state): State<AppState>,
    mut req: Request,
    next: Next,
) -> Result<Response, AppError> {
    // Ambil token dari header Authorization: Bearer <token>
    let token = req.headers()
        .get(AUTHORIZATION)
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.strip_prefix("Bearer "))
        .ok_or(AppError::TidakDibenarkan)?;

    // Sahkan token
    let claims = state.jwt.sahkan_token(token)
        .map_err(|e| {
            tracing::warn!("Token tidak sah: {}", e);
            AppError::TidakDibenarkan
        })?;

    // Semak blacklist (token yang telah di-revoke)
    if state.redis.ada_dalam_blacklist(&claims.jti).await? {
        tracing::warn!(jti = %claims.jti, "Token dalam blacklist");
        return Err(AppError::TidakDibenarkan);
    }

    // Inject claims ke request extensions
    req.extensions_mut().insert(claims);
    Ok(next.run(req).await)
}

// Extractor untuk guna dalam handlers
pub struct Auth(pub Claims);

#[axum::async_trait]
impl<S> axum::extract::FromRequestParts<S> for Auth
where
    S: Send + Sync,
{
    type Rejection = AppError;

    async fn from_request_parts(
        parts: &mut axum::http::request::Parts,
        _state: &S,
    ) -> Result<Self, Self::Rejection> {
        parts.extensions
            .get::<Claims>()
            .cloned()
            .map(Auth)
            .ok_or(AppError::TidakDibenarkan)
    }
}
```

---

## Token Refresh & Revocation

```rust
// src/auth/token.rs
use crate::{state::AppState, auth::jwt::Claims};

pub struct TokenPasangan {
    pub akses:   String,   // short-lived (15 minit)
    pub semak:   String,   // long-lived (7 hari)
}

pub async fn jana_token_pasangan(
    state: &AppState,
    pengguna: &Pengguna,
    peranan: Vec<String>,
) -> Result<TokenPasangan, AppError> {
    // Akses token — pendek
    let claims_akses = buat_claims(pengguna, peranan.clone(), 900); // 15 min
    let token_akses = state.jwt.jana_token(&claims_akses)?;

    // Refresh token — panjang, simpan dalam DB
    let claims_semak = buat_claims(pengguna, peranan, 7 * 24 * 3600); // 7 hari
    let token_semak = state.jwt.jana_token(&claims_semak)?;

    // Simpan refresh token dalam DB untuk validation
    sqlx::query!(
        "INSERT INTO refresh_tokens (jti, pengguna_id, tamat_pada)
         VALUES (?, ?, DATE_ADD(NOW(), INTERVAL 7 DAY))",
        claims_semak.jti, pengguna.id
    )
    .execute(&state.db)
    .await?;

    Ok(TokenPasangan { akses: token_akses, semak: token_semak })
}

pub async fn revoke_token(
    state: &AppState,
    jti:   &str,
    tamat_pada: i64, // dari claims.exp
) -> Result<(), AppError> {
    // Tambah ke blacklist Redis (TTL = masa sehingga token expire)
    let ttl = tamat_pada - chrono::Utc::now().timestamp();
    if ttl > 0 {
        state.redis.blacklist_token(jti, ttl as u64).await?;
    }
    Ok(())
}
```

---

## 🧠 Brain Teaser #1

Mengapa menggunakan `HS256` (symmetric) vs `RS256` (asymmetric) untuk JWT?

<details>
<summary>👀 Jawapan</summary>

```
HS256 (HMAC-SHA256) — Symmetric:
  - Satu kunci untuk sign DAN verify
  - Mudah setup
  - Laju
  - ✓ Sesuai untuk: single service, monolith
  - ✗ BAHAYA kalau: semua services kongsi rahsia yang sama
     (kalau satu service compromise, semua token boleh dibuat!)

RS256 (RSA-SHA256) — Asymmetric:
  - Private key untuk SIGN (hanya auth service tahu)
  - Public key untuk VERIFY (boleh kongsi ke semua services)
  - ✓ Sesuai untuk: microservices, multiple services verify
  - ✓ Services lain boleh verify tanpa tahu cara buat token
  - ✗ Lebih lambat (RSA operations lebih berat)
  - ✗ Key management lebih kompleks

ES256 (ECDSA) — Asymmetric, lebih baru:
  - Sama seperti RS256 tapi kunci lebih kecil, laju sama
  - Paling disyorkan untuk microservices

PILIHAN PRAKTIKAL:
  Single service          → HS256 (mudah, cukup selamat)
  Microservices           → RS256 atau ES256
  Rahsia mesti dibuat dari:
    BUKAN: "mysecret123"
    YA:    openssl rand -hex 64 → 64 bytes random

Saiz kunci minimum:
  HS256: 256 bits (32 bytes) — MINIMUM
  RS256: 2048 bits — MINIMUM (cadang 4096)
  ES256: 256 bits (selamat!)
```
</details>

---

# BAHAGIAN 2: Password Hashing yang Betul 🔑

## Gunakan Argon2 (Bukan MD5/SHA!)

```rust
// src/auth/password.rs
use argon2::{
    Argon2, PasswordHash, PasswordHasher, PasswordVerifier,
    password_hash::{SaltString, Error as ArgonError},
};
use rand::rngs::OsRng;

// Argon2id adalah winner PHC (Password Hashing Competition)
// Lebih baik dari bcrypt, scrypt, PBKDF2

pub fn hash_kata_laluan(kata_laluan: &str) -> Result<String, ArgonError> {
    let garam = SaltString::generate(&mut OsRng);

    let argon2 = Argon2::default(); // parameter default adalah selamat

    let hash = argon2
        .hash_password(kata_laluan.as_bytes(), &garam)?
        .to_string();

    Ok(hash)
}

pub fn sahkan_kata_laluan(
    kata_laluan: &str,
    hash:        &str,
) -> Result<bool, ArgonError> {
    let hash_parsed = PasswordHash::new(hash)?;

    Ok(Argon2::default()
        .verify_password(kata_laluan.as_bytes(), &hash_parsed)
        .is_ok())
}

// Dengan konfigurasi custom (untuk hardware yang berbeza)
pub fn hash_kata_laluan_kuat(kata_laluan: &str) -> Result<String, ArgonError> {
    use argon2::{Algorithm, Params, Version};

    let params = Params::new(
        65536,  // memori: 64MB
        3,      // iterasi: 3
        4,      // parallelism: 4 threads
        None,
    )?;

    let argon2 = Argon2::new(Algorithm::Argon2id, Version::V0x13, params);
    let garam = SaltString::generate(&mut OsRng);

    Ok(argon2.hash_password(kata_laluan.as_bytes(), &garam)?.to_string())
}

// Peraturan Kata Laluan
pub fn semak_kekuatan_kata_laluan(kata_laluan: &str) -> Result<(), &'static str> {
    if kata_laluan.len() < 12 {
        return Err("Kata laluan mestilah sekurang-kurangnya 12 aksara");
    }
    if !kata_laluan.chars().any(|c| c.is_uppercase()) {
        return Err("Mesti ada sekurang-kurangnya satu huruf besar");
    }
    if !kata_laluan.chars().any(|c| c.is_lowercase()) {
        return Err("Mesti ada sekurang-kurangnya satu huruf kecil");
    }
    if !kata_laluan.chars().any(|c| c.is_ascii_digit()) {
        return Err("Mesti ada sekurang-kurangnya satu digit");
    }
    if !kata_laluan.chars().any(|c| "!@#$%^&*()_+-=[]{}|;:,.<>?".contains(c)) {
        return Err("Mesti ada sekurang-kurangnya satu aksara khas");
    }

    // Semak kata laluan biasa (simplified)
    let biasa = ["password123", "12345678", "qwerty123", "admin123"];
    if biasa.iter().any(|&b| kata_laluan.to_lowercase() == b) {
        return Err("Kata laluan ini terlalu biasa");
    }

    Ok(())
}
```

---

# BAHAGIAN 3: Authorization — RBAC 🎭

## Role-Based Access Control

```rust
// src/auth/peranan.rs
use std::collections::HashSet;

// Definisikan peranan sistem
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub enum Peranan {
    SuperAdmin,
    Admin,
    Pengurus,
    Kakitangan,
    Tetamu,
}

// Definisikan kebenaran (permissions)
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub enum Kebenaran {
    // Pekerja
    PekerjaLihat,
    PekerjaUbah,
    PekerjaBuat,
    PekerjaHapus,
    PekerjaLihatGaji,

    // Kehadiran
    KehadiranDaftar,
    KehadiranLihat,
    KehadiranUbah,
    KehadiranEksport,

    // Laporan
    LaporanLihat,
    LaporanEksport,
    LaporanJana,

    // Pentadbiran
    PentadbiranPengguna,
    PentadbiranSistem,
}

// Peta peranan → kebenaran
pub fn kebenaran_untuk_peranan(peranan: &Peranan) -> HashSet<Kebenaran> {
    match peranan {
        Peranan::SuperAdmin => {
            // Semua kebenaran
            [
                Kebenaran::PekerjaLihat, Kebenaran::PekerjaUbah,
                Kebenaran::PekerjaBuat, Kebenaran::PekerjaHapus,
                Kebenaran::PekerjaLihatGaji,
                Kebenaran::KehadiranDaftar, Kebenaran::KehadiranLihat,
                Kebenaran::KehadiranUbah, Kebenaran::KehadiranEksport,
                Kebenaran::LaporanLihat, Kebenaran::LaporanEksport,
                Kebenaran::LaporanJana,
                Kebenaran::PentadbiranPengguna, Kebenaran::PentadbiranSistem,
            ].into()
        }
        Peranan::Admin => {
            [
                Kebenaran::PekerjaLihat, Kebenaran::PekerjaUbah,
                Kebenaran::PekerjaBuat, Kebenaran::PekerjaHapus,
                Kebenaran::PekerjaLihatGaji,
                Kebenaran::KehadiranLihat, Kebenaran::KehadiranUbah,
                Kebenaran::KehadiranEksport,
                Kebenaran::LaporanLihat, Kebenaran::LaporanEksport,
                Kebenaran::LaporanJana,
                Kebenaran::PentadbiranPengguna,
            ].into()
        }
        Peranan::Pengurus => {
            [
                Kebenaran::PekerjaLihat,
                Kebenaran::KehadiranLihat, Kebenaran::KehadiranUbah,
                Kebenaran::LaporanLihat, Kebenaran::LaporanEksport,
            ].into()
        }
        Peranan::Kakitangan => {
            [
                Kebenaran::KehadiranDaftar,
                Kebenaran::KehadiranLihat,  // lihat rekod sendiri sahaja
            ].into()
        }
        Peranan::Tetamu => HashSet::new(), // tiada kebenaran
    }
}

pub fn ada_kebenaran(
    peranan_list: &[String],
    kebenaran:    &Kebenaran,
) -> bool {
    peranan_list.iter().any(|p| {
        let peranan = match p.as_str() {
            "super_admin" => Peranan::SuperAdmin,
            "admin"       => Peranan::Admin,
            "pengurus"    => Peranan::Pengurus,
            "kakitangan"  => Peranan::Kakitangan,
            _             => Peranan::Tetamu,
        };
        kebenaran_untuk_peranan(&peranan).contains(kebenaran)
    })
}
```

---

## Authorization Middleware

```rust
// src/middleware/kebenaran.rs
use axum::{middleware::Next, extract::Request, response::Response};
use crate::{auth::{jwt::Claims, peranan::{Kebenaran, ada_kebenaran}}, error::AppError};

// Macro untuk buat middleware kebenaran
macro_rules! semak_kebenaran {
    ($nama:ident, $kebenaran:expr) => {
        pub async fn $nama(
            req: Request,
            next: Next,
        ) -> Result<Response, AppError> {
            let claims = req.extensions()
                .get::<Claims>()
                .ok_or(AppError::TidakDibenarkan)?;

            if !ada_kebenaran(&claims.peranan, &$kebenaran) {
                tracing::warn!(
                    pengguna = %claims.no_pekerja,
                    kebenaran = ?$kebenaran,
                    "Akses ditolak — kebenaran tidak mencukupi"
                );
                return Err(AppError::Larangan);
            }

            Ok(next.run(req).await)
        }
    };
}

// Jana middleware untuk setiap kebenaran
semak_kebenaran!(boleh_lihat_pekerja,  Kebenaran::PekerjaLihat);
semak_kebenaran!(boleh_ubah_pekerja,   Kebenaran::PekerjaUbah);
semak_kebenaran!(boleh_buat_pekerja,   Kebenaran::PekerjaBuat);
semak_kebenaran!(boleh_hapus_pekerja,  Kebenaran::PekerjaHapus);
semak_kebenaran!(boleh_lihat_gaji,     Kebenaran::PekerjaLihatGaji);
semak_kebenaran!(boleh_eksport,        Kebenaran::KehadiranEksport);
semak_kebenaran!(boleh_admin,          Kebenaran::PentadbiranPengguna);

// IDOR Prevention — pastikan pengguna hanya akses data SENDIRI
pub async fn semak_pemilik_rekod(
    axum::extract::Path(id): axum::extract::Path<u32>,
    req: Request,
    next: Next,
) -> Result<Response, AppError> {
    let claims = req.extensions()
        .get::<Claims>()
        .ok_or(AppError::TidakDibenarkan)?;

    // Admin boleh akses semua
    if ada_kebenaran(&claims.peranan, &Kebenaran::PekerjaLihat) {
        return Ok(next.run(req).await);
    }

    // Pengguna biasa hanya boleh akses rekod sendiri
    let pengguna_id: u32 = claims.sub.parse().unwrap_or(0);
    if pengguna_id != id {
        tracing::warn!(
            pengguna_id, id,
            "IDOR attempt — cuba akses rekod orang lain"
        );
        return Err(AppError::Larangan);
    }

    Ok(next.run(req).await)
}
```

---

# BAHAGIAN 4: Input Validation & Sanitization ✅

## Validation Komprehensif

```rust
// src/models/input.rs
use validator::{Validate, ValidationError};
use serde::Deserialize;
use regex::Regex;

#[derive(Debug, Deserialize, Validate)]
pub struct DaftarPengguna {
    #[validate(
        length(min = 6, max = 20, message = "No pekerja 6-20 aksara"),
        regex(path = "REGEX_NO_PEKERJA", message = "Format: KADA + 3 digit")
    )]
    pub no_pekerja: String,

    #[validate(
        length(min = 2, max = 100),
        custom = "sahkan_nama"
    )]
    pub nama: String,

    #[validate(
        email(message = "Format emel tidak sah"),
        length(max = 255)
    )]
    pub emel: String,

    #[validate(
        length(min = 12, message = "Kata laluan min 12 aksara"),
        custom = "sahkan_kata_laluan"
    )]
    pub kata_laluan: String,

    #[validate(must_match(other = "kata_laluan", message = "Kata laluan tidak sepadan"))]
    pub sahkan_kata_laluan: String,

    #[validate(range(min = 18, max = 70, message = "Umur mesti 18-70 tahun"))]
    pub umur: u8,
}

// Regex patterns
lazy_static::lazy_static! {
    static ref REGEX_NO_PEKERJA: Regex =
        Regex::new(r"^KADA\d{3}$").unwrap();

    static ref REGEX_NAMA: Regex =
        Regex::new(r"^[a-zA-Z\s'-]{2,100}$").unwrap();
}

// Custom validator
fn sahkan_nama(nama: &str) -> Result<(), ValidationError> {
    if !REGEX_NAMA.is_match(nama) {
        return Err(ValidationError::new("nama_tidak_sah")
            .with_message("Nama hanya boleh ada huruf, ruang, tanda petik, dan sempang".into()));
    }
    Ok(())
}

fn sahkan_kata_laluan(kata_laluan: &str) -> Result<(), ValidationError> {
    if !kata_laluan.chars().any(|c| c.is_uppercase()) {
        return Err(ValidationError::new("tiada_huruf_besar")
            .with_message("Perlu sekurang-kurangnya satu huruf besar".into()));
    }
    if !kata_laluan.chars().any(|c| c.is_ascii_digit()) {
        return Err(ValidationError::new("tiada_digit")
            .with_message("Perlu sekurang-kurangnya satu digit".into()));
    }
    if !kata_laluan.chars().any(|c| "!@#$%^&*".contains(c)) {
        return Err(ValidationError::new("tiada_simbol")
            .with_message("Perlu sekurang-kurangnya satu simbol (!@#$%^&*)".into()));
    }
    Ok(())
}

// Guna dalam handler
pub async fn daftar(
    Json(body): Json<DaftarPengguna>,
) -> AppResult<impl IntoResponse> {
    // Validate — return 422 kalau tidak sah
    body.validate().map_err(|errors| {
        let mesej: Vec<String> = errors.field_errors()
            .into_iter()
            .flat_map(|(field, errors)| {
                errors.iter().map(move |e| {
                    format!("{}: {}", field, e.message.as_deref().unwrap_or("tidak sah"))
                })
            })
            .collect();
        AppError::InputTidakSah(mesej.join("; "))
    })?;

    // ... teruskan ...
    Ok(())
}
```

---

## Sanitize Input

```rust
// src/utils/sanitize.rs

// Trim dan normalize whitespace
pub fn trim_bersih(input: &str) -> String {
    input.split_whitespace().collect::<Vec<_>>().join(" ")
}

// Buang aksara HTML berbahaya
pub fn escape_html(input: &str) -> String {
    input
        .replace('&',  "&amp;")
        .replace('<',  "&lt;")
        .replace('>',  "&gt;")
        .replace('"',  "&quot;")
        .replace('\'', "&#x27;")
        .replace('/',  "&#x2F;")
}

// Sanitize untuk output dalam URL
pub fn sanitize_url_param(input: &str) -> String {
    urlencoding::encode(input).to_string()
}

// Semak dan hadkan panjang
pub fn hadkan_panjang(input: &str, maks: usize) -> &str {
    if input.len() > maks {
        &input[..maks]
    } else {
        input
    }
}

// Buang null bytes (untuk file path injection)
pub fn buang_null(input: &str) -> String {
    input.replace('\0', "")
}

// Hadkan path traversal
pub fn selamat_nama_fail(input: &str) -> Option<String> {
    let bersih = input
        .replace("..", "")  // path traversal
        .replace('/', "")
        .replace('\\', "")
        .replace('\0', "");

    if bersih.is_empty() || bersih.len() > 255 {
        None
    } else {
        Some(bersih)
    }
}
```

---

# BAHAGIAN 5: SQL Injection Prevention 💉

## Guna Parameterized Queries Sentiasa

```rust
// src/db/pekerja.rs

// ✅ SELAMAT — parameterized query
pub async fn cari_pekerja(
    db:    &MySqlPool,
    cari:  &str,
    had:   u32,
) -> AppResult<Vec<Pekerja>> {
    // sqlx query! macro auto-parameterize
    // MUSTAHIL ada SQL injection!
    let hasil = sqlx::query_as!(
        Pekerja,
        "SELECT * FROM pekerja WHERE nama LIKE ? LIMIT ?",
        format!("%{}%", cari),  // nilai di-escape oleh driver
        had
    )
    .fetch_all(db)
    .await?;

    Ok(hasil)
}

// ✅ SELAMAT — guna bind
pub async fn cari_dengan_filter(
    db:     &MySqlPool,
    filter: &FilterPekerja,
) -> AppResult<Vec<Pekerja>> {
    let mut query = sqlx::QueryBuilder::new(
        "SELECT * FROM pekerja WHERE 1=1"
    );

    if let Some(ref nama) = filter.nama {
        query.push(" AND nama LIKE ");
        query.push_bind(format!("%{}%", nama));
    }

    if let Some(ref bahagian) = filter.bahagian {
        query.push(" AND bahagian = ");
        query.push_bind(bahagian);
    }

    if let Some(aktif) = filter.aktif {
        query.push(" AND aktif = ");
        query.push_bind(aktif);
    }

    query.push(" ORDER BY nama LIMIT ");
    query.push_bind(filter.had.unwrap_or(50));

    query.build_query_as::<Pekerja>()
        .fetch_all(db)
        .await
        .map_err(Into::into)
}

// ❌ BERBAHAYA — JANGAN BUAT INI!
pub async fn cari_pekerja_berbahaya(
    db:   &MySqlPool,
    nama: &str,
) -> Result<(), sqlx::Error> {
    // SQL INJECTION VULNERABILITY!
    let query = format!("SELECT * FROM pekerja WHERE nama = '{}'", nama);
    // Input: "'; DROP TABLE pekerja; --" → BENCANA!
    sqlx::query(&query).execute(db).await?;
    Ok(())
}
```

---

# BAHAGIAN 6: XSS Prevention 🛡️

## Defense terhadap Cross-Site Scripting

```rust
// src/security/xss.rs
use ammonia::Builder;

// Guna ammonia untuk sanitize HTML (whitelist approach)
pub fn sanitize_html_selamat(html: &str) -> String {
    // Hanya benarkan tags dan attributes yang selamat
    Builder::default()
        .tags(["p", "br", "b", "i", "em", "strong", "ul", "ol", "li",
               "h1", "h2", "h3", "a"].iter().cloned().collect())
        .allowed_classes(std::collections::HashMap::new())
        .clean(html)
        .to_string()
}

// Escape untuk JSON dalam HTML (prevent JSON injection)
pub fn escape_json_dalam_html(input: &str) -> String {
    input
        .replace('<',  "\\u003c")
        .replace('>',  "\\u003e")
        .replace('&',  "\\u0026")
        .replace('\'', "\\u0027")
}

// Content Security Policy header
pub fn csp_header() -> &'static str {
    "default-src 'self'; \
     script-src 'self' 'nonce-{NONCE}'; \
     style-src 'self' 'unsafe-inline'; \
     img-src 'self' data:; \
     font-src 'self'; \
     connect-src 'self'; \
     frame-ancestors 'none'; \
     base-uri 'self'; \
     form-action 'self'"
}
```

---

## Askama Auto-Escape (Built-in XSS Protection)

```html
<!-- Askama auto-escape HTML oleh default! -->
<!-- templates/profil.html -->

<!-- ✅ SELAMAT — auto-escaped -->
<p>Nama: {{ nama }}</p>
<!-- Kalau nama = "<script>alert('xss')</script>"
     Output: &lt;script&gt;alert(&#x27;xss&#x27;)&lt;/script&gt; -->

<!-- ⚠️ HATI-HATI — hanya guna untuk trusted content! -->
<div>{{ kandungan_html|safe }}</div>
<!-- Guna HANYA bila kandungan_html sudah di-sanitize dengan ammonia -->
```

---

# BAHAGIAN 7: CSRF Protection 🔄

## CSRF Token Implementation

```rust
// src/security/csrf.rs
use rand::{RngCore, rngs::OsRng};
use hex;

// Jana token CSRF yang selamat
pub fn jana_csrf_token() -> String {
    let mut bytes = [0u8; 32];
    OsRng.fill_bytes(&mut bytes);
    hex::encode(bytes)
}

// Middleware CSRF
use axum::{
    middleware::Next,
    extract::{Request, State},
    response::Response,
    http::{Method, StatusCode, header},
};

pub async fn csrf_middleware(
    State(state): State<AppState>,
    req: Request,
    next: Next,
) -> Result<Response, AppError> {
    // Hanya semak method yang ubah data
    let kaedah = req.method().clone();
    if matches!(kaedah, Method::GET | Method::HEAD | Method::OPTIONS) {
        return Ok(next.run(req).await);
    }

    // Ambil token dari header
    let token_header = req.headers()
        .get("x-csrf-token")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("");

    // Ambil token dari session
    let token_sesi = dapatkan_csrf_dari_sesi(&req).await?;

    // Bandingkan menggunakan constant-time comparison!
    if !constant_time_compare(token_header, &token_sesi) {
        tracing::warn!(
            uri = %req.uri(),
            "CSRF token tidak sah atau tiada"
        );
        return Err(AppError::Larangan);
    }

    Ok(next.run(req).await)
}

// PENTING: Guna constant-time comparison!
// Elak timing attack
fn constant_time_compare(a: &str, b: &str) -> bool {
    use std::iter::zip;
    if a.len() != b.len() { return false; }
    zip(a.bytes(), b.bytes())
        .fold(0u8, |acc, (x, y)| acc | (x ^ y)) == 0
}

// Double Submit Cookie Pattern (lebih mudah untuk SPA)
pub async fn semak_double_submit(
    req: Request,
    next: Next,
) -> Result<Response, AppError> {
    let kaedah = req.method().clone();
    if matches!(kaedah, Method::GET | Method::HEAD | Method::OPTIONS) {
        return Ok(next.run(req).await);
    }

    // Token dari cookie
    let token_cookie = req.headers()
        .get(header::COOKIE)
        .and_then(|v| v.to_str().ok())
        .and_then(|s| {
            s.split(';')
                .find(|c| c.trim().starts_with("csrf="))
                .map(|c| c.trim().trim_start_matches("csrf=").to_string())
        });

    // Token dari header X-CSRF-Token
    let token_header = req.headers()
        .get("x-csrf-token")
        .and_then(|v| v.to_str().ok())
        .map(|s| s.to_string());

    match (token_cookie, token_header) {
        (Some(c), Some(h)) if constant_time_compare(&c, &h) => {
            Ok(next.run(req).await)
        }
        _ => Err(AppError::Larangan),
    }
}
```

---

# BAHAGIAN 8: Rate Limiting & Brute Force 🚦

## Rate Limiting dengan governor

```rust
// src/middleware/rate_limit.rs
use governor::{
    Quota, RateLimiter,
    state::{keyed::DashMapStateStore, direct::NotKeyed},
    clock::DefaultClock,
    middleware::NoOpMiddleware,
};
use axum::{
    middleware::Next, extract::{Request, ConnectInfo},
    response::Response, http::StatusCode,
};
use std::{num::NonZeroU32, sync::Arc, net::SocketAddr};

// Had kadar mengikut IP
pub type HadKadarIP = RateLimiter<
    std::net::IpAddr,
    DashMapStateStore<std::net::IpAddr>,
    DefaultClock,
    NoOpMiddleware,
>;

pub fn buat_had_kadar_ip(
    permintaan_per_minit: u32,
) -> Arc<HadKadarIP> {
    Arc::new(RateLimiter::keyed(
        Quota::per_minute(NonZeroU32::new(permintaan_per_minit).unwrap())
    ))
}

pub async fn had_kadar_am(
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
    axum::extract::State(had): axum::extract::State<Arc<HadKadarIP>>,
    req: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let ip = addr.ip();

    match had.check_key(&ip) {
        Ok(_) => Ok(next.run(req).await),
        Err(negatif) => {
            tracing::warn!(?ip, "Had kadar dilampaui");
            Err(StatusCode::TOO_MANY_REQUESTS)
        }
    }
}

// Had kadar lebih ketat untuk endpoint sensitif
// (login, register, forgot-password)
pub struct HadKadarSensitif {
    pub login:    Arc<HadKadarIP>,
    pub daftar:   Arc<HadKadarIP>,
    pub reset:    Arc<HadKadarIP>,
}

impl HadKadarSensitif {
    pub fn baru() -> Self {
        HadKadarSensitif {
            login:  buat_had_kadar_ip(5),    // 5 per minit
            daftar: buat_had_kadar_ip(3),    // 3 per minit
            reset:  buat_had_kadar_ip(2),    // 2 per minit
        }
    }
}

// Account lockout selepas N percubaan gagal
pub struct PenghalangBruteForce {
    cuba:         Arc<dashmap::DashMap<String, u32>>,
    had_cuba:     u32,
    tempoh_kunci: std::time::Duration,
}

impl PenghalangBruteForce {
    pub fn baru(had_cuba: u32, tempoh_minit: u64) -> Self {
        PenghalangBruteForce {
            cuba:         Arc::new(dashmap::DashMap::new()),
            had_cuba,
            tempoh_kunci: std::time::Duration::from_secs(tempoh_minit * 60),
        }
    }

    pub fn rekod_gagal(&self, kunci: &str) -> bool {
        let mut rekod = self.cuba.entry(kunci.to_string()).or_insert(0);
        *rekod += 1;
        *rekod >= self.had_cuba
    }

    pub fn reset(&self, kunci: &str) {
        self.cuba.remove(kunci);
    }

    pub fn dikunci(&self, kunci: &str) -> bool {
        self.cuba.get(kunci).map(|v| *v >= self.had_cuba).unwrap_or(false)
    }
}
```

---

# BAHAGIAN 9: Security Headers 🪖

## Middleware Security Headers

```rust
// src/middleware/security_headers.rs
use axum::{
    middleware::Next,
    extract::Request,
    response::Response,
    http::header::{
        self, HeaderName, HeaderValue,
        CONTENT_SECURITY_POLICY,
        X_FRAME_OPTIONS,
        X_CONTENT_TYPE_OPTIONS,
    },
};

pub async fn tambah_security_headers(
    req: Request,
    next: Next,
) -> Response {
    let mut resp = next.run(req).await;
    let headers = resp.headers_mut();

    // ── Prevent Clickjacking ─────────────────────────────────
    headers.insert(
        X_FRAME_OPTIONS,
        HeaderValue::from_static("DENY"),
    );

    // ── Prevent MIME sniffing ────────────────────────────────
    headers.insert(
        X_CONTENT_TYPE_OPTIONS,
        HeaderValue::from_static("nosniff"),
    );

    // ── Referrer Policy ──────────────────────────────────────
    headers.insert(
        HeaderName::from_static("referrer-policy"),
        HeaderValue::from_static("strict-origin-when-cross-origin"),
    );

    // ── HSTS (Hanya untuk HTTPS!) ─────────────────────────────
    headers.insert(
        header::STRICT_TRANSPORT_SECURITY,
        HeaderValue::from_static("max-age=31536000; includeSubDomains; preload"),
    );

    // ── Content Security Policy ───────────────────────────────
    headers.insert(
        CONTENT_SECURITY_POLICY,
        HeaderValue::from_static(
            "default-src 'self'; \
             script-src 'self'; \
             style-src 'self' 'unsafe-inline'; \
             img-src 'self' data: https:; \
             font-src 'self'; \
             connect-src 'self'; \
             frame-ancestors 'none'; \
             base-uri 'self'; \
             form-action 'self'; \
             upgrade-insecure-requests"
        ),
    );

    // ── Permissions Policy ───────────────────────────────────
    headers.insert(
        HeaderName::from_static("permissions-policy"),
        HeaderValue::from_static(
            "camera=(), microphone=(), geolocation=(self), \
             payment=(), usb=(), bluetooth=()"
        ),
    );

    // ── Buang server info header ─────────────────────────────
    headers.remove("server");
    headers.remove("x-powered-by");

    // ── Cache-Control untuk data sensitif ────────────────────
    headers.insert(
        header::CACHE_CONTROL,
        HeaderValue::from_static("no-store, no-cache, must-revalidate"),
    );

    // ── Prevent XSS (legacy, CSP lebih baik) ─────────────────
    headers.insert(
        HeaderName::from_static("x-xss-protection"),
        HeaderValue::from_static("1; mode=block"),
    );

    resp
}
```

---

# BAHAGIAN 10: Mini Project — Secure KADA API 🔒

```rust
// src/main.rs — Kumpul semua security layers

use axum::{Router, middleware};
use tower::ServiceBuilder;
use tower_http::cors::{CorsLayer, Any};
use std::sync::Arc;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = Config::dari_env()?;
    let state  = AppState::baru(&config).await?;

    // Rate limiters
    let had_am       = middleware::from_fn_with_state(
        Arc::clone(&state.had_kadar.am),
        crate::middleware::rate_limit::had_kadar_am
    );

    // CORS — hadkan origins
    let cors = CorsLayer::new()
        .allow_origin([
            "https://app.kada.gov.my".parse().unwrap(),
            "https://admin.kada.gov.my".parse().unwrap(),
        ])
        .allow_methods([
            axum::http::Method::GET,
            axum::http::Method::POST,
            axum::http::Method::PUT,
            axum::http::Method::DELETE,
        ])
        .allow_headers([
            axum::http::header::CONTENT_TYPE,
            axum::http::header::AUTHORIZATION,
            "x-csrf-token".parse().unwrap(),
            "x-correlation-id".parse().unwrap(),
        ])
        .allow_credentials(true);

    // Routes awam (tiada auth)
    let laluan_awam = Router::new()
        .route("/sihat",        get(handlers::am::sihat))
        .route("/auth/masuk",   post(handlers::auth::log_masuk))
        .route("/auth/semak",   post(handlers::auth::semak_token))
        .layer(middleware::from_fn_with_state(
            Arc::clone(&state.had_kadar.login),
            crate::middleware::rate_limit::had_kadar_sensitif
        ));

    // Routes yang perlu authentication
    let laluan_lindung = Router::new()
        .route("/pekerja",      get(handlers::pekerja::senarai)
                                .post(handlers::pekerja::buat))
        .route("/pekerja/:id",  get(handlers::pekerja::dapatkan)
                                .put(handlers::pekerja::kemaskini)
                                .delete(handlers::pekerja::padam))
        .route("/kehadiran",    get(handlers::kehadiran::senarai)
                                .post(handlers::kehadiran::daftar))
        .route("/auth/keluar",  post(handlers::auth::log_keluar))
        .layer(middleware::from_fn_with_state(
            state.clone(),
            crate::middleware::auth::semak_jwt
        ))
        .layer(middleware::from_fn(
            crate::middleware::csrf::csrf_middleware
        ));

    // Routes admin sahaja
    let laluan_admin = Router::new()
        .route("/admin/pengguna", get(handlers::admin::senarai_pengguna))
        .route("/admin/audit",    get(handlers::admin::audit_trail))
        .layer(middleware::from_fn_with_state(
            state.clone(),
            crate::middleware::auth::semak_jwt
        ))
        .layer(middleware::from_fn(
            crate::middleware::kebenaran::boleh_admin
        ));

    let app = Router::new()
        .merge(laluan_awam)
        .nest("/api/v1", laluan_lindung)
        .nest("/api/v1", laluan_admin)
        // Middleware global — apply ke SEMUA routes
        .layer(
            ServiceBuilder::new()
                .layer(cors)
                .layer(middleware::from_fn(
                    crate::middleware::security_headers::tambah_security_headers
                ))
                .layer(middleware::from_fn(
                    crate::middleware::audit::correlation_id_middleware
                ))
                .layer(had_am)  // rate limit global
                .layer(tower_http::trace::TraceLayer::new_for_http())
        )
        .with_state(state);

    let alamat = format!("{}:{}", config.host, config.port);
    let pendengar = tokio::net::TcpListener::bind(&alamat).await?;
    tracing::info!("Server selamat pada https://{}", alamat);
    axum::serve(pendengar, app).await?;

    Ok(())
}
```

---

# 📋 Rujukan Pantas — Security Checklist

## Authentication

```
✔ Guna Argon2id (bukan MD5/SHA1/bcrypt) untuk hash kata laluan
✔ JWT dengan rahsia yang kuat (min 256 bit random)
✔ Token expire yang pendek (akses: 15-60 minit)
✔ Refresh token dengan rotation
✔ Token revocation (blacklist dalam Redis)
✔ Constant-time comparison untuk token
✗ JANGAN simpan kata laluan dalam plain text
✗ JANGAN log kata laluan atau token
```

## Authorization

```
✔ RBAC — definisikan peranan dan kebenaran dengan jelas
✔ Semak kebenaran di SETIAP endpoint
✔ IDOR prevention — pengguna hanya boleh akses data sendiri
✔ Principle of least privilege — bagi kebenaran minimum
✗ JANGAN bergantung pada ID dalam URL sahaja
✗ JANGAN bypass auth untuk "internal" endpoints
```

## Input & Output

```
✔ Validate SEMUA input (length, format, range, type)
✔ Parameterized queries untuk SEMUA database operations
✔ Escape HTML output (Askama buat ini automatik)
✔ Sanitize HTML yang trusted dengan ammonia
✔ Hadkan saiz upload fail
✔ Semak MIME type, bukan extension sahaja
✗ JANGAN trust data dari client
✗ JANGAN buat SQL dengan string concatenation
```

## Transport & Headers

```
✔ HTTPS sahaja dalam production
✔ HSTS header
✔ CSP header
✔ X-Frame-Options: DENY
✔ X-Content-Type-Options: nosniff
✔ CORS — hadkan origins yang dibenarkan
✔ Buang header server/x-powered-by
✗ JANGAN cache response sensitif
```

## Rate Limiting

```
✔ Had kadar untuk semua endpoints
✔ Had kadar lebih ketat untuk login/register/reset
✔ Account lockout selepas N percubaan gagal
✔ Alert bila had kadar dilampaui
✔ Consider IP allowlist untuk admin endpoints
```

---

## 🏆 Cabaran Akhir

Cuba implement salah satu:

1. **2FA (Two-Factor Authentication)** — TOTP dengan Google Authenticator
2. **OAuth2 Integration** — Log masuk dengan Google/Microsoft
3. **API Key Management** — Jana, rotate, dan revoke API keys
4. **Security Scanning Middleware** — Detect SQL injection patterns dalam request
5. **Honeypot Endpoints** — Endpoint palsu yang alert bila diakses (mengesan bot/scanner)

---

*Security bukan feature — ia adalah keperluan asas.*
*Defense in depth: setiap lapisan melindungi lapisan lain.*
*Asumsikan pengguna adalah adversary, data adalah sasaran.* 🦀
