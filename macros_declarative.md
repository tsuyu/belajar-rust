# 🪄 Macros Deklaratif (macro_rules!) dalam Rust — Beginner to Advanced

> Panduan lengkap macro_rules!: dari macro mudah hingga pattern matching kompleks.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu Macro?

```
Fungsi biasa:               Macro:
  tambah(3, 4)                println!("hello")
  ↓                           ↓
  berjalan masa RUNTIME       dikembangkan masa COMPILE TIME
  terima nilai                terima KOD (token)
  return nilai                hasilkan KOD baru

Macro = "kod yang tulis kod lain"
       = metaprogramming

fn tambah(a: i32, b: i32) → terima dua i32
println!(...)              → boleh terima berapa-berapa argument!
vec![1, 2, 3]              → boleh berapa elemen pun!
```

---

## Kenapa Belajar macro_rules!?

```
✔ Kurangkan kod berulang (DRY — Don't Repeat Yourself)
✔ Buat DSL (Domain-Specific Language) dalam Rust
✔ Terima bilangan argument yang fleksibel (variadic)
✔ Ungkapkan pattern yang tidak boleh dibuat dengan fungsi biasa
✔ Digunakan meluas dalam standard library (vec!, println!, assert!)

Rust ada dua jenis macro:
  1. Deklaratif → macro_rules! (yang kita belajar sekarang)
  2. Prosedural → proc macros (lebih kompleks, lain kali)
```

---

## Peta Pembelajaran

```
Bab 1  → macro_rules! Asas
Bab 2  → Pattern Matching dalam Macro
Bab 3  → Fragment Specifiers ($x:ty, $x:expr, dll)
Bab 4  → Variadic Macros — $ (...) *
Bab 5  → Repetition Patterns
Bab 6  → Hygiene & Scope
Bab 7  → Recursive Macros
Bab 8  → Macro Export & Import
Bab 9  → Pattern Lanjutan
Bab 10 → Mini Project: Bina Testing Framework
```

---

# BAB 1: macro_rules! Asas 🌱

## Macro Pertama

```rust
// Define macro dengan macro_rules!
macro_rules! sapa {
    // Pattern → Expansion
    () => {
        println!("Hello, Rust!")
    };
}

// Macro dengan argument
macro_rules! sapa_nama {
    ($nama:expr) => {
        println!("Hello, {}!", $nama)
    };
}

fn main() {
    sapa!();                    // Hello, Rust!
    sapa_nama!("Ali");          // Hello, Ali!
    sapa_nama!("Siti");         // Hello, Siti!

    // Nota: Macro boleh guna () [] {} — semua sama!
    sapa!();
    sapa![];
    sapa! {};
    // Konvensyen: () untuk expression-like, [] untuk array-like, {} untuk block-like
}
```

---

## Syntax macro_rules!

```rust
macro_rules! nama_macro {
    //  ┌── pattern ──┐    ┌──── expansion ────┐
        (PATTERN)      =>  { KOD YANG DIHASILKAN };
    //  ^^^^^^^^^^^         ^^^^^^^^^^^^^^^^^^^^
    //  dipadankan dengan   kod yang dihasilkan
    //  input macro         bila pattern sepadan

    // Boleh ada BANYAK arm (seperti match)
        (PATTERN2)     =>  { KOD LAIN };
        (PATTERN3)     =>  { KOD LAIN LAGI };
}
```

---

## Macro dengan Pelbagai Arms

```rust
macro_rules! nilai_mutlak {
    // Arm 1: satu nombor
    ($x:expr) => {
        if $x >= 0 { $x } else { -$x }
    };
}

macro_rules! maks {
    // Arm 1: dua nilai
    ($a:expr, $b:expr) => {
        if $a > $b { $a } else { $b }
    };

    // Arm 2: tiga nilai (guna macro rekursif)
    ($a:expr, $b:expr, $c:expr) => {
        maks!(maks!($a, $b), $c)
    };
}

fn main() {
    println!("{}", nilai_mutlak!(-5));       // 5
    println!("{}", nilai_mutlak!(3));         // 3

    println!("{}", maks!(3, 7));             // 7
    println!("{}", maks!(3, 7, 5));          // 7
    println!("{}", maks!(10, 2, 8));         // 10
}
```

---

## 🧠 Brain Teaser #1

Apakah output kod ini, dan bagaimana macro dikembangkan?

```rust
macro_rules! kuasa_dua {
    ($x:expr) => {
        $x * $x
    };
}

fn main() {
    let a = 3;
    let hasil = kuasa_dua!(a + 1);
    println!("{}", hasil);
}
```

<details>
<summary>👀 Jawapan</summary>

Output: **7** (bukan 16!)

Macro dikembangkan secara textual:
```rust
let hasil = (a + 1) * (a + 1);
// tetapi! $x = "a + 1" tanpa kurungan
// jadi: a + 1 * a + 1 = 3 + 1 * 3 + 1 = 3 + 3 + 1 = 7
```

**Bug klasik macro!** Penyelesaian — wrap `$x` dengan kurungan:

