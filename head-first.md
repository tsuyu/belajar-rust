# 🦀 Head First Rust (Lite Edition)
> *Belajar Rust cara fun — visual, hands-on, dan tak bosan!*

---

## 📖 Cara Guna Buku Ini

Buku ni bukan untuk dibaca sambil tido. Ia untuk:
- **Cuba kod** — setiap contoh, taip sendiri. Jangan copy-paste!
- **Jawab soalan** — kalau ada 🧠 Brain Teaser, cuba dulu sebelum tengok jawapan.
- **Buat projek** — setiap bab ada mini projek.

> 💬 *"The best way to learn Rust is to write Rust that fails to compile — then fix it."*

---

## 🗺️ Peta Perjalanan

```
Bab 1 → Variables & Types
Bab 2 → Functions & Control Flow
Bab 3 → Ownership (Boss Level!)
Bab 4 → Structs & Enums
Bab 5 → Error Handling
Bab 6 → Collections
Bab 7 → Traits
Bab 8 → Mini Project: CLI Todo App
```

---

# BAB 1: Hello, Rust! 🦀

## Otak kau nak tahu — Kenapa Rust?

| PHP | Rust |
|-----|------|
| `$name = "Ali";` | `let name = "Ali";` |
| Slow (interpreted) | Fast (compiled) |
| Runtime errors | Compile-time errors |
| Garbage collector | Ownership system |
| Web-centric | Anywhere — web, CLI, embedded |

> 🔥 **Fun Fact:** Rust telah jadi bahasa paling "loved" di Stack Overflow survey 9 tahun berturut-turut!

---

## Setup Dalam 3 Minit

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Semak installation
rustc --version
cargo --version

# Buat projek pertama
cargo new hello_rust
cd hello_rust
cargo run
```

---

## Program Pertama Kau

```rust
// src/main.rs
fn main() {
    println!("Hello, Rust! 🦀");
    println!("Nama saya: {}", "Ali");
    println!("Umur saya: {}", 25);
}
```

**Output:**
```
Hello, Rust! 🦀
Nama saya: Ali
Umur saya: 25
```

> 📝 **Perasan tak?** `println!` ada tanda `!` — maknanya ia adalah **macro**, bukan function biasa.

---

## 🧠 Brain Teaser #1

Cuba predict output program ni sebelum run:

```rust
fn main() {
    let x = 5;
    let y = x + 3;
    println!("Jawapan: {}", y * 2);
}
```

<details>
<summary>👀 Tengok Jawapan</summary>

**Output:** `Jawapan: 16`

`y = 5 + 3 = 8`, then `8 * 2 = 16`
</details>

---

# BAB 2: Variables & Types 📦

## Variables — Kotak Untuk Simpan Data

```rust
fn main() {
    // By default, variables IMMUTABLE (tak boleh tukar!)
    let umur = 25;
    // umur = 26;  ← ERROR! Compiler akan marah 😠

    // Nak boleh tukar? Guna `mut`
    let mut markah = 80;
    markah = 95;  // OK!
    println!("Markah baru: {}", markah);

    // Constants — immutable SELAMANYA
    const KADAR_GST: f64 = 0.06;
    println!("GST: {}%", KADAR_GST * 100.0);
}
```

---

## Jenis-Jenis Data (Types)

```
┌─────────────────────────────────────────┐
│              RUST DATA TYPES            │
├──────────────┬──────────────────────────┤
│  Integer     │  i8, i16, i32, i64, u32  │
│  Float       │  f32, f64                │
│  Boolean     │  bool (true/false)       │
│  Character   │  char ('A', '🦀')        │
│  String      │  String, &str            │
│  Tuple       │  (i32, f64, bool)        │
│  Array       │  [i32; 5]                │
└──────────────┴──────────────────────────┘
```

```rust
fn main() {
    let integer: i32 = 42;
    let float: f64 = 3.14;
    let aktif: bool = true;
    let huruf: char = '🦀';
    let nama: &str = "Tsuyu";                    // string slice (borrowed)
    let nama_owned: String = String::from("Ali"); // owned String

    let tuple = (1, 2.5, true);
    println!("Tuple item pertama: {}", tuple.0);

    let array = [10, 20, 30, 40, 50];
    println!("Array item kedua: {}", array[1]);
}
```

---

## Shadowing — Sihir Rust!

```rust
fn main() {
    let nilai = 5;
    let nilai = nilai + 1;   // shadow! nilai baru = 6
    let nilai = nilai * 2;   // shadow lagi! nilai baru = 12
    println!("Nilai: {}", nilai); // Output: 12

    // Boleh tukar TYPE dengan shadowing
    let spaces = "   ";       // &str
    let spaces = spaces.len(); // i32 — jenis berbeza!
    println!("Spaces: {}", spaces);
}
```

> 💡 **Tip:** Shadowing berbeza dengan `mut`. Shadowing boleh tukar type, `mut` tidak boleh.

---

## 🧠 Brain Teaser #2

Kod ni akan compile atau error?

```rust
fn main() {
    let x = 5;
    x = 10;
    println!("{}", x);
}
```

<details>
<summary>👀 Jawapan</summary>

**ERROR!** `x` tidak `mut`. Fix: `let mut x = 5;`
</details>

---

# BAB 3: Functions & Control Flow 🔀

## Functions

```rust
// Function asas
fn sapa(nama: &str) -> String {
    format!("Hello, {}! Selamat datang.", nama)
    // Tiada semicolon = return value (implicitly returned)
}

