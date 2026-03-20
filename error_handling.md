# ⚠️ Error Handling dalam Rust — Beginner to Advanced

> Panduan lengkap: Result<T,E>, unwrap/expect, ? operator, custom errors dan lebih.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Falsafah Error Handling Rust

```
Bahasa lain:              Rust:
  try { ... }               fn buat() -> Result<T, E>
  catch (Exception e) { }   // Error TERPAKSA dihandle!
  // Mudah skip handling!   // Compiler tidak bagi lulus kalau tidak!

Java/Python:              Rust:
  Exception boleh           Result<T, E> — explicit!
  "terbang" tanpa           Caller TAHU fungsi boleh gagal
  diketahui caller          Caller MESTI putuskan apa nak buat

Rust dua jenis error:
  1. Recoverable  → Result<T, E>  — boleh handle, terus jalan
  2. Unrecoverable → panic!()     — bug, program berhenti
```

---

## Peta Pembelajaran

```
Bab 1  → panic! & Unrecoverable Errors
Bab 2  → Result<T, E> Asas
Bab 3  → unwrap() & expect()
Bab 4  → match untuk Handle Result
Bab 5  → ? Operator — Cara Idiomatik
Bab 6  → Custom Error Types
Bab 7  → thiserror & anyhow
Bab 8  → Error Propagation & Chaining
Bab 9  → Pattern Lanjutan
Bab 10 → Mini Project: File Config Reader
```

---

# BAB 1: panic! & Unrecoverable Errors 💥

## Bila Guna panic!

```
panic! = "Sesuatu yang tidak patut berlaku, berlaku. Berhenti sekarang!"

Sesuai untuk:
  ✔ Bug dalam program (assertion failure)
  ✔ Keadaan yang "mustahil" secara logik
  ✔ Testing & prototyping (todo!, unimplemented!)
  ✔ Out-of-bounds access (array index salah)

TIDAK sesuai untuk:
  ✘ Input pengguna yang salah (guna Result)
  ✘ Fail tidak wujud (guna Result)
  ✘ Network error (guna Result)
  ✘ Mana-mana error yang boleh dijangka
```

```rust
fn main() {
    // panic! — langsung berhenti
    // panic!("Sesuatu yang sangat salah berlaku!");

    // Index out of bounds → auto panic
    let v = vec![1, 2, 3];
    // let x = v[10]; // ← PANIC: index out of bounds

    // Guna get() untuk elak panic
    match v.get(10) {
        Some(x) => println!("Ada: {}", x),
        None    => println!("Index tiada — OK, tidak panic"),
    }

    // assert! → panic kalau syarat salah
    let x = 5;
    assert!(x > 0, "x mesti positif, dapat {}", x);

    // assert_eq! → panic kalau tidak sama
    assert_eq!(2 + 2, 4, "matematik rosak!");

    // todo! → panic dengan mesej "not yet implemented"
    // todo!("Fungsi ini belum siap");

    // unimplemented! → sama seperti todo! tapi lebih spesifik
    // unimplemented!("Feature ini tidak akan dilaksanakan");

    // unreachable! → panic bila kod yang "mustahil dicapai" dicapai
    let arah = "utara";
    let darjah = match arah {
        "utara"  => 0,
        "timur"  => 90,
        "selatan"=> 180,
        "barat"  => 270,
        _        => unreachable!("Arah tidak dikenali: {}", arah),
    };
    println!("Darjah: {}", darjah);
}
```

---

## Stack Unwind vs Abort

```toml
# Cargo.toml — tukar panic behavior
[profile.release]
panic = "abort"  # lebih kecil binary, tiada stack unwind
# panic = "unwind"  # default — boleh tangkap dengan std::panic::catch_unwind
```

---

# BAB 2: Result<T, E> Asas 🎯

## Apa Itu Result?

```
Result<T, E> adalah enum dengan dua variant:

  ┌─────────────────────────────────────────────┐
  │  enum Result<T, E> {                        │
  │      Ok(T),   // berjaya — ada nilai T      │
  │      Err(E),  // gagal  — ada error E       │
  │  }                                          │
  └─────────────────────────────────────────────┘

  Ok(42)           ← berjaya, nilai 42
  Err("gagal!")    ← gagal, mesej ralat

  Macam kaunter servis:
    ✅ "Sila ambil nombor giliran 42" → Ok(42)
    ❌ "Sistem rosak" → Err("sistem rosak")
```

```rust
// Result dalam standard library
use std::fs;
use std::num::ParseIntError;

fn main() {
    // fs::read_to_string return Result<String, io::Error>
    let kandungan = fs::read_to_string("ada.txt");
    println!("{:?}", kandungan);

    // parse() return Result<T, ParseIntError>
    let nombor: Result<i32, ParseIntError> = "42".parse();
    println!("{:?}", nombor);   // Ok(42)

    let gagal: Result<i32, ParseIntError> = "abc".parse();
    println!("{:?}", gagal);    // Err(ParseIntError { ... })

    // Buat Result sendiri
    fn bahagi(a: f64, b: f64) -> Result<f64, String> {
        if b == 0.0 {
            Err("Tidak boleh bahagi dengan sifar!".to_string())
        } else {
            Ok(a / b)
        }
    }

    println!("{:?}", bahagi(10.0, 2.0));  // Ok(5.0)
    println!("{:?}", bahagi(10.0, 0.0));  // Err("Tidak boleh bahagi...")
}
```

---

## Result Methods yang Berguna

```rust
fn main() {
    let ok: Result<i32, &str>  = Ok(42);
    let err: Result<i32, &str> = Err("gagal");

    // Semak jenis
    println!("ok.is_ok():  {}", ok.is_ok());   // true
    println!("ok.is_err(): {}", ok.is_err());  // false
    println!("err.is_ok(): {}", err.is_ok());  // false

    // ok() — tukar ke Option (buang error info)
    println!("{:?}", ok.ok());    // Some(42)
    println!("{:?}", err.ok());   // None

    // map() — transform nilai Ok, biarkan Err
    let dua_kali = ok.map(|n| n * 2);
    println!("{:?}", dua_kali);   // Ok(84)

    let dua_kali_err = err.map(|n| n * 2);
    println!("{:?}", dua_kali_err); // Err("gagal") — tidak berubah

    // map_err() — transform nilai Err, biarkan Ok
    let err2 = err.map_err(|e| format!("Error: {}", e));
    println!("{:?}", err2);  // Err("Error: gagal")

    // and_then() — chaining (kalau Ok, buat sesuatu lagi)
    let hasil = ok
        .and_then(|n| if n > 0 { Ok(n * 10) } else { Err("negatif") });
    println!("{:?}", hasil); // Ok(420)

    // or_else() — kalau Err, cuba gantikan
    let recover = err.or_else(|_| Ok::<i32, &str>(0));
    println!("{:?}", recover); // Ok(0)

    // unwrap_or() — nilai default kalau Err
    println!("{}", ok.unwrap_or(0));  // 42
    println!("{}", err.unwrap_or(0)); // 0

    // unwrap_or_else() — compute default lazily
    println!("{}", err.unwrap_or_else(|e| {
        println!("Handling error: {}", e);
        -1
    })); // -1
}
```

