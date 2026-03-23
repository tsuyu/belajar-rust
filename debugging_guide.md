# 🐛 Debugging dalam Rust — Panduan untuk Programmer Baru

> "Semua orang boleh tulis kod yang compiler faham.
>  Programmer baik tulis kod yang manusia faham."
> Panduan lengkap cara debug Rust — dari println! hingga debugger penuh.

---

## Mengapa Debugging Rust Berbeza?

```
Bahasa lain:
  Bug mungkin TERSEMBUNYI semasa runtime:
    - NullPointerException tiba-tiba
    - Buffer overflow yang senyap
    - Data race yang susah reproduce

Rust:
  Banyak bug ditangkap semasa COMPILE TIME:
    - Ownership errors → compiler error
    - Type mismatch → compiler error
    - Borrow violations → compiler error

  Tapi masih ada bug logic yang perlu debug:
    - Algorithm salah
    - Off-by-one errors
    - Wrong assumptions
    - Unexpected data
```

---

## Peta Panduan Debugging

```
Tahap 1 — Asas:          println!, dbg!, eprintln!
Tahap 2 — Compiler:      Baca error mesej dengan baik
Tahap 3 — Logging:       log, tracing crates
Tahap 4 — Debugger:      gdb, lldb, VS Code debugger
Tahap 5 — Test:          Unit test, assert!, panic!
Tahap 6 — Tools:         cargo check, clippy, miri
Tahap 7 — Strategi:      Bisection, rubber duck, minimal example
```

---

# TAHAP 1: println! & dbg! — Cara Paling Mudah 🖨️

## println! — Debug Asas

```rust
fn main() {
    let x = 42;
    let nama = "Ali";
    let data = vec![1, 2, 3, 4, 5];

    // Display — untuk output mesra pengguna
    println!("x = {}", x);
    println!("Nama: {}", nama);

    // Debug — untuk melihat struktur data
    println!("data = {:?}", data);

    // Pretty debug — lebih mudah baca untuk struct/enum besar
    println!("data =\n{:#?}", data);

    // Pelbagai nilai serentak
    println!("x={} nama={} data={:?}", x, nama, data);

    // Print ke stderr (tidak kacau output program)
    eprintln!("DEBUG: x = {}", x);
    eprintln!("WARN:  data kosong!");
}
```

---

## dbg! — Cara Paling Berguna untuk Debug

```rust
fn main() {
    let a = 5;
    let b = 10;

    // dbg! print nama variable, nilai, dan fail+baris
    // DAN return nilai — boleh guna dalam expression!
    let c = dbg!(a + b); // prints: [src/main.rs:7] a + b = 15
    println!("c = {}", c); // c = 15

    // Sangat berguna dalam expression chain!
    let hasil = dbg!(a * 2)     // [src/main.rs:11] a * 2 = 10
        + dbg!(b / 2);          // [src/main.rs:12] b / 2 = 5
    println!("hasil = {}", hasil); // 15

    // Debug dalam iterator chain
    let v = vec![1, 2, 3, 4, 5];
    let total: i32 = v.iter()
        .filter(|&&n| dbg!(n) % 2 == 0)  // print setiap n
        .map(|&n| dbg!(n * n))            // print setiap kuasa
        .sum();
    println!("Total: {}", total);

    // dbg! dengan struct
    #[derive(Debug)]
    struct Pekerja { nama: String, gaji: f64 }

    let p = Pekerja { nama: "Ali".into(), gaji: 4500.0 };
    dbg!(&p); // [src/main.rs:xx] &p = Pekerja { nama: "Ali", gaji: 4500.0 }

    // Remove semua dbg! sebelum production — atau guna #[cfg(debug_assertions)]
}
```

---

## Derive Debug — WAJIB untuk struct & enum