// Multiple parameters
fn tambah(a: i32, b: i32) -> i32 {
    a + b  // expression tanpa semicolon = return
}

fn main() {
    let mesej = sapa("Ali");
    println!("{}", mesej);

    let jumlah = tambah(10, 20);
    println!("10 + 20 = {}", jumlah);
}
```

---

## Control Flow

### if / else

```rust
fn gred(markah: u32) -> &'static str {
    if markah >= 90 {
        "A"
    } else if markah >= 75 {
        "B"
    } else if markah >= 60 {
        "C"
    } else {
        "Gagal"
    }
}

fn main() {
    println!("Gred: {}", gred(85)); // Output: B
}
```

### loop, while, for

```rust
fn main() {
    // for loop — paling common
    for i in 1..=5 {
        println!("Bilangan: {}", i);
    }

    // while loop
    let mut kiraan = 0;
    while kiraan < 3 {
        println!("Kiraan: {}", kiraan);
        kiraan += 1;
    }

    // loop — infinite (guna break untuk keluar)
    let mut n = 0;
    let hasil = loop {
        n += 1;
        if n == 10 {
            break n * 2;  // return nilai dari loop!
        }
    };
    println!("Hasil: {}", hasil); // 20
}
```

---

# BAB 4: OWNERSHIP 🏠 (Boss Level!)

## ⚠️ Konsep Paling Penting Dalam Rust

> Ownership adalah sebab Rust selamat dan laju — tiada garbage collector, tiada memory leak.

### 3 Peraturan Ownership:
```
┌─────────────────────────────────────────────────┐
│  1. Setiap nilai ada SATU owner                 │
│  2. Hanya SATU owner pada satu-satu masa        │
│  3. Bila owner keluar scope → nilai DROP (free) │
└─────────────────────────────────────────────────┘
```

```rust
fn main() {
    let s1 = String::from("hello"); // s1 owns "hello"
    let s2 = s1;                    // ownership MOVED ke s2!

    // println!("{}", s1);          // ← ERROR! s1 sudah invalid
    println!("{}", s2);             // OK
}
```

---

## Clone vs Move

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();   // COPY data — kedua-dua valid

    println!("s1 = {}, s2 = {}", s1, s2); // OK!
}
```

---

## Borrowing & References — Pinjam Sahaja!