---

## 🧠 Brain Teaser #1

Apa output kod ini?

```rust
fn main() {
    let a: Result<i32, &str> = Ok(5);
    let b: Result<i32, &str> = Err("oops");

    let x = a.map(|n| n * 2).unwrap_or(0);
    let y = b.map(|n| n * 2).unwrap_or(0);
    let z = b.map_err(|e| format!("ERR: {}", e)).unwrap_or(-1);

    println!("{} {} {}", x, y, z);
}
```

<details>
<summary>👀 Jawapan</summary>

Output: `10 0 -1`

- `a.map(|n| n*2)` → `Ok(10)`, `.unwrap_or(0)` → `10`
- `b.map(|n| n*2)` → `Err("oops")` (map tidak ubah Err), `.unwrap_or(0)` → `0`
- `b.map_err(...)` → `Err("ERR: oops")` (map_err ubah Err), `.unwrap_or(-1)` → `-1`

**Kunci:** `map()` hanya transform `Ok`, `map_err()` hanya transform `Err`. Kedua-duanya tidak sentuh yang satu lagi.
</details>

---

# BAB 3: unwrap() & expect() 🎲

## unwrap() — Cara Malas (Berbahaya!)

```rust
fn main() {
    let ok: Result<i32, &str> = Ok(42);
    let err: Result<i32, &str> = Err("gagal");

    // unwrap() — ambil nilai Ok, PANIC kalau Err
    let nilai = ok.unwrap();
    println!("Nilai: {}", nilai); // 42

    // unwrap() pada Err → PANIC!
    // let nilai2 = err.unwrap();
    // thread 'main' panicked at 'called `Result::unwrap()` on
    // an `Err` value: "gagal"'

    // unwrap() pada Option juga ada
    let some: Option<i32> = Some(42);
    let none: Option<i32> = None;

    println!("{}", some.unwrap()); // 42
    // none.unwrap(); // PANIC: called `Option::unwrap()` on a `None` value
}
```

---

## expect() — Lebih Baik dari unwrap()

```rust
fn main() {
    let ok: Result<i32, &str> = Ok(42);
    let err: Result<i32, &str> = Err("404");

    // expect() — sama seperti unwrap() tapi mesej panic lebih jelas
    let nilai = ok.expect("Sepatutnya ada nilai di sini");
    println!("{}", nilai); // 42

    // Kalau panic:
    // err.expect("Gagal baca konfigurasi");
    // → panicked at 'Gagal baca konfigurasi: "404"'
    // ↑ mesej kita + nilai error — lebih mudah debug!

    // Berbanding unwrap():
    // err.unwrap();
    // → panicked at 'called `Result::unwrap()` on an `Err` value: "404"'
    // ↑ mesej generik — susah debug kalau banyak unwrap dalam code

    // Option pun ada expect()
    let some: Option<i32> = Some(5);
    println!("{}", some.expect("Mestilah ada nilai!")); // 5
}
```

---

## Bila Guna unwrap() & expect()

```
❌ JANGAN guna dalam production code untuk error yang boleh berlaku
❌ JANGAN guna untuk user input, I/O, network
❌ JANGAN guna dalam library code

✔ BOLEH guna dalam:
   - Test code (#[test])
   - Prototyping (akan betulkan nanti)
   - Bila PASTI nilai adalah Ok (dengan komen kenapa)
   - main() yang simple

Kalau guna dalam production, guna expect() dengan mesej yang jelas:
   Bukan: .unwrap()
   Lebih baik: .expect("Gagal baca config — pastikan fail wujud")
```

```rust
fn main() {
    // ✔ OK — kita tahu ini pasti berjaya
    let regex = regex::Regex::new(r"^\d+$")
        .expect("Regex pattern statik ini mesti valid");

    // ✔ OK — dalam test
    #[cfg(test)]
    fn test_something() {
        let result = buat_sesuatu();
        assert_eq!(result.unwrap(), 42);
    }

    // ❌ Bahaya — user input boleh salah!
    // let n: i32 = input.parse().unwrap(); // panic kalau bukan nombor!

    // ✔ Betul — handle properly
    // let n: i32 = match input.parse() {
    //     Ok(n)  => n,
    //     Err(e) => return Err(e.into()),
    // };
}
```

---

## 🧠 Brain Teaser #2

Mana satu lebih baik dan kenapa?

```rust
// Pilihan A
fn baca_port(s: &str) -> u16 {
    s.parse().unwrap()
}

// Pilihan B
fn baca_port(s: &str) -> u16 {
    s.parse().expect("Port mesti nombor antara 0-65535")
}

// Pilihan C
fn baca_port(s: &str) -> Result<u16, String> {
    s.parse().map_err(|e| format!("Port tidak sah '{}': {}", s, e))
}
```

<details>
<summary>👀 Jawapan</summary>

**Pilihan C adalah terbaik** untuk production code.

- **A** — paling teruk: panic dengan mesej generik, caller tidak boleh handle
- **B** — lebih baik: mesej jelas bila panic, tapi masih panic!
- **C** — terbaik: return `Result`, caller **boleh pilih** nak handle atau panic

Pilihan C:
- Tidak panic (kecuali caller pilih `.unwrap()`)
- Mesej error yang informatif
- Caller boleh propagate dengan `?`
- Library-friendly (caller yang buat keputusan)

**B adalah OK** bila:
- Code dalaman yang DIJAMIN betul
- Contoh: parse IPv4 hardcoded dalam source
- Contoh: regex compile-time
</details>

---

# BAB 4: match untuk Handle Result 🎯

## match yang Proper

