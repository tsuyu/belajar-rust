# 📦 Cargo — Panduan Lengkap Package Manager Rust

> Semua yang perlu tahu tentang Cargo: dari `cargo new` hingga workspace, benchmark, dan publish.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu Cargo?

```
Cargo = Package Manager + Build System + Test Runner + Docs Generator
        untuk Rust — semua dalam satu tool!

Bahasa lain:
  PHP    → Composer
  Node   → npm / yarn / pnpm
  Python → pip / poetry
  Java   → Maven / Gradle

Rust → Cargo (satu je, semua dalam satu!)

Cargo buat apa?
  ✔ Buat projek baru
  ✔ Build dan compile kod
  ✔ Jalankan program
  ✔ Urus dependencies (crates)
  ✔ Jalankan tests
  ✔ Generate dokumentasi
  ✔ Publish crate ke crates.io
  ✔ Benchmark performance
  ✔ Format kod
  ✔ Lint kod
```

---

## Peta Pembelajaran

```
Bahagian 1 → Cargo Basics — Projek Baru
Bahagian 2 → Cargo.toml — Fail Konfigurasi
Bahagian 3 → Build, Run & Check
Bahagian 4 → Dependencies — Urus Crates
Bahagian 5 → Tests dengan Cargo
Bahagian 6 → Cargo Profiles
Bahagian 7 → Workspace
Bahagian 8 → Dokumentasi & Publish
Bahagian 9 → Cargo Tools & Plugins
Bahagian 10 → Mini Project: Setup Projek KADA
```

---

# BAHAGIAN 1: Cargo Basics — Projek Baru 🚀

## Pasang Rust & Cargo

```bash
# Install Rust (Cargo datang sekali)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Verify installation
rustc --version    # rustc 1.xx.x (...)
cargo --version    # cargo 1.xx.x (...)

# Update Rust & Cargo
rustup update

# Lihat semua toolchain yang dipasang
rustup show
```

---

## cargo new — Buat Projek Baru

```bash
# Buat binary project (ada main.rs)
cargo new nama-projek

# Buat library project (ada lib.rs)
cargo new nama-lib --lib

# Buat dalam folder semasa (bukan folder baru)
cargo init

# Buat library dalam folder semasa
cargo init --lib

# Contoh:
cargo new kada-api
cargo new kada-core --lib
```

```
Struktur yang dihasilkan:

cargo new kada-api           cargo new kada-core --lib
├── Cargo.toml               ├── Cargo.toml
└── src/                     └── src/
    └── main.rs                  └── lib.rs
```

---

## Struktur Projek Standard

```
nama-projek/
├── Cargo.toml          ← konfigurasi projek
├── Cargo.lock          ← versi tepat semua dependencies (auto-generated)
├── .gitignore          ← auto-generated
│
├── src/
│   ├── main.rs         ← binary entry point
│   ├── lib.rs          ← library entry point (optional)
│   │
│   ├── modul_a.rs      ← mod modul_a; dalam main.rs
│   └── modul_b/
│       ├── mod.rs      ← atau modul_b.rs di luar folder
│       └── sub.rs
│
├── tests/
│   └── integration_test.rs  ← integration tests
│
├── benches/
│   └── benchmark.rs         ← benchmarks
│
├── examples/
│   └── contoh_guna.rs       ← contoh penggunaan
│
└── target/             ← output build (jangan commit!)
    ├── debug/          ← debug build
    └── release/        ← release build
```

---

## 🧠 Brain Teaser #1

Apakah perbezaan antara `Cargo.toml` dan `Cargo.lock`?

<details>
<summary>👀 Jawapan</summary>

```
Cargo.toml                     Cargo.lock
──────────────────────────────────────────────────────
Ditulis OLEH KITA               Di-generate OLEH CARGO (automatik)
Tentukan RANGE versi            Tentukan versi TEPAT
  tokio = "1"                     tokio 1.35.1 → checksum abc...
Boleh commit                    Binary: YA commit
                                Library: boleh .gitignore

Analogi:
  Cargo.toml = menu restoran     Cargo.lock = resit pembelian
  "Saya nak versi 1.x"           "Anda dapat versi 1.35.1 pada 15/01/2024"
```

**Peraturan:**
- **Binary/Application** → commit `Cargo.lock` (ensure reproducible build)
- **Library/crate** → biasanya `.gitignore` `Cargo.lock` (biar user pilih versi)
</details>

---

# BAHAGIAN 2: Cargo.toml — Fail Konfigurasi 📄

## Anatomy Cargo.toml

