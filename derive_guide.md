# 🔧 Derive dalam Rust — Panduan Lengkap

> Semua tentang `#[derive(...)]`: trait standard, crate popular,
> cara ia berfungsi, dan bila tulis manual.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu Derive?

```
#[derive(Debug, Clone, PartialEq)]

Ini adalah PROC MACRO yang AUTO-GENERATE implementation
trait untuk struct/enum anda.

Tanpa derive — tulis manual (verbose!):
  impl Debug for Pekerja {
      fn fmt(&self, f: &mut Formatter) -> fmt::Result {
          f.debug_struct("Pekerja")
              .field("nama", &self.nama)
              .field("gaji", &self.gaji)
              .finish()
      }
  }

Dengan derive — satu baris!:
  #[derive(Debug)]
  struct Pekerja { nama: String, gaji: f64 }

Derive berfungsi selagi SEMUA FIELD implement trait tersebut!
```

---

## Peta Panduan

```
Bahagian 1  → Debug & Display
Bahagian 2  → Clone & Copy
Bahagian 3  → PartialEq, Eq, PartialOrd, Ord
Bahagian 4  → Hash
Bahagian 5  → Default
Bahagian 6  → From & Into
Bahagian 7  → serde: Serialize & Deserialize
Bahagian 8  → thiserror::Error
Bahagian 9  → Derive Lain yang Popular
Bahagian 10 → Custom Derive
```

---

# BAHAGIAN 1: Debug & Display 🖨️

## Debug — Untuk Programmer

```rust
// #[derive(Debug)] → auto-implement Debug trait
// Membolehkan: {:?} dan {:#?} formatting
// Membolehkan: dbg!() macro

#[derive(Debug)]
struct Pekerja {
    nama:     String,
    id:       u32,
    gaji:     f64,
    aktif:    bool,
}

#[derive(Debug)]
enum StatusKehadiran {
    Hadir,
    Tidak { sebab: String },
    Cuti  { hari: u32 },
}

// Nested struct — semua field mesti implement Debug!
#[derive(Debug)]
struct Jabatan {
    nama:    String,
    ketua:   Pekerja,        // Pekerja sudah ada Debug ✔
    ahli:    Vec<Pekerja>,   // Vec<T> ada Debug kalau T ada Debug ✔
    status:  StatusKehadiran, // StatusKehadiran ada Debug ✔
}

fn main() {
    let p = Pekerja { nama: "Ali Ahmad".into(), id: 1001, gaji: 4500.0, aktif: true };
    let s = StatusKehadiran::Tidak { sebab: "sakit".into() };

    // {:?} — compact, satu baris
    println!("{:?}", p);
    // Pekerja { nama: "Ali Ahmad", id: 1001, gaji: 4500.0, aktif: true }

    // {:#?} — pretty print, multi-line
    println!("{:#?}", p);
    // Pekerja {
    //     nama: "Ali Ahmad",
    //     id: 1001,
    //     gaji: 4500.0,
    //     aktif: true,
    // }

    println!("{:#?}", s);
    // Tidak {
    //     sebab: "sakit",
    // }

    // dbg! macro — debug + print nama variable + lokasi baris
    dbg!(&p);
    // [src/main.rs:30] &p = Pekerja { nama: "Ali Ahmad", ... }
}
```

---

## Display — Untuk Pengguna

```rust
// Display TIDAK boleh derive — kena implement manual!
// Sebab: Rust tidak tahu FORMAT yang anda nak
// (berbeza dari Debug yang ada format standard)

use std::fmt;

#[derive(Debug)]  // derive Debug boleh
struct Ringgit(f64);

// Display kena tulis manual
impl fmt::Display for Ringgit {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "RM{:.2}", self.0)
    }
}

#[derive(Debug)]
struct TitikGPS { lat: f64, lon: f64 }

impl fmt::Display for TitikGPS {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({:.6}°N, {:.6}°E)", self.lat, self.lon)
    }
}

fn main() {
    let harga = Ringgit(49.90);
    println!("{}", harga);   // RM49.90   ← Display (mesra pengguna)
    println!("{:?}", harga); // Ringgit(49.9) ← Debug (untuk programmer)

    let lokasi = TitikGPS { lat: 6.0538, lon: 102.2503 };
    println!("{}", lokasi);  // (6.053800°N, 102.250300°E)
    println!("{:?}", lokasi); // TitikGPS { lat: 6.0538, lon: 102.2503 }
}
```

---

## 🧠 Brain Teaser #1

Kenapa kod ini tidak compile?

```rust
#[derive(Debug)]
struct Rekod {
    nama:  String,
    data:  Vec<u8>,
    kunci: std::sync::MutexGuard<'static, i32>, // ← masalah?
}
```