```rust
use std::fs;
use std::io;

fn main() {
    // Basic match
    match fs::read_to_string("config.txt") {
        Ok(kandungan) => {
            println!("Fail dibaca ({} bytes)", kandungan.len());
            proses_kandungan(&kandungan);
        }
        Err(e) => {
            eprintln!("Gagal baca fail: {}", e);
            // Guna nilai default atau terus
        }
    }

    // Match dengan pelbagai kes error
    match "abc".parse::<i32>() {
        Ok(n) => println!("Nombor: {}", n),
        Err(e) => match e.kind() {
            std::num::IntErrorKind::InvalidDigit => {
                println!("Bukan nombor!");
            }
            std::num::IntErrorKind::PosOverflow => {
                println!("Nombor terlalu besar!");
            }
            _ => println!("Parse error: {}", e),
        }
    }

    // Nested match untuk dua Result serentak
    let a: Result<i32, &str> = Ok(10);
    let b: Result<i32, &str> = Ok(20);

    match (a, b) {
        (Ok(x), Ok(y))    => println!("{} + {} = {}", x, y, x + y),
        (Err(e), _)       => println!("a gagal: {}", e),
        (_, Err(e))       => println!("b gagal: {}", e),
    }
}

fn proses_kandungan(s: &str) {
    println!("Proses: {} aksara", s.len());
}
```

---

## if let & while let dengan Result

```rust
fn main() {
    // if let — untuk satu kes yang penting
    let hasil: Result<i32, &str> = Ok(42);

    if let Ok(n) = hasil {
        println!("Berjaya: {}", n);
    }

    // if let dengan else
    let parse_hasil = "abc".parse::<i32>();
    if let Ok(n) = parse_hasil {
        println!("Nombor: {}", n);
    } else {
        println!("Bukan nombor valid");
    }

    // let else — early return pattern (Rust 1.65+)
    fn proses(input: &str) -> Result<String, String> {
        let Ok(n) = input.trim().parse::<i32>() else {
            return Err(format!("'{}' bukan nombor", input));
        };
        Ok(format!("Dua kali ganda: {}", n * 2))
    }

    println!("{:?}", proses("21"));    // Ok("Dua kali ganda: 42")
    println!("{:?}", proses("abc"));   // Err("'abc' bukan nombor")

    // while let untuk drain iterator yang return Result
    let data = vec!["1", "2", "abc", "4"];
    let mut iter = data.iter();
    while let Some(&s) = iter.next() {
        if let Ok(n) = s.parse::<i32>() {
            println!("Parse OK: {}", n);
        } else {
            println!("Skip '{}' — bukan nombor", s);
        }
    }
}
```

---

## Collect Vec<Result<T, E>> → Result<Vec<T>, E>

```rust
fn main() {
    // Collect — fail terus bila ada satu Err
    let strings = vec!["1", "2", "3", "4"];
    let nombor: Result<Vec<i32>, _> = strings.iter()
        .map(|s| s.parse::<i32>())
        .collect();
    println!("{:?}", nombor); // Ok([1, 2, 3, 4])

    // Ada satu yang gagal → Err terus
    let strings2 = vec!["1", "abc", "3"];
    let nombor2: Result<Vec<i32>, _> = strings2.iter()
        .map(|s| s.parse::<i32>())
        .collect();
    println!("{:?}", nombor2); // Err(ParseIntError { ... })

    // filter_map — skip error, ambil yang berjaya sahaja
    let strings3 = vec!["1", "abc", "3", "xyz", "5"];
    let nombor3: Vec<i32> = strings3.iter()
        .filter_map(|s| s.parse::<i32>().ok())
        .collect();
    println!("{:?}", nombor3); // [1, 3, 5]

    // partition — pisahkan Ok dan Err
    let (berjaya, gagal): (Vec<_>, Vec<_>) = strings3.iter()
        .map(|s| s.parse::<i32>())
        .partition(Result::is_ok);

    let nilai: Vec<i32> = berjaya.into_iter().map(Result::unwrap).collect();
    let ralat: Vec<_>   = gagal.into_iter().map(Result::unwrap_err).collect();
    println!("OK:  {:?}", nilai); // [1, 3, 5]
    println!("Err: {:?}", ralat); // [parse errors]
}
```

---

# BAB 5: ? Operator — Cara Idiomatik ✨

## Apa Itu ? Operator?

```
? operator = "kalau Err, return Err sekarang. Kalau Ok, ambil nilai."

Tanpa ?:                        Dengan ?:
  match fungsi() {                let nilai = fungsi()?;
      Ok(v)  => v,
      Err(e) => return Err(e),
  }

? juga lakukan KONVERSI error automatik (From trait)!
```

```rust
use std::fs;
use std::io;
use std::num::ParseIntError;

// Fungsi yang return Result — perlu ?
fn baca_nombor_dari_fail(path: &str) -> Result<i32, io::Error> {
    let kandungan = fs::read_to_string(path)?; // io::Error jika gagal
    // ? = kalau Err, return Err(e) terus

    // Tapi ini tidak compile — ParseIntError != io::Error!
    // let n: i32 = kandungan.trim().parse()?;

    Ok(0) // simplified
}

// Dengan custom error yang wrap kedua-dua
#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Parse(ParseIntError),
}

impl From<io::Error> for AppError {
    fn from(e: io::Error) -> Self { AppError::Io(e) }
}

impl From<ParseIntError> for AppError {
    fn from(e: ParseIntError) -> Self { AppError::Parse(e) }
}

fn baca_nombor(path: &str) -> Result<i32, AppError> {
    let kandungan = fs::read_to_string(path)?;  // io::Error → AppError::Io
    let n: i32    = kandungan.trim().parse()?;   // ParseIntError → AppError::Parse
    Ok(n * 2)
}
```

---

## ? dalam Pelbagai Konteks

```rust
use std::io;
use std::fs;

// Rantai operasi dengan ?
fn proses_konfigurasi(path: &str) -> Result<Config, AppError> {
    let teks     = fs::read_to_string(path)?;      // io error
    let bersih   = teks.trim().to_string();
    let nilai    = bersih.parse::<u32>()?;          // parse error
    let konfigurasi = Config::dari_nilai(nilai)?;   // config error

    Ok(konfigurasi)
}

// Dalam main() — Result boleh return!
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let kandungan = fs::read_to_string("test.txt")?;
    println!("{}", kandungan);

    let n: i32 = "42".parse()?;
    println!("Nombor: {}", n);

    Ok(()) // ← main kena return Ok(()) jika guna Result
}
```

