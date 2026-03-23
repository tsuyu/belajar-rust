# 📦 Move vs Copy dalam Rust — Panduan Mendalam

> "Kenapa i32 boleh assign dua kali tapi String tidak?"
> Faham move semantics, copy trait, stack vs heap,
> dan implikasi praktikal untuk kod anda.

---

## Masalah Yang Selalu Mengelirukan

```rust
// Ini berfungsi:
let x: i32 = 5;
let y = x;
println!("{}", x); // OK! x masih boleh digunakan

// Ini TIDAK berfungsi:
let s1 = String::from("hello");
let s2 = s1;
println!("{}", s1); // ERROR! s1 sudah "dipindah"
```

```
Kenapa berbeza? Jawapannya: Stack vs Heap + Copy vs Move
```

---

## Peta Pembelajaran

```
Bab 1  → Stack vs Heap — Asas Ingatan
Bab 2  → Copy — "Just Bytes, Just Copy"
Bab 3  → Move — Pemindahan Ownership
Bab 4  → Apa yang Berlaku di Sebalik Tabir
Bab 5  → Jenis yang Copy vs Jenis yang Move
Bab 6  → Copy Trait — Implement Sendiri
Bab 7  → Move dalam Fungsi & Return
Bab 8  → Move dalam Struct & Enum
Bab 9  → Partial Move
Bab 10 → Mini Project: Fahami Setiap Kes
```

---

# BAB 1: Stack vs Heap — Asas Ingatan 🧠

## Dua Tempat Simpan Data

```
STACK (tindanan)                 HEAP (timbunan)
────────────────                 ────────────────
Saiz TETAP semasa compile        Saiz BOLEH BERUBAH semasa runtime
Diurus AUTOMATIK                 Diurus melalui allocator
LAJU (push/pop sahaja)           LEBIH PERLAHAN (perlu allocate/free)
Data hidup dalam SCOPE           Data hidup sampai DIBEBASKAN
                                 (atau owner drop)

Contoh data Stack:               Contoh data Heap:
  i32, f64, bool, char           String (kandungan)
  [i32; 5] (array saiz tetap)    Vec<T> (elemen)
  (i32, bool) (tuple simple)     HashMap, BTreeMap
  struct { x: i32, y: i32 }      Box<T>
  pointer/reference (&T)         sebarang data dynamic

Visualisasi:

  Stack:                  Heap:
  ┌─────────────┐         ┌──────────────────┐
  │ x: i32 = 5 │         │ "hello world..." │ ← kandungan String
  │ y: i32 = 5 │         │ [1, 2, 3, 4, 5]  │ ← kandungan Vec
  │ ptr ────────┼────────→│ ...              │
  │ len: 5      │         └──────────────────┘
  │ cap: 10     │
  └─────────────┘
  ↑ String metadata di stack,
    kandungan di heap
```

---

## String di Memori — Secara Terperinci

```rust
let s = String::from("hello");

// Di stack, s mengandungi 3 bahagian:
// ┌─────────────────────────────────┐
// │ ptr  → alamat memori heap       │ (8 bytes pada 64-bit)
// │ len  = 5 (panjang kandungan)    │ (8 bytes)
// │ cap  = 5 (kapasiti buffer)      │ (8 bytes)
// └─────────────────────────────────┘
//
// Di heap:
// ┌───────────────────┐
// │ h | e | l | l | o │ ← 5 bytes kandungan sebenar
// └───────────────────┘
//
// Total: 24 bytes stack + 5 bytes heap

// i32 = HANYA stack:
// ┌─────────────────────┐
// │ x = 5               │ ← 4 bytes sahaja, semuanya di stack
// └─────────────────────┘
```

---

## 🧠 Brain Teaser #1

Berapa banyak memori yang digunakan, dan di mana?

```rust
let a: i32 = 42;
let b: [i32; 3] = [1, 2, 3];
let c: &str = "hello";
let d: String = String::from("world");
let e: Vec<i32> = vec![10, 20, 30];
let f: Box<i32> = Box::new(99);
```

<details>
<summary>👀 Jawapan</summary>