```rust
macro_rules! kuasa_dua {
    ($x:expr) => {
        ($x) * ($x)  // kurungan penting!
    };
}

// Sekarang: (a + 1) * (a + 1) = 4 * 4 = 16 ✔
```

**Pelajaran:** Sentiasa wrap expansion macro dengan kurungan untuk elak operator precedence bug!
</details>

---

# BAB 2: Pattern Matching dalam Macro 🎯

## Banyak Pattern Arms

```rust
macro_rules! log {
    // Arm 1: hanya mesej
    ($mesej:expr) => {
        println!("[INFO] {}", $mesej);
    };

    // Arm 2: level + mesej
    ($level:expr, $mesej:expr) => {
        println!("[{}] {}", $level, $mesej);
    };

    // Arm 3: level + format string + arguments
    ($level:expr, $fmt:literal, $($arg:expr),*) => {
        println!(concat!("[", $level, "] ", $fmt), $($arg),*);
    };
}

fn main() {
    log!("Program dimulakan");             // [INFO] Program dimulakan
    log!("WARN", "Stok hampir habis");    // [WARN] Stok hampir habis
    log!("ERROR", "Gagal sambung ke {}", "localhost"); // [ERROR] Gagal sambung ke localhost
}
```

---

## Pattern dengan Keyword Khas

```rust
macro_rules! buat_getter {
    // Pattern dengan keyword khas (bukan fragment)
    (get $field:ident dari $struct_nama:ident) => {
        // $ident = identifier (nama variable, fungsi, dll)
        paste::paste! {
            fn [<get_ $field>](&self) -> &str {
                &self.$field
            }
        }
    };
}

// Cara mudah tanpa paste crate:
macro_rules! getter {
    ($nama:ident, $field:ident, $jenis:ty) => {
        pub fn $nama(&self) -> &$jenis {
            &self.$field
        }
    };
}

struct Pengguna {
    nama: String,
    emel: String,
}

impl Pengguna {
    getter!(dapatkan_nama, nama, String);
    getter!(dapatkan_emel, emel, String);
}

fn main() {
    let u = Pengguna {
        nama: "Ali".into(),
        emel: "ali@email.com".into(),
    };

    println!("{}", u.dapatkan_nama()); // Ali
    println!("{}", u.dapatkan_emel()); // ali@email.com
}
```

---

## Literal Patterns

```rust
macro_rules! pada_bila {
    // Match literal boolean
    (true  => $blok:block) => { $blok };
    (false => $blok:block) => { /* buat apa-apa */ };

    // Match literal nombor
    (0 kali => $blok:block) => { /* tak buat apa */ };
    ($n:literal kali => $blok:block) => {
        for _ in 0..$n {
            $blok
        }
    };
}

fn main() {
    pada_bila!(true => {
        println!("Ini dijalankan!");
    });

    pada_bila!(false => {
        println!("Ini TIDAK dijalankan!");
    });

    pada_bila!(3 kali => {
        println!("Hello!");
    });
    // Hello!
    // Hello!
    // Hello!

    pada_bila!(0 kali => {
        println!("Ini pun tak dijalankan!");
    });
}
```

---

# BAB 3: Fragment Specifiers 🔖

## Semua Fragment Specifiers

```
$x:expr    → sebarang expression: 2+3, "hello", fungsi()
$x:ident   → identifier: variable_name, FunctionName
$x:ty      → type: i32, String, Vec<u8>
$x:pat     → pattern: Some(x), (a, b), _
$x:stmt    → statement: let x = 5;
$x:block   → block: { let x = 1; x + 1 }
$x:item    → item: fn, struct, enum, impl
$x:literal → literal: 42, "hello", 3.14, true
$x:meta    → attribute: derive(Debug), cfg(test)
$x:tt      → token tree: satu token atau token dalam ()/{}/[]
$x:lifetime → lifetime: 'a, 'static
$x:vis     → visibility: pub, pub(crate)
$x:path    → path: std::collections::HashMap
```

---

## Contoh Setiap Fragment

```rust
// expr — sebarang expression
macro_rules! guna_expr {
    ($e:expr) => { println!("Nilai: {}", $e) };
}

// ident — nama (identifier)
macro_rules! buat_fungsi {
    ($nama:ident) => {
        fn $nama() { println!("Fungsi {}!", stringify!($nama)) }
    };
}

// ty — type
macro_rules! buat_vec_typed {
    ($jenis:ty) => {
        Vec::<$jenis>::new()
    };
}

// pat — pattern
macro_rules! semak_pattern {
    ($nilai:expr, $pat:pat) => {
        matches!($nilai, $pat)
    };
}

// block
macro_rules! jalankan {
    ($blok:block) => {
        let hasil = $blok;
        println!("Hasil: {:?}", hasil);
    };
}

// literal
macro_rules! ulang_n {
    ($n:literal, $e:expr) => {
        for _ in 0..$n { println!("{}", $e); }
    };
}

// tt — paling fleksibel
macro_rules! cetak_token {
    ($($tt:tt)*) => {
        println!("Token: {}", stringify!($($tt)*))
    };
}

buat_fungsi!(hello_world);

fn main() {
    guna_expr!(2 + 3);          // Nilai: 5
    guna_expr!("hello");         // Nilai: hello

    hello_world();               // Fungsi hello_world!

    let v: Vec<i32> = buat_vec_typed!(i32);
    println!("Vec empty: {}", v.is_empty());

    println!("{}", semak_pattern!(Some(5), Some(_))); // true
    println!("{}", semak_pattern!(None::<i32>, Some(_))); // false

    jalankan!({
        let x = 10;
        x * x
    }); // Hasil: 100

    ulang_n!(3, "Helo!"); // Helo! Helo! Helo!

    cetak_token!(hello world 1 + 2); // Token: hello world 1 + 2
}
```

