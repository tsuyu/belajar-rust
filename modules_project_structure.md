# 📦 Modules & Project Structure dalam Rust — Beginner to Advanced

> Panduan lengkap: mod, pub, use, workspace, dan cara susun projek Rust.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Kenapa Modules Wujud?

```
Tanpa modules — projek kecil OK, projek besar KACAU:

  main.rs (5000 baris) ← semua dalam satu fail? 😱

Dengan modules — teratur dan scalable:

  main.rs
  ├── mod auth         → src/auth.rs
  │   ├── mod login    → src/auth/login.rs
  │   └── mod session  → src/auth/session.rs
  ├── mod database     → src/database.rs
  ├── mod api          → src/api/
  │   ├── mod v1       → src/api/v1.rs
  │   └── mod v2       → src/api/v2.rs
  └── mod utils        → src/utils.rs

Macam PHP tapi:
  PHP: require/include (manual, tiada namespace proper)
  Rust: mod (automatic, namespace yang betul, privacy control)
```

---

## Peta Pembelajaran

```
Bab 1  → mod Asas — Inline Modules
Bab 2  → pub — Kawalan Privasi
Bab 3  → use — Import & Paths
Bab 4  → Split ke Fail Berbeza
Bab 5  → Struktur Folder Lanjutan
Bab 6  → pub(crate), pub(super), pub(in path)
Bab 7  → Crate — Library vs Binary
Bab 8  → Workspace — Projek Besar
Bab 9  → External Crates & Cargo.toml
Bab 10 → Mini Project: Susun Projek KADA API
```

---

# BAB 1: mod Asas — Inline Modules 📦

## mod dalam Satu Fail

```rust
// main.rs — semua dalam satu fail dahulu

// Define module inline
mod kuil {
    // Semua dalam module ini adalah PRIVATE secara default
    
    fn rahsia() -> &'static str {
        "ini private — tidak boleh akses dari luar!"
    }

    pub fn awam() -> &'static str {  // pub = boleh akses dari luar
        "ini public"
    }

    // Nested module
    pub mod bilik {
        pub fn masuk() {
            println!("Masuk bilik!");
            super::awam(); // super = naik satu level (ke kuil)
        }
    }
}

fn main() {
    // Akses dengan path penuh
    println!("{}", kuil::awam());
    // kuil::rahsia(); // ← ERROR! private

    kuil::bilik::masuk();
}
```

---

## Hierarki Module

```rust
mod sistem {
    pub mod pengguna {
        pub struct Pengguna {
            pub nama: String,
            umur: u8,  // private field!
        }

        impl Pengguna {
            pub fn baru(nama: &str, umur: u8) -> Self {
                Pengguna { nama: nama.into(), umur }
            }

            pub fn umur(&self) -> u8 {
                self.umur  // boleh akses field private dalam impl
            }
        }

        pub fn salam(p: &Pengguna) {
            println!("Salam, {}!", p.nama);
        }
    }

    pub mod keselamatan {
        // Akses sibling module dengan super
        pub fn semak(nama: &str) -> bool {
            println!("Semak: {}", nama);
            true
        }
    }
}

fn main() {
    let p = sistem::pengguna::Pengguna::baru("Ali", 25);
    sistem::pengguna::salam(&p);

    // Akses field public
    println!("Nama: {}", p.nama);

    // Akses field private — melalui method
    println!("Umur: {}", p.umur());

    // p.umur = 30; // ← ERROR! umur adalah private

    sistem::keselamatan::semak(&p.nama);
}
```

---

## Module Tree — Bayangkan Pokok

```
crate (root)
└── sistem                     mod sistem
    ├── pengguna               mod pengguna
    │   ├── Pengguna           struct
    │   ├── baru()             fn
    │   └── salam()            fn
    └── keselamatan            mod keselamatan
        └── semak()            fn

Path absolut: crate::sistem::pengguna::Pengguna
Path relatif: sistem::pengguna::Pengguna  (dari root)
              super::keselamatan::semak   (dari dalam pengguna)
              self::baru                  (dalam module sendiri)
```

---

## 🧠 Brain Teaser #1

Apa yang akan berlaku bila kod ini dicompile?

```rust
mod peti {
    struct Rahsia(i32);  // tiada pub!

    pub fn buat() -> Rahsia {
        Rahsia(42)
    }

    pub fn baca(r: &Rahsia) -> i32 {
        r.0
    }
}

fn main() {
    let r = peti::buat();   // A
    let n = peti::baca(&r); // B
    println!("{}", n);      // C
    let r2 = peti::Rahsia(99); // D
}
```

<details>
<summary>👀 Jawapan</summary>

- **A** — ✔ OK! `buat()` adalah `pub`, boleh dipanggil dari luar
- **B** — ✔ OK! `baca()` adalah `pub`, boleh dipanggil dari luar
- **C** — ✔ OK! print 42
- **D** — ❌ ERROR! `Rahsia` struct adalah **private** — tidak boleh akses dari luar `peti`

Menarik: walaupun `Rahsia` private, kita boleh **menggunakan** nilai `Rahsia` yang dicipta oleh fungsi `pub` dalam module yang sama. Yang tidak boleh ialah membuat struct `Rahsia` dari luar atau mengaksesnya secara langsung.

Ini pattern **Opaque Type** — hide implementation, dedahkan interface.
</details>

---

# BAB 2: pub — Kawalan Privasi 🔐

## Hierarki Privasi

```
Private (default):
  Hanya boleh diakses dalam module yang sama DAN module anak.

pub:
  Boleh diakses dari mana-mana sahaja.

pub(crate):
  Boleh diakses dalam crate yang sama sahaja. Tidak expose ke luar.

pub(super):
  Boleh diakses dari module parent sahaja.

pub(in path):
  Boleh diakses dari path yang dinyatakan sahaja.
```

