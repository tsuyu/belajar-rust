# 🧩 Orphan Rule dalam Rust — Panduan Lengkap

> "Kenapa saya tidak boleh implement trait untuk type orang lain?"
> Faham Orphan Rule, kenapa ia wujud, dan cara kerja sekeliling dengan betul.

---

## Situasi Yang Biasa Berlaku

```rust
// Anda ada crate luaran: `serde` dan `uuid`
// Anda nak implement trait serde::Serialize untuk uuid::Uuid
// dalam crate ANDA SENDIRI...

use serde::Serialize;
use uuid::Uuid;

impl Serialize for Uuid {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where S: serde::Serializer
    {
        serializer.serialize_str(&self.to_string())
    }
}

// COMPILER:
// error[E0117]: only traits defined in the current crate can be
// implemented for types defined outside of the crate
//   --> src/main.rs
//    |
//    | impl Serialize for Uuid {
//    |      ^^^^^^^^^ -------- `Uuid` is not defined in the current crate
//    |      |
//    |      `Serialize` is not defined in the current crate
//    |
//    = note: define and implement a trait or new type instead

// Selamat datang ke Orphan Rule! 🔒
```

---

## Apa Itu Orphan Rule?

```
DEFINISI:

  Untuk implement trait T untuk type S,
  SEKURANG-KURANGNYA SATU daripada ini mesti MILIK CRATE ANDA:
    - Trait T, ATAU
    - Type S

  Kalau KEDUA-DUANYA dari crate luar → DILARANG!
  (Ini dipanggil "orphan implementation")

NAMA LAIN:
  - Orphan Rule
  - Coherence Rule
  - "Blanket Implementation Constraint"

FORMULA MUDAH:
  ✔ impl MyTrait   for ExternalType  → OK (trait kita)
  ✔ impl ExternalTrait for MyType    → OK (type kita)
  ✔ impl MyTrait   for MyType        → OK (kedua-dua kita)
  ✘ impl ExternalTrait for ExternalType → DILARANG!
```

---

## Peta Pembelajaran

```
Bab 1 → Kenapa Orphan Rule Wujud?
Bab 2 → Memahami Batasan Penuh
Bab 3 → Cara Kerja Sekeliling #1: Newtype Pattern
Bab 4 → Cara Kerja Sekeliling #2: Local Trait
Bab 5 → Cara Kerja Sekeliling #3: Extension Trait
Bab 6 → Cara Kerja Sekeliling #4: Blanket Impl
Bab 7 → Fundamental Coherence
Bab 8 → Contoh Real-World
Bab 9 → Pattern Lanjutan
Bab 10 → Mini Project: Wrapper Library
```

---

# BAB 1: Kenapa Orphan Rule Wujud? 🛡️

## Masalah Tanpa Orphan Rule

```
Bayangkan Orphan Rule tidak wujud...

  Crate A:              Crate B:
  impl Display          impl Display
  for Vec<String>       for Vec<String>  ← siapa yang compiler pilih?!
  { ... }               { ... }

  Program anda guna KEDUA-DUA Crate A dan B.
  Compiler tidak tahu mana satu impl nak guna!
  → AMBIGUITI! 💥

Ini dipanggil "Coherence Problem":
  Tanpa Orphan Rule, trait impls boleh BERTINDIH
  dan compiler tidak boleh buat keputusan yang konsisten.
```

---

## Orphan Rule Jaga "Global Coherence"

```
Dengan Orphan Rule:

  Untuk setiap pasangan (Trait, Type),
  hanya boleh ada SATU implementation yang sah
  di seluruh ekosistem Rust.

  Crate serde:   → impl Serialize for String  (serde punya String? Tidak,
                                                tapi serde punya Serialize ✔)
  Crate uuid:    → impl Serialize for Uuid    (uuid punya Uuid ✔)
  Crate anda:    → impl Serialize for MyType  (anda punya MyType ✔)

  TIADA KONFLIK! Compiler tahu mana satu untuk setiap type.

Peraturan memastikan:
  ✔ Setiap (Trait, Type) ada paling banyak SATU impl
  ✔ Tiada "impl yang orphan" — selalu ada crate yang bertanggungjawab
  ✔ Downstream crates tidak boleh "break" upstream crates
  ✔ Ekosistem crate saling serasi
```

---

## 🧠 Brain Teaser #1

Tanpa Orphan Rule, apa yang akan berlaku dengan kod ini?

```rust
// crate-a (library):
impl std::fmt::Display for std::vec::Vec<i32> {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "LIST: {:?}", self)
    }
}

// crate-b (library):
impl std::fmt::Display for std::vec::Vec<i32> {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "ARRAY: {:?}", self)
    }
}

// program anda — guna kedua-dua:
use crate_a;
use crate_b;

fn main() {
    let v = vec![1, 2, 3];
    println!("{}", v); // Output apa? "LIST" atau "ARRAY"?
}
```

<details>
<summary>👀 Jawapan</summary>

**Ini adalah "impl conflict"** — program tidak akan compile (atau lebih teruk, behaviour bergantung pada urutan import yang tidak dijangka).

Dalam ekosistem yang besar dengan ribuan crates, ini adalah **mimpi ngeri**:
- Program yang sama boleh berkelakuan berbeza bergantung pada crates mana yang di-import
- Upgrade sesuatu crate boleh mengubah behaviour crate lain secara senyap
- Debug menjadi hampir mustahil

Orphan Rule **mencegah ini sepenuhnya** pada peringkat compile time — bukan runtime!
</details>

---

# BAB 2: Memahami Batasan Penuh 🔍

## Kes yang Dibenarkan vs Dilarang

