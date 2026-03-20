# 🎭 Advanced Traits dalam Rust — Beginner to Advanced

> Panduan lengkap traits: dari asas hingga associated types, operator overloading, dan trait objects.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu Trait?

```
Trait = "kontrak" yang struct mesti penuhi.
        Macam interface dalam Java/C#, tapi lebih berkuasa!

┌─────────────────────────────────────────────────┐
│  trait Bunyi {                                  │
│      fn bunyi(&self) -> &str;  ← "kontrak"     │
│  }                                              │
└─────────────────────────────────────────────────┘
         ↓ implement                ↓ implement
   ┌──────────┐               ┌──────────┐
   │  Kucing  │ "Meow"        │  Anjing  │ "Woof"
   └──────────┘               └──────────┘

Semua yang implement Bunyi boleh dipanggil .bunyi()
walaupun type mereka berbeza!
```

---

## Kenapa Traits Istimewa dalam Rust?

```
Compared to OOP inheritance:
  OOP:  class Anjing extends Haiwan  (hierarki kaku)
  Rust: impl Haiwan for Anjing       (komposisi fleksibel)

Kebaikan Traits:
  ✔ Tiada "diamond problem"
  ✔ Boleh implement untuk type orang lain (Extension traits)
  ✔ Zero-cost abstraction (static dispatch)
  ✔ Trait objects untuk dynamic dispatch
  ✔ Associated types & generics yang berkuasa
  ✔ Conditional implementation
```

---

## Peta Pembelajaran

```
Bab 1  → Trait Asas & Default Methods
Bab 2  → Trait Bounds & Generic Constraints
Bab 3  → Associated Types
Bab 4  → Operator Overloading (std::ops)
Bab 5  → Trait Objects — dyn Trait
Bab 6  → Supertraits & Inheritance
Bab 7  → Blanket Implementations
Bab 8  → Advanced: GATs, impl Trait, RPITIT
Bab 9  → Standard Library Traits Penting
Bab 10 → Mini Project: Plugin System
```

---

# BAB 1: Trait Asas & Default Methods 🌱

## Define & Implement Trait

```rust
// Define trait
trait Huraikan {
    // Required method — mesti implement
    fn hurai(&self) -> String;

    // Default method — boleh override, atau guna default
    fn cetak(&self) {
        println!("{}", self.hurai());
    }

    fn panjang_hurai(&self) -> usize {
        self.hurai().len()  // guna method lain dalam trait!
    }
}

struct Pekerja {
    nama:     String,
    bahagian: String,
}

struct Produk {
    nama:  String,
    harga: f64,
}

// Implement trait untuk Pekerja
impl Huraikan for Pekerja {
    fn hurai(&self) -> String {
        format!("{} ({})", self.nama, self.bahagian)
    }
    // cetak() dan panjang_hurai() guna default
}

// Implement trait untuk Produk
impl Huraikan for Produk {
    fn hurai(&self) -> String {
        format!("{} — RM{:.2}", self.nama, self.harga)
    }

    // Override default method
    fn cetak(&self) {
        println!("🛒 {}", self.hurai());
    }
}

fn main() {
    let p = Pekerja { nama: "Ali".into(), bahagian: "ICT".into() };
    let b = Produk  { nama: "Beras 5kg".into(), harga: 12.90 };

    p.cetak(); // "Ali (ICT)"        ← default
    b.cetak(); // "🛒 Beras 5kg — RM12.90" ← overridden

    println!("Panjang: {}", p.panjang_hurai()); // 8
    println!("Panjang: {}", b.panjang_hurai()); // 20
}
```

---

## Trait dengan Banyak Methods

```rust
use std::fmt;

trait Bentuk {
    fn luas(&self) -> f64;
    fn perimeter(&self) -> f64;
    fn nama(&self) -> &str;

    // Default — berdasarkan method lain
    fn adakah_lebih_besar_dari(&self, lain: &dyn Bentuk) -> bool {
        self.luas() > lain.luas()
    }

    fn ringkasan(&self) -> String {
        format!("{}: luas={:.2}, perimeter={:.2}",
            self.nama(), self.luas(), self.perimeter())
    }
}

struct Bulatan { radius: f64 }
struct Segiempat { lebar: f64, tinggi: f64 }
struct Segitiga { a: f64, b: f64, c: f64 }

impl Bentuk for Bulatan {
    fn luas(&self) -> f64 { std::f64::consts::PI * self.radius.powi(2) }
    fn perimeter(&self) -> f64 { 2.0 * std::f64::consts::PI * self.radius }
    fn nama(&self) -> &str { "Bulatan" }
}

impl Bentuk for Segiempat {
    fn luas(&self) -> f64 { self.lebar * self.tinggi }
    fn perimeter(&self) -> f64 { 2.0 * (self.lebar + self.tinggi) }
    fn nama(&self) -> &str { "Segiempat" }
}

impl Bentuk for Segitiga {
    fn luas(&self) -> f64 {
        let s = (self.a + self.b + self.c) / 2.0;
        (s * (s-self.a) * (s-self.b) * (s-self.c)).sqrt()
    }
    fn perimeter(&self) -> f64 { self.a + self.b + self.c }
    fn nama(&self) -> &str { "Segitiga" }
}

fn main() {
    let bentuk: Vec<Box<dyn Bentuk>> = vec![
        Box::new(Bulatan    { radius: 5.0 }),
        Box::new(Segiempat  { lebar: 4.0, tinggi: 6.0 }),
        Box::new(Segitiga   { a: 3.0, b: 4.0, c: 5.0 }),
    ];

    for b in &bentuk {
        println!("{}", b.ringkasan());
    }

    // Bandingkan
    let bulatan = Bulatan { radius: 5.0 };
    let segi    = Segiempat { lebar: 4.0, tinggi: 6.0 };
    println!("Bulatan lebih besar? {}", bulatan.adakah_lebih_besar_dari(&segi));
}
```

---

## 🧠 Brain Teaser #1

Apakah masalah dengan kod ini?

```rust
trait Cetak {
    fn cetak(&self);
    fn cetak_dua_kali(&self) {
        self.cetak();
        self.cetak();
    }
}

struct Nombor(i32);

// Tidak implement Cetak untuk Nombor!
// Nombor::cetak_dua_kali(???)

fn main() {
    let n = Nombor(42);
    n.cetak_dua_kali(); // adakah ini boleh?
}
```

