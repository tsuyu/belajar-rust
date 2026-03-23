# ⚠️ unwrap() vs ? — Panduan Lengkap Error Handling Rust

> "unwrap() bukan jahat. ? bukan sentiasa lebih baik."
> Faham bila guna mana, cara propagate errors, dan
> bagaimana bina error handling yang robust.

---

## Premis: Dua Sekolah Pemikiran

```
Pemula Rust:
  "Tambah .unwrap() sehingga compile, debug kemudian"

Veteran Rust:
  "Tambah ? sehingga propagate, handle di atas"

Pragmatik Rust:
  "Faham apa yang berlaku untuk SETIAP kes.
   unwrap() ada tempatnya. ? ada tempatnya.
   expect() ada tempatnya. match ada tempatnya."

Realiti:
  unwrap() dalam production code = tanda bahaya
  ? dalam main() tanpa context = output buruk
  Tiada satu saiz untuk semua
```

---

## Peta Pembelajaran

```
Bab 1  → Option & Result — Asas
Bab 2  → unwrap(), expect(), dan panic!
Bab 3  → ? Operator — Magic di Sebaliknya
Bab 4  → Kos & Manfaat: unwrap vs ?
Bab 5  → match & if let — Full Control
Bab 6  → Error Propagation Patterns
Bab 7  → Custom Error Types
Bab 8  → anyhow & thiserror
Bab 9  → Strategi Error Handling Mengikut Layer
Bab 10 → Mini Project: Error Handling yang Betul
```

---

# BAB 1: Option & Result — Asas 🧱

## Dua Container Error Rust

```rust
// Option<T> = "mungkin ada, mungkin tiada"
// Result<T, E> = "berjaya dengan nilai, atau gagal dengan ralat"

// ── Option<T> ─────────────────────────────────────────────────
enum Option<T> {
    Some(T),  // Ada nilai
    None,     // Tiada nilai
}

// Contoh guna Option
fn cari_pekerja(id: u32) -> Option<String> {
    match id {
        1 => Some("Ali Ahmad".into()),
        2 => Some("Siti Hawa".into()),
        _ => None,  // tidak dijumpai
    }
}

// ── Result<T, E> ──────────────────────────────────────────────
enum Result<T, E> {
    Ok(T),   // Berjaya, ada nilai T
    Err(E),  // Gagal, ada ralat E
}

// Contoh guna Result
fn bahagi(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err("Tidak boleh bahagi dengan sifar!".into())
    } else {
        Ok(a / b)
    }
}

fn main() {
    // Option
    println!("{:?}", cari_pekerja(1));  // Some("Ali Ahmad")
    println!("{:?}", cari_pekerja(99)); // None

    // Result
    println!("{:?}", bahagi(10.0, 2.0)); // Ok(5.0)
    println!("{:?}", bahagi(10.0, 0.0)); // Err("Tidak boleh...")
}
```

---

## Cara Handle Option & Result

```rust
fn main() {
    let opt: Option<i32> = Some(42);
    let res: Result<i32, &str> = Ok(42);

    // ── 5 cara handle Option ──────────────────────────────────

    // 1. match — kawalan penuh
    match opt {
        Some(v) => println!("Ada: {}", v),
        None    => println!("Tiada"),
    }

    // 2. if let — bila hanya kes Some yang penting
    if let Some(v) = opt {
        println!("Ada: {}", v);
    }

    // 3. unwrap() — PANIC kalau None
    let v = opt.unwrap(); // OK kalau pasti ada

    // 4. unwrap_or() — nilai default kalau None
    let v = opt.unwrap_or(0);
    let v = opt.unwrap_or_else(|| kira_nilai_default());
    let v = opt.unwrap_or_default(); // guna Default::default()

    // 5. ? — propagate None sebagai None (dalam fungsi yang return Option)
    fn dari_option() -> Option<i32> {
        let v = opt?; // kalau None, return None dari fungsi
        Some(v * 2)
    }

    // ── 5 cara handle Result ──────────────────────────────────

    // 1. match — kawalan penuh
    match res {
        Ok(v)  => println!("Berjaya: {}", v),
        Err(e) => println!("Gagal: {}", e),
    }

    // 2. if let
    if let Ok(v) = res { println!("Berjaya: {}", v); }
    if let Err(e) = res { println!("Gagal: {}", e); }

    // 3. unwrap() / expect()
    let v = res.unwrap();
    let v = res.expect("Sepatutnya berjaya");

    // 4. unwrap_or variants
    let v = res.unwrap_or(0);
    let v = res.unwrap_or_else(|e| { eprintln!("{}", e); 0 });
    let v = res.unwrap_or_default();

    // 5. ? — propagate Err
    fn dari_result() -> Result<i32, &'static str> {
        let v = res?; // kalau Err, return Err dari fungsi
        Ok(v * 2)
    }
}

fn kira_nilai_default() -> i32 { 99 }
```

---

# BAB 2: unwrap(), expect(), dan panic! 💥

## Bila unwrap() SELAMAT

