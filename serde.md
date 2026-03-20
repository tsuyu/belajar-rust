# 📦 Serde dalam Rust — Beginner to Advanced

> Panduan lengkap serde: Serialize, Deserialize, JSON, TOML, custom dan lebih.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu Serde?

```
Serde = SERialization DEserialization

Serialization:   Rust struct/enum → format lain (JSON, TOML, YAML, ...)
Deserialization: Format lain → Rust struct/enum

Tanpa Serde:                 Dengan Serde:
  fn ke_json(p: &Pengguna)    #[derive(Serialize, Deserialize)]
      -> String {             struct Pengguna { ... }
    format!(r#"              
      {{"nama":"{}",          serde_json::to_string(&p)?
       "umur":{}}}"#,         serde_json::from_str(&json)?
      p.nama, p.umur)
  }
  // Menulis sendiri = lambat, mudah salah!
```

---

## Kenapa Serde Sangat Popular?

```
✔ Zero-cost abstraction — tiada overhead runtime
✔ Type-safe — compiler check semasa compile
✔ Flexible — sokong 30+ format!
✔ Ergonomic — #[derive] mudah
✔ Widely used — hampir semua crate Rust guna serde

Format yang disokong:
  serde_json   → JSON  (API, config)
  toml         → TOML  (Cargo.toml, config)
  serde_yaml   → YAML  (Kubernetes, config)
  bincode      → Binary (fast, compact)
  csv          → CSV   (spreadsheet, data)
  postcard     → no_std binary
  ron          → Rusty Object Notation
  ...dan banyak lagi!
```

---

## Peta Pembelajaran

```
Bab 1  → Setup & Derive Asas
Bab 2  → Serialize — Rust → JSON
Bab 3  → Deserialize — JSON → Rust
Bab 4  → Attribute — Customize Tingkah Laku
Bab 5  → Option, Vec & Nested Types
Bab 6  → Enum Serialization
Bab 7  → Custom Serialization
Bab 8  → TOML & Format Lain
Bab 9  → Pattern Lanjutan
Bab 10 → Mini Project: KADA Config & API
```

---

# BAB 1: Setup & Derive Asas ⚙️

## Cargo.toml

```toml
[dependencies]
serde      = { version = "1", features = ["derive"] }
serde_json = "1"

# Format lain (optional):
# toml       = { version = "0.8", features = ["parse", "display"] }
# serde_yaml = "0.9"
# csv        = "1"
# bincode    = "1"
```

---

## Derive Pertama

```rust
use serde::{Serialize, Deserialize};

// Derive KEDUA-DUA sekaligus — paling biasa
#[derive(Debug, Serialize, Deserialize)]
struct Pekerja {
    nama:     String,
    id:       u32,
    gaji:     f64,
    aktif:    bool,
}

fn main() {
    let p = Pekerja {
        nama:  "Ali Ahmad".into(),
        id:    1001,
        gaji:  4500.0,
        aktif: true,
    };

    // Serialize → JSON string
    let json = serde_json::to_string(&p).unwrap();
    println!("JSON: {}", json);
    // {"nama":"Ali Ahmad","id":1001,"gaji":4500.0,"aktif":true}

    // Pretty-print JSON
    let json_cantik = serde_json::to_string_pretty(&p).unwrap();
    println!("{}", json_cantik);
    // {
    //   "nama": "Ali Ahmad",
    //   "id": 1001,
    //   "gaji": 4500.0,
    //   "aktif": true
    // }

    // Deserialize ← JSON string
    let json2 = r#"{"nama":"Siti Hawa","id":1002,"gaji":3800.0,"aktif":false}"#;
    let siti: Pekerja = serde_json::from_str(json2).unwrap();
    println!("{:?}", siti);
}
```

---

## Value API — JSON Tanpa Struct

```rust
use serde_json::{Value, json};

fn main() {
    // json! macro — buat JSON value terus
    let data = json!({
        "nama":  "Ali",
        "umur":  25,
        "hobi":  ["membaca", "mendaki", "memasak"],
        "alamat": {
            "bandar": "Kuala Lumpur",
            "negeri": "Selangor"
        }
    });

    println!("{}", data);
    println!("{}", data["nama"]);           // "Ali"
    println!("{}", data["hobi"][0]);        // "membaca"
    println!("{}", data["alamat"]["bandar"]); // "Kuala Lumpur"

    // Navigasi dengan .get()
    if let Some(nama) = data.get("nama") {
        println!("Nama: {}", nama);
    }

    // Parse JSON string ke Value
    let json_str = r#"{"x": 10, "y": 20}"#;
    let v: Value = serde_json::from_str(json_str).unwrap();
    println!("x = {}, y = {}", v["x"], v["y"]);
}
```

---

## 🧠 Brain Teaser #1

Apa yang `#[derive(Serialize)]` buat di belakang tabir?

```rust
#[derive(Serialize)]
struct Titik { x: f64, y: f64 }
```

<details>
<summary>👀 Jawapan</summary>

`#[derive(Serialize)]` adalah **proc macro** yang generate kod Rust secara automatik. Kod yang dihasilkan lebih kurang:

```rust
impl serde::Serialize for Titik {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        use serde::ser::SerializeStruct;
        let mut state = serializer.serialize_struct("Titik", 2)?;
        state.serialize_field("x", &self.x)?;
        state.serialize_field("y", &self.y)?;
        state.end()
    }
}
```

Serde memisahkan **data model** (Serialize/Deserialize trait) dari **format** (JSON, TOML, dll). Struct kita hanya perlu implement trait — format-specific code ada dalam crate format (serde_json, toml, dll).

Ini sebab struct yang sama boleh di-serialize ke **JSON, TOML, YAML** tanpa tukar apa-apa dalam struct!
</details>

---

# BAB 2: Serialize — Rust → JSON 📤

## to_string, to_vec, to_writer

