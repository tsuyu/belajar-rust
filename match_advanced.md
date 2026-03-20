# 🎯 Advanced Match & Pattern Matching dalam Rust

> Panduan lengkap pattern matching: dari asas hingga nested patterns, guards, bindings dan lebih.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu Pattern Matching?

```
Pattern matching = cara Rust "check bentuk" sesuatu nilai.

Macam acuan kek:
  ┌────────────────┐
  │  Nilai: Some(42)│
  └────────────────┘
       ↓ match
  Some(x) → ✔ sesuai! x = 42
  None    → ✘ tidak sesuai

Berbeza dari if/else biasa:
  if/else → check SYARAT (true/false)
  match   → check BENTUK + ekstrak DATA serentak
```

---

## Kenapa Pattern Matching Rust Berkuasa?

```
Bahasa lain:           Rust:
  if val == Some(x) {    match val {
    use(x);                Some(x) => use(x),
  }                        None    => ...,
                         }

Rust match:
  ✔ Exhaustive — compiler paksa semua kes ditangani
  ✔ Destructure — ekstrak nilai terus dalam pattern
  ✔ Guards — tambah syarat extra
  ✔ Bindings — nama bahagian yang matched
  ✔ Nested — pattern dalam pattern
  ✔ Multiple patterns — | untuk OR
  ✔ Range — match julat nilai
```

---

## Peta Pembelajaran

```
Bab 1  → match Asas & Exhaustive
Bab 2  → Destructuring Structs & Enums
Bab 3  → Tuple & Slice Patterns
Bab 4  → Guards — if dalam match
Bab 5  → Binding — @ operator
Bab 6  → Nested Patterns
Bab 7  → if let, while let, let else
Bab 8  → Pattern dalam Function Parameters
Bab 9  → Pattern Lanjutan
Bab 10 → Mini Project: Interpreter Ekspresi
```

---

# BAB 1: match Asas & Exhaustive 🎪

## Sintaks match

```rust
fn main() {
    let nombor = 7;

    let label = match nombor {
        1        => "satu",
        2        => "dua",
        3 | 4    => "tiga atau empat",   // multiple values dengan |
        5..=9    => "lima hingga sembilan", // range pattern
        10       => "sepuluh",
        _        => "lain-lain",          // wildcard — tangkap semua
    };
    println!("{} → {}", nombor, label);

    // match adalah expression — ada nilai!
    let mutlak = match nombor < 0 {
        true  => -nombor,
        false => nombor,
    };
    println!("|{}| = {}", nombor, mutlak);
}
```

---

## Exhaustive — Semua Kes Mesti Ditangani

```rust
enum Warna { Merah, Hijau, Biru, Kuning }

fn nama_warna(w: Warna) -> &'static str {
    match w {
        Warna::Merah  => "merah",
        Warna::Hijau  => "hijau",
        Warna::Biru   => "biru",
        // Warna::Kuning  ← hilang!
        // ERROR: non-exhaustive patterns — `Kuning` not covered
    }
}

// ✔ Betul — semua kes ditangani
fn nama_warna_betul(w: Warna) -> &'static str {
    match w {
        Warna::Merah  => "merah",
        Warna::Hijau  => "hijau",
        Warna::Biru   => "biru",
        Warna::Kuning => "kuning",
    }
}

// Atau guna wildcard _ untuk "lain-lain"
fn ada_merah(w: Warna) -> bool {
    match w {
        Warna::Merah => true,
        _            => false,  // semua warna lain
    }
}
```

> 💡 **Kelebihan Exhaustive:** Kalau tambah variant baru ke enum, **semua match akan gagal compile** sehingga anda tambah kes untuk variant baru. Tiada bug tersembunyi!

---

## Match dengan Return Value & Block

```rust
fn main() {
    let gred = 78u32;

    // match dengan block — boleh ada banyak baris
    let komen = match gred {
        90..=100 => {
            println!("Tahniah! Cemerlang!");
            "Lulus dengan cemerlang"
        }
        75..=89 => {
            println!("Bagus! Teruskan usaha.");
            "Lulus dengan baik"
        }
        50..=74 => "Lulus, boleh tingkatkan lagi",
        _       => {
            println!("Perlu belajar lebih keras.");
            "Gagal"
        }
    };

    println!("Gred {}: {}", gred, komen);

    // match dengan unit return (tiada nilai)
    let hari = "Sabtu";
    match hari {
        "Sabtu" | "Ahad" => println!("Hujung minggu!"),
        _                => println!("Hari bekerja"),
    }
}
```

---

## 🧠 Brain Teaser #1

Apa yang compiler complain tentang kod ini?

```rust
enum Status { Aktif, Tidak, Pending, Ditangguh }

fn cerita(s: Status) -> &'static str {
    match s {
        Status::Aktif   => "sedang berjalan",
        Status::Tidak   => "tidak aktif",
        _               => "dalam proses",
    }
}

// Kemudian kita tambah variant baru:
// enum Status { Aktif, Tidak, Pending, Ditangguh, DiSenaraiBantah }
```

<details>
<summary>👀 Jawapan</summary>

**Tiada error** — wildcard `_` menangkap semua variant termasuk `DiSenaraiBantah`.

Ini sebenarnya **bahaya** — variant baru `DiSenaraiBantah` secara senyap jatuh ke kes `"dalam proses"` tanpa kita sedar.

**Cara lebih selamat:** Explicit semua variant, tiada wildcard:
```rust
fn cerita(s: Status) -> &'static str {
    match s {
        Status::Aktif          => "sedang berjalan",
        Status::Tidak          => "tidak aktif",
        Status::Pending        => "dalam proses",
        Status::Ditangguh      => "ditangguhkan",
        // Status::DiSenaraiBantah  ← compiler akan ERROR di sini!
    }
}
```
Dengan cara ini, tambah variant baru = compiler alert anda untuk tambah kes.
</details>

---

# BAB 2: Destructuring Structs & Enums 🧩

## Destructure Struct

