# 🔑 Ownership dalam Rust — Topik PALING PENTING

> Faham ini = faham Rust. Tidak faham ini = semua susah.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Kenapa Ownership Wujud?

```
Masalah memori dalam bahasa lain:

C/C++:                         Java/Python/PHP:
  char* p = malloc(100);         String s = "hello";
  free(p);                       // GC handle memory
  *p = 'x'; ← USE AFTER FREE!   // Selamat tapi LAMBAT
  // Program crash! 💥           // GC = overhead!

Rust penyelesaian:
  Ownership system = selamat DAN laju tanpa GC!
  Compiler jaga memori semasa compile time.
  Tiada runtime overhead. Tiada crash.

Ini sebab Rust 9 tahun berturut jadi bahasa
paling "loved" dalam Stack Overflow Survey!
```

---

## Tiga Peraturan Ownership

```
┌─────────────────────────────────────────────────────┐
│  PERATURAN 1:                                       │
│  Setiap nilai ada tepat SATU owner.                 │
│                                                     │
│  PERATURAN 2:                                       │
│  Hanya boleh ada SATU owner pada satu-satu masa.    │
│                                                     │
│  PERATURAN 3:                                       │
│  Apabila owner keluar scope, nilai di-DROP (bebas). │
└─────────────────────────────────────────────────────┘
```

---

## Peta Pembelajaran

```
Bab 1  → Memory Asas — Stack vs Heap
Bab 2  → Ownership Rules (3 Peraturan)
Bab 3  → Move — Pindah Ownership
Bab 4  → Copy — Salin Terus
Bab 5  → Clone — Salin Explicit
Bab 6  → Ownership dalam Fungsi
Bab 7  → Borrow — Pinjam dengan &
Bab 8  → Mutable Borrow — &mut
Bab 9  → Dangling Reference
Bab 10 → Mini Project: Inventory System
```

---

# BAB 1: Memory Asas — Stack vs Heap 🏗️

## Stack vs Heap — Perbezaan Kritikal

```
STACK (Tumpukan):
  ┌─────────┐  ← top
  │  c = 3  │
  │  b = 2  │
  │  a = 1  │
  └─────────┘  ← bottom

  ✔ Sangat LAJU — push/pop O(1)
  ✔ Saiz mesti DIKETAHUI semasa compile
  ✔ Auto-cleanup apabila fungsi tamat
  ✗ Saiz terhad (biasanya ~8MB)

HEAP (Timbunan):
  ┌──────────────────────────────┐
  │  [tidak tersusun]            │
  │  data "Hello" ada di sini    │
  │  data vec [1,2,3] ada di sini│
  │  ...                         │
  └──────────────────────────────┘

  ✔ Saiz FLEKSIBEL — boleh tumbuh semasa runtime
  ✔ Boleh simpan data besar
  ✗ Lebih LAMBAT (perlu allocate/deallocate)
  ✗ Mesti free bila tidak pakai!
```

```rust
fn main() {
    // Stack — saiz tetap, auto manage
    let a: i32 = 5;         // 4 bytes dalam stack
    let b: f64 = 3.14;      // 8 bytes dalam stack
    let c: bool = true;     // 1 byte dalam stack
    let arr: [i32; 3] = [1, 2, 3]; // 12 bytes dalam stack

    // Heap — saiz dinamik, Rust manage via ownership
    let s = String::from("hello"); // data dalam heap
    //      ↑ dalam stack: (ptr, len, cap) = 24 bytes
    //                                       ↓
    //      dalam heap: [ h | e | l | l | o ] = 5 bytes

    let v = vec![1, 2, 3];  // sama — metadata stack, data heap

    println!("{} {} {} {:?}", a, b, c, arr);
    println!("{}", s);
    println!("{:?}", v);
}
```

---

## Visualisasi String dalam Memori

```
let s = String::from("hello");

Stack                 Heap
┌──────────────┐     ┌───────────────────┐
│ ptr  ─────────────▶│ h │ e │ l │ l │ o │
│ len  = 5     │     └───────────────────┘
│ cap  = 5     │       index: 0  1  2  3  4
└──────────────┘

ptr = alamat dalam heap
len = berapa aksara ada (5)
cap = berapa ruang yang diallocate (5)
```

---

## 🧠 Brain Teaser #1

Antara jenis ini, mana yang disimpan dalam stack dan mana dalam heap?

```rust
let a: i32 = 42;
let b: &str = "hello";    // string literal
let c: String = String::from("world"); // owned string
let d: f64 = 3.14;
let e: Vec<i32> = vec![1, 2, 3];
let f: [i32; 5] = [0; 5]; // fixed array
```

<details>
<summary>👀 Jawapan</summary>

```
Stack SAHAJA:
  a: i32        → 4 bytes dalam stack
  d: f64        → 8 bytes dalam stack
  f: [i32; 5]  → 20 bytes dalam stack (saiz tetap, tahu semasa compile)

Stack + Data dalam Biner (bukan heap):
  b: &str       → pointer (8 bytes) dalam stack
                  data "hello" dalam biner (program binary) — bukan heap!
                  sebab itu 'static lifetime

Stack + Heap:
  c: String     → (ptr, len, cap) dalam stack, data "world" dalam heap
  e: Vec<i32>   → (ptr, len, cap) dalam stack, data [1,2,3] dalam heap
```