```rust
use serde::Serialize;
use serde_json;
use std::fs::File;

#[derive(Serialize, Debug)]
struct Produk {
    nama:     String,
    harga:    f64,
    kategori: String,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let produk = vec![
        Produk { nama: "Beras 5kg".into(), harga: 12.90, kategori: "Makanan".into() },
        Produk { nama: "Minyak 1L".into(), harga: 6.50,  kategori: "Makanan".into() },
        Produk { nama: "Sabun".into(),     harga: 2.50,  kategori: "Kebersihan".into() },
    ];

    // to_string — hasilkan String
    let json_str: String = serde_json::to_string(&produk)?;
    println!("String: {}", json_str);

    // to_string_pretty — dengan indent
    let json_cantik = serde_json::to_string_pretty(&produk)?;
    println!("Pretty:\n{}", json_cantik);

    // to_vec — hasilkan Vec<u8> (bytes)
    let json_bytes: Vec<u8> = serde_json::to_vec(&produk)?;
    println!("Bytes: {} bytes", json_bytes.len());

    // to_writer — tulis terus ke Writer (fail, socket, dll)
    let fail = File::create("produk.json")?;
    serde_json::to_writer_pretty(fail, &produk)?;
    println!("Disimpan ke produk.json");

    // Cleanup
    std::fs::remove_file("produk.json")?;
    Ok(())
}
```

---

## Serialize ke Map/Object

```rust
use serde::Serialize;
use std::collections::HashMap;

#[derive(Serialize)]
struct Statistik {
    jumlah:  u32,
    min:     f64,
    max:     f64,
    purata:  f64,
}

fn main() {
    // Struct → JSON object
    let stat = Statistik { jumlah: 100, min: 1.0, max: 99.0, purata: 50.0 };
    println!("{}", serde_json::to_string_pretty(&stat).unwrap());
    // {
    //   "jumlah": 100,
    //   "min": 1.0,
    //   "max": 99.0,
    //   "purata": 50.0
    // }

    // HashMap → JSON object
    let mut m: HashMap<&str, i32> = HashMap::new();
    m.insert("epal",   5);
    m.insert("mangga", 3);
    m.insert("pisang", 8);
    println!("{}", serde_json::to_string(&m).unwrap());
    // {"epal":5,"mangga":3,"pisang":8}

    // Vec → JSON array
    let v = vec!["satu", "dua", "tiga"];
    println!("{}", serde_json::to_string(&v).unwrap());
    // ["satu","dua","tiga"]

    // Tuple → JSON array
    let t = (1, "hello", true, 3.14);
    println!("{}", serde_json::to_string(&t).unwrap());
    // [1,"hello",true,3.14]
}
```

---

# BAB 3: Deserialize — JSON → Rust 📥

## from_str, from_slice, from_reader

```rust
use serde::Deserialize;
use std::fs;

#[derive(Debug, Deserialize)]
struct Konfigurasi {
    host:    String,
    port:    u16,
    debug:   bool,
    pekerja: u32,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // from_str — dari String atau &str
    let json = r#"{
        "host":    "localhost",
        "port":    8080,
        "debug":   true,
        "pekerja": 4
    }"#;

    let cfg: Konfigurasi = serde_json::from_str(json)?;
    println!("{:?}", cfg);
    println!("Server: {}:{}", cfg.host, cfg.port);

    // from_slice — dari &[u8] (bytes)
    let bytes = json.as_bytes();
    let cfg2: Konfigurasi = serde_json::from_slice(bytes)?;
    println!("{:?}", cfg2);

    // from_reader — dari fail / IO
    fs::write("config.json", json)?;
    let fail = fs::File::open("config.json")?;
    let cfg3: Konfigurasi = serde_json::from_reader(fail)?;
    println!("{:?}", cfg3);

    fs::remove_file("config.json")?;
    Ok(())
}
```

---

## Error Handling dalam Deserialization

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Pengguna {
    nama: String,
    umur: u32,
}

fn main() {
    // JSON betul
    let json_ok = r#"{"nama":"Ali","umur":25}"#;
    match serde_json::from_str::<Pengguna>(json_ok) {
        Ok(p)  => println!("✔ {}: {}", p.nama, p.umur),
        Err(e) => println!("✘ {}", e),
    }

    // JSON ada field salah
    let json_salah_type = r#"{"nama":"Siti","umur":"dua puluh"}"#;
    match serde_json::from_str::<Pengguna>(json_salah_type) {
        Ok(p)  => println!("✔ {}", p.nama),
        Err(e) => println!("✘ Type salah: {}", e),
    }

    // JSON missing field
    let json_hilang = r#"{"nama":"Amin"}"#; // tiada umur!
    match serde_json::from_str::<Pengguna>(json_hilang) {
        Ok(p)  => println!("✔ {}", p.nama),
        Err(e) => println!("✘ Field hilang: {}", e),
    }

    // JSON tidak sah
    let json_rosak = r#"{nama: Ali}"#;
    match serde_json::from_str::<Pengguna>(json_rosak) {
        Ok(p)  => println!("✔ {}", p.nama),
        Err(e) => println!("✘ JSON tidak sah: {}", e),
    }
}
```

---

## 🧠 Brain Teaser #2

JSON di bawah boleh atau tidak boleh deserialize ke struct `Pengguna`? Kenapa?

```rust
#[derive(Debug, Deserialize)]
struct Pengguna {
    nama: String,
    umur: u32,
    emel: String,
}

let json1 = r#"{"nama":"Ali","umur":25,"emel":"ali@test.com","bonus_field":"abc"}"#;
let json2 = r#"{"nama":"Ali","umur":25}"#;
```

<details>
<summary>👀 Jawapan</summary>

- **json1** — ✔ **Boleh!** Field tambahan (`"bonus_field"`) **diabaikan** secara default. Serde hanya ambil field yang ada dalam struct.

- **json2** — ❌ **Tidak boleh!** Field `emel` adalah required (tiada `Option` atau default). Serde akan return `Err: missing field 'emel'`.

**Penyelesaian untuk json2:**
```rust
#[derive(Debug, Deserialize)]
struct Pengguna {
    nama: String,
    umur: u32,
    #[serde(default)]   // ← guna "" kalau field tiada
    emel: String,
}
// ATAU
    emel: Option<String>,  // ← None kalau field tiada
