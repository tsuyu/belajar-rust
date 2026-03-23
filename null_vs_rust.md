# 🚫 Null vs Rust — "The Billion Dollar Mistake"

> "I call it my billion-dollar mistake. It was the invention of the null reference."
> — Tony Hoare, pencipta null, 1965
>
> Rust tidak ada null. Tapi ia ada sesuatu yang jauh lebih baik.
> Faham kenapa, dan cara Rust handle "tiada nilai".

---

## Masalah Null

```
Bahasa lain: null boleh muncul di MANA-MANA, BILA-BILA MASA

Java:
  String nama = pekerja.getNama();
  int panjang = nama.length(); // NullPointerException! 💥
  // Siapa tahu nama boleh null? Programmer kena "ingat"

PHP:
  $nilai = $data['kunci']; // Warning: Undefined index
  echo strlen($nilai);     // Fatal error: null passed to strlen

JavaScript:
  const n = obj.pekerja.nama; // Cannot read property 'nama' of undefined
  console.log(n.toUpperCase()); // TypeError: null.toUpperCase

C:
  char* s = cari_nama(1001);
  printf("%d", strlen(s)); // Segmentation fault — s adalah NULL
  // Program CRASH! OS terpaksa bunuh proses!

Masalah utama:
  ✗ Null boleh ada DI MANA-MANA (setiap reference boleh null)
  ✗ Tidak ada cara tahu tanpa dokumentasi atau pengalaman
  ✗ Runtime error — crash semasa production!
  ✗ Sukar debug — crash jauh dari punca sebenar
  ✗ "Billion dollar" dalam kos debugging dan downtime
```

---

## Penyelesaian Rust: Option\<T\>

```rust
// Rust TIADA null.
// Sebaliknya ada Option<T>:

enum Option<T> {
    Some(T),  // Ada nilai
    None,     // Tiada nilai
}

// Perbezaan KRITIKAL:
//   Null: boleh ada pada mana-mana jenis, tidak nampak pada type
//   Option: NAMPAK pada type — terpaksa handle!

// Contoh:
let nama: String = "Ali".into();        // PASTI ada nilai
let nama: Option<String> = Some("Ali".into()); // MUNGKIN ada
let tiada: Option<String> = None;              // Tiada nilai

// Kalau fungsi return String → PASTI ada nilai
// Kalau fungsi return Option<String> → MUNGKIN tiada

// Compiler akan PAKSA anda handle None!
```

---

## Peta Pembelajaran

```
Bab 1  → Kenapa Null Bahaya
Bab 2  → Option<T> — Pengganti Null yang Selamat
Bab 3  → Cara Handle Option
Bab 4  → Option Methods yang Penting
Bab 5  → Option dalam Struct & Fungsi
Bab 6  → Option Chaining
Bab 7  → Option vs Result — Bila Guna Mana
Bab 8  → Pattern Dunia Sebenar
Bab 9  → Anti-patterns — Jangan Buat Ini
Bab 10 → Mini Project: Perbandingan PHP vs Rust
```

---

# BAB 1: Kenapa Null Bahaya 💣

## Null Safety dalam Bahasa Lain

```java
// Java — NullPointerException (NPE) adalah bug paling biasa
public class Contoh {
    public static void main(String[] args) {
        String nama = null;
        System.out.println(nama.length()); // NPE! Crash!

        // Defensive programming yang melelahkan:
        if (nama != null) {
            System.out.println(nama.length()); // pelik dan verbose
        }

        // Masalah: kena check null DI MANA-MANA
        // Kalau lupa satu → production crash
    }
}
```

```php
// PHP — null boleh datang dari mana-mana
function dapatPekerja($id) {
    // Mungkin return null kalau tidak jumpa!
    return $this->db->find($id); // null?
}

$p = dapatPekerja(1001);
echo $p->getNama(); // Fatal error: Call to a member on null
echo strlen($p['nama']); // Warning: Argument must be string
```

```javascript
// JavaScript — undefined dan null berbeza tapi sama bahaya
function ambilNama(pengguna) {
    return pengguna.profil.nama; // TypeError kalau profil undefined
}

// Optional chaining (baru dalam JS, konsep dari bahasa lain):
const nama = pengguna?.profil?.nama; // Returns undefined kalau tidak ada
// Tapi masih mungkin guna undefined tanpa sedar!
```

---

## 🧠 Brain Teaser #1

Bahasa mana yang akan crash dan bahasa mana yang selamat?

```rust
// Rust
fn cari_pekerja(id: u32) -> Option<String> {
    if id == 1 { Some("Ali".into()) } else { None }
}

fn main() {
    let nama = cari_pekerja(99);
    println!("{}", nama); // Compile?
}
```

<details>
<summary>👀 Jawapan</summary>