**Insight penting:** `&str` (string literal) vs `String` (owned string):
- `"hello"` → data dalam binary, tidak boleh tukar
- `String::from("hello")` → data dalam heap, boleh tukar
</details>

---

# BAB 2: Ownership Rules (3 Peraturan) 📏

## Peraturan 1: Satu Owner

```rust
fn main() {
    // Setiap nilai hanya ada SATU owner
    let s = String::from("hello"); // 's' adalah owner

    // Hanya 's' yang bertanggungjawab untuk 'free' data ini
    // Tiada orang lain boleh tuntut ownership
}
// 's' keluar scope → data "hello" di-DROP (free) OTOMATIK
// Tiada memory leak! Tiada manual free!
```

---

## Peraturan 2: Move Bila Assign Baru

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;  // ← MOVE! ownership pindah ke s2

    // s1 TIDAK LAGI VALID!
    // println!("{}", s1); // ← ERROR! value used after move

    println!("{}", s2); // ✔ s2 adalah owner baru
}
```

```
Sebelum move:             Selepas let s2 = s1:
s1 owns "hello"           s1 = INVALID (☠)
                          s2 owns "hello"

Stack    Heap             Stack    Heap
┌────┐  ┌─────┐          ┌────┐  ┌─────┐
│ s1 │─▶│hello│          │ s1 │☠ │hello│
└────┘  └─────┘          └────┘  └─────┘
                          ┌────┐
                          │ s2 │─────────▶
                          └────┘

Kenapa? Kalau KEDUA-DUA s1 dan s2 valid, siapa yang akan
free "hello"? Dua kali free = double free bug! 💥
Rust selesaikan ini dengan Move.
```

---

## Peraturan 3: Drop Apabila Keluar Scope

```rust
fn main() {
    let s1 = String::from("A"); // s1 lahir

    {
        let s2 = String::from("B"); // s2 lahir
        let s3 = String::from("C"); // s3 lahir
        println!("{} {} {}", s1, s2, s3);
    } // ← s3 drop DULU, kemudian s2 (LIFO — last in, first out)
      //   "B" dan "C" freed dari heap

    println!("{}", s1); // s1 masih hidup
} // ← s1 drop, "A" freed dari heap
```

```
Timeline:
  s1 lahir →──────────────────────────────→ s1 drop
  s2 lahir →──────────────→ s2 drop
  s3 lahir →──────→ s3 drop

Order drop: s3 → s2 → s1 (terbalik dari urutan cipta)
```

---

## 🧠 Brain Teaser #2

Berapa kali "hello" di-free dalam kod ini?

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;
    let s3 = s2;
    // s1, s2, s3 semua keluar scope di sini
}
```

<details>
<summary>👀 Jawapan</summary>

**Satu kali sahaja!**

- `s1` di-move ke `s2` → s1 tidak valid, s2 adalah owner
- `s2` di-move ke `s3` → s2 tidak valid, s3 adalah owner
- Hanya `s3` yang keluar scope sebagai owner yang sah
- `s3` drop → "hello" di-free SEKALI sahaja

Ini yang Rust prevent! Dalam bahasa lain tanpa GC:
- s1, s2, s3 semua point ke "hello"
- Bila semua drop → free 3 kali = **double free bug!** 💥

Rust selesaikan ini dengan Move semantics.
</details>

---

# BAB 3: Move — Pindah Ownership 🏃

## Move dalam Pelbagai Situasi

```rust
fn main() {
    // Situasi 1: Assign ke variable lain
    let a = String::from("satu");
    let b = a;          // a MOVED ke b
    // println!("{}", a); // ← ERROR!
    println!("{}", b);  // ✔

    // Situasi 2: Masuk Vec
    let s = String::from("dua");
    let mut v = Vec::new();
    v.push(s);          // s MOVED ke dalam v
    // println!("{}", s); // ← ERROR! s sudah dalam v
    println!("{:?}", v);

    // Situasi 3: Dalam struct
    let teks = String::from("tiga");
    let wrapper = Pembungkus { nilai: teks }; // teks MOVED ke struct
    // println!("{}", teks); // ← ERROR!
    println!("{}", wrapper.nilai);

    // Situasi 4: Return dari fungsi (move keluar)
    let hasil = buat_string(); // ownership dari fungsi ke main
    println!("{}", hasil);
}

struct Pembungkus { nilai: String }

fn buat_string() -> String {
    let s = String::from("empat");
    s  // ownership pindah ke caller — s tidak di-drop!
}
```

---

## Move dalam Match & if let

```rust
fn main() {
    let opt = Some(String::from("hello"));

    // Match CONSUME option (move)
    match opt {
        Some(s) => println!("Ada: {}", s), // s MOVED dari opt
        None    => println!("Tiada"),
    }
    // opt tidak boleh guna sekarang (moved dalam match arm)
    // println!("{:?}", opt); // ← ERROR!

    // Kalau nak guna opt selepas match, guna &:
    let opt2 = Some(String::from("world"));
    match &opt2 {        // borrow, tidak move!
        Some(s) => println!("Ada: {}", s), // s = &String, tidak move
        None    => println!("Tiada"),
    }
    println!("{:?}", opt2); // ✔ opt2 masih valid!
}
```

---

## Move Semantics — Visualisasi Lengkap