<details>
<summary>👀 Jawapan</summary>

`MutexGuard` tidak implement `Debug`! Bila guna `#[derive(Debug)]`, Rust auto-generate kod yang memanggil `Debug` untuk SETIAP field. Kalau satu field tidak implement `Debug`, keseluruhan derive gagal.

**Penyelesaian:**

```rust
// Cara 1: Implement Debug secara manual
use std::fmt;

struct Rekod {
    nama:  String,
    data:  Vec<u8>,
    kunci: std::sync::MutexGuard<'static, i32>,
}

impl fmt::Debug for Rekod {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("Rekod")
            .field("nama", &self.nama)
            .field("data", &self.data)
            .field("kunci", &"<MutexGuard>")  // custom description
            .finish()
    }
}

// Cara 2: Skip field yang tak ada Debug
// (perlu crate `derivative` atau implement manual)
```

**Peraturan:** `#[derive(Debug)]` berfungsi **hanya jika semua field implement Debug**.
</details>

---

# BAHAGIAN 2: Clone & Copy 🧬

## Clone — Explicit Deep Copy

```rust
// #[derive(Clone)] → implement Clone trait
// Membolehkan: .clone() untuk buat salinan penuh

#[derive(Debug, Clone)]
struct Konfigurasi {
    host:     String,     // String implement Clone ✔
    port:     u16,        // u16 implement Clone ✔
    pilihan:  Vec<String>, // Vec<String> implement Clone ✔
}

#[derive(Debug, Clone)]
struct Sambungan {
    config: Konfigurasi,  // Konfigurasi kita ada Clone ✔
    id:     u32,
}

fn main() {
    let cfg = Konfigurasi {
        host:    "localhost".into(),
        port:    8080,
        pilihan: vec!["ssl".into(), "timeout=30".into()],
    };

    let cfg2 = cfg.clone(); // salinan penuh — data berasingan!
    println!("{:?}", cfg);  // ✔ cfg masih valid
    println!("{:?}", cfg2); // ✔ cfg2 adalah salinan

    // Clone digunakan untuk:
    // - Hantar data ke thread
    // - Simpan dalam beberapa tempat
    // - Buat backup sebelum modify
}
```

---

## Copy — Implicit Automatic Copy

```rust
// #[derive(Copy, Clone)] → implement Copy trait
// Copy mesti disertai Clone!
// Membolehkan: assignment auto-copy (tiada move)
// HANYA untuk data stack-only (tiada heap allocation)

// ✔ Boleh derive Copy — semua field adalah Copy types
#[derive(Debug, Clone, Copy)]
struct Titik {
    x: f64,  // f64 implement Copy ✔
    y: f64,  // f64 implement Copy ✔
}

#[derive(Debug, Clone, Copy)]
struct Warna {
    r: u8,   // u8 implement Copy ✔
    g: u8,
    b: u8,
}

// ❌ Tidak boleh Copy — ada String (heap data!)
// #[derive(Clone, Copy)]  // ERROR!
// struct NamaTerpakai {
//     nama: String,  // String tidak implement Copy ✗
// }

fn main() {
    let t1 = Titik { x: 3.0, y: 4.0 };
    let t2 = t1;          // COPY — bukan move!
    println!("{:?}", t1); // ✔ t1 masih valid (Copy tipe)
    println!("{:?}", t2);

    fn luas_lingkaran(pusat: Titik, r: f64) -> f64 {
        std::f64::consts::PI * r * r
    }

    luas_lingkaran(t1, 5.0); // t1 di-copy ke fungsi
    println!("{:?}", t1);    // ✔ t1 masih valid!
}
```

---

## Copy vs Clone — Perbandingan

```
                Copy                    Clone
────────────────────────────────────────────────────────
Cara kerja      Implicit (auto)         Explicit (.clone())
Kos             Sangat murah (memcopy)  Mungkin mahal (heap alloc)
Bila berlaku    Bila assign/pass         Bila panggil .clone()
Data            Stack sahaja            Stack + Heap
Contoh          i32, f64, bool, &T      String, Vec<T>
Constraint      Semua field Copy        Semua field Clone
Dengan apa      Copy + Clone            Clone sahaja
```

---

# BAHAGIAN 3: PartialEq, Eq, PartialOrd, Ord ⚖️

## PartialEq & Eq — Kesamaan