```rust
// ── Kes yang betul guna unwrap() ─────────────────────────────

// 1. Dalam test code — panic adalah OK dalam test
#[test]
fn test_parsing() {
    let n: i32 = "42".parse().unwrap(); // BETUL dalam test
    assert_eq!(n, 42);
}

// 2. Bila logik membuktikan ia tidak boleh fail
fn main() {
    // Kita tahu "42" boleh di-parse sebagai i32
    let n: i32 = "42".parse().unwrap(); // OK — string literal tidak akan fail

    // Kita tahu Vec tidak kosong selepas semak
    let v = vec![1, 2, 3];
    assert!(!v.is_empty());
    let _ = v.first().unwrap(); // OK — sudah semak tidak kosong

    // Regex yang di-compile dari static string
    let re = regex::Regex::new(r"^\d+$").unwrap(); // OK — pattern betul
}

// 3. Prototyping / exploration
fn main_proto() {
    let hasil = buat_sesuatu().unwrap(); // OK semasa prototype
    println!("{}", hasil);
    // TODO: handle error properly before production
}

// 4. Bila error bermaksud bug, bukan user error
fn indeks_betul(v: &[i32], i: usize) -> i32 {
    // Caller patut pastikan i dalam julat
    // Kalau tidak → ia adalah BUG dalam caller
    *v.get(i).expect("Indeks di luar julat — ini adalah BUG!")
}

// 5. Bila mutex poisoned adalah tidak mungkin dalam konteks kita
use std::sync::Mutex;
static DATA: Mutex<Vec<i32>> = Mutex::new(Vec::new());

fn tambah(nilai: i32) {
    DATA.lock().unwrap().push(nilai); // unwrap OK kalau tahu tiada panic dalam lock
}
```

---

## expect() — unwrap() yang Lebih Informatif

```rust
// expect() = unwrap() + mesej yang bermakna

// ❌ BURUK — mesej tidak membantu
let fail = std::fs::File::open("data.txt").unwrap();
// Kalau gagal: "called `Result::unwrap()` on an `Err` value: Os { code: 2, ... }"
// → Tidak tahu fail MANA, KENAPA perlu, dalam KONTEKS apa

// ✔ BAIK — mesej yang berguna
let fail = std::fs::File::open("data.txt")
    .expect("Fail konfigurasi 'data.txt' diperlukan dalam direktori semasa");
// Kalau gagal: "Fail konfigurasi 'data.txt' diperlukan dalam direktori semasa: Os { ... }"
// → Jelas apa yang gagal dan kenapa!

// Panduan mesej expect() yang baik:
// ✔ Terangkan APAKAH yang sepatutnya benar
// ✔ Terangkan KENAPA ia diperlukan
// ✗ Jangan: "unwrap failed" atau "should work" — tidak berguna!

// Contoh-contoh baik:
"jwt_secret" .parse::<u64>()
    .expect("JWT_SECRET dalam .env mesti nombor");

std::env::var("DATABASE_URL")
    .expect("DATABASE_URL mesti ditetapkan dalam .env");

v.first()
    .expect("Vec tidak sepatutnya kosong — sudah disemak di atas");
```

---

## 🧠 Brain Teaser #1

Apakah perbezaan output dan implikasi?

```rust
fn main() {
    let v: Vec<i32> = vec![];

    // A
    let _ = v[0];                        // cara A

    // B
    let _ = v.get(0).unwrap();           // cara B

    // C
    let _ = v.get(0).expect("Vec kosong!"); // cara C
}
```

<details>
<summary>👀 Jawapan</summary>

```
A: v[0] → PANIC dengan "index out of bounds: the len is 0 but the index is 0"
   → Rust runtime semak bounds
   → Tiada cara nak handle, terus panic

B: v.get(0) → return Option: None
   .unwrap() → PANIC dengan "called `Option::unwrap()` on a `None` value"
   → Mesej tidak berguna — tidak tahu KENAPA kosong, dari mana

C: v.get(0) → return Option: None
   .expect("Vec kosong!") → PANIC dengan "Vec kosong!: called `Option::unwrap()` on a `None` value"
   → Mesej lebih berguna, tahu konteks

PERBEZAAN KRITIKAL:
   A  → Debug sukar (kena trace ke mana v[0] dipanggil)
   B  → Debug sukar (mesej generik)
   C  → Debug lebih mudah (ada konteks)
   D (tidak dalam soal) = handle dengan ? atau match → terbaik!

IMPLIKASI:
   .get() lebih selamat dari [] kerana return Option
   Dapat handle None dengan betul
   [] untuk kes yang kita PASTI indeks betul (dan terima panic kalau tidak)
```
</details>

---

# BAB 3: ? Operator — Magic di Sebaliknya 🪄

## Apa yang ? Buat

