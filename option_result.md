# 🎯 Option & Result dalam Rust — Nota Lengkap & Dikembangkan

> Berdasarkan topik: Belajar Rust Dasar #12 — Option & Result
> Dikembangkan dengan contoh lebih lengkap, visual, dan Brain Teaser.
> Gaya Head First — Bahasa Malaysia

---

## Gambaran Besar — Kenapa Option & Result Wujud?

```
Masalah dalam bahasa lain:

PHP / Java:
  $pengguna = cariPengguna(99); // mungkin return null
  echo $pengguna->nama;         // Fatal Error! null pointer! 💥

  try {
      $hasil = bahagi(10, 0);
  } catch (Exception $e) {      // boleh terlupa catch!
      echo $e->getMessage();
  }

Rust penyelesaian:
  Option<T>  → "ada nilai" atau "tiada nilai" (ganti null)
  Result<T,E> → "berjaya" atau "gagal dengan sebab" (ganti exception)

  Compiler PAKSA kita handle kedua-dua kes.
  Tiada null pointer. Tiada exception yang terlupa.
```

---

## Peta Pembelajaran

```
BAHAGIAN A — Option<T>
  A1 → Apa itu Option?
  A2 → Buat Option
  A3 → Baca nilai dari Option
  A4 → Method-method Option
  A5 → Option dalam Struct & Fungsi

BAHAGIAN B — Result<T, E>
  B1 → Apa itu Result?
  B2 → Buat Result
  B3 → Handle Result
  B4 → Operator ?
  B5 → Method-method Result

BAHAGIAN C — Gabungan & Pattern
  C1 → Option + Result bersama
  C2 → Pattern idiomatik
  C3 → Mini Project
```

---

# BAHAGIAN A — Option\<T\> 🔍

## A1: Apa Itu Option?

```
Option<T> adalah enum dalam standard library Rust:

  enum Option<T> {
      Some(T),   // ada nilai bertipe T
      None,      // tiada nilai
  }

Analogi dunia nyata:
  Anda order makanan di kedai:
    Some("Nasi Lemak") → kedai ada stok, dapat
    None               → kedai habis stok, tak dapat

Berbanding NULL:
  PHP:  $x = null;       → tiada perlindungan!
  Rust: let x: Option<i32> = None; → MESTI handle!
```

---

## A2: Buat Option

```rust
fn main() {
    // Cara 1: Terus assign
    let ada: Option<i32>    = Some(42);
    let tiada: Option<i32>  = None;

    // Cara 2: Type inference — Rust tahu sendiri
    let nombor = Some(100);     // Option<i32>
    let teks   = Some("hello"); // Option<&str>

    // Cara 3: Return dari fungsi
    fn cari_indeks(v: &[i32], sasaran: i32) -> Option<usize> {
        for (i, &val) in v.iter().enumerate() {
            if val == sasaran {
                return Some(i); // jumpa → Some
            }
        }
        None // tidak jumpa → None
    }

    let nombor_list = vec![10, 20, 30, 40, 50];

    println!("{:?}", cari_indeks(&nombor_list, 30)); // Some(2)
    println!("{:?}", cari_indeks(&nombor_list, 99)); // None

    // Standard library pun gunakan Option
    let v = vec![1, 2, 3];
    println!("{:?}", v.first()); // Some(1)
    println!("{:?}", v.get(5));  // None — tiada index 5

    let s = "hello world";
    println!("{:?}", s.find('w'));  // Some(6)
    println!("{:?}", s.find('z'));  // None
}
```

---

## A3: Baca Nilai dari Option

```rust
fn main() {
    let nilai = Some(42);
    let tiada: Option<i32> = None;

    // ─── Cara 1: match (paling jelas) ────────────────────────
    match nilai {
        Some(n) => println!("Ada nilai: {}", n),
        None    => println!("Tiada nilai"),
    }

    // ─── Cara 2: if let (bila satu kes penting) ──────────────
    if let Some(n) = nilai {
        println!("Dapat: {}", n);
    }

    // if let dengan else
    if let Some(n) = tiada {
        println!("Ada: {}", n);
    } else {
        println!("Memang tiada!"); // ← ini yang print
    }

    // ─── Cara 3: let else (early return) ─────────────────────
    fn proses(opt: Option<i32>) -> String {
        let Some(n) = opt else {
            return "Tiada nilai untuk diproses".into();
        };
        format!("Nilai x 2 = {}", n * 2)
    }
    println!("{}", proses(Some(5)));  // Nilai x 2 = 10
    println!("{}", proses(None));     // Tiada nilai untuk diproses

    // ─── Cara 4: unwrap() — PANIC kalau None! ─────────────────
    let selamat = Some(10);
    println!("{}", selamat.unwrap()); // 10 — OK

    // JANGAN buat ini kalau tidak pasti ada nilai:
    // tiada.unwrap(); // ← thread 'main' panicked at 'called
    //                 //   `Option::unwrap()` on a `None` value'

    // ─── Cara 5: expect() — PANIC dengan mesej khusus ─────────
    let mesti_ada = Some("penting");
    println!("{}", mesti_ada.expect("Nilai ini sepatutnya wujud!"));

    // ─── Cara 6: unwrap_or() — nilai default kalau None ───────
    println!("{}", nilai.unwrap_or(0));  // 42 (ada nilai)
    println!("{}", tiada.unwrap_or(0)); // 0  (guna default)

    // ─── Cara 7: unwrap_or_else() — compute default lazily ────
    println!("{}", tiada.unwrap_or_else(|| {
        println!("(kira default...)");
        -1
    }));
}
```