```
RUST: TIDAK COMPILE! (compile error, bukan runtime crash)

Error:
  error[E0277]: `Option<String>` doesn't implement `Display`
  Tidak boleh println! secara terus Option<String>!
  Compiler PAKSA anda handle None!

  // Betul:
  match cari_pekerja(99) {
      Some(nama) => println!("{}", nama),
      None       => println!("Tidak dijumpai"),
  }

  // Atau:
  if let Some(nama) = cari_pekerja(99) {
      println!("{}", nama);
  }

JAVA: Compile OK, tapi crash bila run:
  String nama = carіPekerja(99); // return null
  System.out.println(nama);       // "null" (print perkataan null)
  nama.length();                  // NullPointerException! Crash!

PERBEZAAN FUNDAMENTAL:
  Null: Error semasa RUNTIME (production crash)
  Option: Error semasa COMPILE (sebelum deploy!)

  Rust: "Saya tidak boleh compile ini sampai anda handle None"
  Java: "Okay compile, tapi good luck dalam production 😈"
```
</details>

---

# BAB 2: Option\<T\> — Pengganti Null yang Selamat ✅

## Definisi dan Konsep

```rust
// Option<T> adalah enum standard Rust:
// (sudah dalam prelude — tidak perlu import!)

// enum Option<T> {
//     Some(T),
//     None,
// }

// ── Buat Option ───────────────────────────────────────────────
let ada:   Option<i32>    = Some(42);
let tiada: Option<i32>    = None;
let teks:  Option<String> = Some("hello".into());
let kosong: Option<String> = None;

// ── Kenapa lebih baik dari null? ──────────────────────────────

// 1. TYPE SYSTEM encode kemungkinan "tiada"
fn cari_gaji(id: u32) -> f64 {
    // Return type f64 → PASTI ada nilai
    4500.0
}

fn cari_bonus(id: u32) -> Option<f64> {
    // Return type Option<f64> → MUNGKIN tiada bonus
    if id == 1 { Some(500.0) } else { None }
}

// 2. Compiler PAKSA handle
fn main() {
    let gaji  = cari_gaji(1);
    let bonus = cari_bonus(1);

    // gaji: boleh terus guna (pasti ada)
    println!("Gaji: {}", gaji);

    // bonus: MESTI handle
    // println!("Bonus: {}", bonus); // ERROR! Tidak compile!

    // Betul:
    if let Some(b) = bonus {
        println!("Bonus: {}", b);
    } else {
        println!("Tiada bonus");
    }
}

// 3. Dokumentasi yang hidup (self-documenting)
struct Pekerja {
    nama:         String,           // PASTI ada
    bahagian:     String,           // PASTI ada
    emel:         Option<String>,   // MUNGKIN tiada (baru join)
    no_telefon:   Option<String>,   // MUNGKIN tiada
    tarikh_keluar: Option<String>,  // MUNGKIN tiada (masih bekerja)
}
// Tidak perlu baca dokumentasi — type sudah cakap!
```

---

## Option dalam Standard Library

```rust
// Option digunakan MELUAS dalam Rust stdlib

// HashMap::get → Option kerana key mungkin tidak ada
use std::collections::HashMap;
let mut map = HashMap::new();
map.insert("kunci", 42);
let nilai: Option<&i32> = map.get("kunci");   // Some(&42)
let tiada: Option<&i32> = map.get("tiada");   // None

// Vec::first/last → Option kerana Vec mungkin kosong
let v = vec![1, 2, 3];
let pertama: Option<&i32> = v.first();  // Some(&1)
let kosong: Vec<i32> = vec![];
let tiada: Option<&i32> = kosong.first(); // None — Vec kosong!

// str::find → Option kerana mungkin tidak jumpa
let s = "hello world";
let kedudukan: Option<usize> = s.find('w'); // Some(6)
let tiada: Option<usize> = s.find('z');     // None

// Iterator::next → Option kerana iterator mungkin habis
let mut iter = vec![1, 2].into_iter();
let pertama: Option<i32> = iter.next(); // Some(1)
let kedua:   Option<i32> = iter.next(); // Some(2)
let habis:   Option<i32> = iter.next(); // None — habis!

// String::parse → Result (boleh fail dengan error)
// tapi ada methods yang return Option:
let nombor: Option<i32> = "42".parse().ok();  // Some(42)
let gagal:  Option<i32> = "abc".parse().ok(); // None
```

---

# BAB 3: Cara Handle Option 🎛️

## Semua Cara dari Asas ke Lanjutan

