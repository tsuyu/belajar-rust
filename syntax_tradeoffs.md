# ⚖️ Trade-off Syntax dalam Rust — Panduan Jujur

> "Rust bukan bahasa yang mudah untuk ditulis,
>  tapi ia bahasa yang mudah untuk dibaca bila anda faham kenapa."
> Setiap sintaks yang terasa 'berat' ada sebab yang konkrit.

---

## Premis: Rust Bayar Sekarang, Simpan Kemudian

```
Python/PHP/JS:
  Tulis cepat → Run → Debug → Fix → Deploy → Debug production → Fix

Rust:
  Tulis → Compiler complain → Fix → Fix lagi → Fix lagi → Run → Deploy
  (jarang ada bug dalam production)

Ini adalah trade-off SEDAR yang Rust buat:
  "Kita akan buat kamu bayar di compile time,
   supaya kamu tidak perlu bayar di runtime."

Setiap 'berat' dalam syntax Rust ada jawapan kepada soalan:
  "Tanpa ini, apa yang boleh silap?"
```

---

## Peta Pembelajaran

```
Bab 1  → Ownership & Borrow — Kenapa Semua Ini?
Bab 2  → Lifetime — Tanda Pagar yang Memeningkan
Bab 3  → Type Annotations — Bila Verbose, Bila Tidak
Bab 4  → Error Handling — Result vs Exception
Bab 5  → Generics — Kuasa yang Berbayar
Bab 6  → Pattern Matching — Verbose tapi Selamat
Bab 7  → Trait Bounds — Complex tapi Expressif
Bab 8  → Macro vs Function — Dua Dunia
Bab 9  → Closures — Tiga Jenis FnOnce/FnMut/Fn
Bab 10 → Panduan: Terima Trade-off atau Lawan?
```

---

# BAB 1: Ownership & Borrow — Kenapa Semua Ini? 🏠

## Trade-off: Kebebasan vs Keselamatan

```rust
// ── PYTHON (bebas, tapi ada kos tersembunyi) ──────────────────
// def proses(data):
//     result = transform(data)
//     return result
//     # data masih ada, result ada, semua ada
//     # GC akan bersihkan bila-bila masa
//     # Boleh ada memory leak kalau ada circular reference
//     # Thread safety? Tanggungjawab programmer

// ── RUST (berbayar sekarang, selamat kemudian) ────────────────

// Trade-off 1: Kena fikir tentang ownership
fn proses(data: Vec<u8>) -> String {
    // data MOVE ke sini — pemanggil tidak boleh guna lagi
    String::from_utf8_lossy(&data).into_owned()
}   // data di-drop di sini secara automatik

// Kos: pemanggil kena fikir "adakah saya masih perlu data?"
// Manfaat: tiada memory leak, tiada double-free, deterministik

fn main() {
    let data = vec![72, 101, 108, 108, 111];

    // Pilihan 1: Bagi ownership (move)
    let s = proses(data);
    // data tidak boleh digunakan lagi — kena fikir!

    // Pilihan 2: Bagi borrow (pinjam)
    fn proses_pinjam(data: &[u8]) -> String {
        String::from_utf8_lossy(data).into_owned()
    }
    let data2 = vec![72, 101, 108];
    let s2 = proses_pinjam(&data2); // data2 masih valid!
    println!("{}", data2.len()); // OK!

    // Pilihan 3: Clone (mahal, tapi mudah)
    let data3 = vec![72, 101, 108];
    let s3 = proses(data3.clone()); // clone dahulu
    println!("{}", data3.len()); // OK, data3 masih ada
}

// Kos Ownership:
//   ✗ Kena fikir bila setiap function call
//   ✗ Kena pilih: move / borrow / clone
//   ✗ Compiler akan complaint berkali-kali semasa belajar
//
// Manfaat Ownership:
//   ✔ TIADA memory leak (tanpa GC!)
//   ✔ TIADA dangling pointer
//   ✔ TIADA use-after-free
//   ✔ TIADA double-free
//   ✔ Thread safety dijamin oleh compiler
```

---

## 🧠 Brain Teaser #1

Kenapa Rust TIDAK pilih GC seperti Java/Python untuk elak semua ini?

<details>
<summary>👀 Jawapan</summary>