```rust
// #[derive(PartialEq)] → membolehkan == dan !=
// #[derive(Eq)]        → Eq ⊃ PartialEq (untuk HashMap key, total equality)

#[derive(Debug, Clone, PartialEq)]
struct PenggunaId {
    id:   u32,
    nama: String,
}

#[derive(Debug, PartialEq)]
enum Status { Aktif, Tidak, Pending }

fn main() {
    let u1 = PenggunaId { id: 1, nama: "Ali".into() };
    let u2 = PenggunaId { id: 1, nama: "Ali".into() };
    let u3 = PenggunaId { id: 2, nama: "Siti".into() };

    println!("{}", u1 == u2); // true  — semua field sama
    println!("{}", u1 == u3); // false — id berbeza
    println!("{}", u1 != u3); // true

    // PartialEq dalam match
    let s = Status::Aktif;
    if s == Status::Aktif { println!("Aktif!"); }

    // Kenapa PartialEq bukan terus Eq?
    // f64 implement PartialEq tapi BUKAN Eq
    // kerana: f64::NAN != f64::NAN  (NaN tidak sama dengan NaN!)
    let nan = f64::NAN;
    println!("{}", nan == nan); // false! (NaN != NaN)

    // Eq = total equality (a == a sentiasa true)
    // PartialEq = partial equality (a == a mungkin tidak true — untuk NaN)

    // Kalau nak guna sebagai HashMap key — perlu Eq!
    use std::collections::HashMap;
    #[derive(Debug, PartialEq, Eq, Hash)] // Eq diperlukan untuk HashMap key
    struct Key { id: u32 }

    let mut map = HashMap::new();
    map.insert(Key { id: 1 }, "satu");
}
```

---

## PartialOrd & Ord — Perbandingan

```rust
// #[derive(PartialOrd)] → membolehkan <, >, <=, >=
// #[derive(Ord)]        → membolehkan .sort(), .max(), .min()

// Perlu ada PartialEq untuk PartialOrd
// Perlu ada Eq untuk Ord

#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]
struct Versi {
    major: u32,
    minor: u32,
    patch: u32,
}

fn main() {
    let v1 = Versi { major: 1, minor: 0, patch: 0 };
    let v2 = Versi { major: 1, minor: 2, patch: 0 };
    let v3 = Versi { major: 2, minor: 0, patch: 0 };

    // Comparison — ikut urutan field dalam struct!
    println!("{}", v1 < v2);  // true  (1.0.0 < 1.2.0)
    println!("{}", v2 < v3);  // true  (1.2.0 < 2.0.0)
    println!("{}", v1 <= v1); // true

    // Sort Vec — perlu Ord!
    let mut versi = vec![v3.clone(), v1.clone(), v2.clone()];
    versi.sort();
    println!("{:?}", versi);
    // [Versi { major: 1, minor: 0, patch: 0 }, ...]

    // Max dan Min
    let terbaru = versi.iter().max().unwrap();
    println!("Terbaru: {:?}", terbaru);

    // PERHATIAN: #[derive(PartialOrd)] bandingkan field MENGIKUT URUTAN!
    // Field pertama dibandingkan dulu, kemudian kedua, dst.
    // Kalau nak order berbeza → implement manual!
}
```

---

## Custom Ordering

```rust
use std::cmp::Ordering;

#[derive(Debug, Eq, PartialEq, Clone)]
struct Pekerja {
    nama:  String,
    gaji:  u32,
    skor:  u32,
}

// Sort mengikut gaji (descending)
impl Ord for Pekerja {
    fn cmp(&self, lain: &Self) -> Ordering {
        lain.gaji.cmp(&self.gaji) // lain.gaji vs self.gaji = descending
    }
}

impl PartialOrd for Pekerja {
    fn partial_cmp(&self, lain: &Self) -> Option<Ordering> {
        Some(self.cmp(lain))
    }
}

fn main() {
    let mut pekerja = vec![
        Pekerja { nama: "Ali".into(),  gaji: 4500, skor: 85 },
        Pekerja { nama: "Siti".into(), gaji: 6000, skor: 92 },
        Pekerja { nama: "Amin".into(), gaji: 3800, skor: 78 },
    ];

    pekerja.sort(); // sort mengikut gaji descending
    for p in &pekerja {
        println!("{}: RM{}", p.nama, p.gaji);
    }
    // Siti: RM6000
    // Ali: RM4500
    // Amin: RM3800

    // sort_by untuk order berbeza tanpa implement Ord
    pekerja.sort_by_key(|p| p.skor); // sort by skor ascending
    pekerja.sort_by(|a, b| b.skor.cmp(&a.skor)); // sort by skor descending
}
```

---

## 🧠 Brain Teaser #2

Apakah hasil sort ini, dan kenapa?