```rust
// Tambah #[derive(Debug)] untuk semua struct/enum anda!
// Tanpa ini, tidak boleh guna {:?} atau dbg!

// ❌ Tidak boleh debug
struct TidakBolehDebug {
    x: i32,
}

// ✔ Boleh debug
#[derive(Debug)]
struct Pekerja {
    id:       u32,
    nama:     String,
    gaji:     f64,
    aktif:    bool,
}

#[derive(Debug)]
enum Status {
    Aktif,
    Tidak { sebab: String },
}

#[derive(Debug)]
struct Jabatan {
    nama:     String,
    pekerja:  Vec<Pekerja>,  // nested — Vec<Pekerja> boleh debug
                             // kerana Pekerja ada Debug!
}

fn main() {
    let p = Pekerja { id: 1, nama: "Ali".into(), gaji: 4500.0, aktif: true };
    let s = Status::Tidak { sebab: "cuti".into() };

    println!("{:?}", p);   // satu baris
    println!("{:#?}", p);  // multi-line (lebih mudah baca)
    println!("{:#?}", s);

    dbg!(&p);
    dbg!(&s);
}
```

---

## 🧠 Tips #1: Trace Program Flow

```rust
fn kira_faktorial(n: u64) -> u64 {
    eprintln!("→ kira_faktorial({})", n); // trace function calls

    if n == 0 {
        eprintln!("  base case, return 1");
        return 1;
    }

    let hasil = n * kira_faktorial(n - 1);
    eprintln!("← faktorial({}) = {}", n, hasil);
    hasil
}

fn main() {
    let n = 5;
    let hasil = kira_faktorial(n);
    println!("{}! = {}", n, hasil);
}

// Output:
// → kira_faktorial(5)
// → kira_faktorial(4)
// → kira_faktorial(3)
// → kira_faktorial(2)
// → kira_faktorial(1)
// → kira_faktorial(0)
//   base case, return 1
// ← faktorial(1) = 1
// ← faktorial(2) = 2
// ← faktorial(3) = 6
// ← faktorial(4) = 24
// ← faktorial(5) = 120
// 5! = 120
```

---

# TAHAP 2: Baca Mesej Error Compiler 📖

## Anatomi Error Mesej Rust

```
error[E0382]: borrow of moved value: `s`
 --> src/main.rs:6:20
  |
3 |     let s = String::from("hello");
  |         - move occurs because `s` has type `String`,
  |           which does not implement the `Copy` trait
4 |     let s2 = s;
  |              - value moved here
5 |
6 |     println!("{}", s);
  |                    ^ value borrowed here after move

Cara baca:
  1. error[E0382]     ← KOD ERROR — guna untuk cari dokumentasi
  2. borrow of moved  ← RINGKASAN masalah
  3. --> src/main.rs:6:20  ← LOKASI masalah (fail:baris:lajur)
  4. | 4 | let s2 = s;    ← KOD yang bermasalah
  5. |              -      ← PUNCA masalah (s2 = s → moved here)
  6. |                    ^ ← LOKASI error (s digunakan selepas move)

CARA GUNA:
  rustc --explain E0382   ← penjelasan panjang dengan contoh!
  cargo --explain E0382   ← sama
```

---

## Cara Guna rustc --explain

```bash
# Dapat penjelasan penuh untuk setiap error code
rustc --explain E0382

# Output: penjelasan panjang + contoh kod + cara betulkan
# Ini adalah cara TERBAIK untuk faham error baru!

# Error codes yang biasa ditemui:
# E0382 — borrow of moved value
# E0505 — cannot move out of ... because it is borrowed
# E0499 — cannot borrow as mutable more than once
# E0502 — cannot borrow as immutable because borrowed as mutable
# E0308 — mismatched types
# E0412 — cannot find type
# E0425 — cannot find function/value
# E0004 — non-exhaustive patterns (dalam match)
# E0597 — does not live long enough (lifetime)
```

---

## Ikut Cadangan Compiler

```rust
// Rust compiler selalu bagi cadangan "help" atau "consider":

// ERROR:
// fn main() {
//     let s = String::from("hello");
//     let s2 = s;
//     println!("{}", s);  // error!
// }

// CADANGAN COMPILER:
// help: consider cloning the value if the performance cost is acceptable
//   4  |     let s2 = s.clone();

// ← IKUT CADANGAN INI!
fn main() {
    let s = String::from("hello");
    let s2 = s.clone(); // ikut cadangan compiler
    println!("{} {}", s, s2); // ✔
}

// CONTOH LAIN — compiler cadang betulkan typo:
// let hasil = vecc![1, 2, 3];  // typo!
// error: cannot find macro `vecc` in this scope
// help: a macro with a similar name exists: `vec`
// ← Compiler TAHU anda maksudkan vec!
```

