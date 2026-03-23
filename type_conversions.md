# 🔄 Type Conversions dalam Rust — Panduan Lengkap

> Semua cara untuk convert satu type ke type lain dalam Rust:
> `as`, `From`, `Into`, `TryFrom`, `TryInto`, `ToString`, `FromStr`, dan lebih.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Gambaran Besar

```
Rust tidak buat implicit conversion (berbeza dari C/Java/PHP!).
Setiap conversion mesti EXPLICIT — tiada hidden type coercion.

PHP:  $x = 5 + "3";  → 8  (silently convert string ke int!)
Java: int x = 5; long y = x;  → OK, implicit widening
Rust: let x: i32 = 5;
      let y: i64 = x;  // ERROR! mesti explicit!
      let y: i64 = x as i64;  // ✔ explicit conversion

Kenapa explicit?
  ✔ Tiada "surprise" conversion yang tidak dijangka
  ✔ Jelas pada pembaca kod — "di sini ada conversion"
  ✔ Compiler boleh warn tentang lossy conversion
  ✔ Lebih selamat dan predictable
```

---

## Peta Conversions

```
                    NUMERIC
                   ┌─────────────────────────────────┐
  i8,i16,i32,i64   │  as casting (boleh lossy!)       │
  u8,u16,u32,u64   │  From/Into (hanya safe/widening) │
  f32, f64         └─────────────────────────────────┘

                    FALLIBLE
                   ┌─────────────────────────────────┐
  Mungkin gagal    │  TryFrom / TryInto               │
                   └─────────────────────────────────┘

                    STRING
                   ┌─────────────────────────────────┐
  T → String       │  ToString / format! / Display    │
  String → T       │  FromStr / parse()               │
  &str ↔ String    │  &str, to_string(), as_str()     │
                   └─────────────────────────────────┘

                    CUSTOM
                   ┌─────────────────────────────────┐
  Struct ↔ Struct  │  From / Into / TryFrom           │
  Newtype          │  From / Deref                    │
                   └─────────────────────────────────┘
```

---

## Peta Pembelajaran

```
Bab 1  → as Casting — Numeric Conversions
Bab 2  → From & Into — Infallible Conversion
Bab 3  → TryFrom & TryInto — Fallible Conversion
Bab 4  → String Conversions
Bab 5  → Deref Coercions
Bab 6  → Custom Type Conversions
Bab 7  → Pointer & Reference Conversions
Bab 8  → Collection Conversions
Bab 9  → Advanced Patterns
Bab 10 → Mini Project: Type-Safe Unit System
```

---

# BAB 1: as Casting — Numeric Conversions 🔢

## Asas as Casting

```rust
fn main() {
    // Numeric as casting
    let x: i32  = 1000;
    let y: i64  = x as i64;   // widening — selamat
    let z: i16  = x as i16;   // narrowing — mungkin lossy!
    let w: u32  = x as u32;   // i32 → u32 (negatif boleh jadi besar!)
    let f: f64  = x as f64;   // int → float
    let i: i32  = 3.99f64 as i32; // float → int (truncate, bukan round!)

    println!("i32 1000 as i64  = {}", y);  // 1000
    println!("i32 1000 as i16  = {}", z);  // 1000 (ok dalam julat)
    println!("i32 1000 as f64  = {}", f);  // 1000.0
    println!("f64 3.99 as i32  = {}", i);  // 3 (truncate, bukan 4!)

    // Overflow dengan as — wrapping!
    let besar: i32 = 300;
    let kecil: u8  = besar as u8; // 300 → 44 (300 % 256)
    println!("300 as u8 = {}", kecil); // 44 — lossy!

    let negatif: i32 = -1;
    let positif: u32 = negatif as u32; // -1 → 4294967295
    println!("-1 as u32 = {}", positif); // 4294967295!

    // Float ke int — special cases
    let inf  = f64::INFINITY;
    let nan  = f64::NAN;
    println!("inf as i32 = {}", inf as i32); // i32::MAX
    println!("nan as i32 = {}", nan as i32); // 0
}
```

---

## as Casting — Semua Kombinasi

```rust
fn main() {
    // ─── Integer widening (selamat) ───────────────────────────
    let a: u8  = 200;
    let b: u16 = a as u16;  // 200 → 200 ✔
    let c: u32 = a as u32;  // 200 → 200 ✔
    let d: u64 = a as u64;  // 200 → 200 ✔

    let e: i8  = -5;
    let f: i16 = e as i16;  // -5 → -5 ✔ (sign extended)
    let g: i64 = e as i64;  // -5 → -5 ✔

    // ─── Integer narrowing (LOSSY!) ───────────────────────────
    let big: u32 = 1000;
    let small: u8 = big as u8; // 1000 % 256 = 232

    let signed: i32 = -1;
    let uns: u8 = signed as u8; // bit pattern: 255

    // ─── Signed ↔ Unsigned (sama bit pattern) ─────────────────
    let pos: u8 = 200;
    let neg: i8 = pos as i8;    // 200 → -56 (bit pattern sama)
    println!("u8 200 as i8 = {}", neg); // -56

    let neg2: i8 = -56;
    let pos2: u8 = neg2 as u8;  // -56 → 200
    println!("i8 -56 as u8 = {}", pos2); // 200

    // ─── Float ↔ Integer ──────────────────────────────────────
    let f32val: f32 = 3.7;
    let i32val: i32 = f32val as i32; // truncate → 3 (bukan 4!)

    let i: i32 = 42;
    let f: f32 = i as f32; // 42 → 42.0

    // Pointer casting
    let n = 42i32;
    let ptr = &n as *const i32;  // reference → raw pointer
    let addr = ptr as usize;     // pointer → usize (alamat)
    println!("Alamat: {:#x}", addr);

    // bool → int
    let t = true as i32;  // 1
    let f = false as i32; // 0
    println!("{} {}", t, f);
}
```

---

## 🧠 Brain Teaser #1

Apakah output kod ini?

```rust
fn main() {
    let a: i32  = -1;
    let b: u32  = a as u32;
    let c: i64  = b as i64;
    let d: u8   = 256u32 as u8;
    let e: f32  = 3.999f64 as f32;
    let f: i32  = 3.999f64 as i32;

    println!("{} {} {} {} {}", b, c, d, e, f);
}
```