```toml
# ─── Maklumat Package ─────────────────────────────────────────
[package]
name        = "kada-api"          # nama crate (huruf kecil, dash OK)
version     = "0.1.0"             # Semantic Versioning: MAJOR.MINOR.PATCH
edition     = "2021"              # Rust edition (2015, 2018, 2021)
authors     = ["Ali <ali@email.com>"]
description = "KADA Mobile API Backend"
license     = "MIT"               # atau "Apache-2.0" atau "MIT OR Apache-2.0"
repository  = "https://github.com/user/kada-api"
homepage    = "https://kada.gov.my"
readme      = "README.md"
keywords    = ["api", "mobile", "agriculture"]  # max 5, untuk crates.io
categories  = ["web-programming"]               # dari list crates.io
exclude     = ["tests/*", "benches/*"]          # jangan include bila publish
include     = ["src/**/*", "Cargo.toml"]        # atau explicitly include

# ─── Dependencies ──────────────────────────────────────────────
[dependencies]
# Versi exact
serde = "1.0.197"

# Versi range (caret — default)
tokio = "1"           # = "^1" = >=1.0.0, <2.0.0

# Versi tilde
tokio = "~1.35"       # >=1.35.0, <1.36.0 (minor locked)

# Features
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }

# Git dependency
my-crate = { git = "https://github.com/user/my-crate" }
my-crate = { git = "https://github.com/user/my-crate", branch = "main" }
my-crate = { git = "https://github.com/user/my-crate", tag = "v1.2.3" }
my-crate = { git = "https://github.com/user/my-crate", rev = "abc1234" }

# Path dependency (projek local)
my-lib = { path = "../my-lib" }

# Optional dependency
redis = { version = "0.23", optional = true }

# Rename
old-crate = { package = "old-name", version = "1" }

# ─── Dev Dependencies (test & bench sahaja) ───────────────────
[dev-dependencies]
criterion   = { version = "0.5", features = ["html_reports"] }
mockall     = "0.11"
tempfile    = "3"
pretty_assertions = "1"

# ─── Build Dependencies (build.rs sahaja) ─────────────────────
[build-dependencies]
cc = "1"              # untuk compile C code

# ─── Features ─────────────────────────────────────────────────
[features]
default  = ["database"]          # aktif secara default
database = ["dep:sqlx"]          # "dep:" — mark sebagai dependency feature
cache    = ["dep:redis"]
full     = ["database", "cache"] # gabungan features

# ─── Binary Targets ───────────────────────────────────────────
[[bin]]
name = "kada-api"             # nama binary
path = "src/main.rs"          # default

[[bin]]
name = "migrate"
path = "src/bin/migrate.rs"   # binary tambahan

[[bin]]
name = "seed"
path = "src/bin/seed.rs"

# ─── Library Target ───────────────────────────────────────────
[lib]
name = "kada_core"            # nama library (underscore, bukan dash)
path = "src/lib.rs"
crate-type = ["lib"]          # atau "cdylib", "staticlib", "rlib"

# ─── Example Targets ──────────────────────────────────────────
[[example]]
name = "basic_usage"
path = "examples/basic.rs"

# ─── Test Targets ─────────────────────────────────────────────
[[test]]
name = "integration"
path = "tests/integration.rs"

# ─── Bench Targets ────────────────────────────────────────────
[[bench]]
name = "main_bench"
path = "benches/main.rs"
harness = false               # untuk criterion

# ─── Patch (override dependency) ──────────────────────────────
[patch.crates-io]
# Gantikan crate dari crates.io dengan versi local
my-crate = { path = "../my-crate" }

# ─── Profile ──────────────────────────────────────────────────
[profile.dev]
opt-level = 0           # 0=tiada optimasi, 3=optimasi penuh
debug     = true        # include debug info
[profile.release]
opt-level = 3
debug     = false
lto       = true        # Link Time Optimization
codegen-units = 1       # lebih lambat compile, lebih laju binary
strip     = true        # buang debug symbols

# ─── Workspace ────────────────────────────────────────────────
# (dalam workspace root Cargo.toml)
[workspace]
members = ["crates/*", "apps/*"]
resolver = "2"           # dependency resolver version

[workspace.dependencies]
tokio  = { version = "1", features = ["full"] }
serde  = { version = "1", features = ["derive"] }
```

---

## Semantic Versioning — Cara Baca Versi

```
Format: MAJOR.MINOR.PATCH
  MAJOR: Breaking changes
  MINOR: New features (backward compatible)
  PATCH: Bug fixes

Cargo.toml version specifiers:
  "1"         = ^1.0.0    ≥1.0.0, <2.0.0  (caret default)
  "1.2"       = ^1.2.0    ≥1.2.0, <2.0.0
  "1.2.3"     = ^1.2.3    ≥1.2.3, <2.0.0
  "~1.2"      = tilde     ≥1.2.0, <1.3.0
  "~1.2.3"    = tilde     ≥1.2.3, <1.3.0
  "=1.2.3"    = exact     tepat 1.2.3
  ">=1.2, <2" = range     ≥1.2.0 DAN <2.0.0
  "*"         = wildcard  apa-apa versi (tidak disyorkan!)

Contoh popular:
  tokio  = "1"       → latest 1.x
  serde  = "1"       → latest 1.x
  axum   = "0.7"     → ≥0.7.0, <0.8.0 (0.x = tiap minor mungkin breaking)
```

---

# BAHAGIAN 3: Build, Run & Check ⚙️

## Perintah Asas