```rust
mod bangunan {
    pub mod tingkat_bawah {
        pub fn masuk_umum() {
            println!("Awam boleh masuk");
        }

        pub(super) fn akses_pengurusan() {
            // Hanya bangunan:: boleh panggil
            println!("Pengurusan sahaja");
        }

        pub(crate) fn akses_dalaman() {
            // Semua kod dalam crate ini boleh panggil
            println!("Dalaman crate sahaja");
        }

        fn bilik_server() {
            // Private — hanya dalam mod ini
            println!("Server room");
        }
    }

    // parent boleh panggil pub(super)
    pub fn ujian_pengurusan() {
        tingkat_bawah::akses_pengurusan(); // ✔ pub(super) OK
        tingkat_bawah::masuk_umum();       // ✔ pub OK
    }
}

fn main() {
    bangunan::tingkat_bawah::masuk_umum();  // ✔ pub
    bangunan::ujian_pengurusan();            // ✔ pub

    // bangunan::tingkat_bawah::akses_pengurusan(); // ❌ pub(super)
    bangunan::tingkat_bawah::akses_dalaman();       // ✔ pub(crate)
    // bangunan::tingkat_bawah::bilik_server();     // ❌ private
}
```

---

## pub pada Struct Fields

```rust
#[derive(Debug)]
pub struct Konfigurasi {
    pub  host:    String,    // boleh baca & tulis dari luar
    pub  port:    u16,       // boleh baca & tulis dari luar
         debug:   bool,      // PRIVATE — hanya dalam modul ini
         secret:  String,    // PRIVATE — hanya dalam modul ini
}

impl Konfigurasi {
    pub fn baru(host: &str, port: u16) -> Self {
        Konfigurasi {
            host:   host.into(),
            port,
            debug:  false,
            secret: "abc123".into(),
        }
    }

    pub fn aktifkan_debug(&mut self) {
        self.debug = true;
    }

    pub fn adalah_debug(&self) -> bool {
        self.debug
    }
    // secret tidak didedahkan langsung
}

fn main() {
    let mut cfg = Konfigurasi::baru("localhost", 8080);

    // Field public boleh akses terus
    println!("{}:{}", cfg.host, cfg.port);
    cfg.port = 3000; // ✔

    // Field private hanya melalui method
    cfg.aktifkan_debug();
    println!("Debug: {}", cfg.adalah_debug());

    // cfg.secret = "hacked"; // ❌ ERROR! private
    // cfg.debug = true;      // ❌ ERROR! private
}
```

---

## pub pada Enum

```rust
pub enum Status {
    Aktif,           // semua variant ikut pub enum
    Tidak,
    Pending { sebab: String }, // struct variant juga pub
}

// TIDAK boleh hide individual enum variant (semua pub atau semua tidak)
// Tapi boleh hide seluruh enum dengan tidak tambah pub

mod dalaman {
    // enum ini private kepada module dalaman
    enum StatusDalaman { OK, Rosak }

    pub fn semak() -> bool {
        let s = StatusDalaman::OK;
        matches!(s, StatusDalaman::OK)
    }
}
```

---

# BAB 3: use — Import & Paths 📥

## use Asas

```rust
// Tanpa use — perlu tulis path penuh setiap kali
mod matematik {
    pub mod asas {
        pub fn tambah(a: i32, b: i32) -> i32 { a + b }
        pub fn tolak(a: i32, b: i32) -> i32  { a - b }
    }
    pub mod lanjut {
        pub fn kuasa(b: i32, e: u32) -> i32 { b.pow(e) }
    }
}

fn tanpa_use() {
    let r1 = matematik::asas::tambah(3, 4);
    let r2 = matematik::asas::tolak(10, 3);
    let r3 = matematik::lanjut::kuasa(2, 8);
    println!("{} {} {}", r1, r2, r3);
}

// Dengan use — lebih bersih
use matematik::asas::{tambah, tolak};
use matematik::lanjut::kuasa;

fn dengan_use() {
    let r1 = tambah(3, 4);  // terus guna!
    let r2 = tolak(10, 3);
    let r3 = kuasa(2, 8);
    println!("{} {} {}", r1, r2, r3);
}

fn main() {
    tanpa_use();
    dengan_use();
}
```

---

## use Patterns

```rust
use std::collections::{HashMap, HashSet, BTreeMap}; // import banyak sekaligus
use std::fmt::{self, Display, Debug};                // self = fmt module sendiri
use std::io::prelude::*;                             // glob import (kurang disyorkan)

// Rename dengan as
use std::collections::HashMap as Map;
use std::fmt::Display as Papar;

// Re-export dengan pub use
mod dalaman {
    pub struct Perkara(pub i32);
}

pub use dalaman::Perkara; // eksport semula — pengguna library boleh guna terus

fn main() {
    // Guna rename
    let mut m: Map<&str, i32> = Map::new();
    m.insert("satu", 1);

    // Guna re-export (tanpa perlu tahu tentang dalaman::)
    let p = Perkara(42);
    println!("{}", p.0);
}
```

---

## Nested use Paths

```rust
// Lama (bertele-tele)
use std::io::Read;
use std::io::Write;
use std::io::BufReader;
use std::io::BufWriter;

// Baru (lebih bersih)
use std::io::{Read, Write, BufReader, BufWriter};

// Dengan self
use std::io::{self, Read, Write};
// self bermaksud import 'io' module itu sendiri
// Read, Write import function/trait dalam io

fn guna_io(r: &mut dyn io::Read) {
    let _ = r;
}

// Glob — import semua yang pub (gunakan dengan berhati-hati)
use std::collections::*;  // semua dalam collections

fn main() {
    let _m: HashMap<i32, i32> = HashMap::new(); // dari glob
    let _s: HashSet<i32>      = HashSet::new();
}
```

