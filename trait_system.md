# 🎭 Trait System Rust — Lebih dari Sekadar Interface

> "Trait bukan sekadar interface. Ia adalah kontrak, constraint,
>  kapabiliti, dan asas seluruh generic system Rust."
> Deep dive: trait bounds, blanket impl, auto traits, dan pattern lanjutan.

---

## Trait vs Interface — Perbezaan Fundamental

```
Java/PHP Interface:
  interface Printable {
      void print();  // kontrak sahaja
  }

  class Dokumen implements Printable {
      public void print() { ... }
  }
  // ↑ Hanya polymorphism runtime (vtable)
  // ↑ Tidak boleh implement untuk class yang SUDAH WUJUD
  // ↑ Tidak ada static dispatch
  // ↑ Tidak ada blanket implementations
  // ↑ Tidak ada associated types

Rust Trait — JAUH lebih berkuasa:
  trait Papar {
      fn papar(&self);              // method
      fn format(&self) -> String {  // default implementation!
          format!("Default: {:?}", self as *const _)
      }
  }

  // 1. Implement untuk type LUAR (selagi ikut orphan rule)
  // 2. Blanket impl — impl sekali untuk SEMUA types yang qualify
  // 3. Trait bounds — constraint pada generics
  // 4. Associated types — type yang berkaitan
  // 5. Static dispatch (monomorphization) — TIADA runtime overhead!
  // 6. Dynamic dispatch (dyn Trait) — polymorphism bila perlu
  // 7. Auto traits (Send, Sync) — property yang auto-derived
  // 8. Marker traits — trait tanpa method
```

---

## Peta Pembelajaran

```
Bab 1  → Trait Asas & Default Methods
Bab 2  → Associated Types vs Generic Parameters
Bab 3  → Trait Bounds — T: Trait
Bab 4  → Blanket Implementations
Bab 5  → Static Dispatch vs Dynamic Dispatch
Bab 6  → Object Safety
Bab 7  → Auto Traits — Send & Sync
Bab 8  → Marker Traits
Bab 9  → Advanced Trait Patterns
Bab 10 → Mini Project: Type-Safe Builder & Plugin System
```

---

# BAB 1: Trait Asas & Default Methods 🧱

## Lebih dari Interface Biasa

```rust
use std::fmt;

// ── Trait dengan default method ───────────────────────────────
trait Boleh Papar {
    // Method yang MESTI diimplementasi
    fn nama(&self) -> &str;

    // Method dengan DEFAULT implementation!
    // Implementor BOLEH override, tapi tidak PERLU
    fn salam(&self) -> String {
        format!("Halo, saya {}!", self.nama())
    }

    fn ringkasan(&self) -> String {
        format!("[{}]", self.nama())
    }
}

struct Pekerja { nama: String, id: u32 }
struct Robot   { kod:  String }

impl BolehPapar for Pekerja {
    fn nama(&self) -> &str { &self.nama }
    // Guna default salam() dan ringkasan()
}

impl BolehPapar for Robot {
    fn nama(&self) -> &str { &self.kod }
    // Override ringkasan sahaja
    fn ringkasan(&self) -> String {
        format!("[ROBOT:{}]", self.kod)
    }
}

fn main() {
    let p = Pekerja { nama: "Ali".into(), id: 1 };
    let r = Robot   { kod: "R2D2".into() };

    println!("{}", p.salam());     // Halo, saya Ali!
    println!("{}", r.salam());     // Halo, saya R2D2!
    println!("{}", p.ringkasan()); // [Ali]     ← default
    println!("{}", r.ringkasan()); // [ROBOT:R2D2] ← override
}
```

---

## Associated Functions & Constants dalam Trait

```rust
trait Bina {
    // Associated constant
    const VERSI: &'static str;
    const MAX_SAIZ: usize = 1000; // dengan default!

    // Constructor (associated function — tiada self)
    fn baru() -> Self;
    fn dengan_nama(nama: &str) -> Self;

    // Instance method
    fn nama(&self) -> &str;

    // Consuming method
    fn ke_string(self) -> String where Self: Sized;
}

struct Rekod {
    nama: String,
    data: Vec<u8>,
}

impl Bina for Rekod {
    const VERSI: &'static str = "1.0";
    const MAX_SAIZ: usize = 500; // override default

    fn baru() -> Self {
        Rekod { nama: "tiada nama".into(), data: Vec::new() }
    }

    fn dengan_nama(nama: &str) -> Self {
        Rekod { nama: nama.into(), data: Vec::new() }
    }

    fn nama(&self) -> &str { &self.nama }

    fn ke_string(self) -> String {
        format!("Rekod({}, {} bytes)", self.nama, self.data.len())
    }
}

fn main() {
    // Guna trait methods
    let r1 = Rekod::baru();
    let r2 = Rekod::dengan_nama("Laporan");

    println!("Versi:   {}", Rekod::VERSI);     // 1.0
    println!("Max:     {}", Rekod::MAX_SAIZ);  // 500 (override)
    println!("Nama:    {}", r2.nama());
    println!("String:  {}", r2.ke_string());
}
```

---

## 🧠 Brain Teaser #1

Apakah perbezaan antara trait method default yang guna `self` vs yang guna `Self`?