```
GC menyelesaikan masalah dengan kos yang tersembunyi:

1. LATENCY: GC boleh "stop the world" pada bila-bila masa
   → Web server dengan GC boleh ada spike latency 100ms+
   → Game dengan GC boleh ada frame stutter
   → Real-time system dengan GC = tidak sesuai

2. MEMORY: GC biasanya pegang lebih memory dari yang diperlukan
   → Java app biasa guna 2-5× lebih memory dari Rust equivalent
   → GC perlu heap yang besar untuk bekerja dengan efisien

3. PREDICTABILITY: Bila GC akan run? Tidak tahu.
   → Destructor tidak dijamin dipanggil segera
   → Resource (file, connection) mungkin tidak dilepas segera

4. THREAD SAFETY: GC tidak selesaikan data race
   → Python ada GIL (satu thread pada satu masa untuk safety)
   → Java/Go boleh ada data race walaupun dengan GC

Rust trade-off:
  Bayar di SYNTAX (programmer kena fikir ownership)
  Dapat: zero-cost abstraction, deterministic cleanup,
         thread safety, tiada GC overhead

Bahasa lain trade-off:
  Dapat SYNTAX mudah
  Bayar di: runtime overhead, unpredictable latency,
            lebih memory usage, tiada thread safety guarantee

Untuk sistem kritikal (OS, embedded, game engine, web server
dengan latency SLA) → Rust lebih baik.
Untuk scripting, prototyping, data analysis → Python/JS fine.
```
</details>

---

# BAB 2: Lifetime — Tanda Pagar yang Memeningkan ⏳

## Trade-off: Explicit vs Implicit

```rust
// ── TRADE-OFF ─────────────────────────────────────────────────
// Kebanyakan bahasa: lifetime IMPLICIT (programmer tidak nampak)
// Rust: lifetime EXPLICIT bila compiler tidak boleh infer

// ── Bila Rust BOLEH infer (tiada annotation diperlukan) ───────

// Kes mudah: return value dari satu argument
fn pertama_perkataan(s: &str) -> &str {
    s.split_whitespace().next().unwrap_or("")
    // Compiler tahu: return hidup selama `s` hidup
    // Tiada annotation diperlukan!
}

// Kes mudah: fungsi tanpa return reference
fn panjang(s: &str) -> usize {
    s.len()
    // Return adalah usize (bukan reference) — tiada masalah
}

// ── Bila PERLU annotation (multiple references) ───────────────

// ❌ TIDAK COMPILE — compiler tidak tahu reference mana yang direturn
// fn terpanjang(a: &str, b: &str) -> &str {
//     if a.len() >= b.len() { a } else { b }
// }

// ✔ Dengan lifetime annotation
fn terpanjang<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() >= b.len() { a } else { b }
}
// 'a bermaksud: "return hidup selama yang TERPENDEK antara a dan b"

fn main() {
    let s1 = String::from("panjang sekali");
    let hasil;
    {
        let s2 = String::from("xyz");
        hasil = terpanjang(&s1, &s2);
        println!("{}", hasil); // OK — dalam scope s2
    }
    // println!("{}", hasil); // ERROR — s2 sudah drop!
    // Compiler tangkap ini BERKAT lifetime annotation!
}

// ── Lifetime dalam Struct ─────────────────────────────────────

// ❌ Tanpa lifetime — tidak boleh simpan reference dalam struct
// struct Petikan {
//     teks: &str, // error: missing lifetime specifier
// }

// ✔ Dengan lifetime
struct Petikan<'a> {
    teks: &'a str, // 'a bermaksud: Petikan tidak boleh hidup lebih lama dari teks
}

impl<'a> Petikan<'a> {
    fn papar(&self) -> &str {
        self.teks
    }
}

fn main_struct() {
    let teks = String::from("Petikan penting ini");
    let p = Petikan { teks: &teks };
    println!("{}", p.papar());
    // p tidak boleh hidup lebih lama dari `teks`!
}
```

---

## Lifetime Trade-off — Bila Berbaloi?