---

## 🧠 Brain Teaser #2

Padankan `use` dengan kegunaannya:

```rust
// A
use std::fmt;

// B
use std::fmt::Display;

// C
use std::fmt::{self, Display};

// D
use std::fmt::Display as Papar;
```

<details>
<summary>👀 Jawapan</summary>

- **A** — Import modul `fmt` sahaja. Guna sebagai `fmt::Display`, `fmt::Debug`.
- **B** — Import trait `Display` terus. Guna sebagai `Display` sahaja.
- **C** — Import modul `fmt` DAN trait `Display`. Boleh guna `fmt::Debug` dan `Display` serentak.
- **D** — Import `Display` dengan nama alias `Papar`. Guna sebagai `Papar` dalam kod.

**Bila guna C?**
```rust
use std::fmt::{self, Display};

// Guna fmt::Debug (dari self) dan Display secara langsung
fn f<T: Display + fmt::Debug>(val: T) { .. }
```
</details>

---

# BAB 4: Split ke Fail Berbeza 📁

## Cara Rust Split Module ke Fail

```
PHP:
  include 'helper.php';   // explicit include
  require 'config.php';

Rust:
  mod helper;  // deklarasi dalam main.rs/lib.rs
               // Rust cari src/helper.rs ATAU src/helper/mod.rs

Tidak perlu "include" — Rust tahu sendiri!
```

## Struktur Fail Asas

```
src/
├── main.rs          ← root binary
├── auth.rs          ← mod auth
├── database.rs      ← mod database
└── utils.rs         ← mod utils
```

**src/main.rs:**
```rust
// Deklarasi module — Rust cari src/auth.rs
mod auth;
mod database;
mod utils;

fn main() {
    auth::log_masuk("Ali", "password123");
    let conn = database::sambung("localhost");
    utils::log("Program mula");
}
```

**src/auth.rs:**
```rust
// Tiada mod auth { } — fail ini IS the module

pub fn log_masuk(nama: &str, kata_laluan: &str) -> bool {
    println!("Log masuk: {}", nama);
    kata_laluan.len() >= 8
}

fn hash_password(kata_laluan: &str) -> String {
    // private — hanya dalam module ini
    format!("hashed_{}", kata_laluan)
}
```

**src/database.rs:**
```rust
pub struct Sambungan {
    pub host: String,
    port:     u16,
}

pub fn sambung(host: &str) -> Sambungan {
    println!("Sambung ke {}", host);
    Sambungan {
        host: host.into(),
        port: 5432,
    }
}
```

**src/utils.rs:**
```rust
pub fn log(mesej: &str) {
    println!("[LOG] {}", mesej);
}
```

---

## Module dengan Sub-modules (Folder)

```
Bila module ada sub-module, guna folder:

src/
├── main.rs
├── auth.rs          ← boleh ada sub-module
└── auth/            ← folder untuk sub-modules auth
    ├── login.rs     ← sub-module auth::login
    └── session.rs   ← sub-module auth::session
```

**src/auth.rs** (fail untuk mod auth + declare sub-modules):
```rust
// Deklarasi sub-modules
pub mod login;
pub mod session;

// Kod dalam auth module sendiri
pub struct AuthConfig {
    pub jwt_secret: String,
}

pub fn inisialisasi() -> AuthConfig {
    AuthConfig { jwt_secret: "super_secret".into() }
}
```

**src/auth/login.rs:**
```rust
use super::AuthConfig;  // super = naik ke auth module

pub fn log_masuk(nama: &str, kata_laluan: &str, cfg: &AuthConfig) -> bool {
    println!("Login {} dengan secret: {}", nama, cfg.jwt_secret);
    kata_laluan.len() >= 8
}
```

**src/auth/session.rs:**
```rust
pub struct Sesi {
    pub token: String,
    pub pengguna: String,
}

pub fn buat_sesi(pengguna: &str) -> Sesi {
    Sesi {
        token: format!("tok_{}", pengguna),
        pengguna: pengguna.into(),
    }
}
```

**src/main.rs:**
```rust
mod auth;

fn main() {
    let cfg = auth::inisialisasi();
    let ok  = auth::login::log_masuk("Ali", "password123", &cfg);
    if ok {
        let sesi = auth::session::buat_sesi("Ali");
        println!("Sesi: {}", sesi.token);
    }
}
```

---

## mod.rs vs fail.rs (Cara Lama vs Baru)

```
Cara LAMA (mod.rs) — masih berfungsi tapi kurang disyorkan:
  src/
  └── auth/
      ├── mod.rs       ← kod auth module (macam __init__.py Python)
      ├── login.rs
      └── session.rs

Cara BARU (fail.rs) — disyorkan sejak Rust 2018:
  src/
  ├── auth.rs          ← kod auth module
  └── auth/
      ├── login.rs
      └── session.rs

Kenapa baru lebih baik?
  - Semua "main file" untuk module ada nama yang jelas
  - Kurang "mod.rs" yang mengelirukan
  - Editor autocomplete lebih baik

Guna MANA-MANA satu dalam projek — jangan campur!
```

---

# BAB 5: Struktur Folder Lanjutan 🗂️

## Projek Sederhana

```
my-api/
├── Cargo.toml
└── src/
    ├── main.rs              ← entry point
    ├── config.rs            ← mod config
    ├── error.rs             ← mod error (custom errors)
    ├── models/              ← mod models
    │   ├── mod.rs           ← atau models.rs
    │   ├── pengguna.rs
    │   └── produk.rs
    ├── handlers/            ← mod handlers (HTTP handlers)
    │   ├── mod.rs
    │   ├── auth.rs
    │   └── produk.rs
    └── db/                  ← mod db
        ├── mod.rs
        ├── pengguna.rs
        └── produk.rs
```