```rust
trait Contoh: Sized {
    fn a(&self) -> String;        // &self — boleh guna untuk dyn Trait
    fn b(self) -> String;         // self — consume, object-unsafe!
    fn c() -> Self;               // Self — constructor, object-unsafe!
    fn d(&self) -> Box<Self>;     // Box<Self> — object-unsafe!
}
```

<details>
<summary>👀 Jawapan</summary>

```
&self   → borrow, saiz tidak diketahui pada compile, SELAMAT untuk dyn Trait
self    → consume, perlu tahu saiz konkrit, TIDAK selamat untuk dyn Trait
Self    → jenis konkrit (caller type), saiz perlu diketahui, TIDAK selamat
&Self   → reference ke jenis konkrit, TIDAK selamat untuk dyn Trait
Box<Self> → owned, saiz konkrit perlu, TIDAK selamat

OBJECT SAFETY — trait boleh jadi dyn Trait HANYA jika:
  ✔ Semua methods ada receiver (&self, &mut self, Box<Self>)
  ✔ Tiada associated functions (fn baru() -> Self)
  ✔ Tiada type parameters pada method
  ✔ Tiada `where Self: Sized` pada methods yang dipanggil

Cara buat method dengan Self tetapi masih object-safe:
  fn b(self) -> String where Self: Sized; // ← add where Self: Sized
  // Method ini di-exclude dari vtable secara automatik!
  // Boleh dipanggil pada concrete type, tapi bukan dyn Trait
```
</details>

---

# BAB 2: Associated Types vs Generic Parameters 🔀

## Bila Guna Mana?

```rust
// ── Associated Type ─────────────────────────────────────────
// Setiap implementor ada SATU jenis output sahaja
// Lebih ergonomic — tidak perlu specify pada setiap guna

trait Iterator {
    type Item;    // ← associated type
    fn next(&mut self) -> Option<Self::Item>;
}

// Bila guna: trait mempunyai SATU jenis yang berkaitan
// Contoh: Iterator ada satu Item type sahaja

struct Counter { nilai: u32 }

impl Iterator for Counter {
    type Item = u32;    // ditetapkan di sini
    fn next(&mut self) -> Option<u32> {
        if self.nilai < 5 {
            self.nilai += 1;
            Some(self.nilai)
        } else {
            None
        }
    }
}

// ── Generic Parameter ────────────────────────────────────────
// Boleh ada BANYAK implementasi untuk type yang sama
// Perlu specify setiap kali guna

trait Tukar<T> {    // ← generic parameter
    fn tukar(&self) -> T;
}

// SATU struct boleh implement untuk BANYAK T!
struct Suhu(f64);

impl Tukar<f64> for Suhu {
    fn tukar(&self) -> f64 { self.0 }
}

impl Tukar<String> for Suhu {
    fn tukar(&self) -> String { format!("{:.1}°C", self.0) }
}

impl Tukar<u32> for Suhu {
    fn tukar(&self) -> u32 { self.0 as u32 }
}

fn main() {
    let s = Suhu(36.6);

    // Dengan associated type — jelas tanpa annotation
    let mut c = Counter { nilai: 0 };
    while let Some(v) = c.next() { print!("{} ", v); }
    println!();

    // Dengan generic — perlu annotation
    let f: f64    = s.tukar();
    let st: String = s.tukar();
    let n: u32    = s.tukar();
    println!("{} {} {}", f, st, n);
}
```

---

## Associated Types Lanjutan

```rust
use std::ops::Add;

// ── Add trait — contoh dari std ──────────────────────────────
// trait Add<Rhs = Self> {
//     type Output;
//     fn add(self, rhs: Rhs) -> Self::Output;
// }

#[derive(Debug, Clone, Copy)]
struct Vektor2D { x: f64, y: f64 }

impl Add for Vektor2D {
    type Output = Vektor2D;
    fn add(self, lain: Vektor2D) -> Vektor2D {
        Vektor2D { x: self.x + lain.x, y: self.y + lain.y }
    }
}

impl Add<f64> for Vektor2D {    // Rhs = f64 (bukan Self!)
    type Output = Vektor2D;
    fn add(self, skalar: f64) -> Vektor2D {
        Vektor2D { x: self.x + skalar, y: self.y + skalar }
    }
}

fn main() {
    let v1 = Vektor2D { x: 1.0, y: 2.0 };
    let v2 = Vektor2D { x: 3.0, y: 4.0 };

    let v3 = v1 + v2;           // Vektor + Vektor
    let v4 = v1 + 10.0;         // Vektor + f64

    println!("{:?}", v3); // Vektor2D { x: 4.0, y: 6.0 }
    println!("{:?}", v4); // Vektor2D { x: 11.0, y: 12.0 }
}
```

---

# BAB 3: Trait Bounds — T: Trait 🔗

## Pelbagai Cara Tulis Bounds