```rust
// Andaikan:
//   ExternalTrait  = serde::Serialize   (dari crate luar)
//   ExternalType   = std::vec::Vec<T>   (dari crate luar)
//   LocalTrait     = trait kita sendiri
//   LocalType      = struct kita sendiri

// ✔ DIBENARKAN: Trait kita, Type luar
trait LocalTrait { fn buat(&self) -> String; }

impl LocalTrait for Vec<i32> {
    fn buat(&self) -> String { format!("{:?}", self) }
}

// ✔ DIBENARKAN: Trait luar, Type kita
use std::fmt;

struct MyPoint(f64, f64);

impl fmt::Display for MyPoint {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.0, self.1)
    }
}

// ✔ DIBENARKAN: Kedua-dua milik kita
impl LocalTrait for MyPoint {
    fn buat(&self) -> String { self.to_string() }
}

// ❌ DILARANG: Kedua-dua dari luar
// impl fmt::Display for Vec<i32> { ... }  // ERROR!
// impl serde::Serialize for uuid::Uuid { ... } // ERROR!
```

---

## Generic Types — Lebih Kompleks

```rust
// Ada kes yang lebih halus dengan generics...

// ❌ DILARANG: T adalah parameter, kedua-dua trait dan outer type dari luar
// impl<T> fmt::Display for Vec<T> { ... } // ERROR!

// ✔ DIBENARKAN: Kalau ada LOCAL type terlibat
struct MyWrapper<T>(T);

impl<T: fmt::Debug> fmt::Display for MyWrapper<T> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Wrapped({:?})", self.0)
    }
}

// ✔ DIBENARKAN: Local trait untuk external generic
trait MyIter {
    fn my_first(&self) -> Option<&i32>;
}

impl MyIter for Vec<i32> {
    fn my_first(&self) -> Option<&i32> {
        self.first()
    }
}

// Peraturan Ketat (RFC 2451 — "Turbofish"):
// Impl dibenarkan jika:
//   1. Trait adalah lokal, ATAU
//   2. Type yang paling luar adalah lokal
//   3. Semua parameter generic adalah "uncovered" types dari luar
//      (ini bahagian yang rumit!)
```

---

## "Fundamental" Types — Pengecualian Khas

```rust
// Beberapa types dalam std adalah "fundamental" dan
// mendapat pengecualian khas:
//   &T, &mut T, Box<T>

// ✔ Ini DIBENARKAN kerana & dan Box adalah "fundamental":
use std::fmt;

struct MyType;

impl fmt::Display for &MyType {      // & adalah fundamental
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "ref MyType")
    }
}

impl fmt::Display for Box<MyType> {  // Box adalah fundamental
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "boxed MyType")
    }
}

// Sebab: &T dan Box<T> dianggap sebagai "extension" dari T,
// bukan type yang sepenuhnya berasingan dalam konteks coherence.
```

---

# BAB 3: Cara Kerja Sekeliling #1: Newtype Pattern 🎁

## Penyelesaian Klasik — Bungkus Type Luar

```
MASALAH:
  Nak implement serde::Serialize untuk uuid::Uuid
  (kedua-dua dari crate luar → Orphan Rule halang)

PENYELESAIAN — Newtype Pattern:
  Bungkus uuid::Uuid dalam struct kita sendiri!
  Sekarang struct itu adalah LOCAL TYPE → boleh implement!
```

```rust
use serde::{Serialize, Deserialize, Serializer, Deserializer};
use uuid::Uuid;

// ─── Newtype Wrapper ──────────────────────────────────────────

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct MyUuid(Uuid);  // ← struct KITA yang bungkus Uuid orang lain

impl MyUuid {
    fn new() -> Self {
        MyUuid(Uuid::new_v4())
    }

    fn inner(&self) -> &Uuid {
        &self.0
    }
}

// Sekarang boleh implement Serialize untuk MyUuid! (MyUuid adalah LOCAL)
impl Serialize for MyUuid {
    fn serialize<S: Serializer>(&self, s: S) -> Result<S::Ok, S::Error> {
        s.serialize_str(&self.0.to_string())
    }
}

impl<'de> Deserialize<'de> for MyUuid {
    fn deserialize<D: Deserializer<'de>>(d: D) -> Result<Self, D::Error> {
        let s = String::deserialize(d)?;
        Uuid::parse_str(&s)
            .map(MyUuid)
            .map_err(serde::de::Error::custom)
    }
}

// Boleh implement Display juga!
impl std::fmt::Display for MyUuid {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "{}", self.0)
    }
}

// Deref untuk akses mudah ke Uuid methods
impl std::ops::Deref for MyUuid {
    type Target = Uuid;
    fn deref(&self) -> &Uuid { &self.0 }
}

fn main() {
    let id = MyUuid::new();
    let json = serde_json::to_string(&id).unwrap();
    println!("JSON: {}", json); // "550e8400-e29b-41d4-a716-..."

    let id2: MyUuid = serde_json::from_str(&json).unwrap();
    println!("Parse: {}", id2);

    // Deref membolehkan guna Uuid methods terus
    println!("Version: {:?}", id.get_version());
}
```

---

## Newtype dengan From/Into untuk Kemudahan