<details>
<summary>👀 Jawapan</summary>

**Tidak boleh compile** — `Nombor` tidak implement `Cetak`.

Walaupun `cetak_dua_kali` ada default implementation, ia memanggil `self.cetak()` yang merupakan *required method*. Rust tidak boleh tahu apa yang perlu di-print tanpa implementation.

Betulkan:
```rust
impl Cetak for Nombor {
    fn cetak(&self) {
        println!("{}", self.0);
    }
    // cetak_dua_kali() guna default — OK!
}

fn main() {
    let n = Nombor(42);
    n.cetak_dua_kali(); // print "42" dua kali
}
```

**Peraturan:** Default methods boleh memanggil required methods, tapi struct mesti implement SEMUA required methods terlebih dahulu.
</details>

---

# BAB 2: Trait Bounds & Generic Constraints 🔒

## Trait Bounds Asas

```rust
use std::fmt::Display;

// Fungsi yang terima apa-apa type yang implement Display
fn cetak_nilai<T: Display>(nilai: T) {
    println!("Nilai: {}", nilai);
}

// Multiple bounds dengan +
fn cetak_dan_debug<T: Display + std::fmt::Debug>(nilai: T) {
    println!("Display: {}", nilai);
    println!("Debug:   {:?}", nilai);
}

// Where clause — lebih bersih untuk bounds yang panjang
fn bandingkan<T>(a: T, b: T) -> T
where
    T: PartialOrd + Display + Clone,
{
    if a > b {
        println!("{} lebih besar dari {}", a, b);
        a
    } else {
        println!("{} lebih kecil atau sama dengan {}", a, b);
        b
    }
}

fn main() {
    cetak_nilai(42);
    cetak_nilai("hello");
    cetak_nilai(3.14);

    cetak_dan_debug(vec![1, 2, 3]);

    let besar = bandingkan(10, 20);
    println!("Besar: {}", besar);
}
```

---

## Multiple Trait Bounds dalam Struct

```rust
use std::fmt::{Display, Debug};

// Struct generic dengan trait bounds
struct Pembungkus<T: Display + Debug + Clone> {
    nilai: T,
    label: String,
}

impl<T: Display + Debug + Clone> Pembungkus<T> {
    fn baru(nilai: T, label: &str) -> Self {
        Pembungkus { nilai, label: label.into() }
    }

    fn papar(&self) {
        println!("[{}] Display: {}", self.label, self.nilai);
        println!("[{}] Debug:   {:?}", self.label, self.nilai);
    }

    fn dapatkan(&self) -> T {
        self.nilai.clone()
    }
}

// Implement Display untuk Pembungkus
impl<T: Display + Debug + Clone> Display for Pembungkus<T> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Pembungkus({}: {})", self.label, self.nilai)
    }
}

fn main() {
    let w1 = Pembungkus::baru(42i32, "nombor");
    let w2 = Pembungkus::baru("hello", "teks");
    let w3 = Pembungkus::baru(vec![1, 2, 3], "senarai");

    w1.papar();
    w2.papar();
    w3.papar();

    println!("{}", w1); // Pembungkus(nombor: 42)
}
```

---

## Trait Bounds pada Return Type

```rust
use std::fmt::Display;

// impl Trait dalam return position — static dispatch
fn buat_display() -> impl Display {
    "Hello, Rust!"  // return concrete type yang implement Display
}

// Setiap call mesti return TYPE YANG SAMA
fn pilih(pilihan: bool) -> impl Display {
    if pilihan {
        "Ya".to_string()
    } else {
        "Tidak".to_string()
    }
    // ✔ kedua-dua return String — OK
}

// TIDAK boleh return pelbagai type berbeza dengan impl Trait
// fn pilih_berbeza(pilihan: bool) -> impl Display {
//     if pilihan { "Ya" }     // &str
//     else { 42 }             // i32 ← ERROR!
// }

fn main() {
    let d = buat_display();
    println!("{}", d);

    println!("{}", pilih(true));
    println!("{}", pilih(false));
}
```

---

# BAB 3: Associated Types 🔗

## Trait dengan Associated Type

```rust
// Associated type — type yang ditentukan oleh implementor
trait Penukar {
    type Output;  // ← associated type

    fn tukar(&self) -> Self::Output;
}

struct Celsius(f64);
struct Fahrenheit(f64);
struct Kelvin(f64);

impl Penukar for Celsius {
    type Output = Fahrenheit;  // Celsius → Fahrenheit

    fn tukar(&self) -> Fahrenheit {
        Fahrenheit(self.0 * 9.0 / 5.0 + 32.0)
    }
}

impl Penukar for Fahrenheit {
    type Output = Celsius;  // Fahrenheit → Celsius

    fn tukar(&self) -> Celsius {
        Celsius((self.0 - 32.0) * 5.0 / 9.0)
    }
}

fn main() {
    let c = Celsius(100.0);
    let f = c.tukar();
    println!("{}°C = {}°F", c.0, f.0); // 100°C = 212°F

    let f2 = Fahrenheit(98.6);
    let c2 = f2.tukar();
    println!("{}°F = {:.2}°C", f2.0, c2.0); // 98.6°F = 37.00°C
}
```

---

## Associated Types vs Generics

```rust
// ❌ Trait dengan generic parameter — lebih verbose
trait TambahanG<T> {
    fn tambah_g(&self, lain: &T) -> T;
}

// ✔ Trait dengan associated type — lebih bersih
trait Tambahan {
    type Output;
    fn tambah(&self, lain: &Self) -> Self::Output;
}

// Dengan generic: boleh implement BANYAK kali untuk type berbeza
// impl TambahanG<i32> for Vektor2D { ... }
// impl TambahanG<f64> for Vektor2D { ... }

// Dengan associated type: hanya SATU implementation per type
// impl Tambahan for Vektor2D { type Output = Vektor2D; ... }

#[derive(Debug, Clone, Copy)]
struct Vektor2D { x: f64, y: f64 }

impl Tambahan for Vektor2D {
    type Output = Self;

    fn tambah(&self, lain: &Self) -> Self {
        Vektor2D { x: self.x + lain.x, y: self.y + lain.y }
    }
}

fn main() {
    let v1 = Vektor2D { x: 1.0, y: 2.0 };
    let v2 = Vektor2D { x: 3.0, y: 4.0 };
    let v3 = v1.tambah(&v2);
    println!("{:?}", v3); // Vektor2D { x: 4.0, y: 6.0 }
}
```