```bash
# ─── Build ─────────────────────────────────────────────────────

# Build debug mode (lalai)
cargo build
# Output: target/debug/nama-projek

# Build release mode (dioptimasi)
cargo build --release
# Output: target/release/nama-projek

# Build semua targets (tests, examples, benches)
cargo build --all-targets

# Build untuk target platform lain (cross-compile)
cargo build --target x86_64-unknown-linux-musl

# ─── Run ───────────────────────────────────────────────────────

# Build + jalankan (debug mode)
cargo run

# Build + jalankan (release mode)
cargo run --release

# Jalankan binary tertentu (kalau ada banyak [[bin]])
cargo run --bin migrate
cargo run --bin seed

# Hantar arguments ke program
cargo run -- --port 8080 --debug
#            ^^ semua selepas -- adalah args untuk program

# Jalankan example
cargo run --example basic_usage

# ─── Check ─────────────────────────────────────────────────────

# Semak kod (tanpa build binary) — lebih laju!
cargo check

# Check semua targets
cargo check --all-targets

# ─── Clean ─────────────────────────────────────────────────────

# Padam folder target/
cargo clean

# ─── Semak Output ──────────────────────────────────────────────

# Lihat apa yang dicompile
cargo build --verbose

# Lihat masa compile setiap crate
cargo build --timings

# Lihat dependency tree
cargo tree

# Lihat dependency tertentu
cargo tree -p tokio
```

---

## cargo check vs cargo build

```
cargo check:
  ✔ Lebih LAJU (3-10x lebih cepat dari build)
  ✔ Semak syntax, types, borrow checker
  ✔ Tidak hasilkan binary
  ✔ Guna untuk development (save, check, save, check...)

cargo build:
  ✔ Hasilkan binary
  ✔ Compile semua
  ✗ Lebih lambat

Workflow biasa:
  1. Tulis kod
  2. cargo check     ← semak error cepat
  3. Betulkan error
  4. cargo check     ← semak lagi
  5. Bila dah OK: cargo run
```

---

## Build dengan Features

```bash
# Build dengan feature tertentu
cargo build --features "database,cache"

# Build dengan semua features
cargo build --all-features

# Build tanpa default features
cargo build --no-default-features

# Build tanpa default + tambah specific features
cargo build --no-default-features --features "database"

# Run dengan features
cargo run --features "full"
```

---

## 🧠 Brain Teaser #2

Bila perlu guna `cargo build --release` berbanding `cargo build`?

<details>
<summary>👀 Jawapan</summary>

```
cargo build (debug):
  ✔ Compile LAJU (untuk development)
  ✔ Lebih banyak debug info
  ✔ Panic messages lebih jelas
  ✗ Program jalan LAMBAT (tiada optimasi)
  ✗ Binary saiz lebih besar
  Guna: Development, testing, debugging

cargo build --release:
  ✔ Program jalan LAJU (opt-level=3)
  ✔ Binary lebih kecil
  ✗ Compile LAMBAT (perlu lebih masa)
  ✗ Debug lebih susah
  Guna: Production deployment, benchmarking, final testing

PERATURAN:
  Development hari-hari → cargo build / cargo run
  Nak ukur performance  → cargo build --release (WAJIB!)
  Deploy ke server      → cargo build --release
  Buat benchmark        → cargo bench (auto guna release)

Perhatian: Jangan benchmark debug build — hasilnya tidak
bermakna kerana tiada optimasi!
```
</details>

---

# BAHAGIAN 4: Dependencies — Urus Crates 📚

## cargo add & cargo remove

```bash
# ─── Tambah Dependency ─────────────────────────────────────────

# Tambah crate (versi terbaru)
cargo add tokio

# Tambah dengan features
cargo add tokio --features full
cargo add serde --features derive

# Tambah dengan versi tertentu
cargo add tokio@1.35

# Tambah sebagai dev dependency
cargo add --dev criterion
cargo add --dev mockall

# Tambah sebagai build dependency
cargo add --build cc

# Tambah dengan optional
cargo add redis --optional

# ─── Buang Dependency ──────────────────────────────────────────

cargo remove tokio
cargo remove --dev criterion

# ─── Update Dependencies ───────────────────────────────────────

# Update semua ke versi terbaru (dalam range Cargo.toml)
cargo update

# Update crate tertentu sahaja
cargo update tokio
cargo update serde

# ─── Cari Crates ───────────────────────────────────────────────

# Cari crate di crates.io
cargo search tokio
cargo search "async http"

# Lihat maklumat crate
cargo info tokio
```

---

## Cari Crates yang Betul

```
Sumber utama:
  crates.io      → https://crates.io
                   Registry rasmi Rust
                   Semua crates public ada di sini

  lib.rs         → https://lib.rs
                   Search engine lebih baik, ada kategori

  docs.rs        → https://docs.rs
                   Dokumentasi semua crates secara automatik

Popular crates mengikut kategori:

  Web Framework:
    axum        = "0.7"     → modular, composable
    actix-web   = "4"       → performance tinggi
    rocket      = "0.5"     → ergonomic

  Async Runtime:
    tokio       = "1"       → standard de facto untuk async Rust

  Database:
    sqlx        = "0.7"     → async, SQL raw
    diesel      = "2"       → ORM
    sea-orm     = "0.12"    → async ORM

  Serialization:
    serde       = "1"       → serialize/deserialize semua format
    serde_json  = "1"       → JSON

  Error Handling:
    thiserror   = "1"       → derive macro untuk error
    anyhow      = "1"       → application error handling

  Logging:
    tracing     = "0.1"     → structured logging
    log         = "0.4"     → simple logging facade
    env_logger  = "0.10"    → log ke console

  CLI:
    clap        = "4"       → command line argument parser
    dialoguer   = "0.11"    → interactive CLI

  Configuration:
    config      = "0.13"    → multi-source config
    dotenvy     = "0.15"    → .env file

  HTTP Client:
    reqwest     = "0.11"    → HTTP client (tokio-based)
    ureq        = "2"       → simple sync HTTP

  Date/Time:
    chrono      = "0.4"     → date and time
    time        = "0.3"     → alternatif chrono

  UUID:
    uuid        = "1"       → UUID generation

  Hashing:
    bcrypt      = "0.15"    → password hashing
    sha2        = "0.10"    → SHA hashing
```