```rust
use std::fmt;

// Newtype untuk Vec<String> — nak implement Display
struct StringList(Vec<String>);

impl From<Vec<String>> for StringList {
    fn from(v: Vec<String>) -> Self {
        StringList(v)
    }
}

impl From<StringList> for Vec<String> {
    fn from(s: StringList) -> Vec<String> {
        s.0
    }
}

impl fmt::Display for StringList {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}

// Wrapper untuk HashMap dengan custom ordering
use std::collections::HashMap;

struct SortedMap(HashMap<String, i32>);

impl fmt::Display for SortedMap {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        let mut pairs: Vec<(&String, &i32)> = self.0.iter().collect();
        pairs.sort_by_key(|(k, _)| k.as_str());
        write!(f, "{{")?;
        for (i, (k, v)) in pairs.iter().enumerate() {
            if i > 0 { write!(f, ", ")?; }
            write!(f, "{}: {}", k, v)?;
        }
        write!(f, "}}")
    }
}

fn main() {
    let senarai: StringList = vec![
        "Ali".to_string(),
        "Siti".to_string(),
        "Amin".to_string(),
    ].into();

    println!("{}", senarai); // [Ali, Siti, Amin]

    let mut map = HashMap::new();
    map.insert("z-item".to_string(), 3);
    map.insert("a-item".to_string(), 1);
    map.insert("m-item".to_string(), 2);

    let sorted = SortedMap(map);
    println!("{}", sorted); // {a-item: 1, m-item: 2, z-item: 3}
}
```

---

## Kelemahan Newtype

```rust
// Masalah Newtype: Perlu expose SEMUA methods yang nak digunakan

struct MyVec<T>(Vec<T>);

impl<T> MyVec<T> {
    // Terpaksa implement SETIAP method yang nak guna
    fn push(&mut self, item: T)    { self.0.push(item) }
    fn len(&self) -> usize         { self.0.len() }
    fn is_empty(&self) -> bool     { self.0.is_empty() }
    fn iter(&self) -> std::slice::Iter<T> { self.0.iter() }
    // ... dan seterusnya — ini sangat membosankan!
}

// Penyelesaian: Deref → auto-expose semua methods!
use std::ops::{Deref, DerefMut};

struct BetterVec<T>(Vec<T>);

impl<T> Deref for BetterVec<T> {
    type Target = Vec<T>;
    fn deref(&self) -> &Vec<T> { &self.0 }
}

impl<T> DerefMut for BetterVec<T> {
    fn deref_mut(&mut self) -> &mut Vec<T> { &mut self.0 }
}

fn main() {
    let mut v = BetterVec(vec![1, 2, 3]);
    v.push(4);      // ← Vec::push via Deref!
    v.sort();       // ← Vec::sort via Deref!
    println!("{}", v.len()); // ← Vec::len via Deref!
    println!("{:?}", *v);    // deref explicit
}
```

---

# BAB 4: Cara Kerja Sekeliling #2: Local Trait 📋

## Buat Trait Sendiri

```rust
// Kalau nak extend behaviour external types,
// buat trait sendiri yang buat apa yang anda nak!

// MASALAH: Nak tambah method .ke_ringgit() untuk f64
// (tidak boleh impl external trait untuk f64)

// ✔ PENYELESAIAN: Buat extension trait sendiri
trait RinggitExt {
    fn ke_ringgit(&self) -> String;
    fn tambah_cukai(&self, kadar: f64) -> f64;
    fn bulatkan_sen(&self) -> f64;
}

impl RinggitExt for f64 {
    fn ke_ringgit(&self) -> String {
        format!("RM{:.2}", self)
    }

    fn tambah_cukai(&self, kadar: f64) -> f64 {
        self * (1.0 + kadar / 100.0)
    }

    fn bulatkan_sen(&self) -> f64 {
        (self * 100.0).round() / 100.0
    }
}

// Tambah method untuk &str
trait StringExt {
    fn adalah_emel(&self) -> bool;
    fn panjang_perkataan(&self) -> usize;
    fn potong_dengan(&self, pembatas: char) -> Vec<&str>;
}

impl StringExt for str {
    fn adalah_emel(&self) -> bool {
        self.contains('@') && self.contains('.')
    }

    fn panjang_perkataan(&self) -> usize {
        self.split_whitespace().count()
    }

    fn potong_dengan(&self, pembatas: char) -> Vec<&str> {
        self.split(pembatas).collect()
    }
}

fn main() {
    let harga = 99.95f64;
    println!("{}", harga.ke_ringgit());                 // RM99.95
    println!("{}", harga.tambah_cukai(6.0).ke_ringgit()); // RM105.95
    println!("{}", 10.126f64.bulatkan_sen());            // 10.13

    let emel = "ali@email.com";
    println!("{}", emel.adalah_emel());                  // true
    println!("{}", "hello world".panjang_perkataan());   // 2

    let csv = "ali,siti,amin";
    println!("{:?}", csv.potong_dengan(','));
    // ["ali", "siti", "amin"]
}
```

---

## 🧠 Brain Teaser #2

Apakah perbezaan antara Newtype Pattern dan Local Trait?

<details>
<summary>👀 Jawapan</summary>

```
Newtype Pattern:
  Bungkus external type dalam struct baru.
  Guna bila: Nak implement EXTERNAL TRAIT untuk external type.
  Trade-off: Perlu unwrap/wrap type bila hantar ke fungsi yang jangka
             original type. Perlu Deref kalau nak expose methods.

  struct MyUuid(Uuid);
  impl Serialize for MyUuid { ... }  // ← Serialize adalah external

Local Trait:
  Buat TRAIT SENDIRI yang extend external type.
  Guna bila: Nak tambah BEHAVIOR BARU ke external type.
  Trade-off: Caller perlu `use YourCrate::YourTrait` untuk akses
             methods baru. Tidak berfungsi untuk standard traits
             macam Display, Serialize, dll.

  trait MyExt { fn buat_sesuatu(&self); }
  impl MyExt for Vec<i32> { ... }  // ← Vec adalah external

BILA GUNA APA:
  Nak Display, Serialize, Hash, dll. untuk external type?
    → Newtype Pattern (sebab traits ini sudah defined)

  Nak tambah method khas untuk external type?
    → Local Trait (lebih flexible, tiada wrapping overhead)
```
</details>