```rust
fn main() {
    let opt: Option<i32> = Some(42);
    let tiada: Option<i32> = None;

    // ══════════════════════════════════════════════════════════
    // CARA 1: match — kawalan penuh
    // ══════════════════════════════════════════════════════════
    match opt {
        Some(n) => println!("Ada: {}", n),
        None    => println!("Tiada"),
    }

    // ══════════════════════════════════════════════════════════
    // CARA 2: if let — bila hanya kes Some yang penting
    // ══════════════════════════════════════════════════════════
    if let Some(n) = opt {
        println!("Ada: {}", n);
        // tiada else — None diabaikan
    }

    if let Some(n) = opt {
        println!("Ada: {}", n);
    } else {
        println!("Tiada");
    }

    // ══════════════════════════════════════════════════════════
    // CARA 3: while let — loop sehingga None
    // ══════════════════════════════════════════════════════════
    let mut iter = vec![1, 2, 3].into_iter();
    while let Some(n) = iter.next() {
        println!("Item: {}", n);
    }

    // ══════════════════════════════════════════════════════════
    // CARA 4: ? operator — propagate None
    // ══════════════════════════════════════════════════════════
    fn proses() -> Option<i32> {
        let a = opt?;      // kalau None, return None
        let b = tiada?;    // ini akan return None dari proses()
        Some(a + b)
    }

    println!("{:?}", proses()); // None

    // ══════════════════════════════════════════════════════════
    // CARA 5: let else — early return kalau None (Rust 1.65+)
    // ══════════════════════════════════════════════════════════
    fn dengan_let_else(opt: Option<i32>) -> String {
        let Some(n) = opt else {
            return "Tiada nilai!".into();
        };
        // n tersedia di sini
        format!("Nilai: {}", n)
    }

    println!("{}", dengan_let_else(Some(42)));  // Nilai: 42
    println!("{}", dengan_let_else(None));       // Tiada nilai!
}
```

---

# BAB 4: Option Methods yang Penting 🔧

## Method Lengkap dengan Contoh

```rust
fn main() {
    let ada:   Option<i32> = Some(42);
    let tiada: Option<i32> = None;

    // ── Semak ─────────────────────────────────────────────────
    println!("{}", ada.is_some());       // true
    println!("{}", ada.is_none());       // false
    println!("{}", tiada.is_some());     // false
    println!("{}", tiada.is_none());     // true

    // ── Ambil nilai (HATI-HATI!) ──────────────────────────────
    // .unwrap() — PANIC kalau None!
    let n = ada.unwrap();                // 42 — OK
    // tiada.unwrap();                   // PANIC!

    // .expect() — PANIC dengan mesej
    let n = ada.expect("Mestilah ada nilai");  // OK
    // tiada.expect("Ini akan panic!");          // PANIC + mesej

    // .unwrap_or() — nilai default kalau None
    println!("{}", ada.unwrap_or(0));    // 42
    println!("{}", tiada.unwrap_or(0));  // 0

    // .unwrap_or_else() — compute default lazily
    println!("{}", tiada.unwrap_or_else(|| 99));  // 99 (dikira bila None)

    // .unwrap_or_default() — guna Default::default()
    let s: Option<String> = None;
    println!("{}", s.unwrap_or_default()); // "" (String::default())

    let n: Option<i32> = None;
    println!("{}", n.unwrap_or_default()); // 0

    // ── Transform ─────────────────────────────────────────────
    // .map() — ubah nilai kalau Some, kekalkan None
    let double = ada.map(|n| n * 2);         // Some(84)
    let double_tiada = tiada.map(|n| n * 2); // None

    // .map_or() — transform dengan default
    println!("{}", ada.map_or(0, |n| n * 2));    // 84
    println!("{}", tiada.map_or(0, |n| n * 2));  // 0

    // .map_or_else() — lazy version
    println!("{}", tiada.map_or_else(|| 99, |n| n * 2)); // 99

    // .and_then() — chain yang mungkin return None (flatmap)
    fn separuh(n: i32) -> Option<i32> {
        if n % 2 == 0 { Some(n / 2) } else { None }
    }

    println!("{:?}", ada.and_then(separuh));           // Some(21)
    println!("{:?}", Some(3).and_then(separuh));       // None (3 ganjil)
    println!("{:?}", tiada.and_then(separuh));         // None

    // .filter() — None jika tidak lulus predikat
    let besar = ada.filter(|&n| n > 100);  // None (42 tidak > 100)
    let kecil = ada.filter(|&n| n > 10);   // Some(42)

    // ── Combine ───────────────────────────────────────────────
    // .or() — gunakan fallback kalau None
    println!("{:?}", tiada.or(Some(99))); // Some(99)
    println!("{:?}", ada.or(Some(99)));   // Some(42) — ada sudah ada nilai

    // .or_else() — lazy fallback
    println!("{:?}", tiada.or_else(|| Some(kira_default()))); // Some(?)

    // .zip() — combine dua Option
    let a: Option<i32> = Some(1);
    let b: Option<&str> = Some("hello");
    println!("{:?}", a.zip(b));           // Some((1, "hello"))
    println!("{:?}", a.zip(tiada));       // None (salah satu None)

    // ── Convert ───────────────────────────────────────────────
    // Option → Result
    let r: Result<i32, &str> = ada.ok_or("tiada nilai"); // Ok(42)
    let r: Result<i32, &str> = tiada.ok_or("tiada nilai"); // Err("tiada nilai")

    // Result → Option
    let r: Result<i32, &str> = Ok(42);
    let o: Option<i32> = r.ok();   // Some(42) — buang error
    let e: Option<&str> = r.err(); // None — buang Ok

    // Option<Option<T>> → Option<T>
    let nested: Option<Option<i32>> = Some(Some(42));
    let flat: Option<i32> = nested.flatten(); // Some(42)
}

fn kira_default() -> i32 { 99 }
```