```rust
use std::fmt::{Debug, Display};

// ── Cara 1: Inline bounds ─────────────────────────────────────
fn cetak_debug<T: Debug>(x: &T) {
    println!("{:?}", x);
}

// ── Cara 2: Multiple bounds ───────────────────────────────────
fn cetak_semua<T: Debug + Display + Clone>(x: &T) {
    println!("Display: {}", x);
    println!("Debug:   {:?}", x);
    let _ = x.clone();
}

// ── Cara 3: Where clause (lebih readable untuk complex bounds) ─
fn proses<T, U>(t: &T, u: &U) -> String
where
    T: Debug + Display + Clone,
    U: Debug + PartialOrd + Default,
{
    format!("{} dan {:?}", t, u)
}

// ── Cara 4: impl Trait (argument position) ───────────────────
fn cetak_display(x: impl Display) {
    println!("{}", x);
}
// Sama seperti fn cetak_display<T: Display>(x: T)
// Tapi lebih ringkas, tiada nama type

// ── Cara 5: impl Trait (return position) ─────────────────────
fn buat_iterator() -> impl Iterator<Item = i32> {
    vec![1, 2, 3].into_iter()
    // Concrete type disembunyikan! Caller hanya nampak Iterator<Item=i32>
}

// ── Cara 6: Bound pada struct ─────────────────────────────────
struct Pembalut<T: Display> {
    nilai: T,
}

impl<T: Display + Debug> Pembalut<T> {
    fn papar(&self) {
        println!("Display: {}", self.nilai);
        println!("Debug:   {:?}", self.nilai);
    }
}

// ── Cara 7: Bound pada method sahaja (bukan struct) ───────────
struct Bekas<T> {
    nilai: T,
    // T tidak perlu implement apa-apa di sini
}

impl<T> Bekas<T> {
    fn baru(nilai: T) -> Self { Bekas { nilai } }

    // Hanya method ini yang perlu T: Display
    fn papar(&self) where T: Display {
        println!("{}", self.nilai);
    }

    // Hanya method ini yang perlu T: Clone
    fn salin(&self) -> T where T: Clone {
        self.nilai.clone()
    }
}

fn main() {
    let b = Bekas::baru(42i32);
    b.papar();              // 42 (i32: Display ✔)

    let b2 = Bekas::baru(vec![1, 2, 3]);
    // b2.papar();          // ERROR: Vec tidak implement Display
    let _ = b2.salin();     // OK: Vec: Clone ✔
}
```

---

## Higher-Ranked Trait Bounds (HRTB)

```rust
// HRTB — for<'a> — "untuk SEMUA lifetime 'a"
// Diperlukan bila function yang diterima perlu bekerja
// dengan MANA-MANA lifetime, bukan lifetime yang spesifik

// Contoh: fungsi yang terima closure yang borrow dari parameter
fn guna_ref<F>(fungsi: F)
where
    F: for<'a> Fn(&'a str) -> usize  // ← HRTB!
{
    let s = String::from("hello");
    let n = fungsi(&s);
    println!("Panjang: {}", n);
}

fn main() {
    guna_ref(|s| s.len());  // bekerja untuk sebarang &str

    // Tanpa HRTB, perlu specify lifetime konkrit:
    // fn guna_ref<'b, F: Fn(&'b str) -> usize>(fungsi: F, s: &'b str)
    // Ini terlalu restrictive — fungsi terikat pada 'b sahaja
}
```

---

# BAB 4: Blanket Implementations 🌐

## Implement Sekali untuk Semua

```rust
use std::fmt;

// ── Blanket impl dari std — contoh sebenar ────────────────────

// 1. ToString untuk semua type yang ada Display
//    impl<T: fmt::Display> ToString for T { ... }
//    Kesan: 42u32.to_string(), true.to_string(), dll

// 2. Into dari From
//    impl<T, U: From<T>> Into<T> for U { ... }
//    Kesan: implement From<A> for B → dapat Into<B> for A percuma!

// 3. IntoIterator untuk Vec, array, dll
//    Kesan: for x in vec → guna IntoIterator

// ── Tulis blanket impl sendiri ───────────────────────────────

trait Ringkas {
    fn ringkas(&self, panjang: usize) -> String;
}

// Blanket impl: Semua T yang ada Display dapat Ringkas!
impl<T: fmt::Display> Ringkas for T {
    fn ringkas(&self, panjang: usize) -> String {
        let penuh = self.to_string();
        if penuh.len() <= panjang {
            penuh
        } else {
            format!("{}...", &penuh[..panjang.min(penuh.len())])
        }
    }
}

// Sekarang SEMUA type yang ada Display ada .ringkas()!
fn main() {
    println!("{}", 42i32.ringkas(5));                    // "42"
    println!("{}", 3.14f64.ringkas(4));                  // "3.14"
    println!("{}", "Teks yang panjang sekali".ringkas(10)); // "Teks yang..."
    println!("{}", true.ringkas(3));                     // "tru..."
}
```

---

## Blanket Impl yang Lebih Kompleks

```rust
use std::fmt;

// ── Supertrait ────────────────────────────────────────────────
// Trait yang memerlukan trait lain (inheritance-like)

trait Entiti: fmt::Debug + fmt::Display + Clone {
    fn id(&self) -> u64;
    fn nama(&self) -> &str;

    // Default method yang bergantung pada supertraits
    fn maklumat(&self) -> String {
        format!("ID: {}, Nama: {}, Debug: {:?}", self.id(), self.nama(), self)
    }
}

// ── Blanket impl untuk Entiti ─────────────────────────────────
// Mana-mana Entiti boleh di-print dengan format khas
trait EntitíCetak: Entiti {
    fn cetak_kad(&self) {
        println!("┌{'─'*30}┐");
        println!("│ ID:   {:>24} │", self.id());
        println!("│ Nama: {:>24} │", self.nama());
        println!("└{'─'*30}┘");
    }
}

// Blanket: SEMUA Entiti dapat EntítiCetak percuma!
impl<T: Entiti> EntitíCetak for T {}

// ── Contoh implementasi ───────────────────────────────────────
#[derive(Debug, Clone)]
struct PekerjaData {
    id:   u64,
    nama: String,
}

impl fmt::Display for PekerjaData {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Pekerja({}: {})", self.id, self.nama)
    }
}

impl Entiti for PekerjaData {
    fn id(&self) -> u64 { self.id }
    fn nama(&self) -> &str { &self.nama }
}

// PekerjaData kini SECARA AUTOMATIK dapat EntitíCetak!

fn main() {
    let p = PekerjaData { id: 1001, nama: "Ali Ahmad".into() };
    p.cetak_kad();   // auto dari blanket impl!
    println!("{}", p.maklumat());
}
```