```

**Untuk elak field tidak dikenali (json1):**
```rust
#[serde(deny_unknown_fields)]  // ← ERROR kalau ada field extra
struct Pengguna { ... }
```
</details>

---

# BAB 4: Attributes — Customize Tingkah Laku 🎛️

## #[serde(rename)]

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct ApiResponse {
    // Rename field dalam JSON
    #[serde(rename = "user_name")]
    nama_pengguna: String,

    #[serde(rename = "userId")]
    id_pengguna: u32,

    // Rename berbeza untuk serialize dan deserialize
    #[serde(rename(serialize = "birthYear", deserialize = "birth_year"))]
    tahun_lahir: u32,
}

#[derive(Debug, Serialize, Deserialize)]
// Rename SEMUA field mengikut konvensyen
#[serde(rename_all = "camelCase")]
struct RekodPengguna {
    nama_penuh:   String,   // → "namaFull"
    no_telefon:   String,   // → "noTelefon"
    tarikh_lahir: String,   // → "tarikhLahir"
}

fn main() {
    let r = RekodPengguna {
        nama_penuh:   "Ali Ahmad".into(),
        no_telefon:   "012-345-6789".into(),
        tarikh_lahir: "1990-01-15".into(),
    };

    println!("{}", serde_json::to_string_pretty(&r).unwrap());
    // {
    //   "namaFull": "Ali Ahmad",
    //   "noTelefon": "012-345-6789",
    //   "tarikhLahir": "1990-01-15"
    // }
}
```

---

## #[serde(skip)] & #[serde(default)]

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct Pengguna {
    nama:         String,
    emel:         String,

    #[serde(skip)]  // skip dalam KEDUA-DUA serialize DAN deserialize
    kata_laluan:  String,

    #[serde(skip_serializing)]  // skip semasa serialize sahaja
    token_dalam:  String,

    #[serde(skip_deserializing)] // skip semasa deserialize sahaja
    id_sistem:    u32,

    #[serde(default)]  // guna Default::default() kalau field tiada dalam JSON
    aktif:        bool,

    #[serde(default = "nilai_lalai_umur")]  // guna fungsi untuk default
    umur:         u32,
}

fn nilai_lalai_umur() -> u32 { 18 }

fn main() {
    let p = Pengguna {
        nama:        "Ali".into(),
        emel:        "ali@test.com".into(),
        kata_laluan: "rahsia123".into(),   // skip!
        token_dalam: "tok_abc".into(),
        id_sistem:   9999,
        aktif:       true,
        umur:        25,
    };

    let json = serde_json::to_string_pretty(&p).unwrap();
    println!("Serialize:\n{}", json);
    // kata_laluan dan token_dalam tidak akan muncul!

    // Deserialize — id_sistem tidak diambil dari JSON
    let json2 = r#"{"nama":"Siti","emel":"siti@test.com"}"#;
    let siti: Pengguna = serde_json::from_str(json2).unwrap();
    println!("\nDeserialize: {:?}", siti);
    // aktif = false (default), umur = 18 (default function)
}
```

---

## #[serde(flatten)] & #[serde(skip_serializing_if)]

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct Meta {
    dibuat_pada:     String,
    dikemaskini_pada: String,
}

#[derive(Debug, Serialize, Deserialize)]
struct Artikel {
    tajuk:   String,
    kandungan: String,

    #[serde(flatten)]  // field Meta "naik" ke level atas
    meta: Meta,
}

#[derive(Debug, Serialize, Deserialize)]
struct Response<T> {
    data: T,

    #[serde(skip_serializing_if = "Option::is_none")]  // skip kalau None
    ralat: Option<String>,

    #[serde(skip_serializing_if = "Vec::is_empty")]    // skip kalau kosong
    amaran: Vec<String>,
}

fn main() {
    let artikel = Artikel {
        tajuk:     "Hello Rust".into(),
        kandungan: "Rust adalah hebat".into(),
        meta: Meta {
            dibuat_pada:     "2024-01-15".into(),
            dikemaskini_pada: "2024-01-20".into(),
        },
    };

    println!("{}", serde_json::to_string_pretty(&artikel).unwrap());
    // meta fields akan "rata" dengan fields lain!
    // {
    //   "tajuk": "Hello Rust",
    //   "kandungan": "Rust adalah hebat",
    //   "dibuat_pada": "2024-01-15",       ← flatten!
    //   "dikemaskini_pada": "2024-01-20"   ← flatten!
    // }

    let resp: Response<i32> = Response {
        data: 42,
        ralat: None,
        amaran: vec![],
    };
    println!("{}", serde_json::to_string(&resp).unwrap());
    // {"data":42}  ← ralat dan amaran di-skip!
}
```

---

## Semua #[serde(...)] Attributes

```rust
// Pada struct/enum:
#[serde(rename = "NewName")]           // rename
#[serde(rename_all = "camelCase")]     // camelCase/snake_case/PascalCase/UPPER_SNAKE_CASE
#[serde(deny_unknown_fields)]          // error kalau ada field extra
#[serde(default)]                      // guna Default kalau field hilang
#[serde(transparent)]                  // treat newtype struct seperti inner type
#[serde(tag = "type")]                 // enum tagging
#[serde(content = "data")]             // enum content field
#[serde(untagged)]                     // enum tanpa tag

// Pada field:
#[serde(rename = "other_name")]
#[serde(alias = "alt_name")]           // terima nama lain juga
#[serde(default)]                      // nilai default
#[serde(default = "fungsi")]           // custom default function
#[serde(skip)]                         // skip dalam kedua-dua
#[serde(skip_serializing)]             // skip serialize sahaja
#[serde(skip_deserializing)]           // skip deserialize sahaja
#[serde(skip_serializing_if = "...")]  // skip kalau condition
#[serde(flatten)]                      // flatten nested struct
#[serde(with = "modul")]               // custom ser/de functions
#[serde(serialize_with = "fungsi")]    // custom serialize
#[serde(deserialize_with = "fungsi")] // custom deserialize
```

---

# BAB 5: Option, Vec & Nested Types 📋

## Option — Nullable Fields

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct ProfailPengguna {
    nama:       String,
    bio:        Option<String>,      // null atau string
    no_telefon: Option<String>,
    umur:       Option<u32>,
}

