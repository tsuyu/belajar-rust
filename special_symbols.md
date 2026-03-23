# 🔣 Special Symbols dalam Rust — Panduan Lengkap

> Setiap simbol, operator, dan tanda baca dalam Rust — apa maksudnya dan
> cara gunakannya. Dari `::` hingga `..=`, dari `?` hingga `!`.
> Rujukan cepat untuk semua simbol yang mengelirukan.

---

## Kenapa Simbol Rust Mengelirukan?

```
Rust guna banyak simbol yang SAMA untuk maksud BERBEZA
bergantung pada KONTEKS.

Contoh simbol * :
  let x = *ptr;           // dereference
  let y = 5 * 3;          // pendaraban
  use std::io::*;         // glob import
  fn f<T: Clone + Debug>  // trait bound
  impl<T> MyTrait for T   // generic implementation

Contoh simbol & :
  let r = &x;             // borrow (immutable reference)
  fn f(s: &str)           // reference type
  let r = x & y;          // bitwise AND
  fn f<T: Clone + Debug>  // multiple trait bounds (itu +)

Contoh simbol ! :
  println!("hi")          // macro invocation
  let b = !true;          // logical NOT
  fn f() -> !             // never type (diverging)
  #![allow(dead_code)]    // inner attribute
```

---

## Peta Panduan

```
Bahagian 1  → Arithmetic Operators
Bahagian 2  → Comparison & Logical Operators
Bahagian 3  → Bitwise Operators
Bahagian 4  → Reference & Pointer Symbols
Bahagian 5  → Path & Module Symbols
Bahagian 6  → Type & Generic Symbols
Bahagian 7  → Pattern Matching Symbols
Bahagian 8  → Range Symbols
Bahagian 9  → Macro & Attribute Symbols
Bahagian 10 → Miscellaneous Symbols
Bahagian 11 → Lifetime Symbols
Bahagian 12 → Simbol dalam Konteks Berbeza
```

---

# BAHAGIAN 1: Arithmetic Operators ➕

```rust
fn main() {
    let a = 10i32;
    let b = 3i32;

    // ── Standard arithmetic ───────────────────────────────────
    println!("{}", a + b);   // 13  — tambah
    println!("{}", a - b);   // 7   — tolak
    println!("{}", a * b);   // 30  — darab
    println!("{}", a / b);   // 3   — bahagi (integer division, truncate!)
    println!("{}", a % b);   // 1   — modulo / remainder

    // ── Compound assignment ───────────────────────────────────
    let mut x = 10;
    x += 5;   // x = x + 5 = 15
    x -= 3;   // x = x - 3 = 12
    x *= 2;   // x = x * 2 = 24
    x /= 4;   // x = x / 4 = 6
    x %= 4;   // x = x % 4 = 2
    println!("{}", x); // 2

    // ── Unary minus ───────────────────────────────────────────
    let n = -5i32;           // negatif literal
    let m = -n;              // unary minus = 5
    println!("{}", m);

    // ── Float division ────────────────────────────────────────
    let f1 = 10.0f64;
    let f2 = 3.0f64;
    println!("{:.4}", f1 / f2); // 3.3333 (float division, tiada truncate)

    // PERHATIAN: Integer division truncates!
    println!("{}", 7 / 2);   // 3 (bukan 3.5!)
    println!("{}", -7 / 2);  // -3 (truncate toward zero, bukan floor!)
    println!("{}", -7 % 2);  // -1 (remainder, bukan modulo matematik!)
}
```

---

# BAHAGIAN 2: Comparison & Logical 🔍

```rust
fn main() {
    // ── Comparison operators ──────────────────────────────────
    println!("{}", 5 == 5);  // true  — equal
    println!("{}", 5 != 3);  // true  — not equal
    println!("{}", 5 >  3);  // true  — greater than
    println!("{}", 5 >= 5);  // true  — greater or equal
    println!("{}", 3 <  5);  // true  — less than
    println!("{}", 3 <= 3);  // true  — less or equal

    // ── Logical operators ─────────────────────────────────────
    println!("{}", true && false); // false — AND (short-circuit)
    println!("{}", true || false); // true  — OR (short-circuit)
    println!("{}", !true);         // false — NOT

    // Short-circuit evaluation:
    fn ada_effect() -> bool {
        println!("(fungsi dipanggil)");
        true
    }

    // false && ... → false, tidak evaluate sisi kanan!
    let _ = false && ada_effect(); // ada_effect() TIDAK dipanggil

    // true || ... → true, tidak evaluate sisi kanan!
    let _ = true || ada_effect();  // ada_effect() TIDAK dipanggil

    // ── Bukan short-circuit (bitwise) ─────────────────────────
    let _ = false & ada_effect();  // ada_effect() DIPANGGIL!
    let _ = true  | ada_effect();  // ada_effect() DIPANGGIL!
}
```

---

# BAHAGIAN 3: Bitwise Operators 🔢