---

## 🧠 Brain Teaser #2

Kenapa kod ini tidak compile, dan bagaimana betulkan?

```rust
trait A {
    fn kaedah(&self) -> String;
}

trait B {
    fn kaedah(&self) -> String;
}

struct Foo;

impl A for Foo {
    fn kaedah(&self) -> String { "A".into() }
}

impl B for Foo {
    fn kaedah(&self) -> String { "B".into() }
}

fn main() {
    let f = Foo;
    println!("{}", f.kaedah()); // ← ERROR! Ambiguous!
}
```

<details>
<summary>👀 Jawapan</summary>

```rust
// ERROR: multiple applicable items in scope
// Rust tidak tahu nak guna A::kaedah atau B::kaedah!

// PENYELESAIAN: Fully Qualified Syntax (FQS)

fn main() {
    let f = Foo;

    // Cara 1: Fully Qualified Syntax
    println!("{}", <Foo as A>::kaedah(&f));  // "A"
    println!("{}", <Foo as B>::kaedah(&f));  // "B"

    // Cara 2: Cast ke trait object dulu
    let a_ref: &dyn A = &f;
    println!("{}", a_ref.kaedah());  // "A"

    let b_ref: &dyn B = &f;
    println!("{}", b_ref.kaedah());  // "B"
}

// Fully Qualified Syntax juga berguna untuk:
// 1. Panggil method trait yang tersembunyi oleh method struct sendiri
// 2. Panggil associated functions dengan ambiguity
// 3. Explicit tentukan trait mana yang hendak digunakan

struct Counter { nilai: u32 }

impl Counter {
    fn method(&self) -> String { "struct method".into() }
}

trait MyTrait {
    fn method(&self) -> String;
}

impl MyTrait for Counter {
    fn method(&self) -> String { "trait method".into() }
}

fn demo() {
    let c = Counter { nilai: 0 };
    println!("{}", c.method());                           // struct method (default)
    println!("{}", <Counter as MyTrait>::method(&c));    // trait method (explicit)
}
```
</details>

---

# BAB 5: Static Dispatch vs Dynamic Dispatch ⚡

## Monomorphization vs vtable

```rust
use std::fmt::Display;

// ── Static Dispatch (impl Trait / generics) ───────────────────
// Compile-time: Rust generate kod berasingan untuk setiap type
// TIADA runtime overhead — paling laju!

fn cetak_static<T: Display>(x: &T) {
    println!("{}", x);
}
// Compiler generate:
//   fn cetak_static_i32(x: &i32)    { println!("{}", x); }
//   fn cetak_static_f64(x: &f64)    { println!("{}", x); }
//   fn cetak_static_str(x: &&str)   { println!("{}", x); }
// (monomorphization)

// ── Dynamic Dispatch (dyn Trait) ──────────────────────────────
// Runtime: guna vtable (pointer ke implementation)
// Ada overhead (pointer indirection) tapi lebih flexible

fn cetak_dynamic(x: &dyn Display) {
    println!("{}", x);
}
// Satu fungsi sahaja — guna vtable pointer untuk dispatch

// ── Perbandingan ──────────────────────────────────────────────
fn main() {
    let n = 42i32;
    let f = 3.14f64;
    let s = "hello";

    // Static — resolve pada compile time
    cetak_static(&n);
    cetak_static(&f);
    cetak_static(&s);

    // Dynamic — resolve pada runtime
    cetak_dynamic(&n);
    cetak_dynamic(&f);
    cetak_dynamic(&s);

    // PALING KETARA dalam koleksi!

    // ✔ Static — Vec hanya satu type
    let nombor: Vec<i32> = vec![1, 2, 3];
    for n in &nombor { cetak_static(n); }

    // ✔ Dynamic — Vec dengan pelbagai type (perlu Box atau &)
    let campur: Vec<Box<dyn Display>> = vec![
        Box::new(1i32),
        Box::new(3.14f64),
        Box::new("hello"),
    ];
    for item in &campur { cetak_dynamic(item.as_ref()); }
}
```

---

## Return impl Trait vs dyn Trait