---

## A4: Method-method Option Penting

```rust
fn main() {
    let ada:   Option<i32> = Some(10);
    let tiada: Option<i32> = None;

    // ── Semak ──────────────────────────────────────────────────
    println!("ada.is_some():  {}", ada.is_some());   // true
    println!("ada.is_none():  {}", ada.is_none());   // false
    println!("tiada.is_some(): {}", tiada.is_some()); // false
    println!("tiada.is_none(): {}", tiada.is_none()); // true

    // ── Transformasi ───────────────────────────────────────────

    // map() — transform nilai dalam Some, None kekal None
    let dua_kali = ada.map(|n| n * 2);
    println!("{:?}", dua_kali);  // Some(20)
    println!("{:?}", tiada.map(|n| n * 2)); // None

    // map() dengan return type berbeza
    let sebagai_string = ada.map(|n| format!("Nilai: {}", n));
    println!("{:?}", sebagai_string); // Some("Nilai: 10")

    // and_then() — chain operations (flatMap)
    fn sqrt_positif(n: i32) -> Option<f64> {
        if n >= 0 { Some((n as f64).sqrt()) } else { None }
    }
    println!("{:?}", ada.and_then(sqrt_positif));       // Some(3.16...)
    println!("{:?}", Some(-5).and_then(sqrt_positif));  // None
    println!("{:?}", tiada.and_then(sqrt_positif));     // None

    // filter() — None kalau condition gagal
    println!("{:?}", ada.filter(|&n| n > 5));  // Some(10) — lulus
    println!("{:?}", ada.filter(|&n| n > 50)); // None — gagal

    // or() dan or_else() — alternatif
    println!("{:?}", tiada.or(Some(99)));  // Some(99)
    println!("{:?}", ada.or(Some(99)));    // Some(10) — ada kekal

    // ── Tukar Jenis ────────────────────────────────────────────

    // ok_or() — Option → Result
    let r: Result<i32, &str> = ada.ok_or("Tiada nilai!");
    println!("{:?}", r);  // Ok(10)

    let r2: Result<i32, &str> = tiada.ok_or("Tiada nilai!");
    println!("{:?}", r2); // Err("Tiada nilai!")

    // flatten() — Option<Option<T>> → Option<T>
    let nested: Option<Option<i32>> = Some(Some(42));
    println!("{:?}", nested.flatten()); // Some(42)

    let nested2: Option<Option<i32>> = Some(None);
    println!("{:?}", nested2.flatten()); // None

    // zip() — gabung dua Option
    let a = Some(1);
    let b = Some("satu");
    println!("{:?}", a.zip(b));       // Some((1, "satu"))
    println!("{:?}", a.zip(tiada));   // None

    // unzip() — pisah Vec<(A, B)>
    let pasangan: Vec<(i32, &str)> = vec![(1,"a"), (2,"b")];
    let (kiri, kanan): (Vec<i32>, Vec<&str>) = pasangan.into_iter().unzip();
    println!("{:?} {:?}", kiri, kanan);
}
```

---

## A5: Option dalam Struct & Fungsi