```rust
use std::fs;
use std::io;
use std::num::ParseIntError;

// ── Tanpa ? ───────────────────────────────────────────────────
fn baca_port_v1(laluan: &str) -> Result<u16, io::Error> {
    let kandungan = match fs::read_to_string(laluan) {
        Ok(s)  => s,
        Err(e) => return Err(e), // ← return Err awal
    };

    let port: u16 = match kandungan.trim().parse() {
        Ok(n)  => n,
        Err(e) => return Err(io::Error::new(io::ErrorKind::InvalidData, e)),
    };

    Ok(port)
}

// ── Dengan ? ──────────────────────────────────────────────────
fn baca_port_v2(laluan: &str) -> Result<u16, io::Error> {
    let kandungan = fs::read_to_string(laluan)?; // return Err kalau gagal
    let port: u16 = kandungan.trim().parse()
        .map_err(|e| io::Error::new(io::ErrorKind::InvalidData, e))?;
    Ok(port)
}

// ── Apa sebenarnya ? buat ─────────────────────────────────────
// ? = gula sintaks untuk:
//
// expression?
//
// ekspand kepada:
//
// match expression {
//     Ok(val) => val,              // nilai diambil, teruskan
//     Err(e)  => return Err(e.into()), // convert dan return awal
// }
//
// Nota: .into() bermaksud ? boleh AUTO-CONVERT error type!
// Ini menggunakan trait From<SourceError> for TargetError

// ── ? dengan Option ───────────────────────────────────────────
fn cari_panjang(s: Option<&str>) -> Option<usize> {
    let teks = s?;        // return None kalau s adalah None
    Some(teks.len())      // laksana kalau Some
}

fn main() {
    println!("{:?}", cari_panjang(Some("hello"))); // Some(5)
    println!("{:?}", cari_panjang(None));           // None
}
```

---

## Auto-conversion dengan From

```rust
use thiserror::Error;
use std::io;
use std::num::ParseIntError;

// ? boleh convert error type secara automatik
// KALAU From trait diimplementasikan!

#[derive(Debug, Error)]
enum AppError {
    #[error("IO error: {0}")]
    Io(#[from] io::Error),         // From<io::Error> for AppError

    #[error("Parse error: {0}")]
    Parse(#[from] ParseIntError),  // From<ParseIntError> for AppError
}

fn baca_nombor(laluan: &str) -> Result<i32, AppError> {
    let kandungan = std::fs::read_to_string(laluan)?;
    //                                             ↑
    // io::Error auto-convert ke AppError::Io
    // kerana From<io::Error> for AppError

    let n: i32 = kandungan.trim().parse()?;
    //                                    ↑
    // ParseIntError auto-convert ke AppError::Parse
    // kerana From<ParseIntError> for AppError

    Ok(n)
}

// ── Tanpa From — perlu manual convert ────────────────────────
fn tanpa_from(laluan: &str) -> Result<i32, String> {
    let kandungan = std::fs::read_to_string(laluan)
        .map_err(|e| e.to_string())?; // manual convert

    let n: i32 = kandungan.trim().parse()
        .map_err(|e: ParseIntError| e.to_string())?; // manual convert

    Ok(n)
}
```

---

## ? dalam Pelbagai Konteks

```rust
use std::io;

// ── ? dalam async fn ──────────────────────────────────────────
async fn baca_async(laluan: &str) -> Result<String, io::Error> {
    let kandungan = tokio::fs::read_to_string(laluan).await?;
    Ok(kandungan.to_uppercase())
}

// ── ? dalam iterator (TIDAK boleh!) ──────────────────────────
fn proses_list(senarai: &[&str]) -> Result<Vec<i32>, std::num::ParseIntError> {
    // ❌ ? dalam closure tidak berfungsi macam yang disangka
    // let v: Vec<i32> = senarai.iter()
    //     .map(|s| s.parse::<i32>()?)  // ERROR! ? dalam closure
    //     .collect();

    // ✔ Cara betul dengan collect::<Result<Vec<_>, _>>
    let v: Vec<i32> = senarai.iter()
        .map(|s| s.parse::<i32>())
        .collect::<Result<Vec<_>, _>>()?; // collect error, kemudian ?

    Ok(v)
}

fn main() {
    // ✔
    println!("{:?}", proses_list(&["1", "2", "3"]));  // Ok([1, 2, 3])
    // ✔ — error propagated
    println!("{:?}", proses_list(&["1", "abc", "3"])); // Err(ParseIntError...)
}
```

---

# BAB 4: Kos & Manfaat — unwrap vs ? ⚖️

## Perbandingan Sebenar

```
UNWRAP():
  Pros:
    ✔ Ringkas untuk prototype
    ✔ Jelas "ini tidak sepatutnya fail"
    ✔ Fail cepat dengan lokasi yang jelas (dengan backtrace)
    ✔ Sesuai dalam test, main, dan CLI tools sederhana

  Cons:
    ✗ Panic — tidak boleh di-recover
    ✗ Mesej generik (tanpa expect)
    ✗ Tidak memaksa caller untuk fikir tentang ralat
    ✗ Berbahaya dalam library code
    ✗ Berbahaya dalam multi-user web server

? OPERATOR:
  Pros:
    ✔ Propagate error — caller boleh handle
    ✔ Force API user untuk acknowledge ralat
    ✔ Composable — chain operasi yang boleh fail
    ✔ Sesuai untuk library dan application core code

  Cons:
    ✗ Fungsi mesti return Result/Option
    ✗ Error types perlu compatible (atau ada From impl)
    ✗ Boleh "hide" ralat kalau tidak di-handle di atas
    ✗ Dalam main(), perlu handle output ralat dengan baik

PANDUAN RINGKAS:
  Library code        → ? sentiasa
  Application core    → ? dengan custom error
  Binary/CLI main()   → ? atau anyhow, handle output
  Test code           → unwrap()/expect() boleh
  Prototype           → unwrap() boleh, TODO sebelum production
  Web server handler  → ? dengan AppError yang impl IntoResponse
  Setup/init          → expect() dengan mesej yang bermakna
```