fn main() {
    // Some → nilai JSON
    // None → "null" dalam JSON
    let p1 = ProfailPengguna {
        nama:       "Ali".into(),
        bio:        Some("Seorang developer Rust".into()),
        no_telefon: None,
        umur:       Some(25),
    };

    println!("{}", serde_json::to_string_pretty(&p1).unwrap());
    // {
    //   "nama": "Ali",
    //   "bio": "Seorang developer Rust",
    //   "no_telefon": null,    ← None jadi null
    //   "umur": 25
    // }

    // Deserialize null → None, nilai → Some(x)
    let json = r#"{"nama":"Siti","bio":null,"no_telefon":"012-3456789","umur":null}"#;
    let siti: ProfailPengguna = serde_json::from_str(json).unwrap();
    println!("{:?}", siti);
    // bio: None, no_telefon: Some("012-3456789"), umur: None

    // Skip null (tidak tunjuk "null" dalam JSON)
    #[derive(Serialize, Deserialize)]
    struct TanpaNullField {
        nama: String,
        #[serde(skip_serializing_if = "Option::is_none")]
        bio: Option<String>,
    }

    let t = TanpaNullField { nama: "Amin".into(), bio: None };
    println!("{}", serde_json::to_string(&t).unwrap());
    // {"nama":"Amin"}  ← tiada "bio":null!
}
```

---

## Vec & Nested Structs

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct Alamat {
    jalan:  String,
    bandar: String,
    poskod: String,
}

#[derive(Debug, Serialize, Deserialize)]
struct Syarikat {
    nama:    String,
    alamat:  Alamat,            // nested struct
    kakitangan: Vec<String>,    // array of strings
    tahun_tubuh: u32,
}

#[derive(Debug, Serialize, Deserialize)]
struct TindakBalas {
    berjaya: bool,
    data:    Vec<Syarikat>,     // array of nested structs
    jumlah:  u32,
}

fn main() {
    let json = r#"{
        "berjaya": true,
        "jumlah":  1,
        "data": [{
            "nama": "KADA",
            "tahun_tubuh": 1967,
            "alamat": {
                "jalan":  "Jalan Hamzah",
                "bandar": "Kota Bharu",
                "poskod": "15000"
            },
            "kakitangan": ["Ali", "Siti", "Amin"]
        }]
    }"#;

    let tindak_balas: TindakBalas = serde_json::from_str(json).unwrap();
    println!("Syarikat: {}", tindak_balas.data[0].nama);
    println!("Bandar:   {}", tindak_balas.data[0].alamat.bandar);
    println!("Staff:    {:?}", tindak_balas.data[0].kakitangan);
}
```

---

# BAB 6: Enum Serialization 🎭

## Enum dengan Data

```rust
use serde::{Serialize, Deserialize};

// DEFAULT — externally tagged
#[derive(Debug, Serialize, Deserialize)]
enum Arahan {
    Berhenti,                          // → "Berhenti"
    Gerak { x: i32, y: i32 },         // → {"Gerak":{"x":10,"y":20}}
    Tulis(String),                     // → {"Tulis":"hello"}
    TukarWarna(u8, u8, u8),           // → {"TukarWarna":[255,0,0]}
}

// INTERNALLY tagged — tag dalam object
#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "jenis")]
enum Haiwan {
    Kucing { nama: String, warna: String },
    Anjing { nama: String, baka: String },
    // hanya untuk struct variants! bukan tuple/unit
}

// ADJACENTLY tagged — tag dan content berasingan
#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "t", content = "c")]
enum Nilai {
    Nombor(f64),
    Teks(String),
    Senarai(Vec<i32>),
}

// UNTAGGED — cuba match pattern
#[derive(Debug, Serialize, Deserialize)]
#[serde(untagged)]
enum FlexInput {
    Nombor(i64),
    Teks(String),
    Senarai(Vec<i64>),
}

fn main() {
    // Externally tagged
    let a = Arahan::Gerak { x: 10, y: 20 };
    println!("{}", serde_json::to_string(&a).unwrap());
    // {"Gerak":{"x":10,"y":20}}

    // Internally tagged
    let k = Haiwan::Kucing { nama: "Comel".into(), warna: "hitam".into() };
    println!("{}", serde_json::to_string(&k).unwrap());
    // {"jenis":"Kucing","nama":"Comel","warna":"hitam"}

    // Adjacently tagged
    let n = Nilai::Nombor(42.0);
    println!("{}", serde_json::to_string(&n).unwrap());
    // {"t":"Nombor","c":42.0}

    // Untagged
    let flex: FlexInput = serde_json::from_str("42").unwrap();
    println!("{:?}", flex);  // Nombor(42)

    let flex2: FlexInput = serde_json::from_str(r#""hello""#).unwrap();
    println!("{:?}", flex2); // Teks("hello")
}
```

---

## 🧠 Brain Teaser #3

Apakah JSON yang dihasilkan untuk setiap pilihan tagging?

```rust
#[derive(Serialize)]
enum Bentuk {
    Bulatan { radius: f64 },
}

// Option A: default (external)
// Option B: #[serde(tag = "jenis")]
// Option C: #[serde(tag = "t", content = "d")]
// Option D: #[serde(untagged)]

let b = Bentuk::Bulatan { radius: 5.0 };
```

<details>
<summary>👀 Jawapan</summary>

```
A (external, default):
  {"Bulatan":{"radius":5.0}}
  ↑ variant jadi key

B (internal tag = "jenis"):
  {"jenis":"Bulatan","radius":5.0}
  ↑ tag masuk dalam object

C (adjacent tag = "t", content = "d"):
  {"t":"Bulatan","d":{"radius":5.0}}
  ↑ tag dan data berasingan

D (untagged):
  {"radius":5.0}
  ↑ tiada tag! hanya data — deserialize bergantung pada structure matching
```

**Panduan pilih:**
- Default (external): Jelas, sesuai untuk API baru
- Internal tag: Lebih natural, sesuai bila semua variant ada struct data
- Adjacent: Untuk tuple variants atau mixed types
- Untagged: Bila input boleh pelbagai format, tapi deserialize lebih rumit
</details>

---

# BAB 7: Custom Serialization 🔧

## #[serde(with = "modul")]