```rust
// Field optional dalam struct
#[derive(Debug)]
struct Pengguna {
    nama:      String,
    emel:      String,
    no_telefon: Option<String>,  // mungkin ada, mungkin tidak
    bio:       Option<String>,
    umur:      Option<u8>,
}

impl Pengguna {
    fn baru(nama: &str, emel: &str) -> Self {
        Pengguna {
            nama:       nama.into(),
            emel:       emel.into(),
            no_telefon: None,
            bio:        None,
            umur:       None,
        }
    }

    fn dengan_telefon(mut self, no: &str) -> Self {
        self.no_telefon = Some(no.into());
        self
    }

    fn dengan_bio(mut self, bio: &str) -> Self {
        self.bio = Some(bio.into());
        self
    }

    // Method yang return Option
    fn format_telefon(&self) -> Option<String> {
        self.no_telefon.as_ref().map(|no| {
            format!("📞 {}", no)
        })
    }

    // Guna option dalam logik
    fn boleh_layak_pengundi(&self) -> bool {
        self.umur.map_or(false, |u| u >= 21)
    }

    fn papar(&self) {
        println!("Nama:  {}", self.nama);
        println!("Emel:  {}", self.emel);

        // Cara elegaan handle Option dalam output
        if let Some(ref tel) = self.no_telefon {
            println!("Tel:   {}", tel);
        } else {
            println!("Tel:   (tiada)");
        }

        println!("Bio:   {}", self.bio.as_deref().unwrap_or("(tiada)"));
        println!("Umur:  {}", self.umur.map_or("(tiada)".into(),
                              |u| format!("{} tahun", u)));
    }
}

fn cari_pengguna(id: u32) -> Option<Pengguna> {
    match id {
        1 => Some(
            Pengguna::baru("Ali Ahmad", "ali@email.com")
                .dengan_telefon("012-345-6789")
                .dengan_bio("Developer Rust dari Malaysia")
        ),
        2 => Some(Pengguna::baru("Siti Hawa", "siti@email.com")),
        _ => None, // ID tidak wujud
    }
}

fn main() {
    // Cari ID yang ada
    match cari_pengguna(1) {
        Some(p) => p.papar(),
        None    => println!("Pengguna tidak dijumpai"),
    }

    println!();

    // Cari ID yang tidak ada
    match cari_pengguna(99) {
        Some(p) => p.papar(),
        None    => println!("Pengguna ID 99 tidak dijumpai"),
    }

    // Chaining method pada Option
    let nama_pengguna: Option<String> = cari_pengguna(1)
        .map(|p| p.nama.to_uppercase());
    println!("\nNama uppercase: {:?}", nama_pengguna);

    let telefon_format: Option<String> = cari_pengguna(1)
        .and_then(|p| p.format_telefon());
    println!("Telefon format: {:?}", telefon_format);
}
```

---

## 🧠 Brain Teaser A

Apakah output kod ini? Terangkan kenapa.

```rust
fn main() {
    let a: Option<i32> = Some(0);
    let b: Option<i32> = None;

    // 1
    println!("{:?}", a.map(|n| n + 1));

    // 2
    println!("{:?}", b.map(|n| n + 1));

    // 3
    println!("{}", a.unwrap_or(99));

    // 4
    println!("{}", b.unwrap_or(99));

    // 5
    println!("{:?}", a.filter(|&n| n > 0));

    // 6
    println!("{:?}", a.filter(|&n| n == 0));
}
```

<details>
<summary>👀 Jawapan</summary>

```
1. Some(1)     → Some(0).map(n+1) = Some(1)
2. None        → None.map(apapun) = None (map tidak buat apa pada None)
3. 0           → Some(0).unwrap_or(99) = 0 (ada nilai, guna nilai tersebut)
4. 99          → None.unwrap_or(99) = 99 (tiada nilai, guna default)
5. None        → Some(0).filter(n>0) = None (0 tidak > 0, jadi None)
6. Some(0)     → Some(0).filter(n==0) = Some(0) (0 == 0, lulus filter)
```

**Insight penting:** `Some(0)` BUKAN sama dengan `None`! `0` adalah nilai yang sah. `None` bermaksud "tiada nilai langsung". Ini berbeza dari null dalam bahasa lain di mana `0` dan `null` kadang diperlakukan sama.
</details>

---

# BAHAGIAN B — Result\<T, E\> ⚠️

## B1: Apa Itu Result?

```
Result<T, E> adalah enum dalam standard library Rust:

  enum Result<T, E> {
      Ok(T),   // berjaya — ada nilai bertipe T
      Err(E),  // gagal   — ada error bertipe E
  }

Analogi dunia nyata:
  Anda hantar borang permohonan:
    Ok(NoSurat) → permohonan berjaya, dapat nombor surat
    Err(Sebab)  → permohonan ditolak, ada sebab penolakan

Berbanding Exception:
  PHP:  throw new Exception("Gagal"); // boleh terlupa catch
  Rust: return Err("Gagal")           // MESTI handle
```

---

## B2: Buat Result

```rust
use std::num::ParseIntError;

// Fungsi yang mungkin gagal — return Result!
fn bahagi(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err("Tidak boleh bahagi dengan sifar!".to_string())
    } else {
        Ok(a / b)
    }
}

fn parse_umur(input: &str) -> Result<u8, String> {
    // parse() return Result<u8, ParseIntError>
    let umur: u8 = input.trim().parse()
        .map_err(|_| format!("'{}' bukan nombor sah", input))?;

    if umur > 150 {
        return Err(format!("Umur {} tidak munasabah", umur));
    }
    Ok(umur)
}

fn main() {
    // Result dari fungsi kita
    println!("{:?}", bahagi(10.0, 2.0));  // Ok(5.0)
    println!("{:?}", bahagi(10.0, 0.0));  // Err("Tidak boleh bahagi...")

    // Result dari standard library
    let ok:  Result<i32, _> = "42".parse();
    let err: Result<i32, _> = "abc".parse();
    println!("{:?}", ok);   // Ok(42)
    println!("{:?}", err);  // Err(ParseIntError { ... })

    // Result kita
    println!("{:?}", parse_umur("25"));   // Ok(25)
    println!("{:?}", parse_umur("abc"));  // Err("'abc' bukan nombor sah")
    println!("{:?}", parse_umur("200"));  // Err("Umur 200 tidak munasabah")

    // Dalam std library — banyak return Result
    use std::fs;
    let r = fs::read_to_string("tiada.txt");
    println!("Fail: {:?}", r); // Err(Os { code: 2, ... })
}
```