---

## Iterator — Contoh Associated Type Paling Terkenal

```rust
struct Kiraan {
    semasa: u32,
    had:    u32,
}

impl Kiraan {
    fn baru(had: u32) -> Self {
        Kiraan { semasa: 0, had }
    }
}

// Iterator menggunakan associated type!
impl Iterator for Kiraan {
    type Item = u32;  // ← associated type menentukan jenis elemen

    fn next(&mut self) -> Option<Self::Item> {
        if self.semasa < self.had {
            self.semasa += 1;
            Some(self.semasa)
        } else {
            None
        }
    }
}

fn main() {
    let k = Kiraan::baru(5);

    // Semua Iterator methods tersedia!
    let jumlah: u32 = k.sum();
    println!("Jumlah: {}", jumlah); // 15

    // Guna dalam for loop
    for n in Kiraan::baru(3) {
        println!("{}", n); // 1, 2, 3
    }

    // Chaining methods
    let hasil: Vec<u32> = Kiraan::baru(5)
        .filter(|&n| n % 2 != 0)
        .map(|n| n * n)
        .collect();
    println!("{:?}", hasil); // [1, 9, 25]
}
```

---

## 🧠 Brain Teaser #2

Bila nak guna Associated Type vs Generic Parameter dalam trait?

```rust
// Pilihan A: Associated Type
trait Proses {
    type Hasil;
    fn proses(&self) -> Self::Hasil;
}

// Pilihan B: Generic Parameter
trait ProsesG<T> {
    fn proses_g(&self) -> T;
}
```

<details>
<summary>👀 Jawapan</summary>

**Guna Associated Type bila:**
- Satu type hanya ada SATU cara yang logik untuk implement trait
- Contoh: `Iterator` — satu collection hanya ada satu jenis element
- Contoh: `Add` — satu type biasanya hanya ada satu output type
- Lebih bersih, kurang repetition dalam code

```rust
impl Iterator for MyVec {
    type Item = i32;  // hanya satu Item type
}
```

**Guna Generic Parameter bila:**
- Satu type boleh implement trait untuk PELBAGAI types
- Contoh: `From<T>` — `String` boleh convert dari `&str`, `i32`, dll
- Lebih fleksibel

```rust
impl From<&str> for String { ... }
impl From<i32>  for String { ... }
// String implement From BANYAK kali
```

**Petanda mudah:** Kalau hanya ada "satu jawapan yang logik" → Associated Type. Kalau boleh ada "banyak jawapan" → Generic.
</details>

---

# BAB 4: Operator Overloading (std::ops) ➕

## Implement Add, Sub, Mul

```rust
use std::ops::{Add, Sub, Mul, Neg};
use std::fmt;

#[derive(Debug, Clone, Copy, PartialEq)]
struct Vektor2D {
    x: f64,
    y: f64,
}

impl Vektor2D {
    fn baru(x: f64, y: f64) -> Self { Vektor2D { x, y } }
    fn magnitud(&self) -> f64 { (self.x*self.x + self.y*self.y).sqrt() }
    fn normal(&self) -> Self {
        let m = self.magnitud();
        Vektor2D { x: self.x/m, y: self.y/m }
    }
    fn titik(&self, lain: &Self) -> f64 { self.x*lain.x + self.y*lain.y }
}

// v1 + v2
impl Add for Vektor2D {
    type Output = Self;
    fn add(self, lain: Self) -> Self {
        Vektor2D { x: self.x + lain.x, y: self.y + lain.y }
    }
}

// v1 - v2
impl Sub for Vektor2D {
    type Output = Self;
    fn sub(self, lain: Self) -> Self {
        Vektor2D { x: self.x - lain.x, y: self.y - lain.y }
    }
}

// v * scalar
impl Mul<f64> for Vektor2D {
    type Output = Self;
    fn mul(self, skalar: f64) -> Self {
        Vektor2D { x: self.x * skalar, y: self.y * skalar }
    }
}

// scalar * v (kedua arah!)
impl Mul<Vektor2D> for f64 {
    type Output = Vektor2D;
    fn mul(self, v: Vektor2D) -> Vektor2D {
        Vektor2D { x: self * v.x, y: self * v.y }
    }
}

// -v (negasi)
impl Neg for Vektor2D {
    type Output = Self;
    fn neg(self) -> Self {
        Vektor2D { x: -self.x, y: -self.y }
    }
}

impl fmt::Display for Vektor2D {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({:.2}, {:.2})", self.x, self.y)
    }
}

fn main() {
    let v1 = Vektor2D::baru(3.0, 4.0);
    let v2 = Vektor2D::baru(1.0, 2.0);

    println!("v1 = {}", v1);
    println!("v2 = {}", v2);
    println!("v1 + v2 = {}", v1 + v2);
    println!("v1 - v2 = {}", v1 - v2);
    println!("v1 * 2  = {}", v1 * 2.0);
    println!("3 * v2  = {}", 3.0 * v2);
    println!("-v1     = {}", -v1);
    println!("|v1|    = {:.2}", v1.magnitud()); // 5.0
    println!("v1·v2   = {:.2}", v1.titik(&v2));  // 11.0
}
```

---

## Index, IndexMut, Deref

```rust
use std::ops::{Index, IndexMut, Deref, DerefMut};

// Grid 2D yang boleh akses dengan grid[baris][lajur]
struct Grid {
    data:  Vec<Vec<f64>>,
    baris: usize,
    lajur: usize,
}

impl Grid {
    fn baru(baris: usize, lajur: usize, nilai: f64) -> Self {
        Grid {
            data:  vec![vec![nilai; lajur]; baris],
            baris,
            lajur,
        }
    }
}

impl Index<usize> for Grid {
    type Output = Vec<f64>;
    fn index(&self, i: usize) -> &Vec<f64> {
        &self.data[i]
    }
}

impl IndexMut<usize> for Grid {
    fn index_mut(&mut self, i: usize) -> &mut Vec<f64> {
        &mut self.data[i]
    }
}

// Newtype pattern dengan Deref
struct NomborsVec(Vec<i32>);

impl Deref for NomborsVec {
    type Target = Vec<i32>;
    fn deref(&self) -> &Vec<i32> { &self.0 }
}

impl DerefMut for NomborsVec {
    fn deref_mut(&mut self) -> &mut Vec<i32> { &mut self.0 }
}

fn main() {
    let mut grid = Grid::baru(3, 3, 0.0);
    grid[0][0] = 1.0;
    grid[1][1] = 5.0;
    grid[2][2] = 9.0;

    for baris in 0..3 {
        println!("{:?}", &grid[baris]);
    }

    // NomborsVec guna semua Vec methods melalui Deref!
    let mut v = NomborsVec(vec![1, 2, 3]);
    v.push(4);           // Vec::push melalui DerefMut
    println!("{}", v.len()); // Vec::len melalui Deref
    println!("{:?}", *v);   // [1, 2, 3, 4]
}
```