---

## ? dengan Option

```rust
fn main() {
    // ? juga berfungsi dengan Option dalam fungsi yang return Option!
    fn cari_elemen(v: &[i32], sasaran: i32) -> Option<usize> {
        let pos = v.iter().position(|&x| x == sasaran)?;
        // ? pada Option: kalau None, return None terus

        Some(pos * 2) // transformasi pada indeks
    }

    let v = vec![10, 20, 30, 40, 50];
    println!("{:?}", cari_elemen(&v, 30));  // Some(4) ← indeks 2 * 2
    println!("{:?}", cari_elemen(&v, 99));  // None

    // Tukar Option ke Result bila perlu guna ?
    fn dengan_result(opt: Option<i32>) -> Result<i32, String> {
        let n = opt.ok_or_else(|| "Nilai tiada".to_string())?;
        Ok(n * 2)
    }

    println!("{:?}", dengan_result(Some(5)));   // Ok(10)
    println!("{:?}", dengan_result(None));       // Err("Nilai tiada")
}
```

---

## 🧠 Brain Teaser #3

Apakah perbezaan antara dua fungsi ini?

```rust
use std::fs;

// Versi A
fn baca_a(path: &str) -> Result<String, std::io::Error> {
    let isi = fs::read_to_string(path)?;
    Ok(isi.to_uppercase())
}

// Versi B
fn baca_b(path: &str) -> Result<String, std::io::Error> {
    match fs::read_to_string(path) {
        Ok(isi)  => Ok(isi.to_uppercase()),
        Err(e)   => Err(e),
    }
}
```

<details>
<summary>👀 Jawapan</summary>

**Output yang sama** — kedua-dua fungsi buat perkara yang sama persis!

`?` adalah syntactic sugar untuk `match { Ok(v) => v, Err(e) => return Err(e) }`.

**Versi A lebih baik** kerana:
- Lebih ringkas dan mudah baca
- Kurang boilerplate
- Jelas tunjukkan "happy path"
- Lebih idiomatik Rust

**Tambahan:** `?` juga boleh lakukan konversi error melalui `From` trait, yang versi B tidak buat secara automatik. Jadi kalau return type berbeza dari error type, `?` lebih berkuasa.
</details>

---

# BAB 6: Custom Error Types 🛠️

## Error Trait

```rust
use std::fmt;
use std::error::Error;

// Implement Error trait untuk custom error
#[derive(Debug)]
enum AppError {
    TidakJumpai(String),
    ParseGagal(String),
    PermissiDitolak(String),
    Rangkaian { kod: u16, mesej: String },
}

// Display — mesej mesra pengguna
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::TidakJumpai(nama)    =>
                write!(f, "Rekod '{}' tidak ditemui", nama),
            AppError::ParseGagal(input)    =>
                write!(f, "Gagal parse input: '{}'", input),
            AppError::PermissiDitolak(ops) =>
                write!(f, "Permisi ditolak untuk operasi: {}", ops),
            AppError::Rangkaian { kod, mesej } =>
                write!(f, "Ralat rangkaian {}: {}", kod, mesej),
        }
    }
}

// Error trait — standard untuk semua error types
impl Error for AppError {
    // source() — ralat yang menyebabkan ralat ini (optional)
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None  // tiada underlying error dalam contoh ini
    }
}

fn cari_pengguna(id: u32) -> Result<String, AppError> {
    match id {
        1 => Ok("Ali Ahmad".to_string()),
        2 => Ok("Siti Hawa".to_string()),
        _ => Err(AppError::TidakJumpai(format!("ID {}", id))),
    }
}

fn main() {
    match cari_pengguna(3) {
        Ok(nama) => println!("Jumpa: {}", nama),
        Err(e)   => {
            eprintln!("Error: {}", e);
            eprintln!("Debug: {:?}", e);
        }
    }

    // Guna dalam ? chain
    fn proses() -> Result<(), AppError> {
        let nama = cari_pengguna(1)?;
        println!("Proses pengguna: {}", nama);
        Ok(())
    }

    proses().unwrap();
}
```

---

## From Trait untuk Error Conversion

```rust
use std::fs;
use std::num::ParseIntError;
use std::io;
use std::fmt;

#[derive(Debug)]
enum ConfigError {
    FailTidakWujud(io::Error),
    NilaiBukan Nombor(ParseIntError),
    NilaiLuarJulat(i32),
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::FailTidakWujud(e)    => write!(f, "Fail config tidak wujud: {}", e),
            ConfigError::NilaiBuiltNombor(e) => write!(f, "Nilai bukan nombor: {}", e),
            ConfigError::NilaiLuarJulat(n)   => write!(f, "Nilai {} luar julat 1-100", n),
        }
    }
}

// Implement From untuk auto-konversi dengan ?
impl From<io::Error> for ConfigError {
    fn from(e: io::Error) -> Self {
        ConfigError::FailTidakWujud(e)
    }
}

impl From<ParseIntError> for ConfigError {
    fn from(e: ParseIntError) -> Self {
        ConfigError::NilaiBukanNombor(e)
    }
}

fn muat_config(path: &str) -> Result<i32, ConfigError> {
    let teks  = fs::read_to_string(path)?;  // io::Error → ConfigError::FailTidakWujud
    let nilai = teks.trim().parse::<i32>()?; // ParseIntError → ConfigError::NilaiBukanNombor

    if nilai < 1 || nilai > 100 {
        return Err(ConfigError::NilaiLuarJulat(nilai));
    }

    Ok(nilai)
}

fn main() {
    match muat_config("config.txt") {
        Ok(n)                              => println!("Config: {}", n),
        Err(ConfigError::FailTidakWujud(e)) => println!("Fail: {}", e),
        Err(ConfigError::NilaiBukanNombor(e)) => println!("Parse: {}", e),
        Err(ConfigError::NilaiLuarJulat(n)) => println!("Julat: {} luar 1-100", n),
    }
}
```

---

## Box<dyn Error> — Cara Paling Mudah