```
a: i32          → 4 bytes STACK sahaja
b: [i32; 3]     → 12 bytes STACK sahaja (3 × 4 bytes, saiz tetap)
c: &str "hello" → Stack: 16 bytes (ptr + len)
                  Readonly memory: "hello" (dalam binary, bukan heap!)
d: String       → Stack: 24 bytes (ptr + len + cap)
                  Heap:  5 bytes ("world" kandungan)
e: Vec<i32>     → Stack: 24 bytes (ptr + len + cap)
                  Heap:  12 bytes (3 × 4 bytes elemen)
f: Box<i32>     → Stack: 8 bytes (pointer)
                  Heap:  4 bytes (nilai 99)

Jumlah Stack: 4 + 12 + 16 + 24 + 24 + 8 = 88 bytes
Jumlah Heap:  5 + 12 + 4 = 21 bytes
Readonly:     "hello" (dalam binary segment)

Penting: &str tidak allocate heap! Ia hanya pointer ke
         data yang sudah ada (string literal dalam binary).
```
</details>

---

# BAB 2: Copy — "Just Bytes, Just Copy" 📋

## Apa itu Copy?

```rust
// Jenis Copy = boleh diduplikasi dengan hanya menyalin bytes
// TIADA side effect, TIADA destruktor khas diperlukan
// Data SEPENUHNYA di stack (atau saiz tetap)

let x: i32 = 5;
let y = x;     // COPY — salin 4 bytes, dua variabel bebas
               //         x dan y KEDUA-DUANYA valid
println!("{} {}", x, y); // OK! 5 5

// Bayangkan:
// Sebelum:  Stack: [x=5]
// let y = x → salin bytes
// Selepas:  Stack: [x=5, y=5]
// DUAN-DUANYA wujud, bebas, tiada kaitan

// ── Semua jenis yang Copy ─────────────────────────────────────
let a: i8   = 1;    let _a2 = a;    println!("{}", a); // ✔
let b: i32  = 2;    let _b2 = b;    println!("{}", b); // ✔
let c: u64  = 3;    let _c2 = c;    println!("{}", c); // ✔
let d: f32  = 4.0;  let _d2 = d;    println!("{}", d); // ✔
let e: f64  = 5.0;  let _e2 = e;    println!("{}", e); // ✔
let f: bool = true; let _f2 = f;    println!("{}", f); // ✔
let g: char = 'A';  let _g2 = g;    println!("{}", g); // ✔

// Reference = Copy (copy pointer, bukan data)
let s = String::from("hello");
let h: &str   = &s;   let _h2 = h;   println!("{}", h); // ✔
let i: &i32   = &42;  let _i2 = i;   println!("{}", i); // ✔

// Array dan tuple — Copy JIKA semua elemen adalah Copy
let j: [i32; 3] = [1, 2, 3]; let _j2 = j; println!("{:?}", j); // ✔
let k: (i32, bool) = (1, true); let _k2 = k; println!("{:?}", k); // ✔

// Tuple dengan non-Copy = BUKAN Copy
// let bad: (i32, String) = (1, "hello".into());
// let _bad2 = bad;         // MOVE!
// println!("{:?}", bad);   // ERROR!
```

---

# BAB 3: Move — Pemindahan Ownership 🚚

## Move = Pindah Ownership

```rust
let s1 = String::from("hello");
// s1 adalah OWNER data "hello" di heap

let s2 = s1;
// MOVE — ownership berpindah dari s1 ke s2
// s1 kini TIDAK VALID

// println!("{}", s1); // ERROR: value borrowed here after move

// Kenapa Rust buat macam ini?
// Kerana String ada data di heap!
// Kalau Rust biarkan dua owner:
//   s1 dan s2 kedua-duanya fikir mereka perlu FREE data
//   → DOUBLE FREE → undefined behavior!

// Visualisasi:
//
// Sebelum let s2 = s1:
// Stack: [s1: {ptr→heap, len:5, cap:5}]
// Heap:  [h|e|l|l|o]
//
// Selepas let s2 = s1:
// Stack: [s1: INVALID, s2: {ptr→heap, len:5, cap:5}]
// Heap:  [h|e|l|l|o]
//
// s1 metadata di-"nullify" — Rust tandakan tidak valid
// Hanya SATU owner pada satu masa!

// ── Move berlaku dalam pelbagai situasi ──────────────────────

struct Data(String);

let d1 = Data("contoh".into());

// 1. Assignment
let d2 = d1; // MOVE
// d1 tidak valid

// 2. Panggil fungsi
fn guna(d: Data) { println!("{}", d.0); }
let d3 = Data("lagi".into());
guna(d3); // MOVE ke parameter
// d3 tidak valid

// 3. Return dari fungsi
fn buat() -> Data { Data("baru".into()) } // MOVE keluar
let d4 = buat(); // OK — d4 adalah owner

// 4. Dalam Vec
let d5 = Data("dalam vec".into());
let v = vec![d5]; // MOVE ke Vec
// d5 tidak valid
```