---

## PartialEq, Eq, PartialOrd, Ord

```rust
use std::cmp::Ordering;

#[derive(Debug, Clone)]
struct Nilai {
    nombor:  i32,
    label:   String,
}

// Implement PartialEq — samarata berdasarkan nombor sahaja
impl PartialEq for Nilai {
    fn eq(&self, lain: &Self) -> bool {
        self.nombor == lain.nombor
    }
}

impl Eq for Nilai {}  // Marker trait — tiada method

// Implement PartialOrd
impl PartialOrd for Nilai {
    fn partial_cmp(&self, lain: &Self) -> Option<Ordering> {
        Some(self.cmp(lain))
    }
}

// Implement Ord — sort berdasarkan nombor
impl Ord for Nilai {
    fn cmp(&self, lain: &Self) -> Ordering {
        self.nombor.cmp(&lain.nombor)
    }
}

fn main() {
    let mut senarai = vec![
        Nilai { nombor: 30, label: "tiga puluh".into() },
        Nilai { nombor: 10, label: "sepuluh".into() },
        Nilai { nombor: 20, label: "dua puluh".into() },
    ];

    senarai.sort();  // guna Ord
    for v in &senarai {
        println!("{}: {}", v.nombor, v.label);
    }

    let a = Nilai { nombor: 5, label: "A".into() };
    let b = Nilai { nombor: 5, label: "B".into() };
    println!("a == b? {}", a == b); // true (sama nombor!)
    println!("a < b?  {}", a < b);  // false
}
```

---

# BAB 5: Trait Objects — dyn Trait 🎭

## Static vs Dynamic Dispatch

```
Static Dispatch (Generics / impl Trait):
  ┌─────────────────────────────────────────────┐
  │  fn cetak<T: Display>(v: T)                 │
  │                                             │
  │  Compiler buat SATU SALINAN fungsi          │
  │  untuk SETIAP type yang digunakan:          │
  │   cetak_i32(), cetak_str(), cetak_f64()...  │
  │                                             │
  │  + Sangat LAJU (no overhead)                │
  │  - Saiz binary lebih besar (monomorphization)│
  └─────────────────────────────────────────────┘

Dynamic Dispatch (dyn Trait):
  ┌─────────────────────────────────────────────┐
  │  fn cetak(v: &dyn Display)                  │
  │                                             │
  │  SATU fungsi untuk semua types              │
  │  Guna vtable untuk lookup method            │
  │                                             │
  │  + Saiz binary lebih kecil                  │
  │  + Boleh simpan pelbagai types dalam Vec    │
  │  - Sedikit lambat (pointer indirection)     │
  └─────────────────────────────────────────────┘
```

```rust
use std::fmt::Display;

// Static dispatch — T ditentukan semasa compile
fn cetak_static<T: Display>(nilai: &T) {
    println!("Static: {}", nilai);
}

// Dynamic dispatch — type ditentukan semasa runtime
fn cetak_dynamic(nilai: &dyn Display) {
    println!("Dynamic: {}", nilai);
}

fn main() {
    cetak_static(&42);
    cetak_static(&"hello");

    cetak_dynamic(&42);
    cetak_dynamic(&"hello");
    cetak_dynamic(&3.14);

    // Hanya dyn Trait boleh simpan pelbagai types!
    let senarai: Vec<Box<dyn Display>> = vec![
        Box::new(42),
        Box::new("hello"),
        Box::new(3.14),
        Box::new(true),
    ];

    for item in &senarai {
        println!("{}", item);
    }
}
```

---

## Trait Objects dalam Struct

```rust
trait Pemalam {
    fn nama(&self) -> &str;
    fn laksana(&self, input: &str) -> String;
}

struct PemalamBesarHuruf;
struct PemalamGanda;
struct PemalamBalik;

impl Pemalam for PemalamBesarHuruf {
    fn nama(&self) -> &str { "BesarHuruf" }
    fn laksana(&self, input: &str) -> String { input.to_uppercase() }
}

impl Pemalam for PemalamGanda {
    fn nama(&self) -> &str { "Ganda" }
    fn laksana(&self, input: &str) -> String { format!("{}{}", input, input) }
}

impl Pemalam for PemalamBalik {
    fn nama(&self) -> &str { "Balik" }
    fn laksana(&self, input: &str) -> String {
        input.chars().rev().collect()
    }
}

// Struct yang menyimpan Vec<Box<dyn Pemalam>>
struct SalurPaip {
    pemalam: Vec<Box<dyn Pemalam>>,
}

impl SalurPaip {
    fn baru() -> Self { SalurPaip { pemalam: vec![] } }

    fn tambah(mut self, p: Box<dyn Pemalam>) -> Self {
        self.pemalam.push(p);
        self
    }

    fn laksana(&self, mut input: String) -> String {
        for p in &self.pemalam {
            println!("  → [{}]: {} → ...", p.nama(), input);
            input = p.laksana(&input);
        }
        input
    }
}

fn main() {
    let paip = SalurPaip::baru()
        .tambah(Box::new(PemalamBesarHuruf))
        .tambah(Box::new(PemalamGanda))
        .tambah(Box::new(PemalamBalik));

    let hasil = paip.laksana("rust".to_string());
    println!("Hasil akhir: {}", hasil);
    // "rust" → "RUST" → "RUSTRUST" → "TSURTSUR"
}
```

---

## Object Safety — Syarat dyn Trait