```rust
use std::fmt::Display;

// ── impl Trait dalam return position ─────────────────────────
// ✔ Zero overhead — concrete type diketahui compile time
// ✗ Hanya SATU concrete type boleh direturn!

fn buat_genap(n: u32) -> impl Iterator<Item = u32> {
    (0..n).filter(|x| x % 2 == 0)
    // Concrete type: Filter<Range<u32>, {closure}>
}

// ── dyn Trait dalam return position ──────────────────────────
// ✔ Boleh return type yang BERBEZA
// ✗ Ada overhead (heap allocation dengan Box)

fn buat_iterator(pakai_genap: bool) -> Box<dyn Iterator<Item = u32>> {
    if pakai_genap {
        Box::new((0..10).filter(|x| x % 2 == 0))
    } else {
        Box::new((0..10).filter(|x| x % 2 != 0))
        // Type berbeza! Perlu dyn
    }
}

// ── Bila guna mana? ───────────────────────────────────────────
// impl Trait → default choice, bila type sama
// dyn Trait  → bila perlu runtime polymorphism (plugin, callbacks)

fn main() {
    // impl Trait
    for n in buat_genap(10) { print!("{} ", n); }
    println!();

    // dyn Trait
    for n in buat_iterator(true)  { print!("{} ", n); }
    println!();
    for n in buat_iterator(false) { print!("{} ", n); }
    println!();
}
```

---

# BAB 6: Object Safety 🔐

## Bila Trait Boleh Jadi dyn Trait

```rust
// Object-safe trait (boleh jadi dyn Trait):
trait ObjekSelamat {
    fn kaedah(&self) -> String;
    fn kaedah_mut(&mut self) -> String;
    fn kaedah_box(self: Box<Self>) -> String; // Box<Self> OK!

    // Method dengan where Self: Sized di-exclude dari vtable
    fn guna_hanya_konkrit(&self) where Self: Sized {
        // Boleh ada di sini, tapi tidak accessible dari dyn Trait
    }
}

// TIDAK object-safe — ada Self dalam return type
trait TidakSelamat {
    fn klon(&self) -> Self;          // ← Self sebagai return type
    fn buat() -> Self;               // ← associated function
    fn bandingkan<T>(&self, lain: T); // ← generic method
}

// Cara buat Clone-like yang object-safe
trait KlonSelamat {
    fn klon_kotak(&self) -> Box<dyn KlonSelamat>;  // OK!
}

// Contoh guna
fn guna_trait_objek(x: &dyn ObjekSelamat) {
    println!("{}", x.kaedah());
    // x.guna_hanya_konkrit(); // ERROR! Tidak accessible dari dyn
}
```

---

# BAB 7: Auto Traits — Send & Sync 🚀

## Property yang Dicipta Secara Automatik

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::cell::RefCell;
use std::rc::Rc;

// ── Send — selamat untuk HANTAR ke thread lain ───────────────
// Auto-derived kalau semua field adalah Send

struct DataSelamat {
    nama:   String,    // String: Send ✔
    kiraan: u32,       // u32: Send ✔
}
// DataSelamat auto: Send ✔

struct DataTidakSelamat {
    ptr: *mut i32,     // raw pointer: TIDAK Send ✗
}
// DataTidakSelamat auto: !Send

// ── Sync — selamat untuk SHARE antara thread (via &T) ────────
// T: Sync ⟺ &T: Send

// Arc<T> adalah Send + Sync kalau T: Send + Sync
// Rc<T> adalah BUKAN Send atau Sync (tiada atomic reference counting)
// RefCell<T> adalah Send tapi BUKAN Sync
// Mutex<T> adalah Send + Sync kalau T: Send

fn demo_send_sync() {
    // ✔ String: Send — boleh hantar ke thread
    let s = String::from("hello");
    thread::spawn(move || println!("{}", s)).join().unwrap();

    // ✔ Arc<T>: Send + Sync — boleh share
    let shared = Arc::new(Mutex::new(0i32));
    let klon = Arc::clone(&shared);
    thread::spawn(move || {
        *klon.lock().unwrap() += 1;
    }).join().unwrap();

    // ❌ Rc<T>: BUKAN Send — compile error!
    // let rc = Rc::new(42);
    // thread::spawn(move || println!("{}", rc)); // ERROR!

    // ❌ RefCell<T>: BUKAN Sync — tidak boleh share via &
    // let rf = RefCell::new(42);
    // let rf_ref = &rf;
    // thread::spawn(move || rf_ref.borrow()); // ERROR!
}

// ── Implement manual (UNSAFE!) ────────────────────────────────
// Hanya bila anda TAHU ia selamat tapi compiler tidak tahu

struct CustomPtr(*mut i32);

// UNSAFE: kita jamin tidak ada data race
unsafe impl Send for CustomPtr {}
unsafe impl Sync for CustomPtr {}

// ── Negative impls ────────────────────────────────────────────
// Explicitly tandakan sebagai BUKAN Send/Sync
use std::marker::PhantomData;

struct TidakBolehHantar {
    _tidak_send: PhantomData<*mut u8>, // raw pointer → !Send, !Sync
}
// TidakBolehHantar auto !Send, !Sync kerana *mut u8
```

---

## PhantomData — Ghost Type Parameters

```rust
use std::marker::PhantomData;

// PhantomData<T> — "pura-pura" ada T tapi tidak simpan apa-apa
// Guna untuk:
// 1. Bawa lifetime
// 2. Bawa variance
// 3. Elak Send/Sync (seperti di atas)
// 4. Type-state pattern

// ── Type-state pattern dengan PhantomData ────────────────────
struct Dikunci;
struct Dibuka;

struct Kunci<Keadaan> {
    kunci_rahsia: u32,
    _keadaan:     PhantomData<Keadaan>,
}