```rust
// ── Situasi 1: Lifetime MEMANG diperlukan (berbaloi) ──────────
// Parser, interpreter, database query builder — semua perlu
// return references ke input data tanpa copy!

struct Parser<'a> {
    input:     &'a str,
    kedudukan: usize,
}

impl<'a> Parser<'a> {
    fn token_seterusnya(&mut self) -> Option<&'a str> {
        let mula = self.kedudukan;
        while self.kedudukan < self.input.len()
            && !self.input[self.kedudukan..].starts_with(' ')
        {
            self.kedudukan += 1;
        }
        if mula == self.kedudukan { return None; }
        let token = &self.input[mula..self.kedudukan];
        self.kedudukan += 1; // skip space
        Some(token) // return reference ke input ASAL, tiada copy!
    }
}

// Tanpa lifetime: perlu clone setiap token → banyak allocation
// Dengan lifetime: zero-copy parsing → sangat laju!

// ── Situasi 2: Lifetime yang TIDAK diperlukan (elak!) ─────────
// Kalau boleh return owned value atau guna .clone() → buat sahaja
// Jangan tambah lifetime hanya kerana terasa "lebih Rust"

// ❌ Terlalu clever (lifetime tidak perlu)
fn tambah_satu<'a>(v: &'a mut Vec<i32>) -> &'a Vec<i32> {
    v.push(999);
    v // return reference ke v
}
// Kenapa return reference? Lebih baik:

// ✔ Mudah dan jelas
fn tambah_satu_mudah(v: &mut Vec<i32>) {
    v.push(999);
    // Caller masih ada v, tidak perlu return
}
```

---

# BAB 3: Type Annotations — Bila Verbose, Bila Tidak 📝

## Rust Type Inference — Lebih Baik dari Sangka

```rust
// Rust mempunyai type inference yang KUAT
// Kebanyakan masa tidak perlu annotate

fn main() {
    // ── Rust boleh infer ini semua ────────────────────────────
    let x = 42;                          // i32 (default integer)
    let y = 3.14;                        // f64 (default float)
    let s = "hello".to_string();         // String
    let v = vec![1, 2, 3];              // Vec<i32>
    let b = true;                        // bool

    // Inference dari penggunaan!
    let mut senarai = Vec::new(); // Vec<???> — belum tahu
    senarai.push(42i32);         // Sekarang: Vec<i32>

    // Inference dalam chain
    let hasil = v.iter()
        .filter(|&&n| n > 1)     // tidak perlu annotate n
        .map(|&n| n * 2)         // tidak perlu annotate
        .collect::<Vec<_>>();    // perlu hint di sini!

    // ── Bila PERLU annotate ───────────────────────────────────

    // 1. Bila ambiguous (banyak kemungkinan)
    let n: i64 = 42;             // specify: i64, bukan i32
    let s: &str = "hello";       // vs String

    // 2. Bila collect() — compiler tidak tahu nak collect ke apa
    let v: Vec<i32> = vec![1, 2, 3].into_iter().collect();
    // Atau:
    let v = vec![1, 2, 3].into_iter().collect::<Vec<i32>>();
    // Atau:
    let v = vec![1, 2, 3].into_iter().collect::<Vec<_>>(); // _ = infer

    // 3. Dalam return type fungsi (selalu perlu)
    fn tambah(a: i32, b: i32) -> i32 { a + b }

    // 4. Bila parse() — tahu nak parse ke apa?
    let n: u32 = "42".parse().unwrap();
    // Atau:
    let n = "42".parse::<u32>().unwrap();

    // ── Turbofisch ::<> ──────────────────────────────────────
    // Nama rasmi: "turbofish" ::<T>
    // Guna bila function perlu tahu type parameter
    "42".parse::<i32>().unwrap();    // turbofish
    Vec::<i32>::new();               // turbofish
    std::mem::size_of::<String>();   // turbofish
}
```

---

## Bila Annotation Lebih Baik dari Inference

```rust
// ── Trade-off: Implisit vs Eksplisit ─────────────────────────

// Inference yang mengelirukan:
fn fungsi_a() {
    let mut m = std::collections::HashMap::new();
    m.insert("kunci", 42); // baru tahu: HashMap<&str, i32>
    // Pembaca yang melihat `let mut m = ...` tidak tahu type!
}

// Annotation yang jelas:
fn fungsi_b() {
    let mut m: std::collections::HashMap<&str, i32> = std::collections::HashMap::new();
    // Terus tahu apa yang disimpan
    m.insert("kunci", 42);
}

// Atau: guna import untuk kurangkan verbosity
use std::collections::HashMap;

fn fungsi_c() {
    let mut m: HashMap<&str, i32> = HashMap::new();
    // Lebih pendek dan jelas
}

// Panduan:
// ✔ Type inference untuk variable sementara dalam fungsi
// ✔ Explicit type untuk public API (return type, parameter)
// ✔ Explicit type bila type tidak obvious dari RHS
// ✔ Explicit type untuk dokumentasi dalam kod kompleks
// ✗ JANGAN annotate kalau jelas dari context
// ✗ JANGAN annotate SETIAP variable (verbose tidak perlu)
```