**src/main.rs:**
```rust
mod config;
mod error;
mod models;
mod handlers;
mod db;

use config::Konfigurasi;

fn main() {
    let cfg = Konfigurasi::dari_env();
    println!("Server: {}:{}", cfg.host, cfg.port);
}
```

**src/config.rs:**
```rust
pub struct Konfigurasi {
    pub host: String,
    pub port: u16,
}

impl Konfigurasi {
    pub fn dari_env() -> Self {
        Konfigurasi {
            host: std::env::var("HOST").unwrap_or("localhost".into()),
            port: std::env::var("PORT")
                .unwrap_or("8080".into())
                .parse()
                .unwrap_or(8080),
        }
    }
}
```

**src/models/mod.rs** (atau src/models.rs):
```rust
pub mod pengguna;
pub mod produk;

// Re-export untuk kemudahan
pub use pengguna::Pengguna;
pub use produk::Produk;
```

**src/models/pengguna.rs:**
```rust
#[derive(Debug, Clone)]
pub struct Pengguna {
    pub id:   u32,
    pub nama: String,
    pub emel: String,
}

impl Pengguna {
    pub fn baru(id: u32, nama: &str, emel: &str) -> Self {
        Pengguna { id, nama: nama.into(), emel: emel.into() }
    }
}
```

---

## prelude Pattern — Re-export Berguna

```rust
// src/prelude.rs — satu tempat import semua benda penting
pub use crate::config::Konfigurasi;
pub use crate::error::{AppError, Result};
pub use crate::models::{Pengguna, Produk};

// Dalam fail lain — import semua dengan satu baris
// use my_crate::prelude::*;
```

---

## 🧠 Brain Teaser #3

Berikan struktur fail untuk projek Rust yang ada:
- Konfigurasi app
- Modul untuk CRUD pengguna (model + DB operation)
- HTTP handlers
- Custom error types

<details>
<summary>👀 Jawapan</summary>

```
src/
├── main.rs
│   // mod config; mod error; mod models; mod db; mod handlers;
│
├── config.rs
│   // pub struct Config { ... }
│
├── error.rs
│   // pub enum AppError { ... }
│   // pub type Result<T> = std::result::Result<T, AppError>;
│
├── models.rs (atau models/)
│   // pub struct Pengguna { ... }
│   // pub mod pengguna;
│
├── models/
│   ├── pengguna.rs  // pub struct Pengguna
│   └── mod.rs       // pub mod pengguna; pub use pengguna::Pengguna;
│
├── db.rs (atau db/)
│   // pub mod pengguna;
│
├── db/
│   ├── mod.rs       // pub mod pengguna;
│   └── pengguna.rs  // pub fn cari(id: u32) -> Result<Pengguna>
│                    // pub fn simpan(p: &Pengguna) -> Result<()>
│
├── handlers.rs (atau handlers/)
│   // pub mod pengguna;
│
└── handlers/
    ├── mod.rs       // pub mod pengguna;
    └── pengguna.rs  // pub async fn get(id: u32) -> Result<...>
                     // pub async fn create(data: ...) -> Result<...>
```

Guna `error::Result` everywhere untuk kurangkan verbose:
```rust
// error.rs
pub type Result<T> = std::result::Result<T, AppError>;

// db/pengguna.rs
use crate::error::Result;
pub fn cari(id: u32) -> Result<Pengguna> { ... }
```
</details>

---

# BAB 6: pub(crate), pub(super), pub(in path) 🎯

## Kawalan Privasi yang Tepat

```rust
// src/lib.rs — library crate

pub mod auth {
    // pub(crate) — boleh guna dalam crate ini, tidak expose ke pengguna library
    pub(crate) struct SesiDalaman {
        pub(crate) token: String,
    }

    pub mod login {
        // pub(super) — hanya auth module yang boleh akses
        pub(super) fn hash(kata_laluan: &str) -> String {
            format!("hash_{}", kata_laluan)
        }

        pub fn log_masuk(kata_laluan: &str) -> bool {
            let h = hash(kata_laluan);  // ✔ dalam module yang sama
            h.len() > 5
        }
    }

    pub fn semak_sesi(token: &str) -> bool {
        // Boleh akses pub(super) dari sibling
        // Tidak boleh akses login::hash (pub(super) bermaksud auth:: tidak boleh)
        println!("Semak token: {}", token);
        true
    }

    pub(crate) fn buat_sesi_dalaman(token: &str) -> SesiDalaman {
        SesiDalaman { token: token.into() }
    }
}

pub mod database {
    use super::auth::SesiDalaman;  // ✔ pub(crate) boleh akses

    pub fn sambung_dengan_sesi(sesi: &SesiDalaman) {
        println!("Sambung dengan token: {}", sesi.token);
    }
}
```

---

## pub(in path) — Sangat Spesifik

```rust
mod a {
    pub mod b {
        // Hanya kod dalam mod a boleh akses ini
        pub(in crate::a) fn fungsi_a_sahaja() {
            println!("Hanya a boleh akses");
        }

        pub fn fungsi_semua() {
            println!("Semua boleh akses");
            fungsi_a_sahaja(); // ✔ dalam a::b, boleh
        }
    }

    pub fn cuba() {
        b::fungsi_a_sahaja(); // ✔ kita dalam a, boleh!
        b::fungsi_semua();    // ✔ pub, boleh
    }
}

fn main() {
    a::cuba();
    a::b::fungsi_semua();       // ✔ pub
    // a::b::fungsi_a_sahaja(); // ❌ pub(in crate::a) — kita bukan dalam a
}
```

---

# BAB 7: Crate — Library vs Binary 📚

## Binary Crate vs Library Crate