---

# BAB 5: Option dalam Struct & Fungsi 🏗️

## Design Pattern dengan Option

```rust
use std::collections::HashMap;

// ── Struct dengan optional fields ────────────────────────────
#[derive(Debug, Clone)]
struct ProfilPekerja {
    // Field WAJIB — bukan Option
    id:           u32,
    nama:         String,
    no_pekerja:   String,
    bahagian:     String,

    // Field PILIHAN — menggunakan Option
    emel:         Option<String>,     // belum dikemaskini
    no_telefon:   Option<String>,     // optional
    supervisor:   Option<u32>,        // ID supervisor (mungkin tiada)
    tarikh_keluar: Option<String>,    // None = masih bekerja
    nota:         Option<String>,     // nota pentadbiran
}

impl ProfilPekerja {
    fn baru(id: u32, nama: &str, no_pekerja: &str, bahagian: &str) -> Self {
        ProfilPekerja {
            id, nama: nama.into(), no_pekerja: no_pekerja.into(),
            bahagian: bahagian.into(),
            // Semua optional bermula sebagai None
            emel: None, no_telefon: None, supervisor: None,
            tarikh_keluar: None, nota: None,
        }
    }

    fn dengan_emel(mut self, emel: &str) -> Self {
        self.emel = Some(emel.into());
        self
    }

    fn dengan_supervisor(mut self, supervisor_id: u32) -> Self {
        self.supervisor = Some(supervisor_id);
        self
    }

    fn masih_bekerja(&self) -> bool {
        self.tarikh_keluar.is_none()
    }

    fn nama_supervisor<'a>(&self, db: &'a HashMap<u32, ProfilPekerja>)
        -> Option<&'a str>
    {
        // Option chaining — kalau tiada supervisor, return None
        self.supervisor
            .and_then(|id| db.get(&id))
            .map(|s| s.nama.as_str())
    }
}

fn main() {
    let mut db: HashMap<u32, ProfilPekerja> = HashMap::new();

    let ali = ProfilPekerja::baru(1, "Ali Ahmad", "KADA001", "ICT")
        .dengan_emel("ali@email.com")
        .dengan_supervisor(99);

    let ketua = ProfilPekerja::baru(99, "Dr. Hassan", "KADA099", "Pengurusan");

    db.insert(1, ali);
    db.insert(99, ketua);

    let ali = db.get(&1).unwrap();

    // Optional field — safe access
    match &ali.emel {
        Some(e) => println!("Emel: {}", e),
        None    => println!("Tiada emel"),
    }

    println!("Masih bekerja: {}", ali.masih_bekerja()); // true

    // Chaining untuk supervisor
    match ali.nama_supervisor(&db) {
        Some(nama) => println!("Supervisor: {}", nama),
        None       => println!("Tiada supervisor"),
    }
}
```

---

# BAB 6: Option Chaining 🔗

## Gantikan Null Check Berganda

```rust
// Masalah dalam bahasa lain — nested null checks:
//
// Java/C# (sebelum optional chaining):
// if (syarikat != null
//     && syarikat.getPejabat() != null
//     && syarikat.getPejabat().getAlamat() != null) {
//     return syarikat.getPejabat().getAlamat().getPoscode();
// }
// return null;

// Rust dengan Option chaining:

#[derive(Debug)]
struct Alamat { jalan: String, bandar: String, poscode: String }

#[derive(Debug)]
struct Pejabat { nama: String, alamat: Option<Alamat> }

#[derive(Debug)]
struct Syarikat { nama: String, pejabat_utama: Option<Pejabat> }

fn main() {
    let syarikat = Some(Syarikat {
        nama: "KADA".into(),
        pejabat_utama: Some(Pejabat {
            nama: "Ibu Pejabat".into(),
            alamat: Some(Alamat {
                jalan:   "Jalan Kemubu".into(),
                bandar:  "Kota Bharu".into(),
                poscode: "15200".into(),
            }),
        }),
    });

    // ── Cara 1: and_then chaining (idiomatik Rust) ────────────
    let poscode = syarikat
        .as_ref()
        .and_then(|s| s.pejabat_utama.as_ref())
        .and_then(|p| p.alamat.as_ref())
        .map(|a| a.poscode.as_str());

    println!("{:?}", poscode); // Some("15200")

    // ── Cara 2: ? dalam fungsi ────────────────────────────────
    fn dapatkan_poscode(syarikat: &Option<Syarikat>) -> Option<&str> {
        let s = syarikat.as_ref()?;
        let p = s.pejabat_utama.as_ref()?;
        let a = p.alamat.as_ref()?;
        Some(&a.poscode)
    }

    println!("{:?}", dapatkan_poscode(&syarikat)); // Some("15200")

    // ── Cara 3: Nested None ───────────────────────────────────
    let syarikat_tanpa_alamat = Some(Syarikat {
        nama: "Syarikat B".into(),
        pejabat_utama: Some(Pejabat {
            nama: "Pejabat".into(),
            alamat: None,  // ← tiada alamat
        }),
    });

    let p2 = dapatkan_poscode(&syarikat_tanpa_alamat);
    println!("{:?}", p2); // None — selamat! tiada crash!

    let tiada: Option<Syarikat> = None;
    let p3 = dapatkan_poscode(&tiada);
    println!("{:?}", p3); // None — selamat!
}
```