```rust
struct Titik { x: f64, y: f64 }

struct Pekerja {
    nama:     String,
    gaji:     f64,
    bahagian: String,
}

fn main() {
    let t = Titik { x: 3.0, y: 7.0 };

    // Destructure semua field
    let Titik { x, y } = t;
    println!("x={}, y={}", x, y); // 3.0, 7.0

    // Destructure dengan nama berbeza
    let t2 = Titik { x: 1.0, y: 2.0 };
    let Titik { x: a, y: b } = t2;
    println!("a={}, b={}", a, b); // 1.0, 2.0

    // Dalam match
    let p = Titik { x: 0.0, y: 5.0 };
    match p {
        Titik { x: 0.0, y }     => println!("Pada paksi Y, y={}", y),
        Titik { x, y: 0.0 }     => println!("Pada paksi X, x={}", x),
        Titik { x, y }          => println!("Titik ({}, {})", x, y),
    }

    // Ignorekan sebahagian field dengan ..
    let pekerja = Pekerja {
        nama:     "Ali".into(),
        gaji:     5000.0,
        bahagian: "ICT".into(),
    };

    let Pekerja { nama, bahagian, .. } = pekerja; // abaikan `gaji`
    println!("{} dari {}", nama, bahagian);
}
```

---

## Destructure Enum

```rust
#[derive(Debug)]
enum Mesej {
    Berhenti,
    Gerak { x: i32, y: i32 },
    Tulis(String),
    TukarWarna(u8, u8, u8),
}

fn proses(mesej: Mesej) {
    match mesej {
        // Unit variant
        Mesej::Berhenti => {
            println!("Berhenti!");
        }

        // Struct variant — destructure dengan nama field
        Mesej::Gerak { x, y } => {
            println!("Gerak ke ({}, {})", x, y);
        }

        // Tuple variant — destructure dengan posisi
        Mesej::Tulis(teks) => {
            println!("Tulis: {}", teks);
        }

        // Tuple variant berbilang — nama pilihan sendiri
        Mesej::TukarWarna(r, g, b) => {
            println!("Warna RGB: ({}, {}, {})", r, g, b);
        }
    }
}

fn main() {
    proses(Mesej::Berhenti);
    proses(Mesej::Gerak { x: 10, y: 20 });
    proses(Mesej::Tulis("Hello".into()));
    proses(Mesej::TukarWarna(255, 128, 0));
}
```

---

## Destructure Option & Result

```rust
fn main() {
    // Option<T>
    let nilai: Option<i32> = Some(42);

    match nilai {
        Some(n) if n > 0 => println!("Positif: {}", n),
        Some(n)          => println!("Tidak positif: {}", n),
        None             => println!("Tiada nilai"),
    }

    // Result<T, E>
    let hasil: Result<i32, &str> = Ok(100);

    match hasil {
        Ok(n)  => println!("Berjaya: {}", n),
        Err(e) => println!("Gagal: {}", e),
    }

    // Nested Option — Option<Option<T>>
    let nested: Option<Option<i32>> = Some(Some(7));
    match nested {
        Some(Some(n)) => println!("Dalam: {}", n), // 7
        Some(None)    => println!("Luar ada, dalam tiada"),
        None          => println!("Tiada langsung"),
    }

    // Result dalam Option
    let r: Option<Result<i32, &str>> = Some(Ok(5));
    match r {
        Some(Ok(n))  => println!("Ada dan berjaya: {}", n),
        Some(Err(e)) => println!("Ada tapi gagal: {}", e),
        None         => println!("Tiada"),
    }
}
```

---

# BAB 3: Tuple & Slice Patterns 📦

## Tuple Patterns

```rust
fn main() {
    // Match tuple
    let koordinat = (3, -5);
    match koordinat {
        (0, 0) => println!("Asal"),
        (x, 0) => println!("Pada paksi X: {}", x),
        (0, y) => println!("Pada paksi Y: {}", y),
        (x, y) => println!("Titik ({}, {})", x, y),
    }

    // Tuple dengan pelbagai kombinasi
    let (a, b, c) = (true, 42, "hello"); // destructure terus!
    println!("{} {} {}", a, b, c);

    // Match tuple besar — ignore dengan _
    let data = (1, 2, 3, 4, 5);
    match data {
        (first, .., last) => println!("Mula={}, Akhir={}", first, last),
    }

    // Match dua nilai serentak
    let x = 5i32;
    let y = -3i32;
    match (x, y) {
        (1..=5, 1..=5) => println!("Kedua-dua antara 1 dan 5"),
        (n, m) if n > 0 && m > 0 => println!("Kedua-dua positif"),
        (n, m) if n < 0 && m < 0 => println!("Kedua-dua negatif"),
        (n, _) if n > 0 => println!("Hanya x positif: {}", n),
        (_, m) if m > 0 => println!("Hanya y positif: {}", m),
        _               => println!("Campuran lain"),
    }
}
```

---

## Slice Patterns

```rust
fn main() {
    // Match slice berdasarkan panjang dan kandungan
    let v = vec![1, 2, 3];

    match v.as_slice() {
        []           => println!("Kosong"),
        [x]          => println!("Satu elemen: {}", x),
        [x, y]       => println!("Dua elemen: {}, {}", x, y),
        [x, y, z]    => println!("Tiga elemen: {}, {}, {}", x, y, z),
        [first, ..]  => println!("Mula dengan {}, ada lebih", first),
    }

    // Pattern dengan ..rest
    let angka = [10, 20, 30, 40, 50];
    match angka {
        [pertama, tengah @ .., terakhir] => {
            println!("Pertama: {}", pertama);
            println!("Tengah:  {:?}", tengah); // [20, 30, 40]
            println!("Terakhir: {}", terakhir);
        }
    }

    // Guna dalam fungsi
    fn proses_senarai(data: &[i32]) {
        match data {
            [] => println!("Kosong!"),
            [satu] => println!("Satu item: {}", satu),
            [a, b] => println!("Dua item: {} + {} = {}", a, b, a+b),
            [a, b, rest @ ..] => {
                println!("Mula: {}, {}", a, b);
                println!("Baki {} item: {:?}", rest.len(), rest);
            }
        }
    }

    proses_senarai(&[]);
    proses_senarai(&[1]);
    proses_senarai(&[1, 2]);
    proses_senarai(&[1, 2, 3, 4, 5]);
}
```