---

## 🧠 Brain Teaser #2

Kenapa kita perlu fragment `$x:tt` berbanding `$x:expr`?

```rust
// Cuba buat macro yang terima MANA-MANA sintaks:
macro_rules! debug_apa_saja {
    ($($tt:tt)*) => {
        println!("KOD: {}", stringify!($($tt)*));
    };
}

fn main() {
    debug_apa_saja!(let x = 5);         // statement
    debug_apa_saja!(fn hello() {});     // item
    debug_apa_saja!(struct A;);         // item
    debug_apa_saja!(if true { 1 });     // expression
}
```

<details>
<summary>👀 Jawapan</summary>

`$x:tt` (token tree) adalah yang paling fleksibel — boleh terima **apa-apa** token.

| Fragment | Boleh terima |
|----------|-------------|
| `$x:expr` | expression sahaja: `2+3`, `"hello"`, `fungsi()` |
| `$x:stmt` | statement sahaja: `let x = 5;` |
| `$x:item` | item sahaja: `fn`, `struct` |
| `$x:tt`   | **SEMUA** — satu token atau token dalam `()/{}/[]` |

`$($tt:tt)*` = "sifar atau lebih token trees" — paling fleksibel!

Guna `$x:expr` apabila boleh — memberikan **error message yang lebih baik** bila input salah. Guna `$x:tt` bila perlu fleksibiliti maksimum atau manipulate kod secara textual.
</details>

---

# BAB 4: Variadic Macros — $(...) * ✨

## Pola Ulangan Asas

```
$( ... )*   = sifar atau lebih kali
$( ... )+   = satu atau lebih kali
$( ... )?   = sifar atau satu kali (optional)

Separator boleh ditambah:
$( ... ),*  = dipisahkan dengan koma (sifar atau lebih)
$( ... ),+  = dipisahkan dengan koma (satu atau lebih)
```

```rust
// Terima bilangan argumen yang berbeza!
macro_rules! jumlah {
    // Kes asas: tiada argumen
    () => { 0 };

    // Satu atau lebih argumen dipisah koma
    ($($nilai:expr),+) => {
        0 $(+ $nilai)+
    };
}

macro_rules! cetak_semua {
    ($($item:expr),*) => {
        $(
            println!("{}", $item);
        )*
    };
}

fn main() {
    println!("{}", jumlah!());              // 0
    println!("{}", jumlah!(1));             // 1
    println!("{}", jumlah!(1, 2));          // 3
    println!("{}", jumlah!(1, 2, 3, 4, 5)); // 15

    cetak_semua!("epal", "mangga", "pisang");
    // epal
    // mangga
    // pisang

    cetak_semua!(1, 2.0, true, "hello");
    // 1
    // 2
    // true
    // hello
}
```

---

## Membina vec! Sendiri

```rust
// Bayangkan bagaimana vec! dalam standard library dibuat!
macro_rules! vec_ku {
    // Kosong
    () => {
        Vec::new()
    };

    // Satu atau lebih nilai
    ($($x:expr),+ $(,)?) => {
        // $(,)? = optional trailing comma
        {
            let mut v = Vec::new();
            $(
                v.push($x);
            )+
            v
        }
    };

    // Nilai berulang: [nilai; bilangan]
    ($nilai:expr; $bilangan:expr) => {
        vec![0; $bilangan].iter().map(|_| $nilai).collect::<Vec<_>>()
    };
}

fn main() {
    let v1: Vec<i32> = vec_ku![];
    let v2 = vec_ku![1, 2, 3];
    let v3 = vec_ku![1, 2, 3,]; // trailing comma OK!
    let v4 = vec_ku!["a", "b", "c", "d"];

    println!("{:?}", v1); // []
    println!("{:?}", v2); // [1, 2, 3]
    println!("{:?}", v3); // [1, 2, 3]
    println!("{:?}", v4); // ["a", "b", "c", "d"]
}
```

---

## Macro untuk HashMap

```rust
macro_rules! peta {
    // Kosong
    () => {
        std::collections::HashMap::new()
    };

    // key => value pasangan
    ($($k:expr => $v:expr),+ $(,)?) => {
        {
            let mut m = std::collections::HashMap::new();
            $(
                m.insert($k, $v);
            )+
            m
        }
    };
}

fn main() {
    let kosong: std::collections::HashMap<&str, i32> = peta!();

    let markah = peta! {
        "Ali"   => 85,
        "Siti"  => 92,
        "Amin"  => 78,
        "Zara"  => 90,
    };

    for (nama, skor) in &markah {
        println!("{}: {}", nama, skor);
    }
}
```