```rust
use serde::{Serialize, Deserialize, Serializer, Deserializer};
use serde::de::Error;

// Serialize chrono DateTime sebagai timestamp
mod timestamp_saat {
    use serde::{Serializer, Deserializer, Deserialize};

    pub fn serialize<S: Serializer>(
        ts: &u64,
        serializer: S,
    ) -> Result<S::Ok, S::Error> {
        serializer.serialize_u64(*ts)
    }

    pub fn deserialize<'de, D: Deserializer<'de>>(
        deserializer: D,
    ) -> Result<u64, D::Error> {
        u64::deserialize(deserializer)
    }
}

// Custom serialize string sebagai huruf besar
mod huruf_besar {
    use serde::{Serializer, Deserializer, Deserialize};

    pub fn serialize<S: Serializer>(
        s: &str,
        serializer: S,
    ) -> Result<S::Ok, S::Error> {
        serializer.serialize_str(&s.to_uppercase())
    }

    pub fn deserialize<'de, D: Deserializer<'de>>(
        deserializer: D,
    ) -> Result<String, D::Error> {
        let s = String::deserialize(deserializer)?;
        Ok(s.to_lowercase())  // simpan dalam huruf kecil
    }
}

#[derive(Debug, Serialize, Deserialize)]
struct Peristiwa {
    nama:          String,

    #[serde(with = "huruf_besar")]
    kategori:      String,   // disimpan kecil, diexport besar

    #[serde(with = "timestamp_saat")]
    masa_unix:     u64,
}

fn main() {
    let p = Peristiwa {
        nama:      "Login".into(),
        kategori:  "keselamatan".into(),  // simpan kecil
        masa_unix: 1705276800,
    };

    let json = serde_json::to_string_pretty(&p).unwrap();
    println!("{}", json);
    // kategori akan jadi "KESELAMATAN" dalam JSON

    let json2 = r#"{"nama":"Logout","kategori":"AUDIT","masa_unix":1705280400}"#;
    let e: Peristiwa = serde_json::from_str(json2).unwrap();
    println!("{:?}", e);
    // kategori akan jadi "audit" (lowercase) dalam struct
}
```

---

## Custom Serialize & Deserialize Penuh

```rust
use serde::{Serialize, Serializer, Deserialize, Deserializer};
use serde::de::{self, Visitor};
use std::fmt;

// Custom type — Ringgit selalu ada 2 desimal, simpan sebagai integer (sen)
#[derive(Debug, PartialEq)]
struct Ringgit(u64); // dalam sen (100 = RM1.00)

impl Serialize for Ringgit {
    fn serialize<S: Serializer>(&self, s: S) -> Result<S::Ok, S::Error> {
        // Serialize sebagai float: 100 sen → 1.00
        let ringgit = self.0 as f64 / 100.0;
        s.serialize_f64(ringgit)
    }
}

struct RinggitVisitor;

impl<'de> Visitor<'de> for RinggitVisitor {
    type Value = Ringgit;

    fn expecting(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "nilai Ringgit sebagai nombor")
    }

    fn visit_f64<E: de::Error>(self, v: f64) -> Result<Ringgit, E> {
        Ok(Ringgit((v * 100.0).round() as u64))
    }

    fn visit_u64<E: de::Error>(self, v: u64) -> Result<Ringgit, E> {
        Ok(Ringgit(v * 100)) // andaikan input dalam RM, bukan sen
    }
}

impl<'de> Deserialize<'de> for Ringgit {
    fn deserialize<D: Deserializer<'de>>(d: D) -> Result<Self, D::Error> {
        d.deserialize_f64(RinggitVisitor)
    }
}

#[derive(Debug, Serialize, Deserialize)]
struct Invois {
    item:   String,
    harga:  Ringgit,
}

fn main() {
    let inv = Invois { item: "Beras 5kg".into(), harga: Ringgit(1290) };
    let json = serde_json::to_string(&inv).unwrap();
    println!("{}", json); // {"item":"Beras 5kg","harga":12.9}

    let json2 = r#"{"item":"Minyak","harga":6.50}"#;
    let inv2: Invois = serde_json::from_str(json2).unwrap();
    println!("{:?}", inv2); // harga: Ringgit(650)
    println!("Harga: {}sen = RM{:.2}", inv2.harga.0, inv2.harga.0 as f64/100.0);
}
```

---

# BAB 8: TOML & Format Lain 📄

## TOML — Konfigurasi

```toml
# Cargo.toml
[dependencies]
serde = { version = "1", features = ["derive"] }
toml  = "0.8"
```

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct AppConfig {
    app:      AppSection,
    database: DbSection,
    logging:  LogSection,
}

#[derive(Debug, Serialize, Deserialize)]
struct AppSection {
    nama:    String,
    versi:   String,
    port:    u16,
    debug:   bool,
}

#[derive(Debug, Serialize, Deserialize)]
struct DbSection {
    host:     String,
    port:     u16,
    nama_db:  String,
    pool_max: u32,
}

#[derive(Debug, Serialize, Deserialize)]
struct LogSection {
    tahap:     String,
    ke_fail:   bool,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Parse dari TOML string
    let toml_str = r#"
        [app]
        nama    = "KADA API"
        versi   = "2.1.0"
        port    = 8080
        debug   = false

        [database]
        host     = "localhost"
        port     = 5432
        nama_db  = "kada_db"
        pool_max = 20

        [logging]
        tahap   = "INFO"
        ke_fail = true
    "#;

    let config: AppConfig = toml::from_str(toml_str)?;
    println!("{:?}", config);
    println!("App: {}:{}", config.app.nama, config.app.port);
    println!("DB:  {}:{}/{}", config.database.host, config.database.port, config.database.nama_db);

    // Serialize ke TOML
    let toml_output = toml::to_string_pretty(&config)?;
    println!("\nTOML output:\n{}", toml_output);

    // Baca dari fail
    std::fs::write("app_config.toml", toml_str)?;
    let kandungan = std::fs::read_to_string("app_config.toml")?;
    let config2: AppConfig = toml::from_str(&kandungan)?;
    println!("Dari fail: {}", config2.app.versi);
    std::fs::remove_file("app_config.toml")?;