```rust
fn kira_panjang(s: &String) -> usize {  // & = borrow, bukan take ownership
    s.len()
}

fn main() {
    let s1 = String::from("hello");
    let panjang = kira_panjang(&s1);  // bagi reference, bukan ownership
    println!("{} panjangnya {}", s1, panjang); // s1 masih valid!
}
```

### Mutable References

```rust
fn tambah_exclaim(s: &mut String) {
    s.push_str("!!!");
}

fn main() {
    let mut s = String::from("Hello");
    tambah_exclaim(&mut s);
    println!("{}", s); // Hello!!!
}
```

> ⚠️ **Peraturan:** Hanya SATU mutable reference pada satu-satu masa. Ini elak data race!

---

## 🧠 Brain Teaser #3

Kenapa kod ni error?

```rust
fn main() {
    let mut s = String::from("hello");
    let r1 = &mut s;
    let r2 = &mut s;   // ← masalah di sini
    println!("{}, {}", r1, r2);
}
```

<details>
<summary>👀 Jawapan</summary>

**ERROR:** Tidak boleh ada dua mutable references serentak ke `s`. Ini mencegah data race. Fix: guna scope terpisah atau sequential references.
</details>

---

# BAB 5: Structs & Enums 🏗️

## Structs — Blueprint Untuk Data

```rust
// Definisi struct
struct Pekerja {
    nama: String,
    no_pekerja: u32,
    bahagian: String,
    aktif: bool,
}

impl Pekerja {
    // Associated function (macam static method)
    fn baru(nama: &str, no: u32, bahagian: &str) -> Self {
        Pekerja {
            nama: nama.to_string(),
            no_pekerja: no,
            bahagian: bahagian.to_string(),
            aktif: true,
        }
    }

    // Method
    fn perkenalan(&self) -> String {
        format!("Saya {}, No. {}, Bahagian {}",
            self.nama, self.no_pekerja, self.bahagian)
    }
}

fn main() {
    let p = Pekerja::baru("Ali", 1001, "ICT");
    println!("{}", p.perkenalan());
}
```

---

## Enums — Pilihan Yang Terhad

```rust
enum Status {
    Aktif,
    Cuti,
    Berhenti,
}

enum Keputusan {
    Lulus(u32),          // bawa nilai
    Gagal(String),       // bawa mesej
    Pending,
}

fn main() {
    let status = Status::Aktif;

    let keputusan = Keputusan::Lulus(85);

    match keputusan {
        Keputusan::Lulus(markah) => println!("Lulus dengan {}", markah),
        Keputusan::Gagal(sebab)  => println!("Gagal: {}", sebab),
        Keputusan::Pending       => println!("Belum ada keputusan"),
    }
}
```

---

## Option<T> — Nilai Yang Mungkin Tiada

```rust
// Tiada null dalam Rust! Guna Option<T>
fn cari_pekerja(id: u32) -> Option<String> {
    if id == 1001 {
        Some(String::from("Ali"))
    } else {
        None  // tiada null, ada None
    }
}

fn main() {
    match cari_pekerja(1001) {
        Some(nama) => println!("Jumpa: {}", nama),
        None       => println!("Pekerja tidak dijumpai"),
    }

    // Cara ringkas dengan if let
    if let Some(nama) = cari_pekerja(9999) {
        println!("Nama: {}", nama);
    } else {
        println!("Tiada pekerja.");
    }
}
```

---

# BAB 6: Error Handling ⚠️

## Result<T, E> — Cara Rust Handle Error

```rust
use std::fs;

fn baca_fail(path: &str) -> Result<String, std::io::Error> {
    fs::read_to_string(path)
}

fn main() {
    match baca_fail("data.txt") {
        Ok(kandungan) => println!("Fail: {}", kandungan),
        Err(e)        => println!("Error: {}", e),
    }
}
```

## Operator `?` — Shortcut Error Propagation