---

## 🧠 Brain Teaser #2

Apa output kod ini?

```rust
fn analisis(data: &[i32]) -> &str {
    match data {
        [.., last] if *last > 100 => "besar di hujung",
        [first, ..] if *first < 0 => "negatif di depan",
        [x] if *x == 0            => "sifar tunggal",
        []                        => "kosong",
        _                         => "biasa",
    }
}

fn main() {
    println!("{}", analisis(&[]));
    println!("{}", analisis(&[0]));
    println!("{}", analisis(&[-5, 10, 20]));
    println!("{}", analisis(&[10, 20, 200]));
    println!("{}", analisis(&[1, 2, 3]));
}
```

<details>
<summary>👀 Jawapan</summary>

```
kosong
sifar tunggal
negatif di depan
besar di hujung
biasa
```

Perhatikan urutan matching — Rust cuba dari atas ke bawah. `[-5, 10, 20]` memenuhi `[first, ..]` dengan guard `*first < 0`, bukan `_`. `[10, 20, 200]` memenuhi `[.., last]` dengan guard `*last > 100`.
</details>

---

# BAB 4: Guards — if dalam match 🛡️

## Match Guards

```rust
fn main() {
    let num = Some(7);

    // Guard — tambah syarat extra selepas pattern
    match num {
        Some(x) if x < 0  => println!("{} adalah negatif", x),
        Some(x) if x == 0 => println!("Sifar"),
        Some(x) if x > 10 => println!("{} lebih dari 10", x),
        Some(x)            => println!("{} antara 1 dan 10", x),
        None               => println!("Tiada nilai"),
    }

    // Guard dengan binding
    let pasangan = (3, -2);
    match pasangan {
        (x, y) if x == y       => println!("Sama!"),
        (x, y) if x + y == 0   => println!("Jumlah sifar: {} + {}", x, y),
        (x, _) if x % 2 == 0   => println!("x genap: {}", x),
        (x, y)                  => println!("Lain: ({}, {})", x, y),
    }

    // Guard dengan | (multiple patterns + guard)
    let c = 'a';
    match c {
        'a' | 'e' | 'i' | 'o' | 'u' => println!("Vokal kecil"),
        'A' | 'E' | 'I' | 'O' | 'U' => println!("Vokal besar"),
        c if c.is_alphabetic()        => println!("Huruf: {}", c),
        c if c.is_numeric()           => println!("Nombor: {}", c),
        _                             => println!("Lain: {}", c),
    }
}
```

---

## Guards dengan Struct & Complex Conditions

```rust
#[derive(Debug)]
struct Pelajar {
    nama:   String,
    markah: f64,
    hadir:  u32, // hari hadir
}

fn tentukan_keputusan(p: &Pelajar) -> &str {
    match p {
        // Guard periksa berbilang field
        Pelajar { markah, hadir, .. }
            if *markah >= 85.0 && *hadir >= 80  => "Lulus Cemerlang",

        Pelajar { markah, hadir, .. }
            if *markah >= 60.0 && *hadir >= 60  => "Lulus",

        Pelajar { hadir, .. }
            if *hadir < 40                       => "Gagal (hadir rendah)",

        Pelajar { markah, .. }
            if *markah < 40.0                    => "Gagal (markah rendah)",

        _                                         => "Perlu semak semula",
    }
}

fn main() {
    let pelajar = vec![
        Pelajar { nama: "Ali".into(),  markah: 90.0, hadir: 85 },
        Pelajar { nama: "Siti".into(), markah: 70.0, hadir: 65 },
        Pelajar { nama: "Amin".into(), markah: 55.0, hadir: 30 },
        Pelajar { nama: "Zara".into(), markah: 35.0, hadir: 70 },
    ];

    for p in &pelajar {
        println!("{}: {}", p.nama, tentukan_keputusan(p));
    }
}
```

---

## ⚠ Guard Tidak Affect Exhaustiveness

```rust
fn main() {
    let x: i32 = 5;

    // Guards TIDAK dikira untuk exhaustiveness!
    // Kod ini akan ERROR — _ diperlukan walaupun guard cover semua kes
    // match x {
    //     n if n < 0  => println!("negatif"),
    //     n if n >= 0 => println!("tidak negatif"),
    //     // Compiler tidak tahu guard cover semua — perlukan _
    // }

    // ✔ Betul — tambah _ atau kes tanpa guard
    match x {
        n if n < 0  => println!("negatif: {}", n),
        n if n > 0  => println!("positif: {}", n),
        _           => println!("sifar"),
    }
}
```

---

# BAB 5: Binding — @ Operator 🏷️

## @ untuk Bind dan Test Serentak

```rust
fn main() {
    let num = 7;

    // @ — bind nilai ke nama DAN test pattern serentak
    match num {
        n @ 1..=5  => println!("n={} antara 1 dan 5", n),
        n @ 6..=10 => println!("n={} antara 6 dan 10", n),
        n          => println!("n={} di luar julat", n),
    }

    // Tanpa @ — tidak dapat nilai dalam range pattern
    // match num {
    //     1..=5  => println!("antara 1 dan 5 tapi tak tahu nilai!"),
    // }

    // @ dalam enum destructuring
    #[derive(Debug)]
    enum Mesej {
        Nilai(i32),
        Nama(String),
    }

    let m = Mesej::Nilai(42);
    match m {
        Mesej::Nilai(n @ 1..=50)   => println!("Nilai kecil: {}", n),
        Mesej::Nilai(n @ 51..=100) => println!("Nilai besar: {}", n),
        Mesej::Nilai(n)            => println!("Nilai luar: {}", n),
        Mesej::Nama(s)             => println!("Nama: {}", s),
    }
}
```