impl Kunci<Dikunci> {
    fn baru(kunci: u32) -> Self {
        Kunci { kunci_rahsia: kunci, _keadaan: PhantomData }
    }

    fn buka(self, kata_laluan: u32) -> Option<Kunci<Dibuka>> {
        if self.kunci_rahsia == kata_laluan {
            Some(Kunci { kunci_rahsia: self.kunci_rahsia, _keadaan: PhantomData })
        } else {
            None
        }
    }
}

impl Kunci<Dibuka> {
    fn guna(&self) -> String {
        "Kunci sedang digunakan!".into()
    }

    fn kunci_semula(self) -> Kunci<Dikunci> {
        Kunci { kunci_rahsia: self.kunci_rahsia, _keadaan: PhantomData }
    }
}

fn main() {
    let kunci = Kunci::<Dikunci>::baru(1234);
    // kunci.guna(); // ERROR! Kunci masih dikunci!

    if let Some(dibuka) = kunci.buka(1234) {
        println!("{}", dibuka.guna()); // OK!
        let _semula = dibuka.kunci_semula();
        // _semula.guna(); // ERROR! Dikunci semula!
    }
}
```

---

# BAB 8: Marker Traits 🏷️

## Trait Tanpa Method

```rust
// Marker trait = trait tanpa method
// Digunakan sebagai "label" atau "constraint"

// ── Dari std ──────────────────────────────────────────────────
// Copy   → boleh copy bitwise
// Send   → boleh hantar ke thread lain
// Sync   → boleh share antara thread
// Sized  → saiz diketahui pada compile time
// Unpin  → boleh dipindah selepas Pin

// ── Tulis marker trait sendiri ───────────────────────────────
// Untuk type-level tagging

trait DataSensitif {}
trait DataAwam {}

struct IC(String);
struct Nama(String);

impl DataSensitif for IC {}    // IC adalah data sensitif
impl DataAwam for Nama {}      // Nama adalah data awam

// Fungsi yang hanya terima data awam
fn log_data<T: DataAwam + std::fmt::Debug>(data: &T) {
    println!("Log: {:?}", data);
}

// Fungsi yang hanya terima data sensitif — perlu extra care
fn proses_sensitif<T: DataSensitif>(data: &T) {
    println!("Proses data sensitif...");
    // Tidak log! Tidak print ke console!
}

// ── Sealed trait menggunakan marker ──────────────────────────
mod sealed {
    pub trait Tertutup {}
    impl Tertutup for i32  {}
    impl Tertutup for f64  {}
    impl Tertutup for bool {}
    // Orang luar tidak boleh implement Tertutup!
}

trait BolehKira: sealed::Tertutup {
    fn kira(&self) -> f64;
}

impl BolehKira for i32  { fn kira(&self) -> f64 { *self as f64 } }
impl BolehKira for f64  { fn kira(&self) -> f64 { *self } }
impl BolehKira for bool { fn kira(&self) -> f64 { if *self { 1.0 } else { 0.0 } } }

// Pengguna library tidak boleh implement BolehKira untuk type sendiri!
```

---

# BAB 9: Advanced Trait Patterns 🎯

## Trait Composition

```rust
use std::fmt;

// ── Supertrait — inheritance-like ────────────────────────────
trait Haiwan: fmt::Display + fmt::Debug {
    fn nama(&self) -> &str;
    fn bunyi(&self) -> &str;
    fn bergerak(&self) -> String {
        format!("{} bergerak", self.nama())
    }
}

trait HaiwanPeliharaan: Haiwan {
    fn pemilik(&self) -> &str;
    fn penuh_kasih(&self) -> String {
        format!("{} milik {} membuat bunyi {}", self.nama(), self.pemilik(), self.bunyi())
    }
}

// ── Conditional implementation ────────────────────────────────
use std::ops::Add;

#[derive(Debug, Clone, Copy)]
struct Kotak<T> { lebar: T, tinggi: T }

// Luas hanya boleh dikira kalau T boleh didarab dan ditambah
impl<T: Add<Output = T> + std::ops::Mul<Output = T> + Copy> Kotak<T> {
    fn luas(&self) -> T {
        self.lebar * self.tinggi
    }
}

// Papar hanya kalau T: Display
impl<T: fmt::Display> fmt::Display for Kotak<T> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Kotak({}×{})", self.lebar, self.tinggi)
    }
}

fn main() {
    let k = Kotak { lebar: 5, tinggi: 3 };
    println!("Luas: {}", k.luas());  // 15
    println!("{}", k);               // Kotak(5×3)

    let kf = Kotak { lebar: 2.5f64, tinggi: 4.0 };
    println!("Luas: {}", kf.luas()); // 10.0
}
```

---

## GAT — Generic Associated Types (Rust 1.65+)

```rust
// Generic Associated Types — associated types yang boleh ada generics!

trait Koleksi {
    // GAT: Iter<'a> adalah associated type dengan lifetime parameter
    type Iter<'a> where Self: 'a;

    fn iter(&self) -> Self::Iter<'_>;
    fn masuk(&mut self, nilai: i32);
}

struct Timbunan(Vec<i32>);

impl Koleksi for Timbunan {
    type Iter<'a> = std::slice::Iter<'a, i32>;