```rust
// ✔ Object-safe trait — boleh jadi dyn Trait
trait Selamat {
    fn kaedah(&self) -> String;
    fn nilai_lalai(&self) -> i32 { 0 }
}

// ❌ TIDAK object-safe — return Self
// trait TidakSelamat {
//     fn klon(&self) -> Self;  // return Self — saiz tidak diketahui!
// }

// ❌ TIDAK object-safe — generic method
// trait TidakSelamat2 {
//     fn cetak<T: std::fmt::Display>(&self, val: T); // generic method
// }

// Penyelesaian: hilangkan generic dari trait, atau guna helper
trait SelamatV2 {
    fn cetak_nombor(&self, val: i32);   // specific type OK
    fn cetak_teks(&self, val: &str);    // specific type OK
}

// Object safety rules:
// 1. Return type bukan Self (atau guna where Self: Sized)
// 2. Tiada generic type parameters dalam method
// 3. Tiada associated constants (dalam beberapa kes)
```

---

# BAB 6: Supertraits & Inheritance 🏗️

## Supertrait — Trait yang Bergantung pada Trait Lain

```rust
use std::fmt;

// Display memerlukan fmt::Display
// Debug memerlukan fmt::Debug
// Huraikan memerlukan KEDUA-DUANYA
trait Huraikan: fmt::Display + fmt::Debug {
    fn hurai_penuh(&self) -> String {
        format!("Display: {} | Debug: {:?}", self, self)
    }
}

#[derive(Debug)]
struct Item {
    id:   u32,
    nama: String,
}

impl fmt::Display for Item {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Item#{} ({})", self.id, self.nama)
    }
}

// Mesti implement Display + Debug sebelum boleh implement Huraikan
impl Huraikan for Item {
    // hurai_penuh() ada default implementation
}

fn papar_huraian<T: Huraikan>(item: &T) {
    println!("{}", item.hurai_penuh());
}

fn main() {
    let item = Item { id: 1, nama: "Beras".into() };
    papar_huraian(&item);
    // Display: Item#1 (Beras) | Debug: Item { id: 1, nama: "Beras" }
}
```

---

## Fully Qualified Syntax — Selesai Kekeliruan

```rust
trait A {
    fn kaedah(&self) -> String;
}

trait B {
    fn kaedah(&self) -> String;
}

struct S;

impl A for S {
    fn kaedah(&self) -> String { "Dari A".into() }
}

impl B for S {
    fn kaedah(&self) -> String { "Dari B".into() }
}

impl S {
    fn kaedah(&self) -> String { "Dari S terus".into() }
}

fn main() {
    let s = S;

    // Kekeliruan — mana satu?
    // s.kaedah();  // ← ambiguous!

    // Fully qualified syntax
    println!("{}", S::kaedah(&s));   // Dari S terus (method struct)
    println!("{}", A::kaedah(&s));   // Dari A
    println!("{}", B::kaedah(&s));   // Dari B

    // Format lain:
    println!("{}", <S as A>::kaedah(&s)); // Dari A
    println!("{}", <S as B>::kaedah(&s)); // Dari B
}
```

---

## 🧠 Brain Teaser #3

Bagaimana keluarkan nilai `42` dari kod ini?

```rust
trait Nilai {
    fn nilai() -> i32;
}

struct A;
struct B;

impl Nilai for A {
    fn nilai() -> i32 { 10 }
}

impl Nilai for B {
    fn nilai() -> i32 { 32 }
}

fn main() {
    // Dapatkan 42 (A::nilai() + B::nilai())
    // Tanpa menulis nombor 42 terus!
}
```

<details>
<summary>👀 Jawapan</summary>

```rust
fn main() {
    let hasil = <A as Nilai>::nilai() + <B as Nilai>::nilai();
    println!("{}", hasil); // 42

    // Atau:
    println!("{}", A::nilai() + B::nilai()); // 42
}
```

`Nilai` adalah associated function (bukan method — tiada `&self`). Guna `Type::kaedah()` atau `<Type as Trait>::kaedah()`.
</details>

---

# BAB 7: Blanket Implementations 🌐

## Implement Trait untuk Semua Types yang Memenuhi Syarat

```rust
use std::fmt::Display;

// Tambah method `log` untuk SEMUA type yang implement Display
trait Log {
    fn log(&self, label: &str);
    fn log_ralat(&self, label: &str);
}

// Blanket implementation — implement untuk semua T: Display
impl<T: Display> Log for T {
    fn log(&self, label: &str) {
        println!("[INFO]  {} = {}", label, self);
    }
    fn log_ralat(&self, label: &str) {
        eprintln!("[ERROR] {} = {}", label, self);
    }
}

fn main() {
    // SEMUA type yang implement Display boleh guna .log()!
    42.log("nombor");
    "hello".log("teks");
    3.14f64.log("float");
    vec![1,2,3].iter().count().log("panjang"); // usize implement Display

    "ralat berlaku".log_ralat("mesej");
}
```

---

## ToString — Blanket dalam Standard Library

```rust
// Contoh nyata blanket dalam std:
// impl<T: Display> ToString for T { ... }
// Ini bermakna SEMUA type yang implement Display boleh .to_string()!

fn main() {
    let s1: String = 42.to_string();       // i32 → String
    let s2: String = 3.14.to_string();     // f64 → String
    let s3: String = true.to_string();     // bool → String
    let s4: String = 'A'.to_string();      // char → String

    println!("{} {} {} {}", s1, s2, s3, s4);
}
```

---

## Implement Trait untuk Vec<T> dengan Bound

```rust
trait Statistik {
    fn min(&self) -> Option<f64>;
    fn max(&self) -> Option<f64>;
    fn purata(&self) -> Option<f64>;
}

// Implement untuk Vec<T> di mana T boleh jadi f64
impl<T: Into<f64> + Copy> Statistik for Vec<T> {
    fn min(&self) -> Option<f64> {
        self.iter().map(|&x| x.into()).reduce(f64::min)
    }

    fn max(&self) -> Option<f64> {
        self.iter().map(|&x| x.into()).reduce(f64::max)
    }

    fn purata(&self) -> Option<f64> {
        if self.is_empty() { return None; }
        let jumlah: f64 = self.iter().map(|&x| x.into()).sum();
        Some(jumlah / self.len() as f64)
    }
}

fn main() {
    let markah_int: Vec<i32>  = vec![85, 92, 78, 95, 60];
    let suhu_float: Vec<f64>  = vec![28.5, 30.1, 27.8, 29.3];

    println!("Markah — min:{:?} max:{:?} purata:{:.2}",
        markah_int.min(),
        markah_int.max(),
        markah_int.purata().unwrap());

    println!("Suhu   — min:{:?} max:{:?} purata:{:.2}",
        suhu_float.min(),
        suhu_float.max(),
        suhu_float.purata().unwrap());
}
```