---

## @ dalam Struct Patterns

```rust
#[derive(Debug)]
struct Titik { x: i32, y: i32 }

fn main() {
    let t = Titik { x: 7, y: 3 };

    match t {
        // Bind seluruh struct DAN destructure
        ref p @ Titik { x, y } if x > 0 && y > 0 => {
            println!("Kuadran pertama: {:?}", p);
            println!("x={}, y={}", x, y);
        }
        Titik { x: x @ 0, y } => {
            println!("Pada paksi Y dengan nilai y={}", y);
        }
        Titik { x, y } => {
            println!("Titik lain: ({}, {})", x, y);
        }
    }

    // @ dalam match nested — sangat berguna!
    let nilai: Option<i32> = Some(15);
    match nilai {
        Some(n @ 10..=20) => println!("Dalam julat 10-20: {}", n),
        Some(n @ 1..=9)   => println!("Kecil: {}", n),
        Some(n)           => println!("Besar: {}", n),
        None              => println!("Tiada"),
    }
}
```

---

## 🧠 Brain Teaser #3

Apakah output dan apakah fungsi `@` dalam pattern ini?

```rust
fn main() {
    let code = Some(404u32);

    let mesej = match code {
        Some(c @ 200..=299) => format!("Berjaya ({})", c),
        Some(c @ 300..=399) => format!("Redirect ({})", c),
        Some(c @ 400..=499) => format!("Client Error ({})", c),
        Some(c @ 500..=599) => format!("Server Error ({})", c),
        Some(c)             => format!("Kod lain: {}", c),
        None                => "Tiada kod".to_string(),
    };

    println!("{}", mesej);
}
```

<details>
<summary>👀 Jawapan</summary>

Output: `Client Error (404)`

`@` membolehkan:
1. **Test** — adakah nilai dalam julat `400..=499`?
2. **Bind** — jika ya, bind nilai tersebut kepada nama `c`

Tanpa `@`, kita boleh check range tapi tidak dapat nilai:
```rust
400..=499 => format!("Client Error (??)")  // ← tidak ada nilai kod!
```

Dengan `@`:
```rust
c @ 400..=499 => format!("Client Error ({})", c)  // ← ada nilai kod!
```
</details>

---

# BAB 6: Nested Patterns 🪆

## Pattern Dalam Pattern

```rust
#[derive(Debug)]
enum Warna {
    Rgb(u8, u8, u8),
    Hsl(f64, f64, f64),
    Nama(&'static str),
}

#[derive(Debug)]
enum Elemen {
    Teks(String),
    Kotak { lebar: u32, tinggi: u32, warna: Warna },
    Kumpulan(Vec<Elemen>),
}

fn huraikan(e: &Elemen) {
    match e {
        // Match biasa
        Elemen::Teks(s) => println!("Teks: '{}'", s),

        // Nested destructure dalam struct variant
        Elemen::Kotak { warna: Warna::Nama(nama), lebar, tinggi } => {
            println!("Kotak {} ({}×{})", nama, lebar, tinggi);
        }

        Elemen::Kotak { warna: Warna::Rgb(r, g, b), lebar, tinggi } => {
            println!("Kotak RGB({},{},{}) {}×{}", r, g, b, lebar, tinggi);
        }

        Elemen::Kotak { lebar, tinggi, .. } => {
            println!("Kotak lain {}×{}", lebar, tinggi);
        }

        // Match pada Vec dalam enum
        Elemen::Kumpulan(v) if v.is_empty() => {
            println!("Kumpulan kosong");
        }

        Elemen::Kumpulan(v) => {
            println!("Kumpulan {} elemen:", v.len());
            for e2 in v { huraikan(e2); }
        }
    }
}

fn main() {
    let ui = Elemen::Kumpulan(vec![
        Elemen::Teks("Hello".into()),
        Elemen::Kotak {
            lebar: 100, tinggi: 50,
            warna: Warna::Rgb(255, 128, 0),
        },
        Elemen::Kotak {
            lebar: 200, tinggi: 80,
            warna: Warna::Nama("biru"),
        },
    ]);

    huraikan(&ui);
}
```

---

## Nested Option & Result

```rust
fn main() {
    // Option<Vec<Option<i32>>>
    let data: Option<Vec<Option<i32>>> = Some(vec![Some(1), None, Some(3)]);

    match data {
        None => println!("Tiada data"),
        Some(v) if v.is_empty() => println!("Data kosong"),
        Some(v) => {
            for (i, item) in v.iter().enumerate() {
                match item {
                    Some(n) => println!("  [{}] = {}", i, n),
                    None    => println!("  [{}] = (tiada)", i),
                }
            }
        }
    }

    // Result<Option<T>, E>
    let r: Result<Option<String>, &str> = Ok(Some("Ali".into()));
    match r {
        Ok(Some(nama))  => println!("Nama: {}", nama),
        Ok(None)        => println!("Tiada nama"),
        Err(e)          => println!("Error: {}", e),
    }

    // Flatten nested — alternatif kepada nested match
    let flat: Option<i32> = Some(Some(42)).flatten();
    println!("Flatten: {:?}", flat); // Some(42)
}
```

---

# BAB 7: if let, while let, let else ⚡

## if let — Match Ringkas

```rust
fn main() {
    let coin: Option<u32> = Some(25);

    // Cara panjang dengan match
    match coin {
        Some(nilai) => println!("Ada duit: {} sen", nilai),
        None        => {},
    }

    // Cara ringkas dengan if let
    if let Some(nilai) = coin {
        println!("Ada duit: {} sen", nilai);
    }

    // if let dengan else
    if let Some(n) = coin {
        println!("Nilai: {}", n);
    } else {
        println!("Tiada nilai");
    }

    // if let rantai
    let config: Option<&str> = Some("debug");
    if let Some("debug") = config {
        println!("Mod debug aktif");
    } else if let Some(mod_) = config {
        println!("Mod: {}", mod_);
    } else {
        println!("Mod lalai");
    }

    // if let dengan guard
    let num: Option<i32> = Some(7);
    if let Some(n) = num && n > 5 {
        println!("{} lebih dari 5", n);
    }

    // if let dengan enum
    #[derive(Debug)]
    enum Arahan { Maju(u32), Belok(i32), Berhenti }
    let a = Arahan::Maju(10);

    if let Arahan::Maju(jarak) = a {
        println!("Maju {} langkah", jarak);
    }
}
```