    Ok(())
}
```

---

## CSV dengan csv crate

```toml
[dependencies]
csv   = "1"
serde = { version = "1", features = ["derive"] }
```

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct Rekod {
    nama:     String,
    bahagian: String,
    gaji:     f64,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Tulis CSV
    let rekod = vec![
        Rekod { nama: "Ali".into(),  bahagian: "ICT".into(),      gaji: 4500.0 },
        Rekod { nama: "Siti".into(), bahagian: "Kewangan".into(), gaji: 3800.0 },
        Rekod { nama: "Amin".into(), bahagian: "HR".into(),        gaji: 4200.0 },
    ];

    let mut penulis = csv::Writer::from_path("pekerja.csv")?;
    for r in &rekod {
        penulis.serialize(r)?;
    }
    penulis.flush()?;
    println!("CSV ditulis!");

    // Baca CSV
    let mut pembaca = csv::Reader::from_path("pekerja.csv")?;
    println!("\nISI CSV:");
    for hasil in pembaca.deserialize() {
        let r: Rekod = hasil?;
        println!("  {} ({}) — RM{:.2}", r.nama, r.bahagian, r.gaji);
    }

    std::fs::remove_file("pekerja.csv")?;
    Ok(())
}
```

---

# BAB 9: Pattern Lanjutan 🎯

## Generic Struct dengan Serde

```rust
use serde::{Serialize, Deserialize};

// Generic Response wrapper — standard API pattern
#[derive(Debug, Serialize, Deserialize)]
struct ApiResponse<T> {
    berjaya: bool,
    mesej:   String,
    data:    Option<T>,
    ralat:   Option<String>,
}

impl<T: Serialize> ApiResponse<T> {
    fn ok(data: T) -> Self {
        ApiResponse {
            berjaya: true,
            mesej:   "Berjaya".into(),
            data:    Some(data),
            ralat:   None,
        }
    }

    fn gagal(mesej: &str) -> ApiResponse<()> {
        ApiResponse {
            berjaya: false,
            mesej:   mesej.into(),
            data:    None,
            ralat:   Some(mesej.into()),
        }
    }
}

#[derive(Debug, Serialize, Deserialize)]
struct Pengguna { nama: String, emel: String }

fn main() {
    let resp = ApiResponse::ok(Pengguna {
        nama: "Ali".into(),
        emel: "ali@test.com".into(),
    });
    println!("{}", serde_json::to_string_pretty(&resp).unwrap());
    // {
    //   "berjaya": true,
    //   "mesej": "Berjaya",
    //   "data": {"nama":"Ali","emel":"ali@test.com"},
    //   "ralat": null
    // }

    let resp_gagal: ApiResponse<()> = ApiResponse::gagal("Pengguna tidak dijumpai");
    println!("{}", serde_json::to_string_pretty(&resp_gagal).unwrap());
}
```

---

## Partial Deserialization

```rust
use serde::{Deserialize};
use serde_json::Value;

// Ambil subset field sahaja dari JSON besar
#[derive(Debug, Deserialize)]
struct RingkasanSahaja {
    id:   u32,
    nama: String,
    // field lain diabaikan!
}

fn main() {
    let json_besar = r#"{
        "id":          1,
        "nama":        "Ali Ahmad",
        "emel":        "ali@test.com",
        "no_telefon":  "012-345-6789",
        "alamat":      {"jalan": "...", "bandar": "KL"},
        "metadata":    {"dibuat": "2024-01-15", "dikemaskini": "2024-01-20"},
        "permissions": ["read", "write", "admin"]
    }"#;

    // Ambil subset sahaja — field lain diabaikan
    let ringkasan: RingkasanSahaja = serde_json::from_str(json_besar).unwrap();
    println!("{:?}", ringkasan);

    // Atau guna Value untuk akses dinamik
    let v: Value = serde_json::from_str(json_besar).unwrap();
    println!("Bandar: {}", v["alamat"]["bandar"]);
    println!("Perm[0]: {}", v["permissions"][0]);
}
```

---

## Validate Semasa Deserialize

```rust
use serde::{Deserialize, Deserializer};
use serde::de::Error;

fn deserialize_emel<'de, D: Deserializer<'de>>(d: D) -> Result<String, D::Error> {
    let s = String::deserialize(d)?;
    if s.contains('@') && s.contains('.') {
        Ok(s)
    } else {
        Err(D::Error::custom(format!("Emel tidak sah: {}", s)))
    }
}

fn deserialize_umur<'de, D: Deserializer<'de>>(d: D) -> Result<u32, D::Error> {
    let umur = u32::deserialize(d)?;
    if umur < 18 || umur > 120 {
        Err(D::Error::custom(format!("Umur {} tidak munasabah (18-120)", umur)))
    } else {
        Ok(umur)
    }
}

#[derive(Debug, Deserialize)]
struct DaftarMasuk {
    #[serde(deserialize_with = "deserialize_emel")]
    emel: String,

    #[serde(deserialize_with = "deserialize_umur")]
    umur: u32,

    nama: String,
}

fn main() {
    // Emel betul
    let ok = r#"{"emel":"ali@test.com","umur":25,"nama":"Ali"}"#;
    match serde_json::from_str::<DaftarMasuk>(ok) {
        Ok(d)  => println!("✔ {:?}", d),
        Err(e) => println!("✘ {}", e),
    }

    // Emel salah
    let bad_email = r#"{"emel":"bukan-emel","umur":25,"nama":"Ali"}"#;
    match serde_json::from_str::<DaftarMasuk>(bad_email) {
        Ok(d)  => println!("✔ {:?}", d),
        Err(e) => println!("✘ {}", e), // ← error!
    }

    // Umur tidak munasabah
    let bad_age = r#"{"emel":"ali@test.com","umur":200,"nama":"Ali"}"#;
    match serde_json::from_str::<DaftarMasuk>(bad_age) {
        Ok(d)  => println!("✔ {:?}", d),
        Err(e) => println!("✘ {}", e), // ← error!
    }
}
```

---

## 🧠 Brain Teaser #4

Bagaimana serialize `HashMap<String, Vec<i32>>` ke JSON dan parse semula?

<details>
<summary>👀 Jawapan</summary>

```rust
use std::collections::HashMap;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // HashMap<String, Vec<i32>> boleh serialize terus!
    let mut data: HashMap<String, Vec<i32>> = HashMap::new();
    data.insert("pagi".into(),  vec![8, 9, 10]);
    data.insert("tengah".into(), vec![12, 13]);
    data.insert("petang".into(), vec![14, 15, 16, 17]);

    // Serialize → JSON object dengan array values
    let json = serde_json::to_string_pretty(&data)?;
    println!("{}", json);
    // {
    //   "pagi": [8, 9, 10],
    //   "tengah": [12, 13],
    //   "petang": [14, 15, 16, 17]
    // }

    // Deserialize balik
    let balik: HashMap<String, Vec<i32>> = serde_json::from_str(&json)?;
    println!("pagi: {:?}", balik["pagi"]); // [8, 9, 10]

    Ok(())
}
```