```
fn ambil_ownership(s: String) { // s dapat ownership
    println!("{}", s);
}                                // s keluar scope → heap data di-drop

fn pulangkan_ownership(s: String) -> String {
    s  // pulangkan ownership ke caller
}

fn main() {
    let s1 = String::from("hello");

    ambil_ownership(s1);        // s1 MOVED ke fungsi
    // println!("{}", s1);      // ← ERROR! s1 dah gone

    let s2 = String::from("world");
    let s3 = pulangkan_ownership(s2); // s2 MOVED ke fungsi
                                       // kemudian MOVED balik ke s3
    println!("{}", s3); // ✔ s3 adalah owner
}
```

---

# BAB 4: Copy — Salin Terus ✂️

## Apa Itu Copy?

```
Move berlaku untuk data dalam HEAP (tidak tahu saiz semasa compile).

Copy berlaku untuk data dalam STACK SAHAJA (tahu saiz semasa compile):
  - integers:  i8, i16, i32, i64, i128, isize
  - unsigned:  u8, u16, u32, u64, u128, usize
  - floats:    f32, f64
  - bool
  - char
  - tuples (kalau semua element adalah Copy)
  - fixed arrays (kalau element adalah Copy)
  - references: &T (pointer disalin, bukan data)
```

```rust
fn main() {
    // Copy types — assign buat SALINAN, kedua-dua valid
    let x = 5i32;
    let y = x;          // COPY! x masih valid
    println!("{} {}", x, y);  // ✔ kedua-dua OK!

    let a = 3.14f64;
    let b = a;          // COPY
    println!("{} {}", a, b);  // ✔

    let t1 = (1, 2, 3);  // tuple of i32 — Copy
    let t2 = t1;          // COPY
    println!("{:?} {:?}", t1, t2);  // ✔

    // Comparison dengan non-Copy (String)
    let s1 = String::from("hello");
    let s2 = s1;        // MOVE! s1 tidak valid
    // println!("{}", s1); // ← ERROR!
    println!("{}", s2); // ✔

    // KENAPA i32 Copy tapi String tidak?
    // i32 = 4 bytes dalam stack, mudah salin
    // String = data dalam heap, tidak boleh "shallow copy"
    //          kerana dua pointer ke heap yang sama = double free!
}
```

---

## Copy vs Move — Perbandingan Terus

```
COPY TYPES (stack-only data):

  let x = 5;
  let y = x;

  Stack:
  ┌───┐   ┌───┐
  │ 5 │   │ 5 │    ← dua salinan dalam stack, bebas!
  └───┘   └───┘
    x       y      (kedua-dua valid)

MOVE TYPES (heap data):

  let s1 = String::from("hi");
  let s2 = s1;

  Stack:   Heap:
  ┌────┐
  │ s1 │ ☠ (tidak valid)
  └────┘
  ┌────┐  ┌────┐
  │ s2 │─▶│ hi │  ← hanya s2 yang valid, s1 tidak lagi tunjuk ke sini
  └────┘  └────┘

  Kenapa? Kalau buat salinan stack, DUA pointer ke heap yang sama!
  Bila kedua-dua drop → double free! Rust prevent ini dengan Move.
```

---

## Custom Type dengan Copy

```rust
// Implement Copy untuk struct — hanya kalau semua field Copy!
#[derive(Debug, Clone, Copy)]
struct Titik {
    x: f64,  // f64 adalah Copy
    y: f64,  // f64 adalah Copy
}

// TIDAK boleh Copy jika ada String atau Vec!
// #[derive(Clone, Copy)]  // ← ERROR jika ada String
// struct TidakBolehCopy {
//     x: f64,
//     nama: String,  // String bukan Copy!
// }

fn cetak_titik(t: Titik) {  // ambil Titik (Copy)
    println!("({}, {})", t.x, t.y);
}

fn main() {
    let t1 = Titik { x: 3.0, y: 4.0 };
    let t2 = t1;          // COPY kerana Titik implement Copy
    println!("{:?}", t1); // ✔ t1 masih valid!
    println!("{:?}", t2);

    cetak_titik(t1);       // COPY ke fungsi
    println!("{:?}", t1); // ✔ t1 masih valid!
}
```

---

## 🧠 Brain Teaser #3

Adakah kod ini compile? Kenapa?

```rust
fn main() {
    let a = (1, 2.0, true);     // A
    let b = a;
    println!("{:?} {:?}", a, b);

    let c = (1, String::from("x")); // B
    let d = c;
    println!("{:?} {:?}", c, d);

    let e = [1, 2, 3, 4, 5];   // C
    let f = e;
    println!("{:?} {:?}", e, f);
}
```

<details>
<summary>👀 Jawapan</summary>

- **A** — ✔ Compile! `(i32, f64, bool)` semua Copy → tuple pun Copy. `a` dan `b` kedua-duanya valid.

- **B** — ❌ TIDAK compile! `(i32, String)` mengandungi `String` yang bukan Copy → tuple pun bukan Copy → MOVE. `c` tidak valid selepas `let d = c`.

- **C** — ✔ Compile! `[i32; 5]` adalah fixed array di mana `i32` adalah Copy → array pun Copy. `e` dan `f` kedua-duanya valid.

**Peraturan:** Tuple/Array adalah Copy hanya jika SEMUA elemen/field adalah Copy.
</details>

---

# BAB 5: Clone — Salin Explicit 🧬

## Bila Perlu Clone