---

# BAB 5: match & if let — Full Control 🎛️

## Bila Perlu Full Control

```rust
// match memberi kawalan penuh — tahu TEPAT apa yang berlaku

use std::io;

fn proses_fail(laluan: &str) -> String {
    match std::fs::read_to_string(laluan) {
        Ok(kandungan) => {
            // Berjaya — proses kandungan
            kandungan.to_uppercase()
        }
        Err(e) if e.kind() == io::ErrorKind::NotFound => {
            // Fail tidak jumpa — guna default
            eprintln!("Fail '{}' tidak dijumpai, guna nilai lalai", laluan);
            "DEFAULT".into()
        }
        Err(e) if e.kind() == io::ErrorKind::PermissionDenied => {
            // Tiada kebenaran — ini serius
            eprintln!("Tiada kebenaran baca fail '{}'", laluan);
            panic!("Kebenaran fail tidak mencukupi");
        }
        Err(e) => {
            // Ralat lain yang tidak dijangka
            eprintln!("Ralat tidak dijangka: {}", e);
            String::new()
        }
    }
}

// ── match dengan multiple error types ─────────────────────────
#[derive(Debug)]
enum ParsError {
    IoError(io::Error),
    ParseError(std::num::ParseIntError),
    NilaiLuarJulat(i32),
}

fn baca_dan_sahkan(laluan: &str) -> Result<u8, ParsError> {
    let kandungan = std::fs::read_to_string(laluan)
        .map_err(ParsError::IoError)?;

    let n: i32 = kandungan.trim().parse()
        .map_err(ParsError::ParseError)?;

    if n < 0 || n > 255 {
        return Err(ParsError::NilaiLuarJulat(n));
    }

    Ok(n as u8)
}

fn main() {
    match baca_dan_sahkan("nilai.txt") {
        Ok(n)                           => println!("Nilai: {}", n),
        Err(ParsError::IoError(e))
            if e.kind() == io::ErrorKind::NotFound
                                        => println!("Fail tiada"),
        Err(ParsError::IoError(e))      => println!("IO error: {}", e),
        Err(ParsError::ParseError(e))   => println!("Format salah: {}", e),
        Err(ParsError::NilaiLuarJulat(n)) => println!("Nilai {} luar julat", n),
    }
}
```

---

## Method Berguna pada Option & Result

```rust
fn main() {
    let opt: Option<i32>         = Some(42);
    let res: Result<i32, String> = Ok(42);

    // ── Transformations ───────────────────────────────────────

    // .map() — transform nilai kalau ada/Ok
    let doubled = opt.map(|n| n * 2);      // Some(84)
    let doubled = res.map(|n| n * 2);      // Ok(84)

    // .map_err() — transform ralat kalau Err
    let r = res.map_err(|e| e.len());      // Ok(42) — err tidak berubah
    let r: Result<i32, usize> = Err("gagal".into());
    let r2 = r.map_err(|e: String| e.len()); // Err(5)

    // .and_then() / .flat_map() — chain yang mungkin fail
    let r = opt.and_then(|n| if n > 0 { Some(n) } else { None });

    // .or_else() — fallback kalau None/Err
    let r = opt.or_else(|| Some(0));         // Some(42) — ada, guna
    let r: Option<i32> = None;
    let r2 = r.or_else(|| Some(0));          // Some(0) — tiada, guna default

    // .filter() — None kalau tidak lulus predikat (Option sahaja)
    let r = opt.filter(|&n| n > 100);  // None — 42 tidak > 100

    // ── Conversion ────────────────────────────────────────────

    // Option → Result
    let r: Result<i32, &str> = opt.ok_or("tiada nilai");
    let r = opt.ok_or_else(|| "dikira".to_string());

    // Result → Option (buang error)
    let o: Option<i32> = res.ok();         // Some(42) — buang Err
    let o: Option<String> = res.err();     // None — buang Ok

    // ── Debugging ────────────────────────────────────────────

    // .inspect() — log tanpa consume (Rust 1.76+)
    let r = res.inspect(|v| println!("OK: {}", v))
               .inspect_err(|e| println!("Err: {}", e));

    // ── Collecting ───────────────────────────────────────────

    // Vec<Option<T>> → Option<Vec<T>>
    let v: Vec<Option<i32>> = vec![Some(1), Some(2), Some(3)];
    let r: Option<Vec<i32>> = v.into_iter().collect();  // Some([1,2,3])

    // Vec<Result<T,E>> → Result<Vec<T>,E>
    let v: Vec<Result<i32, &str>> = vec![Ok(1), Ok(2), Ok(3)];
    let r: Result<Vec<i32>, &str> = v.into_iter().collect();  // Ok([1,2,3])
}
```