`HashMap`, `Vec`, `String`, semua primitive types — semuanya implement `Serialize/Deserialize` secara automatik dalam standard library!
</details>

---

# BAB 10: Mini Project — KADA Config & API 🏗️

```rust
use serde::{Serialize, Deserialize};
use std::collections::HashMap;
use std::fs;

// ─── Config Types ─────────────────────────────────────────────

#[derive(Debug, Serialize, Deserialize, Clone)]
struct KadaConfig {
    aplikasi: AplikasiConfig,
    pangkalan_data: PangkalanDataConfig,
    keselamatan: KeselamatanConfig,
    ciri: CiriConfig,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
struct AplikasiConfig {
    nama:    String,
    versi:   String,
    port:    u16,
    debug:   bool,
    #[serde(default = "lalai_pekerja")]
    pekerja: u32,
}

fn lalai_pekerja() -> u32 { 4 }

#[derive(Debug, Serialize, Deserialize, Clone)]
struct PangkalanDataConfig {
    url:      String,
    #[serde(default = "lalai_pool")]
    pool_min: u32,
    #[serde(default = "lalai_pool_max")]
    pool_max: u32,
    ssl:      bool,
}

fn lalai_pool() -> u32 { 2 }
fn lalai_pool_max() -> u32 { 20 }

#[derive(Debug, Serialize, Deserialize, Clone)]
struct KeselamatanConfig {
    jwt_rahsia:    String,
    token_tempoh:  u64,     // dalam saat
    cors_asal:     Vec<String>,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
#[serde(rename_all = "snake_case")]
struct CiriConfig {
    gps_aktif:         bool,
    selfie_aktif:      bool,
    notifikasi_aktif:  bool,
    kehadiran_radius:  f64,  // km
}

// ─── API Types ────────────────────────────────────────────────

#[derive(Debug, Serialize, Deserialize)]
struct ApiResponse<T: Serialize> {
    berjaya: bool,
    #[serde(skip_serializing_if = "Option::is_none")]
    data:    Option<T>,
    #[serde(skip_serializing_if = "Option::is_none")]
    mesej:   Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    ralat:   Option<String>,
}

impl<T: Serialize> ApiResponse<T> {
    fn ok(data: T) -> Self {
        ApiResponse { berjaya: true, data: Some(data), mesej: None, ralat: None }
    }
    fn ok_dengan_mesej(data: T, mesej: &str) -> Self {
        ApiResponse {
            berjaya: true,
            data: Some(data),
            mesej: Some(mesej.into()),
            ralat: None,
        }
    }
}

impl ApiResponse<()> {
    fn gagal(ralat: &str) -> Self {
        ApiResponse { berjaya: false, data: None, mesej: None, ralat: Some(ralat.into()) }
    }
}

#[derive(Debug, Serialize, Deserialize, Clone)]
#[serde(rename_all = "camelCase")]
struct Pekerja {
    id:           u32,
    nama:         String,
    no_pekerja:   String,
    bahagian:     String,
    #[serde(skip_serializing_if = "Option::is_none")]
    emel:         Option<String>,
    aktif:        bool,
}

#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
struct DaftarKehadiran {
    id_pekerja:  u32,
    latitud:     f64,
    longitud:    f64,
    masa:        String,
    #[serde(skip_serializing_if = "Option::is_none")]
    selfie_url:  Option<String>,
}

#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
struct RekodKehadiran {
    id:          u32,
    id_pekerja:  u32,
    masa_masuk:  String,
    #[serde(skip_serializing_if = "Option::is_none")]
    masa_keluar: Option<String>,
    lokasi:      Lokasi,
    status:      StatusKehadiran,
}

#[derive(Debug, Serialize, Deserialize)]
struct Lokasi {
    lat: f64,
    lon: f64,
}

#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "SCREAMING_SNAKE_CASE")]
enum StatusKehadiran {
    DalamKawasan,
    LuarKawasan,
    TidakPasti,
}

// ─── Simulasi API ─────────────────────────────────────────────

fn dapatkan_senarai_pekerja() -> ApiResponse<Vec<Pekerja>> {
    let pekerja = vec![
        Pekerja {
            id: 1, nama: "Ali Ahmad".into(), no_pekerja: "KADA001".into(),
            bahagian: "ICT".into(), emel: Some("ali@kada.gov.my".into()), aktif: true,
        },
        Pekerja {
            id: 2, nama: "Siti Hawa".into(), no_pekerja: "KADA002".into(),
            bahagian: "HR".into(), emel: None, aktif: true,
        },
        Pekerja {
            id: 3, nama: "Amin Razak".into(), no_pekerja: "KADA003".into(),
            bahagian: "Kewangan".into(), emel: Some("amin@kada.gov.my".into()), aktif: false,
        },
    ];
    ApiResponse::ok(pekerja)
}

fn daftar_kehadiran(input: DaftarKehadiran) -> Result<ApiResponse<RekodKehadiran>, String> {
    // Validate radius (simulasi)
    let jarak = jarak_haversine(
        input.latitud, input.longitud,
        6.0538, 102.2503 // koordinat KADA
    );

    let status = if jarak <= 1.0 {
        StatusKehadiran::DalamKawasan
    } else if jarak <= 2.0 {
        StatusKehadiran::TidakPasti
    } else {
        StatusKehadiran::LuarKawasan
    };

    let rekod = RekodKehadiran {
        id:          1001,
        id_pekerja:  input.id_pekerja,
        masa_masuk:  input.masa,
        masa_keluar: None,
        lokasi:      Lokasi { lat: input.latitud, lon: input.longitud },
        status,
    };

    Ok(ApiResponse::ok_dengan_mesej(rekod, "Kehadiran berjaya direkodkan"))
}

fn jarak_haversine(lat1: f64, lon1: f64, lat2: f64, lon2: f64) -> f64 {
    let dlat = (lat2 - lat1).to_radians();
    let dlon = (lon2 - lon1).to_radians();
    let a = (dlat/2.0).sin().powi(2)
          + lat1.to_radians().cos() * lat2.to_radians().cos()
          * (dlon/2.0).sin().powi(2);
    6371.0 * 2.0 * a.sqrt().atan2((1.0-a).sqrt())
}

// ─── Main ──────────────────────────────────────────────────────

fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("=== KADA Mobile API Demo ===\n");

    // 1. Load config dari TOML
    let config_toml = r#"
        [aplikasi]
        nama    = "KADA Mobile API"
        versi   = "2.1.0"
        port    = 8080
        debug   = false

        [pangkalan_data]
        url     = "mysql://localhost/kada_db"
        ssl     = true

        [keselamatan]
        jwt_rahsia   = "super_secret_key_production"
        token_tempoh = 86400
        cors_asal    = ["https://app.kada.gov.my", "http://localhost:3000"]

        [ciri]
        gps_aktif        = true
        selfie_aktif     = true
        notifikasi_aktif = false
        kehadiran_radius = 1.0
    "#;

    let config: KadaConfig = toml::from_str(config_toml)?;
    println!("✔ Config dimuatkan: {} v{}", config.aplikasi.nama, config.aplikasi.versi);
    println!("  Port: {}", config.aplikasi.port);
    println!("  GPS: {}, Selfie: {}", config.ciri.gps_aktif, config.ciri.selfie_aktif);

    // 2. GET /api/pekerja
    println!("\n--- GET /api/pekerja ---");
    let resp_senarai = dapatkan_senarai_pekerja();
    let json = serde_json::to_string_pretty(&resp_senarai)?;
    println!("{}", json);

    // 3. POST /api/kehadiran/masuk — dalam kawasan
    println!("\n--- POST /api/kehadiran/masuk (dalam kawasan) ---");
    let input_json = r#"{
        "idPekerja":  1,
        "latitud":    6.0540,
        "longitud":   102.2510,
        "masa":       "2024-01-15T08:00:00",
        "selfieUrl":  null
    }"#;

    let input: DaftarKehadiran = serde_json::from_str(input_json)?;
    match daftar_kehadiran(input) {
        Ok(resp) => println!("{}", serde_json::to_string_pretty(&resp)?),
        Err(e)   => println!("Error: {}", e),
    }

    // 4. POST /api/kehadiran/masuk — luar kawasan
    println!("\n--- POST /api/kehadiran/masuk (luar kawasan) ---");
    let input_luar = DaftarKehadiran {
        id_pekerja: 2,
        latitud:    3.1570,
        longitud:   101.7120,
        masa:       "2024-01-15T08:05:00".into(),
        selfie_url: None,
    };
    match daftar_kehadiran(input_luar) {
        Ok(resp) => println!("{}", serde_json::to_string_pretty(&resp)?),
        Err(e)   => println!("Error: {}", e),
    }

    // 5. Save config ke JSON
    println!("\n--- Save config ke JSON ---");
    let config_json = serde_json::to_string_pretty(&config)?;
    fs::write("kada_config.json", &config_json)?;
    println!("✔ Config disimpan ke kada_config.json");

    // Load semula dari JSON
    let loaded: KadaConfig = serde_json::from_str(&config_json)?;
    println!("✔ Config dimuatkan semula: {}", loaded.aplikasi.nama);

    fs::remove_file("kada_config.json")?;
    Ok(())
}
```