```rust
fn main() {
    let s1 = String::from("hello");

    // Nak guna s1 tapi juga nak buat s2 yang bebas?
    // Guna clone() — buat salinan PENUH (deep copy)
    let s2 = s1.clone();

    println!("s1 = {}", s1); // ✔ s1 masih valid
    println!("s2 = {}", s2); // ✔ s2 adalah salinan bebas

    // Apa yang berlaku dalam memori:
    // s1.clone() = allocate heap baru, salin semua data
    // s1 dan s2 ada data BERBEZA dalam heap (walaupun isinya sama)
}
```

```
Selepas clone():

  Stack    Heap A        Heap B
  ┌────┐  ┌──────┐     ┌──────┐
  │ s1 │─▶│hello │     │hello │◀─┌────┐
  └────┘  └──────┘     └──────┘  │ s2 │
                                  └────┘

  Dua salinan dalam heap yang BERBEZA — bebas untuk drop secara berasingan!
```

---

## Clone vs Copy — Perbezaan Utama

```
COPY:
  - Implisit (automatik)
  - Hanya untuk data kecil dalam stack
  - Sangat CEPAT (stack copy = memcpy bytes)
  - Tidak boleh customize

CLONE:
  - Explicit (perlu panggil .clone())
  - Untuk apa-apa jenis
  - Mungkin LAMBAT (allocate heap baru)
  - Boleh customize (impl Clone for MyType)

Peraturan mudah:
  Copy = "murah, automatik, stack sahaja"
  Clone = "mungkin mahal, pilihan kita, apa-apa type"
```

```rust
fn main() {
    // Clone Vec
    let v1 = vec![1, 2, 3, 4, 5];
    let v2 = v1.clone();
    println!("{:?}", v1); // ✔
    println!("{:?}", v2); // ✔ salinan bebas

    // Clone nested struct
    #[derive(Debug, Clone)]
    struct Rekod {
        nama: String,
        data: Vec<i32>,
    }

    let r1 = Rekod {
        nama: "Ali".into(),
        data: vec![1, 2, 3],
    };
    let r2 = r1.clone(); // DEEP clone — semua field di-clone
    println!("{:?}", r1);
    println!("{:?}", r2);

    // Clone mahal! Elak kalau boleh, guna & (borrow) sebagai gantinya
}
```

---

# BAB 6: Ownership dalam Fungsi 🔧

## Fungsi Ambil Ownership

```rust
fn main() {
    let s = String::from("hello");

    ambil(s);              // s MOVED ke dalam fungsi
    // println!("{}", s);  // ← ERROR! s dah tidak valid

    let x = 5;
    salin(x);              // x COPIED ke dalam fungsi
    println!("{}", x);     // ✔ x masih valid (Copy type!)
}

fn ambil(teks: String) {    // teks dapat ownership
    println!("{}", teks);
} // teks keluar scope → data di-DROP di sini!

fn salin(nombor: i32) {     // nombor dapat salinan
    println!("{}", nombor);
} // nombor keluar scope → tapi i32 drop tidak buat apa (stack!)
```

---

## Pulangkan Ownership

```rust
fn main() {
    let s1 = String::from("hello");

    // Cara BURUK — terpaksa pulang ownership kalau nak guna lagi
    let s2 = pulangkan(s1);
    println!("{}", s2); // ✔

    // Cara yang lebih baik — guna borrowing (&)
    // lihat Bab 7!
}

fn pulangkan(s: String) -> String {
    println!("Dalam fungsi: {}", s);
    s  // pulangkan ownership — s tidak di-drop!
}

// Ini sangat verbose... bayangkan untuk 3 variable!
fn verbose(s1: String, s2: String, s3: String) -> (String, String, String) {
    // Kena return semua tiga untuk caller boleh guna semula
    (s1, s2, s3) // sangat terseksa!
}

// Penyelesaian yang betul — borrowing!
fn lebih_baik(s1: &str, s2: &str, s3: &str) {
    println!("{} {} {}", s1, s2, s3);
    // Caller tidak perlu ambil balik — kita hanya pinjam!
}
```

---

## Pola Ownership Biasa

```rust
fn main() {
    // Pola 1: Consume dan return (transform)
    let s = String::from("hello");
    let s = transform(s);          // s "reborn" dengan nilai baru
    println!("{}", s);             // "HELLO"

    // Pola 2: Borrow untuk baca
    let s2 = String::from("world");
    let panjang = kira_panjang(&s2); // pinjam, tidak ambil ownership
    println!("{} ada {} aksara", s2, panjang); // s2 masih valid!

    // Pola 3: Mutable borrow untuk ubah
    let mut s3 = String::from("hi");
    tambah_seru(&mut s3);           // pinjam untuk ubah
    println!("{}", s3);             // "hi!!!"  s3 masih valid!
}

fn transform(s: String) -> String {
    s.to_uppercase()  // consume s, return baru
}

fn kira_panjang(s: &str) -> usize {
    s.len()  // pinjam sahaja
}

fn tambah_seru(s: &mut String) {
    s.push_str("!!!");  // ubah melalui mutable reference
}
```

---

# BAB 7: Borrow — Pinjam dengan & 📖

## Konsep Borrow

```
OWNERSHIP = anda ada buku, anda bertanggungjawab.
BORROW    = anda pinjam buku dari library.
            anda boleh BACA tapi library masih punya buku.
            bila pinjaman habis, buku kembali ke library.

let s = String::from("hello"); // s = owner
let r = &s;                    // r = borrow dari s
//      ↑
//  & = "reference to" = pinjam!
```