---

# BAB 5: Cara Kerja Sekeliling #3: Extension Trait 🔌

## Pattern Extension Trait yang Idiomatik

```rust
// Extension Trait = Local Trait yang "extend" external type
// Pattern ini sangat popular dalam ekosistem Rust!

// Contoh: itertools crate extend Iterator dengan methods baru
// Contoh: tokio extend standard types dengan async methods

// Mari buat extension trait untuk Iterator

trait IteratorExt: Iterator {
    // Kira dan return (count, sum) serentak
    fn kira_dan_jumlah(self) -> (usize, Self::Item)
    where
        Self: Sized,
        Self::Item: std::iter::Sum + Copy,
    {
        let mut count = 0;
        let mut sum = std::iter::empty::<Self::Item>().sum();
        for item in self {
            count += 1;
            sum = sum + item; // simplified — not truly correct but illustrative
        }
        (count, sum)
    }

    // Chunk iterator ke Vec<Vec<T>>
    fn chunks_of(self, saiz: usize) -> Vec<Vec<Self::Item>>
    where
        Self: Sized,
    {
        let mut hasil = Vec::new();
        let mut chunk = Vec::new();
        for item in self {
            chunk.push(item);
            if chunk.len() == saiz {
                hasil.push(chunk);
                chunk = Vec::new();
            }
        }
        if !chunk.is_empty() {
            hasil.push(chunk);
        }
        hasil
    }

    // Filter dan count dalam satu pass
    fn kira_yang(self, predicate: impl Fn(&Self::Item) -> bool) -> usize
    where
        Self: Sized,
    {
        self.filter(|x| predicate(x)).count()
    }
}

// Implement untuk SEMUA Iterator (blanket implementation)
impl<I: Iterator> IteratorExt for I {}

fn main() {
    let nombor = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // Guna extension methods baru
    let chunks = nombor.iter().cloned().chunks_of(3);
    println!("{:?}", chunks);
    // [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10]]

    let bilangan_genap = nombor.iter().kira_yang(|&&n| n % 2 == 0);
    println!("Genap: {}", bilangan_genap); // 5

    // Caller perlu import trait untuk guna methods baru
    // use my_crate::IteratorExt;  ← ini perlu dalam kod mereka
}
```

---

## ResultExt — Extension untuk Result

```rust
// Extend Result dengan methods berguna

trait ResultExt<T, E> {
    fn log_err(self, konteks: &str) -> Self where E: std::fmt::Display;
    fn ok_atau_log(self, konteks: &str) -> Option<T> where E: std::fmt::Display;
    fn map_err_ke_string(self) -> Result<T, String> where E: std::fmt::Display;
}

impl<T, E: std::fmt::Display> ResultExt<T, E> for Result<T, E> {
    fn log_err(self, konteks: &str) -> Self {
        if let Err(ref e) = self {
            eprintln!("[ERROR] {}: {}", konteks, e);
        }
        self
    }

    fn ok_atau_log(self, konteks: &str) -> Option<T> {
        match self {
            Ok(val) => Some(val),
            Err(e)  => {
                eprintln!("[WARN] {}: {}", konteks, e);
                None
            }
        }
    }

    fn map_err_ke_string(self) -> Result<T, String> {
        self.map_err(|e| e.to_string())
    }
}

fn main() {
    use std::num::ParseIntError;

    // log_err — log bila ada error, teruskan Result
    let r: Result<i32, _> = "abc".parse::<i32>()
        .log_err("Parse port"); // log error, teruskan Err

    // ok_atau_log — convert ke Option, log kalau Err
    let nilai = "42".parse::<i32>()
        .ok_atau_log("Parse nombor"); // Some(42), tiada log

    let tiada = "xyz".parse::<i32>()
        .ok_atau_log("Parse lagi"); // None, log error

    println!("{:?}", r);    // Err(...)
    println!("{:?}", nilai); // Some(42)
    println!("{:?}", tiada); // None
}
```

---

# BAB 6: Cara Kerja Sekeliling #4: Blanket Impl 🌐

## Implement untuk Semua Types yang Memenuhi Syarat

```rust
// Blanket impl = implement trait untuk SEMUA types yang satisfy bounds

trait Ringkas {
    fn ringkas(&self) -> String;
}

// Implement untuk SEMUA types yang ada Display
impl<T: std::fmt::Display> Ringkas for T {
    fn ringkas(&self) -> String {
        let s = format!("{}", self);
        if s.len() > 20 {
            format!("{}...", &s[..17])
        } else {
            s
        }
    }
}

// Kini SEMUA type yang implement Display
// secara automatik dapat .ringkas() !
fn main() {
    println!("{}", 42i32.ringkas());           // "42"
    println!("{}", 3.14f64.ringkas());          // "3.14"
    println!("{}", "Hello, world!".ringkas());  // "Hello, world!"
    println!("{}", "Ini adalah teks yang sangat panjang sekali".ringkas());
    // "Ini adalah teks yang..."
}
```

---

## Blanket Impl dalam Standard Library