```rust
use std::error::Error;

// Box<dyn Error> — boleh simpan APA-APA error!
fn main() -> Result<(), Box<dyn Error>> {
    // io::Error
    let _kandungan = std::fs::read_to_string("test.txt")?;

    // ParseIntError
    let _n: i32 = "42".parse()?;

    // Custom error
    fn custom() -> Result<(), Box<dyn Error>> {
        Err("Ralat custom!".into())  // &str boleh jadi Box<dyn Error>
    }
    custom()?;

    Ok(())
}

// Kebaikan:    Mudah, cepat, terima semua error
// Keburukan:   Caller tidak tahu jenis error spesifik
//              Tidak boleh match pada jenis error
//              Sedikit overhead (heap allocation)
```

---

# BAB 7: thiserror & anyhow 📦

## thiserror — Custom Errors Tanpa Boilerplate

```toml
[dependencies]
thiserror = "1"
```

```rust
use thiserror::Error;
use std::io;
use std::num::ParseIntError;

// thiserror — derive macro yang generate Display, Error, From automatik!
#[derive(Debug, Error)]
enum AppError {
    #[error("Fail tidak wujud: {0}")]
    Fail(#[from] io::Error),

    #[error("Nilai bukan nombor: {0}")]
    Parse(#[from] ParseIntError),

    #[error("Pengguna '{nama}' tidak dijumpai")]
    TidakJumpai { nama: String },

    #[error("Nilai {nilai} luar julat [{min}, {max}]")]
    LuarJulat { nilai: i32, min: i32, max: i32 },

    #[error("Ralat tidak dijangka")]
    Lain(#[from] Box<dyn std::error::Error + Send + Sync>),
}

fn proses_fail(path: &str) -> Result<i32, AppError> {
    let teks  = std::fs::read_to_string(path)?;   // auto: io::Error → AppError::Fail
    let nilai = teks.trim().parse::<i32>()?;       // auto: ParseIntError → AppError::Parse

    if nilai < 0 || nilai > 1000 {
        return Err(AppError::LuarJulat { nilai, min: 0, max: 1000 });
    }

    Ok(nilai)
}

fn main() {
    match proses_fail("data.txt") {
        Ok(n)                    => println!("Nilai: {}", n),
        Err(AppError::Fail(e))   => println!("I/O: {}", e),
        Err(AppError::Parse(e))  => println!("Parse: {}", e),
        Err(AppError::LuarJulat { nilai, min, max }) =>
            println!("{} luar julat [{}, {}]", nilai, min, max),
        Err(e) => println!("Lain: {}", e),
    }
}
```

---

## anyhow — Error Handling Mudah

```toml
[dependencies]
anyhow = "1"
```

```rust
use anyhow::{Context, Result, bail, ensure, anyhow};

// anyhow::Result = Result<T, anyhow::Error>
// Boleh simpan MANA-MANA error + context!

fn baca_fail(path: &str) -> Result<String> {
    std::fs::read_to_string(path)
        .with_context(|| format!("Gagal baca fail '{}'", path))
}

fn parse_nombor(s: &str) -> Result<i32> {
    s.parse::<i32>()
        .with_context(|| format!("Gagal parse '{}' sebagai integer", s))
}

fn proses(path: &str) -> Result<i32> {
    let kandungan = baca_fail(path)?;
    let nilai     = parse_nombor(kandungan.trim())?;

    // bail! — return Err dengan mesej
    if nilai < 0 {
        bail!("Nilai mesti positif, dapat {}", nilai);
    }

    // ensure! — assertion yang return Err
    ensure!(nilai <= 1000, "Nilai {} terlalu besar (had: 1000)", nilai);

    // anyhow! — buat error inline
    if nilai == 42 {
        return Err(anyhow!("42 adalah nombor larangan!"));
    }

    Ok(nilai * 2)
}

fn main() {
    match proses("nilai.txt") {
        Ok(n)  => println!("Hasil: {}", n),
        Err(e) => {
            println!("Error: {}", e);
            // anyhow simpan chain error — lihat semua
            println!("Punca:\n{:?}", e);
        }
    }
}

// Bila guna apa:
// thiserror → library code (expose error types yang jelas kepada caller)
// anyhow    → application code (tidak perlu caller match error jenis)
```

---

## 🧠 Brain Teaser #4

Bilakah kita patut guna `thiserror` vs `anyhow`?

<details>
<summary>👀 Jawapan</summary>

```
thiserror — untuk LIBRARY / crate yang akan digunakan orang lain:
  ✔ Caller boleh match error jenis yang spesifik
  ✔ API yang jelas: "fungsi ini boleh gagal dengan X atau Y"
  ✔ Type-safe error handling
  Contoh: sqlx, reqwest, tokio

anyhow — untuk APPLICATION code (binary, main program):
  ✔ Cepat & mudah — tidak perlu define error types
  ✔ Context berguna untuk debugging
  ✔ Boleh campurkan error dari pelbagai library
  Contoh: main.rs, CLI tools, scripts

Panduan ringkas:
  "Adakah caller perlu tahu JENIS error?"
    Ya  → thiserror (atau custom Error enum)
    Tidak → anyhow (atau Box<dyn Error>)
```
</details>

---

# BAB 8: Error Propagation & Chaining 🔗

## Error Context

```rust
use std::fmt;

#[derive(Debug)]
struct ErrorDenganKonteks {
    mesej:  String,
    punca:  Option<Box<dyn std::error::Error + Send + Sync>>,
}

impl ErrorDenganKonteks {
    fn baru(mesej: &str) -> Self {
        ErrorDenganKonteks { mesej: mesej.into(), punca: None }
    }

    fn dengan_punca<E: std::error::Error + Send + Sync + 'static>(
        mesej: &str, punca: E
    ) -> Self {
        ErrorDenganKonteks {
            mesej: mesej.into(),
            punca: Some(Box::new(punca)),
        }
    }
}

impl fmt::Display for ErrorDenganKonteks {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.mesej)?;
        if let Some(punca) = &self.punca {
            write!(f, "\n  → punca: {}", punca)?;
        }
        Ok(())
    }
}

impl std::error::Error for ErrorDenganKonteks {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        self.punca.as_ref().map(|e| e.as_ref() as _)
    }
}

fn main() {
    use std::io;

    fn operasi_dalam() -> Result<(), io::Error> {
        Err(io::Error::new(io::ErrorKind::NotFound, "fail.txt tidak wujud"))
    }

    fn operasi_luar() -> Result<(), ErrorDenganKonteks> {
        operasi_dalam().map_err(|e| {
            ErrorDenganKonteks::dengan_punca("Gagal muat konfigurasi", e)
        })
    }

    match operasi_luar() {
        Ok(())  => println!("OK"),
        Err(e)  => {
            println!("Error: {}", e);
            // Traverse error chain
            let mut punca: &dyn std::error::Error = &e;
            while let Some(p) = punca.source() {
                println!("  Disebabkan: {}", p);
                punca = p;
            }
        }
    }
}
```