```rust
fn main() {
    let s1 = String::from("hello");
    let r1 = &s1;       // r1 pinjam s1
    let r2 = &s1;       // r2 pinjam s1 (boleh banyak &!)
    let r3 = &s1;       // r3 pinjam s1 (boleh banyak &!)

    println!("{} {} {}", r1, r2, r3); // ✔

    println!("{}", s1); // ✔ s1 masih owner dan valid!
    // r1, r2, r3 selesai (last use di println atas)

    // Guna dalam fungsi
    fn hitung_vokal(s: &String) -> usize {
        s.chars().filter(|c| "aeiouAEIOU".contains(*c)).count()
    }

    let vokal = hitung_vokal(&s1);
    println!("{} ada {} vokal", s1, vokal); // ✔ s1 masih valid!
}
```

---

## Peraturan Borrow

```
PERATURAN BORROW:

  Pada satu-satu masa, SAMA ADA:
    ┌──────────────────────────────────────┐
    │  Banyak & (shared/immutable borrow)  │
    │         ATAU                         │
    │  Tepat SATU &mut (mutable borrow)    │
    │  (tiada & lain boleh wujud serentak) │
    └──────────────────────────────────────┘

KENAPA?
  Banyak reader   = OK (tiada siapa tukar data)
  Satu writer     = OK (dia yang kawal)
  Reader + Writer = BAHAYA! (race condition, data inconsistency)
```

```rust
fn main() {
    let mut s = String::from("hello");

    // ✔ Banyak immutable borrow serentak — OK
    let r1 = &s;
    let r2 = &s;
    println!("{} {}", r1, r2); // r1, r2 selesai di sini

    // ✔ Selepas immutable borrow selesai, mutable borrow OK
    let r3 = &mut s;
    r3.push_str(" world");
    println!("{}", r3); // r3 selesai

    // ✔ Selepas mutable borrow selesai, boleh buat baru
    println!("{}", s);

    // ❌ TIDAK BOLEH — immutable + mutable serentak
    // let r4 = &s;
    // let r5 = &mut s;  // ERROR! r4 masih aktif
    // println!("{} {}", r4, r5);
}
```

---

## Non-Lexical Lifetimes (NLL) — Rust 2018+

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;          // immutable borrow mula
    println!("{}", r1);   // LAST USE r1 ← borrow berakhir di sini!
    // (bukan di hujung scope seperti versi Rust lama)

    let r2 = &mut s;      // ✔ OK! r1 sudah selesai
    r2.push_str(" world");
    println!("{}", r2);

    // Ini yang orang panggil NLL (Non-Lexical Lifetimes)
    // Borrow berakhir di LAST USE, bukan hujung scope {}
    // Ini buat Rust lebih ergonomik!
}
```

---

## 🧠 Brain Teaser #4

Adakah kod ini compile?

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let r = &v[0]; // borrow elemen pertama

    v.push(4);     // cuba tambah elemen
    println!("{}", r); // guna r
}
```

<details>
<summary>👀 Jawapan</summary>

**Tidak compile!**

Masalahnya lebih halus: `v.push(4)` memerlukan `&mut v`. Tapi `r = &v[0]` masih ada borrow aktif ke dalam `v`.

```
r = &v[0]   ← borrow immutable (ke dalam Vec)
v.push(4)   ← Vec perlu &mut self, mungkin REALLOCATE memory!
              Kalau reallocate, r akan tunjuk ke memori LAMA yang sudah freed!
println!(r) ← DANGLING REFERENCE! 💥
```

Rust prevent ini untuk lindungi daripada iterator invalidation bug yang biasa dalam C++!

**Penyelesaian:**
```rust
let mut v = vec![1, 2, 3];
let first = v[0]; // COPY nilai (i32 adalah Copy), bukan reference!
v.push(4);
println!("{}", first); // ✔ first adalah i32, bukan reference ke v
```

Atau:
```rust
let mut v = vec![1, 2, 3];
v.push(4);           // push dulu
let r = &v[0];       // borrow selepas push
println!("{}", r);   // ✔
```
</details>

---

# BAB 8: Mutable Borrow — &mut 🖊️

## &mut — Borrow untuk Ubah

```rust
fn main() {
    let mut s = String::from("hello");

    ubah(&mut s);           // hantar mutable reference
    println!("{}", s);      // "hello, world!"  ← s berubah!

    // Mutable reference boleh UBAH nilai
    let mut x = 5;
    let rx = &mut x;
    *rx += 10;              // * = dereference — akses nilai sebenar
    println!("{}", x);      // 15

    // Mutable reference dalam struct update
    let mut v = vec![1, 2, 3];
    let rv = &mut v;
    rv.push(4);             // ubah melalui reference
    println!("{:?}", v);    // [1, 2, 3, 4]
}

fn ubah(s: &mut String) {
    s.push_str(", world");  // ubah data melalui reference
}
```

---

## Hanya SATU &mut pada Satu Masa

```rust
fn main() {
    let mut s = String::from("hello");

    // ✔ Satu &mut — OK
    let r1 = &mut s;
    println!("{}", r1);  // r1 selesai

    // ✔ &mut baru selepas yang lama selesai — OK
    let r2 = &mut s;
    println!("{}", r2);

    // ❌ DUA &mut serentak — ERROR!
    // let r3 = &mut s;
    // let r4 = &mut s;   // ERROR! dua mutable reference serentak
    // println!("{} {}", r3, r4);

    // ❌ & dan &mut serentak — ERROR!
    // let r5 = &s;
    // let r6 = &mut s;   // ERROR! immutable + mutable borrow serentak
    // println!("{} {}", r5, r6);
}
```