---

## cargo tree — Faham Dependency Graph

```bash
# Lihat semua dependencies dalam bentuk tree
cargo tree

# Output contoh:
# nama-projek v0.1.0
# ├── axum v0.7.4
# │   ├── async-trait v0.1.77
# │   ├── bytes v1.5.0
# │   ├── http v1.1.0
# │   └── tokio v1.35.1
# │       ├── bytes v1.5.0 (*)  ← (*) = duplikat, ditunjuk semula
# │       └── ...
# └── serde v1.0.197
#     └── serde_derive v1.0.197 (proc-macro)

# Lihat dependents (siapa guna package ini?)
cargo tree --invert tokio

# Lihat features yang aktif
cargo tree -f "{p} {f}"

# Cari kenapa sesuatu crate masuk
cargo tree -p bytes

# Format berbeza
cargo tree --prefix none    # flat list tanpa prefix
cargo tree --depth 2        # hanya 2 level dalam

# Cari duplicate (crate yang ada lebih satu versi)
cargo tree --duplicates
```

---

# BAHAGIAN 5: Tests dengan Cargo 🧪

## cargo test

```bash
# ─── Jalankan Tests ────────────────────────────────────────────

# Jalankan semua tests
cargo test

# Jalankan tests dalam release mode
cargo test --release

# Jalankan tests dengan nama tertentu
cargo test nama_fungsi_test

# Jalankan semua tests dalam modul tertentu
cargo test modul::

# Jalankan integration tests sahaja
cargo test --test integration_test

# Jalankan doc tests sahaja
cargo test --doc

# Jalankan unit tests sahaja (tiada integration, tiada doc tests)
cargo test --lib

# ─── Test Output ───────────────────────────────────────────────

# Tunjuk output println! dalam tests (biasanya disembunyikan)
cargo test -- --nocapture
# atau
cargo test -- --show-output

# Jalankan tests satu per satu (bukan parallel)
cargo test -- --test-threads=1

# ─── Filter & Skip ─────────────────────────────────────────────

# Jalankan hanya tests yang contain perkataan ini
cargo test tambah

# Skip tests yang contain perkataan ini
cargo test -- --skip slow_test

# List semua tests tanpa jalankan
cargo test -- --list

# ─── Features dalam Test ───────────────────────────────────────

cargo test --features "database"
cargo test --all-features
```

---

## Tulis Tests yang Baik

```rust
// src/matematik.rs — contoh kod yang diuji

pub fn tambah(a: i32, b: i32) -> i32 { a + b }
pub fn bahagi(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 { None } else { Some(a / b) }
}
pub fn adalah_prima(n: u32) -> bool {
    if n < 2 { return false; }
    !(2..=(n as f64).sqrt() as u32).any(|i| n % i == 0)
}

// Unit tests — dalam fail yang sama
#[cfg(test)]
mod tests {
    use super::*;  // import semua dari modul parent

    // Test biasa
    #[test]
    fn test_tambah_asas() {
        assert_eq!(tambah(2, 3), 5);
    }

    // Test dengan banyak cases
    #[test]
    fn test_tambah_pelbagai() {
        assert_eq!(tambah(0, 0), 0);
        assert_eq!(tambah(-1, 1), 0);
        assert_eq!(tambah(-5, -3), -8);
        assert_eq!(tambah(100, 200), 300);
    }

    // Test Option
    #[test]
    fn test_bahagi_normal() {
        assert_eq!(bahagi(10.0, 2.0), Some(5.0));
        assert_eq!(bahagi(0.0, 5.0), Some(0.0));
    }

    #[test]
    fn test_bahagi_sifar() {
        assert_eq!(bahagi(5.0, 0.0), None);
    }

    // Test yang dijangka PANIC
    #[test]
    #[should_panic]
    fn test_yang_panic() {
        let v: Vec<i32> = vec![];
        let _ = v[0]; // index out of bounds!
    }

    // Test yang dijangka panic dengan mesej tertentu
    #[test]
    #[should_panic(expected = "index out of bounds")]
    fn test_panic_dengan_mesej() {
        let v: Vec<i32> = vec![];
        let _ = v[0];
    }

    // Ignore test sementara
    #[test]
    #[ignore]
    fn test_lambat_yang_skip() {
        // cargo test -- --ignored  ← untuk jalankan ini
        std::thread::sleep(std::time::Duration::from_secs(10));
    }
}
```

---

## Integration Tests

```rust
// tests/integration_test.rs
// Integration tests ada dalam folder tests/
// Ia guna crate sebagai library (dari luar)

use nama_projek::matematik::{tambah, bahagi};

#[test]
fn test_matematik_dari_luar() {
    assert_eq!(tambah(100, 200), 300);
    assert!(bahagi(10.0, 3.0).is_some());
    assert!(bahagi(10.0, 0.0).is_none());
}

// tests/common/mod.rs — helper untuk integration tests
pub fn setup() -> Vec<i32> {
    vec![1, 2, 3, 4, 5]
}
```