---

## Handling Errors dalam Iterator

```rust
fn main() {
    let data = vec!["1", "2", "tiga", "4", "lima", "6"];

    // Cara 1: Kumpul semua, gagal bila ada error
    let hasil: Result<Vec<i32>, _> = data.iter()
        .map(|s| s.parse::<i32>())
        .collect();
    println!("Cara 1: {:?}", hasil); // Err pada "tiga"

    // Cara 2: Skip error, ambil yang berjaya
    let berjaya: Vec<i32> = data.iter()
        .filter_map(|s| s.parse::<i32>().ok())
        .collect();
    println!("Cara 2: {:?}", berjaya); // [1, 2, 4, 6]

    // Cara 3: Pisahkan OK dan Err
    let (ok_vec, err_vec): (Vec<_>, Vec<_>) = data.iter()
        .map(|s| s.parse::<i32>().map_err(|e| (s, e)))
        .partition(Result::is_ok);

    let nilai: Vec<i32>  = ok_vec.into_iter().map(|r| r.unwrap()).collect();
    let ralat: Vec<_>    = err_vec.into_iter().map(|r| r.unwrap_err()).collect();

    println!("OK:  {:?}", nilai);
    println!("ERR: {:?}", ralat.iter().map(|(s, _)| s).collect::<Vec<_>>());

    // Cara 4: Proses dengan side effects untuk error
    for s in &data {
        match s.parse::<i32>() {
            Ok(n)  => println!("  ✔ {} → {}", s, n),
            Err(e) => eprintln!("  ✘ '{}' gagal: {}", s, e),
        }
    }
}
```

---

# BAB 9: Pattern Lanjutan 🎯

## Never Type (!) dalam Error Handling

```rust
// ! — type untuk "tidak pernah return"
// Berguna untuk fungsi yang sentiasa panic atau loop selamanya

fn sentiasa_gagal() -> Result<i32, !> {
    Ok(42)  // sentiasa Ok, never Err
}

// Dalam match — ! boleh "jadi apa-apa type"
fn muat_atau_panic(path: &str) -> String {
    std::fs::read_to_string(path).unwrap_or_else(|e| {
        panic!("Fail kritikal tidak wujud: {}", e)
        // panic! return ! yang coerce ke String
    })
}
```

---

## Result dalam Struct Fields

```rust
use std::fmt;

// Simpan error state dalam struct
struct Sambungan {
    host:   String,
    port:   u16,
    status: Result<(), String>,
}

impl Sambungan {
    fn cuba_sambung(host: &str, port: u16) -> Self {
        let status = if host == "localhost" {
            Ok(())
        } else {
            Err(format!("Tidak dapat sambung ke {}:{}", host, port))
        };

        Sambungan { host: host.into(), port, status }
    }

    fn berjaya(&self) -> bool { self.status.is_ok() }

    fn hantar(&self, mesej: &str) -> Result<(), &str> {
        match &self.status {
            Ok(())   => {
                println!("Hantar '{}' ke {}:{}", mesej, self.host, self.port);
                Ok(())
            }
            Err(e) => Err(e.as_str()),
        }
    }
}

fn main() {
    let s1 = Sambungan::cuba_sambung("localhost", 8080);
    let s2 = Sambungan::cuba_sambung("jauh.server.com", 8080);

    println!("s1 OK? {}", s1.berjaya()); // true
    println!("s2 OK? {}", s2.berjaya()); // false

    match s1.hantar("Hello!") {
        Ok(())  => println!("Berjaya hantar"),
        Err(e)  => println!("Gagal: {}", e),
    }

    match s2.hantar("Hello!") {
        Ok(())  => println!("Berjaya hantar"),
        Err(e)  => println!("Gagal: {}", e),
    }
}
```

---

## Retry Pattern dengan Result

```rust
use std::time::Duration;
use std::thread;

fn cuba_operasi(percubaan: u32) -> Result<String, String> {
    if percubaan < 3 {
        Err(format!("Gagal percubaan #{}", percubaan))
    } else {
        Ok(format!("Berjaya pada percubaan #{}", percubaan))
    }
}

fn dengan_retry<F, T, E>(
    mut fungsi: F,
    maks_cuba: u32,
    tunggu_ms: u64,
) -> Result<T, E>
where
    F:   FnMut(u32) -> Result<T, E>,
    E:   std::fmt::Debug,
{
    let mut cuba = 1;
    loop {
        match fungsi(cuba) {
            Ok(nilai) => {
                println!("✔ Berjaya selepas {} percubaan", cuba);
                return Ok(nilai);
            }
            Err(e) if cuba < maks_cuba => {
                println!("✘ Percubaan {} gagal: {:?} — cuba lagi...", cuba, e);
                thread::sleep(Duration::from_millis(tunggu_ms));
                cuba += 1;
            }
            Err(e) => {
                println!("✘ Semua {} percubaan gagal", maks_cuba);
                return Err(e);
            }
        }
    }
}

fn main() {
    match dengan_retry(cuba_operasi, 5, 100) {
        Ok(hasil)  => println!("Hasil: {}", hasil),
        Err(e)     => println!("Gagal: {:?}", e),
    }
}
```

---

## 🧠 Brain Teaser #5 (Advanced)

Tulis fungsi `validate_pengguna` yang check:
1. Nama tidak boleh kosong
2. Umur antara 18-100
3. Email mesti ada '@'

Return `Result<Pengguna, Vec<String>>` — kumpul SEMUA ralat, bukan berhenti pada yang pertama.

<details>
<summary>👀 Jawapan</summary>