```rust
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord)]
struct Item {
    keutamaan: u8,
    nama:      String,
}

fn main() {
    let mut items = vec![
        Item { keutamaan: 3, nama: "Zara".into() },
        Item { keutamaan: 1, nama: "Mia".into()  },
        Item { keutamaan: 3, nama: "Ali".into()  },
        Item { keutamaan: 1, nama: "Bob".into()  },
        Item { keutamaan: 2, nama: "Siti".into() },
    ];

    items.sort();
    for i in &items { println!("{:?}", i); }
}
```

<details>
<summary>👀 Jawapan</summary>

Output:
```
Item { keutamaan: 1, nama: "Bob" }
Item { keutamaan: 1, nama: "Mia" }
Item { keutamaan: 2, nama: "Siti" }
Item { keutamaan: 3, nama: "Ali" }
Item { keutamaan: 3, nama: "Zara" }
```

**Kenapa?** `#[derive(Ord)]` bandingkan field **mengikut urutan deklarasi** dalam struct:
1. Bandingkan `keutamaan` dulu
2. Kalau `keutamaan` sama, bandingkan `nama` pula (alphabetical)

Jadi `keutamaan: 1, nama: "Bob"` < `keutamaan: 1, nama: "Mia"` kerana `"Bob" < "Mia"` secara alphabetical.

Jika anda nak sort mengikut `keutamaan` sahaja (ignore `nama`), perlu implement `Ord` manual!
</details>

---

# BAHAGIAN 4: Hash 🔑

## Hash — Untuk HashMap & HashSet

```rust
use std::collections::{HashMap, HashSet};

// #[derive(Hash)] → membolehkan guna sebagai key dalam HashMap/HashSet
// Perlu juga: PartialEq + Eq

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct PengecamPekerja {
    no_pekerja: String,
    bahagian:   String,
}

// Enum pun boleh derive Hash!
#[derive(Debug, PartialEq, Eq, Hash, Clone)]
enum Bahagian { ICT, HR, Kewangan, Pertanian }

fn main() {
    // Guna sebagai HashMap key
    let mut direktori: HashMap<PengecamPekerja, String> = HashMap::new();

    direktori.insert(
        PengecamPekerja { no_pekerja: "KADA001".into(), bahagian: "ICT".into() },
        "Ali Ahmad".into()
    );
    direktori.insert(
        PengecamPekerja { no_pekerja: "KADA002".into(), bahagian: "HR".into() },
        "Siti Hawa".into()
    );

    // Cari
    let kunci = PengecamPekerja { no_pekerja: "KADA001".into(), bahagian: "ICT".into() };
    println!("{:?}", direktori.get(&kunci)); // Some("Ali Ahmad")

    // Guna dalam HashSet
    let mut bahagian_set: HashSet<Bahagian> = HashSet::new();
    bahagian_set.insert(Bahagian::ICT);
    bahagian_set.insert(Bahagian::HR);
    bahagian_set.insert(Bahagian::ICT); // duplikat — diabaikan!
    println!("{}", bahagian_set.len()); // 2

    // PERATURAN PENTING:
    // Jika a == b, maka hash(a) == hash(b) MESTI benar!
    // (tapi hash(a) == hash(b) tidak bermakna a == b — collision boleh berlaku)
    // Bila implement PartialEq manual, Hash pun mesti manual!
}
```

---

## Custom Hash — Bila Nak Hash Sebahagian Field

```rust
use std::hash::{Hash, Hasher};
use std::collections::HashMap;

// Hash hanya berdasarkan 'id', bukan 'nama'
#[derive(Debug)]
struct Pengguna {
    id:   u32,
    nama: String, // nama boleh berubah, id tidak
}

// PartialEq berdasarkan id sahaja
impl PartialEq for Pengguna {
    fn eq(&self, lain: &Self) -> bool {
        self.id == lain.id
    }
}
impl Eq for Pengguna {}

// Hash berdasarkan id sahaja — MESTI konsisten dengan PartialEq!
impl Hash for Pengguna {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.id.hash(state); // hanya hash id
        // tidak hash nama — sebab PartialEq hanya guna id
    }
}

fn main() {
    let p1 = Pengguna { id: 1, nama: "Ali".into() };
    let p2 = Pengguna { id: 1, nama: "Ali Ahmad".into() }; // nama berbeza!

    // p1 == p2 kerana id sama
    println!("{}", p1 == p2); // true

    let mut map: HashMap<Pengguna, String> = HashMap::new();
    map.insert(p1, "Jabatan ICT".into());

    // Cari dengan p2 (id sama, nama berbeza) — BERJAYA!
    let cari = Pengguna { id: 1, nama: "nama tidak penting".into() };
    println!("{:?}", map.get(&cari)); // Some("Jabatan ICT")
}
```