---

## B3: Handle Result

```rust
use std::fs;

fn main() {
    // ─── Cara 1: match ─────────────────────────────────────────
    match fs::read_to_string("test.txt") {
        Ok(kandungan) => println!("Kandungan: {}", kandungan),
        Err(e)        => println!("Gagal baca fail: {}", e),
    }

    // ─── Cara 2: if let ────────────────────────────────────────
    if let Ok(n) = "42".parse::<i32>() {
        println!("Parse berjaya: {}", n);
    }

    if let Err(e) = "abc".parse::<i32>() {
        println!("Parse gagal: {}", e);
    }

    // ─── Cara 3: unwrap() — PANIC kalau Err ───────────────────
    let n = "42".parse::<i32>().unwrap(); // OK
    println!("{}", n);
    // "abc".parse::<i32>().unwrap(); // ← PANIC!

    // ─── Cara 4: expect() — PANIC dengan mesej ────────────────
    let n2 = "42".parse::<i32>().expect("Input mestilah nombor bulat");
    println!("{}", n2);

    // ─── Cara 5: unwrap_or() — nilai default ──────────────────
    let n3 = "abc".parse::<i32>().unwrap_or(0);
    println!("{}", n3); // 0 (guna default)

    // ─── Cara 6: unwrap_or_else() ─────────────────────────────
    let n4 = "abc".parse::<i32>().unwrap_or_else(|e| {
        println!("Parse gagal: {}, guna 0", e);
        0
    });
    println!("{}", n4);

    // ─── Cara 7: is_ok() / is_err() ───────────────────────────
    let r: Result<i32, _> = "100".parse();
    if r.is_ok() {
        println!("OK! nilai = {}", r.unwrap());
    }

    // ─── Cara 8: let else ──────────────────────────────────────
    fn proses(input: &str) -> String {
        let Ok(n) = input.parse::<i32>() else {
            return format!("'{}' bukan nombor", input);
        };
        format!("Nombor dikuasa: {}", n * n)
    }

    println!("{}", proses("5"));    // Nombor dikuasa: 25
    println!("{}", proses("hello")); // 'hello' bukan nombor
}
```

---

## B4: Operator ? — Cara Idiomatik

```rust
use std::fs;
use std::num::ParseIntError;

// Tanpa ? — verbose!
fn baca_nombor_lama(path: &str) -> Result<i32, String> {
    let kandungan = match fs::read_to_string(path) {
        Ok(s)  => s,
        Err(e) => return Err(e.to_string()),
    };

    let n = match kandungan.trim().parse::<i32>() {
        Ok(n)  => n,
        Err(e) => return Err(e.to_string()),
    };

    Ok(n * 2)
}

// Dengan ? — bersih dan idiomatik!
fn baca_nombor(path: &str) -> Result<i32, String> {
    let kandungan = fs::read_to_string(path)
        .map_err(|e| e.to_string())?; // ? = return Err kalau gagal

    let n = kandungan.trim().parse::<i32>()
        .map_err(|e| e.to_string())?;

    Ok(n * 2) // kalau semua OK, return ini
}

// ? dengan custom error
#[derive(Debug)]
enum AppError {
    Io(std::io::Error),
    Parse(ParseIntError),
}

impl From<std::io::Error> for AppError {
    fn from(e: std::io::Error) -> Self { AppError::Io(e) }
}

impl From<ParseIntError> for AppError {
    fn from(e: ParseIntError) -> Self { AppError::Parse(e) }
}

fn baca_nombor_typed(path: &str) -> Result<i32, AppError> {
    let kandungan = fs::read_to_string(path)?;  // auto convert → AppError::Io
    let n = kandungan.trim().parse::<i32>()?;    // auto convert → AppError::Parse
    Ok(n * 2)
}

// ? dalam main() — main boleh return Result!
fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Buat fail sementara untuk demo
    fs::write("nombor.txt", "21")?;

    let n = baca_nombor("nombor.txt")?;
    println!("Nombor x2 = {}", n); // 42

    fs::remove_file("nombor.txt")?;

    println!("Semua berjaya!");
    Ok(()) // main kena return Ok(())
}
```

### Cara ? Berfungsi

```
fn baca() -> Result<i32, MyError> {
    let n = parse("abc")?;
    //                  ↑
    //  Ini sama dengan:
    //  let n = match parse("abc") {
    //      Ok(v)  => v,
    //      Err(e) => return Err(e.into()), // .into() = convert error type!
    //  };

    Ok(n)
}

? BUAT DUA PERKARA:
  1. Kalau Ok → ambil nilai, teruskan
  2. Kalau Err → convert error (via From trait) dan RETURN terus
```