```rust
fn main() {
    let a: u8 = 0b1100_1010; // 202
    let b: u8 = 0b1010_1100; // 172

    // ── Bitwise operators ─────────────────────────────────────
    println!("{:08b}", a & b);  // 1000_1000 — AND
    println!("{:08b}", a | b);  // 1110_1110 — OR
    println!("{:08b}", a ^ b);  // 0110_0110 — XOR
    println!("{:08b}", !a);     // 0011_0101 — NOT (bitwise complement)

    // ── Shift operators ───────────────────────────────────────
    let x: u8 = 0b0000_1111; // 15

    println!("{:08b}", x << 2); // 0011_1100 = 60 — shift kiri (×4)
    println!("{:08b}", x >> 2); // 0000_0011 = 3  — shift kanan (÷4)

    // Shift kiri = darab dengan kuasa dua
    println!("{}", 1u32 << 10); // 1024 (2^10)
    println!("{}", 5u32 << 3);  // 40 (5 × 8)

    // Shift kanan = bahagi dengan kuasa dua (integer)
    println!("{}", 100u32 >> 2); // 25 (100 ÷ 4)

    // AWAS: Arithmetic shift kanan untuk signed integers
    let neg: i8 = -8;           // 0b11111000
    println!("{}", neg >> 1);   // -4 (sign-extended, bukan 0b01111100)

    // ── Compound bitwise assignment ───────────────────────────
    let mut flags: u32 = 0;
    flags |= 0b0001; // set bit 0
    flags |= 0b0100; // set bit 2
    flags &= !0b0001; // clear bit 0
    flags ^= 0b0110; // toggle bits 1 dan 2
    println!("{:08b}", flags);

    // ── Guna praktikal: bit flags ─────────────────────────────
    const BACA:   u8 = 0b001; // 1
    const TULIS:  u8 = 0b010; // 2
    const LAKSANA: u8 = 0b100; // 4

    let kebenaran = BACA | TULIS; // boleh baca DAN tulis

    if kebenaran & BACA != 0   { println!("Boleh baca"); }
    if kebenaran & TULIS != 0  { println!("Boleh tulis"); }
    if kebenaran & LAKSANA != 0 { println!("Boleh laksana"); }
    else                        { println!("Tidak boleh laksana"); }
}
```

---

# BAHAGIAN 4: Reference & Pointer Symbols 🔗

```rust
fn main() {
    let x = 42i32;
    let mut y = 10i32;

    // ── & — Ampersand ─────────────────────────────────────────

    // 1. Buat immutable reference (borrow)
    let r: &i32 = &x;
    println!("{}", r); // 42

    // 2. Dalam type signature — reference type
    fn cetak(s: &str)    { println!("{}", s); }  // &str type
    fn tambah(v: &[i32]) { println!("{}", v.iter().sum::<i32>()); }

    // 3. Pattern matching — borrow dalam pattern
    let v = vec![1, 2, 3];
    for &n in &v { print!("{} ", n); } // &n = destructure reference
    println!();

    // 4. Bitwise AND (pada integers)
    let bitwise = 0b1100u8 & 0b1010u8; // 0b1000 = 8
    println!("{}", bitwise);

    // ── * — Asterisk ──────────────────────────────────────────

    // 1. Dereference — akses nilai di belakang reference
    let r = &x;
    println!("{}", *r);  // 42 — dereference
    *r;                  // BACA nilai

    let rm = &mut y;
    *rm += 5;            // TULIS melalui mutable reference

    // 2. Pendaraban
    println!("{}", 3 * 4); // 12

    // 3. Dalam type — raw pointer
    let ptr: *const i32 = &x as *const i32;
    let mut_ptr: *mut i32 = &mut y as *mut i32;

    // 4. Glob import dalam use
    use std::collections::*; // import semua
    let _: HashMap<i32, i32> = HashMap::new();

    // 5. Dalam impl untuk implement trait untuk all types
    // impl<T> MyTrait for *const T { ... }

    // ── && — Double ampersand ─────────────────────────────────

    // 1. Logical AND
    let a = true && false; // false

    // 2. Buat reference ke reference
    let rr: &&i32 = &&x;
    println!("{}", **rr); // 42 — deref dua kali

    // ── ** — Double deref ─────────────────────────────────────
    let boxed = Box::new(5i32);
    println!("{}", *boxed); // 5 — deref Box
    // **boxed  // kalau ada Box<Box<T>>
}
```

---

# BAHAGIAN 5: Path & Module Symbols 📦