---

## 🧠 Brain Teaser #3

Tulis macro `min!` yang berfungsi untuk satu, dua, atau lebih nilai.

```rust
min!(5)           // → 5
min!(3, 7)        // → 3
min!(8, 2, 5)     // → 2
min!(10, 3, 7, 1) // → 1
```

<details>
<summary>👀 Jawapan</summary>

```rust
macro_rules! min {
    // Satu nilai — return terus
    ($a:expr) => { $a };

    // Dua nilai
    ($a:expr, $b:expr) => {
        if $a < $b { $a } else { $b }
    };

    // Tiga atau lebih — rekursif!
    ($a:expr, $b:expr, $($baki:expr),+) => {
        min!(min!($a, $b), $($baki),+)
    };
}

fn main() {
    println!("{}", min!(5));              // 5
    println!("{}", min!(3, 7));           // 3
    println!("{}", min!(8, 2, 5));        // 2
    println!("{}", min!(10, 3, 7, 1));    // 1
}
```

Cara kerja rekursif untuk `min!(8, 2, 5)`:
```
min!(8, 2, 5)
= min!(min!(8, 2), 5)
= min!(2, 5)
= 2  ✔
```
</details>

---

# BAB 5: Repetition Patterns 🔄

## Bina Struct dengan Macro

```rust
macro_rules! buat_struct {
    // struct dengan nama dan fields
    (
        struct $nama:ident {
            $($field:ident: $jenis:ty),* $(,)?
        }
    ) => {
        #[derive(Debug, Clone)]
        struct $nama {
            $($field: $jenis,)*
        }

        impl $nama {
            fn baru($($field: $jenis),*) -> Self {
                $nama {
                    $($field,)*
                }
            }
        }
    };
}

buat_struct! {
    struct Pengguna {
        nama: String,
        umur: u32,
        emel: String,
    }
}

buat_struct! {
    struct Titik {
        x: f64,
        y: f64,
    }
}

fn main() {
    let p = Pengguna::baru("Ali".into(), 25, "ali@email.com".into());
    println!("{:?}", p);

    let t = Titik::baru(3.0, 4.0);
    println!("{:?}", t);
}
```

---

## Pattern Matching Dalaman

```rust
macro_rules! tanda {
    // Lebih dari satu
    ($($item:expr),+ yang $sifat:ident) => {
        {
            let mut v = Vec::new();
            $(
                if $item.$sifat() {
                    v.push($item);
                }
            )+
            v
        }
    };
}

// Macro untuk buat enum dengan Display otomatik
macro_rules! enum_dengan_display {
    (
        enum $nama:ident {
            $($variant:ident = $teks:literal),+ $(,)?
        }
    ) => {
        #[derive(Debug, Clone, PartialEq)]
        enum $nama {
            $($variant,)+
        }

        impl std::fmt::Display for $nama {
            fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
                match self {
                    $($nama::$variant => write!(f, $teks),)+
                }
            }
        }
    };
}

enum_dengan_display! {
    enum Status {
        Aktif = "Aktif ✔",
        Tidak = "Tidak Aktif ✘",
        Pending = "Sedang Diproses...",
    }
}

fn main() {
    let s = Status::Aktif;
    println!("{}", s);       // Aktif ✔
    println!("{:?}", s);     // Aktif

    let t = Status::Pending;
    println!("{}", t);       // Sedang Diproses...
}
```

---

## Nested Repetition

```rust
// Dua lapisan repetition
macro_rules! jadual {
    (
        $($baris:expr => [$($sel:expr),*]),* $(,)?
    ) => {
        {
            let mut data = Vec::new();
            $(
                let mut baris_data = Vec::new();
                $(
                    baris_data.push($sel.to_string());
                )*
                data.push(($baris.to_string(), baris_data));
            )*
            data
        }
    };
}

fn main() {
    let data = jadual! {
        "Bab 1" => ["Pengenalan", "Asas", "Contoh"],
        "Bab 2" => ["Lanjutan", "Pattern"],
        "Bab 3" => ["Projek"],
    };

    for (bab, kandungan) in &data {
        println!("{}: {:?}", bab, kandungan);
    }
    // Bab 1: ["Pengenalan", "Asas", "Contoh"]
    // Bab 2: ["Lanjutan", "Pattern"]
    // Bab 3: ["Projek"]
}
```

---

# BAB 6: Hygiene & Scope 🛡️

## Macro Hygiene — Tidak Boleh "Cemar" Scope

```rust
macro_rules! guna_temp {
    ($x:expr) => {
        {
            // 'temp' dalam macro TIDAK sama dengan 'temp' dalam caller!
            let temp = $x * 2;
            println!("Dalaman: {}", temp);
            temp + 1
        }
    };
}

fn main() {
    let temp = 100; // variable ini

    let hasil = guna_temp!(5); // 'temp' dalam macro ≠ 'temp' ini

    // Rust macro adalah "hygienic" — variable dalam macro tidak
    // boleh cemar atau dicemar oleh variable dalam caller!
    println!("temp luar: {}", temp);   // masih 100!
    println!("hasil: {}", hasil);       // 11
}
```