---

## 🧠 Brain Teaser #2

Apa output kod ini? Kenapa?

```rust
fn main() {
    let v: Vec<i32> = vec![10, 20, 30];

    let hasil = v.get(5)       // indeks 5 tidak ada
        .map(|&n| n * 2)
        .filter(|&n| n > 100)
        .unwrap_or(0);

    println!("{}", hasil);

    let hasil2 = v.get(1)      // indeks 1 = 20
        .map(|&n| n * 2)       // 40
        .filter(|&n| n > 100)  // 40 > 100? Tidak!
        .unwrap_or(0);

    println!("{}", hasil2);

    let hasil3 = v.get(2)      // indeks 2 = 30
        .map(|&n| n * 2)       // 60
        .filter(|&n| n > 50)   // 60 > 50? Ya!
        .unwrap_or(0);

    println!("{}", hasil3);
}
```

<details>
<summary>👀 Jawapan</summary>

```
Output:
  0    ← v.get(5) = None → map = None → filter = None → unwrap_or = 0
  0    ← v.get(1) = Some(&20) → map = Some(40) → filter(40>100=false) = None → 0
  60   ← v.get(2) = Some(&30) → map = Some(60) → filter(60>50=true) = Some(60) → 60

Kenapa ini hebat?
  Tiada crash! Tiada null check! Tiada IndexOutOfBounds!
  Chain method berfungsi dengan betul walaupun ada None.
  None "menular" — bila ada None dalam chain, semua lepas itu jadi None.
  Kita handle semuanya di HUJUNG dengan unwrap_or.

Berbanding Java:
  int[] v = {10, 20, 30};
  // v[5] → ArrayIndexOutOfBoundsException! CRASH!
  // Tiada cara yang elegan untuk chain seperti ini
```
</details>

---

# BAB 7: Option vs Result — Bila Guna Mana ⚖️

## Perbezaan Fundamental

```rust
// Option<T> — soalan: "Ada atau tidak ada?"
// Tiada maklumat KENAPA tiada
// Guna: cari dalam koleksi, nilai pilihan, perhitungan mungkin gagal secara logik

// Result<T, E> — soalan: "Berjaya atau gagal? Kalau gagal, kenapa?"
// Ada maklumat ralat dalam Err(E)
// Guna: operasi I/O, parsing, network, sebarang operasi yang boleh fail

fn demo_bila_guna() {
    use std::collections::HashMap;
    use std::num::ParseIntError;

    // ── Option: cari dalam HashMap ─────────────────────────────
    let mut map = HashMap::new();
    map.insert("kunci", 42);

    let nilai: Option<&i32> = map.get("kunci");
    // None bermaksud: "tiada dalam map" — bukan ralat!

    // ── Result: parse string ───────────────────────────────────
    let r: Result<i32, ParseIntError> = "42".parse();
    // Err bermaksud: "gagal, dan ini sebabnya"

    // ── Option: first element ─────────────────────────────────
    let v = vec![1, 2, 3];
    let pertama: Option<&i32> = v.first();
    // None bermaksud: "Vec kosong" — bukan ralat!

    // ── Result: baca fail ─────────────────────────────────────
    let r: Result<String, std::io::Error> = std::fs::read_to_string("fail.txt");
    // Err bermaksud: fail tidak ada, permission denied, dll

    // ── Tukar antara Option dan Result ───────────────────────
    let opt: Option<i32> = Some(42);

    // Option → Result (bila perlu tahu kenapa None)
    let r: Result<i32, &str> = opt.ok_or("nilai tiada");
    let r: Result<i32, String> = opt.ok_or_else(|| "dikira: tiada".into());

    // Result → Option (buang maklumat error)
    let r: Result<i32, &str> = Ok(42);
    let o: Option<i32> = r.ok(); // Some(42), atau None kalau Err

    // ── Panduan ringkas ───────────────────────────────────────
    // "Adakah ini ralat atau hanya 'tiada'?"
    //
    // HashMap::get → Option (tiada dalam map = normal, bukan ralat)
    // File::open   → Result (gagal buka = ralat dengan sebab)
    // Vec::first   → Option (kosong = normal, bukan ralat)
    // str::parse   → Result (format salah = ralat dengan sebab)
}
```

---

# BAB 8: Pattern Dunia Sebenar 🌍

## Pattern yang Selalu Muncul