---

# BAB 8: Advanced — impl Trait, GATs, dan Lagi 🚀

## impl Trait — Input & Output Position

```rust
use std::fmt::Display;

// Input position — abstract over concrete type
fn jumlah_display(a: impl Display, b: impl Display) -> String {
    format!("{} dan {}", a, b)
}

// Output position — hide concrete return type
fn buat_greeter(nama: String) -> impl Fn() -> String {
    move || format!("Hello, {}!", nama)
}

// impl Trait dalam struct field — TIDAK BOLEH
// struct S { field: impl Display }  // ← ERROR!

// Guna generic sebagai gantinya
struct S<T: Display> { field: T }

fn main() {
    println!("{}", jumlah_display(42, "hello"));

    let sapa = buat_greeter("Ali".into());
    println!("{}", sapa());  // Hello, Ali!
    println!("{}", sapa());  // boleh guna berkali-kali
}
```

---

## RPITIT — Return Position impl Trait in Trait

```rust
// Rust 1.75+ — boleh return impl Trait dari trait method!
trait Pengeluar {
    fn hasilkan(&self) -> impl Iterator<Item = i32>;
}

struct NomborGenap { had: i32 }
struct NomborPrima { had: i32 }

impl Pengeluar for NomborGenap {
    fn hasilkan(&self) -> impl Iterator<Item = i32> {
        (0..=self.had).filter(|n| n % 2 == 0)
    }
}

fn adalah_prima(n: i32) -> bool {
    if n < 2 { return false; }
    !(2..=(n as f64).sqrt() as i32).any(|i| n % i == 0)
}

impl Pengeluar for NomborPrima {
    fn hasilkan(&self) -> impl Iterator<Item = i32> {
        (2..=self.had).filter(|&n| adalah_prima(n))
    }
}

fn main() {
    let genap = NomborGenap { had: 10 };
    let prima = NomborPrima { had: 20 };

    print!("Genap: ");
    for n in genap.hasilkan() { print!("{} ", n); }
    println!();

    print!("Prima: ");
    for n in prima.hasilkan() { print!("{} ", n); }
    println!();
}
```

---

## Sealed Trait — Hadkan Implementation

```rust
// "Sealed trait" — hanya code dalam module ini boleh implement
mod sealed {
    // Trait private — orang lain tidak boleh implement
    pub trait Sealed {}

    // Implement hanya untuk types kita nak
    impl Sealed for i32   {}
    impl Sealed for f64   {}
    impl Sealed for String {}
}

// Trait public yang bergantung pada Sealed
pub trait TipeKami: sealed::Sealed {
    fn proses(&self) -> String;
}

impl TipeKami for i32 {
    fn proses(&self) -> String { format!("i32: {}", self) }
}

impl TipeKami for f64 {
    fn proses(&self) -> String { format!("f64: {:.2}", self) }
}

// Pengguna LUAR tidak boleh implement TipeKami untuk type mereka
// kerana mereka tidak boleh implement sealed::Sealed!

fn main() {
    let a: i32 = 42;
    let b: f64 = 3.14;

    println!("{}", a.proses()); // i32: 42
    println!("{}", b.proses()); // f64: 3.14
}
```

---

# BAB 9: Standard Library Traits Penting 📚

## From, Into, TryFrom, TryInto

```rust
use std::convert::{TryFrom, TryInto};

// From<T> — convert dari T, tidak boleh gagal
#[derive(Debug)]
struct Meter(f64);

#[derive(Debug)]
struct Kilometer(f64);

impl From<Meter> for Kilometer {
    fn from(m: Meter) -> Self {
        Kilometer(m.0 / 1000.0)
    }
}

// Bila implement From, Into adalah automatik!
// impl Into<Kilometer> for Meter  ← tidak perlu implement manual

// TryFrom<T> — convert yang mungkin gagal
#[derive(Debug)]
struct AngkaPositif(u32);

impl TryFrom<i32> for AngkaPositif {
    type Error = String;

    fn try_from(nilai: i32) -> Result<Self, Self::Error> {
        if nilai < 0 {
            Err(format!("{} adalah negatif!", nilai))
        } else {
            Ok(AngkaPositif(nilai as u32))
        }
    }
}

fn main() {
    // From
    let jarak_m = Meter(1500.0);
    let jarak_km = Kilometer::from(jarak_m);
    println!("{:?}", jarak_km); // Kilometer(1.5)

    // Into (automatik dari From)
    let m2 = Meter(2000.0);
    let km2: Kilometer = m2.into(); // guna Into yang auto-generated
    println!("{:?}", km2); // Kilometer(2.0)

    // TryFrom
    println!("{:?}", AngkaPositif::try_from(42));  // Ok(AngkaPositif(42))
    println!("{:?}", AngkaPositif::try_from(-5));  // Err("...")

    // TryInto (auto dari TryFrom)
    let hasil: Result<AngkaPositif, _> = 10i32.try_into();
    println!("{:?}", hasil);  // Ok(AngkaPositif(10))
}
```

---

## Clone, Copy, Drop

```rust
// Clone — buat salinan explicit
#[derive(Debug, Clone)]
struct DataBesar {
    data: Vec<i32>,
    nama: String,
}

// Copy — buat salinan implisit (murah, stack-only)
#[derive(Debug, Clone, Copy)]
struct Titik { x: f32, y: f32 }

// Drop — cleanup semasa destruksi
struct Sambungan {
    host: String,
}

impl Drop for Sambungan {
    fn drop(&mut self) {
        println!("Tutup sambungan ke {}", self.host);
    }
}

fn main() {
    // Clone — explicit copy
    let d1 = DataBesar { data: vec![1,2,3], nama: "asal".into() };
    let d2 = d1.clone(); // explicit clone
    println!("{:?}", d1); // d1 masih valid!
    println!("{:?}", d2);

    // Copy — implicit copy
    let t1 = Titik { x: 1.0, y: 2.0 };
    let t2 = t1; // copy (bukan move) kerana Copy trait
    println!("{:?}", t1); // t1 masih valid!
    println!("{:?}", t2);

    // Drop
    {
        let _s = Sambungan { host: "localhost".into() };
        println!("Dalam scope");
    } // ← drop() dipanggil di sini!
    println!("Selepas scope");
}
```

---