---

## while let — Loop Sehingga Pattern Gagal

```rust
fn main() {
    // Classic stack pop
    let mut stack = vec![1, 2, 3, 4, 5];
    while let Some(top) = stack.pop() {
        println!("Pop: {}", top); // 5, 4, 3, 2, 1
    }

    // Dengan channel receiver
    use std::sync::mpsc;
    let (tx, rx) = mpsc::channel::<i32>();

    // Hantar beberapa nilai
    for i in 1..=5 { tx.send(i).unwrap(); }
    drop(tx); // tutup channel

    // while let untuk drain channel
    while let Ok(mesej) = rx.recv() {
        println!("Terima: {}", mesej);
    }
    println!("Channel ditutup");

    // while let dengan iterator
    let mut iter = vec!["Ali", "Siti", "Amin"].into_iter();
    while let Some(nama) = iter.next() {
        println!("Nama: {}", nama);
    }
}
```

---

## let else — Anti-Pattern untuk Early Return

```rust
fn proses_input(input: &str) -> Result<i32, String> {
    // let else — sebaliknya return/break/continue/panic
    let Ok(nombor) = input.trim().parse::<i32>() else {
        return Err(format!("'{}' bukan nombor!", input));
    };

    let Some(hasil) = nombor.checked_mul(2) else {
        return Err("Overflow!".to_string());
    };

    Ok(hasil)
}

fn baca_konfigurasi(data: &str) -> Option<(String, u16)> {
    let mut bahagian = data.splitn(2, ':');

    // let else elakkan nested if let
    let Some(host) = bahagian.next() else { return None; };
    let Some(port_str) = bahagian.next() else { return None; };
    let Ok(port) = port_str.trim().parse::<u16>() else { return None; };

    Some((host.to_string(), port))
}

fn main() {
    println!("{:?}", proses_input("42"));     // Ok(84)
    println!("{:?}", proses_input("abc"));    // Err(...)
    println!("{:?}", proses_input("9999999")); // Err(Overflow)

    println!("{:?}", baca_konfigurasi("localhost:8080")); // Some(...)
    println!("{:?}", baca_konfigurasi("localhost"));      // None
    println!("{:?}", baca_konfigurasi("host:abc"));       // None
}
```

---

## let else vs match vs if let

```
let else  → untuk "jika tidak match, keluar (return/break/panic)"
            → terbaik untuk validation di awal fungsi
            → elak nesting yang dalam

if let    → untuk "jika match, buat sesuatu, kalau tidak, OK"
            → terbaik bila ada kes match yang penting

match     → untuk banyak kes, atau exhaustive check
            → terbaik bila banyak pattern berbeza

Urutan pilih:
  1. Banyak kes?            → match
  2. "Balik awal jika gagal"?  → let else
  3. "Buat bila ada"?       → if let
  4. Loop sehingga gagal?   → while let
```

---

# BAB 8: Pattern dalam Function Parameters 🔧

## Destructure dalam Parameter

```rust
// Destructure tuple dalam parameter terus
fn tambah_koordinat(&(x1, y1): &(i32, i32), &(x2, y2): &(i32, i32)) -> (i32, i32) {
    (x1 + x2, y1 + y2)
}

// Destructure struct dalam parameter
struct Titik { x: f64, y: f64 }
fn jarak_dari_asal(Titik { x, y }: &Titik) -> f64 {
    (x * x + y * y).sqrt()
}

fn main() {
    let a = (3, 4);
    let b = (1, 2);
    println!("{:?}", tambah_koordinat(&a, &b)); // (4, 6)

    let t = Titik { x: 3.0, y: 4.0 };
    println!("Jarak: {}", jarak_dari_asal(&t)); // 5.0

    // Dalam closure
    let pasangan = vec![(1, 'a'), (2, 'b'), (3, 'c')];
    pasangan.iter().for_each(|(n, c)| println!("{}: {}", n, c));

    // Closure dengan destructure struct
    #[derive(Debug)]
    struct Item { id: u32, nilai: i32 }
    let items = vec![
        Item { id: 1, nilai: 10 },
        Item { id: 2, nilai: -5 },
        Item { id: 3, nilai: 7 },
    ];

    let positif: Vec<u32> = items.iter()
        .filter(|Item { nilai, .. }| *nilai > 0)
        .map(|Item { id, .. }| *id)
        .collect();

    println!("ID positif: {:?}", positif); // [1, 3]
}
```

---

## Pattern dalam for loop

```rust
fn main() {
    // Destructure dalam for loop
    let pasangan: Vec<(i32, i32)> = vec![(1, 2), (3, 4), (5, 6)];
    for (a, b) in &pasangan {
        println!("{} + {} = {}", a, b, a + b);
    }

    // enumerate dengan destructure
    let buah = ["epal", "mangga", "pisang"];
    for (i, nama) in buah.iter().enumerate() {
        println!("[{}] {}", i, nama);
    }

    // Destructure struct dalam for
    #[derive(Debug)]
    struct Pelajar { nama: String, markah: u32 }
    let kelas = vec![
        Pelajar { nama: "Ali".into(),  markah: 85 },
        Pelajar { nama: "Siti".into(), markah: 92 },
    ];

    for Pelajar { nama, markah } in &kelas {
        println!("{}: {}", nama, markah);
    }

    // Ignore dengan _ dalam for
    let triad = vec![(1, 2, 3), (4, 5, 6), (7, 8, 9)];
    for (a, _, c) in &triad {
        println!("{} + {} = {}", a, c, a + c);
    }
}
```

---

## 🧠 Brain Teaser #4