---

# BAB 4: Apa yang Berlaku di Sebalik Tabir 🔍

## Move bukan "Pindah Bytes"

```rust
// SALAH FAHAM biasa: "Move = salin bytes, kemudian padam asal"
// BETUL: Move = SEMANTIK Rust, bukan operasi berbeza di hardware!

// Pada peringkat assembly:
// Copy DAN Move KEDUA-DUANYA lakukan operasi yang SAMA:
//   → salin bytes dari lokasi A ke lokasi B

// Perbezaannya adalah pada PERINGKAT SEMANTIK (bahasa):
//   Copy: kedua-dua lokasi BOLEH DIGUNAKAN
//   Move: hanya lokasi BARU boleh digunakan

// Rust enforce ini semasa COMPILE TIME, bukan runtime!

// Contoh: Stack allocation
struct Mudah { x: i32, y: f64 }
// Mudah: 4 + 4 (padding) + 8 = 16 bytes di stack

let a = Mudah { x: 1, y: 2.0 };
let b = a; // MOVE — 16 bytes disalin, a tidak valid

// Pada assembly, `a` dan `b` adalah lokasi stack berbeza.
// "move" hanya bererti compiler sekat akses ke `a`.
// Tidak ada perbezaan runtime antara copy dan move kecuali
// untuk jenis yang perlu destructor (Drop)!

// Perbezaan SEBENAR yang penting:
// Copy type: TIADA Drop implementation
// Move type: ADA Drop implementation (perlu cleanup)
//
// Jika String boleh Copy:
//   let s1 = String::from("hello");
//   let s2 = s1; // "copy"
//   // Bila s1 drop → free heap
//   // Bila s2 drop → free HEAP YANG SAMA → double free → CRASH!
//
// Sebab itu String TIDAK boleh Copy!
// Drop dan Copy adalah MUTEX (tidak boleh bersama)
```

---

## Clone vs Move vs Copy

```rust
let s = String::from("hello");

// MOVE:  pindah ownership, asal tidak valid
let s_moved = s;
// println!("{}", s); // ERROR

// CLONE: buat salinan sepenuhnya (new heap allocation)
let s2 = String::from("hello");
let s_cloned = s2.clone();
println!("{}", s2);      // OK — masih valid
println!("{}", s_cloned); // OK — salinan bebas

// COPY:  (hanya untuk Copy types)
let n: i32 = 42;
let n_copy = n;
println!("{} {}", n, n_copy); // OK — kedua-dua valid

// Kos:
//   Move:  O(1) — cuma salin pointer/metadata, tiada heap allocation
//   Copy:  O(n) untuk stack data (biasanya kecil)
//   Clone: O(n) — perlu allocate heap baru + salin kandungan
//   (Clone biasanya lebih MAHAL dari Move atau Copy)
```

---

# BAB 5: Jenis Copy vs Jenis Move 📊

## Carta Lengkap

```rust
// ════════════════════════════════════════════════════════
// JENIS YANG COPY (boleh guna selepas assign)
// ════════════════════════════════════════════════════════

// Primitif semua Copy:
i8, i16, i32, i64, i128, isize  // integer signed
u8, u16, u32, u64, u128, usize  // integer unsigned
f32, f64                         // float
bool                             // boolean
char                             // unicode codepoint

// Reference = Copy (salin pointer, bukan data)
&T                               // immutable reference
// NOTA: &mut T BUKAN Copy! Hanya satu mutable ref boleh wujud

// Array dengan elemen Copy:
[i32; 5]                         // Copy
[bool; 100]                      // Copy

// Tuple dengan semua elemen Copy:
(i32, f64)                       // Copy
(bool, u8, char)                 // Copy

// Tuple dengan mana-mana non-Copy:
// (i32, String) → BUKAN Copy (kerana String)

// ════════════════════════════════════════════════════════
// JENIS YANG MOVE (tidak boleh guna selepas assign)
// ════════════════════════════════════════════════════════

// String types (ada heap allocation):
String                           // Move
Box<T>                           // Move (owns heap)
Vec<T>                           // Move
HashMap<K, V>                    // Move
HashSet<T>                       // Move
BTreeMap<K, V>                   // Move

// Smart pointers (bukan Rc/Arc — ada reference counting):
Box<T>                           // Move
// (Rc<T> dan Arc<T> adalah Clone tapi BUKAN Copy)

// Closure yang capture environment:
// kalau closure capture non-Copy variable → Move closure

// Struct yang ada non-Copy field:
// struct Pekerja { nama: String } → MOVE (kerana String)

// Enum yang ada non-Copy variant:
// enum Val { Num(i32), Text(String) } → MOVE

// ════════════════════════════════════════════════════════
// BOLEH JADI SAMA-SAMA (bergantung pada implementasi):
// ════════════════════════════════════════════════════════

// Struct dengan semua Copy fields → BOLEH jadi Copy (dengan derive)
#[derive(Copy, Clone, Debug)]
struct Titik { x: f64, y: f64 }
// Titik adalah Copy kerana f64 adalah Copy

// Struct dengan non-Copy field → TIDAK BOLEH Copy
// #[derive(Copy, Clone)]
// struct TidakBoleh { x: i32, nama: String } // ERROR!
```