```
Binary Crate:
  - Ada src/main.rs
  - Boleh dijalankan (cargo run)
  - Entry point: fn main()
  - Satu projek boleh ada BANYAK binary

Library Crate:
  - Ada src/lib.rs
  - Tidak boleh dijalankan terus
  - Diguna oleh crate lain
  - Satu projek hanya boleh ada SATU library
```

## Projek dengan Kedua-duanya

```
my-project/
├── Cargo.toml
└── src/
    ├── lib.rs      ← library (boleh digunakan oleh orang lain)
    ├── main.rs     ← binary (gunakan library sendiri)
    └── bin/
        ├── server.rs    ← binary tambahan: cargo run --bin server
        └── migrate.rs   ← binary tambahan: cargo run --bin migrate
```

**Cargo.toml:**
```toml
[package]
name    = "my-project"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "my-project"  # binary utama (src/main.rs)

[[bin]]
name = "server"
path = "src/bin/server.rs"

[[bin]]
name = "migrate"
path = "src/bin/migrate.rs"

[lib]
name = "my_project_lib"
path = "src/lib.rs"
```

**src/lib.rs:**
```rust
// Eksport public API library
pub mod auth;
pub mod database;
pub mod models;

// Convenience re-exports
pub use models::Pengguna;
pub use auth::log_masuk;
```

**src/main.rs:**
```rust
// Guna library sendiri dengan nama crate
use my_project_lib::Pengguna;
use my_project_lib::log_masuk;

fn main() {
    println!("Main binary");
}
```

**src/bin/server.rs:**
```rust
use my_project_lib::database;

fn main() {
    println!("HTTP Server");
    database::sambung("localhost");
}
```

---

## Tests dalam Modules

```rust
// src/matematik.rs

pub fn tambah(a: i32, b: i32) -> i32 { a + b }
pub fn bahagi(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 { None } else { Some(a / b) }
}

// Unit tests — dalam fail yang sama
#[cfg(test)]
mod tests {
    use super::*;  // import semua dari matematik.rs

    #[test]
    fn test_tambah() {
        assert_eq!(tambah(2, 3), 5);
        assert_eq!(tambah(-1, 1), 0);
    }

    #[test]
    fn test_bahagi_normal() {
        assert_eq!(bahagi(10.0, 2.0), Some(5.0));
    }

    #[test]
    fn test_bahagi_sifar() {
        assert_eq!(bahagi(5.0, 0.0), None);
    }
}
```

```
tests/                   ← Integration tests
├── integration_test.rs  ← Test public API dari luar
└── common/
    └── mod.rs           ← Helper untuk integration tests
```

**tests/integration_test.rs:**
```rust
// Integration test — guna crate sebagai black box
use my_project_lib::Pengguna;

#[test]
fn test_buat_pengguna() {
    let p = Pengguna::baru(1, "Ali", "ali@email.com");
    assert_eq!(p.nama, "Ali");
}
```

---

# BAB 8: Workspace — Projek Besar 🏢

## Apa Itu Workspace?

```
Workspace = kumpulan crate yang berkongsi:
  - Cargo.lock yang sama
  - Output folder yang sama (target/)
  - Build cache bersama

Sesuai untuk:
  - Monorepo (satu repo, banyak crate)
  - Projek besar yang perlu bahagi jadi bahagian
  - Microservices yang ada shared code
```

## Struktur Workspace

```
kada-workspace/
├── Cargo.toml          ← workspace root
├── crates/
│   ├── kada-core/      ← shared library
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── lib.rs
│   ├── kada-api/       ← HTTP API server
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── main.rs
│   ├── kada-cli/       ← Command line tool
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── main.rs
│   └── kada-worker/    ← Background worker
│       ├── Cargo.toml
│       └── src/
│           └── main.rs
└── target/             ← dikongsi oleh semua crate
```

**kada-workspace/Cargo.toml** (workspace root):
```toml
[workspace]
members = [
    "crates/kada-core",
    "crates/kada-api",
    "crates/kada-cli",
    "crates/kada-worker",
]

# Kongsi dependencies antara semua member
[workspace.dependencies]
tokio    = { version = "1", features = ["full"] }
serde    = { version = "1", features = ["derive"] }
thiserror = "1"
```

**crates/kada-core/Cargo.toml:**
```toml
[package]
name    = "kada-core"
version = "0.1.0"
edition = "2021"

[dependencies]
serde     = { workspace = true }  # guna versi dari workspace
thiserror = { workspace = true }
```

**crates/kada-api/Cargo.toml:**
```toml
[package]
name    = "kada-api"
version = "0.1.0"
edition = "2021"

[dependencies]
kada-core = { path = "../kada-core" }  # depend on sibling crate!
tokio     = { workspace = true }
```

---

## Guna Workspace

```bash
# Build semua
cargo build

# Build satu crate sahaja
cargo build -p kada-api

# Run binary tertentu
cargo run -p kada-api

# Test semua
cargo test

# Test crate tertentu
cargo test -p kada-core
```

---

# BAB 9: External Crates & Cargo.toml 📦

## Cargo.toml — Pengurusan Dependency

```toml
[package]
name        = "kada-mobile-api"
version     = "0.1.0"
edition     = "2021"
authors     = ["Ali Ahmad <ali@kada.gov.my>"]
description = "KADA Mobile App Backend API"
license     = "MIT"

[dependencies]
# Versi tepat
serde       = "1.0.197"
# Versi fleksibel (major.minor.*)
tokio       = { version = "1", features = ["full"] }
# Git dependency
# my-crate  = { git = "https://github.com/user/repo" }
# Path dependency
# kada-core = { path = "../kada-core" }

[dev-dependencies]
# Hanya untuk tests
mockall     = "0.11"
tempfile    = "3"

[build-dependencies]
# Untuk build scripts
build-helper = "0.3"

[features]
# Feature flags
default   = ["database"]
database  = ["sqlx"]
cache     = ["redis"]

[dependencies.sqlx]
version  = "0.7"
optional = true
features = ["mysql", "runtime-tokio"]

[dependencies.redis]
version  = "0.23"
optional = true
```