```rust
// std library sendiri guna blanket impl secara meluas!

// Contoh 1: ToString — auto-derived dari Display
// impl<T: fmt::Display> ToString for T { ... }
// → Semua type yang ada Display boleh .to_string()

// Contoh 2: Into dari From
// impl<T, U: From<T>> Into<T> for U { ... }
// → Implement From → dapat Into secara automatik!

// Kita boleh buat serupa:
trait Muat {
    fn muat_dari(sumber: &str) -> Option<Self> where Self: Sized;
}

trait MuatMudah: Muat {
    fn muat_atau_default(sumber: &str) -> Self where Self: Default {
        Self::muat_dari(sumber).unwrap_or_default()
    }
}

// Blanket: SEMUA type yang implement Muat dapat MuatMudah
impl<T: Muat + Default> MuatMudah for T {}

// Buat dua types yang implement Muat
#[derive(Debug, Default)]
struct NilaiInt(i32);

#[derive(Debug, Default)]
struct NilaiBool(bool);

impl Muat for NilaiInt {
    fn muat_dari(s: &str) -> Option<Self> {
        s.parse::<i32>().ok().map(NilaiInt)
    }
}

impl Muat for NilaiBool {
    fn muat_dari(s: &str) -> Option<Self> {
        match s.to_lowercase().as_str() {
            "true" | "1" | "ya" => Some(NilaiBool(true)),
            "false" | "0" | "tidak" => Some(NilaiBool(false)),
            _ => None,
        }
    }
}

fn main() {
    // Kedua-dua dapat MuatMudah secara automatik!
    let n = NilaiInt::muat_atau_default("42");
    println!("{:?}", n); // NilaiInt(42)

    let n2 = NilaiInt::muat_atau_default("bukan nombor");
    println!("{:?}", n2); // NilaiInt(0) ← default

    let b = NilaiBool::muat_atau_default("ya");
    println!("{:?}", b); // NilaiBool(true)
}
```

---

# BAB 7: Fundamental Coherence & Edge Cases 🎯

## Kes Rumit: Generic Parameters

```rust
// Peraturan yang lebih tepat:
// Impl dibenarkan kalau LOCAL TYPE "covers" type parameter

// ✔ OK: Local type covers T
struct Wrapper<T>(T);

impl<T: std::fmt::Display> std::fmt::Debug for Wrapper<T> {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "Wrapper({})", self.0)
    }
}

// ✔ OK: Trait dari local crate, T adalah uncovered
trait MyTrait<T> { fn laksana(&self, x: T); }
impl<T> MyTrait<T> for Vec<T>
where T: Clone
{
    fn laksana(&self, _x: T) { }
}

// ❌ DILARANG: External trait, external type, T tidak covered
// impl<T> std::fmt::Display for Vec<T>
// where T: std::fmt::Display
// { ... } // ERROR!

// Kenapa T tidak "covered"?
// T boleh jadi local type → implementation ini MUNGKIN conflict
// dengan implementation crate lain bila T adalah type mereka.
```

---

## Negative Impls & Auto Traits (Advanced)

```rust
// Rust ada "auto traits" — Send, Sync, Unpin, UnwindSafe
// Ini di-implement secara automatik kalau semua fields implement

// TIDAK perlu implement manual (auto-derived):
struct SafeType {
    data: i32,     // i32: Send + Sync
    name: String,  // String: Send + Sync
}
// SafeType auto-implement Send + Sync!

// Kalau ada field yang TIDAK Send/Sync:
use std::rc::Rc;
struct NotSafe {
    data: Rc<i32>,  // Rc TIDAK Send/Sync!
}
// NotSafe auto NOT Send + NOT Sync!

// Explicit opt-out (unsafe):
struct ManualOverride(i32);

// unsafe kerana kita jamin ia selamat tapi compiler tidak boleh verify
unsafe impl Send for ManualOverride {}
unsafe impl Sync for ManualOverride {}

// Negative impl (nightly Rust):
// impl !Send for MyType {}  // Explicitly NOT Send
```

---

## Overlapping Impls — Bila Dua Impl Conflict

```rust
// Rust tidak benarkan dua impl yang boleh conflict:

trait MyTrait {
    fn kaedah(&self) -> String;
}

// Impl 1: untuk semua T yang implement Display
// impl<T: std::fmt::Display> MyTrait for T { ... }

// Impl 2: untuk i32 specifically
// impl MyTrait for i32 { ... }

// ERROR! i32 implement Display → kedua-dua impl boleh apply untuk i32!
// Rust tidak tahu mana satu nak guna.

// Penyelesaian: Specialization (nightly only, tidak stable lagi)
// atau reka semula trait anda.

// Cara selamat: Pilih SATU approach sahaja
trait MyTraitSafe {
    fn kaedah(&self) -> String;
}

// Approach A: Blanket impl (guna Display)
// impl<T: std::fmt::Display> MyTraitSafe for T {
//     fn kaedah(&self) -> String { format!("Display: {}", self) }
// }

// Approach B: Manual impl untuk setiap type
impl MyTraitSafe for i32 {
    fn kaedah(&self) -> String { format!("i32: {}", self) }
}
impl MyTraitSafe for String {
    fn kaedah(&self) -> String { format!("String: {}", self) }
}
// (pilih A atau B, bukan kedua-dua!)
```

---

# BAB 8: Contoh Real-World 🌍

## Serde + External Types

```rust
use serde::{Serialize, Deserialize};

// MASALAH: Nak serialize chrono::DateTime ke format custom
// chrono punya DateTime, serde punya Serialize → Orphan!

// PENYELESAIAN: Modul dengan serialize/deserialize functions
// (pattern popular dalam ecosystem Rust)

mod datetime_format {
    use chrono::{DateTime, Utc, TimeZone};
    use serde::{Serializer, Deserializer, Deserialize};

    const FORMAT: &str = "%Y-%m-%dT%H:%M:%S%.3fZ";

    pub fn serialize<S: Serializer>(
        dt: &DateTime<Utc>,
        s: S,
    ) -> Result<S::Ok, S::Error> {
        s.serialize_str(&dt.format(FORMAT).to_string())
    }

    pub fn deserialize<'de, D: Deserializer<'de>>(
        d: D,
    ) -> Result<DateTime<Utc>, D::Error> {
        let s = String::deserialize(d)?;
        Utc.datetime_from_str(&s, FORMAT)
           .map_err(serde::de::Error::custom)
    }
}

#[derive(Debug, Serialize, Deserialize)]
struct Peristiwa {
    nama: String,
    #[serde(with = "datetime_format")]
    masa: chrono::DateTime<chrono::Utc>,
}
```