---

## Kenapa Hanya Satu &mut?

```
Bayangkan bank account:

  Thread 1: ambil baki → baki = RM1000
  Thread 2: ambil baki → baki = RM1000
  Thread 1: tolak RM500 → baki = RM500, simpan
  Thread 2: tolak RM500 → baki = RM500, simpan

  Akhirnya: RM500 bukan RM0! DATA RACE BUG! 💥

Dengan &mut hanya satu:
  Thread 1 lock → Thread 2 tunggu
  Thread 1 siap → Thread 2 baru boleh proceed
  Tiada data race!

Rust enforce ini pada compile time — tiada runtime overhead!
```

---

# BAB 9: Dangling Reference 💀

## Apa Itu Dangling Reference?

```
Dangling Reference = reference yang tunjuk ke memori yang SUDAH BEBAS.

Dalam C++:
  int* p;
  {
      int x = 5;
      p = &x;         // p tunjuk ke x
  }                   // x di-destroy di sini!
  *p = 10;            // 💥 SEGFAULT! p tunjuk ke sampah!

Rust PREVENT ini pada compile time!
Tidak boleh ada dangling reference dalam safe Rust.
```

---

## Contoh Dangling — Rust Tangkap!

```rust
// ❌ TIDAK COMPILE — dangling reference
// fn buat_dangling() -> &String {
//     let s = String::from("hello");
//     &s   // return reference ke s
// }        // ← s di-DROP di sini! Reference tunjuk ke TIADA!

// ✔ Betul — return owned String
fn buat_string() -> String {
    let s = String::from("hello");
    s  // move ownership keluar — s tidak di-drop!
}

// ✔ Betul — return reference ke data yang hidup lebih lama
fn panjang(s: &String) -> usize {
    s.len()  // reference s datang dari luar, dijamin hidup
}

fn main() {
    let s = buat_string();
    println!("{}", s);

    let p = panjang(&s);
    println!("{}", p);
}
```

---

## Dangling dengan Scope

```rust
fn main() {
    let r;
    {
        let x = 5;
        r = &x;         // ERROR! x tidak hidup cukup lama
        // r cuba outlive x!
    } // x di-drop di sini
    // println!("{}", r); // ← jika dibiar, r = dangling!

    // Rust error:
    // error[E0597]: `x` does not live long enough

    // ✔ Betul — r dan x dalam scope yang sama
    let x2 = 5;
    let r2 = &x2;
    println!("{}", r2); // ✔ x2 masih hidup
}
```

---

## Visualisasi Lifetime

```
fn main() {            ← main scope mula
    let r;             ← r lahir (tapi tiada nilai lagi)
    {
        let x = 5;     ← x lahir
        r = &x;        ← r cuba point ke x
    }                  ← x MATI di sini!
    println!("{}", r); ← r cuba point ke memori yang dah mati!
                          COMPILER: TIDAK BOLEH! 🛑
}

Lifetime r:  ├─────────────────────────────────────────┤
Lifetime x:       ├──────────┤
                  ↑          ↑
                  x lahir    x mati
                  
r cuba hidup lebih lama dari x = DANGLING!
Rust catch ini semasa compile, bukan runtime.
```

---

## Contoh Praktikal: Iterator Invalidation

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5];

    // ✔ OK — iterate immutably
    for x in &v {
        println!("{}", x);
    }

    // ❌ TIDAK BOLEH — ubah sambil iterate
    // for x in &v {
    //     if *x == 3 {
    //         v.push(10); // ERROR! v dipinjam oleh for loop
    //     }
    // }

    // ✔ Penyelesaian — kumpul dulu, kemudian ubah
    let tambah: Vec<i32> = v.iter()
        .filter(|&&x| x == 3)
        .map(|&x| x + 10)
        .collect();

    v.extend(tambah);
    println!("{:?}", v); // [1, 2, 3, 4, 5, 13]
}
```

---

# BAB 10: Mini Project — Inventory System 🏪

> Projek ini menggabungkan SEMUA konsep ownership.

```rust
use std::collections::HashMap;

// ─── Types ────────────────────────────────────────────────────

#[derive(Debug, Clone)]
struct Produk {
    id:       u32,
    nama:     String,
    harga:    f64,
    stok:     u32,
    kategori: String,
}

impl Produk {
    fn baru(id: u32, nama: &str, harga: f64, stok: u32, kategori: &str) -> Self {
        Produk {
            id,
            nama:     nama.into(),    // &str → String (ownership)
            harga,
            stok,
            kategori: kategori.into(),
        }
    }

    // & — baca sahaja, tidak ambil ownership
    fn ringkasan(&self) -> String {
        format!("[{}] {} — RM{:.2} (stok: {})", self.id, self.nama, self.harga, self.stok)
    }

    // Nilai (Copy) — dikembalikan tanpa perlu reference
    fn nilai_stok(&self) -> f64 {
        self.harga * self.stok as f64
    }
}

// ─── Inventory ────────────────────────────────────────────────

struct Inventori {
    produk: HashMap<u32, Produk>,
    id_seterusnya: u32,
}

impl Inventori {
    fn baru() -> Self {
        Inventori {
            produk: HashMap::new(),
            id_seterusnya: 1,
        }
    }