---

## Perhatikan WARNING — Sering Tanda Bug

```rust
#[allow(unused)] // matikan warning untuk belajar
fn main() {
    // WARNING: unused variable
    let x = 5;
    // warning: unused variable `x`
    // = note: if this is intentional, prefix with `_`
    // → Mungkin anda lupa guna x!

    // WARNING: unused Result (mungkin error tidak ditangani!)
    std::fs::write("fail.txt", "data");
    // warning: unused `Result` that must be used
    // = note: this `Result` may be an `Err` variant
    // → BERBAHAYA! Error diabaikan secara senyap!

    // Cara betul:
    if let Err(e) = std::fs::write("fail.txt", "data") {
        eprintln!("Gagal: {}", e);
    }

    // WARNING: dead code
    // fn tidak_digunakan() {}
    // warning: function is never used: `tidak_digunakan`
    // → Code yang tidak berguna — mungkin logic salah?
}
```

---

# TAHAP 3: Logging — Untuk Kod Yang Lebih Serius 📋

## log + env_logger crates

```toml
# Cargo.toml
[dependencies]
log        = "0.4"
env_logger = "0.11"
```

```rust
use log::{debug, info, warn, error, trace};

fn proses_data(data: &[i32]) -> i32 {
    debug!("Mula proses {} item", data.len()); // level: debug

    if data.is_empty() {
        warn!("Data kosong!");
        return 0;
    }

    let hasil: i32 = data.iter().sum();
    info!("Proses selesai, hasil = {}", hasil); // level: info

    if hasil < 0 {
        error!("Hasil negatif tidak dijangka: {}", hasil);
    }

    trace!("Nilai-nilai: {:?}", data); // level: trace (paling verbose)
    hasil
}

fn main() {
    // Setup logger — baca RUST_LOG env variable
    env_logger::init();

    let data = vec![1, 2, 3, 4, 5];
    let hasil = proses_data(&data);
    println!("Hasil: {}", hasil);
}

// Cara run dengan log level berbeza:
// RUST_LOG=debug cargo run     → tunjuk semua termasuk debug
// RUST_LOG=info cargo run      → tunjuk info, warn, error sahaja
// RUST_LOG=warn cargo run      → tunjuk warn dan error sahaja
// RUST_LOG=error cargo run     → tunjuk error sahaja
// RUST_LOG=trace cargo run     → tunjuk SEMUA (paling verbose)
// RUST_LOG=modul=debug cargo run → debug untuk modul tertentu sahaja
```

---

## tracing — Lebih Berkuasa (untuk async)

```toml
[dependencies]
tracing            = "0.1"
tracing-subscriber = "0.3"
```

```rust
use tracing::{debug, info, warn, error, instrument, span, Level};

// #[instrument] — auto-log function entry/exit dengan arguments!
#[instrument]
fn kira(a: i32, b: i32) -> i32 {
    info!("Mengira jumlah");
    let hasil = a + b;
    debug!("Hasil: {}", hasil);
    hasil
}

#[instrument(skip(kata_laluan))] // skip sensitive data dari log!
fn log_masuk(nama: &str, kata_laluan: &str) -> bool {
    info!("Cuba log masuk");
    // kata_laluan tidak akan appear dalam log!
    true
}

fn main() {
    // Setup tracing subscriber
    tracing_subscriber::fmt()
        .with_max_level(Level::DEBUG)
        .init();

    let hasil = kira(5, 3);
    println!("Hasil: {}", hasil);

    log_masuk("Ali", "rahsia123");
}

// Output:
// DEBUG modul::kira{a=5 b=3}: Mengira jumlah
// DEBUG modul::kira{a=5 b=3}: Hasil: 8
// INFO  modul::log_masuk{nama="Ali"}: Cuba log masuk
//                          ↑ kata_laluan di-skip!
```