---

## Display untuk Vec — Newtype Solution

```rust
use std::fmt;

// MASALAH: Vec<T> tiada Display — nak format cantik
// Vec adalah external, Display adalah external → Orphan!

// PENYELESAIAN 1: Newtype
struct PaparVec<T>(Vec<T>);

impl<T: fmt::Display> fmt::Display for PaparVec<T> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[")?;
        for (i, item) in self.0.iter().enumerate() {
            if i > 0 { write!(f, ", ")?; }
            write!(f, "{}", item)?;
        }
        write!(f, "]")
    }
}

// PENYELESAIAN 2: Helper function (tiada newtype overhead)
fn format_vec<T: fmt::Display>(v: &[T]) -> String {
    let items: Vec<String> = v.iter().map(|x| x.to_string()).collect();
    format!("[{}]", items.join(", "))
}

fn main() {
    let nombor = vec![1, 2, 3, 4, 5];
    let nama = vec!["Ali", "Siti", "Amin"];

    // Newtype approach
    println!("{}", PaparVec(nombor.clone())); // [1, 2, 3, 4, 5]
    println!("{}", PaparVec(nama.clone()));   // [Ali, Siti, Amin]

    // Helper function approach
    println!("{}", format_vec(&nombor));      // [1, 2, 3, 4, 5]
    println!("{}", format_vec(&nama));        // [Ali, Siti, Amin]
}
```

---

## Axum + Custom Response Type

```rust
// MASALAH: Nak impl axum::response::IntoResponse
// untuk types dari crate lain

use axum::response::{IntoResponse, Response};
use axum::http::StatusCode;
use axum::Json;

// ❌ TIDAK BOLEH: axum trait + external type
// impl IntoResponse for serde_json::Value { ... } // Error!

// ✔ PENYELESAIAN: Newtype wrapper
struct JsonResponse(serde_json::Value);

impl IntoResponse for JsonResponse {
    fn into_response(self) -> Response {
        Json(self.0).into_response()
    }
}

// Atau lebih praktikal — buat ApiResponse sendiri
#[derive(serde::Serialize)]
struct ApiResponse<T: serde::Serialize> {
    berjaya: bool,
    data: Option<T>,
    mesej: Option<String>,
}

impl<T: serde::Serialize> IntoResponse for ApiResponse<T> {
    fn into_response(self) -> Response {
        let status = if self.berjaya {
            StatusCode::OK
        } else {
            StatusCode::BAD_REQUEST
        };
        (status, Json(self)).into_response()
    }
}
```

---

# BAB 9: Pattern Lanjutan 🎯

## Sealed Trait — Guna Orphan Rule sebagai Fitur!

```rust
// Sealed Trait = trait yang SENGAJA menggunakan Orphan Rule
// untuk HADKAN siapa yang boleh implement!

// Buat inner sealed trait yang private
mod sealed {
    pub trait Sealed {}

    // Hanya implement untuk types yang kita izinkan
    impl Sealed for i32 {}
    impl Sealed for f64 {}
    impl Sealed for String {}
    impl Sealed for bool {}
}

// Public trait yang depend on Sealed
// Orang luar TIDAK BOLEH implement ini kerana mereka
// tidak boleh implement sealed::Sealed (private module!)
pub trait NilaiSelamat: sealed::Sealed {
    fn ke_json_value(&self) -> String;
}

impl NilaiSelamat for i32 {
    fn ke_json_value(&self) -> String { self.to_string() }
}

impl NilaiSelamat for f64 {
    fn ke_json_value(&self) -> String { self.to_string() }
}

impl NilaiSelamat for String {
    fn ke_json_value(&self) -> String { format!("\"{}\"", self) }
}

impl NilaiSelamat for bool {
    fn ke_json_value(&self) -> String { self.to_string() }
}

// Pengguna library boleh GUNA trait, tapi tidak boleh IMPLEMENT!
// Ini berguna untuk:
//   - API yang nak control jenis apa yang boleh digunakan
//   - Prevent future-compatibility breaks
//   - Type-state pattern

fn main() {
    fn papar_json(val: &impl NilaiSelamat) {
        println!("{}", val.ke_json_value());
    }

    papar_json(&42i32);          // 42
    papar_json(&3.14f64);        // 3.14
    papar_json(&"hello".to_string()); // "hello"

    // struct Asing;
    // impl NilaiSelamat for Asing {}  // ← ERROR! Tidak boleh implement
}
```

---

## Workaround Melalui Cargo Features

```toml
# Cara sebenar yang digunakan oleh crates popular!
# uuid crate sendiri ada feature untuk serde:

[dependencies]
uuid = { version = "1", features = ["v4", "serde"] }
#                                          ^^^
#                         Ini enable impl Serialize/Deserialize
#                         dalam uuid crate sendiri!

chrono = { version = "0.4", features = ["serde"] }
# Sama! chrono sendiri implement Serialize apabila feature "serde" aktif.
```

```rust
// Bagaimana uuid buat ini:
// (dalam uuid/src/lib.rs)

#[cfg(feature = "serde")]
impl serde::Serialize for Uuid {
    // Boleh! Sebab ini DALAM CRATE uuid sendiri
    // uuid punya Uuid → boleh implement external trait serde::Serialize
    fn serialize<S>(&self, s: S) -> Result<S::Ok, S::Error>
    where S: serde::Serializer
    {
        s.serialize_str(&self.to_string())
    }
}

// Pelajaran: Kalau anda library author, expose feature flags
// untuk integrasi dengan crates lain!
```