---

# BAB 6: Copy Trait — Implement Sendiri 🛠️

## Bila & Cara Implement Copy

```rust
// #[derive(Copy, Clone)] — cara paling mudah
// Syarat: SEMUA field mestilah Copy

#[derive(Debug, Clone, Copy)]
struct Koordinat {
    lat: f64,  // f64: Copy ✔
    lon: f64,  // f64: Copy ✔
}

#[derive(Debug, Clone, Copy)]
struct Warna {
    r: u8,  // u8: Copy ✔
    g: u8,  // u8: Copy ✔
    b: u8,  // u8: Copy ✔
    a: u8,  // u8: Copy ✔
}

#[derive(Debug, Clone, Copy, PartialEq)]
enum Arah {
    Utara,
    Selatan,
    Timur,
    Barat,
}

fn main() {
    let k1 = Koordinat { lat: 6.05, lon: 102.25 };
    let k2 = k1;            // COPY!
    println!("{:?}", k1);   // masih valid
    println!("{:?}", k2);   // salinan bebas

    let arah = Arah::Utara;
    let arah2 = arah;        // COPY!
    println!("{:?}", arah);  // masih valid

    // Guna kes: koordinat untuk GPS tracking
    fn log_koordinat(k: Koordinat) {
        println!("({}, {})", k.lat, k.lon);
    }

    log_koordinat(k1); // Copy ke parameter
    log_koordinat(k1); // Masih boleh guna k1!
}

// ── Bila TIDAK patut implement Copy ──────────────────────────

// 1. Ada heap allocation → tidak boleh
// #[derive(Copy)] // ERROR!
// struct Ada Heap { data: Vec<u8> }

// 2. Resource yang perlu cleanup → tidak patut
// (walaupun secara teknikal mungkin dengan unsafe)
// Fail handle, DB connection, Lock guard → JANGAN Copy!

// 3. Ada semantik "unique ownership" yang penting
// Kalau type bermaksud "hanya satu owner" → jangan Copy
// (mutex guard, file handle, dll)
```

---

## 🧠 Brain Teaser #2

Kenapa kod ini compile dengan `#[derive(Copy)]` tapi tidak dengan variasi lain?

```rust
// A — Compile?
#[derive(Copy, Clone)]
struct A {
    x: i32,
    y: &'static str,
}

// B — Compile?
#[derive(Copy, Clone)]
struct B {
    x: i32,
    y: String,
}

// C — Compile?
#[derive(Copy, Clone)]
struct C {
    x: i32,
    y: &str,   // tiada lifetime
}
```

<details>
<summary>👀 Jawapan</summary>

```
A: ✔ COMPILE
   &'static str adalah Copy (reference adalah Copy)
   'static bermakna data hidup selama program berjalan
   i32 adalah Copy
   → Semua field Copy → A boleh Copy

B: ✗ TIDAK COMPILE
   String BUKAN Copy (ada heap allocation, ada Drop)
   Copy dan Drop adalah mutually exclusive!
   Error: "the trait `Copy` may not be implemented for this type;
           field `y` does not implement `Copy`"

C: ✗ TIDAK COMPILE (syntax error)
   &str tanpa lifetime bukan jenis yang lengkap dalam struct.
   Perlu: &'a str dengan lifetime parameter, atau
          &'static str untuk data static.
   struct C<'a> { x: i32, y: &'a str }
   → Ini boleh Copy jika lifetime ditentukan!

   #[derive(Copy, Clone)]
   struct C<'a> {
       x: i32,
       y: &'a str,  // reference = Copy
   }
   → Ini ✔ COMPILE

PELAJARAN:
   - Reference (&T) = Copy
   - String (owned) = Move
   - Lifetime ('static atau 'a) diperlukan dalam struct
```
</details>