---

# BAB 4: Error Handling — Result vs Exception 🚨

## Trade-off: Explicit vs Implicit

```rust
// ── JAVA (exception — implicit, mudah tulis, sukar trace) ─────
// void proses() throws IOException {
//     File f = new File("data.txt");
//     Scanner sc = new Scanner(f);   // checked exception
//     int n = Integer.parseInt(sc.nextLine()); // unchecked exception (hidden!)
//     // Stack trace panjang, sukar tahu di mana exception berlaku
// }

// ── RUST (Result — explicit, verbose, tapi JELAS) ─────────────

// ❌ Verbose tapi selamat
fn proses_verbose() -> Result<i32, Box<dyn std::error::Error>> {
    let kandungan = match std::fs::read_to_string("data.txt") {
        Ok(k) => k,
        Err(e) => return Err(Box::new(e)),
    };

    let n = match kandungan.trim().parse::<i32>() {
        Ok(n) => n,
        Err(e) => return Err(Box::new(e)),
    };

    Ok(n * 2)
}

// ✔ Dengan ? operator — masih explicit tapi ringkas
fn proses() -> Result<i32, Box<dyn std::error::Error>> {
    let kandungan = std::fs::read_to_string("data.txt")?;
    let n: i32 = kandungan.trim().parse()?;
    Ok(n * 2)
}

// ── Kos ─────────────────────────────────────────────────────
// ✗ Return type mesti Result
// ✗ Kena pikir tentang setiap error
// ✗ Chain error types mungkin perlu conversion

// ── Manfaat ─────────────────────────────────────────────────
// ✔ Error path KELIHATAN dalam kod
// ✔ Compiler paksa handle setiap error (atau explicitly ignore)
// ✔ Error type tahu (bukan generic Exception)
// ✔ Performance: tiada stack unwinding overhead

// ── Trade-off boleh dikurangkan dengan anyhow ─────────────────
use anyhow::{Result, Context};

fn proses_anyhow() -> Result<i32> {
    let kandungan = std::fs::read_to_string("data.txt")
        .context("Gagal baca data.txt")?;
    let n: i32 = kandungan.trim().parse()
        .context("Fail tidak mengandungi integer yang sah")?;
    Ok(n * 2)
}
// Hampir seringkas Python! Tapi masih jelas.
```

---

## Kenapa Exception Lebih Berbahaya

```rust
// Dalam bahasa dengan exception:
// function yang kelihatan "biasa" boleh throw exception!
// Programmer kena hafal mana yang boleh throw.

// Rust:
//   fn cari(id: u32) -> Pekerja        = PASTI berjaya
//   fn cari(id: u32) -> Option<Pekerja> = mungkin tiada
//   fn cari(id: u32) -> Result<Pekerja, Error> = mungkin gagal dengan sebab

// Type signature IS the documentation!
// Tidak perlu baca source code atau JavaDoc untuk tahu kemungkinan fail.

fn demo_signature() {
    // Ini PASTI return String (tidak boleh gagal)
    fn format_nama(nama: &str) -> String {
        format!("Pekerja: {}", nama)
    }

    // Ini MUNGKIN return String (boleh tiada)
    fn cari_bahagian(id: u32) -> Option<String> {
        if id == 1 { Some("ICT".into()) } else { None }
    }

    // Ini MUNGKIN gagal (boleh ada error dengan sebab)
    fn baca_konfigurasi(laluan: &str) -> Result<String, std::io::Error> {
        std::fs::read_to_string(laluan)
    }

    // Caller TAHU dari type signature — tidak perlu baca docs!
}
```

---

# BAB 5: Generics — Kuasa yang Berbayar 🔧

## Trade-off: Zero-cost vs Monomorphization