```rust
use std::fs;

fn proses_fail(path: &str) -> Result<String, std::io::Error> {
    let kandungan = fs::read_to_string(path)?;  // ? = return Err kalau gagal
    Ok(kandungan.to_uppercase())
}

fn main() {
    match proses_fail("data.txt") {
        Ok(data) => println!("{}", data),
        Err(e)   => println!("Gagal: {}", e),
    }
}
```

> 💡 **`?`** adalah cara idiomatic Rust untuk propagate error — sama macam PHP `throw` tapi lebih explicit dan selamat.

---

## Custom Error Dengan `thiserror`

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum AppError {
    #[error("Pekerja tidak dijumpai: {0}")]
    TidakJumpai(String),

    #[error("Akses ditolak")]
    AksesDitolak,

    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
}
```

---

# BAB 7: Collections 📚

## Vec<T> — Dynamic Array

```rust
fn main() {
    let mut senarai: Vec<String> = Vec::new();

    senarai.push(String::from("Ali"));
    senarai.push(String::from("Ahmad"));
    senarai.push(String::from("Siti"));

    // Iterate
    for nama in &senarai {
        println!("- {}", nama);
    }

    println!("Jumlah: {}", senarai.len());

    // Remove
    senarai.retain(|n| n != "Ahmad");
    println!("Selepas buang Ahmad: {:?}", senarai);
}
```

---

## HashMap — Key-Value Store

```rust
use std::collections::HashMap;

fn main() {
    let mut markah: HashMap<&str, u32> = HashMap::new();

    markah.insert("Ali", 85);
    markah.insert("Siti", 92);
    markah.insert("Amin", 78);

    // Akses nilai
    if let Some(m) = markah.get("Ali") {
        println!("Ali dapat: {}", m);
    }

    // Insert kalau belum ada
    markah.entry("Zara").or_insert(70);

    // Iterate
    for (nama, nilai) in &markah {
        println!("{}: {}", nama, nilai);
    }
}
```

---

# BAB 8: Traits 🎭

## Traits — Interface Dalam Rust

```rust
trait Boleh Sapa {
    fn sapa(&self) -> String;
    fn nama(&self) -> &str;  // required method
}

struct Manusia {
    nama: String,
}

struct Robot {
    id: u32,
}

impl BolehSapa for Manusia {
    fn sapa(&self) -> String {
        format!("Assalamualaikum, saya {}", self.nama)
    }
    fn nama(&self) -> &str { &self.nama }
}

impl BolehSapa for Robot {
    fn sapa(&self) -> String {
        format!("BEEP BOOP. ID: {}", self.id)
    }
    fn nama(&self) -> &str { "Robot" }
}

fn buat_perkenalan(entiti: &dyn BolehSapa) {
    println!("{}", entiti.sapa());
}

fn main() {
    let ali = Manusia { nama: "Ali".to_string() };
    let r2d2 = Robot { id: 42 };

    buat_perkenalan(&ali);
    buat_perkenalan(&r2d2);
}
```

---

# BAB 9: Mini Project — CLI Todo App ✅

## Bina Todo App Dari Scratch!

```
todo-app/
├── Cargo.toml
└── src/
    └── main.rs
```

**Cargo.toml:**
```toml
[package]
name = "todo-app"
version = "0.1.0"
edition = "2021"

[dependencies]
```

**src/main.rs:**
```rust
use std::io::{self, Write};

#[derive(Debug)]
struct Todo {
    id: u32,
    tajuk: String,
    selesai: bool,
}

impl Todo {
    fn baru(id: u32, tajuk: &str) -> Self {
        Todo {
            id,
            tajuk: tajuk.to_string(),
            selesai: false,
        }
    }

    fn papar(&self) {
        let status = if self.selesai { "✅" } else { "⬜" };
        println!("{}  [{}] {}", status, self.id, self.tajuk);
    }
}