Tulis fungsi menggunakan pattern matching yang terima `Vec<(String, Option<u32>)>` dan return jumlah semua nilai `Some`, abaikan `None`.

<details>
<summary>👀 Jawapan</summary>

```rust
fn jumlah_ada(data: &[(String, Option<u32>)]) -> u32 {
    data.iter()
        .filter_map(|(_, nilai)| *nilai)  // destructure tuple, ambil Some
        .sum()
}

// Atau dengan match eksplisit:
fn jumlah_ada_v2(data: &[(String, Option<u32>)]) -> u32 {
    let mut jumlah = 0;
    for (_, nilai) in data {
        if let Some(n) = nilai {
            jumlah += n;
        }
    }
    jumlah
}

fn main() {
    let data = vec![
        ("Ali".into(),  Some(85)),
        ("Siti".into(), None),
        ("Amin".into(), Some(70)),
        ("Zara".into(), Some(90)),
    ];
    println!("Jumlah: {}", jumlah_ada(&data));   // 245
    println!("Jumlah: {}", jumlah_ada_v2(&data)); // 245
}
```
</details>

---

# BAB 9: Pattern Lanjutan 🎯

## matches! Macro

```rust
fn main() {
    let nilai: Option<i32> = Some(7);

    // matches! — return bool, berguna dalam filter
    println!("{}", matches!(nilai, Some(x) if x > 5)); // true
    println!("{}", matches!(nilai, None));               // false

    // Sangat berguna dalam iterator
    let list = vec![Some(1), None, Some(3), None, Some(5)];
    let ada_nilai: Vec<&Option<i32>> = list.iter()
        .filter(|x| matches!(x, Some(_)))
        .collect();
    println!("{}", ada_nilai.len()); // 3

    // matches! dengan enum
    #[derive(Debug)]
    enum Status { Aktif, Tidak, Pending }
    let statuses = vec![Status::Aktif, Status::Tidak, Status::Pending, Status::Aktif];

    let aktif_count = statuses.iter()
        .filter(|s| matches!(s, Status::Aktif))
        .count();
    println!("Aktif: {}", aktif_count); // 2
}
```

---

## Or-Patterns dalam Struct

```rust
#[derive(Debug)]
enum Token {
    Plus,
    Minus,
    Kali,
    Bahagi,
    Nombor(f64),
    Huruf(String),
}

fn adalah_operator(t: &Token) -> bool {
    matches!(t, Token::Plus | Token::Minus | Token::Kali | Token::Bahagi)
}

fn nilai_nombor(t: &Token) -> Option<f64> {
    match t {
        Token::Nombor(n) => Some(*n),
        _                => None,
    }
}

fn main() {
    let tokens = vec![
        Token::Nombor(3.0),
        Token::Plus,
        Token::Nombor(4.0),
        Token::Kali,
        Token::Nombor(2.0),
    ];

    for t in &tokens {
        if adalah_operator(t) {
            println!("{:?} adalah operator", t);
        } else if let Some(n) = nilai_nombor(t) {
            println!("Nombor: {}", n);
        }
    }
}
```

---

## Refutability — Refutable vs Irrefutable

```rust
fn main() {
    // IRREFUTABLE — sentiasa match, boleh guna dalam let biasa
    let (x, y) = (1, 2);             // OK — tuple sentiasa match
    let [a, b, c] = [1, 2, 3];       // OK — array 3 elemen sentiasa match

    // REFUTABLE — mungkin tidak match, TIDAK boleh dalam let biasa
    // let Some(n) = nilai;           // ERROR! nilai mungkin None

    // Guna if let untuk refutable patterns
    let nilai: Option<i32> = Some(5);
    if let Some(n) = nilai { println!("{}", n); } // OK

    // let else untuk refutable
    let Some(n) = nilai else { return; };
    println!("n = {}", n);

    // IRREFUTABLE dalam function param — OK
    fn ambil_x(&(x, _): &(i32, i32)) -> i32 { x }
    println!("{}", ambil_x(&(3, 4))); // 3
}
```

---

## Pattern dalam impl & Traits

```rust
use std::fmt;

#[derive(Debug, Clone, PartialEq)]
enum Nilai {
    Int(i64),
    Float(f64),
    Bool(bool),
    Teks(String),
    Senarai(Vec<Nilai>),
    Tiada,
}

impl Nilai {
    fn adalah_truthy(&self) -> bool {
        match self {
            Nilai::Bool(false) | Nilai::Tiada => false,
            Nilai::Int(0)                     => false,
            Nilai::Teks(s) if s.is_empty()    => false,
            Nilai::Senarai(v) if v.is_empty() => false,
            _                                 => true,
        }
    }

    fn jenis(&self) -> &str {
        match self {
            Nilai::Int(_)     => "int",
            Nilai::Float(_)   => "float",
            Nilai::Bool(_)    => "bool",
            Nilai::Teks(_)    => "teks",
            Nilai::Senarai(_) => "senarai",
            Nilai::Tiada      => "tiada",
        }
    }
}

impl fmt::Display for Nilai {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Nilai::Int(n)     => write!(f, "{}", n),
            Nilai::Float(n)   => write!(f, "{:.2}", n),
            Nilai::Bool(b)    => write!(f, "{}", b),
            Nilai::Teks(s)    => write!(f, "\"{}\"", s),
            Nilai::Senarai(v) => {
                write!(f, "[")?;
                for (i, item) in v.iter().enumerate() {
                    if i > 0 { write!(f, ", ")?; }
                    write!(f, "{}", item)?;
                }
                write!(f, "]")
            }
            Nilai::Tiada => write!(f, "tiada"),
        }
    }
}

fn main() {
    let vals = vec![
        Nilai::Int(42),
        Nilai::Float(3.14),
        Nilai::Bool(true),
        Nilai::Teks("hello".into()),
        Nilai::Senarai(vec![Nilai::Int(1), Nilai::Int(2)]),
        Nilai::Tiada,
        Nilai::Int(0),
        Nilai::Bool(false),
    ];

    for v in &vals {
        println!("{} ({}) → truthy: {}", v, v.jenis(), v.adalah_truthy());
    }
}
```