```rust
// ── Generics dalam Rust (compile-time, zero-cost) ─────────────

// SATU definisi:
fn terbesar<T: PartialOrd>(senarai: &[T]) -> &T {
    let mut terbesar = &senarai[0];
    for item in &senarai[1..] {
        if item > terbesar { terbesar = item; }
    }
    terbesar
}

// Compiler GENERATE kod berasingan untuk setiap type:
fn terbesar_i32(senarai: &[i32]) -> &i32 { /* ... */ }
fn terbesar_f64(senarai: &[f64]) -> &f64 { /* ... */ }
fn terbesar_char(senarai: &[char]) -> &char { /* ... */ }

// Kos:
//   ✗ Binary size lebih besar (monomorphization)
//   ✗ Compile time lebih lama
//   ✗ Syntax dengan bounds boleh jadi panjang

// Manfaat:
//   ✔ TIADA runtime overhead (no boxing, no vtable)
//   ✔ Compiler boleh optimize setiap version secara berasingan
//   ✔ Type safety pada compile time

// ── Bila generic bounds menjadi kompleks ─────────────────────

// Ini boleh jadi terlalu panjang:
fn fungsi_kompleks<T, U, V>(a: T, b: U) -> V
where
    T: Clone + Debug + Display + PartialOrd + Into<V>,
    U: Clone + Debug + Display + PartialOrd + Into<V>,
    V: Debug + Default + Add<Output = V>,
{
    todo!()
}

// Penyelesaian: guna trait objects (dyn Trait) bila type flexibility lebih penting
fn fungsi_dinamik(a: &dyn std::fmt::Display, b: &dyn std::fmt::Display) {
    println!("{} {}", a, b);
    // Kos: runtime overhead (vtable)
    // Manfaat: lebih mudah baca, binary lebih kecil
}

// Penyelesaian 2: buat trait sendiri yang gabung beberapa
trait ProfilLengkap: Clone + std::fmt::Debug + std::fmt::Display + PartialOrd {}
impl<T: Clone + std::fmt::Debug + std::fmt::Display + PartialOrd> ProfilLengkap for T {}

fn fungsi_bersih<T: ProfilLengkap>(x: T) { println!("{}", x); }
```

---

# BAB 6: Pattern Matching — Verbose tapi Selamat 🎯

## Trade-off: Safety vs Terseness

```rust
// ── JAVA/PHP (switch — mudah tapi tidak selamat) ──────────────
// switch (status) {
//     case "AKTIF": proses_aktif(); break;
//     case "CUTI":  proses_cuti(); break;
//     // Lupa "TAMAT_KONTRAK"? Tidak ada compile error!
//     // Runtime: tiada yang dipanggil, bug tersembunyi!
// }

// ── RUST (match — exhaustive, compiler paksa semua kes) ───────

#[derive(Debug)]
enum StatusPekerja {
    Aktif,
    Cuti { hari: u32 },
    TamatKontrak,
    Digantung { sebab: String },
}

fn proses_status(status: &StatusPekerja) -> String {
    match status {
        StatusPekerja::Aktif               => "Pekerja aktif".into(),
        StatusPekerja::Cuti { hari }       => format!("Cuti {} hari", hari),
        StatusPekerja::TamatKontrak        => "Kontrak tamat".into(),
        StatusPekerja::Digantung { sebab } => format!("Digantung: {}", sebab),
        // Lupa satu variant → COMPILE ERROR!
        // "non-exhaustive patterns"
    }
}

// Kos:
//   ✗ Setiap variant mesti dihandle (boleh terasa repetitive)
//   ✗ Syntax lebih panjang dari switch/case

// Manfaat:
//   ✔ Compiler paksa handle SEMUA variant
//   ✔ Bila tambah variant baru → compiler tunjukkan semua tempat yang kena ubah
//   ✔ Pattern matching lebih ekspresif dari switch

fn main() {
    let s = StatusPekerja::Cuti { hari: 3 };
    println!("{}", proses_status(&s));

    // __ untuk ignore yang tidak perlu
    match s {
        StatusPekerja::Aktif   => println!("Aktif"),
        StatusPekerja::Cuti { .. } => println!("Sedang bercuti"),
        _ => println!("Status lain"), // catch-all
    }
}
```

---

# BAB 7: Trait Bounds — Complex tapi Expressif 📐

## Trade-off: Verbosity vs Precision