---

## Guna Feature Flags

```rust
// src/lib.rs
pub mod core;

#[cfg(feature = "database")]
pub mod database;

#[cfg(feature = "cache")]
pub mod cache;

// Dalam kod
pub fn sambung_db(url: &str) {
    #[cfg(feature = "database")]
    {
        println!("Sambung DB: {}", url);
    }

    #[cfg(not(feature = "database"))]
    {
        println!("Database feature tidak aktif");
    }
}
```

```bash
# Build tanpa features
cargo build

# Build dengan feature spesifik
cargo build --features "database,cache"

# Build dengan semua features
cargo build --all-features
```

---

## 🧠 Brain Teaser #4

Seorang pembangun ada projek dengan:
- `auth-lib` — library untuk authentication
- `api-server` — HTTP server yang guna auth-lib
- `cli-tool` — command line yang guna auth-lib

Cara terbaik untuk susun projek ini?

<details>
<summary>👀 Jawapan</summary>

**Workspace** adalah jawapan terbaik!

```
my-project/
├── Cargo.toml          ← workspace
├── auth-lib/
│   ├── Cargo.toml      ← [lib] name = "auth_lib"
│   └── src/lib.rs
├── api-server/
│   ├── Cargo.toml      ← [bin], depend on auth-lib
│   └── src/main.rs
└── cli-tool/
    ├── Cargo.toml      ← [bin], depend on auth-lib
    └── src/main.rs
```

**Workspace Cargo.toml:**
```toml
[workspace]
members = ["auth-lib", "api-server", "cli-tool"]
```

**api-server/Cargo.toml:**
```toml
[dependencies]
auth-lib = { path = "../auth-lib" }
```

**Kebaikan:**
- Share satu `Cargo.lock` — versi dependency konsisten
- Share `target/` — compile sekali, guna untuk semua
- Boleh `cargo test --workspace` untuk test semua sekaligus
- Mudah refactor shared code dalam auth-lib
</details>

---

# BAB 10: Mini Project — Susun Projek KADA API 🏗️

> Bina struktur projek lengkap untuk KADA Mobile API menggunakan semua konsep.

## Struktur Projek

```
kada-api/
├── Cargo.toml
├── .env
├── src/
│   ├── main.rs           ← entry point
│   ├── lib.rs            ← library root (re-exports)
│   ├── config.rs         ← konfigurasi app
│   ├── error.rs          ← custom error types
│   ├── prelude.rs        ← common imports
│   ├── models/
│   │   ├── mod.rs
│   │   ├── pekerja.rs
│   │   └── kehadiran.rs
│   ├── db/
│   │   ├── mod.rs
│   │   ├── pekerja.rs
│   │   └── kehadiran.rs
│   ├── handlers/
│   │   ├── mod.rs
│   │   ├── pekerja.rs
│   │   └── kehadiran.rs
│   └── utils/
│       ├── mod.rs
│       └── gps.rs
└── tests/
    └── integration_test.rs
```

## Kod Lengkap

**Cargo.toml:**
```toml
[package]
name    = "kada-api"
version = "0.1.0"
edition = "2021"

[dependencies]
thiserror = "1"
chrono    = { version = "0.4", features = ["serde"] }
```

---

**src/error.rs:**
```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("Rekod tidak dijumpai: {0}")]
    TidakJumpai(String),

    #[error("Input tidak sah: {0}")]
    InputTidakSah(String),

    #[error("Akses ditolak: {0}")]
    AksesDitolak(String),

    #[error("Ralat dalaman: {0}")]
    Dalaman(String),
}

// Shorthand Result type — guna di mana-mana dalam projek
pub type Result<T> = std::result::Result<T, AppError>;
```

---

**src/config.rs:**
```rust
#[derive(Debug, Clone)]
pub struct Konfigurasi {
    pub host:        String,
    pub port:        u16,
    pub db_url:      String,
    pub jwt_secret:  String,
    pub debug_mode:  bool,
}

impl Konfigurasi {
    pub fn muat() -> Self {
        Konfigurasi {
            host:       std::env::var("HOST").unwrap_or("0.0.0.0".into()),
            port:       std::env::var("PORT")
                            .unwrap_or("8080".into())
                            .parse()
                            .unwrap_or(8080),
            db_url:     std::env::var("DATABASE_URL")
                            .unwrap_or("mysql://localhost/kada_db".into()),
            jwt_secret: std::env::var("JWT_SECRET")
                            .unwrap_or("rahsia".into()),
            debug_mode: std::env::var("DEBUG")
                            .map(|v| v == "true")
                            .unwrap_or(false),
        }
    }

    pub fn url_server(&self) -> String {
        format!("{}:{}", self.host, self.port)
    }
}

impl Default for Konfigurasi {
    fn default() -> Self {
        Self::muat()
    }
}
```

---

**src/models/mod.rs:**
```rust
pub mod pekerja;
pub mod kehadiran;

// Re-export untuk kemudahan
pub use pekerja::Pekerja;
pub use kehadiran::Kehadiran;
```