---

# BAHAGIAN 5: Default 📋

## Default — Nilai Lalai

```rust
// #[derive(Default)] → implement Default trait
// Membolehkan: Type::default() atau Default::default()
// Semua field mesti implement Default!

// Default values:
// bool → false
// integer → 0
// float → 0.0
// String → ""
// Vec → vec![]
// Option → None
// () → ()

#[derive(Debug, Default)]
struct Tetapan {
    debug:       bool,     // false
    log_level:   u8,       // 0
    nama_server: String,   // ""
    pilihan:     Vec<String>, // []
    tamat_saat:  Option<u32>, // None
}

fn main() {
    // Cara 1: ::default()
    let t = Tetapan::default();
    println!("{:#?}", t);

    // Cara 2: Struct update syntax — guna default untuk field yang tidak dinyatakan
    let t2 = Tetapan {
        debug:     true,
        log_level: 2,
        ..Tetapan::default() // baki guna default
    };
    println!("{:#?}", t2);

    // Dalam fungsi yang mungkin tiada parameter
    fn buat_dengan_pilihan(tetapan: Option<Tetapan>) -> Tetapan {
        tetapan.unwrap_or_default() // guna default kalau None!
    }

    let t3 = buat_dengan_pilihan(None);
    println!("{:?}", t3.debug); // false
}
```

---

## Custom Default

```rust
// Kalau nilai default bukan 0/false/"", implement manual!

#[derive(Debug)]
struct KonfigServer {
    host:     String,
    port:     u16,
    workers:  u32,
    debug:    bool,
}

impl Default for KonfigServer {
    fn default() -> Self {
        KonfigServer {
            host:    "0.0.0.0".into(),  // bukan ""
            port:    8080,               // bukan 0
            workers: 4,                  // bukan 0
            debug:   false,
        }
    }
}

fn main() {
    let cfg = KonfigServer::default();
    println!("{}:{}", cfg.host, cfg.port); // 0.0.0.0:8080

    // Override sebahagian
    let cfg_dev = KonfigServer {
        debug: true,
        port: 3000,
        ..KonfigServer::default()
    };
    println!("{}:{} debug={}", cfg_dev.host, cfg_dev.port, cfg_dev.debug);
    // 0.0.0.0:3000 debug=true
}
```

---

# BAHAGIAN 6: From & Into 🔄

## From — Conversion

```rust
// From TIDAK boleh derive secara standard.
// Perlu implement manual.
// TAPI, bila implement From<A> for B,
// Rust AUTO-derive Into<B> for A!

#[derive(Debug)]
struct Meter(f64);

#[derive(Debug)]
struct Kilometer(f64);

// Implement From — dapat Into percuma!
impl From<Meter> for Kilometer {
    fn from(m: Meter) -> Self {
        Kilometer(m.0 / 1000.0)
    }
}

impl From<Kilometer> for Meter {
    fn from(k: Kilometer) -> Self {
        Meter(k.0 * 1000.0)
    }
}

fn main() {
    let m = Meter(1500.0);

    // Guna From
    let km = Kilometer::from(Meter(1500.0));
    println!("{:?}", km); // Kilometer(1.5)

    // Guna Into (auto-derived!)
    let km2: Kilometer = Meter(2000.0).into();
    println!("{:?}", km2); // Kilometer(2.0)
}
```

---

# BAHAGIAN 7: serde — Serialize & Deserialize 📦

## serde::Serialize & Deserialize