```rust
use std::collections::HashMap;

// ── Pattern 1: Default value ──────────────────────────────────
struct Tetapan {
    warna_tema:  Option<String>,
    bahasa:      Option<String>,
    ukuran_fon:  Option<u32>,
}

impl Tetapan {
    fn warna_tema(&self) -> &str {
        self.warna_tema.as_deref().unwrap_or("biru")
    }

    fn bahasa(&self) -> &str {
        self.bahasa.as_deref().unwrap_or("ms")
    }

    fn ukuran_fon(&self) -> u32 {
        self.ukuran_fon.unwrap_or(14)
    }
}

// ── Pattern 2: Early return dengan ? ─────────────────────────
fn proses(
    db: &HashMap<u32, String>,
    id: u32,
) -> Option<String> {
    let nama = db.get(&id)?;       // return None kalau tiada
    let trimmed = nama.trim();
    if trimmed.is_empty() { return None; }
    Some(trimmed.to_uppercase())
}

// ── Pattern 3: Collect Option ─────────────────────────────────
fn parse_semua(input: &[&str]) -> Option<Vec<i32>> {
    // Kalau ADA SATU gagal → semua None
    input.iter()
        .map(|s| s.parse::<i32>().ok())
        .collect()
}

fn main() {
    // Collect — semua mesti Some
    println!("{:?}", parse_semua(&["1", "2", "3"]));    // Some([1,2,3])
    println!("{:?}", parse_semua(&["1", "abc", "3"]));  // None (ada yang gagal)
}

// ── Pattern 4: Lazy initialization ───────────────────────────
struct LazyData {
    cache: Option<Vec<u8>>,
}

impl LazyData {
    fn baru() -> Self { LazyData { cache: None } }

    fn data(&mut self) -> &[u8] {
        // Kira hanya sekali, cache untuk seterusnya
        self.cache.get_or_insert_with(|| {
            println!("(mengira data...)");
            vec![1, 2, 3, 4, 5]
        })
    }
}

fn demo_lazy() {
    let mut ld = LazyData::baru();
    println!("{:?}", ld.data()); // (mengira data...) [1,2,3,4,5]
    println!("{:?}", ld.data()); // [1,2,3,4,5] ← tidak kira semula!
}

// ── Pattern 5: Option dalam HashMap ──────────────────────────
fn kemas_atau_tetap(
    map: &mut HashMap<String, i32>,
    kunci: &str,
    nilai: Option<i32>,
) {
    match nilai {
        Some(v) => { map.insert(kunci.into(), v); }
        None    => { /* tidak ubah apa-apa */ }
    }
}

// ── Pattern 6: Flatten nested Option ─────────────────────────
fn cari_berganda(
    map1: &HashMap<u32, u32>,
    map2: &HashMap<u32, String>,
    kunci1: u32,
) -> Option<&String> {
    let kunci2 = map1.get(&kunci1)?;  // Option<&u32>
    map2.get(kunci2)                   // Option<&String>
}
```

---

# BAB 9: Anti-patterns — Jangan Buat Ini ⛔

## Cara Salah Guna Option

```rust
fn main() {
    let opt: Option<i32> = Some(42);

    // ══════════════════════════════════════════════════════════
    // ❌ ANTI-PATTERN 1: unwrap() tanpa fikir
    // ══════════════════════════════════════════════════════════
    let n = opt.unwrap(); // PANIC kalau None!
    // Bila guna? Hanya bila PASTI ada nilai atau dalam test

    // ══════════════════════════════════════════════════════════
    // ❌ ANTI-PATTERN 2: is_some() + unwrap()
    // ══════════════════════════════════════════════════════════
    if opt.is_some() {
        let n = opt.unwrap(); // TIDAK PERLU! Guna if let!
        println!("{}", n);
    }

    // ✔ BETUL:
    if let Some(n) = opt {
        println!("{}", n);
    }

    // ══════════════════════════════════════════════════════════
    // ❌ ANTI-PATTERN 3: Simulate null dengan Option:None
    // ══════════════════════════════════════════════════════════
    fn cari(id: u32) -> Option<i32> {
        // JANGAN guna None sebagai "error" indicator
        // kalau ada sebab kenapa gagal → guna Result!
        if id == 0 { return None; } // kenapa gagal? "id = 0"?
        Some(id as i32)
    }

    // ✔ BETUL kalau ada maklumat error:
    fn cari_betul(id: u32) -> Result<i32, String> {
        if id == 0 { return Err("ID tidak boleh sifar".into()); }
        Ok(id as i32)
    }

    // ══════════════════════════════════════════════════════════
    // ❌ ANTI-PATTERN 4: Nested Option
    // ══════════════════════════════════════════════════════════
    let nested: Option<Option<i32>> = Some(Some(42));
    // Pelik — kenapa ada dua lapisan?

    // ✔ BETUL: flatten atau reka semula
    let flat = nested.flatten(); // Some(42)

    // ══════════════════════════════════════════════════════════
    // ❌ ANTI-PATTERN 5: match untuk perkara yang ada method
    // ══════════════════════════════════════════════════════════
    let hasil = match opt {
        Some(n) => n * 2,
        None    => 0,
    };

    // ✔ BETUL: guna map_or
    let hasil = opt.map_or(0, |n| n * 2);

    // ══════════════════════════════════════════════════════════
    // ❌ ANTI-PATTERN 6: Option<bool> (gunakan bool sahaja!)
    // ══════════════════════════════════════════════════════════
    let aktif: Option<bool> = Some(true); // Kenapa bukan bool?!
    // Option<bool> = tiga keadaan: Some(true), Some(false), None
    // Kalau ini yang dikehendaki, perlu reka enum!

    enum StatusAktif { Aktif, TidakAktif, BelumDiketahui }
    // Ini lebih jelas!

    // ══════════════════════════════════════════════════════════
    // ❌ ANTI-PATTERN 7: Clone bila boleh borrow
    // ══════════════════════════════════════════════════════════
    let s: Option<String> = Some("hello".into());

    // Buruk:
    let panjang = s.clone().map(|x| x.len()); // clone tidak perlu!

    // ✔ Betul: as_deref atau as_ref
    let panjang = s.as_deref().map(|x| x.len()); // &str, tiada clone
    let panjang = s.as_ref().map(|x| x.len());   // &String, tiada clone
}
```