---

## B5: Method-method Result Penting

```rust
fn main() {
    let ok:  Result<i32, &str> = Ok(42);
    let err: Result<i32, &str> = Err("rosak");

    // ── Semak ──────────────────────────────────────────────────
    println!("ok.is_ok():   {}", ok.is_ok());   // true
    println!("ok.is_err():  {}", ok.is_err());  // false
    println!("err.is_ok():  {}", err.is_ok());  // false
    println!("err.is_err(): {}", err.is_err()); // true

    // ── Transformasi ───────────────────────────────────────────

    // map() — transform nilai Ok, Err kekal
    println!("{:?}", ok.map(|n| n * 2));   // Ok(84)
    println!("{:?}", err.map(|n| n * 2));  // Err("rosak") — tidak berubah

    // map_err() — transform nilai Err, Ok kekal
    println!("{:?}", ok.map_err(|e| format!("ERROR: {}", e)));
    // Ok(42) — tidak berubah
    println!("{:?}", err.map_err(|e| format!("ERROR: {}", e)));
    // Err("ERROR: rosak")

    // and_then() — chain (flatMap)
    fn semak_positif(n: i32) -> Result<i32, &'static str> {
        if n > 0 { Ok(n) } else { Err("Mesti positif!") }
    }

    println!("{:?}", ok.and_then(semak_positif));  // Ok(42)
    println!("{:?}", Ok(-5).and_then(semak_positif)); // Err("Mesti positif!")
    println!("{:?}", err.and_then(semak_positif)); // Err("rosak") — tidak sample

    // or_else() — recover dari Err
    println!("{:?}", err.or_else(|_| Ok::<i32, &str>(0))); // Ok(0)
    println!("{:?}", ok.or_else(|_| Ok::<i32, &str>(0)));  // Ok(42)

    // ── Tukar Jenis ────────────────────────────────────────────

    // ok() — Result → Option (buang error info)
    println!("{:?}", ok.ok());   // Some(42)
    println!("{:?}", err.ok());  // None

    // err() — Result → Option<E> (ambil error)
    println!("{:?}", ok.err());  // None
    println!("{:?}", err.err()); // Some("rosak")

    // ── Collect Vec<Result<T,E>> → Result<Vec<T>,E> ───────────
    let strings = vec!["1", "2", "3"];
    let nombor: Result<Vec<i32>, _> = strings.iter()
        .map(|s| s.parse::<i32>())
        .collect();
    println!("{:?}", nombor); // Ok([1, 2, 3])

    let strings2 = vec!["1", "abc", "3"];
    let nombor2: Result<Vec<i32>, _> = strings2.iter()
        .map(|s| s.parse::<i32>())
        .collect();
    println!("{:?}", nombor2); // Err(ParseIntError)
}
```

---

## 🧠 Brain Teaser B

Apakah output dan apakah yang berlaku pada setiap baris?

```rust
fn main() {
    let r1: Result<i32, &str> = Ok(5);
    let r2: Result<i32, &str> = Err("gagal");

    // A
    let a = r1.map(|n| n * 10).unwrap_or(0);
    println!("A: {}", a);

    // B
    let b = r2.map(|n| n * 10).unwrap_or(0);
    println!("B: {}", b);

    // C
    let c = r2.map_err(|e| format!("[ERR] {}", e)).unwrap_or(-1);
    println!("C: {}", c);

    // D
    let d: Result<i32, &str> = r1.and_then(|n| {
        if n > 3 { Ok(n * 2) } else { Err("terlalu kecil") }
    });
    println!("D: {:?}", d);

    // E
    let e = r2.or(Ok(99));
    println!("E: {:?}", e);
}
```

<details>
<summary>👀 Jawapan</summary>

```
A: 50    → Ok(5).map(n*10) = Ok(50), .unwrap_or(0) = 50
B: 0     → Err("gagal").map(...) = Err("gagal"), .unwrap_or(0) = 0
C: -1    → Err("gagal").map_err(...) = Err("[ERR] gagal"), .unwrap_or(-1) = -1
D: Ok(10) → Ok(5).and_then(n>3 → Ok(n*2)) = Ok(10) (5>3 lulus)
E: Ok(99) → Err("gagal").or(Ok(99)) = Ok(99) (recover dari Err)
```

**Ingat:**
- `map()` hanya transform `Ok`, biarkan `Err`
- `map_err()` hanya transform `Err`, biarkan `Ok`
- `and_then()` chain pada `Ok`, bypass pada `Err`
- `or()` guna alternatif pada `Err`, bypass pada `Ok`
</details>

---

# BAHAGIAN C — Gabungan & Pattern 🔗

## C1: Option + Result Bersama