```rust
// ── :: — Double colon (path separator) ───────────────────────

// 1. Akses item dalam module
use std::collections::HashMap;
let v = std::vec::Vec::<i32>::new();

// 2. Akses associated function / method
String::from("hello");
Vec::<i32>::new();
i32::MAX;
i32::MIN;

// 3. Enum variants
let opt: Option<i32> = Option::Some(42);
let result: Result<i32, &str> = Result::Ok(42);

// 4. Trait method dengan fully qualified syntax
let s = <i32 as ToString>::to_string(&42);
println!("{}", s);

// 5. Type parameters (turbofish)
let n = "42".parse::<i32>().unwrap();
let v = vec![1, 2, 3].iter().collect::<Vec<_>>();

// ── crate:: ───────────────────────────────────────────────────
// Bermula dari root crate
// use crate::modul::Item;

// ── super:: ───────────────────────────────────────────────────
// Naik satu level dalam modul hierarki
mod luar {
    pub fn fungsi_luar() {}

    mod dalam {
        pub fn fungsi_dalam() {
            super::fungsi_luar(); // naik ke 'luar'
        }
    }
}

// ── self:: ────────────────────────────────────────────────────
mod modul {
    pub fn a() {}
    pub fn b() {
        self::a(); // dalam modul yang sama
    }
}

fn main() {
    // Turbofish ::<> — tentukan type parameter
    let v: Vec<i32> = Vec::new();
    let v2 = Vec::<i32>::new();          // equivalent
    let n = "42".parse::<i32>().unwrap(); // turbofish pada method
}
```

---

# BAHAGIAN 6: Type & Generic Symbols 🏷️

```rust
// ── < > — Angle brackets ──────────────────────────────────────

// 1. Generic type parameter
struct Container<T> { nilai: T }
fn bungkus<T>(x: T) -> Container<T> { Container { nilai: x } }

// 2. Multiple type parameters
struct Peta<K, V> { kunci: K, nilai: V }

// 3. Trait bounds
fn cetak<T: std::fmt::Display>(x: T) { println!("{}", x); }

// 4. Where clause (alternative syntax)
fn cetak_v2<T>(x: T) where T: std::fmt::Display { println!("{}", x); }

// ── + — Plus dalam trait bounds ───────────────────────────────

// Multiple trait bounds
fn cetak_dan_debug<T: std::fmt::Display + std::fmt::Debug>(x: T) {
    println!("Display: {}", x);
    println!("Debug:   {:?}", x);
}

// Lifetime + trait bound
fn cetak_ref<'a, T: std::fmt::Display + 'a>(x: &'a T) {
    println!("{}", x);
}

// dyn Trait dengan multiple traits
fn buat_boxed() -> Box<dyn std::fmt::Display + Send + Sync> {
    Box::new(42i32)
}

// ── impl ──────────────────────────────────────────────────────

// 1. Implement methods untuk struct
struct Bulatan { r: f64 }
impl Bulatan {
    fn luas(&self) -> f64 { std::f64::consts::PI * self.r * self.r }
}

// 2. Implement trait untuk struct
impl std::fmt::Display for Bulatan {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "Bulatan(r={})", self.r)
    }
}

// 3. Dalam function parameter (impl Trait)
fn cetak_display(x: impl std::fmt::Display) { println!("{}", x); }

// 4. Dalam return position (impl Trait)
fn buat_iterator() -> impl Iterator<Item = i32> {
    vec![1, 2, 3].into_iter()
}

// ── dyn ───────────────────────────────────────────────────────
// Dynamic dispatch — trait object

fn cetak_dynamic(x: &dyn std::fmt::Display) { println!("{}", x); }

let items: Vec<Box<dyn std::fmt::Display>> = vec![
    Box::new(42i32),
    Box::new("hello"),
    Box::new(3.14f64),
];

// ── ? — Question mark dalam type ──────────────────────────────
// (lihat juga bahagian operator ?)
type Result<T> = std::result::Result<T, String>;

fn main() {}
```

---

# BAHAGIAN 7: Pattern Matching Symbols 🎯