---

# TAHAP 4: Debugger — Untuk Bug Yang Susah 🔬

## Setup VS Code Debugger

```bash
# Install extension: "rust-analyzer" dan "CodeLLDB"
# atau "C/C++" extension (untuk gdb)

# Buat .vscode/launch.json:
```

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "lldb",
            "request": "launch",
            "name": "Debug Rust Program",
            "program": "${workspaceFolder}/target/debug/${workspaceFolderBasename}",
            "args": [],
            "cwd": "${workspaceFolder}",
            "preLaunchTask": "cargo build"
        }
    ]
}
```

```json
// .vscode/tasks.json
{
    "version": "2.0.0",
    "tasks": [
        {
            "type": "cargo",
            "command": "build",
            "label": "cargo build"
        }
    ]
}
```

```
Cara guna VS Code debugger:
  1. Letak breakpoint (klik kiri di sebelah nombor baris)
  2. Tekan F5 untuk start debug
  3. Program akan pause di breakpoint
  4. F10 = step over (lompat baris ke baris)
  5. F11 = step into (masuk ke dalam fungsi)
  6. Shift+F11 = step out (keluar dari fungsi)
  7. F5 = continue (terus sampai breakpoint seterusnya)
  8. Tengok panel Variables untuk lihat nilai semasa
```

---

## GDB / LLDB dari Terminal

```bash
# Build dengan debug info (default dengan cargo build)
cargo build

# GDB (Linux/Windows)
gdb target/debug/nama-program

# LLDB (macOS/Linux)
lldb target/debug/nama-program

# Perintah asas GDB/LLDB:
(gdb) run                # jalankan program
(gdb) break main         # breakpoint di fungsi main
(gdb) break src/main.rs:15  # breakpoint di baris 15
(gdb) continue (c)       # terus dari breakpoint
(gdb) next (n)           # lompat baris ke baris
(gdb) step (s)           # masuk ke dalam fungsi
(gdb) print x            # print nilai variable x
(gdb) info locals        # tunjuk semua local variables
(gdb) backtrace (bt)     # tunjuk call stack
(gdb) quit (q)           # keluar debugger

# LLDB commands (sedikit berbeza):
(lldb) run
(lldb) breakpoint set --name main
(lldb) breakpoint set --file main.rs --line 15
(lldb) continue
(lldb) next
(lldb) step
(lldb) frame variable   # tunjuk local variables
(lldb) bt               # backtrace
```

---

## rust-gdb — Wrapper dengan Pretty Printing

```bash
# rust-gdb = gdb + pretty printers untuk Rust types
# Menunjukkan Vec, String, Option dengan format yang lebih baik

rust-gdb target/debug/nama-program

# rust-lldb juga ada:
rust-lldb target/debug/nama-program

# Contoh output dengan rust-gdb:
# (gdb) print v
# $1 = Vec<i32>(len: 3, cap: 4) = {1, 2, 3}
# 
# Berbanding gdb biasa:
# $1 = alloc::vec::Vec<i32> {buf: ..., len: 3}
#   (susah baca!)
```

---

# TAHAP 5: Tests sebagai Alat Debug 🧪

## assert! dan assert_eq! — Debug Aktif

```rust
fn bahagi(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 { None } else { Some(a / b) }
}

fn kira_purata(data: &[f64]) -> Option<f64> {
    if data.is_empty() { return None; }
    let jumlah: f64 = data.iter().sum();
    Some(jumlah / data.len() as f64)
}

// ── Unit Tests ────────────────────────────────────────────────
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_bahagi_normal() {
        assert_eq!(bahagi(10.0, 2.0), Some(5.0));
        assert_eq!(bahagi(0.0, 5.0), Some(0.0));
    }

    #[test]
    fn test_bahagi_sifar() {
        assert_eq!(bahagi(5.0, 0.0), None);
    }

    #[test]
    fn test_purata_normal() {
        let data = vec![1.0, 2.0, 3.0, 4.0, 5.0];
        let purata = kira_purata(&data).unwrap();
        // assert dengan tolerance untuk float!
        assert!((purata - 3.0).abs() < 1e-10,
                "Purata salah: {} != 3.0", purata);
    }

    #[test]
    fn test_purata_kosong() {
        assert_eq!(kira_purata(&[]), None);
    }

    // Test dengan mesej yang informatif
    #[test]
    fn test_dengan_mesej() {
        let hasil = bahagi(10.0, 3.0).unwrap();
        assert!(
            (hasil - 3.3333).abs() < 0.001,
            "Nilai tidak tepat: dapat {:.4}, jangka ~3.3333",
            hasil
        );
    }
}