---

## Internal Variable dengan $crate

```rust
// $crate — refer ke root crate semasa
// Berguna untuk macro yang boleh digunakan dari crate lain

#[macro_export]  // export supaya crate lain boleh guna
macro_rules! buat_error {
    ($mesej:expr) => {
        // $crate::Error = rujuk exact type dari crate ini
        // walaupun macro digunakan dari luar crate
        format!("[KADA-ERROR] {}", $mesej)
    };
}

// Macro yang inject variable ke scope
// (ini hanya boleh dilakukan dengan cara khas)
macro_rules! masuk_dengan_timer {
    ($blok:block) => {
        {
            let __mula = std::time::Instant::now();
            let hasil = $blok;
            println!("Masa: {}μs", __mula.elapsed().as_micros());
            hasil
        }
    };
}

fn main() {
    let e = buat_error!("Sambungan gagal");
    println!("{}", e);

    let hasil = masuk_dengan_timer!({
        let mut jumlah = 0i64;
        for i in 0..1_000_000 {
            jumlah += i;
        }
        jumlah
    });
    println!("Hasil: {}", hasil);
}
```

---

# BAB 7: Recursive Macros 🔄

## Macro yang Panggil Dirinya Sendiri

```rust
// Kira panjang list semasa compile
macro_rules! panjang_list {
    () => { 0usize };
    ($head:expr $(, $tail:expr)*) => {
        1 + panjang_list!($($tail),*)
    };
}

// Tukar tuple ke Vec
macro_rules! tuple_ke_vec {
    ($($x:expr),*) => {
        vec![$($x),*]
    };
}

// Rekursif dengan accumulator
macro_rules! darab_semua {
    ($x:expr) => { $x };
    ($first:expr, $($rest:expr),+) => {
        $first * darab_semua!($($rest),+)
    };
}

fn main() {
    const PANJANG: usize = panjang_list!(1, 2, 3, 4, 5);
    println!("Panjang: {}", PANJANG); // 5

    let v = tuple_ke_vec!(10, 20, 30, 40);
    println!("{:?}", v);

    println!("{}", darab_semua!(2, 3, 4)); // 24 = 2*3*4
    println!("{}", darab_semua!(1, 2, 3, 4, 5)); // 120
}
```

---

## Stack-like Macro

```rust
// Bina senarai secara rekursif — pattern "TT muncher"
macro_rules! senarai_nombor {
    // Kes asas — kosong
    (@acc [$($acc:expr),*]) => {
        vec![$($acc),*]
    };

    // Tambah nombor ke accumulator
    (@acc [$($acc:expr),*] $n:expr $(, $baki:expr)*) => {
        senarai_nombor!(@acc [$($acc,)* $n] $($baki),*)
    };

    // Entry point
    ($($item:expr),* $(,)?) => {
        senarai_nombor!(@acc [] $($item),*)
    };
}

fn main() {
    let v = senarai_nombor![3, 1, 4, 1, 5, 9];
    println!("{:?}", v); // [3, 1, 4, 1, 5, 9]
}
```

---

## 🧠 Brain Teaser #4

Tulis macro `println_semua!` yang terima satu format string dan berapa-berapa argument, kemudian print setiap argument pada baris berasingan.

```rust
println_semua!("Nilai: {}", 1, 2, 3);
// Nilai: 1
// Nilai: 2
// Nilai: 3
```

<details>
<summary>👀 Jawapan</summary>

```rust
macro_rules! println_semua {
    ($fmt:literal, $($arg:expr),+ $(,)?) => {
        $(
            println!($fmt, $arg);
        )+
    };
}

fn main() {
    println_semua!("Nilai: {}", 1, 2, 3);
    // Nilai: 1
    // Nilai: 2
    // Nilai: 3

    println_semua!("Nama: {}", "Ali", "Siti", "Amin");
    // Nama: Ali
    // Nama: Siti
    // Nama: Amin
}
```

Cara kerja: `$($arg:expr),+` match satu atau lebih expression dipisah koma. Kemudian `$(println!($fmt, $arg);)+` expand satu `println!` untuk setiap `$arg`.
</details>

---

# BAB 8: Macro Export & Import 📤

## #[macro_export] — Share Macro Antara Modules

```rust
// src/lib.rs atau mana-mana fail

// Macro yang boleh digunakan dari crate lain
#[macro_export]
macro_rules! assert_hampir_sama {
    ($a:expr, $b:expr) => {
        assert_hampir_sama!($a, $b, 1e-10)
    };

    ($a:expr, $b:expr, $epsilon:expr) => {
        {
            let diff = ($a - $b).abs();
            if diff > $epsilon {
                panic!(
                    "assertion failed: |{} - {}| = {} > epsilon {}",
                    $a, $b, diff, $epsilon
                );
            }
        }
    };
}

#[macro_export]
macro_rules! kad_debug {
    ($e:expr) => {
        {
            let val = $e;
            println!("[DEBUG] {} = {:?}", stringify!($e), val);
            val
        }
    };
}
```

---

## Import Macro dari Crate Lain