```rust
#[derive(Debug)]
struct Pengguna {
    nama:  String,
    umur:  u8,
    email: String,
}

fn validate_pengguna(
    nama: &str, umur: u8, email: &str
) -> Result<Pengguna, Vec<String>> {
    let mut ralat = Vec::new();

    if nama.trim().is_empty() {
        ralat.push("Nama tidak boleh kosong".to_string());
    }

    if umur < 18 {
        ralat.push(format!("Umur {} terlalu muda (min 18)", umur));
    } else if umur > 100 {
        ralat.push(format!("Umur {} tidak munasabah (max 100)", umur));
    }

    if !email.contains('@') {
        ralat.push(format!("Email '{}' tidak sah (tiada '@')", email));
    }

    if ralat.is_empty() {
        Ok(Pengguna {
            nama:  nama.trim().to_string(),
            umur,
            email: email.to_string(),
        })
    } else {
        Err(ralat)
    }
}

fn main() {
    match validate_pengguna("", 15, "bukan-email") {
        Ok(p)         => println!("Sah: {:?}", p),
        Err(ralat) => {
            println!("Terdapat {} ralat:", ralat.len());
            for r in &ralat {
                println!("  ✘ {}", r);
            }
        }
    }

    match validate_pengguna("Ali", 25, "ali@email.com") {
        Ok(p)     => println!("Sah: {:?}", p),
        Err(r)    => println!("Ralat: {:?}", r),
    }
}
```
</details>

---

# BAB 10: Mini Project — File Config Reader 📄

```rust
use std::collections::HashMap;
use std::fmt;
use std::fs;
use std::num::ParseIntError;
use std::path::Path;

// ─── Error Types ──────────────────────────────────────────────

#[derive(Debug)]
enum ConfigError {
    FailTidakWujud { path: String },
    FormatTidakSah { baris: usize, kandungan: String },
    NilaiTidakSah  { kunci: String, nilai: String, jangka: String },
    KunciTidakWujud(String),
    IoError(std::io::Error),
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::FailTidakWujud { path } =>
                write!(f, "Fail config '{}' tidak wujud", path),
            ConfigError::FormatTidakSah { baris, kandungan } =>
                write!(f, "Format tidak sah pada baris {}: '{}'", baris, kandungan),
            ConfigError::NilaiTidakSah { kunci, nilai, jangka } =>
                write!(f, "Nilai tidak sah untuk '{}': '{}' (jangka: {})", kunci, nilai, jangka),
            ConfigError::KunciTidakWujud(kunci) =>
                write!(f, "Kunci '{}' tidak ditemui dalam config", kunci),
            ConfigError::IoError(e) =>
                write!(f, "I/O error: {}", e),
        }
    }
}

impl std::error::Error for ConfigError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            ConfigError::IoError(e) => Some(e),
            _                       => None,
        }
    }
}

impl From<std::io::Error> for ConfigError {
    fn from(e: std::io::Error) -> Self { ConfigError::IoError(e) }
}

// ─── Config Reader ────────────────────────────────────────────

struct ConfigReader {
    data: HashMap<String, String>,
}

impl ConfigReader {
    // Muat dari fail
    fn dari_fail(path: &str) -> Result<Self, ConfigError> {
        if !Path::new(path).exists() {
            return Err(ConfigError::FailTidakWujud { path: path.to_string() });
        }

        let kandungan = fs::read_to_string(path)?; // io::Error → ConfigError::IoError
        Self::dari_teks(&kandungan)
    }

    // Parse dari string (berguna untuk testing!)
    fn dari_teks(teks: &str) -> Result<Self, ConfigError> {
        let mut data = HashMap::new();

        for (no_baris, baris) in teks.lines().enumerate() {
            let baris = baris.trim();

            // Skip baris kosong dan komen
            if baris.is_empty() || baris.starts_with('#') {
                continue;
            }

            // Parse KEY = VALUE
            let mut bahagian = baris.splitn(2, '=');
            let kunci = match bahagian.next() {
                Some(k) if !k.trim().is_empty() => k.trim().to_string(),
                _ => return Err(ConfigError::FormatTidakSah {
                    baris:     no_baris + 1,
                    kandungan: baris.to_string(),
                }),
            };

            let nilai = match bahagian.next() {
                Some(v) => v.trim().to_string(),
                None    => return Err(ConfigError::FormatTidakSah {
                    baris:     no_baris + 1,
                    kandungan: baris.to_string(),
                }),
            };

            data.insert(kunci, nilai);
        }

        Ok(ConfigReader { data })
    }

    // Ambil nilai sebagai &str
    fn dapatkan(&self, kunci: &str) -> Result<&str, ConfigError> {
        self.data
            .get(kunci)
            .map(String::as_str)
            .ok_or_else(|| ConfigError::KunciTidakWujud(kunci.to_string()))
    }

    // Ambil nilai sebagai String
    fn dapatkan_string(&self, kunci: &str) -> Result<String, ConfigError> {
        Ok(self.dapatkan(kunci)?.to_string())
    }

    // Ambil nilai sebagai u16 (untuk port)
    fn dapatkan_u16(&self, kunci: &str) -> Result<u16, ConfigError> {
        let nilai = self.dapatkan(kunci)?;
        nilai.parse::<u16>().map_err(|_| ConfigError::NilaiTidakSah {
            kunci:  kunci.to_string(),
            nilai:  nilai.to_string(),
            jangka: "nombor 0-65535".to_string(),
        })
    }

    // Ambil nilai sebagai bool
    fn dapatkan_bool(&self, kunci: &str) -> Result<bool, ConfigError> {
        let nilai = self.dapatkan(kunci)?;
        match nilai.to_lowercase().as_str() {
            "true" | "yes" | "1"  => Ok(true),
            "false" | "no" | "0"  => Ok(false),
            _ => Err(ConfigError::NilaiTidakSah {
                kunci:  kunci.to_string(),
                nilai:  nilai.to_string(),
                jangka: "true/false/yes/no/1/0".to_string(),
            }),
        }
    }

    // Ambil dengan nilai lalai
    fn dapatkan_atau(&self, kunci: &str, lalai: &str) -> String {
        self.dapatkan(kunci).unwrap_or(lalai).to_string()
    }

    fn dapatkan_u16_atau(&self, kunci: &str, lalai: u16) -> u16 {
        self.dapatkan_u16(kunci).unwrap_or(lalai)
    }

    fn dapatkan_bool_atau(&self, kunci: &str, lalai: bool) -> bool {
        self.dapatkan_bool(kunci).unwrap_or(lalai)
    }

    fn papar(&self) {
        println!("Config ({} entri):", self.data.len());
        let mut kunci: Vec<&String> = self.data.keys().collect();
        kunci.sort();
        for k in kunci {
            println!("  {:<25} = {}", k, self.data[k]);
        }
    }
}

// ─── Main ──────────────────────────────────────────────────────

fn main() {
    // Config dalam string (untuk demo — dalam production load dari fail)
    let config_teks = r#"
# Konfigurasi Aplikasi KADA Mobile API
# =====================================

app_nama    = KADA Mobile API
app_versi   = 2.1.0
app_port    = 8080
app_debug   = false
app_workers = 4

# Database
db_host     = localhost
db_port     = 5432
db_nama     = kada_production
db_ssl      = true
db_timeout  = 30

# Cache
cache_aktif = true
cache_ttl   = 300

# Baris yang akan menyebabkan error (disabled dengan #):
# port_salah = 99999
# bool_salah = maybe
    "#;

    println!("=== Muat Konfigurasi ===\n");

    // Parse config
    let config = match ConfigReader::dari_teks(config_teks) {
        Ok(c)  => {
            println!("✔ Config berjaya dimuatkan\n");
            c
        }
        Err(e) => {
            eprintln!("✘ Gagal muat config: {}", e);
            std::process::exit(1);
        }
    };

    config.papar();

    println!("\n=== Akses Nilai ===\n");

    // Akses nilai dengan Result handling
    match config.dapatkan("app_nama") {
        Ok(nama)  => println!("✔ Nama App:  {}", nama),
        Err(e)    => println!("✘ Error: {}", e),
    }

    match config.dapatkan_u16("app_port") {
        Ok(port)  => println!("✔ Port:      {}", port),
        Err(e)    => println!("✘ Error: {}", e),
    }

    match config.dapatkan_bool("app_debug") {
        Ok(debug) => println!("✔ Debug:     {}", debug),
        Err(e)    => println!("✘ Error: {}", e),
    }

    // Kunci yang tidak wujud
    match config.dapatkan("kunci_tiada") {
        Ok(v)     => println!("✔ Jumpa: {}", v),
        Err(e)    => println!("✘ {}", e),
    }

    println!("\n=== Nilai Dengan Default ===\n");

    // Guna unwrap_or / default values
    println!("Host DB:   {}", config.dapatkan_atau("db_host", "localhost"));
    println!("Port DB:   {}", config.dapatkan_u16_atau("db_port", 5432));
    println!("Kunci X:   {}", config.dapatkan_atau("kunci_tidak_ada", "nilai_lalai"));

    println!("\n=== Test Error Cases ===\n");

    // Buat config dengan error
    let config_rosak = "baris_tanpa_equal_sign";
    match ConfigReader::dari_teks(config_rosak) {
        Ok(_)    => println!("OK (tidak dijangka)"),
        Err(e)   => println!("✔ Error dijangka: {}", e),
    }

    // Nilai tidak sah
    let config_nilai_salah = "port = abc_bukan_nombor";
    match ConfigReader::dari_teks(config_nilai_salah) {
        Ok(c)  => match c.dapatkan_u16("port") {
            Ok(p)  => println!("Port: {}", p),
            Err(e) => println!("✔ Error dijangka: {}", e),
        },
        Err(e) => println!("✔ Error: {}", e),
    }

    println!("\n=== Selesai ===");
}
```