---

# BAB 7: Move dalam Fungsi & Return 🔄

## Ownership Flow

```rust
// ── Move ke fungsi ────────────────────────────────────────────

fn ambil_ownership(s: String) -> usize {
    s.len() // s di-drop bila fungsi habis
}

fn pinjam_sahaja(s: &str) -> usize {
    s.len() // tidak move, hanya pinjam
}

fn main() {
    let s = String::from("hello world");

    // ❌ Move ke fungsi — s tidak valid selepas
    let panjang = ambil_ownership(s);
    // println!("{}", s); // ERROR!

    // ✔ Pinjam — s masih valid
    let s2 = String::from("hello world");
    let panjang2 = pinjam_sahaja(&s2);
    println!("{}", s2); // OK!

    // ── Return nilai dari fungsi ──────────────────────────────

    fn buat_string() -> String {
        let s = String::from("dibuat dalam fungsi");
        s // MOVE keluar — ownership pindah ke caller
    }

    let s3 = buat_string(); // s3 adalah owner
    println!("{}", s3);
    // s3 di-drop bila keluar scope

    // ── Ambil dan kembalikan ──────────────────────────────────

    fn proses_dan_kembalikan(s: String) -> (String, usize) {
        let panjang = s.len();
        (s, panjang) // kembalikan ownership
    }

    let s4 = String::from("data");
    let (s4, panjang4) = proses_dan_kembalikan(s4);
    println!("{}: {} chars", s4, panjang4); // OK!
}
```

---

## Move dalam Closure

```rust
fn main() {
    let nama = String::from("Ali");

    // ── Closure yang BORROW ───────────────────────────────────
    let salam = || println!("Halo, {}!", nama);
    salam(); // OK
    salam(); // OK
    println!("{}", nama); // OK — nama masih valid

    // ── Closure yang MOVE (move keyword) ─────────────────────
    let nama2 = String::from("Siti");

    let salam_pindah = move || println!("Halo, {}!", nama2);
    // nama2 dipindah KE DALAM closure!

    // println!("{}", nama2); // ERROR — nama2 sudah move ke closure

    salam_pindah(); // OK
    salam_pindah(); // OK (closure sendiri masih valid)

    // ── Kenapa perlu move closure? ────────────────────────────
    let nama3 = String::from("Amin");

    // Thread perlu 'static lifetime — perlu move!
    std::thread::spawn(move || {
        println!("Thread: {}", nama3);
        // nama3 dipindah ke thread
    }).join().unwrap();

    // println!("{}", nama3); // ERROR — sudah move ke thread

    // ── FnOnce — closure yang consume capture ─────────────────
    let data = vec![1, 2, 3];

    // FnOnce: boleh panggil sekali sahaja (data dipindah dalam closure)
    let proses: Box<dyn FnOnce() -> Vec<i32>> = Box::new(move || {
        data // MOVE data keluar dari closure
    });

    let hasil = proses(); // panggil sekali
    // proses();           // ERROR! FnOnce sudah consumed
    println!("{:?}", hasil);
}
```

---

# BAB 8: Move dalam Struct & Enum 🏗️

## Struct Update Syntax & Move

```rust
#[derive(Debug)]
struct Pekerja {
    id:       u32,
    nama:     String,
    bahagian: String,
    gaji:     f64,
}

fn main() {
    let p1 = Pekerja {
        id:       1,
        nama:     "Ali Ahmad".into(),
        bahagian: "ICT".into(),
        gaji:     4500.0,
    };

    // ── Struct update syntax ──────────────────────────────────
    let p2 = Pekerja {
        id:   2,
        nama: "Siti Hawa".into(), // override nama
        ..p1                       // baki dari p1
    };

    // p1.bahagian dan p1.gaji sudah MOVE ke p2!
    // Tapi p1.id masih valid (u32: Copy)
    // p1.nama belum move (override dengan nilai baru)

    // println!("{}", p1.bahagian); // ERROR! bahagian sudah move
    println!("{}", p1.id);      // OK! Copy
    // println!("{:?}", p1);    // ERROR! partial move

    println!("{:?}", p2);

    // ── Partial struct move ───────────────────────────────────
    let p3 = Pekerja {
        id:       3,
        nama:     "Amin".into(),
        bahagian: "HR".into(),
        gaji:     3800.0,
    };

    let nama = p3.nama; // MOVE field nama sahaja
    println!("{}", nama);

    // p3.nama sudah move, tapi field lain masih ada!
    println!("{}", p3.id);      // OK
    println!("{}", p3.bahagian); // OK
    println!("{}", p3.gaji);    // OK
    // println!("{:?}", p3);    // ERROR! partial move — tidak boleh guna keseluruhan
}
```