---

# BAB 10: Mini Project — Interpreter Ekspresi 🧮

```rust
use std::collections::HashMap;
use std::fmt;

// ─── AST (Abstract Syntax Tree) ───────────────────────────────

#[derive(Debug, Clone, PartialEq)]
enum Expr {
    Nombor(f64),
    Pemboleh(String),
    BinaryOp {
        op:    Op,
        kiri:  Box<Expr>,
        kanan: Box<Expr>,
    },
    UnaryOp {
        op:    UnaryOp,
        operan: Box<Expr>,
    },
    If {
        syarat:  Box<Expr>,
        jika_ya: Box<Expr>,
        jika_tidak: Box<Expr>,
    },
    Senarai(Vec<Expr>),
}

#[derive(Debug, Clone, PartialEq)]
enum Op { Tambah, Tolak, Kali, Bahagi, KuasaDua, Mod }

#[derive(Debug, Clone, PartialEq)]
enum UnaryOp { Negatif, Mutlak }

// ─── Nilai Runtime ────────────────────────────────────────────

#[derive(Debug, Clone, PartialEq)]
enum Nilai {
    Nombor(f64),
    Senarai(Vec<Nilai>),
}

impl fmt::Display for Nilai {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Nilai::Nombor(n) => {
                if n.fract() == 0.0 && n.abs() < 1e10 {
                    write!(f, "{}", *n as i64)
                } else {
                    write!(f, "{:.4}", n)
                }
            }
            Nilai::Senarai(v) => {
                write!(f, "[")?;
                for (i, x) in v.iter().enumerate() {
                    if i > 0 { write!(f, ", ")?; }
                    write!(f, "{}", x)?;
                }
                write!(f, "]")
            }
        }
    }
}

// ─── Error ────────────────────────────────────────────────────

#[derive(Debug)]
enum InterpError {
    BahagiBukanSifar,
    PembolehTidakDitemui(String),
    JenisTidakSokong,
}

impl fmt::Display for InterpError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            InterpError::BahagiBukanSifar         => write!(f, "Bahagi dengan sifar!"),
            InterpError::PembolehTidakDitemui(v)  => write!(f, "Pemboleh '{}' tidak wujud", v),
            InterpError::JenisTidakSokong         => write!(f, "Jenis tidak disokong"),
        }
    }
}

// ─── Interpreter ──────────────────────────────────────────────

struct Interpreter {
    persekitaran: HashMap<String, Nilai>,
}

impl Interpreter {
    fn baru() -> Self {
        Interpreter { persekitaran: HashMap::new() }
    }

    fn tetap(&mut self, nama: &str, nilai: Nilai) {
        self.persekitaran.insert(nama.to_string(), nilai);
    }

    fn nilaikan(&self, expr: &Expr) -> Result<Nilai, InterpError> {
        match expr {
            // Nilai literal — terus return
            Expr::Nombor(n) => Ok(Nilai::Nombor(*n)),

            // Pembolehubah — cari dalam persekitaran
            Expr::Pemboleh(nama) => {
                self.persekitaran
                    .get(nama)
                    .cloned()
                    .ok_or_else(|| InterpError::PembolehTidakDitemui(nama.clone()))
            }

            // Operasi unary
            Expr::UnaryOp { op, operan } => {
                match (op, self.nilaikan(operan)?) {
                    (UnaryOp::Negatif, Nilai::Nombor(n)) => Ok(Nilai::Nombor(-n)),
                    (UnaryOp::Mutlak,  Nilai::Nombor(n)) => Ok(Nilai::Nombor(n.abs())),
                    _                                     => Err(InterpError::JenisTidakSokong),
                }
            }

            // Operasi binary — nested pattern matching!
            Expr::BinaryOp { op, kiri, kanan } => {
                match (op, self.nilaikan(kiri)?, self.nilaikan(kanan)?) {
                    // Operasi nombor + nombor
                    (Op::Tambah,    Nilai::Nombor(a), Nilai::Nombor(b)) => Ok(Nilai::Nombor(a + b)),
                    (Op::Tolak,     Nilai::Nombor(a), Nilai::Nombor(b)) => Ok(Nilai::Nombor(a - b)),
                    (Op::Kali,      Nilai::Nombor(a), Nilai::Nombor(b)) => Ok(Nilai::Nombor(a * b)),
                    (Op::KuasaDua,  Nilai::Nombor(a), Nilai::Nombor(b)) => Ok(Nilai::Nombor(a.powf(b))),
                    (Op::Mod,       Nilai::Nombor(a), Nilai::Nombor(b)) => Ok(Nilai::Nombor(a % b)),

                    (Op::Bahagi, Nilai::Nombor(_), Nilai::Nombor(b)) if b == 0.0 => {
                        Err(InterpError::BahagiBukanSifar)
                    }
                    (Op::Bahagi,    Nilai::Nombor(a), Nilai::Nombor(b)) => Ok(Nilai::Nombor(a / b)),

                    // Gabung senarai
                    (Op::Tambah, Nilai::Senarai(mut a), Nilai::Senarai(b)) => {
                        a.extend(b);
                        Ok(Nilai::Senarai(a))
                    }

                    _ => Err(InterpError::JenisTidakSokong),
                }
            }

            // If expression
            Expr::If { syarat, jika_ya, jika_tidak } => {
                match self.nilaikan(syarat)? {
                    Nilai::Nombor(n) if n != 0.0 => self.nilaikan(jika_ya),
                    Nilai::Nombor(_)             => self.nilaikan(jika_tidak),
                    _                            => Err(InterpError::JenisTidakSokong),
                }
            }

            // Senarai — nilaikan setiap elemen
            Expr::Senarai(elems) => {
                let hasil: Result<Vec<Nilai>, _> = elems.iter()
                    .map(|e| self.nilaikan(e))
                    .collect();
                Ok(Nilai::Senarai(hasil?))
            }
        }
    }
}

// ─── Helper untuk bina Expr ───────────────────────────────────

fn num(n: f64) -> Expr { Expr::Nombor(n) }
fn var(s: &str) -> Expr { Expr::Pemboleh(s.into()) }
fn tambah(a: Expr, b: Expr) -> Expr {
    Expr::BinaryOp { op: Op::Tambah, kiri: Box::new(a), kanan: Box::new(b) }
}
fn tolak(a: Expr, b: Expr) -> Expr {
    Expr::BinaryOp { op: Op::Tolak, kiri: Box::new(a), kanan: Box::new(b) }
}
fn kali(a: Expr, b: Expr) -> Expr {
    Expr::BinaryOp { op: Op::Kali, kiri: Box::new(a), kanan: Box::new(b) }
}
fn bahagi(a: Expr, b: Expr) -> Expr {
    Expr::BinaryOp { op: Op::Bahagi, kiri: Box::new(a), kanan: Box::new(b) }
}
fn kuasa(a: Expr, b: Expr) -> Expr {
    Expr::BinaryOp { op: Op::KuasaDua, kiri: Box::new(a), kanan: Box::new(b) }
}
fn jika(s: Expr, y: Expr, t: Expr) -> Expr {
    Expr::If { syarat: Box::new(s), jika_ya: Box::new(y), jika_tidak: Box::new(t) }
}

// ─── Main ──────────────────────────────────────────────────────

fn main() {
    let mut interp = Interpreter::baru();

    // Tetapkan pembolehubah
    interp.tetap("x",   Nilai::Nombor(10.0));
    interp.tetap("y",   Nilai::Nombor(3.0));
    interp.tetap("pi",  Nilai::Nombor(std::f64::consts::PI));

    let ekspresi: Vec<(&str, Expr)> = vec![
        // 10 + 3
        ("x + y",        tambah(var("x"), var("y"))),
        // (10 + 3) * 2
        ("(x+y) * 2",    kali(tambah(var("x"), var("y")), num(2.0))),
        // 10^2 + 3^2
        ("x² + y²",      tambah(kuasa(var("x"), num(2.0)), kuasa(var("y"), num(2.0)))),
        // if x > 5 then x else y  (guna x - 5 sebagai syarat)
        ("if x>5: x, y", jika(tolak(var("x"), num(5.0)), var("x"), var("y"))),
        // 10 / 3
        ("x / y",        bahagi(var("x"), var("y"))),
        // bahagi dengan sifar
        ("x / 0",        bahagi(var("x"), num(0.0))),
        // pemboleh tidak wujud
        ("z",            var("z")),
        // senarai
        ("[1,2,3]+[4,5]", tambah(
            Expr::Senarai(vec![num(1.0), num(2.0), num(3.0)]),
            Expr::Senarai(vec![num(4.0), num(5.0)]),
        )),
    ];

    println!("{:-<45}", "");
    println!("{:<20} {}", "Ekspresi", "Hasil");
    println!("{:-<45}", "");

    for (label, expr) in &ekspresi {
        match interp.nilaikan(expr) {
            Ok(nilai)  => println!("{:<20} = {}", label, nilai),
            Err(e)     => println!("{:<20} ! {}", label, e),
        }
    }
    println!("{:-<45}", "");
}
```