```rust
use std::fmt;

// ── Simple trait bound ────────────────────────────────────────
fn cetak<T: fmt::Display>(x: T) {
    println!("{}", x);
}

// ── Lebih kompleks bila perlu banyak constraint ───────────────
fn proses_kompleks<T>(x: &T) -> String
where
    T: fmt::Debug       // boleh debug print
     + fmt::Display     // boleh display print
     + Clone            // boleh di-clone
     + Send + Sync,     // thread-safe
{
    format!("{:?} = {}", x, x)
}

// ── Versus dynamic dispatch ───────────────────────────────────
fn proses_dinamik(x: &(dyn fmt::Display + fmt::Debug + Send + Sync)) -> String {
    format!("{:?} = {}", x, x)
}

// Generic (T: Trait):
//   ✔ Zero-cost (monomorphization)
//   ✔ Compiler optimize per type
//   ✗ Lebih besar binary
//   ✗ Syntax panjang bila banyak bounds

// dyn Trait:
//   ✔ Binary lebih kecil
//   ✔ Syntax lebih ringkas
//   ✗ Runtime overhead (vtable lookup)
//   ✗ Tidak semua trait boleh jadi dyn (object safety)

// ── Pilihan pragmatik ─────────────────────────────────────────
// Hot path, performance kritikal → generic
// Plugin system, heterogeneous collection → dyn Trait
// Library API → generic (lebih flexible untuk caller)
// Application code → either, pilih yang lebih readable
```

---

# BAB 8: Macro vs Function — Dua Dunia 🎭

## Trade-off: Power vs Complexity

```rust
// ── Kenapa Rust ada macro? ────────────────────────────────────
// Fungsi biasa ada limitasi:
//   1. Bilangan argument mesti tetap
//   2. Argument mesti mempunyai type
//   3. Tidak boleh generate kod

// ── println! — kenapa macro dan bukan fungsi? ─────────────────
// fn println(format: ???, ...args) ???
// Masalah: format string checked AT COMPILE TIME!
// Bagaimana fungsi biasa boleh check "ada {} atau tidak"?

// println!("{} {}", x, y)     → COMPILE ERROR kalau x tidak ada
// println!("{}", x, y)        → COMPILE ERROR (extra argument)
// println!("{} {} {}", x, y)  → COMPILE ERROR (kurang argument)

// Ini hanya boleh dengan macro — compile-time code generation!

// ── Macro yang mengelirukan ───────────────────────────────────
// Trade-off: macro boleh buat apa sahaja, tapi sukar debug

macro_rules! buat_getter {
    ($field:ident, $type:ty) => {
        pub fn $field(&self) -> &$type {
            &self.$field
        }
    };
}

struct Data {
    nama:  String,
    nilai: i32,
}

impl Data {
    buat_getter!(nama,  String);
    buat_getter!(nilai, i32);
}

// Kos Macro:
//   ✗ Syntax aneh ($ident, $expr, $tt)
//   ✗ Error message boleh mengelirukan
//   ✗ IDE support kurang baik dari fungsi
//   ✗ Sukar debug

// Manfaat Macro:
//   ✔ Code generation (kurang boilerplate)
//   ✔ Variadic arguments (vec!, println!)
//   ✔ Compile-time checks (format strings)
//   ✔ DSL (domain-specific languages)

// Panduan: Guna fungsi dulu. Guna macro HANYA bila fungsi tidak cukup.
```

---

# BAB 9: Closures — Tiga Jenis yang Mengelirukan 🔐

## Trade-off: Precision vs Complexity