**src/models/pekerja.rs:**
```rust
use crate::error::{AppError, Result};

#[derive(Debug, Clone)]
pub struct Pekerja {
    pub id:       u32,
    pub nama:     String,
    pub no_kakitangan: String,
    pub bahagian: String,
    pub emel:     Option<String>,
}

impl Pekerja {
    pub fn baru(
        id:       u32,
        nama:     &str,
        no_kaki:  &str,
        bahagian: &str,
    ) -> Self {
        Pekerja {
            id,
            nama:     nama.into(),
            no_kakitangan: no_kaki.into(),
            bahagian: bahagian.into(),
            emel:     None,
        }
    }

    pub fn dengan_emel(mut self, emel: &str) -> Self {
        self.emel = Some(emel.into());
        self
    }

    pub fn validate(&self) -> Result<()> {
        if self.nama.trim().is_empty() {
            return Err(AppError::InputTidakSah("Nama tidak boleh kosong".into()));
        }
        if self.no_kakitangan.len() < 4 {
            return Err(AppError::InputTidakSah(
                format!("No. kakitangan '{}' tidak sah", self.no_kakitangan)
            ));
        }
        Ok(())
    }
}
```

**src/models/kehadiran.rs:**
```rust
use crate::error::Result;

#[derive(Debug, Clone)]
pub struct Koordinat {
    pub latitud:  f64,
    pub longitud: f64,
}

impl Koordinat {
    pub fn baru(lat: f64, lon: f64) -> Self {
        Koordinat { latitud: lat, longitud: lon }
    }
}

#[derive(Debug, Clone)]
pub struct Kehadiran {
    pub id:             u32,
    pub id_pekerja:     u32,
    pub masa_masuk:     String,
    pub masa_keluar:    Option<String>,
    pub koordinat:      Koordinat,
    pub gambar_selfie:  Option<String>,
}

impl Kehadiran {
    pub fn rekod_masuk(
        id:         u32,
        id_pekerja: u32,
        koordinat:  Koordinat,
        masa:       &str,
    ) -> Self {
        Kehadiran {
            id,
            id_pekerja,
            masa_masuk:    masa.into(),
            masa_keluar:   None,
            koordinat,
            gambar_selfie: None,
        }
    }

    pub fn rekod_keluar(&mut self, masa: &str) {
        self.masa_keluar = Some(masa.into());
    }
}
```

---

**src/db/mod.rs:**
```rust
pub mod pekerja;
pub mod kehadiran;
```

**src/db/pekerja.rs:**
```rust
use crate::error::{AppError, Result};
use crate::models::Pekerja;

// Simulasi database — dalam production guna sqlx/diesel
static mut SIMPANAN: Vec<Pekerja> = Vec::new();

pub fn cari_semua() -> Vec<Pekerja> {
    // Simulasi
    vec![
        Pekerja::baru(1, "Ali Ahmad",  "KADA001", "ICT"),
        Pekerja::baru(2, "Siti Hawa",  "KADA002", "HR"),
        Pekerja::baru(3, "Amin Razak", "KADA003", "Kewangan"),
    ]
}

pub fn cari_mengikut_id(id: u32) -> Result<Pekerja> {
    cari_semua()
        .into_iter()
        .find(|p| p.id == id)
        .ok_or_else(|| AppError::TidakJumpai(format!("Pekerja ID {}", id)))
}

pub fn simpan(pekerja: &Pekerja) -> Result<()> {
    pekerja.validate()?;
    println!("[DB] Simpan pekerja: {}", pekerja.nama);
    Ok(())
}
```

**src/db/kehadiran.rs:**
```rust
use crate::error::Result;
use crate::models::Kehadiran;

pub fn rekod(k: &Kehadiran) -> Result<()> {
    println!("[DB] Rekod kehadiran pekerja ID {}", k.id_pekerja);
    println!("     Masa masuk: {}", k.masa_masuk);
    println!("     GPS: ({:.4}, {:.4})",
        k.koordinat.latitud, k.koordinat.longitud);
    Ok(())
}

pub fn cari_hari_ini(id_pekerja: u32) -> Vec<Kehadiran> {
    // Simulasi
    println!("[DB] Cari kehadiran pekerja {} hari ini", id_pekerja);
    vec![]
}
```

---

**src/handlers/mod.rs:**
```rust
pub mod pekerja;
pub mod kehadiran;
```

**src/handlers/pekerja.rs:**
```rust
use crate::db;
use crate::error::{AppError, Result};
use crate::models::Pekerja;

pub fn senarai_pekerja() -> Result<Vec<Pekerja>> {
    let senarai = db::pekerja::cari_semua();
    println!("[API] GET /pekerja → {} rekod", senarai.len());
    Ok(senarai)
}

pub fn dapatkan_pekerja(id: u32) -> Result<Pekerja> {
    println!("[API] GET /pekerja/{}", id);
    db::pekerja::cari_mengikut_id(id)
}

pub fn daftar_pekerja(
    nama:     &str,
    no_kaki:  &str,
    bahagian: &str,
) -> Result<Pekerja> {
    let id_baru = 100; // dalam production, dapat dari DB
    let pekerja = Pekerja::baru(id_baru, nama, no_kaki, bahagian);
    pekerja.validate()?;
    db::pekerja::simpan(&pekerja)?;
    println!("[API] POST /pekerja → ID {}", id_baru);
    Ok(pekerja)
}
```

**src/handlers/kehadiran.rs:**
```rust
use crate::db;
use crate::error::{AppError, Result};
use crate::models::{Kehadiran, Koordinat};
use crate::utils::gps;

pub fn daftar_masuk(
    id_pekerja: u32,
    lat:        f64,
    lon:        f64,
    masa:       &str,
) -> Result<Kehadiran> {
    // Semak GPS dalam kawasan kerja
    if !gps::dalam_kawasan(lat, lon) {
        return Err(AppError::AksesDitolak(
            "Lokasi di luar kawasan kerja".into()
        ));
    }

    let koordinat = Koordinat::baru(lat, lon);
    let kehadiran = Kehadiran::rekod_masuk(1, id_pekerja, koordinat, masa);
    db::kehadiran::rekod(&kehadiran)?;

    println!("[API] POST /kehadiran/masuk → pekerja {}", id_pekerja);
    Ok(kehadiran)
}
```