---

# BAB 6: Error Propagation Patterns 🔄

## Pattern 1: Chain dengan ?

```rust
use std::{fs, io, path::Path};
use serde::Deserialize;

#[derive(Deserialize, Debug)]
struct KonfigDB { host: String, port: u16, nama_db: String }

fn muat_konfig(laluan: &Path) -> Result<KonfigDB, Box<dyn std::error::Error>> {
    let kandungan = fs::read_to_string(laluan)?;  // io::Error
    let konfig: KonfigDB = serde_json::from_str(&kandungan)?; // serde::Error
    Ok(konfig)
}
// Box<dyn Error> = boleh accommodate apa-apa error (tapi hilang type info)
```

---

## Pattern 2: map_err untuk Convert

```rust
fn ambil_port() -> Result<u16, String> {
    let s = std::env::var("PORT")
        .map_err(|_| "PORT tidak ditetapkan dalam env".to_string())?;

    let port: u16 = s.parse()
        .map_err(|_| format!("PORT '{}' bukan nombor yang sah", s))?;

    if port < 1024 {
        return Err(format!("Port {} tidak dibenarkan (< 1024)", port));
    }

    Ok(port)
}
```

---

## Pattern 3: Fallback dengan or_else

```rust
fn muat_konfigurasi(laluan_utama: &str, laluan_backup: &str) -> String {
    std::fs::read_to_string(laluan_utama)
        .or_else(|_| std::fs::read_to_string(laluan_backup))
        .unwrap_or_else(|_| "konfigurasi lalai".into())
}
```

---

## Pattern 4: Context dengan anyhow

```rust
use anyhow::{Context, Result, bail, ensure};

fn proses_data(laluan: &str) -> Result<Vec<i32>> {
    let kandungan = std::fs::read_to_string(laluan)
        .with_context(|| format!("Gagal baca fail '{}'", laluan))?;
    //              ↑ tambah context ke error tanpa hilang original!

    kandungan
        .lines()
        .enumerate()
        .map(|(i, baris)| {
            baris.trim().parse::<i32>()
                .with_context(|| format!("Baris {} '{}' bukan integer", i + 1, baris))
        })
        .collect()
}

fn sahkan_umur(umur: i32) -> Result<()> {
    ensure!(umur >= 0,   "Umur tidak boleh negatif: {}", umur);
    ensure!(umur <= 150, "Umur tidak munasabah: {}", umur);
    Ok(())
}

fn status_aktif(aktif: bool) -> Result<()> {
    if !aktif { bail!("Pengguna tidak aktif"); }
    Ok(())
}

fn main() -> Result<()> {
    // Error message chain:
    // "Gagal baca fail 'data.txt': Os { code: 2, kind: NotFound, ... }"
    let data = proses_data("data.txt")?;
    println!("{:?}", data);
    Ok(())
}
```

---

## 🧠 Brain Teaser #2

Kod ini ada masalah. Apa masalahnya dan bagaimana betulkan?

```rust
fn proses(senarai: &[&str]) -> Result<Vec<i32>, String> {
    let mut hasil = Vec::new();
    for s in senarai {
        match s.parse::<i32>() {
            Ok(n)  => hasil.push(n),
            Err(e) => return Err(e.to_string()),
        }
    }
    Ok(hasil)
}

fn main() {
    let data = vec!["1", "2", "abc", "4"];
    match proses(&data) {
        Ok(v)  => println!("OK: {:?}", v),
        Err(e) => println!("Ralat: {}", e),
    }
}
```

Mesej ralat tidak berguna: "invalid digit found in string"
Tidak tahu MANA yang gagal!

<details>
<summary>👀 Jawapan</summary>

```rust
// MASALAH: Error message tidak ada konteks — "invalid digit found in string"
// Tidak tahu indeks atau nilai yang gagal!

// ✔ PENYELESAIAN 1: Tambah konteks ke error
fn proses_v1(senarai: &[&str]) -> Result<Vec<i32>, String> {
    senarai.iter().enumerate()
        .map(|(i, s)| {
            s.parse::<i32>()
                .map_err(|e| format!("Indeks {}: '{}' — {}", i, s, e))
        })
        .collect::<Result<Vec<_>, _>>()
}

// ✔ PENYELESAIAN 2: Guna anyhow untuk context chain
use anyhow::{Context, Result};

fn proses_v2(senarai: &[&str]) -> Result<Vec<i32>> {
    senarai.iter().enumerate()
        .map(|(i, s)| {
            s.parse::<i32>()
                .with_context(|| format!("Item ke-{} '{}' bukan integer", i + 1, s))
        })
        .collect()
}

fn main() {
    let data = vec!["1", "2", "abc", "4"];

    // Output yang berguna:
    // "Item ke-3 'abc' bukan integer: invalid digit found in string"
    match proses_v2(&data) {
        Ok(v)  => println!("{:?}", v),
        Err(e) => println!("Ralat: {:#}", e), // {:#} untuk full chain
    }
}

// PELAJARAN: Error handling yang baik = konteks yang cukup
// untuk debug tanpa membuka source code.
// Soal: "Boleh saya debug ini dari error message sahaja?"
// Kalau tidak → tambah konteks!
```
</details>