---

# 📋 Rujukan Pantas — Pattern Matching Cheat Sheet

## Semua Pattern

```rust
// Literal
match x { 1 => .., "hello" => .., true => .. }

// Range
match x { 1..=5 => .., 'a'..='z' => .. }

// Wildcard
match x { _ => .. }       // tangkap semua, ignore nilai
match x { _x => .. }      // tangkap semua, bind tapi mark sebagai "mungkin unused"

// Variable binding
match x { n => println!("{}", n) }

// Tuple
match (a, b) { (1, 2) => .., (x, y) => .. }

// Struct
match p { Point { x, y } => .., Point { x: 0, .. } => .. }

// Enum
match e { Some(x) => .., None => .. }
match e { Ok(v) => .., Err(e) => .. }

// OR pattern
match x { 1 | 2 | 3 => .., 'a' | 'b' => .. }

// Guard
match x { n if n > 0 => .., _ => .. }

// @ binding
match x { n @ 1..=5 => println!("{}", n) }

// Slice
match s { [] => .., [x] => .., [x, ..] => .., [x, .., z] => .., [a, rest @ ..] => .. }

// Reference
match &x { &n => .. }
match x  { ref n => .. }  // bind by reference tanpa move
```

## Shorthand Patterns

```rust
// if let
if let Some(x) = opt { .. }
if let Ok(v) = res { .. }
if let Enum::Variant(data) = val { .. }

// if let chain (Rust 1.64+)
if let Some(x) = opt && x > 5 { .. }

// while let
while let Some(x) = iter.next() { .. }
while let Ok(msg) = rx.recv() { .. }

// let else
let Some(x) = opt else { return; };
let Ok(v) = parse() else { return Err(...); };

// matches! macro
matches!(val, Some(x) if x > 0)
matches!(val, Ok(_) | Err(_))
```

## Peraturan Pattern

```
Irrefutable  → sentiasa match → boleh guna dalam let, fn param, for
Refutable    → mungkin tidak match → mesti dalam if let, while let, match

Exhaustive   → compiler check semua kes dalam match
Guards       → tidak dikira untuk exhaustiveness
Wildcard _   → irrefutable, catch-all
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **JSON Parser Mini** — parse string JSON ringkas guna pattern matching pada chars
2. **Calculator dengan Precedence** — eval `3 + 4 * 2` guna AST + pattern matching
3. **State Machine** — traffic light dengan enum + match exhaustive
4. **Command Parser** — parse arahan CLI `add item`, `remove 3`, `list all`
5. **Type Checker Mini** — semak type ekspresi sebelum evaluate

---

*Advanced Match in Rust — dari `match` asas hingga nested patterns dan interpreter penuh. Selamat belajar!* 🦀