```rust
fn main() {
    // ── _ — Underscore ────────────────────────────────────────

    // 1. Wildcard pattern — match apa-apa, ignore nilai
    let x = 5;
    match x {
        1 => println!("satu"),
        _ => println!("lain-lain"), // wildcard
    }

    // 2. Ignore variable (suppress unused warning)
    let _tidak_dipakai = 42;

    // 3. Dalam destructure — ignore bahagian
    let (a, _, c) = (1, 2, 3); // ignore tengah
    let (first, ..) = (1, 2, 3, 4, 5); // ignore baki

    // 4. Dalam struct pattern
    struct Titik { x: i32, y: i32, z: i32 }
    let t = Titik { x: 1, y: 2, z: 3 };
    let Titik { x, .. } = t; // ignore y dan z

    // 5. Dalam tanda nombor: 1_000_000 (digit separator)
    let juta = 1_000_000i32;
    let pi   = 3.141_592_653f64;
    println!("{}", juta); // 1000000
    println!("{}", pi);   // 3.141592653

    // 6. Type placeholder: let x: Vec<_> = ...
    let v: Vec<_> = [1, 2, 3].iter().map(|x| x * 2).collect();

    // ── | — Or dalam patterns ─────────────────────────────────
    let n = 3;
    match n {
        1 | 2 | 3 => println!("satu, dua, atau tiga"),
        4 | 5     => println!("empat atau lima"),
        _         => println!("lain"),
    }

    // Or dalam if let
    if let Some(1) | Some(2) = Some(1) {
        println!("1 atau 2");
    }

    // ── @ — At operator ───────────────────────────────────────
    // Bind nilai DAN test pattern serentak

    let nombor = 7u32;
    match nombor {
        n @ 1..=5  => println!("{} dalam 1-5", n),
        n @ 6..=10 => println!("{} dalam 6-10", n), // n = 7
        n          => println!("{} di luar julat", n),
    }

    // @ dalam if let
    if let Some(n @ 1..=100) = Some(42) {
        println!("Dapat {} (dalam 1-100)", n);
    }

    // ── .. — Double dot (range dan rest) ──────────────────────

    // Dalam struct pattern — ignore fields lain
    struct Config { host: String, port: u16, debug: bool }
    let cfg = Config { host: "localhost".into(), port: 8080, debug: true };

    let Config { host, .. } = cfg; // hanya ambil host, ignore lain

    // Dalam tuple/array pattern
    let tuple = (1, 2, 3, 4, 5);
    let (first2, .., last) = tuple; // first2=1, last=5

    // Dalam slice pattern
    match [1, 2, 3, 4, 5].as_slice() {
        [first3, rest @ ..] => println!("Mula: {}, Baki: {:?}", first3, rest),
        []                  => println!("Kosong"),
    }

    // ── if guard dalam match ───────────────────────────────────
    let pair = (2, -2);
    match pair {
        (x, y) if x == y   => println!("Sama"),
        (x, y) if x + y == 0 => println!("Bertentangan: {}, {}", x, y),
        _                  => println!("Lain"),
    }
}
```

---

# BAHAGIAN 8: Range Symbols 📏

```rust
fn main() {
    // ── .. — Range (exclusive end) ────────────────────────────
    let r = 1..5; // 1, 2, 3, 4 (tidak termasuk 5!)
    for n in 1..5 { print!("{} ", n); } // 1 2 3 4
    println!();

    // ── ..= — Range inclusive ─────────────────────────────────
    let r = 1..=5; // 1, 2, 3, 4, 5 (termasuk 5!)
    for n in 1..=5 { print!("{} ", n); } // 1 2 3 4 5
    println!();

    // ── ..n — RangeTo (dari awal hingga n, exclusive) ─────────
    let v = vec![1, 2, 3, 4, 5];
    let slice = &v[..3]; // [1, 2, 3]
    println!("{:?}", slice);

    // ── ..=n — RangeToInclusive ───────────────────────────────
    let slice2 = &v[..=3]; // [1, 2, 3, 4]
    println!("{:?}", slice2);

    // ── n.. — RangeFrom (dari n hingga akhir) ────────────────
    let slice3 = &v[2..]; // [3, 4, 5]
    println!("{:?}", slice3);

    // ── .. — RangeFull (semua) ────────────────────────────────
    let slice4 = &v[..]; // [1, 2, 3, 4, 5]
    println!("{:?}", slice4);

    // ── Range dalam match ─────────────────────────────────────
    let gred = 85u32;
    let label = match gred {
        90..=100 => "A",
        80..=89  => "B", // ← ini match
        70..=79  => "C",
        60..=69  => "D",
        _        => "E",
    };
    println!("Gred: {}", label); // B

    // ── Range dengan chars ────────────────────────────────────
    let huruf = 'f';
    match huruf {
        'a'..='z' => println!("huruf kecil"),
        'A'..='Z' => println!("huruf besar"),
        '0'..='9' => println!("digit"),
        _         => println!("lain"),
    }

    // ── Step dalam range ──────────────────────────────────────
    for n in (0..10).step_by(2) { print!("{} ", n); } // 0 2 4 6 8
    println!();

    for n in (0..=10).step_by(3) { print!("{} ", n); } // 0 3 6 9
    println!();

    // Reverse range
    for n in (0..5).rev() { print!("{} ", n); } // 4 3 2 1 0
    println!();
}
```

---

# BAHAGIAN 9: Macro & Attribute Symbols 🪄