---

# 📋 Rujukan Pantas — Serde Cheat Sheet

## Setup

```toml
serde      = { version = "1", features = ["derive"] }
serde_json = "1"          # JSON
toml       = "0.8"        # TOML
serde_yaml = "0.9"        # YAML
csv        = "1"          # CSV
bincode    = "1"          # Binary
```

## Derive

```rust
#[derive(Serialize, Deserialize)]  // kedua-dua
#[derive(Serialize)]               // serialize sahaja
#[derive(Deserialize)]             // deserialize sahaja
```

## Serialize Functions

```rust
serde_json::to_string(&v)          // → Result<String>
serde_json::to_string_pretty(&v)   // → Result<String> (indent)
serde_json::to_vec(&v)             // → Result<Vec<u8>>
serde_json::to_writer(w, &v)       // → Result<()> (ke Writer)
```

## Deserialize Functions

```rust
serde_json::from_str(s)            // &str → Result<T>
serde_json::from_slice(b)          // &[u8] → Result<T>
serde_json::from_reader(r)         // Reader → Result<T>
```

## Attributes Penting

```rust
// Struct/Enum level
#[serde(rename = "NewName")]
#[serde(rename_all = "camelCase")]   // camelCase/snake_case/PascalCase
#[serde(deny_unknown_fields)]
#[serde(default)]
#[serde(transparent)]

// Field level
#[serde(rename = "new_name")]
#[serde(alias = "alt_name")]
#[serde(default)]
#[serde(default = "fn_name")]
#[serde(skip)]
#[serde(skip_serializing)]
#[serde(skip_deserializing)]
#[serde(skip_serializing_if = "Option::is_none")]
#[serde(flatten)]
#[serde(with = "module")]
#[serde(serialize_with = "fn")]
#[serde(deserialize_with = "fn")]

// Enum tagging
#[serde(tag = "type")]              // internal
#[serde(tag = "t", content = "c")]  // adjacent
#[serde(untagged)]                  // untagged
```

## rename_all Values

```
"lowercase"       → nama → "nama"
"UPPERCASE"       → nama → "NAMA"
"PascalCase"      → nama_field → "NamaField"
"camelCase"       → nama_field → "namaField"
"snake_case"      → namaField → "nama_field"
"SCREAMING_SNAKE_CASE" → namaField → "NAMA_FIELD"
"kebab-case"      → nama_field → "nama-field"
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **JSON Differ** — bandingkan dua JSON, keluarkan field yang berbeza
2. **Config Merger** — merge beberapa TOML config dengan priority
3. **API Client** — guna `reqwest` + serde untuk call REST API sebenar
4. **Schema Validator** — validate JSON mengikut schema yang didefine dalam Rust struct
5. **Data Migration** — convert struktur JSON lama ke format baru dengan serde

---

*Serde in Rust — dari `#[derive(Serialize)]` mudah hingga custom serializer dan API lengkap. Selamat belajar!* 🦀