---

## Doc Tests

```rust
/// Fungsi tambah dua nombor bulat.
///
/// # Contoh
///
/// ```
/// use nama_projek::matematik::tambah;
///
/// assert_eq!(tambah(2, 3), 5);
/// assert_eq!(tambah(-1, 1), 0);
/// ```
///
/// # Panic
///
/// Fungsi ini tidak panic.
pub fn tambah(a: i32, b: i32) -> i32 {
    a + b
}

// cargo test --doc  ← jalankan doc tests!
// Doc tests pastikan dokumentasi sentiasa betul dan up-to-date.
```

---

# BAHAGIAN 6: Cargo Profiles ⚡

## Profiles Explained

```toml
# Cargo.toml

# ── Profile: dev (default untuk cargo build/run) ─────────────
[profile.dev]
opt-level  = 0          # 0-3: tahap optimasi
debug      = true       # include debug symbols
debug-assertions = true # aktifkan debug_assert!()
overflow-checks = true  # semak integer overflow
lto        = false      # Link-Time Optimization
panic      = "unwind"   # "unwind" atau "abort"
incremental = true      # compile incremental (lebih laju rebuild)
codegen-units = 256     # unit kompilasi (lebih = compile laju, run lambat)

# ── Profile: release (untuk cargo build --release) ───────────
[profile.release]
opt-level  = 3          # optimasi penuh
debug      = false
debug-assertions = false
overflow-checks = false
lto        = true       # lebih lambat compile, lebih laju run
panic      = "abort"    # lebih kecil binary
codegen-units = 1       # optimasi lebih baik
strip      = true       # buang debug symbols dari binary

# ── Profile: test (untuk cargo test) ─────────────────────────
[profile.test]
opt-level  = 0
debug      = true

# ── Profile: bench (untuk cargo bench) ───────────────────────
[profile.bench]
opt-level  = 3
debug      = false

# ── Custom Profile ────────────────────────────────────────────
[profile.production]
inherits   = "release"    # turun dari release
lto        = "fat"        # full LTO
opt-level  = "s"          # optimasi untuk saiz (bukan kelajuan)
strip      = "symbols"

[profile.dev-fast]
inherits   = "dev"
opt-level  = 1            # sedikit optimasi untuk dev
```

---

## Override Profile untuk Dependency

```toml
# Build dependencies dengan optimasi walaupun dalam dev mode
# (berguna kalau dependency sangat lambat dalam debug)
[profile.dev.package."*"]
opt-level = 2             # semua dependencies: opt level 2

[profile.dev.package.regex]
opt-level = 3             # regex: opt level 3 (ia sangat lambat tanpa opt)

[profile.dev.package.image]
opt-level = 3             # image processing: opt level 3
```

---

# BAHAGIAN 7: Workspace 🏢

## Setup Workspace

```bash
# Buat workspace baru
mkdir kada-workspace
cd kada-workspace

# Buat Cargo.toml workspace root (JANGAN cargo new!)
cat > Cargo.toml << 'EOF'
[workspace]
members = [
    "crates/kada-core",
    "crates/kada-api",
    "crates/kada-cli",
]
resolver = "2"

[workspace.dependencies]
tokio  = { version = "1", features = ["full"] }
serde  = { version = "1", features = ["derive"] }
thiserror = "1"
EOF

# Buat member crates
cargo new crates/kada-core --lib
cargo new crates/kada-api
cargo new crates/kada-cli
```

---

## Workspace Cargo.toml — Lengkap

```toml
# kada-workspace/Cargo.toml (workspace root)

[workspace]
members = [
    "crates/kada-core",
    "crates/kada-api",
    "crates/kada-cli",
    "crates/kada-worker",
]
resolver = "2"

# Shared dependencies — member crates reference dengan workspace = true
[workspace.dependencies]
# Runtime
tokio     = { version = "1", features = ["full"] }

# Serialization
serde     = { version = "1", features = ["derive"] }
serde_json = "1"

# Error handling
thiserror = "1"
anyhow    = "1"

# Logging
tracing   = "0.1"
tracing-subscriber = "0.3"

# Local crates
kada-core = { path = "crates/kada-core" }

# Shared linting rules
[workspace.lints.rust]
unused_imports = "warn"

[workspace.lints.clippy]
all = "warn"
```

```toml
# crates/kada-api/Cargo.toml — member crate

[package]
name    = "kada-api"
version = "0.1.0"
edition = "2021"

[dependencies]
# Guna versi dari workspace (workspace = true)
tokio     = { workspace = true }
serde     = { workspace = true }
thiserror = { workspace = true }
kada-core = { workspace = true }  # depend on sibling crate!

# Dependency khusus untuk api (tidak dikongsi)
axum      = "0.7"
tower     = "0.4"
```

---

## Perintah Workspace

```bash
# ─── Build ─────────────────────────────────────────────────────

# Build semua member
cargo build

# Build member tertentu
cargo build -p kada-api
cargo build --package kada-core

# Build semua members dengan release
cargo build --release --workspace