```rust
// Tukar antara Option dan Result
fn main() {
    let opt: Option<i32> = Some(42);
    let none: Option<i32> = None;

    // Option → Result dengan ok_or()
    let r1: Result<i32, &str> = opt.ok_or("Tiada nilai!");
    let r2: Result<i32, &str> = none.ok_or("Tiada nilai!");
    println!("{:?}", r1); // Ok(42)
    println!("{:?}", r2); // Err("Tiada nilai!")

    // Option → Result dengan ok_or_else()
    let r3: Result<i32, String> = none.ok_or_else(|| {
        format!("Gagal pada {}", "runtime")
    });
    println!("{:?}", r3); // Err("Gagal pada runtime")

    // Result → Option dengan ok()
    let res: Result<i32, &str> = Ok(10);
    let err: Result<i32, &str> = Err("salah");
    println!("{:?}", res.ok()); // Some(10)
    println!("{:?}", err.ok()); // None — info error hilang!

    // Result → Option dengan err()
    println!("{:?}", err.err()); // Some("salah")
    println!("{:?}", res.err()); // None

    // Transpose — Option<Result<T,E>> ↔ Result<Option<T>,E>
    let a: Option<Result<i32, &str>> = Some(Ok(5));
    let b: Result<Option<i32>, &str> = a.transpose();
    println!("{:?}", b); // Ok(Some(5))

    let c: Option<Result<i32, &str>> = Some(Err("ops"));
    let d: Result<Option<i32>, &str> = c.transpose();
    println!("{:?}", d); // Err("ops")
}
```

---

## C2: Pattern Idiomatik

```rust
use std::collections::HashMap;

// Pattern 1: Chaining ? dengan berbilang operasi
fn proses_fail(path: &str) -> Result<Vec<i32>, Box<dyn std::error::Error>> {
    let kandungan = std::fs::read_to_string(path)?;

    let nombor: Result<Vec<i32>, _> = kandungan
        .lines()
        .map(|baris| baris.trim().parse::<i32>())
        .collect();

    Ok(nombor?)
}

// Pattern 2: Option chaining
fn dapatkan_bandar(pengguna: &HashMap<&str, HashMap<&str, &str>>, nama: &str) -> Option<String> {
    pengguna
        .get(nama)?              // Option<HashMap>
        .get("alamat")?          // Option<&str>
        .split(',')
        .nth(1)                  // Option<&str>
        .map(|s| s.trim().to_string()) // transform
}

// Pattern 3: filter_map — skip error, ambil yang berjaya
fn parse_semua_yang_boleh(input: &[&str]) -> Vec<i32> {
    input.iter()
        .filter_map(|s| s.parse::<i32>().ok()) // ok() = Result → Option
        .collect()
}

// Pattern 4: partition — pisahkan Ok dan Err
fn parse_dengan_laporan(input: &[&str]) -> (Vec<i32>, Vec<String>) {
    let (ok_vec, err_vec): (Vec<_>, Vec<_>) = input.iter()
        .map(|s| s.parse::<i32>().map_err(|e| format!("'{}': {}", s, e)))
        .partition(Result::is_ok);

    let berjaya: Vec<i32> = ok_vec.into_iter().map(Result::unwrap).collect();
    let gagal: Vec<String> = err_vec.into_iter().map(Result::unwrap_err).collect();
    (berjaya, gagal)
}

fn main() {
    // Pattern 3
    let input = ["1", "dua", "3", "empat", "5"];
    let nombor = parse_semua_yang_boleh(&input);
    println!("Parse berjaya: {:?}", nombor); // [1, 3, 5]

    // Pattern 4
    let (berjaya, gagal) = parse_dengan_laporan(&input);
    println!("OK:  {:?}", berjaya);  // [1, 3, 5]
    println!("ERR: {:?}", gagal);    // ["'dua': ...", "'empat': ..."]
}
```

---

## C3: Perbandingan — Bila Guna Option vs Result

```
GUNA Option<T> bila:
  ✔ "Ada nilai" atau "tiada nilai"
  ✔ Tiada maklumat kenapa tiada nilai
  ✔ Contoh:
      - Cari dalam senarai (jumpa atau tidak jumpa)
      - Field struct yang optional
      - Elemen pertama dalam Vec kosong
      - Ambil dari HashMap (ada key atau tidak)

  fn cari(senarai: &[i32], cari: i32) -> Option<usize>
  fn ambil_elemen(v: &Vec<T>, i: usize) -> Option<&T>
  struct Pengguna { telefon: Option<String> }

GUNA Result<T, E> bila:
  ✔ Operasi yang boleh berjaya atau gagal
  ✔ Perlu tahu KENAPA gagal
  ✔ Contoh:
      - Baca/tulis fail
      - Parse string ke nombor
      - Network request
      - Database query
      - Validate input pengguna

  fn baca_fail(path: &str) -> Result<String, io::Error>
  fn parse(s: &str) -> Result<i32, ParseError>
  fn simpan_db(data: &Data) -> Result<(), DbError>

MUDAH INGAT:
  "Ada atau tiada?"        → Option
  "Berjaya atau gagal?"    → Result
```

---

## Mini Project: Sistem Markah Pelajar 🎓