---

# 📋 Rujukan Pantas — Error Handling Cheat Sheet

## Hierarki Keputusan

```
Error boleh berlaku?
  ├── Boleh recovery?
  │     ├── Ya  → Result<T, E>
  │     └── Tidak → panic! / unreachable!
  │
  └── User input / I/O / Network?
        ├── Ya  → MESTI Result<T, E>
        └── Tidak → mungkin panic OK untuk bugs
```

## Cara Handle Result

```rust
// 1. match — paling eksplisit
match result {
    Ok(v)  => use(v),
    Err(e) => handle(e),
}

// 2. if let — bila hanya kes Ok penting
if let Ok(v) = result { use(v) }

// 3. ? — propagate ke caller (paling idiomatik)
let v = result?;

// 4. unwrap() — PANIC kalau Err (testing/prototyping sahaja)
let v = result.unwrap();

// 5. expect() — PANIC dengan mesej (lebih baik dari unwrap)
let v = result.expect("Mesej yang membantu");

// 6. unwrap_or() — nilai default kalau Err
let v = result.unwrap_or(default_val);

// 7. unwrap_or_else() — compute default lazily
let v = result.unwrap_or_else(|e| compute_default(e));
```

## Result Methods

```rust
result.is_ok()              // bool
result.is_err()             // bool
result.ok()                 // Option<T> (buang Err)
result.err()                // Option<E> (buang Ok)
result.map(|v| ...)         // transform Ok
result.map_err(|e| ...)     // transform Err
result.and_then(|v| ...)    // chain (flatMap)
result.or_else(|e| ...)     // recover dari Err
result.unwrap_or(default)   // T atau default
result.unwrap_or_else(f)    // T atau f(e)
```

## Error Traits

```rust
// Minimum untuk custom error
impl std::fmt::Display for MyError { ... }
impl std::error::Error for MyError { }

// Untuk ? conversion
impl From<OtherError> for MyError {
    fn from(e: OtherError) -> Self { ... }
}
```

## Crates yang Berguna

```
thiserror  → derive macro untuk Error, Display, From
anyhow     → application error (context, chain)
color-eyre → seperti anyhow tapi output cantik
miette     → diagnostic errors dengan source spans
snafu      → complex error hierarki
```

## Golden Rules

```
✔ Guna Result untuk semua yang boleh gagal
✔ Guna ? untuk propagate (bukan match yang panjang)
✔ Guna thiserror untuk library errors
✔ Guna anyhow untuk application code
✔ expect() lebih baik dari unwrap() (tambah konteks)
✔ Collect semua error bila boleh (bukan fail-fast sahaja)
✘ Jangan unwrap() dalam production I/O code
✘ Jangan buat Result<T, String> dalam library (guna enum)
✘ Jangan ignore Result dengan let _ = fungsi()
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **CSV Validator** — parse CSV, validate setiap baris, kumpul SEMUA ralat
2. **REST Client** — chain HTTP request dengan proper error types + retry
3. **Database Migration** — jalankan SQL migrations dengan rollback bila error
4. **Form Validator** — validate form input, return `Vec<FieldError>`
5. **Config Builder** — builder pattern yang accumulate errors sebelum build

---

*Error Handling in Rust — dari `panic!` hingga `thiserror` dan custom error chains. Selamat belajar!* 🦀