```toml
# Cargo.toml
[dependencies]
nama-crate = "1.0"
```

```rust
// Cara 1: use dengan path
use nama_crate::assert_hampir_sama;

// Cara 2: #[macro_use] (cara lama)
#[macro_use]
extern crate nama_crate;

// Cara 3: Macro dalam modul yang sama, guna use
mod matematik {
    macro_rules! kuasa {
        ($b:expr, $e:expr) => { ($b as f64).powi($e) };
    }

    pub(crate) use kuasa; // export ke parent module
}

fn main() {
    use matematik::kuasa;
    println!("{}", kuasa!(2, 10)); // 1024
}
```

---

## Macro dalam Module

```rust
mod utils {
    // Macro dalam module — guna #[macro_export] ATAU pub use
    macro_rules! _log_impl {  // nama dengan _ = "private"
        ($level:literal, $msg:expr) => {
            println!("[{}] {}", $level, $msg);
        };
    }

    pub macro_rules! log_info {
        ($msg:expr) => {
            $crate::utils::_log_impl!("INFO", $msg);
        };
    }

    // Cara lain: export dengan pub use selepas define
    macro_rules! tambah_prefix {
        ($s:expr) => { format!("PREFIX_{}", $s) };
    }

    pub(crate) use tambah_prefix;
}

fn main() {
    utils::log_info!("Program dimulakan");

    let s = utils::tambah_prefix!("test");
    println!("{}", s); // PREFIX_test
}
```

---

# BAB 9: Pattern Lanjutan 🎯

## stringify! & concat!

```rust
fn main() {
    // stringify! — tukar token ke string literal
    let nama = stringify!(Ali Ahmad);
    println!("{}", nama); // "Ali Ahmad"

    let kod = stringify!(let x = 5 + 3;);
    println!("{}", kod);  // "let x = 5 + 3;"

    // concat! — gabung literal semasa compile
    let s = concat!("Hello", " ", "World", "!");
    println!("{}", s); // "Hello World!"

    // Guna dalam macro
    macro_rules! nama_dengan_prefix {
        ($prefix:literal, $nama:ident) => {
            concat!($prefix, "::", stringify!($nama))
        };
    }

    let path = nama_dengan_prefix!("crate", main);
    println!("{}", path); // "crate::main"
}
```

---

## Macro untuk Testing Assertion

```rust
macro_rules! assert_err {
    ($expr:expr) => {
        assert!(
            $expr.is_err(),
            "Jangkakan Err, dapat Ok: {:?}",
            $expr
        )
    };
}

macro_rules! assert_ok {
    ($expr:expr) => {
        assert!(
            $expr.is_ok(),
            "Jangkakan Ok, dapat Err: {:?}",
            $expr
        )
    };
    ($expr:expr, $expected:expr) => {
        {
            let result = $expr;
            assert!(result.is_ok(), "Jangkakan Ok: {:?}", result);
            assert_eq!(result.unwrap(), $expected);
        }
    };
}

fn bahagi(a: i32, b: i32) -> Result<i32, &'static str> {
    if b == 0 { Err("sifar") } else { Ok(a / b) }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_bahagi() {
        assert_ok!(bahagi(10, 2), 5);
        assert_ok!(bahagi(15, 3));
        assert_err!(bahagi(10, 0));
    }
}

fn main() {
    assert_ok!(bahagi(10, 2), 5);
    assert_err!(bahagi(5, 0));
    println!("Semua assertion lulus!");
}
```

---

## Builder Macro

```rust
macro_rules! bina_builder {
    (
        struct $nama:ident {
            $($field:ident: $jenis:ty = $default:expr),* $(,)?
        }
    ) => {
        #[derive(Debug)]
        struct $nama {
            $($field: $jenis,)*
        }

        paste::paste! {
            struct [<$nama Builder>] {
                $($field: $jenis,)*
            }

            impl [<$nama Builder>] {
                fn baru() -> Self {
                    [<$nama Builder>] {
                        $($field: $default,)*
                    }
                }

                $(
                fn $field(mut self, val: $jenis) -> Self {
                    self.$field = val;
                    self
                }
                )*

                fn bina(self) -> $nama {
                    $nama {
                        $(self.$field,)*
                    }
                }
            }
        }
    };
}

// Tanpa paste crate — versi mudah:
macro_rules! struct_dengan_default {
    (
        struct $nama:ident {
            $($field:ident: $jenis:ty = $default:expr),* $(,)?
        }
    ) => {
        #[derive(Debug, Default)]
        struct $nama {
            $($field: $jenis,)*
        }

        impl $nama {
            fn baru() -> Self {
                $nama {
                    $($field: $default,)*
                }
            }
        }
    };
}

struct_dengan_default! {
    struct Konfigurasi {
        host: String = String::from("localhost"),
        port: u16 = 8080,
        debug: bool = false,
        max_sambungan: u32 = 100,
    }
}

fn main() {
    let mut cfg = Konfigurasi::baru();
    println!("{:?}", cfg);
    // Konfigurasi { host: "localhost", port: 8080, debug: false, max_sambungan: 100 }

    cfg.port = 3000;
    cfg.debug = true;
    println!("{:?}", cfg);
}
```