```rust
// ── ! — Exclamation mark ──────────────────────────────────────

// 1. Macro invocation
println!("Hello!");         // println adalah macro
vec![1, 2, 3];              // vec! adalah macro
format!("Nilai: {}", 42);   // format! adalah macro
panic!("Sesuatu salah!");   // panic! adalah macro
assert!(1 + 1 == 2);        // assert! adalah macro
todo!("Belum siap");        // todo! adalah macro

// Boleh guna () [] {} untuk macro:
println!("sama");           // dengan ()
println!["sama"];           // dengan []
println!{"sama"};           // dengan {}
// Konvensyen: println!() untuk expression-like, vec![] untuk array-like

// 2. Logical NOT
let bukan_benar = !true;  // false
let bukan_palsu = !false; // true

// 3. Bitwise NOT (pada integer)
let x: u8 = 0b1010_1010;
let y: u8 = !x; // 0b0101_0101

// 4. Never type — fungsi yang tidak pernah return
fn selama_lamanya() -> ! {
    loop {}
}
fn sentiasa_panic() -> ! {
    panic!("Tidak boleh balik!");
}

// 5. Inner attribute (bukan outer)
// #![allow(unused)]  ← apply ke seluruh fail/modul
// #[allow(unused)]   ← apply ke item seterusnya

// ── # — Hash / Pound ──────────────────────────────────────────

// 1. Outer attribute — apply ke item selepas
#[derive(Debug, Clone)]
struct MyStruct { x: i32 }

#[test]
fn test_sesuatu() {
    assert_eq!(1 + 1, 2);
}

#[allow(unused_variables)]
fn fungsi() {
    let tidak_dipakai = 42;
}

#[cfg(test)]
mod tests {
    // Hanya compile dalam test mode
}

// 2. Doc comment (/// atau //!)
/// Ini adalah dokumentasi fungsi
/// yang akan muncul dalam `cargo doc`
fn fungsi_dengan_doc() {}

//! Ini adalah inner doc comment untuk modul/crate

// ── Common Attributes ─────────────────────────────────────────
// #[derive(...)]           — auto-implement traits
// #[test]                  — tandakan sebagai test function
// #[cfg(...)]              — conditional compilation
// #[allow/warn/deny]       — lint control
// #[inline]                — hint untuk inline function
// #[repr(...)]             — memory layout control
// #[must_use]              — warn jika return value diabaikan
// #[deprecated]            — tandakan sebagai deprecated
// #[doc = "..."]           — documentation
// #[no_mangle]             — jangan ubah nama (untuk FFI)
// #[export_name = "name"]  — nama untuk export (FFI)

fn main() {
    // Macro dengan format strings
    let name = "KADA";
    let port = 8080;

    println!("Server {} pada port {}", name, port);
    println!("{name} pada {port}");  // captured variable (Rust 1.58+)
    println!("{:?}", vec![1, 2, 3]); // debug format
    println!("{:#?}", vec![1, 2, 3]); // pretty debug
    println!("{:5}", 42);            // padding lebar 5
    println!("{:05}", 42);           // zero-padded
    println!("{:.3}", 3.14159);      // 3 desimal: 3.142
    println!("{:>10}", "kanan");     // right-align lebar 10
    println!("{:<10}", "kiri");      // left-align lebar 10
    println!("{:^10}", "tengah");    // center lebar 10
    println!("{:b}", 42);            // binary: 101010
    println!("{:o}", 42);            // octal: 52
    println!("{:x}", 255);           // hex lowercase: ff
    println!("{:X}", 255);           // hex uppercase: FF
    println!("{:#x}", 255);          // hex dengan prefix: 0xff
    println!("{:e}", 1234567.0);     // scientific: 1.234567e6
    println!("{:p}", &42i32);        // pointer address
}
```

---

# BAHAGIAN 10: Miscellaneous Symbols 🔧

```rust
fn main() {
    // ── ; — Semicolon ─────────────────────────────────────────

    // 1. Statement terminator
    let x = 5;           // statement
    println!("{}", x);   // statement

    // 2. Dalam array type — pisahkan type dan saiz
    let arr: [i32; 5] = [0; 5]; // [type; size]

    // 3. Dalam array literal — nilai yang diulang
    let zeros = [0i32; 100]; // 100 sifar

    // Expression vs Statement:
    let a = {
        let x = 5;
        x + 1    // ← tiada semicolon = expression = return value
    };           // a = 6

    let b = {
        let x = 5;
        x + 1;   // ← ada semicolon = statement = return ()
    };           // b = ()

    // ── , — Comma ─────────────────────────────────────────────

    // 1. Pisahkan arguments dalam fungsi
    fn tambah(a: i32, b: i32) -> i32 { a + b }

    // 2. Dalam struct/enum definition
    struct Titik { x: f64, y: f64 }
    enum Arah { Utara, Selatan, Timur, Barat }

    // 3. Trailing comma (dibolehkan dan digalakkan!)
    let v = vec![
        1,
        2,
        3, // trailing comma OK!
    ];

    // ── -> — Arrow ────────────────────────────────────────────

    // 1. Return type dalam function signature
    fn kira(a: i32, b: i32) -> i32 { a + b }

    // 2. Dalam closure
    let double = |x: i32| -> i32 { x * 2 };
    let triple = |x| x * 3; // type inference — boleh tanpa ->

    // 3. Dalam match arm (lihat =>)

    // ── => — Fat arrow ────────────────────────────────────────

    // 1. Dalam match arm
    let x = 5;
    match x {
        1 => println!("satu"),
        2 => println!("dua"),
        _ => println!("lain"),
    }

    // 2. Dalam macro_rules! pattern
    macro_rules! sayhi {
        ($name:expr) => {       // ← => dalam macro rules
            println!("Hi, {}!", $name)
        };
    }
    sayhi!("Rust");

    // ── : — Colon ─────────────────────────────────────────────

    // 1. Type annotation
    let n: i32 = 42;
    let s: &str = "hello";

    // 2. Dalam struct definition
    struct Config { host: String, port: u16 }

    // 3. Dalam function parameters
    fn process(data: &[i32], limit: usize) {}

    // 4. Trait bounds
    fn cetak<T: std::fmt::Display>(x: T) {}

    // 5. Pattern dengan type
    let n: i32 = match "42".parse() { Ok(n) => n, Err(_) => 0 };

    // 6. Struct/enum update
    // let s2 = Config { host: "new".into(), ..s1 };
}
```