```toml
# Cargo.toml
[dependencies]
serde      = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
use serde::{Serialize, Deserialize};

// #[derive(Serialize)]   → boleh convert ke JSON/TOML/dll
// #[derive(Deserialize)] → boleh convert dari JSON/TOML/dll

#[derive(Debug, Serialize, Deserialize, Clone, PartialEq)]
struct Pekerja {
    id:       u32,
    nama:     String,
    gaji:     f64,
    aktif:    bool,

    // Optional field
    emel:     Option<String>,
}

#[derive(Debug, Serialize, Deserialize)]
enum StatusPekerja { Tetap, Kontrak, Percubaan }

// Dengan serde attributes
#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]   // field jadi camelCase dalam JSON
struct PekerjaAPI {
    nama_penuh:    String,   // → "namaFull"
    no_kakitangan: String,   // → "noKakitangan"
    tarikh_lahir:  String,   // → "tarikhLahir"

    #[serde(skip_serializing_if = "Option::is_none")]
    no_telefon:    Option<String>, // skip kalau None dalam JSON

    #[serde(default)]
    aktif:         bool,           // default false kalau tiada dalam JSON

    #[serde(rename = "dept")]      // guna nama "dept" dalam JSON
    bahagian:      String,
}

fn main() {
    let p = Pekerja {
        id: 1, nama: "Ali Ahmad".into(),
        gaji: 4500.0, aktif: true,
        emel: Some("ali@email.com".into()),
    };

    // Serialize → JSON
    let json = serde_json::to_string_pretty(&p).unwrap();
    println!("{}", json);

    // Deserialize ← JSON
    let json_input = r#"{
        "id": 2,
        "nama": "Siti Hawa",
        "gaji": 3800.0,
        "aktif": false,
        "emel": null
    }"#;

    let p2: Pekerja = serde_json::from_str(json_input).unwrap();
    println!("{:?}", p2);

    // Serialize ← struct
    let api = PekerjaAPI {
        nama_penuh:    "Ali Ahmad".into(),
        no_kakitangan: "KADA001".into(),
        tarikh_lahir:  "1990-01-15".into(),
        no_telefon:    None,   // akan di-skip dalam JSON
        aktif:         true,
        bahagian:      "ICT".into(),
    };

    println!("{}", serde_json::to_string_pretty(&api).unwrap());
    // {
    //   "namaFull": "Ali Ahmad",
    //   "noKakitangan": "KADA001",
    //   ...
    //   "dept": "ICT"
    //   (tiada no_telefon kerana None dan skip_serializing_if)
    // }
}
```

---

# BAHAGIAN 8: thiserror::Error 🚨

## thiserror — Custom Error Types

```toml
[dependencies]
thiserror = "1"
```

```rust
use thiserror::Error;

// #[derive(Error)] → implement std::error::Error + Display

#[derive(Debug, Error)]
enum AppError {
    // #[error("...")] → Display message
    #[error("Fail tidak dijumpai: {laluan}")]
    FailTidakJumpai { laluan: String },

    // #[from] → implement From<io::Error> for AppError
    // Membolehkan ? operator auto-convert!
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),

    // Boleh ada multiple from
    #[error("Parse error: {0}")]
    Parse(#[from] std::num::ParseIntError),

    // Custom dengan data
    #[error("Nilai {nilai} luar julat [{min}, {max}]")]
    LuarJulat { nilai: i32, min: i32, max: i32 },

    // Dengan source error (chained errors)
    #[error("Konfigurasi tidak sah")]
    KonfigTidakSah {
        #[source] // mark sebagai punca (accessible via .source())
        punca: Box<dyn std::error::Error + Send + Sync>,
    },

    // Transparent — delegate Display dan Source ke inner error
    #[error(transparent)]
    Lain(#[from] Box<dyn std::error::Error + Send + Sync>),
}

fn baca_port(path: &str) -> Result<u16, AppError> {
    let kandungan = std::fs::read_to_string(path)?;  // io::Error → AppError::Io
    let port: u16 = kandungan.trim().parse()?;         // ParseIntError → AppError::Parse

    if port == 0 || port > 65535 {
        return Err(AppError::LuarJulat {
            nilai: port as i32, min: 1, max: 65535
        });
    }

    Ok(port)
}

fn main() {
    match baca_port("port.txt") {
        Ok(p)                          => println!("Port: {}", p),
        Err(AppError::Io(e))           => println!("IO: {}", e),
        Err(AppError::Parse(e))        => println!("Parse: {}", e),
        Err(AppError::LuarJulat { nilai, min, max }) =>
            println!("Luar julat: {} ({}~{})", nilai, min, max),
        Err(AppError::FailTidakJumpai { laluan }) =>
            println!("Fail tiada: {}", laluan),
        Err(e) => println!("Ralat lain: {}", e),
    }
}
```

---

# BAHAGIAN 9: Derive Lain yang Popular 🌟

## clap — CLI Argument Parsing

```toml
[dependencies]
clap = { version = "4", features = ["derive"] }
```

```rust
use clap::Parser;

// #[derive(Parser)] → auto-parse command line arguments!
#[derive(Parser, Debug)]
#[command(name = "kada-cli")]
#[command(about = "KADA Mobile CLI Tool")]
struct CliArgs {
    #[arg(short, long)]
    host: String,

    #[arg(short, long, default_value_t = 8080)]
    port: u16,

    #[arg(short, long)]
    debug: bool,

    #[arg(short, long, value_name = "FILE")]
    konfigurasi: Option<std::path::PathBuf>,
}

fn main() {
    let args = CliArgs::parse();
    println!("Sambung ke {}:{}", args.host, args.port);
    if args.debug { println!("Debug mode aktif"); }
}

// Guna:
// cargo run -- --host localhost --port 3000 --debug
// cargo run -- -h localhost -p 3000 -d
// cargo run -- --help  ← auto-generated help!
```