---

## 🧠 Brain Teaser #3

Apakah yang berlaku di sini, dan bagaimana betulkan?

```rust
// crate saya: my-utils
use std::fmt;
use chrono::NaiveDate;

// Saya nak semua NaiveDate boleh print dengan format Malaysia
// Tapi ini Orphan Rule violation...

impl fmt::Display for NaiveDate {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{}/{}/{}", self.day(), self.month(), self.year())
    }
}

fn main() {
    let tarikh = NaiveDate::from_ymd_opt(2024, 1, 15).unwrap();
    println!("{}", tarikh);
}
```

<details>
<summary>👀 Jawapan</summary>

**Masalah:** `fmt::Display` dari std dan `NaiveDate` dari chrono — kedua-dua external, Orphan Rule halang.

**4 cara penyelesaian:**

```rust
// ── Cara 1: Newtype ──────────────────────────────────────────
use chrono::NaiveDate;
use std::fmt;

struct TarikhMalaysia(NaiveDate);

impl fmt::Display for TarikhMalaysia {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{}/{}/{}", self.0.day(), self.0.month(), self.0.year())
    }
}

let t = TarikhMalaysia(NaiveDate::from_ymd_opt(2024, 1, 15).unwrap());
println!("{}", t); // 15/1/2024

// ── Cara 2: Helper function ─────────────────────────────────
fn format_tarikh(d: &NaiveDate) -> String {
    format!("{}/{}/{}", d.day(), d.month(), d.year())
}

let t = NaiveDate::from_ymd_opt(2024, 1, 15).unwrap();
println!("{}", format_tarikh(&t)); // 15/1/2024

// ── Cara 3: Extension Trait ─────────────────────────────────
trait TarikhExt {
    fn format_malaysia(&self) -> String;
}

impl TarikhExt for NaiveDate {
    fn format_malaysia(&self) -> String {
        format!("{}/{}/{}", self.day(), self.month(), self.year())
    }
}

println!("{}", t.format_malaysia()); // 15/1/2024

// ── Cara 4: chrono format (paling mudah!) ─────────────────
println!("{}", t.format("%d/%m/%Y")); // 15/01/2024
// chrono dah ada format() — selalu check docs dulu!
```
</details>

---

# BAB 10: Mini Project — Wrapper Library 🏗️

```rust
// Bina "extension library" untuk projek KADA
// yang extend external types dengan methods berguna

use std::fmt;
use std::collections::HashMap;

// ─── 1. Extension Traits ──────────────────────────────────────

/// Extend String dan &str dengan helpers untuk Bahasa Malaysia
pub trait MalayStringExt {
    fn huruf_besar_pertama(&self) -> String;
    fn adalah_no_ic(&self) -> bool;
    fn adalah_no_telefon_my(&self) -> bool;
    fn topeng_ic(&self) -> String;
}

impl MalayStringExt for str {
    fn huruf_besar_pertama(&self) -> String {
        let mut c = self.chars();
        match c.next() {
            None    => String::new(),
            Some(f) => f.to_uppercase().to_string() + c.as_str(),
        }
    }

    fn adalah_no_ic(&self) -> bool {
        // Format: YYMMDD-SS-NNNN (12 digit, dengan atau tanpa dash)
        let digit: String = self.chars().filter(|c| c.is_ascii_digit()).collect();
        digit.len() == 12
    }

    fn adalah_no_telefon_my(&self) -> bool {
        let digit: String = self.chars().filter(|c| c.is_ascii_digit()).collect();
        // Malaysia: 01x-xxxxxxx (10-11 digit)
        (digit.len() == 10 || digit.len() == 11)
            && (digit.starts_with("01") || digit.starts_with("03"))
    }

    fn topeng_ic(&self) -> String {
        let digit: String = self.chars().filter(|c| c.is_ascii_digit()).collect();
        if digit.len() != 12 {
            return "IC tidak sah".to_string();
        }
        format!("{}**-**-{}", &digit[..6], &digit[10..])
    }
}

/// Extend f64 untuk operasi kewangan Ringgit Malaysia
pub trait RinggitExt {
    fn ke_rm(&self) -> String;
    fn tambah_gst(&self) -> f64;     // GST 6%
    fn tambah_sst(&self) -> f64;     // SST 8%
    fn bulatkan_sen(&self) -> f64;
    fn ke_perkataan(&self) -> String;
}

impl RinggitExt for f64 {
    fn ke_rm(&self) -> String {
        format!("RM{:.2}", self.bulatkan_sen())
    }

    fn tambah_gst(&self) -> f64 {
        (self * 1.06).bulatkan_sen()
    }

    fn tambah_sst(&self) -> f64 {
        (self * 1.08).bulatkan_sen()
    }

    fn bulatkan_sen(&self) -> f64 {
        (self * 100.0).round() / 100.0
    }

    fn ke_perkataan(&self) -> String {
        let ringgit = *self as u64;
        let sen = ((self - ringgit as f64) * 100.0).round() as u64;
        if sen == 0 {
            format!("Ringgit Malaysia {}", ringgit)
        } else {
            format!("Ringgit Malaysia {} Sen {}", ringgit, sen)
        }
    }
}

// ─── 2. Newtype Wrappers ──────────────────────────────────────

/// Wrapper untuk Vec<T> dengan Display yang cantik
pub struct Senarai<T>(pub Vec<T>);

impl<T: fmt::Display> fmt::Display for Senarai<T> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        if self.0.is_empty() {
            return write!(f, "(kosong)");
        }
        for (i, item) in self.0.iter().enumerate() {
            if i > 0 { write!(f, ", ")?; }
            write!(f, "{}", item)?;
        }
        Ok(())
    }
}

impl<T> From<Vec<T>> for Senarai<T> {
    fn from(v: Vec<T>) -> Self { Senarai(v) }
}

impl<T> std::ops::Deref for Senarai<T> {
    type Target = Vec<T>;
    fn deref(&self) -> &Vec<T> { &self.0 }
}

/// Wrapper untuk HashMap dengan sorted Display
pub struct PetaTersusun<K: Ord, V>(pub HashMap<K, V>);

impl<K: Ord + fmt::Display, V: fmt::Display> fmt::Display for PetaTersusun<K, V> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        let mut pasangan: Vec<(&K, &V)> = self.0.iter().collect();
        pasangan.sort_by_key(|(k, _)| *k);
        writeln!(f, "{{")?;
        for (k, v) in pasangan {
            writeln!(f, "  {}: {}", k, v)?;
        }
        write!(f, "}}")
    }
}

// ─── 3. Demonstrate ───────────────────────────────────────────

fn main() {
    println!("=== String Extensions ===");

    let nama = "ali ahmad";
    println!("Capitalize: {}", nama.huruf_besar_pertama()); // Ali ahmad

    let ic = "900115-10-1234";
    println!("IC valid: {}", ic.adalah_no_ic());   // true
    println!("IC topeng: {}", ic.topeng_ic());      // 900115**-**-34

    let telefon = "012-345-6789";
    println!("Telefon valid: {}", telefon.adalah_no_telefon_my()); // true

    println!("\n=== Ringgit Extensions ===");

    let harga = 99.95f64;
    println!("Harga:     {}", harga.ke_rm());           // RM99.95
    println!("GST (6%):  {}", harga.tambah_gst().ke_rm()); // RM105.95
    println!("SST (8%):  {}", harga.tambah_sst().ke_rm()); // RM107.95
    println!("Perkataan: {}", harga.ke_perkataan());

    let jumlah = 1234.56f64;
    println!("{}", jumlah.ke_perkataan()); // Ringgit Malaysia 1234 Sen 56

    println!("\n=== Senarai Wrapper ===");

    let pekerja: Senarai<&str> = vec!["Ali", "Siti", "Amin"].into();
    println!("Pekerja: {}", pekerja); // Ali, Siti, Amin
    println!("Bilangan: {}", pekerja.len()); // 3 (via Deref)

    let kosong: Senarai<i32> = vec![].into();
    println!("Kosong: {}", kosong); // (kosong)

    println!("\n=== PetaTersusun Wrapper ===");

    let mut gaji = HashMap::new();
    gaji.insert("Zara".to_string(), 4500.0f64.ke_rm());
    gaji.insert("Ali".to_string(),  5000.0f64.ke_rm());
    gaji.insert("Siti".to_string(), 4200.0f64.ke_rm());

    let peta = PetaTersusun(gaji);
    println!("{}", peta);
    // {
    //   Ali: RM5000.00
    //   Siti: RM4200.00
    //   Zara: RM4500.00
    // }
}
```