---

## 🧠 Brain Teaser #5 (Advanced)

Tulis macro `impl_from!` yang auto-implement `From` trait untuk wrapper types.

```rust
// Guna selepas define:
struct Meter(f64);
struct Kilogram(f64);

impl_from!(f64 => Meter, Kilogram);
// Sepatutnya sama dengan:
// impl From<f64> for Meter { fn from(v: f64) -> Self { Meter(v) } }
// impl From<f64> for Kilogram { fn from(v: f64) -> Self { Kilogram(v) } }

fn main() {
    let m: Meter = 5.0.into();
    let k: Kilogram = 75.0.into();
}
```

<details>
<summary>👀 Jawapan</summary>

```rust
macro_rules! impl_from {
    ($sumber:ty => $($sasaran:ident),+ $(,)?) => {
        $(
            impl From<$sumber> for $sasaran {
                fn from(v: $sumber) -> Self {
                    $sasaran(v)
                }
            }
        )+
    };
}

struct Meter(f64);
struct Kilogram(f64);
struct Celsius(f64);

impl_from!(f64 => Meter, Kilogram, Celsius);

fn main() {
    let m: Meter    = 1500.0.into();
    let k: Kilogram = 75.5.into();
    let c: Celsius  = 37.0.into();

    println!("{}m, {}kg, {}°C", m.0, k.0, c.0);
}
```

Macro dikembangkan:
```
impl_from!(f64 => Meter, Kilogram, Celsius)
→
impl From<f64> for Meter    { fn from(v: f64) -> Self { Meter(v) } }
impl From<f64> for Kilogram { fn from(v: f64) -> Self { Kilogram(v) } }
impl From<f64> for Celsius  { fn from(v: f64) -> Self { Celsius(v) } }
```
</details>

---

# BAB 10: Mini Project — Bina Testing Framework 🧪

```rust
// Bina framework testing ringkas guna macro_rules!

// ─── Core Macros ──────────────────────────────────────────────

// Assertion macros
macro_rules! jangka_sama {
    ($kiri:expr, $kanan:expr) => {
        {
            let k = $kiri;
            let r = $kanan;
            if k != r {
                println!("  ✘ GAGAL: jangka_sama!({})",
                    stringify!($kiri == $kanan));
                println!("    Kiri:  {:?}", k);
                println!("    Kanan: {:?}", r);
                false
            } else {
                true
            }
        }
    };
}

macro_rules! jangka_benar {
    ($cond:expr) => {
        {
            if !$cond {
                println!("  ✘ GAGAL: jangka_benar!({})",
                    stringify!($cond));
                false
            } else {
                true
            }
        }
    };
}

macro_rules! jangka_palsu {
    ($cond:expr) => {
        {
            if $cond {
                println!("  ✘ GAGAL: jangka_palsu!({})",
                    stringify!($cond));
                false
            } else {
                true
            }
        }
    };
}

macro_rules! jangka_err {
    ($expr:expr) => {
        {
            let r = $expr;
            if r.is_ok() {
                println!("  ✘ GAGAL: jangka_err!({}) dapat Ok",
                    stringify!($expr));
                false
            } else {
                true
            }
        }
    };
}

macro_rules! jangka_ok {
    ($expr:expr) => {
        {
            let r = $expr;
            if r.is_err() {
                println!("  ✘ GAGAL: jangka_ok!({}) dapat Err: {:?}",
                    stringify!($expr), r);
                false
            } else {
                true
            }
        }
    };
}

// Test runner macro
macro_rules! kes_ujian {
    ($nama:ident { $($assertion:expr);+ $(;)? }) => {
        fn $nama() -> (usize, usize) {
            println!("  Ujian: {}", stringify!($nama));
            let mut lulus  = 0usize;
            let mut gagal  = 0usize;
            $(
                if $assertion {
                    lulus += 1;
                } else {
                    gagal += 1;
                }
            )+
            (lulus, gagal)
        }
    };
}

// Suite runner
macro_rules! jalankan_ujian {
    ($($ujian:ident),+ $(,)?) => {
        {
            println!("\n{'═'*50}");
            println!("  SUITE UJIAN");
            println!("{'═'*50}");

            let mut jumlah_lulus = 0usize;
            let mut jumlah_gagal = 0usize;

            $(
                let (l, g) = $ujian();
                jumlah_lulus += l;
                jumlah_gagal += g;
                if g == 0 {
                    println!("  ✔ {} ({} assertion lulus)", stringify!($ujian), l);
                } else {
                    println!("  ✘ {} ({} lulus, {} gagal)", stringify!($ujian), l, g);
                }
            )+

            println!("{'─'*50}");
            println!("  Jumlah: {} lulus, {} gagal", jumlah_lulus, jumlah_gagal);
            if jumlah_gagal == 0 {
                println!("  🎉 Semua ujian lulus!");
            } else {
                println!("  ⚠ Ada {} ujian gagal!", jumlah_gagal);
            }
            println!("{'═'*50}");

            jumlah_gagal == 0
        }
    };
}

// ─── Fungsi yang nak diuji ────────────────────────────────────

fn tambah(a: i32, b: i32) -> i32 { a + b }
fn bahagi(a: f64, b: f64) -> Result<f64, &'static str> {
    if b == 0.0 { Err("Bahagi dengan sifar") } else { Ok(a / b) }
}
fn adalah_prima(n: u32) -> bool {
    if n < 2 { return false; }
    if n == 2 { return true; }
    if n % 2 == 0 { return false; }
    !(3..=(n as f64).sqrt() as u32).step_by(2).any(|i| n % i == 0)
}
fn terbalik(s: &str) -> String { s.chars().rev().collect() }

// ─── Define Test Cases ────────────────────────────────────────

kes_ujian!(ujian_tambah {
    jangka_sama!(tambah(2, 3),   5);
    jangka_sama!(tambah(0, 0),   0);
    jangka_sama!(tambah(-1, 1),  0);
    jangka_sama!(tambah(100, 50), 150)
});

kes_ujian!(ujian_bahagi {
    jangka_ok!(bahagi(10.0, 2.0));
    jangka_sama!(bahagi(10.0, 2.0).unwrap(), 5.0);
    jangka_err!(bahagi(5.0, 0.0));
    jangka_ok!(bahagi(0.0, 5.0))
});

kes_ujian!(ujian_prima {
    jangka_benar!(adalah_prima(2));
    jangka_benar!(adalah_prima(3));
    jangka_benar!(adalah_prima(7));
    jangka_benar!(adalah_prima(97));
    jangka_palsu!(adalah_prima(1));
    jangka_palsu!(adalah_prima(4));
    jangka_palsu!(adalah_prima(100))
});

kes_ujian!(ujian_terbalik {
    jangka_sama!(terbalik("hello"), "olleh".to_string());
    jangka_sama!(terbalik("Rust"),  "tsuR".to_string());
    jangka_sama!(terbalik(""),      "".to_string());
    jangka_sama!(terbalik("a"),     "a".to_string())
});

// Ujian yang akan gagal (untuk demo)
kes_ujian!(ujian_demo_gagal {
    jangka_sama!(tambah(2, 2), 4);      // ✔ betul
    jangka_sama!(tambah(2, 2), 5);      // ✘ salah — demo gagal
    jangka_benar!(2 + 2 == 4)           // ✔ betul
});

// ─── Main ──────────────────────────────────────────────────────

fn main() {
    let semua_lulus = jalankan_ujian!(
        ujian_tambah,
        ujian_bahagi,
        ujian_prima,
        ujian_terbalik,
        ujian_demo_gagal,
    );

    std::process::exit(if semua_lulus { 0 } else { 1 });
}
```