---

# BAHAGIAN 11: Lifetime Symbols ⏳

```rust
// ── ' — Apostrophe (lifetime) ─────────────────────────────────

// 1. Lifetime annotation
fn terpanjang<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() > s2.len() { s1 } else { s2 }
}

// 2. 'static — data hidup selama program
let s: &'static str = "hello, world!";

// 3. Lifetime dalam struct
struct Petikan<'a> {
    teks: &'a str, // pinjam dari luar
}

// 4. Lifetime dalam impl
impl<'a> Petikan<'a> {
    fn baru(teks: &'a str) -> Self { Petikan { teks } }
    fn papar(&self) { println!("{}", self.teks); }
}

// 5. Multiple lifetimes
fn pilih<'a, 'b>(x: &'a str, _y: &'b str) -> &'a str { x }

// 6. Lifetime bounds
use std::fmt::Display;
fn terpanjang_papar<'a, T>(
    s1: &'a str, s2: &'a str, tambahan: T
) -> &'a str
where T: Display + 'a,  // T mesti hidup sekurang-kurangnya 'a
{
    println!("{}", tambahan);
    if s1.len() > s2.len() { s1 } else { s2 }
}

// ── ' — Loop label ────────────────────────────────────────────
// (sama simbol, konteks berbeza!)

// Label untuk break/continue dalam nested loops
'luar: for i in 0..5 {
    for j in 0..5 {
        if i + j == 6 {
            break 'luar; // break loop luar!
        }
        print!("({},{}) ", i, j);
    }
}
println!();

// Continue dengan label
'atas: for i in 0..3 {
    for j in 0..3 {
        if j == 1 {
            continue 'atas; // skip ke iterasi seterusnya loop luar
        }
        print!("({},{}) ", i, j);
    }
}
println!();

fn main() {}
```

---

# BAHAGIAN 12: Simbol dalam Konteks Berbeza 🔄

```rust
fn main() {
    // ── ? — Question mark ─────────────────────────────────────

    // 1. Error propagation dalam fungsi yang return Result/Option
    fn baca(path: &str) -> Result<String, std::io::Error> {
        let s = std::fs::read_to_string(path)?; // return Err jika gagal
        Ok(s.to_uppercase())
    }

    // 2. Dalam main yang return Result
    // fn main() -> Result<(), Box<dyn std::error::Error>> {
    //     let s = std::fs::read_to_string("fail.txt")?;
    //     Ok(())
    // }

    // 3. Option dengan ?
    fn ambil_pertama(v: &[i32]) -> Option<i32> {
        let &first = v.first()?; // return None jika kosong
        Some(first * 2)
    }

    // ── .. — Double dot (pelbagai konteks) ────────────────────

    // 1. Range (exclusive)
    for i in 0..5 { print!("{} ", i); } // 0 1 2 3 4
    println!();

    // 2. Struct update syntax
    #[derive(Debug, Clone)]
    struct Config { host: String, port: u16, debug: bool }
    let c1 = Config { host: "a".into(), port: 80, debug: false };
    let c2 = Config { port: 443, ..c1.clone() }; // guna semua dari c1 kecuali port

    // 3. Ignore rest dalam pattern
    let (x, ..) = (1, 2, 3, 4, 5); // hanya ambil x
    let Config { host, .. } = c1;   // hanya ambil host

    // 4. Rest dalam slice pattern
    if let [first, .., last] = [1, 2, 3, 4, 5].as_slice() {
        println!("Mula: {}, Akhir: {}", first, last);
    }

    // ── => ────────────────────────────────────────────────────

    // 1. Match arm
    let n = match 5 { 1 => "satu", _ => "lain" };

    // 2. Dalam macro_rules!
    // macro_rules! myMacro {
    //     ($x:expr) => { println!("{}", $x) };
    // }

    // ── - ─────────────────────────────────────────────────────

    // 1. Penolakan (subtraction)
    let diff = 10 - 3; // 7

    // 2. Unary minus (negation)
    let neg = -5;

    // 3. Dalam nama crate (snake-case untuk crate)
    // cargo add serde-json  (nama pada crates.io)
    // use serde_json;       (dalam Rust code, guna underscore!)

    // ── | dalam closure (pipe) ────────────────────────────────
    let double = |x| x * 2;        // tanpa parameter type
    let add = |x: i32, y: i32| x + y; // dengan parameter type
    let greet = |name: &str| {     // multi-line closure
        let msg = format!("Hello, {}!", name);
        println!("{}", msg);
        msg
    };

    // Closure capture modes
    let data = vec![1, 2, 3];
    let by_ref  = || println!("{:?}", data);    // borrow
    let consume  = move || println!("{:?}", data); // move ownership

    // ── {} dalam strings ──────────────────────────────────────
    println!("{}", 42);          // default (Display)
    println!("{:?}", vec![1,2]); // debug
    println!("{:#?}", vec![1,2]); // pretty debug
    println!("{:5}", 42);        // width 5
    println!("{:05}", 42);       // zero-padded width 5
    println!("{:.2}", 3.14159);  // 2 decimal places
    println!("{:8.2}", 3.14);    // width 8, 2 decimal places
    println!("{:>10}", "kanan"); // right-align
    println!("{:<10}", "kiri");  // left-align
    println!("{:^10}", "tengah"); // center
    println!("{:*>10}", "kanan"); // right-align dengan * fill
    println!("{:b}", 42);        // binary
    println!("{:#b}", 42);       // binary dengan 0b prefix
    println!("{:o}", 42);        // octal
    println!("{:#o}", 42);       // octal dengan 0o prefix
    println!("{:x}", 255);       // hex lowercase
    println!("{:X}", 255);       // hex uppercase
    println!("{:#x}", 255);      // hex dengan 0x prefix
    println!("{:e}", 1000000.0); // scientific notation

    // Named arguments
    let name = "Rust";
    let version = 2021;
    println!("{name} edition {version}"); // captured variables
    println!("{0} {1} {0}", "aba"); // positional arguments
}
```