```rust
// Rust ada TIGA jenis closure trait:
//   Fn     — boleh dipanggil berkali-kali, borrow immutable
//   FnMut  — boleh dipanggil berkali-kali, borrow mutable
//   FnOnce — boleh dipanggil SEKALI SAHAJA, consume capture

// Kenapa perlu TIGA? Kerana ownership!

// ── FnOnce — closure yang consume capture ─────────────────────
let nama = String::from("Ali");

let salam = move || {
    println!("Halo, {}!", nama);
    drop(nama); // consume nama!
};

salam(); // OK
// salam(); // ERROR! FnOnce sudah consumed

// ── FnMut — closure yang ubah capture ────────────────────────
let mut kiraan = 0;
let mut tambah = || {
    kiraan += 1; // ubah kiraan!
    kiraan
};

println!("{}", tambah()); // 1
println!("{}", tambah()); // 2
// let _borrow = &kiraan; // ERROR! tambah masih meminjam kiraan

// ── Fn — closure yang hanya baca ─────────────────────────────
let sempadan = 10;
let lebih_besar = |n: i32| n > sempadan; // borrow sempadan (immutable)

println!("{}", lebih_besar(15)); // true
println!("{}", lebih_besar(5));  // false
let juga_baca = sempadan + 1;    // OK! sempadan tidak di-consume

// ── Kos ─────────────────────────────────────────────────────
// ✗ Programmer kena faham perbezaan Fn/FnMut/FnOnce
// ✗ Error message bila salah guna boleh mengelirukan
// ✗ Callback API perlu specify mana satu

// ── Manfaat ──────────────────────────────────────────────────
// ✔ Compiler memaksa penggunaan yang betul
// ✔ Thread safety (FnOnce tidak boleh dihantar ke banyak thread)
// ✔ No dangling references dalam closure

// ── Panduan ───────────────────────────────────────────────────
// Tulis Fn dulu dalam API
// Kalau compiler complaint → naik ke FnMut
// Kalau masih complaint → FnOnce
// Paling restrictive yang berjalan = pilihan terbaik
fn guna_callback<F: Fn(i32) -> i32>(f: F, nilai: i32) -> i32 {
    f(nilai)
}
```

---

# BAB 10: Panduan — Terima Trade-off atau Lawan? 🎯

## Bila Rust "Berat" — Apa Yang Patut Dibuat

```rust
// ══════════════════════════════════════════════════════════════
// TRADE-OFF 1: Borrow Checker Complaint
// ══════════════════════════════════════════════════════════════

// ❌ LAWAN (clone semata-mata untuk puas hati compiler)
struct Sistem { data: Vec<String> }

impl Sistem {
    fn proses_buruk(&self) -> Vec<String> {
        self.data.iter()
            .map(|s| s.clone()) // clone tidak perlu!
            .filter(|s| s.len() > 3)
            .collect()
    }
}

// ✔ TERIMA (faham kenapa dan betulkan dengan betul)
impl Sistem {
    fn proses_baik(&self) -> Vec<&str> {
        self.data.iter()
            .map(|s| s.as_str()) // return reference!
            .filter(|s| s.len() > 3)
            .collect()
    }

    fn proses_dengan_ownership(self) -> Vec<String> {
        // consume self, return owned data
        self.data.into_iter()
            .filter(|s| s.len() > 3)
            .collect()
    }
}

// ══════════════════════════════════════════════════════════════
// TRADE-OFF 2: Error Handling Verbose
// ══════════════════════════════════════════════════════════════

// ❌ LAWAN (unwrap semua kerana penat)
fn baca_konfigurasi_buruk() -> String {
    std::fs::read_to_string("konfig.toml").unwrap()
    // Akan panic bila fail tidak ada!
}

// ✔ TERIMA (guna ? dan anyhow untuk kurangkan verbosity)
fn baca_konfigurasi_baik() -> anyhow::Result<String> {
    Ok(std::fs::read_to_string("konfig.toml")?)
}

// ✔ ATAU terima bahawa ini memang perlu verbose untuk konteks ini
fn baca_konfigurasi_jelas() -> String {
    match std::fs::read_to_string("konfig.toml") {
        Ok(k)  => k,
        Err(e) => {
            eprintln!("Tidak dapat baca konfig.toml: {}", e);
            eprintln!("Sila buat fail konfig.toml dalam direktori ini.");
            std::process::exit(1);
        }
    }
}

// ══════════════════════════════════════════════════════════════
// TRADE-OFF 3: Lifetime Complexity
// ══════════════════════════════════════════════════════════════

// Pilihan A: Guna owned data (elak lifetime sepenuhnya)
struct KonfigA {
    host: String,    // owned, tiada lifetime
    laluan: String,  // owned, tiada lifetime
}

// Pilihan B: Guna lifetime (tiada copy, lebih efisien)
struct KonfigB<'a> {
    host: &'a str,    // borrow, perlu lifetime
    laluan: &'a str,
}

// Bila pilih A: bila struct perlu hidup bebas / disimpan
// Bila pilih B: bila struct sementara, data dari sumber lain

// ══════════════════════════════════════════════════════════════
// TRADE-OFF 4: Verbosity Umum
// ══════════════════════════════════════════════════════════════

// Rust lebih verbose dari Python, tapi tidak semestinya buruk:

// Python (ringkas):
// result = [(x, y) for x in xs for y in ys if x > y]

// Rust (lebih panjang):
let result: Vec<(i32, i32)> = xs.iter()
    .flat_map(|&x| {
        ys.iter()
            .filter(move |&&y| x > y)
            .map(move |&y| (x, y))
    })
    .collect();

// Atau dengan nested for (lebih jelas):
let mut result = Vec::new();
for &x in &xs {
    for &y in &ys {
        if x > y { result.push((x, y)); }
    }
}

// PANDUAN: Jangan kejar "pendek" — kejar "jelas"
// Rust yang jelas > Rust yang pendek
// Rust yang jelas > Python yang pendek (kerana safety)
```