## Display & Debug

```rust
use std::fmt;

struct Matrik {
    a: f64, b: f64,
    c: f64, d: f64,
}

// Debug — untuk programmer, guna {:?}
impl fmt::Debug for Matrik {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("Matrik")
            .field("a", &self.a)
            .field("b", &self.b)
            .field("c", &self.c)
            .field("d", &self.d)
            .finish()
    }
}

// Display — untuk pengguna, guna {}
impl fmt::Display for Matrik {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "┌{:6.2} {:6.2}┐\n└{:6.2} {:6.2}┘",
            self.a, self.b, self.c, self.d)
    }
}

fn main() {
    let m = Matrik { a: 1.0, b: 2.0, c: 3.0, d: 4.0 };

    println!("Debug:   {:?}", m);
    println!("Pretty:  {:#?}", m);
    println!("Display:\n{}", m);
}
```

---

## Default, Hash, PartialEq

```rust
use std::collections::HashMap;
use std::hash::{Hash, Hasher};

// Default — nilai lalai
#[derive(Debug)]
struct Konfigurasi {
    host:     String,
    port:     u16,
    debug:    bool,
    max_conn: u32,
}

impl Default for Konfigurasi {
    fn default() -> Self {
        Konfigurasi {
            host:     "localhost".into(),
            port:     8080,
            debug:    false,
            max_conn: 100,
        }
    }
}

// Custom Hash — hash hanya berdasarkan id
#[derive(Debug, Eq, PartialEq)]
struct Pengguna {
    id:   u32,
    nama: String,
}

impl Hash for Pengguna {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.id.hash(state); // hanya hash id, ignore nama
    }
}

fn main() {
    // Default
    let cfg = Konfigurasi::default();
    println!("{:?}", cfg);

    // Guna default dengan sebahagian override
    let cfg2 = Konfigurasi {
        port: 3000,
        debug: true,
        ..Konfigurasi::default()  // baki guna default
    };
    println!("Port: {}, Debug: {}", cfg2.port, cfg2.debug);

    // Hash dalam HashMap
    let mut direktori: HashMap<Pengguna, String> = HashMap::new();
    let u = Pengguna { id: 1, nama: "Ali".into() };
    direktori.insert(u, "ICT".into());

    // Cari dengan struct lain yang sama id
    let cari = Pengguna { id: 1, nama: "Ali Berbeza".into() };
    println!("{:?}", direktori.get(&cari)); // Some("ICT") — sama hash!
}
```

---

# BAB 10: Mini Project — Plugin System 🔌

```rust
use std::collections::HashMap;
use std::fmt;

// ─── Trait Definisi ───────────────────────────────────────────

trait Pemalam: fmt::Debug {
    fn nama(&self)    -> &str;
    fn versi(&self)   -> &str;
    fn huraian(&self) -> &str;

    // Process input — return output atau error
    fn proses(&self, input: &str) -> Result<String, String>;

    // Boleh handle input ini?
    fn boleh_proses(&self, input: &str) -> bool {
        !input.is_empty()
    }

    // Priority — pemalam dengan priority tinggi diproses dulu
    fn prioriti(&self) -> u8 { 5 }
}

// ─── Pemalam Konkrit ──────────────────────────────────────────

#[derive(Debug)]
struct PemalamUbahBesarHuruf;

#[derive(Debug)]
struct PemalamBilangPerkataan;

#[derive(Debug)]
struct PemalamBalikTeks;

#[derive(Debug)]
struct PemalamSensorKata {
    kata_larangan: Vec<String>,
}

#[derive(Debug)]
struct PemalamHadPanjang {
    had: usize,
}

impl Pemalam for PemalamUbahBesarHuruf {
    fn nama(&self)    -> &str { "UbahBesarHuruf" }
    fn versi(&self)   -> &str { "1.0" }
    fn huraian(&self) -> &str { "Tukar semua huruf kepada huruf besar" }
    fn prioriti(&self) -> u8  { 3 }

    fn proses(&self, input: &str) -> Result<String, String> {
        Ok(input.to_uppercase())
    }
}

impl Pemalam for PemalamBilangPerkataan {
    fn nama(&self)    -> &str { "BilangPerkataan" }
    fn versi(&self)   -> &str { "1.2" }
    fn huraian(&self) -> &str { "Bilang dan lapor bilangan perkataan" }

    fn proses(&self, input: &str) -> Result<String, String> {
        let bil = input.split_whitespace().count();
        Ok(format!("[{} perkataan] {}", bil, input))
    }
}

impl Pemalam for PemalamBalikTeks {
    fn nama(&self)    -> &str { "BalikTeks" }
    fn versi(&self)   -> &str { "1.0" }
    fn huraian(&self) -> &str { "Balikkan setiap perkataan" }

    fn proses(&self, input: &str) -> Result<String, String> {
        let hasil = input.split_whitespace()
            .map(|w| w.chars().rev().collect::<String>())
            .collect::<Vec<_>>()
            .join(" ");
        Ok(hasil)
    }
}

impl Pemalam for PemalamSensorKata {
    fn nama(&self)    -> &str { "SensorKata" }
    fn versi(&self)   -> &str { "2.0" }
    fn huraian(&self) -> &str { "Sensor kata-kata yang tidak sesuai" }
    fn prioriti(&self) -> u8  { 8 }  // prioriti tinggi!

    fn proses(&self, input: &str) -> Result<String, String> {
        let mut output = input.to_string();
        for kata in &self.kata_larangan {
            let ganti = "*".repeat(kata.len());
            output = output.replace(kata.as_str(), &ganti);
        }
        Ok(output)
    }
}

impl Pemalam for PemalamHadPanjang {
    fn nama(&self)    -> &str { "HadPanjang" }
    fn versi(&self)   -> &str { "1.0" }
    fn huraian(&self) -> &str { "Had panjang teks" }
    fn prioriti(&self) -> u8  { 9 }  // proses awal!

    fn boleh_proses(&self, input: &str) -> bool {
        input.len() <= self.had
    }

    fn proses(&self, input: &str) -> Result<String, String> {
        if input.len() > self.had {
            Err(format!("Input terlalu panjang ({} > {})", input.len(), self.had))
        } else {
            Ok(input.to_string())
        }
    }
}

// ─── Plugin Registry ──────────────────────────────────────────

struct RegistriPemalam {
    pemalam: Vec<Box<dyn Pemalam>>,
}

impl RegistriPemalam {
    fn baru() -> Self {
        RegistriPemalam { pemalam: vec![] }
    }

    fn daftar(mut self, p: Box<dyn Pemalam>) -> Self {
        println!("  ✔ Daftar pemalam: {} v{}", p.nama(), p.versi());
        self.pemalam.push(p);
        self
    }

    fn senarai(&self) {
        println!("\n{'─'*50}");
        println!("{:^50}", "SENARAI PEMALAM");
        println!("{'─'*50}");
        let mut sorted: Vec<&Box<dyn Pemalam>> = self.pemalam.iter().collect();
        sorted.sort_by_key(|p| std::cmp::Reverse(p.prioriti()));

        for p in &sorted {
            println!("  [{:2}] {:<20} v{:<5} {}", 
                p.prioriti(), p.nama(), p.versi(), p.huraian());
        }
        println!("{'─'*50}");
    }

    fn proses(&self, input: &str) -> Result<String, String> {
        println!("\n📥 Input: \"{}\"", input);
        println!("{'─'*40}");

        // Sort by priority (tinggi dulu)
        let mut urutan: Vec<&Box<dyn Pemalam>> = self.pemalam.iter().collect();
        urutan.sort_by_key(|p| std::cmp::Reverse(p.prioriti()));

        let mut output = input.to_string();

        for p in urutan {
            if !p.boleh_proses(&output) {
                println!("  ⊘ [{}] skip (tidak sesuai)", p.nama());
                continue;
            }

            match p.proses(&output) {
                Ok(hasil) => {
                    println!("  ✔ [{}] → \"{}\"", p.nama(), &hasil[..hasil.len().min(50)]);
                    output = hasil;
                }
                Err(e) => {
                    println!("  ✘ [{}] GAGAL: {}", p.nama(), e);
                    return Err(format!("Pemalam '{}' gagal: {}", p.nama(), e));
                }
            }
        }

        println!("{'─'*40}");
        println!("📤 Output: \"{}\"", output);
        Ok(output)
    }
}

// ─── Main ──────────────────────────────────────────────────────

fn main() {
    println!("=== Plugin System Demo ===\n");

    println!("Mendaftarkan pemalam...");
    let registri = RegistriPemalam::baru()
        .daftar(Box::new(PemalamHadPanjang     { had: 100 }))
        .daftar(Box::new(PemalamSensorKata     {
            kata_larangan: vec!["buruk".into(), "jahat".into()]
        }))
        .daftar(Box::new(PemalamBilangPerkataan))
        .daftar(Box::new(PemalamUbahBesarHuruf))
        .daftar(Box::new(PemalamBalikTeks));

    registri.senarai();

    // Test 1: Input biasa
    println!("\n=== Test 1: Input Biasa ===");
    let _ = registri.proses("hello world dari rust");

    // Test 2: Ada kata larangan
    println!("\n=== Test 2: Kata Larangan ===");
    let _ = registri.proses("ini teks buruk yang jahat");

    // Test 3: Teks terlalu panjang
    println!("\n=== Test 3: Terlalu Panjang ===");
    let panjang = "a".repeat(150);
    let _ = registri.proses(&panjang);

    // Test 4: Bina registri berbeza
    println!("\n=== Test 4: Registri Minimal ===");
    let registri2 = RegistriPemalam::baru()
        .daftar(Box::new(PemalamBalikTeks));
    let _ = registri2.proses("rust adalah hebat");
}
```