---

# BAHAGIAN BONUS: Simbol Jarang tapi Penting 💎

```rust
fn main() {
    // ── @ — At dalam struct pattern ───────────────────────────

    // Bind nilai ke nama DAN check pattern
    struct Rekod { id: u32, nama: String }

    let r = Rekod { id: 42, nama: "Ali".into() };

    // @ dalam match
    match r.id {
        id @ 1..=100 => println!("ID {} dalam julat", id),
        id            => println!("ID {} luar julat", id),
    }

    // ── .. dalam format ───────────────────────────────────────
    // (bukan double dot, tapi format fill)
    println!("{:.5}", "hello world"); // "hello" (truncate string!)

    // ── $ — Dollar (dalam macro_rules!) ──────────────────────
    macro_rules! cetak_x_kali {
        ($e:expr, $n:expr) => {
            for _ in 0..$n {
                println!("{}", $e);
            }
        };
    }
    cetak_x_kali!("Hello!", 3);

    // ── ## — Double hash dalam macro ──────────────────────────
    // (token pasting dalam macro_rules!)
    macro_rules! buat_fungsi {
        ($nama:ident) => {
            fn $nama() {
                println!("Fungsi: {}", stringify!($nama));
            }
        };
    }
    buat_fungsi!(salam_dunia);
    salam_dunia();

    // ── as — Type cast keyword ────────────────────────────────
    let x: i32 = 42;
    let y: f64 = x as f64;      // integer ke float
    let z: u8  = 300i32 as u8;  // narrowing (lossy!)

    // ── as dalam use ──────────────────────────────────────────
    use std::fmt::Display as Papar;
    use std::collections::HashMap as Map;

    // ── Box<T> ────────────────────────────────────────────────
    let boxed = Box::new(5i32);       // data dalam heap
    println!("{}", *boxed);            // deref untuk akses nilai

    // ── .. dalam use ──────────────────────────────────────────
    use std::io::{self, Read, Write};  // self = io module itu sendiri

    // ── | dalam type ──────────────────────────────────────────
    // (Or-pattern dalam match — bukan union type!)
    // Note: Rust tiada union type dalam safe code
    // (ada union keyword tapi untuk C interop dalam unsafe)

    // ── ~ tilde ───────────────────────────────────────────────
    // TIDAK digunakan dalam Rust moden (dulu ada, dah removed)

    // ── $ crate dalam macro ───────────────────────────────────
    #[macro_export]
    macro_rules! log_info {
        ($msg:expr) => {
            // $crate merujuk kepada crate yang define macro ini
            println!("[INFO] {}", $msg);
        };
    }

    // ── /* */ — Block comment ─────────────────────────────────
    let x = /* nilai penting */ 42;
    /* Ini adalah
       block comment
       berbilang baris */

    // ── // — Line comment ─────────────────────────────────────
    let y = 5; // ini comment

    // ── /// — Doc comment (outer) ─────────────────────────────
    // (untuk item selepas comment)

    // ── //! — Doc comment (inner) ─────────────────────────────
    // (untuk item yang mengandungi comment)
}

/// Contoh fungsi dengan doc comment.
///
/// # Arguments
/// * `x` - nilai pertama
///
/// # Examples
/// ```
/// let hasil = tambah_satu(5);
/// assert_eq!(hasil, 6);
/// ```
fn tambah_satu(x: i32) -> i32 {
    x + 1
}
```

---

# 📋 Rujukan Pantas — Simbol Cheat Sheet

## Simbol Mengikut Fungsi

```
ARITHMETIC:
  +  -  *  /  %           → tambah, tolak, darab, bahagi, modulo
  +=  -=  *=  /=  %=      → compound assignment