---

# BAB 10: Mini Project — PHP vs Rust 🆚

```rust
// ══════════════════════════════════════════════════════════════
// Simulasi: Sistem cari pekerja
// PHP vs Rust — lihat perbezaan keselamatan
// ══════════════════════════════════════════════════════════════

// ─── SIMULASI PHP (dalam komen) ───────────────────────────────
//
// function cariPekerja($id) {
//     // Mungkin return null atau Pekerja!
//     if ($id == 1) return new Pekerja("Ali", "ICT", 4500);
//     return null; // ← bahaya!
// }
//
// function prosesGaji($id) {
//     $p = cariPekerja($id);
//     // Programmer MESTI ingat $p boleh null!
//     // Kalau lupa: Fatal error: Call to member function
//     return $p->getGaji() * 1.1; // CRASH kalau $p null!
//
//     // Defensive (penat dan mudah lupa):
//     if ($p !== null) {
//         return $p->getGaji() * 1.1;
//     }
//     return 0;
// }
//
// ─── RUST ──────────────────────────────────────────────────────

use std::collections::HashMap;

#[derive(Debug, Clone)]
struct Pekerja {
    nama:     String,
    bahagian: String,
    gaji:     f64,
    emel:     Option<String>,      // OPTIONAL
    supervisor_id: Option<u32>,    // OPTIONAL
}

impl Pekerja {
    fn baru(nama: &str, bahagian: &str, gaji: f64) -> Self {
        Pekerja {
            nama:          nama.into(),
            bahagian:      bahagian.into(),
            gaji,
            emel:          None,
            supervisor_id: None,
        }
    }

    fn dengan_emel(mut self, emel: &str) -> Self {
        self.emel = Some(emel.into());
        self
    }

    fn dengan_supervisor(mut self, id: u32) -> Self {
        self.supervisor_id = Some(id);
        self
    }

    fn emel_domain(&self) -> Option<&str> {
        // Optional chaining yang selamat!
        self.emel
            .as_deref()
            .and_then(|e| e.split('@').nth(1))
    }
}

struct Sistem {
    pekerja: HashMap<u32, Pekerja>,
}

impl Sistem {
    fn baru() -> Self {
        Sistem { pekerja: HashMap::new() }
    }

    fn tambah(&mut self, id: u32, pekerja: Pekerja) {
        self.pekerja.insert(id, pekerja);
    }

    // Return type JELAS: Option<&Pekerja>
    // Caller MESTI handle None — compiler paksa!
    fn cari(&self, id: u32) -> Option<&Pekerja> {
        self.pekerja.get(&id)
    }

    fn kira_gaji_naik(&self, id: u32, kadar: f64) -> Option<f64> {
        // ? propagate None dengan elegan
        let pekerja = self.cari(id)?;
        Some(pekerja.gaji * (1.0 + kadar))
    }

    fn nama_supervisor(&self, id: u32) -> Option<&str> {
        let pekerja = self.cari(id)?;
        let sup_id  = pekerja.supervisor_id?;
        let sup     = self.cari(sup_id)?;
        Some(&sup.nama)
    }

    fn ringkasan_gaji(&self, id: u32) -> String {
        match self.cari(id) {
            Some(p) => format!(
                "{} ({}): RM{:.2}{}",
                p.nama,
                p.bahagian,
                p.gaji,
                p.emel.as_deref()
                    .map(|e| format!(" <{}>", e))
                    .unwrap_or_default()
            ),
            None => format!("Pekerja #{} tidak dijumpai", id),
        }
    }
}

fn main() {
    println!("{'═'*60}");
    println!("{:^60}", "Sistem Pekerja — Rust vs Null");
    println!("{'═'*60}\n");

    let mut sistem = Sistem::baru();

    // Tambah pekerja
    sistem.tambah(1, Pekerja::baru("Ali Ahmad", "ICT", 4500.0)
        .dengan_emel("ali@kada.gov.my")
        .dengan_supervisor(99)
    );
    sistem.tambah(2, Pekerja::baru("Siti Hawa", "HR", 3800.0)
        // Sengaja tiada emel — ini OK!
    );
    sistem.tambah(99, Pekerja::baru("Dr. Hassan", "Pengurusan", 8000.0)
        .dengan_emel("hassan@kada.gov.my")
    );

    println!("=== Cari Pekerja ===\n");

    // ── Cari yang ada ────────────────────────────────────────
    match sistem.cari(1) {
        Some(p) => println!("✔ Jumpa: {} ({})", p.nama, p.bahagian),
        None    => println!("✗ Tidak dijumpai"),
    }

    // ── Cari yang tidak ada ───────────────────────────────────
    match sistem.cari(999) {
        Some(p) => println!("✔ Jumpa: {}", p.nama),
        None    => println!("✗ ID 999 tidak dijumpai — SELAMAT, tiada crash!"),
    }

    println!("\n=== Ringkasan Gaji ===\n");
    for id in [1, 2, 99, 999] {
        println!("{}", sistem.ringkasan_gaji(id));
    }

    println!("\n=== Kira Kenaikan Gaji ===\n");
    for id in [1u32, 2, 999] {
        match sistem.kira_gaji_naik(id, 0.10) {
            Some(gaji_baru) => println!("Pekerja #{}: RM{:.2} → RM{:.2}",
                id,
                sistem.cari(id).unwrap().gaji,
                gaji_baru),
            None => println!("Pekerja #{} tidak dijumpai — tiada crash!", id),
        }
    }

    println!("\n=== Cari Supervisor ===\n");
    for id in [1u32, 2, 99, 999] {
        match sistem.nama_supervisor(id) {
            Some(nama) => println!("Supervisor pekerja #{}: {}", id, nama),
            None       => println!("Pekerja #{}: tiada supervisor atau tidak dijumpai", id),
        }
    }

    println!("\n=== Optional Fields ===\n");
    for id in [1u32, 2, 99] {
        if let Some(p) = sistem.cari(id) {
            // Akses emel — selamat!
            match p.emel_domain() {
                Some(domain) => println!("{}: domain emel = {}", p.nama, domain),
                None         => println!("{}: tiada emel", p.nama),
            }
        }
    }

    println!("\n{'═'*60}");
    println!("  Tiada satu pun NullPointerException!");
    println!("  Semua kes None dihandle — dipaksa oleh compiler.");
    println!("{'═'*60}");
}
```