---

# 📋 Rujukan Pantas — Advanced Traits Cheat Sheet

## Definisi & Implementasi

```rust
// Define trait
trait NamaTrait {
    fn required(&self) -> String;                    // mesti implement
    fn dengan_default(&self) -> i32 { 0 }            // optional override
    fn guna_lain(&self) -> String { self.required() } // guna method lain
}

// Implement
impl NamaTrait for MyStruct { fn required(&self) -> String { .. } }

// Trait bounds
fn f<T: TraitA + TraitB>(x: T) { .. }
fn f<T>(x: T) where T: TraitA + TraitB { .. }
fn f(x: &impl TraitA) { .. }         // input position
fn f() -> impl TraitA { .. }         // output position
```

## Associated Types vs Generics

```
Associated type:   Satu implementation per type
                   → Iterator, Add, Deref

Generic param:     Banyak implementation per type
                   → From, Into, PartialEq
```

## Trait Objects

```rust
// Static dispatch (compile-time, laju)
fn f<T: Trait>(x: &T)
fn f(x: &impl Trait)

// Dynamic dispatch (runtime, fleksibel)
fn f(x: &dyn Trait)
Box<dyn Trait>
Vec<Box<dyn Trait>>

// Object safety:
// ✔ &self, &mut self methods
// ✔ Return non-Self types
// ✗ Return Self
// ✗ Generic methods
```

## Operator Overloading

```rust
std::ops::Add     → a + b
std::ops::Sub     → a - b
std::ops::Mul     → a * b
std::ops::Div     → a / b
std::ops::Rem     → a % b
std::ops::Neg     → -a
std::ops::Index   → a[i]
std::ops::Deref   → *a
std::ops::Fn/FnMut/FnOnce → a()
```

## Standard Traits Penting

```
Display     → {}        untuk output ke pengguna
Debug       → {:?}      untuk debugging
Clone       → .clone()  explicit deep copy
Copy        → implicit  copy (stack types)
Default     → ::default() nilai lalai
From/Into   → lossless  conversion
TryFrom/TryInto → fallible conversion
Hash        → dalam HashMap/HashSet
PartialEq/Eq → == dan !=
PartialOrd/Ord → <, >, <=, >=
Iterator    → for loop, .map(), .filter()
Deref       → * dan auto-deref
Drop        → destructor
Send        → boleh hantar merentasi thread
Sync        → boleh share reference merentasi thread
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **Builder Pattern** — trait `Builder` dengan associated type `Product`
2. **State Machine** — trait `State` dengan `transition()` return `Box<dyn State>`
3. **Serializer** — trait `Serializable` yang boleh convert ke JSON/TOML/CSV
4. **Command Pattern** — trait `Command` dengan `execute()` dan `undo()`
5. **Visitor Pattern** — trait `Visitor` untuk traverse AST tanpa ubah struct

---

*Advanced Traits in Rust — dari trait asas hingga blanket implementations dan plugin system. Selamat belajar!* 🦀