---

## Ringkasan Trade-offs

```
SYNTAX RUST                    KOS                    MANFAAT
═══════════════════════════════════════════════════════════════════
Ownership & borrow           Fikir setiap assign    Tiada memory leak
                                                    Thread safety
                                                    Deterministic cleanup

Lifetime annotation          Verbose untuk struct   Zero-copy parsing
                             dengan reference        Return ref tanpa alloc

Explicit error (Result)      Setiap ? perlu         Error path jelas
                             return type            Compiler paksa handle

Exhaustive match             Handle semua variant   Tiada forgotten case
                                                    Compiler bantu refactor

Generic bounds               Syntax panjang         Zero-cost abstraction
                             Compile lebih lama      Per-type optimization

Macro complexity             Sukar debug            Variadic args
                             Sukar baca             Code generation
                                                    Compile-time checks

FnOnce/FnMut/Fn              Perlu faham 3 jenis   Thread safety guarantee
                                                    No dangling closure
═══════════════════════════════════════════════════════════════════

PRINSIP AKHIR:
  Rust bayar sekarang (syntax complexity)
  Supaya tidak bayar kemudian (runtime bugs, crashes, security issues)

  Setiap "berat" dalam syntax Rust ada jawapan kepada:
  "Tanpa ini, apa yang boleh silap secara diam-diam?"
```

---

## Panduan Praktis: Lawan atau Terima?

```
LAWAN (betulkan dengan cara yang betul):
  → Jangan .clone() semata-mata untuk escape borrow
  → Jangan .unwrap() semata-mata untuk escape Result
  → Jangan padam lifetime dengan Arc semata-mata untuk convenience
  → Fahami kenapa compiler complaint, betulkan dari punca

TERIMA (ini adalah feature, bukan bug):
  → Ownership — ianya melindungi anda dari memory bug
  → Exhaustive match — ianya akan menyelamatkan anda bila refactor
  → Result<T,E> — ianya memaksa anda fikir tentang error
  → Verbose generics — ianya lebih precise dari duck typing

SHORTCUT yang LEGITIMATE:
  → anyhow untuk kurangkan error type boilerplate
  → Arc<T> untuk kurangkan lifetime complexity
  → #[derive(...)] untuk kurangkan boilerplate trait
  → Struct update syntax (..) untuk kurangkan repetition
  → Type alias untuk kurangkan type yang panjang:
      type AppResult<T> = Result<T, AppError>;
      type Db = Arc<sqlx::MySqlPool>;
```

---

# 📋 Rujukan Pantas — Trade-off Cheat Sheet

```
MASALAH                    CARA RUST          SHORTCUT LEGITIMATE
───────────────────────────────────────────────────────────────────
String/Vec selepas move    Guna &str / &[T]   Clone kalau perlu
                           Redesign ownership

Lifetime dalam struct      Guna owned String  Arc<str> / Arc<T>
                           Tambah 'a

Banyak error types         Guna From/Into     anyhow::Error
                           Custom error enum

Pattern match verbose      Guna _ wildcard    if let + else
                           Guna @ binding

Generic bounds panjang     Guna where clause  dyn Trait
                           Buat trait alias   Box<dyn Trait>

Closure type mengelirukan  Mulakan dengan Fn  impl Fn(T) -> U
                           Naik ke FnMut/Once

Type annotation panjang    Guna _             type Alias = ...
                           (Vec<_>, HashMap<_,_>)
```

---

*Rust syntax yang "berat" adalah trade yang deliberate.*
*Tiada lunch percuma — semua bahasa ada trade-off.*
*Rust pilih: bayar sekarang dalam syntax, simpan kemudian dalam keselamatan.* 🦀