---

## Move dalam Enum

```rust
#[derive(Debug)]
enum Respons {
    Berjaya(String),
    Gagal { kod: u32, sebab: String },
    Kosong,
}

fn main() {
    let r = Respons::Berjaya("Data diterima".into());

    // ── Match menggunakan move ────────────────────────────────
    match r {
        Respons::Berjaya(data) => {
            // data di-MOVE keluar dari enum!
            println!("Berjaya: {}", data);
        }
        Respons::Gagal { kod, sebab } => {
            // kod (u32) = Copy, sebab (String) = Move
            println!("Gagal {}: {}", kod, sebab);
        }
        Respons::Kosong => println!("Kosong"),
    }
    // r tidak valid selepas match (kerana data dalam move)

    // ── Match dengan reference ────────────────────────────────
    let r2 = Respons::Gagal { kod: 404, sebab: "Tiada".into() };

    match &r2 {
        Respons::Berjaya(data) => println!("{}", data), // data: &String
        Respons::Gagal { kod, sebab } => {
            // kod: &u32, sebab: &String — PINJAM sahaja
            println!("Gagal {}: {}", kod, sebab);
        }
        Respons::Kosong => {}
    }

    println!("{:?}", r2); // OK! r2 masih valid
}
```

---

# BAB 9: Partial Move & Workarounds 🧩

## Partial Move

```rust
#[derive(Debug)]
struct Rekod {
    id:   u32,    // Copy
    nama: String, // Move
    data: Vec<u8>, // Move
}

fn main() {
    let r = Rekod {
        id:   1,
        nama: "Ali".into(),
        data: vec![1, 2, 3],
    };

    // Partial move — ambil satu field
    let nama = r.nama; // MOVE nama sahaja

    // Copy field masih boleh akses
    println!("ID: {}", r.id);    // OK (Copy)

    // Move field tidak boleh akses lagi
    // println!("{}", r.nama);   // ERROR

    // Keseluruhan struct tidak boleh guna
    // println!("{:?}", r);      // ERROR (partial move)

    // ── Cara elak partial move ────────────────────────────────

    // Cara 1: Clone dahulu
    let r2 = Rekod { id: 2, nama: "Siti".into(), data: vec![4, 5] };
    let nama2 = r2.nama.clone(); // clone, bukan move
    println!("{:?}", r2);         // OK! r2 masih lengkap

    // Cara 2: Guna reference
    let r3 = Rekod { id: 3, nama: "Amin".into(), data: vec![] };
    let nama3: &str = &r3.nama;  // pinjam, bukan move
    println!("{}", nama3);
    println!("{:?}", r3);         // OK! r3 masih lengkap

    // Cara 3: std::mem::take — ambil field, ganti dengan Default
    let mut r4 = Rekod { id: 4, nama: "Zara".into(), data: vec![1] };
    let nama4 = std::mem::take(&mut r4.nama); // ambil, ganti dengan ""
    println!("Ambil: {}", nama4); // "Zara"
    println!("r4.nama selepas take: '{}'", r4.nama); // ""

    // Cara 4: std::mem::replace — ambil dan ganti dengan nilai lain
    let mut r5 = Rekod { id: 5, nama: "Hadi".into(), data: vec![] };
    let nama_lama = std::mem::replace(&mut r5.nama, "Nama Baru".into());
    println!("Lama: {}", nama_lama); // "Hadi"
    println!("Baru: {}", r5.nama);   // "Nama Baru"

    // Cara 5: Destructure semua sekaligus
    let r6 = Rekod { id: 6, nama: "Farid".into(), data: vec![1, 2] };
    let Rekod { id, nama, data } = r6; // destructure semua
    println!("id={}, nama={}, data={:?}", id, nama, data);
    // r6 tidak valid selepas destructure
}
```

---

# BAB 10: Mini Project — Fahami Setiap Kes 🏗️