---

# 📋 Rujukan Pantas — Orphan Rule Cheat Sheet

## Formula Mudah

```
✔ BOLEH: impl Trait<External>   for External   jika Trait KITA
✔ BOLEH: impl External<Trait>   for External   jika Type KITA
✔ BOLEH: impl Local             for Local
✘ TIDAK: impl ExternalTrait     for ExternalType
```

## Penyelesaian Mengikut Senario

```
SENARIO                              PENYELESAIAN
──────────────────────────────────────────────────────────────
Nak Display/Debug untuk external     Newtype Pattern
  type?
Nak Serialize/Deserialize untuk      Newtype atau #[serde(with)]
  external type?
Nak tambah methods baru ke           Extension Trait (Local Trait)
  external type?
Nak extend Iterator/Result/Option?   Extension Trait + Blanket Impl
Nak restrict siapa boleh implement?  Sealed Trait
Kongsi fungsi tanpa traits?          Helper functions
Library author nak sokong serde?     Cargo feature + cfg attribute
```

## Tiga Pendekatan Utama

```rust
// 1. NEWTYPE PATTERN
struct MyExt(ExternalType);
impl ExternalTrait for MyExt { ... }
// Pro: boleh implement external traits
// Con: overhead wrapping/unwrapping, perlu expose methods

// 2. LOCAL TRAIT (EXTENSION TRAIT)
trait MyTrait { fn tambah_method(&self); }
impl MyTrait for ExternalType { ... }
// Pro: ergonomic, tiada wrapping
// Con: caller perlu `use MyTrait`, tidak boleh implement external traits

// 3. HELPER FUNCTION
fn helper(x: &ExternalType) -> String { ... }
// Pro: paling mudah, tiada overhead
// Con: tidak boleh chain method, kurang ergonomic
```

---

## 🏆 Cabaran Akhir

Cuba implement salah satu:

1. **Newtype untuk `IpAddr`** — implement `serde::Serialize` yang serialize IP sebagai string `"192.168.1.1"` bukan object
2. **Extension Trait untuk `Vec<f64>`** — tambah `.statistik()` yang return `(min, max, purata, std_dev)` dalam satu call
3. **Sealed Trait untuk database types** — buat trait `DapatDisimpan` yang hanya boleh diimplement untuk jenis yang anda tentukan
4. **Wrapper untuk `HashMap` dengan TTL** — newtype yang auto-expire keys selepas tempoh tertentu
5. **`OptionExt` dan `ResultExt`** — tambah methods berguna yang tidak ada dalam standard library

---

*Orphan Rule bukan musuh — ia adalah penjaga ekosistem Rust yang sihat.*
*Faham batasannya, dan anda akan menulis kod yang lebih baik.* 🦀