COMPARISON:
  ==  !=  >  >=  <  <=    → perbandingan

LOGICAL:
  &&  ||  !               → AND, OR, NOT (short-circuit)
  &   |   !               → AND, OR, NOT (bitwise / tanpa short-circuit)

BITWISE:
  &   |   ^   !           → AND, OR, XOR, NOT
  <<  >>                  → shift kiri, shift kanan
  &=  |=  ^=  <<=  >>=   → compound bitwise assignment

REFERENCE:
  &x          → buat reference (borrow)
  &mut x      → buat mutable reference
  *r          → dereference
  &T          → reference type
  *const T    → raw immutable pointer
  *mut T      → raw mutable pointer

PATH:
  ::          → path separator (module::item)
  crate::     → dari root crate
  super::     → naik satu level
  self::      → dalam module semasa
  ::<T>       → turbofish (explicit type parameter)

RANGE:
  a..b        → range exclusive [a, b)
  a..=b       → range inclusive [a, b]
  ..b         → range dari awal
  a..         → range hingga akhir
  ..          → full range

PATTERN:
  _           → wildcard (ignore)
  |           → or pattern
  @           → bind dan test
  ..          → ignore rest

ERROR:
  ?           → propagate error (return Err/None jika gagal)

MACRO:
  name!(...)  → macro invocation
  #[...]      → outer attribute
  #![...]     → inner attribute

LIFETIME:
  'a          → lifetime parameter
  'static     → data hidup selama program
  'label:     → loop label (untuk break/continue)

CLOSURE:
  |args| expr → closure
  move |args| → move closure

FORMAT:
  {}          → Display
  {:?}        → Debug
  {:#?}       → Pretty Debug
  {:5}        → lebar 5
  {:.2}       → 2 decimal
  {:>}  {:<}  {:^}  → right/left/center align
  {:b}  {:o}  {:x}  {:X}  → binary/octal/hex
  {:#b} {:#x} → dengan prefix
```

## Simbol yang SAMA, Konteks BERBEZA

```
*   →  dereference:  *ptr
    →  pendaraban:   5 * 3
    →  glob import:  use std::io::*
    →  raw pointer type: *const T, *mut T
    →  implement untuk all: impl<T> Trait for *const T

&   →  borrow:       let r = &x
    →  reference type: fn f(s: &str)
    →  bitwise AND:  a & b
    →  pattern:      for &n in &v { }
    →  trait bound:  T: Display + Debug (itu +, bukan &)

!   →  macro:        println!(...)
    →  logical NOT:  !true
    →  bitwise NOT:  !0b1010
    →  never type:   fn f() -> !
    →  inner attr:   #![allow(...)]

|   →  or pattern:   1 | 2 | 3
    →  closure:      |x| x * 2
    →  bitwise OR:   a | b

..  →  range:        1..5
    →  rest pattern: let (x, ..) = tuple
    →  struct update: Config { port, ..other }
    →  slice rest:   [first, rest @ ..]

'   →  lifetime:     'a, 'static
    →  loop label:   'outer: loop { }
    →  char literal: 'A', '\n', '\u{1F600}'

:   →  type ann:     let x: i32
    →  in struct:    struct S { x: i32 }
    →  trait bound:  T: Display
    →  dict-like:    match { key: val }
    →  format spec:  {:5.2}
```

---

## 🏆 Cabaran Akhir

Boleh baca kod ini tanpa tengok rujukan?

```rust
use std::{fmt, io::Read, collections::HashMap as Map};

#[derive(Debug, Clone, PartialEq)]
struct DataStore<'a, T: fmt::Display + 'a> {
    data:  &'a [T],
    index: Map<&'a str, usize>,
}

impl<'a, T: fmt::Display> DataStore<'a, T> {
    fn baru(data: &'a [T]) -> Self {
        DataStore { data, index: Map::new() }
    }

    fn dapatkan(&self, kunci: &str) -> Option<&T> {
        self.index.get(kunci).and_then(|&i| self.data.get(i))
    }
}

fn proses<T>(items: &[T]) -> Result<Vec<String>, Box<dyn std::error::Error>>
where T: fmt::Display + fmt::Debug,
{
    items.iter()
         .enumerate()
         .map(|(i, item)| Ok(format!("[{:03}] {}", i, item)))
         .collect()
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let data = vec![1i32, 2, 3, 4, 5];
    let store = DataStore::baru(&data);

    if let Some(v) = store.dapatkan("kunci") {
        println!("{}", v);
    }

    let hasil = proses(&data)?;
    for (i, s) in hasil.iter().enumerate() {
        println!("{:>3}: {}", i + 1, s);
    }

    Ok(())
}
```

---

*Simbol dalam Rust — setiap tanda ada maksud yang tepat.*
*Baca konteks, faham maksud, tulis dengan yakin.* 🦀