```rust
use std::collections::HashMap;

// ─── Types ────────────────────────────────────────────────────

#[derive(Debug, Clone)]
struct Pelajar {
    nama:   String,
    markah: Vec<f64>,
}

#[derive(Debug)]
enum MarkahError {
    PelajarTidakWujud(String),
    TiadaMarkah(String),
    MarkahTidakSah { nilai: f64 },
}

impl std::fmt::Display for MarkahError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            MarkahError::PelajarTidakWujud(nama) =>
                write!(f, "Pelajar '{}' tidak wujud", nama),
            MarkahError::TiadaMarkah(nama) =>
                write!(f, "Pelajar '{}' tiada markah lagi", nama),
            MarkahError::MarkahTidakSah { nilai } =>
                write!(f, "Markah {} tidak sah (0.0 - 100.0)", nilai),
        }
    }
}

// ─── Sistem ───────────────────────────────────────────────────

struct SistemMarkah {
    pelajar: HashMap<String, Pelajar>,
}

impl SistemMarkah {
    fn baru() -> Self {
        SistemMarkah { pelajar: HashMap::new() }
    }

    fn daftar(&mut self, nama: &str) {
        self.pelajar.insert(nama.into(), Pelajar {
            nama:   nama.into(),
            markah: Vec::new(),
        });
        println!("✔ Daftar pelajar: {}", nama);
    }

    // Kembalikan Option — pelajar mungkin ada atau tidak
    fn cari(&self, nama: &str) -> Option<&Pelajar> {
        self.pelajar.get(nama)
    }

    // Kembalikan Result — rekod markah boleh gagal dengan sebab
    fn rekod_markah(&mut self, nama: &str, markah: f64) -> Result<(), MarkahError> {
        // Validate markah
        if markah < 0.0 || markah > 100.0 {
            return Err(MarkahError::MarkahTidakSah { nilai: markah });
        }

        // Cari pelajar — kalau tidak ada, return Err
        let pelajar = self.pelajar.get_mut(nama)
            .ok_or_else(|| MarkahError::PelajarTidakWujud(nama.into()))?;

        pelajar.markah.push(markah);
        println!("✔ Rekod markah {:.1} untuk {}", markah, nama);
        Ok(())
    }

    // Result kerana mungkin tiada pelajar atau tiada markah
    fn purata(&self, nama: &str) -> Result<f64, MarkahError> {
        let pelajar = self.pelajar.get(nama)
            .ok_or_else(|| MarkahError::PelajarTidakWujud(nama.into()))?;

        if pelajar.markah.is_empty() {
            return Err(MarkahError::TiadaMarkah(nama.into()));
        }

        let jumlah: f64 = pelajar.markah.iter().sum();
        Ok(jumlah / pelajar.markah.len() as f64)
    }

    // Option kerana markah mungkin kosong, tiada "error" per se
    fn markah_tertinggi(&self, nama: &str) -> Option<f64> {
        self.pelajar
            .get(nama)?                      // Option<&Pelajar>
            .markah
            .iter()
            .cloned()
            .reduce(f64::max)               // Option<f64>
    }

    // Gred berdasarkan markah — Option kerana markah mungkin tiada
    fn gred(&self, nama: &str) -> Option<char> {
        let purata = self.purata(nama).ok()?; // Result → Option

        let gred = match purata as u32 {
            90..=100 => 'A',
            75..=89  => 'B',
            60..=74  => 'C',
            50..=59  => 'D',
            _        => 'E',
        };
        Some(gred)
    }

    fn laporan(&self) {
        println!("\n{'═'*50}");
        println!("{:^50}", "LAPORAN MARKAH PELAJAR");
        println!("{'═'*50}");
        println!("{:<20} {:>8} {:>8} {:>6}", "Nama", "Purata", "Tertinggi", "Gred");
        println!("{'─'*50}");

        let mut nama_senarai: Vec<&String> = self.pelajar.keys().collect();
        nama_senarai.sort();

        for nama in nama_senarai {
            let purata = self.purata(nama)
                .map(|p| format!("{:.1}", p))
                .unwrap_or("N/A".into());

            let tertinggi = self.markah_tertinggi(nama)
                .map(|m| format!("{:.1}", m))
                .unwrap_or("N/A".into());

            let gred = self.gred(nama)
                .map(|g| g.to_string())
                .unwrap_or("?".into());

            println!("{:<20} {:>8} {:>8} {:>6}", nama, purata, tertinggi, gred);
        }
        println!("{'═'*50}");
    }
}

// ─── Main ──────────────────────────────────────────────────────

fn main() {
    let mut sistem = SistemMarkah::baru();

    // Daftar pelajar
    println!("=== Daftar Pelajar ===");
    sistem.daftar("Ali Ahmad");
    sistem.daftar("Siti Hawa");
    sistem.daftar("Amin Razak");

    println!("\n=== Rekod Markah ===");

    // Rekod markah - guna match untuk handle Result
    let markah_ali = vec![85.0, 92.0, 78.0, 90.0, 88.0];
    for m in markah_ali {
        match sistem.rekod_markah("Ali Ahmad", m) {
            Ok(())  => {},
            Err(e)  => println!("  ✘ {}", e),
        }
    }

    // Markah siti - ada yang tidak sah!
    let markah_siti = vec![75.0, 105.0, 80.0, -5.0, 70.0];
    for m in markah_siti {
        match sistem.rekod_markah("Siti Hawa", m) {
            Ok(())  => {},
            Err(e)  => println!("  ✘ {}", e),
        }
    }

    // Pelajar yang tidak wujud
    match sistem.rekod_markah("Zara Lina", 95.0) {
        Ok(())  => {},
        Err(e)  => println!("  ✘ {}", e),
    }

    println!("\n=== Cari Pelajar (Option) ===");

    // cari() return Option
    match sistem.cari("Ali Ahmad") {
        Some(p) => println!("  Jumpa: {} ({} markah)", p.nama, p.markah.len()),
        None    => println!("  Tidak jumpa"),
    }

    match sistem.cari("Orang Tidak Ada") {
        Some(p) => println!("  Jumpa: {}", p.nama),
        None    => println!("  Tidak jumpa (dijangka)"),
    }

    println!("\n=== Purata & Gred (Result + Option) ===");

    // Pelajar ada markah
    match sistem.purata("Ali Ahmad") {
        Ok(p)  => println!("  Ali Ahmad purata: {:.1}", p),
        Err(e) => println!("  Error: {}", e),
    }

    // Pelajar tiada markah
    match sistem.purata("Amin Razak") {
        Ok(p)  => println!("  Amin Razak purata: {:.1}", p),
        Err(e) => println!("  Error: {}", e), // TiadaMarkah
    }

    // Pelajar tidak wujud
    match sistem.purata("Hantu") {
        Ok(p)  => println!("  Hantu purata: {:.1}", p),
        Err(e) => println!("  Error: {}", e), // PelajarTidakWujud
    }

    // Gred dengan Option chaining
    println!("\n=== Gred ===");
    for nama in &["Ali Ahmad", "Siti Hawa", "Amin Razak", "Hantu"] {
        let gred_str = sistem.gred(nama)
            .map(|g| format!("Gred {}", g))
            .unwrap_or_else(|| "Tiada gred (markah kosong/pelajar tiada)".into());
        println!("  {}: {}", nama, gred_str);
    }

    // Laporan keseluruhan
    sistem.laporan();
}
```