<details>
<summary>👀 Jawapan</summary>

```
4294967295 4294967295 0 4.0 3
```

Penjelasan:
- `a = -1 as u32` → **4294967295** (u32::MAX — bit pattern -1 dalam two's complement)
- `b as i64` → **4294967295** (widening u32 ke i64 — nilai sama)
- `256u32 as u8` → **0** (256 % 256 = 0 — overflow wraps!)
- `3.999f64 as f32` → **4.0** (f64 → f32 rounding — nilai 3.999 dibulatkan ke 4.0 dalam f32 precision!)
- `3.999f64 as i32` → **3** (float ke int = TRUNCATE bukan round! Potong bahagian desimal)

**Insight penting:**
- `as` dengan float ke integer = **truncate** (potong), bukan round
- `as` dengan integer overflow = **wrapping** (bukan panic, bukan saturate)
- `as` dengan f64 ke f32 = rounding (kerana precision f32 berbeza)

Kalau nak **saturating** cast: `u8::try_from(val).unwrap_or(u8::MAX)`
Kalau nak **checked** cast: `u8::try_from(val)` (return Result)
</details>

---

# BAB 2: From & Into — Infallible Conversion 🔄

## Kefahaman From & Into

```
From<T>:  "Saya boleh buat diri saya DARI T"
Into<T>:  "Saya boleh jadi T"

HUBUNGAN: Implement From<A> untuk B
          → Rust AUTO-implement Into<B> untuk A
          Jadi hanya implement From — dapat Into percuma!

Bila guna From vs Into:
  From  → dalam function implementation / constructor
  Into  → dalam function signature untuk flexibility

Rule: impl From<A> for B
      Then: A::into() → B    (auto-generated Into)
            B::from(a)       (manual From)
```

```rust
// From<T> — implement ini
#[derive(Debug)]
struct Celsius(f64);

#[derive(Debug)]
struct Fahrenheit(f64);

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self {
        Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
    }
}

impl From<Fahrenheit> for Celsius {
    fn from(f: Fahrenheit) -> Self {
        Celsius((f.0 - 32.0) * 5.0 / 9.0)
    }
}

fn main() {
    let titik_didih = Celsius(100.0);

    // Guna From
    let f = Fahrenheit::from(Celsius(100.0));
    println!("100°C = {:?}", f); // Fahrenheit(212.0)

    // Guna Into (auto-generated!)
    let f2: Fahrenheit = Celsius(100.0).into();
    println!("{:?}", f2); // Fahrenheit(212.0)

    // Balik arah
    let c: Celsius = Fahrenheit(98.6).into();
    println!("98.6°F = {:.2}°C", c.0); // 37.00°C

    // Standard library examples
    let s1 = String::from("hello");    // From<&str> for String
    let s2: String = "hello".into();   // Into<String> for &str

    let v: Vec<i32> = vec![1, 2, 3];
    let s: std::collections::HashSet<i32> = v.into_iter().collect();
}
```

---

## From dalam Standard Library

```rust
fn main() {
    // String conversions
    let s1 = String::from("hello");    // &str → String
    let s2 = String::from('A');         // char → String
    let s3 = String::from(vec!['a', 'b']); // Vec<char> → String (tidak ada, guna collect)

    // Number → String
    // (tiada From<i32> for String, guna .to_string())

    // Vec dari array
    let arr = [1, 2, 3, 4, 5];
    let v: Vec<i32> = Vec::from(arr);
    let v2: Vec<i32> = arr.into();     // Into (auto dari From)

    // Box
    let boxed: Box<i32> = Box::from(42);
    let boxed2: Box<str> = Box::from("hello");
    let boxed3: Box<[i32]> = Box::from([1, 2, 3]);

    // Option
    let opt: Option<i32> = Some(42);
    // Some(x) adalah From<T> untuk Option<T> — auto-wrapping

    // PathBuf
    use std::path::PathBuf;
    let path = PathBuf::from("/tmp/file.txt");
    let path2: PathBuf = "/tmp/other.txt".into();

    // Error wrapping (paling penting!)
    // impl From<io::Error> for MyError { ... }
    // Membolehkan ? operator auto-convert errors!
}
```

---

## impl From untuk Custom Types

```rust
// ─── Newtype Pattern ──────────────────────────────────────────
struct UserId(u32);
struct Emel(String);

impl From<u32> for UserId {
    fn from(id: u32) -> Self { UserId(id) }
}

impl From<String> for Emel {
    fn from(s: String) -> Self { Emel(s) }
}

impl From<&str> for Emel {
    fn from(s: &str) -> Self { Emel(s.to_string()) }
}

// ─── Enum conversion ──────────────────────────────────────────
#[derive(Debug)]
enum Warna { Merah, Hijau, Biru, Khas(u8, u8, u8) }

impl From<(u8, u8, u8)> for Warna {
    fn from((r, g, b): (u8, u8, u8)) -> Self {
        match (r, g, b) {
            (255, 0, 0) => Warna::Merah,
            (0, 255, 0) => Warna::Hijau,
            (0, 0, 255) => Warna::Biru,
            (r, g, b)   => Warna::Khas(r, g, b),
        }
    }
}

// ─── Struct conversion ────────────────────────────────────────
#[derive(Debug)]
struct PenggunaCsv { nama: String, umur: u32 }

#[derive(Debug)]
struct PenggunaDb { id: u64, nama: String, umur: u32, aktif: bool }

impl From<PenggunaCsv> for PenggunaDb {
    fn from(csv: PenggunaCsv) -> Self {
        PenggunaDb {
            id:    0, // akan assign oleh DB
            nama:  csv.nama,
            umur:  csv.umur,
            aktif: true,
        }
    }
}

fn main() {
    let id: UserId = 42u32.into();
    let emel: Emel = "ali@email.com".into();
    let emel2: Emel = String::from("siti@email.com").into();

    let merah: Warna = (255u8, 0, 0).into();
    let hijau: Warna = (0u8, 255, 0).into();
    let khas:  Warna = (100u8, 150, 200).into();
    println!("{:?}", merah); // Merah
    println!("{:?}", khas);  // Khas(100, 150, 200)

    let csv = PenggunaCsv { nama: "Ali".into(), umur: 25 };
    let db: PenggunaDb = csv.into();
    println!("{:?}", db);
}
```

---

# BAB 3: TryFrom & TryInto — Fallible Conversion 🛡️

## Bila From Tidak Cukup

```
From<T>:    Sentiasa berjaya — tiada failure case
TryFrom<T>: Mungkin gagal — return Result<Self, Error>

Guna TryFrom bila:
  - Nilai mungkin luar julat
  - Validasi diperlukan
  - Conversion mungkin tidak masuk akal
```

```rust
use std::convert::{TryFrom, TryInto};

// Standard library TryFrom
fn main() {
    // i32 → u8: mungkin gagal (overflow atau negatif)
    let ok: Result<u8, _>  = u8::try_from(200i32);
    let err: Result<u8, _> = u8::try_from(300i32);
    let neg: Result<u8, _> = u8::try_from(-1i32);

    println!("{:?}", ok);  // Ok(200)
    println!("{:?}", err); // Err(TryFromIntError(()))
    println!("{:?}", neg); // Err(TryFromIntError(()))

    // TryInto (auto dari TryFrom)
    let hasil: Result<u8, _> = 100i32.try_into();
    println!("{:?}", hasil); // Ok(100)

    // Dengan proper error handling
    match 300i32.try_into() {
        Ok(n)  => println!("OK: u8 = {}", n),
        Err(e) => println!("Err: {}", e), // out of range integral type conversion
    }

    // Guna dalam fungsi
    fn masuk_port(port: i32) -> Result<u16, String> {
        u16::try_from(port).map_err(|_| format!("Port tidak sah: {}", port))
    }

    println!("{:?}", masuk_port(8080));  // Ok(8080)
    println!("{:?}", masuk_port(-1));    // Err("Port tidak sah: -1")
    println!("{:?}", masuk_port(70000)); // Err("Port tidak sah: 70000")
}
```

---

## Custom TryFrom

```rust
use std::convert::TryFrom;

#[derive(Debug)]
struct Umur(u8);

#[derive(Debug)]
struct UmurError {
    nilai: i32,
    sebab: &'static str,
}

impl std::fmt::Display for UmurError {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "Umur tidak sah ({}): {}", self.nilai, self.sebab)
    }
}

impl TryFrom<i32> for Umur {
    type Error = UmurError;

    fn try_from(nilai: i32) -> Result<Self, Self::Error> {
        match nilai {
            n if n < 0    => Err(UmurError { nilai: n, sebab: "negatif" }),
            n if n > 150  => Err(UmurError { nilai: n, sebab: "tidak munasabah" }),
            n             => Ok(Umur(n as u8)),
        }
    }
}

// TryFrom untuk struct validation
#[derive(Debug)]
struct Emel(String);

#[derive(Debug, thiserror::Error)]
#[error("Format emel tidak sah: '{0}'")]
struct EmelError(String);

impl TryFrom<String> for Emel {
    type Error = EmelError;

    fn try_from(s: String) -> Result<Self, Self::Error> {
        if s.contains('@') && s.contains('.') && s.len() > 5 {
            Ok(Emel(s))
        } else {
            Err(EmelError(s))
        }
    }
}

impl TryFrom<&str> for Emel {
    type Error = EmelError;

    fn try_from(s: &str) -> Result<Self, Self::Error> {
        Emel::try_from(s.to_string())
    }
}

fn main() {
    // Umur
    println!("{:?}", Umur::try_from(25));   // Ok(Umur(25))
    println!("{:?}", Umur::try_from(-1));   // Err(...)
    println!("{:?}", Umur::try_from(200));  // Err(...)

    // Emel
    let e1: Result<Emel, _> = "ali@email.com".try_into();
    let e2: Result<Emel, _> = "bukan-emel".try_into();
    println!("{:?}", e1); // Ok(Emel("ali@email.com"))
    println!("{:?}", e2); // Err(EmelError("bukan-emel"))
}
```

---

## 🧠 Brain Teaser #2

Bila nak guna `From` vs `TryFrom`?

```rust
// Scenario A: Convert Meter ke Centimeter
// Scenario B: Convert i32 ke u8
// Scenario C: Convert &str ke Email (validate format)
// Scenario D: Convert String ke u32 (parse)
// Scenario E: Convert Celsius ke Fahrenheit
// Scenario F: Convert u8 ke u32
```

<details>
<summary>👀 Jawapan</summary>

```
A. Meter → Centimeter:
   From ✔ — sentiasa berjaya (nilai × 100, tiada possibility gagal)
   impl From<Meter> for Centimeter { fn from(m) -> Self { Centimeter(m.0 * 100.0) } }

B. i32 → u8:
   TryFrom ✔ — mungkin gagal (overflow, negatif)
   Standard library dah provide: u8::try_from(val)

C. &str → Email (validate):
   TryFrom ✔ — validation boleh gagal
   impl TryFrom<&str> for Email { ... }

D. String → u32 (parse):
   FromStr / parse() ✔ — parse boleh gagal
   "42".parse::<u32>() — ini bukan From/TryFrom, tapi FromStr trait

E. Celsius → Fahrenheit:
   From ✔ — formula matematik, sentiasa berjaya
   impl From<Celsius> for Fahrenheit { ... }

F. u8 → u32:
   From ✔ — u8 (0-255) selalu muat dalam u32, tiada loss
   Standard library dah provide: u32::from(val: u8)

RINGKASAN:
  Sentiasa berjaya + tiada loss → From
  Mungkin gagal ATAU ada loss    → TryFrom
  String parsing                  → FromStr / parse()
```
</details>

---

# BAB 4: String Conversions 🔤

## &str ↔ String ↔ &[u8] ↔ Vec\<u8\>

```rust
fn main() {
    // ─── &str ↔ String ────────────────────────────────────────

    // &str → String
    let s1: String = "hello".to_string();       // paling common
    let s2: String = String::from("hello");      // explicit From
    let s3: String = "hello".to_owned();         // semantik "buat owned copy"
    let s4: String = format!("{}", "hello");     // via format!
    let s5: String = ["hel", "lo"].concat();     // gabung &str

    // String → &str
    let owned = String::from("hello");
    let borrowed: &str = &owned;                 // auto-deref
    let borrowed2: &str = owned.as_str();        // explicit
    let borrowed3: &str = &owned[..];            // slice

    // ─── String ↔ Bytes ───────────────────────────────────────

    // String → Vec<u8>
    let s = String::from("hello");
    let bytes: Vec<u8> = s.into_bytes();         // consume String
    let bytes2 = "hello".as_bytes();             // &[u8], borrow

    // Vec<u8> → String (fallible — mungkin bukan UTF-8!)
    let v = vec![104u8, 101, 108, 108, 111]; // "hello"
    let s = String::from_utf8(v).unwrap();        // Result<String, FromUtf8Error>
    let s2 = unsafe { String::from_utf8_unchecked(b"hello".to_vec()) }; // unsafe!

    // &[u8] → &str (fallible)
    let bytes = b"hello world";  // &[u8]
    let s: &str = std::str::from_utf8(bytes).unwrap(); // Result<&str, Utf8Error>

    println!("{}", s);

    // ─── char ↔ String ────────────────────────────────────────

    let c: char = 'A';
    let s: String = c.to_string();
    let s2: String = String::from(c);
    let s3 = format!("{}", c);

    // String → char (kalau satu char)
    let single = "A";
    let ch: Option<char> = single.chars().next();
    println!("{:?}", ch); // Some('A')
}
```

---

## T → String: to_string() dan Display

```rust
use std::fmt;

// CARA PALING COMMON: implement Display, dapat to_string() percuma!
// (Blanket impl: impl<T: Display> ToString for T)

#[derive(Debug)]
struct Titik { x: f64, y: f64 }

impl fmt::Display for Titik {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

fn main() {
    let t = Titik { x: 3.0, y: 4.0 };

    // Semua ini berfungsi kerana ada Display:
    let s1 = t.to_string();          // "guna ToString blanket impl"
    let s2 = format!("{}", t);       // "guna Display"
    let s3 = format!("{:?}", t);     // "guna Debug"

    println!("{}", s1); // (3, 4)
    println!("{}", s2); // (3, 4)
    println!("{}", s3); // Titik { x: 3.0, y: 4.0 }

    // Primitives pun ada to_string():
    let n: i32 = 42;
    let f: f64 = 3.14;
    let b: bool = true;

    println!("{}", n.to_string()); // "42"
    println!("{}", f.to_string()); // "3.14"
    println!("{}", b.to_string()); // "true"

    // Number dalam base lain
    println!("{:b}", 42u8); // "101010" (binary)
    println!("{:o}", 42u8); // "52"     (octal)
    println!("{:x}", 42u8); // "2a"     (hex lowercase)
    println!("{:X}", 42u8); // "2A"     (hex uppercase)

    // Boleh format! pun
    let hex = format!("{:#010x}", 255u32); // "0x000000ff"
}
```

---

## String → T: parse() dan FromStr

```rust
use std::str::FromStr;

fn main() {
    // ─── parse() — String/&str → T ────────────────────────────

    // Parse nombor
    let n: i32    = "42".parse().unwrap();
    let f: f64    = "3.14".parse().unwrap();
    let b: bool   = "true".parse().unwrap();
    let c: char   = "A".parse().unwrap();

    // Parse dengan turbofish
    let n2 = "100".parse::<u64>().unwrap();

    // Parse dengan proper error handling
    match "abc".parse::<i32>() {
        Ok(n)  => println!("Parse OK: {}", n),
        Err(e) => println!("Parse gagal: {}", e),
    }

    // ─── Custom FromStr ────────────────────────────────────────

    #[derive(Debug)]
    struct Warna { r: u8, g: u8, b: u8 }

    #[derive(Debug)]
    struct WarnaParseError(String);

    impl fmt::Display for WarnaParseError {
        fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
            write!(f, "Warna tidak sah: {}", self.0)
        }
    }

    impl FromStr for Warna {
        type Err = WarnaParseError;

        // Parse "R,G,B" atau "#RRGGBB"
        fn from_str(s: &str) -> Result<Self, Self::Err> {
            let s = s.trim();

            // Format "#RRGGBB"
            if s.starts_with('#') && s.len() == 7 {
                let r = u8::from_str_radix(&s[1..3], 16).map_err(|_| WarnaParseError(s.to_string()))?;
                let g = u8::from_str_radix(&s[3..5], 16).map_err(|_| WarnaParseError(s.to_string()))?;
                let b = u8::from_str_radix(&s[5..7], 16).map_err(|_| WarnaParseError(s.to_string()))?;
                return Ok(Warna { r, g, b });
            }

            // Format "R,G,B"
            let parts: Vec<&str> = s.split(',').collect();
            if parts.len() != 3 {
                return Err(WarnaParseError(s.to_string()));
            }

            let r = parts[0].trim().parse::<u8>().map_err(|_| WarnaParseError(s.to_string()))?;
            let g = parts[1].trim().parse::<u8>().map_err(|_| WarnaParseError(s.to_string()))?;
            let b = parts[2].trim().parse::<u8>().map_err(|_| WarnaParseError(s.to_string()))?;

            Ok(Warna { r, g, b })
        }
    }

    // Guna parse() dengan custom type!
    let w1: Warna = "#FF8000".parse().unwrap();
    let w2: Warna = "255, 128, 0".parse().unwrap();
    let w3: Result<Warna, _> = "bukan warna".parse();

    println!("{:?}", w1); // Warna { r: 255, g: 128, b: 0 }
    println!("{:?}", w2); // Warna { r: 255, g: 128, b: 0 }
    println!("{:?}", w3); // Err(WarnaParseError("bukan warna"))
}
```

---

# BAB 5: Deref Coercions 🪄

## Automatic Dereferencing

```rust
// Deref coercion = Rust auto-dereference dalam context tertentu
// String → &str
// Vec<T> → &[T]
// Box<T> → &T
// Arc<T> → &T

fn cetak(s: &str) { println!("{}", s); }
fn jumlah(v: &[i32]) -> i32 { v.iter().sum() }

fn main() {
    // String boleh digunakan di mana &str dijangka
    let s = String::from("hello");
    cetak(&s);          // &String → &str (Deref coercion!)
    cetak(&s[..]);      // explicit slice
    cetak(s.as_str());  // explicit conversion

    // Vec boleh digunakan di mana &[T] dijangka
    let v = vec![1, 2, 3, 4, 5];
    println!("{}", jumlah(&v));      // &Vec<i32> → &[i32] (Deref!)
    println!("{}", jumlah(&v[1..])); // slice

    // Box<T> → &T
    let boxed = Box::new(42i32);
    let r: &i32 = &boxed;           // &Box<i32> → &i32
    println!("{}", r);               // 42

    // Chaining coercions
    let boxed_string = Box::new(String::from("hello"));
    cetak(&boxed_string); // &Box<String> → &String → &str (dua coercions!)
}
```

---

## Implement Deref untuk Custom Types

```rust
use std::ops::{Deref, DerefMut};

struct StackVec<T> {
    data: Vec<T>,
    had:  usize,
}

impl<T> StackVec<T> {
    fn baru(had: usize) -> Self {
        StackVec { data: Vec::with_capacity(had), had }
    }

    fn push(&mut self, item: T) -> bool {
        if self.data.len() < self.had {
            self.data.push(item);
            true
        } else {
            false
        }
    }
}

// Deref → StackVec boleh digunakan seperti slice!
impl<T> Deref for StackVec<T> {
    type Target = [T];
    fn deref(&self) -> &[T] { &self.data }
}

impl<T> DerefMut for StackVec<T> {
    fn deref_mut(&mut self) -> &mut [T] { &mut self.data }
}

fn main() {
    let mut sv: StackVec<i32> = StackVec::baru(5);
    sv.push(1); sv.push(2); sv.push(3);

    // Guna semua slice methods via Deref!
    println!("Len: {}", sv.len());      // 3
    println!("First: {:?}", sv.first()); // Some(1)
    println!("Sum: {}", sv.iter().sum::<i32>()); // 6
    sv.sort();                           // mutable!
    println!("{:?}", &*sv);              // [1, 2, 3]
}
```

---

# BAB 6: Custom Type Conversions 🏗️

## Conversion Pattern yang Lengkap

```rust
use std::fmt;
use std::convert::TryFrom;

// ─── Type dengan penuh conversions ────────────────────────────

#[derive(Debug, Clone, PartialEq)]
struct Ringgit {
    sen: u64, // simpan dalam sen untuk ketepatan
}

#[derive(Debug, thiserror::Error)]
pub enum RinggitError {
    #[error("Nilai negatif tidak dibenarkan")]
    Negatif,
    #[error("Nilai terlalu besar: {0}")]
    TerlaluBesar(f64),
    #[error("Format tidak sah: '{0}'")]
    FormatTidakSah(String),
}

impl Ringgit {
    pub fn baru(rm: f64) -> Result<Self, RinggitError> {
        if rm < 0.0 {
            Err(RinggitError::Negatif)
        } else if rm > 1_000_000_000.0 {
            Err(RinggitError::TerlaluBesar(rm))
        } else {
            Ok(Ringgit { sen: (rm * 100.0).round() as u64 })
        }
    }

    pub fn sebagai_rm(&self) -> f64 {
        self.sen as f64 / 100.0
    }
}

// From<u64> — dari sen (sentiasa berjaya)
impl From<u64> for Ringgit {
    fn from(sen: u64) -> Self { Ringgit { sen } }
}

// TryFrom<f64> — dari RM (mungkin gagal)
impl TryFrom<f64> for Ringgit {
    type Error = RinggitError;
    fn try_from(rm: f64) -> Result<Self, Self::Error> {
        Ringgit::baru(rm)
    }
}

// TryFrom<&str> — parse dari string "RM50.00" atau "50.00"
impl TryFrom<&str> for Ringgit {
    type Error = RinggitError;
    fn try_from(s: &str) -> Result<Self, Self::Error> {
        let s = s.trim().trim_start_matches("RM");
        s.parse::<f64>()
            .map_err(|_| RinggitError::FormatTidakSah(s.to_string()))
            .and_then(Ringgit::baru)
    }
}

// Display
impl fmt::Display for Ringgit {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "RM{:.2}", self.sebagai_rm())
    }
}

// Into<f64>
impl From<Ringgit> for f64 {
    fn from(r: Ringgit) -> f64 { r.sebagai_rm() }
}

fn main() {
    // Pelbagai cara buat Ringgit
    let r1 = Ringgit::baru(50.00).unwrap();
    let r2: Ringgit = 5000u64.into();           // dari sen
    let r3: Result<Ringgit, _> = 29.99f64.try_into();
    let r4: Result<Ringgit, _> = "RM15.50".try_into();
    let r5: Result<Ringgit, _> = (-5.0f64).try_into();

    println!("{}", r1);           // RM50.00
    println!("{}", r2);           // RM50.00
    println!("{:?}", r3);         // Ok(Ringgit { sen: 2999 })
    println!("{:?}", r4);         // Ok(...)
    println!("{:?}", r5);         // Err(Negatif)

    // Tukar ke f64
    let nilai: f64 = r1.into();
    println!("{}", nilai);        // 50.0
}
```

---

# BAB 7: Pointer & Reference Conversions 🔗

## Reference ↔ Raw Pointer

```rust
fn main() {
    let x = 42i32;
    let mut y = 99i32;

    // Reference → Raw Pointer (SAFE)
    let ptr_const: *const i32 = &x as *const i32;
    let ptr_const2 = std::ptr::addr_of!(x);  // safer way
    let ptr_mut: *mut i32 = &mut y as *mut i32;
    let ptr_mut2 = std::ptr::addr_of_mut!(y);

    // Raw Pointer → Reference (UNSAFE — verify validity dulu!)
    unsafe {
        let r: &i32 = &*ptr_const;     // deref + reborrow
        let r2 = ptr_const.as_ref();   // Option<&i32>
        let r3 = ptr_mut.as_mut();     // Option<&mut i32>
        println!("{:?}", r2);           // Some(42)
        println!("{:?}", r3);           // Some(99)
    }

    // Pointer ↔ Integer (untuk debug/FFI)
    let addr = ptr_const as usize;
    println!("Alamat: {:#x}", addr);

    // usize → pointer (SANGAT BERBAHAYA!)
    // let ptr_dari_addr = addr as *const i32;
}
```

---

## Slice ↔ Pointer

```rust
fn main() {
    let data = [1i32, 2, 3, 4, 5];
    let v = vec![10i32, 20, 30, 40, 50];

    // Slice → raw pointer
    let ptr: *const i32 = data.as_ptr();
    let ptr_v: *const i32 = v.as_ptr();

    // Raw pointer → slice (UNSAFE)
    unsafe {
        let slice = std::slice::from_raw_parts(ptr, data.len());
        println!("{:?}", slice); // [1, 2, 3, 4, 5]
    }

    // &[T] ↔ &[u8] (transmute bytes)
    let floats = [1.0f32, 2.0, 3.0];
    let bytes: &[u8] = unsafe {
        std::slice::from_raw_parts(
            floats.as_ptr() as *const u8,
            floats.len() * std::mem::size_of::<f32>()
        )
    };
    println!("Bytes: {:?}", bytes);
}
```

---

# BAB 8: Collection Conversions 📚

## Vec ↔ Pelbagai Collection

```rust
use std::collections::{HashMap, HashSet, BTreeMap, BTreeSet, VecDeque};

fn main() {
    // ─── Vec ↔ HashSet ────────────────────────────────────────
    let v = vec![1, 2, 3, 2, 1, 4, 3];

    // Vec → HashSet (buang duplikat)
    let set: HashSet<i32> = v.iter().copied().collect();
    println!("Set: {:?}", set); // {1, 2, 3, 4}

    // Vec → HashSet via From
    let set2: HashSet<i32> = HashSet::from_iter(v.iter().copied());

    // HashSet → Vec (tiada urutan dijamin!)
    let v2: Vec<i32> = set.into_iter().collect();

    // Vec → BTreeSet (tersusun)
    let bset: BTreeSet<i32> = v.iter().copied().collect();
    let sorted_v: Vec<i32> = bset.into_iter().collect();
    println!("Sorted: {:?}", sorted_v); // [1, 2, 3, 4]

    // ─── Vec ↔ HashMap ────────────────────────────────────────
    let pasangan = vec![("Ali", 85), ("Siti", 92), ("Amin", 78)];

    // Vec<(K, V)> → HashMap
    let map: HashMap<&str, i32> = pasangan.iter().copied().collect();
    println!("{:?}", map.get("Ali")); // Some(85)

    // HashMap → Vec<(K, V)>
    let balik: Vec<(&&str, &i32)> = map.iter().collect();

    // ─── Vec ↔ VecDeque ───────────────────────────────────────
    let v = vec![1, 2, 3, 4, 5];

    let deque: VecDeque<i32> = v.into_iter().collect();
    // atau: VecDeque::from(vec![...])

    let v2: Vec<i32> = deque.into_iter().collect();

    // ─── Iterator Methods untuk Collect ───────────────────────
    let words = vec!["hello", "world", "hello", "rust"];

    // Kira kemunculan
    let count: HashMap<&&str, usize> = words.iter()
        .fold(HashMap::new(), |mut m, w| {
            *m.entry(w).or_insert(0) += 1;
            m
        });

    // Group by panjang
    let by_len: HashMap<usize, Vec<&&str>> = words.iter()
        .fold(HashMap::new(), |mut m, w| {
            m.entry(w.len()).or_default().push(w);
            m
        });

    println!("{:?}", count);
    println!("{:?}", by_len);
}
```

---

## 🧠 Brain Teaser #3

Bagaimana convert `Vec<Result<T, E>>` → `Result<Vec<T>, E>` dan
`Vec<Option<T>>` → `Option<Vec<T>>`?

<details>
<summary>👀 Jawapan</summary>

```rust
fn main() {
    // Vec<Result<T, E>> → Result<Vec<T>, E>
    // collect() boleh buat ini!
    let results = vec![Ok(1), Ok(2), Ok(3)];
    let v: Result<Vec<i32>, String> = results.into_iter().collect();
    println!("{:?}", v); // Ok([1, 2, 3])

    // Kalau ada Err, return Err pertama:
    let results2 = vec![Ok(1), Err("gagal".to_string()), Ok(3)];
    let v2: Result<Vec<i32>, String> = results2.into_iter().collect();
    println!("{:?}", v2); // Err("gagal")

    // Vec<Option<T>> → Option<Vec<T>>
    let opts = vec![Some(1), Some(2), Some(3)];
    let v3: Option<Vec<i32>> = opts.into_iter().collect();
    println!("{:?}", v3); // Some([1, 2, 3])

    // Kalau ada None, return None:
    let opts2 = vec![Some(1), None, Some(3)];
    let v4: Option<Vec<i32>> = opts2.into_iter().collect();
    println!("{:?}", v4); // None

    // Praktik: parse senarai string ke Vec<i32>
    let strings = vec!["1", "2", "3", "abc", "5"];

    // Cara 1: Fail bila ada yang gagal
    let all: Result<Vec<i32>, _> = strings.iter()
        .map(|s| s.parse::<i32>())
        .collect();
    println!("{:?}", all); // Err(...)

    // Cara 2: Skip yang gagal
    let valid: Vec<i32> = strings.iter()
        .filter_map(|s| s.parse().ok())
        .collect();
    println!("{:?}", valid); // [1, 2, 3, 5]
}
```
</details>

---

# BAB 9: Advanced Patterns 🎯

## Type Aliases vs Newtypes

```rust
// Type alias — SAMA type, cuma nama berbeza
type Meter = f64;         // Meter IS f64
type Kilometer = f64;     // Kilometer IS f64
type KoordinatGPS = (f64, f64);

fn tambah_meter(a: Meter, b: Meter) -> Meter { a + b }

// Problem dengan type alias:
fn main() {
    let a: Meter = 1000.0;
    let b: Kilometer = 1.0;
    // tambah_meter(a, b) — TIDAK ADA ERROR! kedua-dua f64!
}

// ─────────────────────────────────────────────────────────────

// Newtype — TYPE BARU yang wrap type lain
struct Meter(f64);
struct Kilometer(f64);

impl From<Kilometer> for Meter {
    fn from(k: Kilometer) -> Self { Meter(k.0 * 1000.0) }
}

fn tambah_meter(a: Meter, b: Meter) -> Meter {
    Meter(a.0 + b.0)
}

// Sekarang:
// let b: Kilometer = Kilometer(1.0);
// tambah_meter(a, b);  // ERROR! type mismatch!

// Mesti convert explicitly:
// tambah_meter(a, b.into());  // ✔
```

---

## Transmute — Last Resort

```rust
use std::mem;

fn main() {
    // transmute — tukar type interpretation bitwise
    // SANGAT BERBAHAYA, guna HANYA bila tiada alternatif

    // f32 bits → u32 (ada safe alternative: .to_bits())
    let f: f32 = 1.0;
    let bits_unsafe: u32 = unsafe { mem::transmute(f) };
    let bits_safe: u32 = f.to_bits(); // LEBIH BAIK!
    println!("{} == {}: {}", bits_unsafe, bits_safe, bits_unsafe == bits_safe);

    // Tukar lifetime (SANGAT BERBAHAYA — verify correctness manual)
    // Contoh: extend lifetime reference (hanya bila betul-betul selamat)
    fn extend_lifetime<'b, T>(r: &'b T) -> &'static T {
        unsafe { mem::transmute(r) }
        // WARNING: Caller MESTI pastikan T hidup selama 'static!
    }

    // Alternatives kepada transmute:
    // cast pointer types
    // bytemuck::cast() — safe checked transmute
    // .to_bits() / .from_bits() untuk float
    // u8::from_ne_bytes() / .to_ne_bytes() untuk bytes ↔ integer
}
```

---

## bytes ↔ Integer (Safe Alternatives)

```rust
fn main() {
    // ─── Integer ↔ Bytes ──────────────────────────────────────

    let n: u32 = 0x12345678;

    // to_be_bytes() — big-endian bytes
    let be = n.to_be_bytes();
    println!("BE: {:?}", be); // [18, 52, 86, 120]

    // to_le_bytes() — little-endian bytes
    let le = n.to_le_bytes();
    println!("LE: {:?}", le); // [120, 86, 52, 18]

    // to_ne_bytes() — native-endian bytes (platform-dependent)
    let ne = n.to_ne_bytes();

    // Bytes → Integer
    let n2 = u32::from_be_bytes(be);
    let n3 = u32::from_le_bytes(le);
    println!("{} == {} == {}", n, n2, n3); // sama

    // Float ↔ bits
    let f: f64 = std::f64::consts::PI;
    let bits: u64 = f.to_bits();
    let f2 = f64::from_bits(bits);
    println!("{} == {}: {}", f, f2, f == f2);

    // ─── bytemuck untuk safe casting ──────────────────────────
    // [dependencies]
    // bytemuck = "1"

    // Dengan bytemuck:
    // bytemuck::cast::<f32, u32>(3.14) — safe, compile-time checked
    // bytemuck::cast_slice::<u8, u32>(&bytes) — slice casting
}
```

---

# BAB 10: Mini Project — Type-Safe Unit System 🏗️

```rust
use std::fmt;
use std::ops::{Add, Sub, Mul, Div};
use std::convert::TryFrom;

// ─── Unit Traits ──────────────────────────────────────────────

/// Trait untuk semua unit ukuran
pub trait Unit: Copy + fmt::Display {
    fn nilai(&self) -> f64;
    fn unit_symbol() -> &'static str;
}

// ─── Length Units ─────────────────────────────────────────────

macro_rules! buat_unit_panjang {
    ($nama:ident, $simbol:literal, $ke_meter:expr) => {
        #[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
        pub struct $nama(pub f64);

        impl $nama {
            pub fn baru(nilai: f64) -> Option<Self> {
                if nilai.is_finite() && nilai >= 0.0 {
                    Some($nama(nilai))
                } else {
                    None
                }
            }
        }

        impl Unit for $nama {
            fn nilai(&self) -> f64 { self.0 }
            fn unit_symbol() -> &'static str { $simbol }
        }

        impl fmt::Display for $nama {
            fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
                write!(f, "{:.4} {}", self.0, $simbol)
            }
        }

        impl Add for $nama {
            type Output = Self;
            fn add(self, rhs: Self) -> Self { $nama(self.0 + rhs.0) }
        }

        impl Sub for $nama {
            type Output = Self;
            fn sub(self, rhs: Self) -> Self { $nama((self.0 - rhs.0).abs()) }
        }

        impl Mul<f64> for $nama {
            type Output = Self;
            fn mul(self, rhs: f64) -> Self { $nama(self.0 * rhs) }
        }

        impl Div<f64> for $nama {
            type Output = Self;
            fn div(self, rhs: f64) -> Self { $nama(self.0 / rhs) }
        }

        // Tukar ke Meter (base unit)
        impl From<$nama> for Meter {
            fn from(val: $nama) -> Meter {
                Meter(val.0 * $ke_meter)
            }
        }

        // Tukar dari Meter
        impl From<Meter> for $nama {
            fn from(m: Meter) -> Self {
                $nama(m.0 / $ke_meter)
            }
        }
    };
}

// Define unit panjang
buat_unit_panjang!(Milimeter,  "mm",  0.001);
buat_unit_panjang!(Sentimeter, "cm",  0.01);
buat_unit_panjang!(Meter,      "m",   1.0);
buat_unit_panjang!(Kilometer,  "km",  1000.0);
buat_unit_panjang!(Kaki,       "ft",  0.3048);
buat_unit_panjang!(Inci,       "in",  0.0254);
buat_unit_panjang!(Batu,       "mi",  1609.344);

// ─── Suhu ─────────────────────────────────────────────────────

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Celsius(pub f64);

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Fahrenheit(pub f64);

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Kelvin(pub f64);

#[derive(Debug)]
pub struct SuhuError(pub String);

impl fmt::Display for SuhuError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Suhu tidak sah: {}", self.0)
    }
}

impl fmt::Display for Celsius {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{:.2}°C", self.0)
    }
}
impl fmt::Display for Fahrenheit {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{:.2}°F", self.0)
    }
}
impl fmt::Display for Kelvin {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{:.2}K", self.0)
    }
}

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self { Fahrenheit(c.0 * 9.0/5.0 + 32.0) }
}

impl From<Fahrenheit> for Celsius {
    fn from(f: Fahrenheit) -> Self { Celsius((f.0 - 32.0) * 5.0/9.0) }
}

impl From<Celsius> for Kelvin {
    fn from(c: Celsius) -> Self { Kelvin(c.0 + 273.15) }
}

impl From<Kelvin> for Celsius {
    fn from(k: Kelvin) -> Self { Celsius(k.0 - 273.15) }
}

impl TryFrom<Kelvin> for Celsius {
    type Error = SuhuError;
    fn try_from(k: Kelvin) -> Result<Self, Self::Error> {
        if k.0 < 0.0 {
            Err(SuhuError(format!("{} lebih rendah dari sifar mutlak (0K)", k)))
        } else {
            Ok(Celsius(k.0 - 273.15))
        }
    }
}

// ─── Demo ──────────────────────────────────────────────────────

fn main() {
    println!("=== Unit Panjang ===\n");

    let jarak = Kilometer(1.5);
    let jarak_m: Meter  = jarak.into();
    let jarak_cm: Sentimeter = jarak_m.into();
    let jarak_kaki: Kaki = Meter::from(jarak).into();

    println!("{}  =  {}  =  {}  =  {}", jarak, jarak_m, jarak_cm, jarak_kaki);

    // Operasi
    let a = Meter(100.0);
    let b = Meter(50.0);
    println!("\n{} + {} = {}", a, b, a + b);
    println!("{} - {} = {}", a, b, a - b);
    println!("{} × 2 = {}", a, a * 2.0);

    // Buta unit (berbeza unit, boleh convert dulu)
    let inci = Inci(72.0);
    let m: Meter = inci.into();
    println!("\n{} = {}", inci, m);

    // Jarak KL ke Kemubu dalam pelbagai unit
    let kl_kemubu = Kilometer(170.0);
    println!("\nKL → Kemubu:");
    println!("  {} = {} = {} = {}",
        kl_kemubu,
        Meter::from(kl_kemubu),
        Batu::from(Meter::from(kl_kemubu)),
        Kaki::from(Meter::from(kl_kemubu))
    );

    println!("\n=== Suhu ===\n");

    let air_mendidih = Celsius(100.0);
    let air_beku    = Celsius(0.0);
    let badan       = Fahrenheit(98.6);

    println!("Air mendidih:");
    println!("  {} = {} = {}",
        air_mendidih,
        Fahrenheit::from(air_mendidih),
        Kelvin::from(air_mendidih)
    );

    println!("Air beku:");
    println!("  {} = {} = {}",
        air_beku,
        Fahrenheit::from(air_beku),
        Kelvin::from(air_beku)
    );

    println!("Suhu badan:");
    let c_badan = Celsius::from(badan);
    println!("  {} = {} = {}",
        badan, c_badan, Kelvin::from(c_badan)
    );

    // TryFrom untuk validasi
    let sifar_mutlak = Kelvin(0.0);
    match Celsius::try_from(sifar_mutlak) {
        Ok(c)  => println!("\n0K = {}", c),
        Err(e) => println!("\n0K: {}", e),
    }

    let suhu_tak_sah = Kelvin(-10.0);
    match Celsius::try_from(suhu_tak_sah) {
        Ok(c)  => println!("-10K = {}", c),
        Err(e) => println!("-10K tidak sah: {}", e),
    }
}
```

---

# 📋 Rujukan Pantas — Type Conversions Cheat Sheet

## Numeric Conversions

```rust
// as casting — explicit, boleh lossy
let x = val as TargetType;
300u32 as u8  → 44 (wrapping overflow)
-1i32 as u32  → 4294967295 (bit pattern)
3.9f64 as i32 → 3 (truncate, bukan round)

// Checked / Saturating / Wrapping
i32::try_from(val)          // Result — gagal kalau overflow
i32::saturating_from(...)   // clamp ke min/max
val.saturating_add(other)
val.checked_add(other)      // Option
val.wrapping_add(other)     // wraps around
```

## From / Into

```rust
// Implement From → dapat Into percuma!
impl From<A> for B { fn from(a: A) -> B { ... } }

B::from(a)      // menggunakan From
a.into()        // menggunakan Into (auto dari From)

// Standard:
String::from("hello")   // &str → String
"hello".to_string()     // &str → String
vec.into_iter()         // Vec → IntoIter
```

## TryFrom / TryInto

```rust
impl TryFrom<A> for B {
    type Error = MyError;
    fn try_from(a: A) -> Result<B, MyError> { ... }
}

B::try_from(a)?    // menggunakan TryFrom
a.try_into()?      // menggunakan TryInto (auto)
```

## String Conversions

```rust
// T → String
val.to_string()           // via Display
format!("{}", val)        // via Display
format!("{:?}", val)      // via Debug

// String → T
"42".parse::<i32>()       // via FromStr → Result
i32::from_str("42")       // sama

// &str ↔ String
"literal".to_string()     // → String
owned.as_str()            // → &str
&owned                    // → &str (deref coercion)

// String ↔ Bytes
s.into_bytes()            // String → Vec<u8>
s.as_bytes()              // String → &[u8]
String::from_utf8(bytes)  // Vec<u8> → Result<String>
```

## Collection Conversions

```rust
iter.collect::<Vec<T>>()
iter.collect::<HashSet<T>>()
iter.collect::<HashMap<K,V>>()    // dari Iterator<Item=(K,V)>
iter.collect::<Result<Vec<T>,E>>()  // Vec<Result> → Result<Vec>
iter.collect::<Option<Vec<T>>>()    // Vec<Option> → Option<Vec>
```

---

## 🏆 Cabaran Akhir

Cuba implement salah satu:

1. **Currency System** — `Ringgit`, `Dollar`, `Euro` dengan From/Into antara semua, TryFrom untuk validate
2. **Color Conversion** — RGB ↔ HSL ↔ Hex string dengan From dan Display
3. **Time Units** — `Microsecond`, `Millisecond`, `Second`, `Minute`, `Hour` dengan operasi
4. **ByteSize** — `Byte`, `Kilobyte`, `Megabyte`, `Gigabyte` dengan FromStr untuk "10MB"
5. **Koordinat** — WGS84 ↔ MGRS ↔ Decimal Degrees dengan proper error handling

---

*Type conversions dalam Rust — explicit, safe, dan expressive.*
*Bila Rust paksa anda explicit, ia sebenarnya mengajar anda berfikir tentang data anda.* 🦀