    fn iter(&self) -> std::slice::Iter<'_, i32> {
        self.0.iter()
    }

    fn masuk(&mut self, nilai: i32) {
        self.0.push(nilai);
    }
}

fn main() {
    let mut t = Timbunan(vec![]);
    t.masuk(1);
    t.masuk(2);
    t.masuk(3);

    for n in t.iter() {
        print!("{} ", n);
    }
    println!();
}
```

---

## Impl Trait dalam Trait (RPITIT — Rust 1.75+)

```rust
// Return Position impl Trait in Trait
// Sebelum ini, tidak boleh guna impl Trait dalam trait!

trait Pemproses {
    // Sekarang boleh!
    fn proses(&self) -> impl Iterator<Item = i32>;
    fn transformasi(&self, n: i32) -> impl fmt::Display;
}

struct PembahagianDua;

impl Pemproses for PembahagianDua {
    fn proses(&self) -> impl Iterator<Item = i32> {
        (1..=10).filter(|n| n % 2 == 0)
    }

    fn transformasi(&self, n: i32) -> impl fmt::Display {
        n * n  // return i32 yang impl Display
    }
}

fn main() {
    let p = PembahagianDua;
    for n in p.proses() { print!("{} ", n); }
    println!();
    println!("{}", p.transformasi(5));
}
```

---

# BAB 10: Mini Project — Plugin System & Type-Safe Builder 🏗️

```rust
use std::collections::HashMap;

// ─── BAHAGIAN 1: Plugin System (dyn Trait) ───────────────────

trait Plugin: std::fmt::Debug {
    fn nama(&self) -> &str;
    fn versi(&self) -> &str;
    fn laksana(&self, input: &str) -> String;
    fn boleh_guna(&self, _konteks: &HashMap<String, String>) -> bool {
        true // default: sentiasa boleh guna
    }
}

// Plugins yang berbeza
#[derive(Debug)]
struct PluginBesarkan;

impl Plugin for PluginBesarkan {
    fn nama(&self) -> &str    { "Besarkan" }
    fn versi(&self) -> &str   { "1.0" }
    fn laksana(&self, input: &str) -> String { input.to_uppercase() }
}

#[derive(Debug)]
struct PluginBalik;

impl Plugin for PluginBalik {
    fn nama(&self) -> &str    { "Balik" }
    fn versi(&self) -> &str   { "1.0" }
    fn laksana(&self, input: &str) -> String {
        input.chars().rev().collect()
    }
}

#[derive(Debug)]
struct PluginHitung { huruf: char }

impl Plugin for PluginHitung {
    fn nama(&self) -> &str    { "Hitung" }
    fn versi(&self) -> &str   { "1.0" }
    fn laksana(&self, input: &str) -> String {
        let kiraan = input.chars().filter(|&c| c == self.huruf).count();
        format!("{}: {} kali dalam '{}'", self.huruf, kiraan, input)
    }
}

struct RegistriPlugin {
    plugins: Vec<Box<dyn Plugin>>,
}

impl RegistriPlugin {
    fn baru() -> Self { RegistriPlugin { plugins: Vec::new() } }

    fn daftar(&mut self, plugin: Box<dyn Plugin>) {
        println!("Plugin '{}' v{} didaftarkan", plugin.nama(), plugin.versi());
        self.plugins.push(plugin);
    }

    fn laksana_semua(&self, input: &str) -> Vec<(String, String)> {
        self.plugins.iter()
            .map(|p| (p.nama().to_string(), p.laksana(input)))
            .collect()
    }

    fn cari(&self, nama: &str) -> Option<&dyn Plugin> {
        self.plugins.iter()
            .find(|p| p.nama() == nama)
            .map(|p| p.as_ref())
    }
}

// ─── BAHAGIAN 2: Type-Safe Builder (impl Trait + PhantomData) ─

use std::marker::PhantomData;

// State types
struct BelumAda;
struct SudahAda;

struct PembinaPesan<Tajuk, Kandungan, Penerima> {
    tajuk:     Option<String>,
    kandungan: Option<String>,
    penerima:  Option<String>,
    _t: PhantomData<(Tajuk, Kandungan, Penerima)>,
}

impl PembinaPesan<BelumAda, BelumAda, BelumAda> {
    fn baru() -> Self {
        PembinaPesan {
            tajuk: None, kandungan: None, penerima: None,
            _t: PhantomData,
        }
    }
}

impl<K, P> PembinaPesan<BelumAda, K, P> {
    fn tajuk(self, tajuk: &str)
        -> PembinaPesan<SudahAda, K, P>
    {
        PembinaPesan {
            tajuk:     Some(tajuk.into()),
            kandungan: self.kandungan,
            penerima:  self.penerima,
            _t:        PhantomData,
        }
    }
}

impl<T, P> PembinaPesan<T, BelumAda, P> {
    fn kandungan(self, kandungan: &str)
        -> PembinaPesan<T, SudahAda, P>
    {
        PembinaPesan {
            tajuk:     self.tajuk,
            kandungan: Some(kandungan.into()),
            penerima:  self.penerima,
            _t:        PhantomData,
        }
    }
}

impl<T, K> PembinaPesan<T, K, BelumAda> {
    fn penerima(self, penerima: &str)
        -> PembinaPesan<T, K, SudahAda>
    {
        PembinaPesan {
            tajuk:     self.tajuk,
            kandungan: self.kandungan,
            penerima:  Some(penerima.into()),
            _t:        PhantomData,
        }
    }
}