    // &mut self — ubah inventori
    fn tambah(&mut self, nama: &str, harga: f64, stok: u32, kategori: &str) -> u32 {
        let id = self.id_seterusnya;
        self.id_seterusnya += 1;
        // Produk::baru() buat String baru (owned)
        let p = Produk::baru(id, nama, harga, stok, kategori);
        self.produk.insert(id, p); // p MOVED ke dalam HashMap
        println!("  ✔ Tambah: {} (ID: {})", nama, id);
        id  // return id (Copy — u32)
    }

    // &self — baca sahaja, return reference
    fn cari(&self, id: u32) -> Option<&Produk> {
        self.produk.get(&id)
        // Return &Produk — borrow dari HashMap
        // Tidak pindah ownership!
    }

    // &mut self — ubah stok
    fn kemaskini_stok(&mut self, id: u32, stok_baru: u32) -> bool {
        match self.produk.get_mut(&id) {  // get_mut = &mut Produk
            Some(p) => {
                println!("  ✔ Stok {} {} → {}", p.nama, p.stok, stok_baru);
                p.stok = stok_baru; // ubah melalui &mut
                true
            }
            None => {
                println!("  ✘ ID {} tidak dijumpai", id);
                false
            }
        }
    }

    // &mut self — consume (move) produk keluar dari HashMap
    fn buang(&mut self, id: u32) -> Option<Produk> {
        match self.produk.remove(&id) {
            Some(p) => {
                println!("  ✔ Buang: {} (ID: {})", p.nama, p.id);
                Some(p) // p MOVED keluar — caller dapat ownership
            }
            None => {
                println!("  ✘ ID {} tidak dijumpai", id);
                None
            }
        }
    }

    // &self — baca semua, return Vec<&Produk> (borrow)
    fn senarai_mengikut_kategori(&self, kategori: &str) -> Vec<&Produk> {
        self.produk.values()
            .filter(|p| p.kategori == kategori)
            .collect()
        // Return Vec of borrows — data masih dalam self.produk
        // Caller tidak perlu free/clone data ini
    }

    // &self — kira nilai (return Copy type — f64)
    fn nilai_total(&self) -> f64 {
        self.produk.values()
            .map(|p| p.nilai_stok())  // p adalah &Produk
            .sum()
    }

    // &self — clone untuk dapatkan owned copy semua produk
    fn eksport_semua(&self) -> Vec<Produk> {
        self.produk.values()
            .cloned()  // clone setiap &Produk → Produk (owned)
            .collect()
        // Ini mahal! Guna &self dan &Produk kalau boleh
    }

    // &self — papar laporan, borrow sahaja
    fn laporan(&self) {
        println!("\n{'═'*55}");
        println!("{:^55}", "LAPORAN INVENTORI");
        println!("{'═'*55}");

        // Kumpul mengikut kategori (borrow)
        let mut ikut_kat: HashMap<&str, Vec<&Produk>> = HashMap::new();
        for p in self.produk.values() {
            ikut_kat.entry(p.kategori.as_str())
                    .or_default()
                    .push(p);
        }

        let mut kategori: Vec<&&str> = ikut_kat.keys().collect();
        kategori.sort();

        for kat in kategori {
            let senarai = &ikut_kat[kat];
            println!("\n  📁 {} ({} item):", kat, senarai.len());
            let mut s: Vec<&&Produk> = senarai.iter().collect();
            s.sort_by(|a, b| a.nama.cmp(&b.nama));
            for p in s {
                println!("    {}", p.ringkasan());
            }
        }

        println!("\n{'─'*55}");
        println!("  Jumlah produk:    {}", self.produk.len());
        println!("  Nilai inventori:  RM{:.2}", self.nilai_total());
        println!("{'═'*55}");
    }
}

// ─── Transaksi ─────────────────────────────────────────────────

struct Transaksi<'a> {
    inventori: &'a mut Inventori,  // mutable borrow!
    rekod: Vec<String>,
}

impl<'a> Transaksi<'a> {
    fn baru(inv: &'a mut Inventori) -> Self {
        Transaksi { inventori: inv, rekod: Vec::new() }
    }

    // &mut self kerana ubah rekod + inventori
    fn jual(&mut self, id: u32, kuantiti: u32) -> bool {
        match self.inventori.produk.get_mut(&id) {
            Some(p) if p.stok >= kuantiti => {
                p.stok -= kuantiti;
                let mesej = format!("JUAL: {} x{} @ RM{:.2}",
                    p.nama, kuantiti, p.harga);
                self.rekod.push(mesej.clone()); // clone String untuk simpan
                println!("  ✔ {}", mesej);
                true
            }
            Some(p) => {
                println!("  ✘ Stok {} tidak cukup (ada: {}, minta: {})",
                    p.nama, p.stok, kuantiti);
                false
            }
            None => {
                println!("  ✘ Produk ID {} tidak dijumpai", id);
                false
            }
        }
    }

    // & borrow rekod — return &Vec
    fn rekod_transaksi(&self) -> &Vec<String> {
        &self.rekod  // borrow Vec, tidak move!
    }
}

// ─── Main ──────────────────────────────────────────────────────