---

# 📋 Rujukan Pantas — Option Cheat Sheet

## Buat Option

```rust
let ada:   Option<i32>    = Some(42);
let tiada: Option<i32>    = None;
let dari_kondisi          = if syarat { Some(nilai) } else { None };
```

## Handle Option

```rust
match opt { Some(v) => ..., None => ... }   // kawalan penuh
if let Some(v) = opt { ... }                 // bila hanya Some penting
if let Some(v) = opt { ... } else { ... }   // dengan else
while let Some(v) = iter.next() { ... }     // dalam loop
let Some(v) = opt else { return; };         // let else (Rust 1.65+)
opt?                                         // propagate None (dalam fungsi return Option)
```

## Methods Penting

```rust
// Semak
opt.is_some()               // → bool
opt.is_none()               // → bool

// Ambil (HATI-HATI)
opt.unwrap()                // PANIC kalau None
opt.expect("mesej")         // PANIC + mesej
opt.unwrap_or(default)      // nilai default
opt.unwrap_or_else(|| ...)  // lazy default
opt.unwrap_or_default()     // Default::default()

// Transform (selamat)
opt.map(|v| ...)            // ubah Some, kekalkan None
opt.map_or(d, |v| ...)      // transform dengan default
opt.and_then(|v| ...)       // chain yang mungkin None (flatmap)
opt.filter(|v| ...)         // None jika gagal predikat
opt.or(other)               // fallback kalau None
opt.or_else(|| ...)         // lazy fallback

// Borrow
opt.as_ref()                // Option<T> → Option<&T>
opt.as_deref()              // Option<String> → Option<&str>
opt.as_mut()                // Option<T> → Option<&mut T>

// Convert
opt.ok_or(err)              // Option → Result
opt.flatten()               // Option<Option<T>> → Option<T>
result.ok()                 // Result → Option (buang error)
```

## Peraturan Emas

```
1. Option BUKAN null — ia adalah null yang selamat dengan API
2. Compiler PAKSA handle None — tiada surprise
3. ? lebih baik dari unwrap() untuk propagate
4. unwrap() hanya dalam test atau bila pasti ada
5. Guna expect() dengan mesej berguna kalau terpaksa unwrap
6. map/and_then/filter lebih elegan dari match untuk transform
7. Option → tiada maklumat kenapa None
   Result → ada maklumat kenapa Err
8. as_deref() untuk Option<String> → Option<&str> tanpa clone
```

---

*Tony Hoare minta maaf atas null.*
*Rust tidak ada null — ada Option yang compiler paksa anda handle.*
*Billion dollar mistake bertukar jadi zero dollar problem.* 🦀