---

# BAB 7: Custom Error Types 🏗️

## thiserror — Error Types yang Kemas

```rust
use thiserror::Error;
use std::path::PathBuf;

#[derive(Debug, Error)]
pub enum AppError {
    // Error dengan field
    #[error("Fail '{laluan}' tidak dijumpai")]
    FailTidakJumpai { laluan: PathBuf },

    // Error dengan wrapped error (auto From)
    #[error("Ralat IO: {0}")]
    Io(#[from] std::io::Error),

    // Error dengan named dari wrapped error
    #[error("Format data tidak sah di baris {baris}: {punca}")]
    FormatSalah {
        baris: usize,
        punca: String,
    },

    // Error dengan source (untuk chain)
    #[error("Gagal parse konfigurasi")]
    ParseKonfig {
        #[source] // accessible via .source()
        punca: Box<dyn std::error::Error + Send + Sync>,
    },

    // Error transparent (delegate Display ke inner)
    #[error(transparent)]
    Lain(#[from] anyhow::Error),
}

// Guna:
fn baca_fail(laluan: &str) -> Result<String, AppError> {
    let p = std::path::PathBuf::from(laluan);
    if !p.exists() {
        return Err(AppError::FailTidakJumpai { laluan: p });
    }
    Ok(std::fs::read_to_string(&p)?) // io::Error → auto convert via From
}

fn main() {
    match baca_fail("tidak_ada.txt") {
        Ok(k)  => println!("{}", k),
        Err(AppError::FailTidakJumpai { laluan }) =>
            println!("Fail '{}' tiada!", laluan.display()),
        Err(AppError::Io(e)) =>
            println!("IO error: {}", e),
        Err(e) =>
            println!("Ralat lain: {}", e),
    }
}
```

---

# BAB 8: anyhow & thiserror 📦

## Bila Guna Mana

```rust
// ── thiserror — untuk LIBRARY code ───────────────────────────
// Bila caller perlu tahu exact error type untuk handle

use thiserror::Error;

#[derive(Debug, Error)]
pub enum LibraryError {
    #[error("Tidak dijumpai: {0}")]
    TidakJumpai(String),
    #[error("Tak sah: {0}")]
    TakSah(String),
}

// Library user boleh:
// match err { LibraryError::TidakJumpai(_) => ... }

// ── anyhow — untuk APPLICATION code ──────────────────────────
// Bila kita hanya perlu propagate dan log, tidak match

use anyhow::{anyhow, bail, ensure, Context, Result};

fn proses_app(input: &str) -> Result<String> {
    let n: i32 = input.parse()
        .context("Input mesti nombor integer")?;

    ensure!(n > 0, "Nombor mesti positif, dapat: {}", n);

    if n > 1000 {
        bail!("Nombor {} terlalu besar (max 1000)", n);
    }

    Ok(format!("Proses: {}", n * 2))
}

fn main() -> Result<()> {
    // anyhow::Result dalam main() → output error yang kemas!
    let hasil = proses_app("abc")?;
    println!("{}", hasil);
    Ok(())
}

// ── Gabungan keduanya (pattern biasa) ────────────────────────

// Library crate: guna thiserror untuk public API
// Binary crate:  guna anyhow untuk application logic

fn guna_library() -> anyhow::Result<()> {
    let hasil = library_fn()
        .context("Operasi library gagal")?; // Convert LibraryError → anyhow
    Ok(())
}

fn library_fn() -> Result<(), LibraryError> {
    Err(LibraryError::TidakJumpai("item".into()))
}
```

---

# BAB 9: Strategi Mengikut Layer 🏛️

## Error Handling Berbeza untuk Layer Berbeza