# ─── Run ───────────────────────────────────────────────────────

# Jalankan binary dari member tertentu
cargo run -p kada-api
cargo run -p kada-cli -- --help

# ─── Test ──────────────────────────────────────────────────────

# Test semua members
cargo test --workspace

# Test member tertentu
cargo test -p kada-core

# ─── Lain-lain ─────────────────────────────────────────────────

cargo check --workspace     # check semua
cargo clippy --workspace    # lint semua
cargo fmt --all             # format semua
cargo doc --workspace       # generate docs semua

# Lihat semua packages dalam workspace
cargo metadata --format-version 1 | jq '.packages[].name'
```

---

## 🧠 Brain Teaser #3

Bilakah perlu guna Workspace berbanding projek biasa?

<details>
<summary>👀 Jawapan</summary>

```
Guna Workspace bila:

1. Projek besar yang perlu dibahagi kepada bahagian:
   - kada-core (shared library)
   - kada-api  (HTTP server)
   - kada-cli  (command line tool)
   - kada-worker (background jobs)

2. Microservices yang share banyak kod:
   - Setiap service ada binary sendiri
   - Tapi kongsi types, models, utilities

3. Monorepo — semua kod dalam satu repo:
   - Senang manage dependencies (satu Cargo.lock)
   - CI/CD lebih mudah
   - Refactor merentas crates lebih selamat

4. Library + binary dalam satu repo:
   - Library (public API untuk orang lain)
   - Binary (tool untuk penggunaan sendiri)

KEBAIKAN Workspace:
  ✔ Satu Cargo.lock — versi konsisten semua
  ✔ Kongsi target/ folder — compile sekali, guna semua
  ✔ Workspace dependencies — versi crate dikongsi
  ✔ cargo test --workspace — test semua serentak
  ✔ Dependency antara crates adalah local path
  ✗ Setup lebih kompleks dari projek biasa
  ✗ Compile semua crates = lebih lambat dari satu crate

Kalau projek kecil (< 5000 baris) → projek biasa cukup
Kalau projek besar atau multi-binary → pertimbangkan Workspace
```
</details>

---

# BAHAGIAN 8: Dokumentasi & Publish 📖

## Buat Dokumentasi

```bash
# Generate HTML docs untuk projek dan dependencies
cargo doc

# Generate dan buka dalam browser terus
cargo doc --open

# Generate docs untuk semua dependencies (termasuk private items)
cargo doc --document-private-items

# Generate untuk package tertentu dalam workspace
cargo doc -p kada-core --open
```

---

## Tulis Docs yang Baik

```rust
//! # Modul Matematik
//!
//! Koleksi fungsi matematik untuk KADA API.
//!
//! ## Contoh Penggunaan
//!
//! ```rust
//! use kada_core::matematik::tambah;
//! assert_eq!(tambah(2, 3), 5);
//! ```
//!
//! (//! = inner doc comment untuk modul/crate)

/// Tambah dua nombor bulat.
///
/// # Arguments
///
/// * `a` - Nombor pertama
/// * `b` - Nombor kedua
///
/// # Returns
///
/// Jumlah `a` dan `b`.
///
/// # Examples
///
/// ```
/// # use kada_core::matematik::tambah;
/// assert_eq!(tambah(2, 3), 5);
/// assert_eq!(tambah(-1, 1), 0);
/// ```
///
/// # Panics
///
/// Fungsi ini tidak panic.
///
/// # See Also
///
/// * [`tolak`] - untuk penolakan
pub fn tambah(a: i32, b: i32) -> i32 {
    a + b
}
```

---

## Publish ke crates.io

```bash
# ─── Persediaan ────────────────────────────────────────────────

# Daftar di crates.io (pakai GitHub login)
# https://crates.io/settings/tokens

# Simpan token
cargo login

# ─── Semak Sebelum Publish ─────────────────────────────────────

# Semak Cargo.toml betul
cargo verify-project

# Dry run — tunjuk apa akan diinclude tanpa publish
cargo package --list

# Semak kalau akan berjaya publish (tanpa actually publish)
cargo publish --dry-run

# ─── Publish ───────────────────────────────────────────────────

# Publish ke crates.io!
cargo publish

# Publish versi crate dalam workspace
cargo publish -p kada-core

# ─── Selepas Publish ───────────────────────────────────────────

# Yank versi (buang dari results tapi tidak delete)
cargo yank --version 1.0.0
cargo yank --version 1.0.0 --undo  # cancel yank
```

---

# BAHAGIAN 9: Cargo Tools & Plugins 🔧

## Tools Wajib Ada

```bash
# ─── Format Kod ────────────────────────────────────────────────

# Format semua kod mengikut standard Rust
cargo fmt

# Semak kalau kod sudah formatted (untuk CI)
cargo fmt --check

# ─── Lint / Clippy ─────────────────────────────────────────────

# Run clippy linter
cargo clippy

# Clippy dengan semua lints
cargo clippy -- -D warnings   # treat warnings sebagai errors

# Clippy untuk semua packages dalam workspace
cargo clippy --workspace

# ─── Install cargo-edit (cargo add/remove/upgrade) ─────────────
cargo install cargo-edit

cargo add serde             # tambah dependency
cargo remove serde          # buang dependency
cargo upgrade               # upgrade semua ke versi terbaru
cargo upgrade tokio         # upgrade satu crate