// Hanya boleh build bila SEMUA field sudah diset!
impl PembinaPesan<SudahAda, SudahAda, SudahAda> {
    fn bina(self) -> Pesan {
        Pesan {
            tajuk:     self.tajuk.unwrap(),
            kandungan: self.kandungan.unwrap(),
            penerima:  self.penerima.unwrap(),
        }
    }
}

#[derive(Debug)]
struct Pesan {
    tajuk:     String,
    kandungan: String,
    penerima:  String,
}

// ─── Demo ─────────────────────────────────────────────────────

fn main() {
    println!("=== Plugin System ===\n");

    let mut reg = RegistriPlugin::baru();
    reg.daftar(Box::new(PluginBesarkan));
    reg.daftar(Box::new(PluginBalik));
    reg.daftar(Box::new(PluginHitung { huruf: 'a' }));

    println!("\nLaksana semua plugin pada 'halo dunia':");
    for (nama, hasil) in reg.laksana_semua("halo dunia") {
        println!("  {:10} → {}", nama, hasil);
    }

    if let Some(plugin) = reg.cari("Balik") {
        println!("\nPlugin Balik: {}", plugin.laksana("Rust"));
    }

    println!("\n=== Type-Safe Builder ===\n");

    // Mesti set semua field — compiler enforce!
    let pesan = PembinaPesan::baru()
        .tajuk("Laporan Bulanan")
        .kandungan("Kehadiran 95% bulan ini")
        .penerima("ketua@kada.gov.my")
        .bina();         // Hanya boleh panggil bila semua set!

    println!("{:#?}", pesan);

    // Ini tidak akan compile!
    // let tidak_lengkap = PembinaPesan::baru()
    //     .tajuk("Tanpa kandungan")
    //     .bina(); // ERROR: tidak ada method bina() untuk keadaan ini!
}
```

---

# 📋 Rujukan Pantas — Trait System Cheat Sheet

## Definisi & Syntax

```rust
// Basic trait
trait NamaTrait {
    fn method(&self) -> ReturnType;
    fn method_default(&self) -> String {
        "default".into()  // boleh override
    }
}

// Trait dengan bounds (supertrait)
trait Lanjutan: Debug + Display + Clone {
    fn lebih(&self);
}

// Implement
impl NamaTrait for MyType {
    fn method(&self) -> ReturnType { ... }
}

// Blanket impl
impl<T: Display> NamaTrait for T { ... }

// Bounds dalam fungsi
fn f<T: Trait1 + Trait2>(x: T) { ... }
fn f<T>(x: T) where T: Trait1 + Trait2 { ... }
fn f(x: impl Trait) { ... }       // argument position
fn f() -> impl Trait { ... }      // return position
fn f(x: &dyn Trait) { ... }      // dynamic dispatch
```

## Trait Penting dari std

```
fmt::Display     → {} format, auto-generate toString
fmt::Debug       → {:?} format, #[derive(Debug)]
Clone            → .clone(), #[derive(Clone)]
Copy             → auto-copy bitwise, #[derive(Copy)]
PartialEq + Eq   → == dan !=
PartialOrd + Ord → <, >, <=, >=, .sort()
Hash             → untuk HashMap/HashSet key
Default          → ::default(), #[derive(Default)]
From / Into      → type conversion
Iterator         → for loops, .map(), .filter(), ...
Add/Sub/Mul/Div  → operator overloading
Deref            → * operator, coercions
Drop             → destructor
Send + Sync      → thread safety (auto traits)
```

## Auto Traits

```
Send:  selamat untuk HANTAR ke thread lain
Sync:  selamat untuk SHARE (&T) antara thread (T: Sync ↔ &T: Send)

Auto-derived KECUALI ada field yang:
  *const T, *mut T → !Send, !Sync
  Rc<T>            → !Send, !Sync
  RefCell<T>       → !Sync (tapi Send kalau T: Send)
  UnsafeCell<T>    → !Sync

Manual override (UNSAFE!):
  unsafe impl Send for MyType {}
  unsafe impl Sync for MyType {}
```

## Static vs Dynamic Dispatch

```
impl Trait / generics  → Static (compile-time, zero overhead)
dyn Trait              → Dynamic (runtime vtable, overhead)

Guna static bila:  performance kritikal, type diketahui compile time
Guna dynamic bila: plugin system, koleksi pelbagai type, runtime decision
```

---

## 🏆 Cabaran Akhir

Cuba implement salah satu:

1. **Custom Iterator** — implement `Iterator` dengan `map`, `filter`, `take` chains
2. **Type-state Machine** — guna PhantomData untuk enforce state transitions (TCP connection: Closed → Listening → Connected → Closed)
3. **Trait Alias** — buat trait yang merupakan kombinasi beberapa traits (T: Debug + Display + Clone + PartialEq)
4. **Strategy Pattern** — guna `dyn Trait` untuk tukar algorithm pada runtime
5. **Derive Macro** — tulis proc macro yang auto-implement trait untuk struct

---

*Trait system Rust bukan sekadar interface — ia adalah bahasa untuk
 express capability, constraint, dan relationship antara types.*
*Kuasai trait, kuasai generic programming dalam Rust.* 🦀