```rust
use thiserror::Error;
use axum::{response::{IntoResponse, Json}, http::StatusCode};
use anyhow::Context;

// ── Layer 1: Database / Repository ────────────────────────────
// Guna ? dan return Result<T, sqlx::Error> atau custom DbError

#[derive(Debug, Error)]
pub enum DbError {
    #[error("Rekod tidak dijumpai: id={0}")]
    TidakJumpai(u32),
    #[error("Ralat pangkalan data: {0}")]
    Sqlx(#[from] sqlx::Error),
}

async fn cari_pekerja_db(
    pool: &sqlx::MySqlPool,
    id: u32,
) -> Result<Pekerja, DbError> {
    sqlx::query_as!(Pekerja, "SELECT * FROM pekerja WHERE id = ?", id)
        .fetch_optional(pool)
        .await? // sqlx::Error → DbError::Sqlx
        .ok_or(DbError::TidakJumpai(id))
}

// ── Layer 2: Service / Business Logic ─────────────────────────
// Convert domain error, tambah business context

#[derive(Debug, Error)]
pub enum ServiceError {
    #[error("Pekerja {id} tidak dijumpai")]
    PekerjaTidakJumpai { id: u32 },
    #[error("Pekerja tidak aktif")]
    PekerjaAktif,
    #[error("Ralat sistem: {0}")]
    Sistem(#[from] DbError),
}

async fn dapatkan_pekerja_aktif(
    pool: &sqlx::MySqlPool,
    id: u32,
) -> Result<Pekerja, ServiceError> {
    let pekerja = cari_pekerja_db(pool, id)
        .await
        .map_err(|e| match e {
            DbError::TidakJumpai(id) => ServiceError::PekerjaTidakJumpai { id },
            e => ServiceError::Sistem(e),
        })?;

    if !pekerja.aktif {
        return Err(ServiceError::PekerjaAktif);
    }

    Ok(pekerja)
}

// ── Layer 3: Handler (API) ─────────────────────────────────────
// Convert ke HTTP response

#[derive(Debug, Error)]
pub enum AppError {
    #[error("{0}")]
    Service(#[from] ServiceError),
}

impl IntoResponse for AppError {
    fn into_response(self) -> axum::response::Response {
        let (status, mesej) = match &self {
            AppError::Service(ServiceError::PekerjaTidakJumpai { id }) =>
                (StatusCode::NOT_FOUND, format!("Pekerja {} tidak dijumpai", id)),
            AppError::Service(ServiceError::PekerjaAktif) =>
                (StatusCode::FORBIDDEN, "Pekerja tidak aktif".into()),
            AppError::Service(ServiceError::Sistem(_)) =>
                (StatusCode::INTERNAL_SERVER_ERROR, "Ralat sistem".into()),
        };

        (status, Json(serde_json::json!({ "ralat": mesej }))).into_response()
    }
}

// Handler bersih — hanya ? tanpa penuh match
async fn handler_pekerja(
    State(pool): State<sqlx::MySqlPool>,
    Path(id): Path<u32>,
) -> Result<Json<Pekerja>, AppError> {
    let pekerja = dapatkan_pekerja_aktif(&pool, id).await?;
    Ok(Json(pekerja))
}

struct Pekerja { id: u32, nama: String, aktif: bool }
```

---

# BAB 10: Mini Project — Error Handling yang Betul 🏗️

```rust
use std::path::{Path, PathBuf};
use std::collections::HashMap;
use thiserror::Error;
use anyhow::Context;

// ─── Error Types ──────────────────────────────────────────────

#[derive(Debug, Error)]
pub enum KonfigurasError {
    #[error("Fail konfigurasi '{laluan}' tidak dijumpai")]
    FailTidakJumpai { laluan: PathBuf },

    #[error("Format JSON tidak sah dalam '{laluan}': {punca}")]
    FormatTidakSah { laluan: PathBuf, punca: serde_json::Error },

    #[error("Kunci wajib '{kunci}' tidak ada dalam konfigurasi")]
    KunciTiada { kunci: String },

    #[error("Nilai '{nilai}' untuk '{kunci}' tidak sah: {sebab}")]
    NilaiTidakSah { kunci: String, nilai: String, sebab: String },

    #[error("Ralat IO: {0}")]
    Io(#[from] std::io::Error),
}

#[derive(Debug, Error)]
pub enum AppError {
    #[error("Konfigurasi tidak sah: {0}")]
    Konfigurasi(#[from] KonfigurasError),

    #[error("Operasi tidak dibenarkan: {0}")]
    Kebenaran(String),

    #[error(transparent)]
    Lain(#[from] anyhow::Error),
}

// ─── Config Loader ────────────────────────────────────────────

#[derive(Debug, Clone)]
pub struct Konfigurasi {
    pub host:       String,
    pub port:       u16,
    pub log_level:  String,
    pub pekerja:    u32,
}

impl Konfigurasi {
    pub fn muat(laluan: &Path) -> Result<Self, KonfigurasError> {
        // Semak fail wujud
        if !laluan.exists() {
            return Err(KonfigurasError::FailTidakJumpai {
                laluan: laluan.to_owned()
            });
        }

        // Baca fail — io::Error auto convert ke KonfigurasError::Io
        let kandungan = std::fs::read_to_string(laluan)?;

        // Parse JSON
        let peta: HashMap<String, serde_json::Value> =
            serde_json::from_str(&kandungan).map_err(|e| {
                KonfigurasError::FormatTidakSah {
                    laluan: laluan.to_owned(),
                    punca: e,
                }
            })?;

        // Extract fields dengan context
        let host = Self::ambil_string(&peta, "host")?;
        let port = Self::ambil_u16(&peta, "port")?;
        let log_level = Self::ambil_string(&peta, "log_level")?;
        let pekerja = Self::ambil_u32(&peta, "workers")?;

        Ok(Konfigurasi { host, port, log_level, pekerja })
    }

    fn ambil_string(
        peta:  &HashMap<String, serde_json::Value>,
        kunci: &str,
    ) -> Result<String, KonfigurasError> {
        peta.get(kunci)
            .ok_or_else(|| KonfigurasError::KunciTiada {
                kunci: kunci.into()
            })?
            .as_str()
            .ok_or_else(|| KonfigurasError::NilaiTidakSah {
                kunci: kunci.into(),
                nilai: peta[kunci].to_string(),
                sebab: "dijangka string".into(),
            })
            .map(|s| s.to_owned())
    }

    fn ambil_u16(
        peta:  &HashMap<String, serde_json::Value>,
        kunci: &str,
    ) -> Result<u16, KonfigurasError> {
        let n = peta.get(kunci)
            .ok_or_else(|| KonfigurasError::KunciTiada { kunci: kunci.into() })?
            .as_u64()
            .ok_or_else(|| KonfigurasError::NilaiTidakSah {
                kunci: kunci.into(),
                nilai: peta[kunci].to_string(),
                sebab: "dijangka nombor".into(),
            })?;

        if n > u16::MAX as u64 {
            return Err(KonfigurasError::NilaiTidakSah {
                kunci: kunci.into(),
                nilai: n.to_string(),
                sebab: format!("mestilah 0-{}", u16::MAX),
            });
        }

        Ok(n as u16)
    }

    fn ambil_u32(
        peta:  &HashMap<String, serde_json::Value>,
        kunci: &str,
    ) -> Result<u32, KonfigurasError> {
        let n = peta.get(kunci)
            .ok_or_else(|| KonfigurasError::KunciTiada { kunci: kunci.into() })?
            .as_u64()
            .ok_or_else(|| KonfigurasError::NilaiTidakSah {
                kunci: kunci.into(),
                nilai: peta[kunci].to_string(),
                sebab: "dijangka nombor positif".into(),
            })?;
        Ok(n as u32)
    }
}

// ─── Penggunaan ────────────────────────────────────────────────

fn jalankan_app() -> Result<(), AppError> {
    let laluan = Path::new("konfig.json");

    // Error dari konfigurasi auto-convert ke AppError::Konfigurasi
    let konfig = Konfigurasi::muat(laluan)?;

    println!("Server: {}:{}", konfig.host, konfig.port);
    println!("Workers: {}", konfig.pekerja);

    Ok(())
}

fn main() {
    match jalankan_app() {
        Ok(())  => println!("Berjaya!"),
        Err(AppError::Konfigurasi(KonfigurasError::FailTidakJumpai { laluan })) => {
            eprintln!("❌ Fail '{}' tidak dijumpai.", laluan.display());
            eprintln!("   Salin 'konfig.json.example' ke 'konfig.json' dan isi nilai.");
            std::process::exit(1);
        }
        Err(AppError::Konfigurasi(e)) => {
            eprintln!("❌ Masalah konfigurasi: {}", e);
            std::process::exit(1);
        }
        Err(e) => {
            eprintln!("❌ Ralat tidak dijangka: {}", e);
            std::process::exit(2);
        }
    }
}
```