---

## strum — Enum Utilities

```toml
[dependencies]
strum        = "0.26"
strum_macros = "0.26"
```

```rust
use strum::{EnumIter, EnumString, Display, IntoEnumIterator};
use strum_macros::EnumIs;

#[derive(Debug, EnumIter, EnumString, Display, PartialEq, EnumIs)]
enum Bahagian {
    #[strum(serialize = "ict", serialize = "ICT")]
    ICT,
    #[strum(serialize = "hr", serialize = "HR")]
    HR,
    #[strum(serialize = "kewangan")]
    Kewangan,
}

fn main() {
    // EnumIter — iterate semua variants!
    for b in Bahagian::iter() {
        println!("{}", b);
    }
    // ICT
    // HR
    // Kewangan

    // EnumString — parse dari string!
    use std::str::FromStr;
    let b = Bahagian::from_str("ict").unwrap();
    println!("{:?}", b); // ICT

    // Display — to_string() auto!
    let s = Bahagian::ICT.to_string();
    println!("{}", s); // ICT

    // EnumIs — auto-generate is_xxx() methods!
    let bah = Bahagian::ICT;
    println!("{}", bah.is_ict()); // true
    println!("{}", bah.is_hr());  // false
}
```

---

## builder_derive / typed-builder

```toml
[dependencies]
typed-builder = "0.18"
```

```rust
use typed_builder::TypedBuilder;

// #[derive(TypedBuilder)] → type-safe builder pattern!
#[derive(Debug, TypedBuilder)]
struct Pesanan {
    id:         u32,
    item:       String,

    #[builder(default = 1)]
    kuantiti:   u32,           // default value

    #[builder(default, setter(strip_option))]
    nota:       Option<String>, // optional field

    #[builder(setter(transform = |s: &str| s.to_uppercase()))]
    kod:        String,         // transform input
}

fn main() {
    let pesanan = Pesanan::builder()
        .id(1001)
        .item("Beras 5kg".into())
        .kuantiti(3)
        .nota("Urgent".into())   // setter(strip_option) — tidak perlu Some(...)
        .kod("abc-123")          // auto uppercase → "ABC-123"
        .build();

    println!("{:#?}", pesanan);
}
```

---

## derivative — Conditional Derives

```toml
[dependencies]
derivative = "2"
```

```rust
use derivative::Derivative;

// Guna bila field tertentu tidak implement trait standard
// atau nak customize behavior derive

#[derive(Derivative)]
#[derivative(Debug, Clone, PartialEq)]
struct Sambungan {
    host:   String,
    port:   u16,

    // Skip field ini dalam Debug output (sensitive!)
    #[derivative(Debug = "ignore")]
    kata_laluan: String,

    // Field ini tidak affect equality comparison
    #[derivative(PartialEq = "ignore")]
    masa_cipta: std::time::Instant,
}

fn main() {
    let s = Sambungan {
        host:        "localhost".into(),
        port:        5432,
        kata_laluan: "rahsia123".into(),
        masa_cipta:  std::time::Instant::now(),
    };

    // Debug output tidak tunjuk kata_laluan!
    println!("{:?}", s);
    // Sambungan { host: "localhost", port: 5432 }
}
```

---

# BAHAGIAN 10: Custom Derive 🔨

## Buat Derive Macro Sendiri

```toml
# Cargo.toml untuk proc macro crate
[lib]
proc-macro = true

[dependencies]
proc-macro2 = "1"
quote       = "1"
syn         = { version = "2", features = ["full"] }
```

```rust
// src/lib.rs — dalam PROC MACRO CRATE (berasingan!)

use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

// Buat derive macro bernama "Ringkasan"
#[proc_macro_derive(Ringkasan)]
pub fn derive_ringkasan(input: TokenStream) -> TokenStream {
    // Parse input sebagai Rust struct/enum
    let input = parse_macro_input!(input as DeriveInput);
    let nama = &input.ident; // nama struct

    // Generate kod
    let expanded = quote! {
        impl #nama {
            pub fn ringkasan(&self) -> String {
                format!("[{}] {:?}", stringify!(#nama), self)
            }
        }
    };

    TokenStream::from(expanded)
}
```

```rust
// Dalam crate lain yang guna macro kita:
use my_derive_crate::Ringkasan;

#[derive(Debug, Ringkasan)] // guna derive macro kita!
struct Laporan {
    tajuk: String,
    halaman: u32,
}

fn main() {
    let lap = Laporan { tajuk: "Laporan 2024".into(), halaman: 50 };
    println!("{}", lap.ringkasan());
    // [Laporan] Laporan { tajuk: "Laporan 2024", halaman: 50 }
}
```