---

# 📋 Rujukan Pantas — Macros Cheat Sheet

## Struktur Asas

```rust
macro_rules! nama {
    // Pattern → Expansion
    ($x:expr) => { /* kod */ };
    ($a:expr, $b:expr) => { /* kod berbeza */ };
    () => { /* tiada argumen */ };
}
```

## Fragment Specifiers

```
$x:expr     → sebarang expression
$x:ident    → identifier (nama)
$x:ty       → type
$x:pat      → pattern
$x:stmt     → statement
$x:block    → { ... }
$x:item     → fn/struct/enum/impl
$x:literal  → 42, "hello", true
$x:tt       → token tree (paling fleksibel)
$x:lifetime → 'a
$x:vis      → pub, pub(crate)
$x:path     → std::vec::Vec
$x:meta     → derive(Debug)
```

## Repetition

```
$( ... )*   → sifar atau lebih kali
$( ... )+   → satu atau lebih kali
$( ... )?   → sifar atau satu kali (optional)

Dengan separator:
$( ... ),*  → sifar atau lebih, dipisah koma
$( ... ),+  → satu atau lebih, dipisah koma
$(,)?       → optional trailing comma
```

## Built-in Macros Berguna dalam Expansion

```rust
stringify!($x)          // token → &str
concat!("a", "b")       // gabung literal → &str
format_args!(...)       // macam format!
file!()                 // nama fail semasa
line!()                 // nombor baris semasa
column!()               // nombor lajur semasa
module_path!()          // path module semasa
env!("VAR")             // nilai environment variable semasa compile
```

## Tips

```
1. Wrap expansion dengan () untuk elak operator precedence bug
2. Guna $(,)? untuk trailing comma yang optional
3. Guna @ident pattern untuk internal rules
4. $crate:: untuk rujuk crate sendiri dalam #[macro_export]
5. Debug dengan: cargo expand (crate cargo-expand)
6. Test macro dengan unit tests macam biasa
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **`impl_display!`** — auto-implement `Display` untuk enum dengan label setiap variant
2. **`bitmask!`** — generate bitflag constants dan operasi AND/OR/XOR
3. **`sql!`** — macro yang validate sintaks SQL asas pada compile time
4. **`route!`** — mini web framework macro: `route!(GET "/api/users" => handler_fn)`
5. **`fsm!`** — Finite State Machine generator dari deklarasi states dan transitions

---

*Macros dalam Rust — dari `macro_rules!` mudah hingga testing framework sendiri. Selamat belajar!* 🦀