# ─── Install cargo-watch (auto rebuild) ────────────────────────
cargo install cargo-watch

cargo watch                 # auto run default (check)
cargo watch -x run          # auto run bila ada perubahan
cargo watch -x test         # auto test bila ada perubahan
cargo watch -x "run -- --port 8080"
cargo watch -x check -x test -x run  # chain commands

# ─── Install cargo-expand (expand macros) ──────────────────────
cargo install cargo-expand

cargo expand                # expand semua macros
cargo expand modul          # expand modul tertentu

# ─── Install cargo-outdated (semak outdated deps) ──────────────
cargo install cargo-outdated

cargo outdated              # lihat deps yang ada update

# ─── Install cargo-audit (security audit) ──────────────────────
cargo install cargo-audit

cargo audit                 # semak vulnerability dalam deps

# ─── Install cargo-nextest (test runner lebih laju) ────────────
cargo install cargo-nextest

cargo nextest run           # lebih laju dari cargo test
cargo nextest run -p modul  # test package tertentu

# ─── Install cargo-flamegraph (profiling) ──────────────────────
cargo install flamegraph

cargo flamegraph            # generate flamegraph SVG

# ─── Install cargo-criterion ────────────────────────────────────
cargo install cargo-criterion

# ─── Install sccache (build cache untuk CI) ────────────────────
cargo install sccache
export RUSTC_WRAPPER=sccache

# ─── Install cargo-deny (license check) ────────────────────────
cargo install cargo-deny

cargo deny check            # check licenses, advisories, bans

# ─── Semua dalam satu (bukan cargo install, tapi useful) ───────
# tokei — count lines of code
# https://github.com/XAMPPRocky/tokei
cargo install tokei
tokei                       # tunjuk statistik kod

# hyperfine — benchmark command line
cargo install hyperfine
hyperfine "cargo build --release"
```

---

## rustfmt.toml — Konfigurasi Formatting

```toml
# rustfmt.toml atau .rustfmt.toml

max_width             = 100    # panjang baris max (default 100)
tab_spaces            = 4      # saiz indent (default 4)
use_tabs              = false  # guna spaces bukan tabs
newline_style         = "Unix" # "Unix", "Windows", "Native"
edition               = "2021"

# Imports
imports_granularity   = "Module"   # "Preserve", "Crate", "Module", "Item"
group_imports         = "StdExternalCrate"  # groupkan imports

# Functions
fn_single_line        = false      # fungsi satu baris?
where_single_line     = false      # where clause satu baris?

# Comments
comment_width         = 80
normalize_comments    = true
wrap_comments         = true

# Misc
trailing_comma        = "Vertical" # "Always", "Never", "Vertical"
use_field_init_shorthand = true    # { x } bukan { x: x }
match_block_trailing_comma = true
```

---

## clippy.toml — Konfigurasi Clippy

```toml
# clippy.toml atau .clippy.toml

# Panjang fungsi max
too-many-arguments-threshold = 7

# Panjang baris
max-fn-params-bools = 3

# Panjang literal
literal-representation-threshold = 5

# Cyclomatic complexity
cyclomatic-complexity-threshold = 25
```

---

## CI/CD dengan Cargo

```yaml
# .github/workflows/ci.yml

name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy

      - name: Cache cargo
        uses: Swatinem/rust-cache@v2

      - name: Format check
        run: cargo fmt --check

      - name: Clippy
        run: cargo clippy --all-targets -- -D warnings

      - name: Build
        run: cargo build --all-targets

      - name: Test
        run: cargo test --all-targets

      - name: Security audit
        run: |
          cargo install cargo-audit
          cargo audit
```

---

# BAHAGIAN 10: Mini Project — Setup Projek KADA API 🏗️

## Buat Projek dari Scratch

```bash
# ─── 1. Setup Workspace ────────────────────────────────────────

mkdir kada-mobile-backend
cd kada-mobile-backend

# Buat workspace Cargo.toml
cat > Cargo.toml << 'TOML'
[workspace]
members = [
    "crates/kada-core",
    "crates/kada-api",
    "crates/kada-cli",
]
resolver = "2"

[workspace.dependencies]
# Async runtime
tokio      = { version = "1", features = ["full"] }
# Web framework
axum       = "0.7"
tower      = "0.4"
tower-http = { version = "0.5", features = ["cors", "trace"] }
# Serialization
serde      = { version = "1", features = ["derive"] }
serde_json = "1"
# Error handling
thiserror  = "1"
anyhow     = "1"
# Logging
tracing    = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
# Config
dotenvy    = "0.15"
# Local crates
kada-core  = { path = "crates/kada-core" }
TOML

# ─── 2. Buat Core Library ──────────────────────────────────────

cargo new crates/kada-core --lib

cat > crates/kada-core/Cargo.toml << 'TOML'
[package]
name    = "kada-core"
version = "0.1.0"
edition = "2021"

[dependencies]
serde     = { workspace = true }
thiserror = { workspace = true }
TOML

# ─── 3. Buat API Binary ────────────────────────────────────────

cargo new crates/kada-api