---

**src/utils/mod.rs:**
```rust
pub mod gps;
```

**src/utils/gps.rs:**
```rust
const PUSAT_LAT: f64 = 6.0538;   // Koordinat KADA Kemubu
const PUSAT_LON: f64 = 102.2503;
const RADIUS_KM: f64 = 1.0;      // 1km radius

pub fn dalam_kawasan(lat: f64, lon: f64) -> bool {
    let jarak = kira_jarak(lat, lon, PUSAT_LAT, PUSAT_LON);
    println!("GPS ({:.4}, {:.4}) jarak dari pejabat: {:.2}km", lat, lon, jarak);
    jarak <= RADIUS_KM
}

fn kira_jarak(lat1: f64, lon1: f64, lat2: f64, lon2: f64) -> f64 {
    // Haversine formula (simplified)
    let dlat = (lat2 - lat1).to_radians();
    let dlon = (lon2 - lon1).to_radians();
    let a = (dlat / 2.0).sin().powi(2)
          + lat1.to_radians().cos()
          * lat2.to_radians().cos()
          * (dlon / 2.0).sin().powi(2);
    let c = 2.0 * a.sqrt().atan2((1.0 - a).sqrt());
    6371.0 * c // km
}
```

---

**src/prelude.rs:**
```rust
// Common imports — guna `use kada_api::prelude::*;` dalam tests
pub use crate::config::Konfigurasi;
pub use crate::error::{AppError, Result};
pub use crate::models::{Kehadiran, Koordinat, Pekerja};
```

---

**src/lib.rs:**
```rust
pub mod config;
pub mod error;
pub mod models;
pub mod db;
pub mod handlers;
pub mod utils;
pub mod prelude;
```

---

**src/main.rs:**
```rust
use kada_api::config::Konfigurasi;
use kada_api::handlers;
use kada_api::error::Result;

fn main() -> Result<()> {
    let cfg = Konfigurasi::muat();
    println!("🚀 KADA Mobile API");
    println!("   Server: {}", cfg.url_server());
    println!("   Debug:  {}", cfg.debug_mode);
    println!();

    // Test handlers
    println!("=== Senarai Pekerja ===");
    let pekerja = handlers::pekerja::senarai_pekerja()?;
    for p in &pekerja {
        println!("  {} — {} ({})", p.id, p.nama, p.bahagian);
    }

    println!("\n=== Daftar Pekerja Baru ===");
    match handlers::pekerja::daftar_pekerja("Zara Lina", "KADA004", "ICT") {
        Ok(p)  => println!("  ✔ Daftar berjaya: {}", p.nama),
        Err(e) => println!("  ✘ Gagal: {}", e),
    }

    println!("\n=== Daftar Masuk (GPS) ===");
    // Koordinat dalam kawasan
    match handlers::kehadiran::daftar_masuk(1, 6.0540, 102.2510, "08:00:00") {
        Ok(k)  => println!("  ✔ Masuk berjaya: {}", k.masa_masuk),
        Err(e) => println!("  ✘ Gagal: {}", e),
    }

    // Koordinat LUAR kawasan
    match handlers::kehadiran::daftar_masuk(2, 3.1570, 101.7120, "08:05:00") {
        Ok(_)  => println!("  ✔ Masuk berjaya"),
        Err(e) => println!("  ✘ Gagal (dijangka): {}", e),
    }

    Ok(())
}
```

---

# 📋 Rujukan Pantas — Modules Cheat Sheet

## Syntax Asas

```rust
// Define module (inline)
mod nama { .. }

// Define module (fail lain) — Rust cari src/nama.rs
mod nama;

// Kawalan privacy
pub              // boleh akses dari mana-mana
pub(crate)       // dalam crate sahaja
pub(super)       // dalam parent module sahaja
pub(in crate::a) // dalam path yang nyatakan sahaja
// tiada pub     // private (default)

// Import
use path::to::Item;
use path::{A, B, C};
use path::{self, A};  // import module + item
use path::*;          // glob (kurang digalakkan)
use path as Alias;    // rename

// Re-export
pub use path::Item;
```

## Path Types

```rust
crate::modul::Item      // dari root crate
super::Item             // naik satu level
self::Item              // module semasa
Item                    // relatif (kalau dah use)
```

## Struktur Fail

```
src/
├── main.rs         // binary root
├── lib.rs          // library root
├── modul.rs        // mod modul; dalam lib.rs/main.rs
└── modul/
    ├── mod.rs      // ATAU modul.rs — jangan guna kedua-dua!
    └── sub.rs      // mod sub; dalam modul/mod.rs atau modul.rs
```

## Cargo.toml Keys

```toml
[lib]           # library crate
[[bin]]         # binary crate (boleh ada banyak)
[workspace]     # workspace root
members = [..]  # crate dalam workspace
[dependencies]  # external dependencies
[dev-dependencies]   # untuk tests sahaja
[build-dependencies] # untuk build scripts
[features]      # optional features
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **Library Crate** — buat `kad-utils` library dengan module `string`, `number`, `date` yang boleh digunakan oleh orang lain
2. **Workspace** — pisahkan projek sedia ada kepada `core-lib`, `api-server`, `cli` dalam workspace
3. **Plugin System** — guna `pub trait Plugin` dalam library, binar load plugins dari modules berbeza
4. **Feature Flags** — projek yang ada feature `sqlite`, `postgres`, `mysql` yang boleh dipilih semasa compile
5. **Monorepo** — setup workspace untuk `frontend-types` (shared types), `backend-api`, `mobile-sync`

---

*Modules & Project Structure in Rust — dari `mod` inline hingga workspace monorepo. Selamat belajar!* 🦀