---

# 📋 Rujukan Pantas

## Option\<T\>

```rust
// Buat
Some(nilai)           // ada nilai
None                  // tiada nilai

// Semak
opt.is_some()         // → bool
opt.is_none()         // → bool

// Ambil nilai (hati-hati!)
opt.unwrap()          // T atau PANIC
opt.expect("msg")     // T atau PANIC dengan mesej
opt.unwrap_or(def)    // T atau nilai default
opt.unwrap_or_else(f) // T atau compute default

// Transform
opt.map(|v| ...)      // Some(f(v)) atau None
opt.and_then(|v| ...) // chain operations
opt.filter(|v| ...)   // None kalau condition gagal
opt.or(other)         // kalau None, guna other
opt.or_else(|| ...)   // kalau None, compute other

// Tukar jenis
opt.ok_or(err)        // Option → Result
opt.ok_or_else(|| err)
opt.flatten()         // Option<Option<T>> → Option<T>
opt.zip(other)        // (Some, Some) → Some((a,b))
```

## Result\<T, E\>

```rust
// Buat
Ok(nilai)             // berjaya
Err(ralat)            // gagal

// Semak
res.is_ok()           // → bool
res.is_err()          // → bool

// Ambil nilai (hati-hati!)
res.unwrap()          // T atau PANIC
res.expect("msg")     // T atau PANIC dengan mesej
res.unwrap_or(def)    // T atau nilai default
res.unwrap_or_else(f) // T atau compute default

// Transform
res.map(|v| ...)      // transform Ok
res.map_err(|e| ...)  // transform Err
res.and_then(|v| ...) // chain operations
res.or_else(|e| ...)  // recover dari Err

// Tukar jenis
res.ok()              // Result → Option<T>
res.err()             // Result → Option<E>

// Propagate error — GUNA INI!
let val = fungsi()?;  // return Err terus kalau gagal
```

## Perbandingan Pantas

```
                Option<T>     Result<T,E>
Ada nilai:       Some(v)        Ok(v)
Tiada/Gagal:     None           Err(e)
Tukar:           ok_or(e)       ok()
? operator:      ✔ (dalam fn    ✔ (dalam fn
                 return Option) return Result)
```

## Bila Guna Apa?

```
Option → "ada atau tiada?" — cari, field optional
Result → "berjaya atau gagal?" — I/O, parse, validate
```

---

*Option & Result — asas Rust untuk nilai selamat tanpa null dan exception. Selamat belajar!* 🦀
