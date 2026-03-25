# ⚡ Rust — Quick Reference Cheatsheet
> Rujukan pantas Rust.
> Cari topik, salin syntax, teruskan kerja.

---

## 📑 Kandungan

| Topik | | Topik |
|---|---|---|
| [Hello World](#hello-world) | | [Closures](#closures) |
| [Fundamentals](#fundamentals) | | [Iterators](#iterators) |
| [Naming Conventions](#naming-conventions) | | [Error Handling](#error-handling) |
| [Variables](#variables) | | [Structs](#structs) |
| [Data Types](#data-types) | | [Enums](#enums) |
| [Operators](#operators) | | [Traits](#traits) |
| [Strings](#strings) | | [Generics](#generics) |
| [Functions](#functions) | | [Lifetimes](#lifetimes) |
| [Control Structures](#control-structures) | | [Modules](#modules) |
| [Ownership](#ownership) | | [Collections](#collections) |
| [Borrowing](#borrowing) | | [Files & IO](#files--io) |
| [Pattern Matching](#pattern-matching) | | [Smart Pointers](#smart-pointers) |
| [Option](#option) | | [Concurrency](#concurrency) |
| [Result](#result) | | [Cargo](#cargo) |
| [Attributes](#attributes) | | [Testing](#testing) |

---

# Hello World

```rust
fn main() {
    println!("Hello, World!");
}
```

```bash
cargo new hello_world
cd hello_world
cargo run
```

---

# Fundamentals

```
Compiled:         rustc / cargo
Typed:            Static, Strong
Memory:           Ownership (no GC)
Paradigm:         Multi (functional + OOP-like)
Null:             None (guna Option<T>)
Exception:        None (guna Result<T,E>)
```

**Keyword Utama**

```
let       → declare variable (immutable by default)
mut       → buat variable mutable
const     → constant (perlu type annotation)
static    → static variable (hidup sepanjang program)
fn        → function
struct    → structure
enum      → enumeration
impl      → implement methods
trait     → define shared behaviour
type      → type alias
use       → import
mod       → module
pub       → public visibility
async     → async function
await     → tunggu async
unsafe    → unsafe block
where     → trait bound clause
```

---

# Naming Conventions

| Perkara | Konvensyen | Contoh |
|---|---|---|
| Variable | snake_case | `nama_pekerja` |
| Function | snake_case | `kira_jumlah()` |
| Constant | SCREAMING_SNAKE | `MAX_SAIZ` |
| Static | SCREAMING_SNAKE | `VERSI_APP` |
| Type / Struct / Enum | PascalCase | `RekodPekerja` |
| Trait | PascalCase | `BolehPapar` |
| Module | snake_case | `pengurusan_fail` |
| Lifetime | lowercase short | `'a`, `'static` |
| Generic | PascalCase / short | `T`, `K`, `V`, `Item` |

---

# Variables

```rust
// Immutable (default)
let x = 5;
let y: i32 = 10;

// Mutable
let mut kiraan = 0;
kiraan += 1;

// Constant — perlu type, known at compile time
const MAX: u32 = 100;

// Static
static NAMA_APP: &str = "KADA";

// Shadowing — declare semula dengan nama sama
let x = 5;
let x = x + 1;         // x = 6
let x = x.to_string(); // x = "6" (boleh tukar type!)

// Destructuring
let (a, b, c) = (1, 2, 3);
let [x, y, z] = [1, 2, 3];

// Underscore — abaikan nilai / variable
let _ = fungsi_dengan_return();
let _tidak_guna = 5;

// Type inference
let v = vec![1, 2, 3];    // Vec<i32>
let m = HashMap::new();   // akan infer kemudian
```

---

# Data Types

## Integer

| Type | Bits | Min | Max |
|---|---|---|---|
| `i8` | 8 | -128 | 127 |
| `i16` | 16 | -32,768 | 32,767 |
| `i32` | 32 | -2.1B | 2.1B |
| `i64` | 64 | huge | huge |
| `i128` | 128 | huger | huger |
| `isize` | arch | ptr size | ptr size |
| `u8` | 8 | 0 | 255 |
| `u16` | 16 | 0 | 65,535 |
| `u32` | 32 | 0 | 4.3B |
| `u64` | 64 | 0 | huge |
| `u128` | 128 | 0 | huger |
| `usize` | arch | 0 | ptr size |

```rust
let a: i32 = -42;
let b: u64 = 1_000_000; // _ sebagai separator
let c = 0xFF;           // hex
let d = 0b1010;         // binary
let e = 0o17;           // octal
```

## Float

```rust
let x: f32 = 3.14;
let y: f64 = 2.71828;  // default
```

## Boolean

```rust
let t: bool = true;
let f: bool = false;
```

## Character

```rust
let c: char = 'A';
let emoji: char = '🦀';  // Unicode!
```

## Tuple

```rust
let t: (i32, f64, bool) = (42, 3.14, true);
let (x, y, z) = t;          // destructure
println!("{}", t.0);         // akses by index
```

## Array

```rust
let arr: [i32; 5] = [1, 2, 3, 4, 5];
let zeros = [0; 10];         // [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
println!("{}", arr[0]);      // akses by index
println!("{}", arr.len());   // 5
```

## Slice

```rust
let arr = [1, 2, 3, 4, 5];
let slice: &[i32] = &arr[1..3]; // [2, 3]
let semua: &[i32] = &arr[..];   // semua element
```

## Type Casting

```rust
let x: i32 = 42;
let y = x as f64;        // i32 → f64
let z = x as u8;         // i32 → u8 (truncate!)
let s = x.to_string();   // i32 → String
let n: i32 = "42".parse().unwrap(); // String → i32
```

---

# Operators

## Arithmetic

```
+    Tambah
-    Tolak
*    Darab
/    Bahagi (integer div untuk int)
%    Modulo / Remainder
**   (tiada! guna .pow())
```

## Comparison

```
==   Equal
!=   Not equal
<    Less than
>    Greater than
<=   Less than or equal
>=   Greater than or equal
```

## Logical

```
&&   AND (short-circuit)
||   OR  (short-circuit)
!    NOT
```

## Bitwise

```
&    AND
|    OR
^    XOR
!    NOT
<<   Shift left
>>   Shift right
```

## Assignment

```
=    Assign
+=   Add assign
-=   Sub assign
*=   Mul assign
/=   Div assign
%=   Mod assign
&=   Bitwise AND assign
|=   Bitwise OR assign
^=   Bitwise XOR assign
<<=  Shift left assign
>>=  Shift right assign
```

## Lain-lain

```
..    Range (exclusive): 0..5 = 0,1,2,3,4
..=   Range (inclusive): 0..=5 = 0,1,2,3,4,5
?     Error propagation
&     Reference / borrow
&mut  Mutable reference
*     Dereference
::    Path separator
.     Field / method access
->    Return type
=>    Match arm
|     Closure parameter / OR pattern
@     Binding in pattern
_     Wildcard
```

## Operator Precedence (tinggi ke rendah)

```
1.  Method call, field access:   .
2.  Unary:                       -x, !x, *x, &x, &mut x
3.  Dereference:                 as
4.  Multiply / Divide:           * / %
5.  Add / Subtract:              + -
6.  Shift:                       << >>
7.  Bitwise AND:                 &
8.  Bitwise XOR:                 ^
9.  Bitwise OR:                  |
10. Comparison:                  == != < > <= >=
11. Logical AND:                 &&
12. Logical OR:                  ||
13. Range:                       .. ..=
14. Assignment:                  = += -= ...
15. Return / break / continue:   return break continue
```

---

# Strings

## Dua Jenis String

```
&str    → String slice, immutable, borrowed, stack
String  → Owned, mutable, heap allocated
```

## Buat String

```rust
let s1: &str    = "Hello";                    // literal
let s2: String  = String::new();              // empty
let s3: String  = String::from("Hello");      // from literal
let s4: String  = "Hello".to_string();        // dari &str
let s5: String  = format!("{} {}", "Hello", "World");
```

## Convert

```rust
let owned: String = s1.to_string();           // &str → String
let owned: String = s1.to_owned();            // &str → String
let borrow: &str  = &s3;                      // String → &str
let borrow: &str  = s3.as_str();              // String → &str
```

## String Methods

```rust
let s = String::from("Hello, World!");

s.len()                    // 13 (bytes, bukan chars!)
s.is_empty()               // false
s.contains("World")        // true
s.starts_with("Hello")     // true
s.ends_with("!")           // true
s.to_uppercase()           // "HELLO, WORLD!"
s.to_lowercase()           // "hello, world!"
s.trim()                   // buang whitespace depan/belakang
s.trim_start()             // buang whitespace depan
s.trim_end()               // buang whitespace belakang
s.replace("World", "Rust") // "Hello, Rust!"
s.replacen("l", "L", 2)   // ganti 2 kali pertama
s.split(", ")              // iterator splits
s.split_whitespace()       // split by whitespace
s.chars()                  // iterator atas chars (Unicode!)
s.bytes()                  // iterator atas bytes
s.lines()                  // iterator atas baris
s.find("World")            // Option<usize>
s.rfind("l")               // Option<usize> (dari kanan)
s.parse::<i32>()           // Result<i32, _>
s.repeat(3)                // "Hello!Hello!Hello!"
s.capacity()               // kapasiti buffer

// Slice
let slice = &s[0..5];      // "Hello"

// Concat
let s = "Hello".to_string() + " World"; // guna +
let s = format!("{} {}", "Hello", "World"); // format!
```

## String Append

```rust
let mut s = String::from("Hello");
s.push(' ');              // tambah char
s.push_str("World");      // tambah &str
s += " Rust";             // += operator
```

---

# Functions

```rust
// Asas
fn salam() {
    println!("Halo!");
}

// Dengan parameter
fn tambah(a: i32, b: i32) -> i32 {
    a + b           // tiada semicolon = return value
}

// Dengan return eksplisit
fn bahagi(a: f64, b: f64) -> f64 {
    if b == 0.0 {
        return 0.0; // return awal
    }
    a / b
}

// Multiple return (tuple)
fn min_maks(v: &[i32]) -> (i32, i32) {
    let min = *v.iter().min().unwrap();
    let max = *v.iter().max().unwrap();
    (min, max)
}

// Generic function
fn terbesar<T: PartialOrd>(a: T, b: T) -> T {
    if a > b { a } else { b }
}

// Dengan where clause
fn cetak<T>(x: T)
where
    T: std::fmt::Display + std::fmt::Debug,
{
    println!("{} {:?}", x, x);
}

// Never return type
fn crash() -> ! {
    panic!("program berhenti!");
}

// Associated function (macam static method)
struct Pekerja { nama: String }
impl Pekerja {
    fn baru(nama: &str) -> Self {      // → Self, bukan &self
        Pekerja { nama: nama.into() }
    }
}
```

---

# Control Structures

## If / Else

```rust
// Asas
if x > 0 {
    println!("positif");
} else if x < 0 {
    println!("negatif");
} else {
    println!("sifar");
}

// If sebagai expression
let label = if x > 0 { "positif" } else { "negatif" };
```

## Loop

```rust
// loop — infinite, perlu break
let hasil = loop {
    kiraan += 1;
    if kiraan == 10 {
        break kiraan * 2; // break dengan nilai
    }
};

// while
while x > 0 {
    x -= 1;
}

// while let
while let Some(nilai) = iter.next() {
    println!("{}", nilai);
}

// for
for i in 0..5 { print!("{} ", i); }       // 0 1 2 3 4
for i in 0..=5 { print!("{} ", i); }      // 0 1 2 3 4 5
for item in &vec { println!("{}", item); } // borrow
for item in vec { println!("{}", item); }  // consume

// Enumerate
for (i, val) in vec.iter().enumerate() {
    println!("{}: {}", i, val);
}

// Loop label (nested break)
'luar: for i in 0..5 {
    for j in 0..5 {
        if i == j { break 'luar; }
    }
}
```

## Match

```rust
match x {
    1       => println!("satu"),
    2 | 3   => println!("dua atau tiga"),
    4..=10  => println!("empat hingga sepuluh"),
    n       => println!("lain: {}", n), // binding
    _       => println!("wildcard"),
}

// Match sebagai expression
let teks = match x {
    1 => "satu",
    2 => "dua",
    _ => "lain",
};

// Match dengan guard
match x {
    n if n < 0  => println!("negatif"),
    n if n == 0 => println!("sifar"),
    n           => println!("positif: {}", n),
}

// Destructure dalam match
match tuple {
    (0, 0) => println!("origin"),
    (x, 0) => println!("x-axis: {}", x),
    (0, y) => println!("y-axis: {}", y),
    (x, y) => println!("({}, {})", x, y),
}
```

## if let / while let / let else

```rust
// if let
if let Some(nilai) = option {
    println!("{}", nilai);
}

if let Ok(n) = "42".parse::<i32>() {
    println!("{}", n);
}

// while let
while let Some(item) = stack.pop() {
    println!("{}", item);
}

// let else (Rust 1.65+)
let Some(nilai) = option else {
    return; // atau panic!, break, continue
};
// nilai tersedia di sini
```

---

# Ownership

```
Peraturan Ownership:
  1. Setiap nilai ada SATU owner
  2. Hanya SATU owner pada satu masa
  3. Bila owner keluar scope → nilai di-drop
```

```rust
// Move
let s1 = String::from("hello");
let s2 = s1;             // s1 tidak valid lagi!
// println!("{}", s1);  // ERROR

// Copy (untuk Copy types: i32, f64, bool, char, &T, array/tuple of Copy)
let x = 5;
let y = x;               // x masih valid (Copy)
println!("{}", x);       // OK

// Clone
let s1 = String::from("hello");
let s2 = s1.clone();     // deep copy — mahal!
println!("{}", s1);      // OK

// Move ke fungsi
fn ambil(s: String) { } // s masuk, s di-drop selepas fungsi
let s = String::from("hello");
ambil(s);
// println!("{}", s);   // ERROR — s sudah move

// Return ownership
fn beri() -> String { String::from("hello") }
let s = beri();           // dapat ownership
```

---

# Borrowing

```
Peraturan Borrow:
  1. BANYAK &T (immutable) ATAU SATU &mut T
  2. TIDAK KEDUA-DUANYA serentak
  3. Reference mesti sentiasa valid
```

```rust
// Immutable borrow
fn panjang(s: &String) -> usize { s.len() }
let s = String::from("hello");
let len = panjang(&s);
println!("{}", s); // masih valid!

// Mutable borrow
fn tambah(s: &mut String) { s.push_str(" world"); }
let mut s = String::from("hello");
tambah(&mut s);

// Tidak boleh: dua mutable borrow serentak
let mut s = String::from("hello");
let r1 = &mut s;
// let r2 = &mut s; // ERROR!

// Tidak boleh: immutable + mutable serentak
let r1 = &s;
let r2 = &s;       // OK — banyak immutable boleh
// let r3 = &mut s; // ERROR! ada immutable aktif
println!("{} {}", r1, r2); // r1, r2 tamat di sini
let r3 = &mut s;   // OK sekarang

// Slice borrow
let arr = [1, 2, 3, 4, 5];
let slice = &arr[1..3];    // [2, 3]
```

---

# Pattern Matching

```rust
// Struct destructure
struct Titik { x: i32, y: i32 }
let Titik { x, y } = titik;

// Enum destructure
enum Mesej {
    Keluar,
    Pindah { x: i32, y: i32 },
    Tulis(String),
}

match mesej {
    Mesej::Keluar            => println!("Keluar"),
    Mesej::Pindah { x, y }  => println!("{} {}", x, y),
    Mesej::Tulis(teks)       => println!("{}", teks),
}

// @ binding
match x {
    n @ 1..=10 => println!("dalam julat: {}", n),
    _          => println!("luar julat"),
}

// Tuple struct
struct Warna(i32, i32, i32);
let Warna(r, g, b) = warna;

// Nested
match ((1, 2), (3, 4)) {
    ((x, _), (_, y)) => println!("{} {}", x, y), // 1 4
}

// Or pattern
match x {
    1 | 2 | 3 => println!("kecil"),
    _         => println!("besar"),
}

// Ref dalam pattern
match &nilai {
    ref v => println!("{}", v), // v adalah &T
}
```

---

# Option

```rust
// Buat
let ada:   Option<i32> = Some(42);
let tiada: Option<i32> = None;

// Semak
ada.is_some()                    // true
ada.is_none()                    // false

// Ambil nilai
ada.unwrap()                     // 42 atau PANIC
ada.expect("mesej")              // 42 atau PANIC+mesej
ada.unwrap_or(0)                 // 42 atau 0
ada.unwrap_or_else(|| kira())    // 42 atau hasil kira()
ada.unwrap_or_default()          // 42 atau Default

// Transform
ada.map(|n| n * 2)               // Some(84)
ada.map_or(0, |n| n * 2)        // 84
ada.and_then(|n| Some(n * 2))   // Some(84) (flatmap)
ada.filter(|&n| n > 0)          // Some(42)
ada.or(Some(0))                  // Some(42)
ada.or_else(|| Some(0))         // Some(42)
ada.zip(Some("str"))             // Some((42, "str"))
ada.flatten()                    // untuk Option<Option<T>>

// Convert
ada.ok_or("ralat")              // Ok(42)
result.ok()                      // Option (buang error)

// Pattern
match ada {
    Some(n) => println!("{}", n),
    None    => println!("tiada"),
}

if let Some(n) = ada { println!("{}", n); }
let Some(n) = ada else { return; }; // let else

// ? operator dalam Option fn
fn cari() -> Option<i32> {
    let n = option_fn()?; // return None kalau None
    Some(n + 1)
}
```

---

# Result

```rust
// Buat
let ok:  Result<i32, String> = Ok(42);
let err: Result<i32, String> = Err("gagal".into());

// Semak
ok.is_ok()                      // true
ok.is_err()                     // false

// Ambil nilai
ok.unwrap()                     // 42 atau PANIC
ok.expect("mesej")              // 42 atau PANIC+mesej
ok.unwrap_or(0)                 // 42 atau 0
ok.unwrap_or_else(|_| 0)       // 42 atau 0
ok.unwrap_or_default()          // Default

// Transform
ok.map(|n| n * 2)               // Ok(84)
ok.map_err(|e| e.len())        // Ok(42) — transform error
ok.and_then(|n| Ok(n * 2))    // Ok(84)
ok.or_else(|_| Ok(0))          // Ok(42)

// Convert
ok.ok()                         // Some(42)
ok.err()                        // None

// Pattern
match ok {
    Ok(n)  => println!("{}", n),
    Err(e) => println!("ralat: {}", e),
}

// ? operator — propagate error
fn proses() -> Result<i32, String> {
    let n = boleh_gagal()?;  // return Err kalau Err
    Ok(n + 1)
}

// Collect Vec<Result> → Result<Vec>
let hasil: Result<Vec<i32>, _> = vec!["1","2","3"]
    .iter()
    .map(|s| s.parse::<i32>())
    .collect();
```

---

# Closures

```rust
// Asas
let tambah = |a, b| a + b;
let hasil = tambah(2, 3); // 5

// Dengan type annotation
let tambah = |a: i32, b: i32| -> i32 { a + b };

// Capture environment
let x = 5;
let tambah_x = |n| n + x;    // borrow x
println!("{}", tambah_x(3));  // 8

// Move capture
let s = String::from("hello");
let cetak = move || println!("{}", s); // s di-move ke closure
cetak();
// println!("{}", s); // ERROR — s sudah move

// Tiga jenis trait closure:
// Fn    — boleh dipanggil berkali-kali, borrow immutable
// FnMut — boleh dipanggil berkali-kali, borrow mutable
// FnOnce — boleh dipanggil SEKALI, consume capture

// Dalam fungsi
fn guna<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 { f(x) }
fn guna_dyn(f: &dyn Fn(i32) -> i32, x: i32) -> i32 { f(x) }
fn simpan(f: Box<dyn Fn()>) { f(); }
```

---

# Iterators

```rust
let v = vec![1, 2, 3, 4, 5];

// Buat iterator
v.iter()          // borrow, yield &T
v.iter_mut()      // mutable borrow, yield &mut T
v.into_iter()     // consume, yield T

// Adapters (lazy)
.map(|x| x * 2)
.filter(|&&x| x > 2)
.filter_map(|x| if x > 2 { Some(x * 2) } else { None })
.flat_map(|x| vec![x, x * 2])
.flatten()
.take(3)
.skip(2)
.take_while(|&&x| x < 4)
.skip_while(|&&x| x < 3)
.enumerate()           // (index, item)
.zip(other_iter)       // (item1, item2)
.chain(other_iter)     // sambung iterator
.peekable()            // boleh peek
.rev()                 // reverse (perlu DoubleEndedIterator)
.step_by(2)            // setiap 2 langkah
.cloned()              // &T → T (clone)
.copied()              // &T → T (copy)
.inspect(|x| dbg!(x)) // debug tanpa consume

// Consumers
.collect::<Vec<_>>()
.collect::<String>()
.collect::<HashMap<_,_>>()
.sum::<i32>()
.product::<i32>()
.count()
.min()              // Option<&T>
.max()              // Option<&T>
.min_by_key(|x| x.field)
.max_by_key(|x| x.field)
.any(|&x| x > 3)    // bool
.all(|&x| x > 0)    // bool
.find(|&&x| x > 3)  // Option<&T>
.position(|&x| x > 3) // Option<usize>
.fold(0, |acc, x| acc + x)
.reduce(|acc, x| acc + x) // Option<T>
.for_each(|x| println!("{}", x))
.last()             // Option<T>
.nth(2)             // Option<T>
.unzip()            // (Vec<A>, Vec<B>)

// Custom iterator
struct Kiraan(u32);
impl Iterator for Kiraan {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        self.0 += 1;
        if self.0 < 6 { Some(self.0) } else { None }
    }
}
```

---

# Error Handling

```rust
// Panic — untuk bugs, bukan expected errors
panic!("sesuatu yang tidak sepatutnya berlaku");
assert!(x > 0, "x mesti positif");
assert_eq!(a, b, "a dan b mesti sama");

// Result — untuk expected errors yang boleh di-handle
fn baca_fail(laluan: &str) -> Result<String, std::io::Error> {
    std::fs::read_to_string(laluan)
}

// ? operator
fn proses() -> Result<(), Box<dyn std::error::Error>> {
    let kandungan = std::fs::read_to_string("fail.txt")?;
    let n: i32 = kandungan.trim().parse()?;
    println!("{}", n);
    Ok(())
}

// Custom error dengan thiserror
use thiserror::Error;
#[derive(Debug, Error)]
enum AppError {
    #[error("Tidak dijumpai: {0}")]
    TidakJumpai(String),
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
}

// anyhow untuk application code
use anyhow::{Result, Context, bail, ensure};

fn app() -> Result<()> {
    let n = "abc".parse::<i32>()
        .context("Gagal parse nombor")?;
    ensure!(n > 0, "Nombor mesti positif");
    if n > 100 { bail!("Terlalu besar: {}", n); }
    Ok(())
}

// map_err
let hasil = "abc".parse::<i32>()
    .map_err(|e| format!("Parse gagal: {}", e));
```

---

# Structs

```rust
// Regular struct
struct Pekerja {
    id:       u32,
    nama:     String,
    gaji:     f64,
    aktif:    bool,
}

// Buat
let mut p = Pekerja {
    id:    1,
    nama:  String::from("Ali"),
    gaji:  4500.0,
    aktif: true,
};

// Akses
println!("{}", p.nama);
p.gaji = 5000.0;

// Struct update syntax
let p2 = Pekerja { id: 2, nama: "Siti".into(), ..p };

// Tuple struct
struct Titik(f64, f64);
let t = Titik(1.0, 2.0);
println!("{}", t.0); // akses by index

// Unit struct
struct Penanda;
let p = Penanda;

// Implement methods
impl Pekerja {
    // Associated function (constructor)
    fn baru(id: u32, nama: &str, gaji: f64) -> Self {
        Pekerja { id, nama: nama.into(), gaji, aktif: true }
    }

    // Method
    fn salam(&self) -> String {
        format!("Halo, saya {}", self.nama)
    }

    // Mutable method
    fn naikkan_gaji(&mut self, peratus: f64) {
        self.gaji *= 1.0 + peratus / 100.0;
    }

    // Consuming method
    fn ke_string(self) -> String {
        format!("{}: RM{}", self.nama, self.gaji)
    }
}

// Derive macros
#[derive(Debug, Clone, PartialEq, Eq, Hash, Default)]
struct Data { x: i32, y: i32 }
```

---

# Enums

```rust
// Asas
enum Arah { Utara, Selatan, Timur, Barat }

// Dengan data
enum Mesej {
    Keluar,
    Pindah { x: i32, y: i32 },    // named fields
    Tulis(String),                  // tuple-like
    Warna(u8, u8, u8),             // multiple values
}

// Implement methods
impl Mesej {
    fn proses(&self) {
        match self {
            Mesej::Keluar           => println!("Keluar"),
            Mesej::Pindah { x, y } => println!("{},{}", x, y),
            Mesej::Tulis(s)        => println!("{}", s),
            Mesej::Warna(r, g, b)  => println!("#{:02X}{:02X}{:02X}", r, g, b),
        }
    }
}

// Option<T> — enum terbina dalam
enum Option<T> { Some(T), None }

// Result<T,E> — enum terbina dalam
enum Result<T, E> { Ok(T), Err(E) }

// Derive
#[derive(Debug, Clone, PartialEq)]
enum Status { Aktif, TidakAktif, Tangguh }
```

---

# Traits

```rust
// Define trait
trait BolehSalam {
    fn salam(&self) -> String;

    // Default method
    fn salam_rasmi(&self) -> String {
        format!("Selamat datang, {}!", self.salam())
    }
}

// Implement
struct Pekerja { nama: String }
impl BolehSalam for Pekerja {
    fn salam(&self) -> String { self.nama.clone() }
    // salam_rasmi() ada default — tidak perlu implement
}

// Trait sebagai parameter
fn cetak_salam(x: &impl BolehSalam) {       // impl Trait
    println!("{}", x.salam());
}
fn cetak_salam<T: BolehSalam>(x: &T) {     // generic bound
    println!("{}", x.salam());
}
fn cetak_salam(x: &dyn BolehSalam) {       // trait object
    println!("{}", x.salam());
}

// Multiple bounds
fn proses<T: Clone + std::fmt::Debug>(x: T) { }
fn proses<T>(x: T) where T: Clone + std::fmt::Debug { }

// Return impl Trait
fn buat_salam() -> impl BolehSalam { Pekerja { nama: "Ali".into() } }

// Trait objects (dynamic dispatch)
let list: Vec<Box<dyn BolehSalam>> = vec![
    Box::new(Pekerja { nama: "Ali".into() }),
];

// Standard traits biasa
#[derive(Debug)]     // fmt::Debug — {:?}
#[derive(Display)]   // fmt::Display — {} (manual impl)
#[derive(Clone)]     // .clone()
#[derive(Copy)]      // auto-copy (perlu Clone)
#[derive(PartialEq)] // == dan !=
#[derive(Eq)]        // guarantees full equality
#[derive(PartialOrd)]// <, >, <=, >=
#[derive(Ord)]       // full ordering
#[derive(Hash)]      // untuk HashMap key
#[derive(Default)]   // ::default()
```

---

# Generics

```rust
// Generic function
fn terbesar<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list.iter() {
        if item > largest { largest = item; }
    }
    largest
}

// Generic struct
struct Pasangan<T, U> {
    pertama:  T,
    kedua:    U,
}

// Generic impl
impl<T: std::fmt::Display, U: std::fmt::Display> Pasangan<T, U> {
    fn papar(&self) {
        println!("{} {}", self.pertama, self.kedua);
    }
}

// Generic enum
enum Sama<T> { Ada(T), Tiada }

// Const generic
fn panjang_array<const N: usize>(arr: &[i32; N]) -> usize { N }

// Associated types dalam trait
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

// Trait alias (workaround)
trait KlonPapar: Clone + std::fmt::Display {}
impl<T: Clone + std::fmt::Display> KlonPapar for T {}
```

---

# Lifetimes

```rust
// Lifetime annotation
fn terpanjang<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() >= s2.len() { s1 } else { s2 }
}

// Struct dengan lifetime
struct Petikan<'a> {
    teks: &'a str,
}
impl<'a> Petikan<'a> {
    fn papar(&self) -> &str { self.teks }
}

// Multiple lifetimes
fn gabung<'a, 'b>(s1: &'a str, _s2: &'b str) -> &'a str { s1 }

// 'static — hidup sepanjang program
let s: &'static str = "teks literal";

// Lifetime elision rules (tidak perlu tulis bila):
// 1. Setiap reference parameter dapat lifetime sendiri
// 2. Kalau satu input lifetime → assign ke semua output
// 3. Kalau ada &self/&mut self → assign ke output

fn pertama(s: &str) -> &str { &s[0..1] } // OK tanpa annotation
```

---

# Modules

```rust
// Dalam fail yang sama
mod matematik {
    pub fn tambah(a: i32, b: i32) -> i32 { a + b }

    mod peribadi { // private by default
        pub fn rahsia() -> &'static str { "rahsia" }
    }
}

// Guna
use matematik::tambah;
let n = tambah(2, 3);

// Fail berasingan: src/matematik.rs
// Atau folder: src/matematik/mod.rs

// Re-export
pub use matematik::tambah;

// Import pelbagai
use std::collections::{HashMap, HashSet};
use std::io::{self, Read, Write};

// Wildcard (guna berhati-hati)
use matematik::*;

// Alias
use std::collections::HashMap as Map;

// Nested path
use crate::modul::{
    SubModul,
    fungsi_ini,
    Struct::{ kaedah_a, kaedah_b },
};

// Super
mod induk {
    fn induk_fn() {}
    mod anak {
        use super::induk_fn; // akses induk
    }
}

// crate — root
use crate::modul::Sesuatu;
```

---

# Collections

## Vec\<T\>

```rust
let mut v: Vec<i32> = Vec::new();
let mut v = vec![1, 2, 3];

v.push(4);
v.pop()              // Option<i32>
v.insert(1, 99)      // masuk pada index
v.remove(1)          // buang & return element
v.len()
v.is_empty()
v.contains(&3)
v.get(0)             // Option<&i32>
v[0]                 // &i32 (PANIC kalau OOB)
v.first()            // Option<&i32>
v.last()             // Option<&i32>
v.iter()             // iterator
v.sort()
v.sort_by(|a, b| a.cmp(b))
v.sort_by_key(|x| x.field)
v.reverse()
v.dedup()            // buang consecutive duplicates
v.retain(|&x| x > 0)
v.extend([4, 5, 6])
v.append(&mut v2)
v.clear()
v.truncate(3)
v.drain(1..3)        // buang dan return
v.split_at(2)        // (&[T], &[T])
v.windows(3)         // overlapping slices
v.chunks(2)          // non-overlapping slices
```

## HashMap\<K, V\>

```rust
use std::collections::HashMap;
let mut map: HashMap<String, i32> = HashMap::new();

map.insert("a".into(), 1);
map.get("a")              // Option<&i32>
map.get_mut("a")          // Option<&mut i32>
map["a"]                  // i32 (PANIC kalau tiada)
map.contains_key("a")     // bool
map.remove("a")           // Option<i32>
map.len()
map.is_empty()
map.keys()                // iterator
map.values()              // iterator
map.values_mut()          // mutable iterator
map.iter()                // (key, value) iterator

// Entry API
map.entry("b".into()).or_insert(0);
map.entry("b".into()).or_default();
map.entry("b".into()).or_insert_with(|| 42);
map.entry("b".into()).and_modify(|v| *v += 1).or_insert(0);

// Buat dari iterator
let map: HashMap<_, _> = vec![("a", 1), ("b", 2)]
    .into_iter().collect();
```

## HashSet\<T\>

```rust
use std::collections::HashSet;
let mut set: HashSet<i32> = HashSet::new();

set.insert(1);
set.contains(&1)    // bool
set.remove(&1)      // bool
set.len()
set.is_empty()

// Set operations
set.union(&other)
set.intersection(&other)
set.difference(&other)
set.symmetric_difference(&other)
set.is_subset(&other)
set.is_superset(&other)
```

## VecDeque\<T\>

```rust
use std::collections::VecDeque;
let mut dq: VecDeque<i32> = VecDeque::new();

dq.push_back(1)     // masuk belakang
dq.push_front(0)    // masuk depan
dq.pop_back()       // Option<T>
dq.pop_front()      // Option<T>
dq.front()          // Option<&T>
dq.back()           // Option<&T>
```

## BinaryHeap\<T\>

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

let mut heap: BinaryHeap<i32> = BinaryHeap::new();
heap.push(3); heap.push(1); heap.push(4);
heap.pop()         // Some(4) — max keluar dulu!
heap.peek()        // Option<&T>

// Min-heap
let mut min: BinaryHeap<Reverse<i32>> = BinaryHeap::new();
min.push(Reverse(3));
min.pop()          // Some(Reverse(1)) — min keluar dulu
```

---

# Files & IO

```rust
use std::fs;
use std::io::{self, Read, Write, BufRead, BufReader, BufWriter};
use std::path::Path;

// Baca fail (mudah)
let kandungan = fs::read_to_string("fail.txt")?;

// Baca fail (bytes)
let bytes = fs::read("fail.txt")?;

// Tulis fail
fs::write("fail.txt", "kandungan")?;
fs::write("fail.txt", bytes)?;

// Append
let mut fail = fs::OpenOptions::new()
    .append(true)
    .open("fail.txt")?;
writeln!(fail, "baris baru")?;

// Baca baris per baris
let fail = fs::File::open("fail.txt")?;
let pembaca = BufReader::new(fail);
for baris in pembaca.lines() {
    let baris = baris?;
    println!("{}", baris);
}

// Tulis dengan buffer
let fail = fs::File::create("fail.txt")?;
let mut penulis = BufWriter::new(fail);
writeln!(penulis, "baris 1")?;
penulis.flush()?;

// Path operations
let laluan = Path::new("/tmp/fail.txt");
laluan.exists()
laluan.is_file()
laluan.is_dir()
laluan.extension()        // Option<&OsStr>
laluan.file_name()        // Option<&OsStr>
laluan.parent()           // Option<&Path>

// PathBuf (owned)
use std::path::PathBuf;
let mut pb = PathBuf::from("/tmp");
pb.push("subfolder");
pb.push("fail.txt");

// Directory operations
fs::create_dir("folder")?;
fs::create_dir_all("a/b/c")?;
fs::remove_file("fail.txt")?;
fs::remove_dir("folder")?;
fs::remove_dir_all("folder")?;
fs::copy("src.txt", "dst.txt")?;
fs::rename("lama.txt", "baru.txt")?;
fs::read_dir(".")?               // iterator
```

---

# Smart Pointers

```rust
// Box<T> — heap allocation
let b = Box::new(5);
println!("{}", *b); // dereference

// Recursive type
enum Senarai {
    Nod(i32, Box<Senarai>),
    Akhir,
}

// Rc<T> — reference counting (single thread)
use std::rc::Rc;
let a = Rc::new(5);
let b = Rc::clone(&a);   // clone pointer, bukan data
println!("{}", Rc::strong_count(&a)); // 2

// Arc<T> — atomic Rc (multi thread)
use std::sync::Arc;
let a = Arc::new(5);
let b = Arc::clone(&a);

// RefCell<T> — runtime borrow checking
use std::cell::RefCell;
let rf = RefCell::new(5);
*rf.borrow_mut() += 1;    // mutable borrow
println!("{}", *rf.borrow()); // immutable borrow

// Rc<RefCell<T>> — shared mutable (single thread)
let shared = Rc::new(RefCell::new(vec![1, 2, 3]));
shared.borrow_mut().push(4);

// Arc<Mutex<T>> — shared mutable (multi thread)
use std::sync::Mutex;
let shared = Arc::new(Mutex::new(0));
*shared.lock().unwrap() += 1;

// Cell<T> — interior mutability untuk Copy types
use std::cell::Cell;
let c = Cell::new(5);
c.set(10);
println!("{}", c.get());

// Weak<T> — non-owning reference (break cycles)
use std::rc::Weak;
let strong = Rc::new(5);
let weak: Weak<i32> = Rc::downgrade(&strong);
if let Some(val) = weak.upgrade() { println!("{}", val); }

// Cow<'a, B> — clone on write
use std::borrow::Cow;
fn process(s: &str) -> Cow<str> {
    if s.contains(' ') {
        Cow::Owned(s.replace(' ', "_"))
    } else {
        Cow::Borrowed(s)
    }
}
```

---

# Concurrency

```rust
use std::thread;
use std::sync::{Arc, Mutex, RwLock};
use std::sync::mpsc;

// Spawn thread
let handle = thread::spawn(|| {
    println!("thread baru!");
});
handle.join().unwrap(); // tunggu selesai

// Move closure
let v = vec![1, 2, 3];
let h = thread::spawn(move || println!("{:?}", v));
h.join().unwrap();

// Mutex
let m = Arc::new(Mutex::new(0));
let m1 = Arc::clone(&m);
thread::spawn(move || {
    *m1.lock().unwrap() += 1;
}).join().unwrap();

// RwLock
let rw = Arc::new(RwLock::new(vec![1,2,3]));
let r = rw.read().unwrap();   // banyak reader OK
let w = rw.write().unwrap();  // exclusive writer

// Channel (mpsc)
let (tx, rx) = mpsc::channel::<i32>();
let tx1 = tx.clone();
thread::spawn(move || tx1.send(1).unwrap());
thread::spawn(move || tx.send(2).unwrap());
println!("{}", rx.recv().unwrap());

// Bounded channel
let (tx, rx) = mpsc::sync_channel::<i32>(5);

// Atomic
use std::sync::atomic::{AtomicU64, Ordering};
let kiraan = Arc::new(AtomicU64::new(0));
kiraan.fetch_add(1, Ordering::Relaxed);
kiraan.load(Ordering::Relaxed);

// Thread safety traits
// Send: boleh hantar ke thread lain
// Sync: boleh share (&T) antara thread
```

---

# Attributes

```rust
// Derive
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]

// Allow/Deny lints
#[allow(unused_variables)]
#[deny(unused_imports)]
#[warn(dead_code)]

// Conditional compilation
#[cfg(test)]
#[cfg(debug_assertions)]
#[cfg(feature = "mysql")]
#[cfg(target_os = "linux")]

// Test
#[test]
#[should_panic]
#[should_panic(expected = "mesej")]
#[ignore]

// Inline
#[inline]
#[inline(always)]
#[inline(never)]

// Deprecation
#[deprecated(since = "1.0.0", note = "Guna fungsi_baru()")]

// Repr
#[repr(C)]        // C-compatible layout
#[repr(u8)]       // enum dengan u8 discriminant

// Serde
#[serde(rename = "nama_lain")]
#[serde(skip)]
#[serde(default)]
#[serde(with = "modul_custom")]

// Macro/proc-macro
#[macro_export]
#[proc_macro]

// Feature gate
#![feature(specialization)] // nightly sahaja
```

---

# Testing

```rust
// Unit test dalam mod
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_tambah() {
        assert_eq!(tambah(2, 3), 5);
    }

    #[test]
    #[should_panic(expected = "overflow")]
    fn test_overflow() {
        i32::MAX + 1; // akan panic
    }

    #[test]
    fn test_result() -> Result<(), String> {
        let n: i32 = "42".parse().map_err(|e| e.to_string())?;
        assert_eq!(n, 42);
        Ok(())
    }

    #[test]
    #[ignore = "lambat"]
    fn test_lambat() { /* ... */ }
}

// Macros assertion
assert!(expr)
assert!(expr, "mesej {}", val)
assert_eq!(a, b)
assert_eq!(a, b, "mesej")
assert_ne!(a, b)
assert!(matches!(opt, Some(n) if n > 0))
```

---

# Cargo

## Perintah Utama

```bash
# Projek baru
cargo new nama_projek          # binary
cargo new nama_projek --lib    # library

# Build
cargo build                    # debug
cargo build --release          # optimized
cargo check                    # semak tanpa compile
cargo clean                    # padam target/

# Run
cargo run                      # debug
cargo run --release            # optimized
cargo run --bin nama_binary    # specific binary

# Test
cargo test                     # semua test
cargo test nama_test           # filter
cargo test -- --nocapture      # tunjuk println!
cargo test -- --ignored        # test yg di-ignore

# Docs
cargo doc                      # jana docs
cargo doc --open               # buka dalam browser

# Dependencies
cargo add serde                # tambah dependency
cargo add serde --features derive
cargo update                   # update dependencies

# Lints
cargo clippy                   # linter
cargo fmt                      # formatter

# Audit
cargo audit                    # check vulnerabilities
cargo tree                     # dependency tree
```

## Cargo.toml

```toml
[package]
name    = "nama-projek"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
axum  = "0.7"

[dev-dependencies]
mockall = "0.12"

[build-dependencies]
build-script = "0.1"

[features]
default  = []
mysql    = ["sqlx/mysql"]
postgres = ["sqlx/postgres"]

[profile.release]
opt-level = 3
lto       = true

[workspace]
members = ["crate-a", "crate-b"]
```

---

# Macros Biasa

```rust
// Output
print!("tanpa newline");
println!("dengan newline");
eprintln!("ke stderr");

// Format
format!("Hello, {}!", nama)
format!("{:?}", debug_value)
format!("{:#?}", pretty_debug)
format!("{:05}", n)           // zero-padded
format!("{:.2}", f)           // 2 decimal places
format!("{:>10}", s)          // align kanan
format!("{:<10}", s)          // align kiri
format!("{:^10}", s)          // tengah
format!("{:b}", n)            // binary
format!("{:x}", n)            // hex lowercase
format!("{:X}", n)            // hex uppercase
format!("{:o}", n)            // octal
format!("{:e}", f)            // scientific notation

// Collections
vec![1, 2, 3]
vec![0; 10]           // 10 zeros

// Assertion
assert!(condition)
assert_eq!(a, b)
assert_ne!(a, b)
debug_assert!(condition)  // only in debug

// Panic
panic!("mesej")
todo!()                   // placeholder, will panic
unimplemented!()          // not implemented
unreachable!()            // should never reach here

// Debug
dbg!(expression)          // print file:line = value
dbg!(&vec)

// Environment
env!("CARGO_PKG_VERSION") // compile time
option_env!("VAR")        // Option<&str>
include_str!("fail.txt")  // embed file as &str
include_bytes!("img.png") // embed file as &[u8]
concat!("a", "b", "c")   // "abc" at compile time
stringify!(expression)    // expression as &str

// Write to buffer
let mut s = String::new();
write!(s, "{}", nilai)?;
writeln!(s, "{}", nilai)?;
```

---

# Common Patterns

## Type Alias

```rust
type Hasil<T> = Result<T, AppError>;
type PetaGaji = HashMap<String, f64>;
```

## Newtype

```rust
struct Ringgit(f64);
struct PekerjaId(u32);
```

## Builder

```rust
struct Pembina { x: i32, y: i32 }
impl Pembina {
    fn baru() -> Self { Pembina { x: 0, y: 0 } }
    fn x(mut self, x: i32) -> Self { self.x = x; self }
    fn y(mut self, y: i32) -> Self { self.y = y; self }
    fn bina(self) -> Struct { Struct { x: self.x, y: self.y } }
}
```

## RAII Guard

```rust
struct Guard(Resource);
impl Drop for Guard {
    fn drop(&mut self) { self.0.cleanup(); }
}
```

---

# Quick Reference Table

| PHP | Rust |
|---|---|
| `null` | `None` |
| `?Type` | `Option<Type>` |
| `try-catch` | `match result` / `?` |
| `array` | `Vec<T>` / `HashMap<K,V>` |
| `class` | `struct` + `impl` |
| `interface` | `trait` |
| `extends` | composition |
| `implements` | `impl Trait for Struct` |
| `$this->` | `self.` |
| `static::` | `Self::` |
| `echo` | `print!` / `println!` |
| `var_dump` | `dbg!` / `{:?}` |
| `count($arr)` | `v.len()` |
| `array_push` | `v.push()` |
| `array_map` | `.map()` |
| `array_filter` | `.filter()` |
| `implode(",", $a)` | `a.join(",")` |
| `explode(",", $s)` | `s.split(",").collect()` |
| `strlen($s)` | `s.len()` |
| `str_contains` | `s.contains()` |
| `trim($s)` | `s.trim()` |

---

*quick reference untuk Rust* 🦀eference untuk Rust* 🦀