```rust
// Demonstrasi komprehensif semua kes move vs copy

use std::collections::HashMap;

// ── Struct Copy ────────────────────────────────────────────────
#[derive(Debug, Clone, Copy, PartialEq)]
struct GPS {
    lat: f64,
    lon: f64,
}

impl GPS {
    fn baru(lat: f64, lon: f64) -> Self {
        GPS { lat, lon }
    }

    fn jarak_dari(&self, lain: GPS) -> f64 {
        // lain: Copy — guna tanpa move
        let dlat = (lain.lat - self.lat).to_radians();
        let dlon = (lain.lon - self.lon).to_radians();
        let a = (dlat/2.0).sin().powi(2)
              + self.lat.to_radians().cos()
              * lain.lat.to_radians().cos()
              * (dlon/2.0).sin().powi(2);
        6371.0 * 2.0 * a.sqrt().atan2((1.0-a).sqrt())
    }
}

// ── Struct Move ────────────────────────────────────────────────
#[derive(Debug, Clone)]
struct Pekerja {
    id:        u32,
    nama:      String,
    lokasi:    GPS,      // Copy field
    rekod:     Vec<String>, // Move field
}

// ── Sistem yang demonstrasi ────────────────────────────────────
struct SistemKehadiran {
    pekerja:   HashMap<u32, Pekerja>,
    pejabat:   GPS,
    rekod_log: Vec<String>,
}

impl SistemKehadiran {
    fn baru() -> Self {
        SistemKehadiran {
            pekerja:   HashMap::new(),
            pejabat:   GPS::baru(6.0538, 102.2503), // KADA Kemubu
            rekod_log: Vec::new(),
        }
    }

    fn daftar_pekerja(&mut self, pekerja: Pekerja) {
        // pekerja MOVE ke HashMap
        let id = pekerja.id;
        self.pekerja.insert(id, pekerja);
        // pekerja tidak valid selepas ini
    }

    // Return &Pekerja — bukan move, pinjam
    fn cari(&self, id: u32) -> Option<&Pekerja> {
        self.pekerja.get(&id)
    }

    fn daftar_masuk(&mut self, id: u32, lokasi_semasa: GPS) {
        // GPS adalah Copy — boleh hantar terus
        let pejabat = self.pejabat; // COPY GPS

        if let Some(pekerja) = self.pekerja.get_mut(&id) {
            let jarak = lokasi_semasa.jarak_dari(pejabat); // Copy GPS ke fungsi

            let entri = if jarak <= 0.1 {
                format!("[MASUK] {} — dalam kawasan ({:.3}km)", pekerja.nama, jarak)
            } else {
                format!("[MASUK] {} — luar kawasan ({:.3}km)!", pekerja.nama, jarak)
            };

            pekerja.rekod.push(entri.clone()); // push ke Vec milik pekerja
            self.rekod_log.push(entri);          // push ke log global
        }
    }

    fn cetak_laporan(&self) {
        println!("\n=== Laporan Kehadiran ===");
        // iterate — pinjam semua, tiada move
        for (id, pekerja) in &self.pekerja {
            println!("\nPekerja #{}: {}", id, pekerja.nama);
            for entri in &pekerja.rekod {
                println!("  {}", entri);
            }
        }
    }

    fn eksport_log(self) -> Vec<String> {
        // self MOVE — consume SistemKehadiran
        // Kembalikan ownership log
        self.rekod_log
        // self (dan semua data dalam HashMap) di-drop di sini
    }
}

fn main() {
    println!("=== Move vs Copy dalam Tindakan ===\n");

    // ── Demo 1: GPS adalah Copy ───────────────────────────────
    println!("--- GPS (Copy Type) ---");
    let gps1 = GPS::baru(6.0538, 102.2503);
    let gps2 = gps1;           // COPY!
    println!("gps1: {:?}", gps1); // OK!
    println!("gps2: {:?}", gps2); // OK!
    println!("Jarak: {:.3}km", gps1.jarak_dari(gps2)); // gps2 di-copy ke fungsi

    // ── Demo 2: Pekerja adalah Move ───────────────────────────
    println!("\n--- Pekerja (Move Type) ---");
    let p1 = Pekerja {
        id:     1,
        nama:   "Ali Ahmad".into(),
        lokasi: GPS::baru(6.0540, 102.2510),
        rekod:  Vec::new(),
    };

    let p2 = p1.clone(); // Clone — p1 masih valid!
    println!("Selepas clone: {:?}", p1.nama); // OK

    // ── Demo 3: Sistem lengkap ────────────────────────────────
    println!("\n--- Sistem Kehadiran ---");
    let mut sistem = SistemKehadiran::baru();

    // Daftar pekerja — MOVE ke sistem
    sistem.daftar_pekerja(p1);  // p1 move ke sistem
    sistem.daftar_pekerja(p2);  // p2 move ke sistem
    // p1, p2 tidak valid lagi!

    sistem.daftar_pekerja(Pekerja {
        id:     3,
        nama:   "Amin Razak".into(),
        lokasi: GPS::baru(6.0600, 102.2600), // jauh dari pejabat
        rekod:  Vec::new(),
    });

    // Daftar masuk dengan GPS (Copy — boleh hantar terus)
    let lokasi_ali   = GPS::baru(6.0539, 102.2504); // dekat pejabat
    let lokasi_siti  = GPS::baru(6.0538, 102.2503); // tepat pejabat
    let lokasi_amin  = GPS::baru(6.0700, 102.2700); // jauh

    sistem.daftar_masuk(1, lokasi_ali);
    sistem.daftar_masuk(2, lokasi_siti);
    sistem.daftar_masuk(3, lokasi_amin);

    // Masih boleh guna GPS (Copy!)
    println!("Lokasi Ali masih valid: {:?}", lokasi_ali);

    // Cetak laporan (pinjam sistem)
    sistem.cetak_laporan();

    // Cari pekerja (pinjam)
    if let Some(p) = sistem.cari(1) {
        println!("\nPekerja 1: {}", p.nama);
        // p adalah &Pekerja — pinjam sahaja
    }

    // eksport_log — CONSUME sistem (Move!)
    let log = sistem.eksport_log();
    // sistem tidak valid lagi!

    println!("\n=== Log Global ===");
    for entri in &log {
        println!("{}", entri);
    }

    println!("\n=== Ringkasan Move vs Copy ===");
    println!("GPS (Copy):     boleh guna selepas assign, pass ke fungsi, dll");
    println!("Pekerja (Move): ownership pindah, perlu clone untuk kekalkan");
    println!("&Pekerja (ref): pinjam sahaja, asal masih valid");
}
```