fn main() {
    // Buat inventori (owned)
    let mut inv = Inventori::baru();

    println!("=== Tambah Produk ===");
    let id1 = inv.tambah("Beras 5kg",    12.90,  50, "Makanan");
    let id2 = inv.tambah("Minyak 1L",     6.50,  30, "Makanan");
    let id3 = inv.tambah("Sabun Mandi",   2.50,  100,"Kebersihan");
    let id4 = inv.tambah("Detergen 1kg",  8.90,  40, "Kebersihan");
    let id5 = inv.tambah("Pensil 2B",     1.20,  200,"Alat Tulis");

    println!("\n=== Cari Produk (Borrow) ===");
    // cari() return &Produk — borrow dari inv
    match inv.cari(id1) {
        Some(p) => println!("  Jumpa: {}", p.ringkasan()),
        None    => println!("  Tidak jumpa"),
    }
    // inv masih valid — kita hanya borrow

    println!("\n=== Kemaskini Stok (&mut) ===");
    inv.kemaskini_stok(id3, 150); // &mut borrow

    println!("\n=== Buang Produk (Move keluar) ===");
    if let Some(produk_lama) = inv.buang(id5) {
        // produk_lama: Produk (owned!)
        println!("  Produk dibuang: {}", produk_lama.nama);
        println!("  Nilai stok dibuang: RM{:.2}", produk_lama.nilai_stok());
        // produk_lama di-drop di sini bila keluar scope
    }

    println!("\n=== Senarai Mengikut Kategori (Borrow) ===");
    {
        // senarai adalah Vec<&Produk> — semua borrow dari inv
        let senarai = inv.senarai_mengikut_kategori("Makanan");
        for p in &senarai {
            println!("  {}", p.ringkasan());
        }
        // senarai di-drop di sini — borrow selesai
        // data dalam inv tidak terjejas!
    }

    println!("\n=== Transaksi Jualan ===");
    {
        // Transaksi ambil &mut inv — mutable borrow
        let mut tx = Transaksi::baru(&mut inv);
        tx.jual(id1, 5);   // jual beras
        tx.jual(id2, 10);  // jual minyak
        tx.jual(id2, 30);  // cuba jual lebih dari stok

        println!("\n  Rekod transaksi:");
        for r in tx.rekod_transaksi() { // borrow rekod
            println!("    {}", r);
        }
        // tx di-drop di sini → borrow selesai
    }
    // inv boleh digunakan semula selepas tx di-drop!

    println!("\n=== Eksport (Clone) ===");
    // eksport_semua() clone semua Produk — mahal tapi perlu kadangkala
    let senarai_eksport = inv.eksport_semua();
    println!("  Eksport {} produk", senarai_eksport.len());
    // senarai_eksport adalah Vec<Produk> yang bebas — tidak berkaitan dengan inv

    // Laporan akhir
    inv.laporan();
}
```

---

# 📋 Rujukan Pantas — Ownership Cheat Sheet

## Tiga Peraturan

```
1. Setiap nilai ada tepat SATU owner
2. Hanya boleh ada SATU owner pada satu masa (move = tukar owner)
3. Apabila owner keluar scope → nilai di-DROP (auto free)
```

## Move vs Copy

```
MOVE (non-Copy types — heap data):
  String, Vec<T>, Box<T>, HashMap, dll
  let a = String::from("x");
  let b = a;      // a MOVED ke b, a tidak valid
  fungsi(a);      // a MOVED ke fungsi, a tidak valid

COPY (Copy types — stack-only data):
  i32, f64, bool, char, &T, [T;N] (jika T Copy), tuple (jika semua Copy)
  let x = 5;
  let y = x;      // COPY! x masih valid
  fungsi(x);      // COPY ke fungsi, x masih valid
```

## Borrowing

```
& (shared/immutable borrow):
  let r = &s;           // baca sahaja
  fn f(s: &String)      // terima reference
  Boleh ada banyak & serentak!

&mut (mutable borrow):
  let r = &mut s;       // baca dan tulis
  fn f(s: &mut String)  // terima mutable reference
  Hanya SATU &mut pada satu masa!
  Tiada & lain boleh wujud serentak!
```

## Peraturan Borrow

```
BOLEH:    banyak &T serentak
BOLEH:    satu &mut T (tiada & lain)
TIDAK:    &T + &mut T serentak
TIDAK:    dua &mut T serentak
```

## Dangling Reference

```
Rust prevent: reference tidak boleh outlive data yang dipinjam.
Compiler check lifetime pada compile time — tiada runtime error!
```

## Clone

```
.clone()  → deep copy, heap allocation baru
          → guna hanya bila perlu (mahal!)
          → lebih baik guna & kalau cukup
```

## Cara Fikir

```
"Siapa yang bertanggungjawab untuk FREE data ini?"
  → Owner!

"Nak guna data tapi tidak nak ambil tanggungjawab?"
  → Borrow dengan &

"Nak ubah data orang lain?"
  → Mutable borrow dengan &mut (minta izin!)

"Nak ada salinan sendiri yang bebas?"
  → Clone (mahal!) atau guna Copy type
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu dan perhatikan bagaimana ownership mengalir:

1. **Stack Calculator** — `Vec<f64>` sebagai stack, push/pop dengan ownership betul
2. **Text Buffer** — struct dengan `String` dalaman, method borrow dan mutable borrow
3. **Cache** — `HashMap<String, Vec<u8>>` dengan fungsi get (&) dan set (&mut)
4. **Graph** — `HashMap<String, Vec<String>>` untuk adjacency list, BFS dengan &
5. **Event System** — `Vec<Box<dyn Fn()>>` untuk event listeners, ownership callbacks

---

*Ownership in Rust — inti pati Rust. Faham ini = faham segalanya.* 🦀