fn main() {
    let mut senarai: Vec<Todo> = Vec::new();
    let mut id_semasa: u32 = 1;

    println!("🦀 Rust Todo App");
    println!("================");
    println!("Arahan: tambah <tajuk> | selesai <id> | papar | keluar\n");

    loop {
        print!("> ");
        io::stdout().flush().unwrap();

        let mut input = String::new();
        io::stdin().read_line(&mut input).unwrap();
        let input = input.trim();

        if input.starts_with("tambah ") {
            let tajuk = &input[7..];
            senarai.push(Todo::baru(id_semasa, tajuk));
            println!("✔ Ditambah: {}", tajuk);
            id_semasa += 1;

        } else if input.starts_with("selesai ") {
            if let Ok(id) = input[8..].parse::<u32>() {
                if let Some(todo) = senarai.iter_mut().find(|t| t.id == id) {
                    todo.selesai = true;
                    println!("✅ Selesai: {}", todo.tajuk);
                } else {
                    println!("❌ ID tidak dijumpai.");
                }
            }

        } else if input == "papar" {
            if senarai.is_empty() {
                println!("(tiada todo)");
            } else {
                for todo in &senarai {
                    todo.papar();
                }
            }

        } else if input == "keluar" {
            println!("Terima kasih! Selamat tinggal 🦀");
            break;

        } else {
            println!("Arahan tidak dikenali.");
        }
    }
}
```

**Cara run:**
```bash
cargo run
```

**Contoh sesi:**
```
> tambah Belajar Rust
✔ Ditambah: Belajar Rust
> tambah Buat KADA Mobile App
✔ Ditambah: Buat KADA Mobile App
> papar
⬜  [1] Belajar Rust
⬜  [2] Buat KADA Mobile App
> selesai 1
✅ Selesai: Belajar Rust
> papar
✅  [1] Belajar Rust
⬜  [2] Buat KADA Mobile App
> keluar
Terima kasih! Selamat tinggal 🦀
```

---

# 🎯 Rujukan Pantas (Cheat Sheet)

## PHP → Rust Perbandingan

| PHP | Rust |
|-----|------|
| `$x = 5;` | `let x = 5;` |
| `$x = 5; $x = 6;` | `let mut x = 5; x = 6;` |
| `echo "hello";` | `println!("hello");` |
| `function f($a) {}` | `fn f(a: i32) {}` |
| `null` | `None` |
| `try/catch` | `Result<T, E>` + `?` |
| `array()` | `Vec<T>` |
| `array_push` | `.push()` |
| `class Foo {}` | `struct Foo {}` + `impl Foo {}` |
| `interface Bar` | `trait Bar` |
| `throw new Exception` | `return Err(...)` |

---

## Symbols Yang Selalu Confuse

| Symbol | Maksud |
|--------|--------|
| `&` | Borrow/reference |
| `&mut` | Mutable borrow |
| `*` | Dereference |
| `?` | Propagate error |
| `::` | Path separator / associated function |
| `->` | Return type |
| `=>` | Match arm |
| `..=` | Inclusive range |
| `..` | Exclusive range / struct update syntax |

---

## 🏆 Cabaran Akhir

Cuba bina salah satu daripada ini:

1. **Kalkulator CLI** — tambah, tolak, darab, bahagi dengan error handling
2. **Phonebook** — simpan & cari kenalan (nama + nombor telefon)
3. **Penjana Password** — random password berdasarkan panjang yang ditetapkan
4. **Student Grade Tracker** — input markah, papar gred & purata

---

## 📚 Sumber Lanjut

- [The Rust Book](https://doc.rust-lang.org/book/) — buku rasmi, percuma online
- [Rustlings](https://github.com/rust-lang/rustlings) — latihan interaktif
- [Rust By Example](https://doc.rust-lang.org/rust-by-example/) — belajar dari contoh
- [crates.io](https://crates.io) — library ecosystem Rust

---

*Head First Rust (Lite) — Dibina dengan ❤️ untuk pembangun yang nak faham Rust tanpa pening kepala.*