cat > crates/kada-api/Cargo.toml << 'TOML'
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
tokio      = { workspace = true }
axum       = { workspace = true }
serde      = { workspace = true }
serde_json = { workspace = true }
thiserror  = { workspace = true }
anyhow     = { workspace = true }
tracing    = { workspace = true }
tracing-subscriber = { workspace = true }
dotenvy    = { workspace = true }
kada-core  = { workspace = true }

[dev-dependencies]
tokio-test = "0.4"
TOML

# ─── 4. Buat CLI Tool ──────────────────────────────────────────

cargo new crates/kada-cli

cat > crates/kada-cli/Cargo.toml << 'TOML'
[package]
name    = "kada-cli"
version = "0.1.0"
edition = "2021"

[dependencies]
clap      = { version = "4", features = ["derive"] }
anyhow    = { workspace = true }
kada-core = { workspace = true }
TOML

# ─── 5. Setup tambahan ─────────────────────────────────────────

# .gitignore
cat > .gitignore << 'EOF'
/target
.env
*.env
EOF

# .env
cat > .env << 'EOF'
DATABASE_URL=mysql://localhost/kada_db
JWT_SECRET=dev_secret_key
PORT=8080
LOG_LEVEL=debug
EOF

# rustfmt.toml
cat > rustfmt.toml << 'EOF'
max_width = 100
imports_granularity = "Module"
group_imports = "StdExternalCrate"
EOF

echo "Projek KADA berjaya dibuat!"
```

---

## Verify Setup

```bash
# Check semua OK
cargo check --workspace

# Semua tests OK
cargo test --workspace

# Format semua kod
cargo fmt --all

# Lint
cargo clippy --workspace

# Lihat structure
cargo tree

# Build release
cargo build --release --workspace
```

---

## Struktur Akhir

```
kada-mobile-backend/
├── Cargo.toml              ← Workspace root
├── Cargo.lock              ← Lock file
├── .gitignore
├── .env
├── rustfmt.toml
│
├── crates/
│   ├── kada-core/          ← Shared library
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── models/
│   │       │   ├── mod.rs
│   │       │   ├── pekerja.rs
│   │       │   └── kehadiran.rs
│   │       └── error.rs
│   │
│   ├── kada-api/           ← HTTP API server
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs
│   │       ├── bin/
│   │       │   └── migrate.rs
│   │       ├── handlers/
│   │       ├── middleware/
│   │       └── routes.rs
│   │
│   └── kada-cli/           ← CLI tool
│       ├── Cargo.toml
│       └── src/
│           └── main.rs
│
└── target/                 ← Dikongsi semua crates!
    ├── debug/
    └── release/
        ├── kada-api
        ├── migrate
        └── kada-cli
```

---

# 📋 Rujukan Pantas — Cargo Cheat Sheet

## Perintah Harian

```bash
cargo new nama        # projek binary baru
cargo new nama --lib  # projek library baru
cargo init            # init dalam folder semasa

cargo check           # semak tanpa build (LAJU!)
cargo build           # build debug
cargo build --release # build release (untuk production)
cargo run             # build + run
cargo run --release   # build + run release
cargo run -- args     # hantar args ke program
cargo test            # jalankan semua tests
cargo fmt             # format kod
cargo clippy          # lint kod
cargo clean           # padam target/
```

## Dependencies

```bash
cargo add crate-name              # tambah dependency
cargo add crate@1.2               # versi tertentu
cargo add crate --features feat   # dengan features
cargo add --dev crate             # dev dependency
cargo remove crate                # buang dependency
cargo update                      # update semua
cargo update crate                # update satu
cargo search kata                 # cari di crates.io
cargo tree                        # lihat dependency tree
cargo tree --duplicates           # cari duplicate versions
```

## Workspace

```bash
cargo build -p package     # build package tertentu
cargo test -p package      # test package tertentu
cargo run -p package       # run package tertentu
cargo test --workspace     # test semua
cargo build --workspace    # build semua
```

## Cargo.toml Version Specifiers

```
"1"        = ^1.0.0  → ≥1.0.0 <2.0.0
"1.2"      = ^1.2.0  → ≥1.2.0 <2.0.0
"~1.2"               → ≥1.2.0 <1.3.0
"=1.2.3"             → tepat 1.2.3
">=1, <2"            → range
```

## Tools Berguna

```bash
cargo install cargo-edit      # cargo add/remove/upgrade
cargo install cargo-watch     # auto rebuild on save
cargo install cargo-expand    # lihat expanded macros
cargo install cargo-outdated  # semak outdated deps
cargo install cargo-audit     # security audit
cargo install cargo-nextest   # faster test runner
cargo install flamegraph      # performance profiling
cargo install tokei           # count lines of code
```

## Profiles

```bash
cargo build              # profile: dev
cargo build --release    # profile: release
cargo test               # profile: test
cargo bench              # profile: bench
```

---

## 🏆 Cabaran Akhir

Cuba setup semua ini untuk projek anda sendiri:

1. **Buat workspace** dengan 3 crates: `core`, `api`, `cli`
2. **Setup shared dependencies** dalam workspace root
3. **Tambah CI workflow** dengan GitHub Actions
4. **Configure clippy** untuk enforce code quality
5. **Tulis doc tests** untuk semua fungsi public

---

*Cargo — Swiss Army Knife Rust. Master Cargo = master Rust workflow.* 🦀