---

## Bila Tulis Manual vs Derive?

```
Guna DERIVE bila:
  ✔ Behaviour standard adalah mencukupi
  ✔ Semua field implement trait berkenaan
  ✔ Tiada logik khas diperlukan

Tulis MANUAL bila:
  ✔ Ada field sensitif yang nak skip (kata laluan dalam Debug)
  ✔ Nak format Display yang khas (bukan default Debug output)
  ✔ Nak equality berdasarkan subset field sahaja
  ✔ Nak ordering yang berbeza dari urutan field
  ✔ Ada field yang tidak implement trait berkenaan
  ✔ Nak hash berdasarkan subset field
  ✔ Ada logik validasi dalam Default
```

---

# 📋 Rujukan Pantas — Derive Cheat Sheet

## Standard Library Derives

```rust
// Basic
#[derive(Debug)]         // {:?} dan {:#?} formatting, dbg!()
#[derive(Clone)]         // .clone() — explicit deep copy
#[derive(Copy)]          // implicit copy (Clone juga diperlukan!)

// Equality & Comparison
#[derive(PartialEq)]     // == dan !=
#[derive(Eq)]            // total equality (perlu PartialEq)
#[derive(PartialOrd)]    // <, >, <=, >= (perlu PartialEq)
#[derive(Ord)]           // .sort(), .max(), .min() (perlu Eq + PartialOrd)

// Collections
#[derive(Hash)]          // HashMap/HashSet key (perlu Eq + PartialEq)

// Default values
#[derive(Default)]       // Type::default() atau Default::default()
```

## Popular Crate Derives

```rust
// serde
#[derive(Serialize)]     // T → JSON/TOML/dll
#[derive(Deserialize)]   // JSON/TOML/dll → T

// thiserror
#[derive(Error)]         // Custom error types dengan Display + Error

// clap
#[derive(Parser)]        // CLI argument parsing
#[derive(Args)]          // CLI subcommand args
#[derive(Subcommand)]    // CLI subcommands

// strum
#[derive(EnumIter)]      // Iterate semua enum variants
#[derive(EnumString)]    // Parse enum dari string
#[derive(Display)]       // Display untuk enum
#[derive(EnumIs)]        // is_variant() methods

// others
#[derive(TypedBuilder)]  // Type-safe builder pattern
#[derive(Derivative)]    // Conditional/customized derives
```

## Kombinasi yang Biasa Digunakan

```rust
// Struct data biasa
#[derive(Debug, Clone, PartialEq)]

// Struct yang juga HashMap key
#[derive(Debug, Clone, PartialEq, Eq, Hash)]

// Enum state/status
#[derive(Debug, Clone, PartialEq, Eq)]

// Config/settings struct
#[derive(Debug, Clone, Default)]

// API response/request struct
#[derive(Debug, Serialize, Deserialize)]

// Full data struct
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize, Default)]

// Small Copy types (koordinat, warna, dll)
#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]

// Sortable data
#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]

// Error types
#[derive(Debug, thiserror::Error)]
```

## Peraturan Penting

```
1. Semua field mesti implement trait yang anda derive.
   (atau compile error!)

2. Copy mesti disertai Clone.

3. Eq mesti disertai PartialEq.

4. Ord mesti disertai Eq + PartialOrd + PartialEq.

5. Hash mesti disertai Eq + PartialEq.
   (Dan consistency rule: a == b → hash(a) == hash(b))

6. Kalau implement PartialEq manual,
   JANGAN derive Hash — implement manual juga!
   (Consistency mesti dijaga)

7. derive(Ord) bandingkan field mengikut URUTAN DEKLARASI.
   Kalau nak order berbeza, implement manual.
```

---

## 🏆 Cabaran Akhir

Cuba design struct/enum yang sesuai dengan semua derives yang betul:

1. **`RekodLog`** — untuk simpan log entries, boleh serialize ke JSON, boleh sort by timestamp
2. **`PengecamUnik`** — untuk guna sebagai HashMap key, ada UUID dan nama
3. **`StatusPermohonan`** — enum state machine, boleh display kepada pengguna
4. **`KonfigurasiApp`** — settings dengan nilai default yang sesuai, boleh serialize/deserialize
5. **`ApiError`** — custom error untuk web API dengan mesej yang informatif

---

*Derive adalah kuasa. Guna dengan bijak, derive yang sesuai,*
*dan tulis manual bila derive tidak cukup.* 🦀