---

# 📋 Rujukan Pantas

## Carta Keputusan

```
Nak handle error bagaimana?

 Fail → OK untuk program berhenti?
   YA  → unwrap() atau expect("konteks bermakna")
   TIDAK → teruskan...

 Boleh handle di sini?
   YA → match/if let dengan logik spesifik
   TIDAK → teruskan...

 Perlu tambah context?
   YA → .with_context(|| "...") (anyhow)
        atau .map_err(|e| MyError::dari(e))
   TIDAK → teruskan...

 Return type adalah Result/Option?
   YA → guna ?
   TIDAK → perlu ubah return type atau handle di sini
```

## Kaedah-kaedah Penting

```rust
// UNWRAP variants
.unwrap()                    // panic kalau None/Err
.expect("mesej")             // panic + mesej berguna
.unwrap_or(value)            // nilai default kalau None/Err
.unwrap_or_else(|| compute())// compute default
.unwrap_or_default()         // Default::default()

// TRANSFORM
.map(|v| ...)                // ubah Ok/Some
.map_err(|e| ...)            // ubah Err
.and_then(|v| ...)           // chain mungkin-fail
.or_else(|e| ...)            // fallback

// PROPAGATE
expr?                        // propagate kalau Err/None
.context("...")              // anyhow: tambah context
.with_context(|| "...")      // anyhow: lazy context

// CONVERT
.ok()                        // Result → Option (buang error)
.ok_or(err)                  // Option → Result
.ok_or_else(|| err)          // Option → Result (lazy)

// COLLECT
.collect::<Result<Vec<_>,_>>()  // Vec<Result> → Result<Vec>
.collect::<Option<Vec<_>>>()    // Vec<Option> → Option<Vec>
```

## Golden Rules

```
1. unwrap() dalam test = OK
2. unwrap() dalam prototype = OK (tandakan TODO)
3. unwrap() dalam production = BAHAYA (kecuali kes spesifik)
4. expect() > unwrap() bila terpaksa guna
5. ? dalam library code = sentiasa
6. ? dalam handler = dengan AppError yang impl IntoResponse
7. Error message yang baik = boleh debug tanpa source code
8. Tambah context pada setiap boundary (IO → Service → API)
9. Log error sebelum convert ke generic response
10. Fail fast dengan panic HANYA untuk programming errors/bugs
```

---

*unwrap() dan ? adalah alat yang berbeza untuk situasi berbeza.*
*Guna dengan sedar — tahu apa yang berlaku bila ralat muncul.*
*Error yang baik = debug yang mudah = production yang stabil.* 🦀