---

# 📋 Rujukan Pantas — Move vs Copy

## Carta Ringkas

```
SOALAN: Adakah jenis ini Copy atau Move?

Ada heap allocation? (String, Vec, HashMap, Box)
  YA  → MOVE
  TIDAK → teruskan...

Ada implementasi Drop?
  YA  → MOVE (Copy dan Drop tidak boleh bersama)
  TIDAK → teruskan...

Semua field/elemen adalah Copy?
  YA  → MUNGKIN Copy (kalau derive Copy)
  TIDAK → MOVE

Adakah #[derive(Copy, Clone)]?
  YA  → COPY
  TIDAK → MOVE (default untuk struct/enum)
```

## Jadual Jenis

```
Copy Types                       Move Types
──────────────────────────────────────────────────────
Semua integer (i32, u64, ...)    String
Semua float (f32, f64)           Vec<T>
bool, char                       HashMap<K,V>, HashSet<T>
&T (immutable reference)         Box<T>
[T; N] jika T: Copy              Rc<T>, Arc<T>
(A, B) jika A,B: Copy            Closure (yang capture move)
struct/enum jika derive(Copy)    struct/enum dengan non-Copy field
  + semua field Copy             File, Mutex, MutexGuard, dll
```

## Kaedah Berguna

```rust
// Bila perlu "move" tapi kekalkan placeholder
std::mem::take(&mut field)      // ambil field, ganti dengan Default
std::mem::replace(&mut f, new)  // ambil field, ganti dengan new

// Bila perlu guna selepas move
value.clone()                    // buat salinan
&value                           // pinjam sahaja

// Periksa sama ada Copy
fn is_copy<T: Copy>(_: &T) {}
is_copy(&42i32);                 // OK — i32 adalah Copy
// is_copy(&String::new());      // ERROR — String bukan Copy
```

## Peraturan Mudah

```
1. Primitif (int, float, bool, char) = COPY
2. Reference (&T) = COPY (pointer yang kecil)
3. String, Vec, HashMap = MOVE (ada heap)
4. Struct anda = MOVE melainkan anda #[derive(Copy)] + semua field Copy
5. Clone = buat salinan, MAHAL
6. Move = pindah ownership, MURAH
7. Copy = salin bytes, MURAH (tapi terhad pada jenis kecil)
8. Drop + Copy = MUSTAHIL (Rust tidak benarkan)
```

---

*Move semantics adalah jiwa Rust — ownership yang jelas, resource yang selamat.*
*Fahami stack vs heap, dan separuh misteri Move vs Copy terungkai.* 🦀