fn main() {
    // assert! dalam kod biasa — debug assertion
    let data = vec![1.0, 2.0, 3.0];
    debug_assert!(!data.is_empty(), "data mestilah tidak kosong");
    // debug_assert! hanya aktif dalam debug mode, tiada overhead release!
}
```

---

## Tulis Test DULU sebelum Debug

```rust
// Cara terbaik debug: tulis test untuk reproduce bug,
// kemudian fix bug, kemudian test pass!

// Contoh: Laporan bug "kira_gaji() salah untuk pekerja kontrak"

fn kira_gaji(asas: f64, jenis: &str, jam_lebih: f64) -> f64 {
    match jenis {
        "tetap"   => asas + jam_lebih * 15.0,
        "kontrak" => asas * 0.85 + jam_lebih * 12.0, // ← bug di sini?
        _         => asas,
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    // Tulis test yang GAGAL dulu (reproduce bug)
    #[test]
    fn test_gaji_kontrak() {
        // Kes: asas=3000, jam_lebih=5, jenis=kontrak
        // Jangka: 3000 * 0.85 + 5 * 12.0 = 2550 + 60 = 2610
        let gaji = kira_gaji(3000.0, "kontrak", 5.0);
        assert_eq!(gaji, 2610.0,
            "Gaji kontrak salah: dapat {}, jangka 2610.0", gaji);
    }

    // Test edge cases
    #[test]
    fn test_gaji_tanpa_lebih() {
        assert_eq!(kira_gaji(5000.0, "tetap", 0.0), 5000.0);
        assert_eq!(kira_gaji(3000.0, "kontrak", 0.0), 2550.0);
    }

    #[test]
    fn test_jenis_tidak_dikenali() {
        assert_eq!(kira_gaji(4000.0, "sambilan", 0.0), 4000.0);
    }
}

// Jalankan: cargo test
// Kalau test gagal, output tunjukkan dengan jelas apa salah!
```

---

# TAHAP 6: Cargo Tools — Debug dengan Tools 🔧

## cargo check — Semak Pantas

```bash
# cargo check = semak error tanpa build binary
# 3-10x lebih laju dari cargo build!
# Guna ini untuk semak error semasa menulis kod

cargo check

# Semak semua targets (tests, examples, benchmarks)
cargo check --all-targets
```

---

## cargo clippy — Cari Bug dengan Linter

```bash
# Install clippy (biasanya ada dengan rustup)
rustup component add clippy

# Jalankan clippy
cargo clippy

# Clippy lebih ketat dari compiler biasa!
# Ia akan tangkap:
# - code smell
# - potential bugs
# - non-idiomatic code
# - performance issues

# Contoh warning clippy:
# warning: this boolean expression can be simplified
#   --> src/main.rs:5:8
#    | 5 |     if x == true {
#    |       ^^^^^^^^^^^^ help: try: `if x {`
#
# warning: useless use of `vec!`
#   | 5 |     let v = vec![1, 2, 3].len();
#    |                  ^^^^^^^^^ help: try: `[1, 2, 3].len()`

# Guna clippy dengan semua warnings sebagai errors (untuk CI):
cargo clippy -- -D warnings

# Fix automatically
cargo clippy --fix
```

---

## cargo test -- --nocapture

```bash
# Secara default, output dalam test disembunyikan!
# Guna --nocapture untuk lihat println! dalam test:

cargo test -- --nocapture

# Jalankan test tertentu sahaja:
cargo test nama_test

# Jalankan semua test dalam modul:
cargo test modul::

# Jalankan test yang ada perkataan tertentu dalam nama:
cargo test tambah

# List semua test tanpa jalankan:
cargo test -- --list

# Jalankan test yang biasanya di-ignore:
cargo test -- --ignored
```

---

## cargo run dengan RUST_BACKTRACE

```bash
# Bila program panic, tunjukkan full stack trace:
RUST_BACKTRACE=1 cargo run

# Tunjukkan LEBIH banyak detail (termasuk library code):
RUST_BACKTRACE=full cargo run

# Contoh output dengan RUST_BACKTRACE=1:
# thread 'main' panicked at 'index out of bounds: the len is 3
# but the index is 5', src/main.rs:7:5
# stack backtrace:
#    0: rust_begin_unwind
#    1: core::panicking::panic_fmt
#    2: core::slice::index::slice_index_usize_fail
#    3: my_program::main
#          at ./src/main.rs:7:5
#    ...
# ↑ Ini tunjukkan EXACTLY di mana panic berlaku!
```

---

## Miri — Detect Undefined Behavior

```bash
# Miri = Rust interpreter yang detect UB dan memory errors
# Sangat berguna untuk code yang ada unsafe

# Install miri:
rustup +nightly component add miri

# Jalankan tests dengan miri:
cargo +nightly miri test

# Jalankan program dengan miri:
cargo +nightly miri run

# Miri akan detect:
# - Use after free
# - Buffer overflows
# - Invalid pointer arithmetic
# - Data races (dengan -Zmiri-track-raw-pointers)
# - Uninitialised memory reads
```

---

# TAHAP 7: Strategi Debug yang Berkesan 🧠

## Strategi 1: Bisection — Halang Masalah

```
MASALAH: Program bagi output salah untuk data besar.
         Susah nak tahu kat mana salahnya.

CARA BISECTION:
  1. Bahagi input kepada SEPARUH
  2. Test bahagian pertama — betul?
  3. Kalau betul → bug di bahagian kedua
  4. Kalau salah → bug di bahagian pertama
  5. Ulang dengan bahagian yang bermasalah
  6. Terus sampai jumpa baris yang berpunca!

Macam binary search untuk bug!
  Input 1000 item → test 500 → test 250 → test 125 → ...
  Max 10 iterasi untuk input 1000 item!
```

```rust
fn proses_data(data: &[i32]) -> Vec<i32> {
    // Bug tersembunyi di sini...
    data.iter()
        .map(|&n| {
            if n > 100 { n / 0 } // BUG! divide by zero bila n > 100
            else { n * 2 }
        })
        .collect()
}

fn debug_bisection() {
    let data: Vec<i32> = (1..=200).collect();

    // Test separuh pertama (1..=100)
    let bahagian1 = &data[..100];
    let _r1 = proses_data(bahagian1); // Ini tidak panic
    eprintln!("Bahagian 1 (1-100): OK");

    // Test separuh kedua (101..=200)
    let bahagian2 = &data[100..];
    let _r2 = proses_data(bahagian2); // Ini PANIC!
    eprintln!("Bahagian 2 (101-200): OK"); // tidak sampai sini

    // Sudah tahu bug di bahagian 2!
    // Ulang dengan 101..=150, kemudian 101..=125, dst.
}
```

---

## Strategi 2: Minimal Reproducing Example

```
MASALAH: Bug dalam program yang besar.
         Susah nak report atau debug.

CARA:
  1. Buat FAIL BARU (main.rs kecil)
  2. Salin KOD MINIMUM yang reproduce bug
  3. Buang semua yang tidak berkaitan
  4. Terus buang sehingga tidak boleh buang lagi
  5. Program kecil yang reproduce bug = minimal example

KENAPA BERGUNA:
  - Lebih mudah untuk fikir tentang masalah
  - Boleh share di Stack Overflow / forum Rust
  - Sering kali PROSES membuat minimal example
    sudah selesaikan masalah! (anda faham punca)
```

```rust
// Contoh: Bug dalam sistem besar...
// Buat minimal example:

fn main() {
    // Bug: Vec tidak dikemaskini seperti dijangka
    let mut v = vec![1, 2, 3];

    // Minimal code yang reproduce bug:
    for i in 0..v.len() {
        v.push(i as i32 * 10); // PROBLEM: mengubah vec semasa iterate!
        // len() berubah setiap iterasi → infinite-ish loop!
    }

    println!("{:?}", v);
}

// Sekarang boleh fokus pada masalah ini sahaja.
// Penyelesaian: collect indices dahulu, atau guna iter() dengan map
```

---

## Strategi 3: Rubber Duck Debugging

```
CARA:
  1. Ambil "rubber duck" (atau rakan, atau diri sendiri)
  2. Terangkan kod anda BARIS DEMI BARIS kepada rubber duck
  3. Terangkan APA yang sepatutnya berlaku
  4. Terangkan APA yang sebenarnya berlaku
  5. Terangkan KENAPA anda fikir ia berlaku

KENAPA BERKESAN:
  Bila anda terangkan dengan perkataan, otak anda
  perlu "format ulang" pemikiran.
  Sering kali anda SENDIRI jumpa bug semasa terangkan!

  "Oh... saya kata ia loop 5 kali tapi sebenarnya..."
```

---

## Strategi 4: Semak Assumptions

```rust
fn main() {
    let data = vec![1, 2, 3, 4, 5];

    // ❌ Assume data tidak kosong
    let purata = data.iter().sum::<i32>() / data.len() as i32;

    // ✔ Semak assumption secara explicit
    assert!(!data.is_empty(), "Data mesti tidak kosong untuk kira purata");
    let purata = data.iter().sum::<i32>() / data.len() as i32;

    // ❌ Assume nilai dalam julat
    let indeks = kira_indeks(&data);
    let nilai = data[indeks]; // mungkin panic!

    // ✔ Semak assumption
    let indeks = kira_indeks(&data);
    assert!(indeks < data.len(),
            "Indeks {} luar julat untuk data panjang {}", indeks, data.len());
    let nilai = data[indeks];

    // Guna debug_assert! untuk check yang mahal (hanya dalam debug mode):
    debug_assert!(data.iter().all(|&n| n > 0),
                  "Semua nilai mesti positif");
}

fn kira_indeks(v: &[i32]) -> usize { 0 } // placeholder
```

---

## Strategi 5: Divide and Conquer dengan println!

```rust
fn proses_kompleks(data: &[i32]) -> i32 {
    // Step 1: Filter
    let difilter: Vec<i32> = data.iter()
        .filter(|&&n| n > 0)
        .copied()
        .collect();
    eprintln!("[DEBUG step1] difilter: {:?}", difilter);

    // Step 2: Transform
    let ditransform: Vec<i32> = difilter.iter()
        .map(|&n| n * n)
        .collect();
    eprintln!("[DEBUG step2] ditransform: {:?}", ditransform);

    // Step 3: Aggregate
    let hasil: i32 = ditransform.iter().sum();
    eprintln!("[DEBUG step3] hasil: {}", hasil);

    hasil
}

fn main() {
    let input = vec![-3, -1, 0, 2, 4, -2, 6];
    eprintln!("[DEBUG input] {:?}", input);

    let output = proses_kompleks(&input);
    println!("Output: {}", output);

    // Bila dah selesai debug, buang semua eprintln! DEBUG
    // atau gunakan log crate untuk kontrol dengan RUST_LOG
}
```

---

# BAHAGIAN BONUS: Quick Debug Toolkit 🛠️

## Template Debug yang Berguna

```rust
// Simpan ini sebagai snippet untuk guna bila debug!

// ── Trace masuk/keluar fungsi ─────────────────────────────────
macro_rules! trace_fn {
    ($name:expr) => {
        eprintln!("[TRACE] → {}", $name);
    };
    ($name:expr, $($arg:expr),*) => {
        eprintln!("[TRACE] → {}({})", $name, format!("{:?}", ($($arg,)*)));
    };
}

// ── Debug dengan label ────────────────────────────────────────
macro_rules! debug_val {
    ($val:expr) => {
        eprintln!("[DEBUG] {}:{} {} = {:?}",
            file!(), line!(), stringify!($val), $val)
    };
}

// ── Ukur masa ─────────────────────────────────────────────────
macro_rules! ukur_masa {
    ($label:expr, $code:expr) => {{
        let mula = std::time::Instant::now();
        let hasil = $code;
        eprintln!("[MASA] {} = {:?}", $label, mula.elapsed());
        hasil
    }};
}

fn main() {
    let x = 42;
    debug_val!(x);              // [DEBUG] src/main.rs:55 x = 42

    trace_fn!("proses", x);     // [TRACE] → proses((42,))

    let hasil = ukur_masa!("kira", {
        let mut s = 0i64;
        for i in 0..1_000_000 { s += i; }
        s
    });
    println!("Hasil: {}", hasil);
    // [MASA] kira = 1.234ms
}
```

---

## Cargo Watch — Auto-check bila Simpan

```bash
# Install cargo-watch
cargo install cargo-watch

# Auto cargo check bila fail berubah (simpan)
cargo watch -x check

# Auto run tests bila berubah
cargo watch -x test

# Auto check, kemudian test bila berubah
cargo watch -x check -x test

# Auto run bila berubah
cargo watch -x run

# Ini SANGAT berguna semasa belajar — feedback segera!
```

---

## Ringkasan: Cara Pilih Tool Debug

```
SITUASI                          TOOL
─────────────────────────────────────────────────────────────
"Nilai variable apa sekarang?"   dbg!() atau println!()
"Kat mana program sampai?"       eprintln!() dengan label
"Error compiler tak faham"       rustc --explain E0xxx
"Banyak warning — nak fix"       cargo clippy
"Test gagal, nak lihat output"   cargo test -- --nocapture
"Program panic, nak trace"       RUST_BACKTRACE=1 cargo run
"Bug logic yang susah"           unit tests + bisection
"Perlu lihat line per line"      VS Code debugger / rust-gdb
"Ada unsafe code, takut UB"      cargo miri test
"Program lambat / memory leak"   cargo flamegraph / heaptrack
"Log berstruktur untuk server"   tracing crate

UNTUK PROGRAMMER BARU:
  1. Mulakan dengan: println! / dbg!
  2. Belajar baca error message
  3. Guna: cargo clippy selalu
  4. Tulis unit tests
  5. Setup VS Code debugger
  6. Baru belajar gdb/lldb bila perlu
```

---

## Perintah Debugging Harian

```bash
# Check error dengan cepat
cargo check

# Lihat semua warnings (termasuk yang biasanya hidden)
cargo check --all-targets 2>&1 | head -100

# Jalankan dengan backtrace penuh
RUST_BACKTRACE=1 cargo run

# Jalankan tests dengan output
cargo test -- --nocapture

# Jalankan test tertentu
cargo test nama_fungsi

# Lint dan cadangan
cargo clippy

# Format kod (bersihkan sebelum baca)
cargo fmt

# Expand macros (lihat apa macro hasilkan)
cargo install cargo-expand
cargo expand

# Auto-check bila simpan
cargo watch -x check
```

---

## Sumber Bantuan

```
Bila stuck, tanya di:
  📖 The Rust Book         → https://doc.rust-lang.org/book/
  📖 Rust by Example       → https://doc.rust-lang.org/rust-by-example/
  💬 Rust Forum            → https://users.rust-lang.org/
  💬 Rust Discord          → https://discord.gg/rust-lang
  💬 Reddit r/rust         → https://reddit.com/r/rust
  🔍 Rust Playground       → https://play.rust-lang.org/
     (paste kod, share link, minta tolong!)

Cara tanya soalan yang baik:
  1. Sertakan FULL ERROR MESSAGE (copy-paste)
  2. Sertakan KOD MINIMUM yang reproduce masalah
  3. Terangkan APA yang anda jangka berlaku
  4. Terangkan APA yang sebenarnya berlaku
  5. Nyatakan apa yang sudah anda cuba
```

---

*"Debugging adalah seni mencari salah faham antara apa yang anda tulis*
*dan apa yang komputer baca. Dalam Rust, compiler adalah rakan debugging*
*terbaik anda — dengar apa yang ia cakap."* 🦀
