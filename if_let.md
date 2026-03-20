# 🔍 if let dalam Rust — Beginner to Advanced

> Panduan lengkap if let, while let, let else, dan if let chains.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu if let?

```
match yang terlalu verbose untuk satu kes:

  match nilai {
      Some(x) => println!("{}", x),
      _        => {}             // baris ini sia-sia!
  }

if let — lebih ringkas, sama maksud:

  if let Some(x) = nilai {
      println!("{}", x);
  }
  // Kalau tidak match → skip, tiada panic, tiada masalah!
```

**`if let` = match + hanya peduli SATU pattern, abaikan yang lain.**

---

## Kenapa if let Wujud?

```
Tanpa if let (verbose):             Dengan if let (bersih):
  match opt {                         if let Some(n) = opt {
      Some(n) => proses(n),               proses(n);
      None    => {},                  }
  }

Tanpa if let (panjang):             Dengan if let + else:
  match res {                         if let Ok(v) = res {
      Ok(v)  => guna(v),                  guna(v);
      Err(e) => log(e),               } else {
  }                                       log("error");
                                      }
```

---

## Peta Pembelajaran

```
Bab 1  → if let Asas
Bab 2  → if let dengan else & else if
Bab 3  → if let Chains (Rust 1.64+)
Bab 4  → while let
Bab 5  → let else — Early Return Pattern
Bab 6  → if let dengan Enums
Bab 7  → if let dalam Struct Methods
Bab 8  → if let vs match — Bila Pilih Apa
Bab 9  → Pattern Lanjutan dengan if let
Bab 10 → Mini Project: Sistem Log Parser
```

---

# BAB 1: if let Asas 🌱

## Sintaks Asas

```rust
fn main() {
    // Option<T> — kes paling biasa
    let hadiah: Option<&str> = Some("🎁 Kereta mewah");

    // Cara lama (verbose)
    match hadiah {
        Some(hadiah) => println!("Dapat: {}", hadiah),
        None         => {},   // buat apa-apa pun tidak
    }

    // Cara baru dengan if let
    if let Some(hadiah) = hadiah {
        println!("Dapat: {}", hadiah);
    }
    // Kalau None → langkau blok, tiada error!

    // Dengan pelbagai jenis
    let nombor: Option<i32> = Some(42);
    if let Some(n) = nombor {
        println!("Nombor: {}", n);
        println!("Kali dua: {}", n * 2);
    }

    // None — tiada output, tiada masalah
    let tiada: Option<i32> = None;
    if let Some(n) = tiada {
        println!("Ini tidak akan print: {}", n);
    }
    println!("Program terus jalan!");
}
```

---

## if let dengan Result

```rust
fn bahagi(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err("Tidak boleh bahagi dengan sifar!".to_string())
    } else {
        Ok(a / b)
    }
}

fn main() {
    // if let dengan Ok
    if let Ok(hasil) = bahagi(10.0, 2.0) {
        println!("10 ÷ 2 = {}", hasil); // 5.0
    }

    // if let dengan Err
    if let Err(mesej) = bahagi(5.0, 0.0) {
        println!("Error: {}", mesej);
    }

    // Kedua-dua serentak
    let hasil1 = bahagi(10.0, 3.0);
    if let Ok(jawapan) = hasil1 {
        println!("Berjaya: {:.4}", jawapan); // 3.3333
    }

    // Dengan parse
    let input = "42";
    if let Ok(nombor) = input.parse::<i32>() {
        println!("Parse berjaya: {}", nombor * 2); // 84
    }

    let bukan_nombor = "abc";
    if let Ok(n) = bukan_nombor.parse::<i32>() {
        println!("Ini tidak print: {}", n);
    }
    println!("Parse gagal — terus jalan");
}
```

---

## if let dengan Enum

```rust
#[derive(Debug)]
enum Warna {
    Merah,
    Hijau,
    Biru,
    Rgb(u8, u8, u8),
    Nama(&'static str),
}

fn main() {
    let w = Warna::Rgb(255, 128, 0);

    // Hanya peduli Rgb
    if let Warna::Rgb(r, g, b) = w {
        println!("Warna RGB: #{:02X}{:02X}{:02X}", r, g, b); // #FF8000
    }

    // Unit variant
    let w2 = Warna::Merah;
    if let Warna::Merah = w2 {
        println!("Warna merah!");
    }

    // String data dalam variant
    let w3 = Warna::Nama("putih");
    if let Warna::Nama(nama) = w3 {
        println!("Nama warna: {}", nama);
    }
}
```

---

## 🧠 Brain Teaser #1

Apa output kod ini, dan kenapa?

```rust
fn main() {
    let a: Option<i32> = Some(0);
    let b: Option<i32> = None;
    let c: Option<i32> = Some(-5);

    if let Some(n) = a {
        println!("A: {}", n);
    }

    if let Some(n) = b {
        println!("B: {}", n);
    }

    if let Some(n) = c {
        if n > 0 {
            println!("C positif: {}", n);
        }
    }

    println!("Selesai");
}
```

<details>
<summary>👀 Jawapan</summary>

Output:
```
A: 0
Selesai
```

- `a = Some(0)` → match, print `A: 0`
- `b = None` → tidak match, langkau
- `c = Some(-5)` → match `Some(n)`, `n = -5`, tapi `n > 0` adalah false → langkau `println!`
- `println!("Selesai")` → sentiasa print

**Penting:** `if let` hanya check **pattern** — `Some(0)` adalah valid `Some(n)`. Check `n > 0` perlu dibuat secara berasingan atau guna guard.
</details>

---

# BAB 2: if let dengan else & else if 🔀

## if let + else

```rust
fn main() {
    let stok: Option<u32> = Some(5);

    // Guna else untuk kes tidak match
    if let Some(kuantiti) = stok {
        println!("Stok ada: {} unit", kuantiti);
    } else {
        println!("Stok habis!");
    }

    // None kes
    let stok_kosong: Option<u32> = None;
    if let Some(k) = stok_kosong {
        println!("Ada: {}", k);
    } else {
        println!("Tiada stok — perlu reorder!"); // ← ini print
    }

    // Result dengan else
    fn connect(host: &str) -> Result<String, String> {
        if host == "localhost" {
            Ok(format!("Berjaya sambung ke {}", host))
        } else {
            Err(format!("Gagal sambung ke {}", host))
        }
    }

    if let Ok(msg) = connect("localhost") {
        println!("✔ {}", msg);
    } else {
        println!("✘ Sambungan gagal");
    }

    // Guna else untuk dapat Err value
    match connect("server.jauh") {
        Ok(m)  => println!("OK: {}", m),
        Err(e) => println!("Err: {}", e),
    }
    // Atau — tapi kena match Err:
    if let Err(e) = connect("server.jauh") {
        println!("Error: {}", e);
    }
}
```

---

## if let + else if + else if let

```rust
fn main() {
    let input: Option<i32> = Some(85);

    // Gabungan if let, else if, else if let, else
    if let Some(n) = input {
        if n >= 90 {
            println!("Cemerlang: {}", n);
        } else if n >= 75 {
            println!("Baik: {}", n);       // ← ini print
        } else {
            println!("Perlu usaha: {}", n);
        }
    } else {
        println!("Tiada markah");
    }

    // Pelbagai jenis dalam chain
    let kod_status: Result<u16, &str> = Ok(404);

    if let Ok(200) = kod_status {
        println!("Berjaya");
    } else if let Ok(kod) = kod_status {
        // Ini match -- 404 != 200, tapi Ok(_) ya
        println!("Kod lain: {}", kod);     // ← ini print
    } else if let Err(e) = kod_status {
        println!("Error: {}", e);
    }
}
```

---

## Nested if let

```rust
#[derive(Debug)]
struct Pengguna {
    nama:  String,
    emel:  Option<String>,
    umur:  Option<u8>,
}

fn main() {
    let pengguna = Pengguna {
        nama:  "Ali".into(),
        emel:  Some("ali@email.com".into()),
        umur:  Some(25),
    };

    // Nested if let — boleh tapi boleh jadi "callback hell"
    if let Some(ref emel) = pengguna.emel {
        if let Some(umur) = pengguna.umur {
            if umur >= 18 {
                println!("{} (umur {}) boleh daftar: {}",
                    pengguna.nama, umur, emel);
            }
        }
    }

    // ↑ Boleh digantikan dengan if let chain (Bab 3)
    // atau with match yang lebih jelas
    match (&pengguna.emel, &pengguna.umur) {
        (Some(emel), Some(&umur)) if umur >= 18 => {
            println!("Cara match: {} boleh daftar", emel);
        }
        (Some(_), Some(_)) => println!("Terlalu muda"),
        _ => println!("Data tidak lengkap"),
    }
}
```

---

## 🧠 Brain Teaser #2

Tulis semula kod ini menggunakan `if let` agar lebih ringkas:

```rust
fn main() {
    let nilai: Option<i32> = Some(42);

    match nilai {
        Some(n) if n > 0 => println!("Positif: {}", n),
        Some(n)          => println!("Tidak positif: {}", n),
        None             => println!("Tiada"),
    }
}
```

<details>
<summary>👀 Jawapan</summary>

```rust
fn main() {
    let nilai: Option<i32> = Some(42);

    if let Some(n) = nilai {
        if n > 0 {
            println!("Positif: {}", n);
        } else {
            println!("Tidak positif: {}", n);
        }
    } else {
        println!("Tiada");
    }
}
```

**Nota:** Dalam kes ini, `match` sebenarnya lebih jelas dan ringkas kerana ada 3 kes dengan guard! `if let` lebih sesuai bila hanya ada 1-2 kes yang penting. Pilih yang lebih mudah dibaca.
</details>

---

# BAB 3: if let Chains — Rust 1.64+ ⛓️

## Masalah: Nested if let = Kod Teruk

```rust
// ❌ Nested if let — susah dibaca, "pyramid of doom"
fn proses_lama(a: Option<i32>, b: Option<i32>, c: Option<i32>) {
    if let Some(x) = a {
        if let Some(y) = b {
            if let Some(z) = c {
                println!("{} + {} + {} = {}", x, y, z, x+y+z);
            }
        }
    }
}

// ✔ if let chain — satu baris, bersih!
fn proses_baru(a: Option<i32>, b: Option<i32>, c: Option<i32>) {
    if let Some(x) = a
        && let Some(y) = b
        && let Some(z) = c
    {
        println!("{} + {} + {} = {}", x, y, z, x+y+z);
    }
}

fn main() {
    proses_lama(Some(1), Some(2), Some(3));  // 1 + 2 + 3 = 6
    proses_baru(Some(1), Some(2), Some(3));  // 1 + 2 + 3 = 6

    proses_lama(Some(1), None, Some(3));     // tiada output
    proses_baru(Some(1), None, Some(3));     // tiada output
}
```

---

## if let Chain dengan Conditions

```rust
fn main() {
    // Gabungan if let + &&  + syarat biasa
    let nama: Option<String> = Some("Ali Ahmad".into());
    let umur: Option<u8>     = Some(20);
    let aktif = true;

    // if let chain — semua mesti true untuk masuk blok
    if let Some(n) = &nama
        && let Some(u) = umur
        && u >= 18
        && aktif
    {
        println!("{} (umur {}) layak mendaftar!", n, u);
    }

    // Contoh praktikal — validate form
    let emel:  Option<&str> = Some("ali@contoh.com");
    let pass:  Option<&str> = Some("password123");
    let sahkan: Option<&str> = Some("password123");

    if let Some(e) = emel
        && let Some(p) = pass
        && let Some(s) = sahkan
        && p == s
        && e.contains('@')
    {
        println!("✔ Pendaftaran sah: {}", e);
    } else {
        println!("✘ Data tidak lengkap atau tidak sah");
    }

    // Gagal kerana password tidak sepadan
    let pass2:  Option<&str> = Some("abc");
    let sahkan2: Option<&str> = Some("xyz");

    if let Some(e) = emel
        && let Some(p) = pass2
        && let Some(s) = sahkan2
        && p == s    // ← false! abc != xyz
    {
        println!("Ini tidak print");
    } else {
        println!("✘ Password tidak sepadan");  // ← ini print
    }
}
```

---

## if let Chain dalam Real Code

```rust
use std::collections::HashMap;

fn main() {
    let mut cache: HashMap<&str, String> = HashMap::new();
    cache.insert("sesi_123", "Ali Ahmad|ICT|admin".into());

    fn dapatkan_sesi<'a>(
        cache: &'a HashMap<&str, String>,
        token: &str,
    ) -> Option<(&'a str, &'a str, &'a str)> {
        let data = cache.get(token)?;
        let mut bahagian = data.splitn(3, '|');
        Some((
            bahagian.next()?,
            bahagian.next()?,
            bahagian.next()?,
        ))
    }

    let token = "sesi_123";

    // if let chain untuk extract nested data
    if let Some((nama, bahagian, peranan)) = dapatkan_sesi(&cache, token)
        && peranan == "admin"
    {
        println!("Admin log masuk: {} dari {}", nama, bahagian);
    } else {
        println!("Akses ditolak");
    }

    // Token tidak wujud
    if let Some((nama, _, _)) = dapatkan_sesi(&cache, "token_palsu")
        && nama.len() > 0
    {
        println!("Ini tidak print");
    } else {
        println!("Sesi tidak ditemui"); // ← ini print
    }
}
```

---

# BAB 4: while let 🔄

## while let Asas

```rust
fn main() {
    // Pop dari stack sehingga kosong
    let mut stack = vec![1, 2, 3, 4, 5];

    println!("Pop dari stack:");
    while let Some(nilai) = stack.pop() {
        println!("  {}", nilai); // 5, 4, 3, 2, 1
    }
    println!("Stack kosong: {}", stack.is_empty()); // true

    // Drain iterator
    let mut iter = vec!["Ali", "Siti", "Amin"].into_iter();
    while let Some(nama) = iter.next() {
        println!("Nama: {}", nama);
    }
}
```

---

## while let dengan Channel

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<String>();

    // Spawn thread untuk hantar mesej
    thread::spawn(move || {
        for i in 1..=5 {
            tx.send(format!("Mesej #{}", i)).unwrap();
        }
        // tx drop di sini — channel ditutup
    });

    // Terima sehingga channel ditutup
    while let Ok(mesej) = rx.recv() {
        println!("Terima: {}", mesej);
    }
    println!("Channel ditutup, semua mesej diterima!");
}
```

---

## while let dengan Custom Iterator

```rust
struct Kiraan {
    semasa: u32,
    had:    u32,
}

impl Kiraan {
    fn baru(had: u32) -> Self { Kiraan { semasa: 0, had } }
}

impl Iterator for Kiraan {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        if self.semasa < self.had {
            self.semasa += 1;
            Some(self.semasa)
        } else {
            None
        }
    }
}

fn main() {
    let mut kira = Kiraan::baru(5);

    while let Some(n) = kira.next() {
        println!("Nilai: {}", n); // 1, 2, 3, 4, 5
    }

    // Cara yang lebih idiomatik (guna for loop):
    for n in Kiraan::baru(3) {
        println!("For: {}", n); // 1, 2, 3
    }
}
```

---

## while let dengan Break & Continue

```rust
fn main() {
    let mut data = vec![
        Some(1),
        Some(2),
        None,      // ← while let akan stop di sini!
        Some(4),
        Some(5),
    ];
    data.reverse();

    // Akan berhenti di None
    while let Some(Some(n)) = data.pop() {
        println!("Dapat: {}", n); // 1, 2 sahaja (stop bila ada None)
    }

    // Guna break untuk logik lebih kompleks
    let mut nombor = vec![2, 4, 6, 7, 8, 10];
    nombor.reverse();
    while let Some(n) = nombor.pop() {
        if n % 2 != 0 {
            println!("Jumpa ganjil: {} — berhenti!", n);
            break;
        }
        println!("Genap: {}", n);
    }
}
```

---

## 🧠 Brain Teaser #3

Apa perbezaan antara `while let` dan `for` loop?

```rust
let v = vec![1, 2, 3, 4, 5];

// Cara A:
for x in &v { println!("{}", x); }

// Cara B:
let mut iter = v.iter();
while let Some(x) = iter.next() { println!("{}", x); }
```

<details>
<summary>👀 Jawapan</summary>

Output SAMA — kedua-duanya print 1 2 3 4 5.

**Perbezaan:**

| | `for` loop | `while let` |
|--|--|--|
| Sintaks | Ringkas, idiomatik | Verbose |
| Kawalan | Terhad | Lebih fleksibel |
| `break` custom | Ya | Ya |
| Skip item | `continue` | `continue` |
| Guna ke-`n` | `iter.nth(n)` | Boleh |

**Bila guna `while let` berbanding `for`:**
- Nak akses iterator secara manual (contoh: `iter.next()` dua kali serentak)
- Nak check pattern yang lebih kompleks (contoh: `Some(Some(x))`)
- Pop dari stack/queue (`vec.pop()`)
- Guna channel `rx.recv()`

Dalam kebanyakan kes, `for` loop lebih idiomatik dan ringkas.
</details>

---

# BAB 5: let else — Early Return Pattern ↩️

## Masalah yang let else Selesaikan

```rust
// ❌ Tanpa let else — perlu nested if let atau match
fn proses_lama(input: Option<&str>) -> Result<i32, String> {
    if let Some(s) = input {
        if let Ok(n) = s.parse::<i32>() {
            if n > 0 {
                return Ok(n * 2);
            } else {
                return Err("Perlu nombor positif".into());
            }
        } else {
            return Err(format!("'{}' bukan nombor", s));
        }
    } else {
        return Err("Input tiada".into());
    }
}

// ✔ Dengan let else — linear, mudah dibaca
fn proses_baru(input: Option<&str>) -> Result<i32, String> {
    let Some(s) = input else {
        return Err("Input tiada".into());
    };

    let Ok(n) = s.parse::<i32>() else {
        return Err(format!("'{}' bukan nombor", s));
    };

    if n <= 0 {
        return Err("Perlu nombor positif".into());
    }

    Ok(n * 2)
}

fn main() {
    println!("{:?}", proses_baru(None));              // Err("Input tiada")
    println!("{:?}", proses_baru(Some("abc")));       // Err("'abc' bukan nombor")
    println!("{:?}", proses_baru(Some("-5")));        // Err("Perlu nombor positif")
    println!("{:?}", proses_baru(Some("21")));        // Ok(42)
}
```

---

## let else — Pelbagai Konteks

```rust
fn main() {
    // let else dalam fungsi yang return Option
    fn dapatkan_emel(id: u32) -> Option<String> {
        let pengguna = vec![
            (1u32, "ali@email.com"),
            (2u32, "siti@email.com"),
        ];

        // Cari pengguna dengan id
        let Some((_, emel)) = pengguna.iter().find(|(uid, _)| *uid == id) else {
            return None;  // tidak jumpa → return None
        };

        Some(emel.to_string())
    }

    println!("{:?}", dapatkan_emel(1)); // Some("ali@email.com")
    println!("{:?}", dapatkan_emel(9)); // None

    // let else dalam loop
    let data = vec!["10", "abc", "30", "40"];
    let mut jumlah = 0i32;

    for s in &data {
        let Ok(n) = s.parse::<i32>() else {
            println!("Skip '{}' — bukan nombor", s);
            continue;  // skip ke iterasi seterusnya
        };
        jumlah += n;
    }
    println!("Jumlah: {}", jumlah); // 80 (10+30+40)

    // let else dengan break dalam loop
    let mut nombor = vec![2, 4, 6, 8, 11, 12];
    let mut hasil = vec![];
    loop {
        let Some(n) = nombor.first().copied() else {
            break;  // kosong → keluar
        };
        nombor.remove(0);

        let true = (n % 2 == 0) else {
            println!("Berhenti di ganjil: {}", n);
            break;
        };

        hasil.push(n);
    }
    println!("Genap sebelum ganjil: {:?}", hasil); // [2,4,6,8]
}
```

---

## let else vs if let — Perbandingan

```
if let Some(x) = opt {        let Some(x) = opt else {
    // guna x                     // diverge (return/break/panic)
    // x hanya dalam blok ini  };
}                              // x tersedia di luar!

PERBEZAAN UTAMA:

if let    → x hanya dalam blok if
            → kes "tidak match" dalam else
            → untuk proses bila match

let else  → x tersedia selepas let else
            → blok else MESTI diverge (return/break/continue/panic)
            → untuk "jika tidak match, keluar"
```

```rust
fn main() {
    let opt = Some(42);

    // if let — x hanya dalam blok
    if let Some(x) = opt {
        println!("x = {}", x); // x ada di sini
    }
    // println!("{}", x);  // ← ERROR! x tiada di sini

    // let else — x tersedia di bawah
    let Some(x) = opt else { return; };
    println!("x = {}", x); // ← x TERSEDIA di sini!
    println!("guna x lagi: {}", x * 2);
}
```

---

## 🧠 Brain Teaser #4

Apakah masalah kod ini, dan bagaimana betulkan?

```rust
fn ambil_port(config: &str) -> u16 {
    let Some(nilai) = config.split_once('=').map(|(_, v)| v) else {
        println!("Tiada '=' dalam config");
        // tiada return/break/continue/panic!
    };
    nilai.trim().parse().unwrap_or(8080)
}

fn main() {
    println!("{}", ambil_port("port=3000"));
    println!("{}", ambil_port("tiada"));
}
```

<details>
<summary>👀 Jawapan</summary>

**Masalah:** Blok `else` dalam `let else` MESTI diverge (return, break, continue, atau panic). Kod ini hanya ada `println!` dalam else — ini ERROR compile!

```
error[E0308]: `else` clause of `let...else` does not diverge
```

**Betulkan:**
```rust
fn ambil_port(config: &str) -> u16 {
    let Some(nilai) = config.split_once('=').map(|(_, v)| v) else {
        println!("Tiada '=' dalam config");
        return 8080;  // ← mesti ada return!
    };
    nilai.trim().parse().unwrap_or(8080)
}
```

Atau guna `panic!` jika input tidak sah adalah bug:
```rust
    } else { panic!("Config tidak sah: {}", config); };
```

Atau `unreachable!()` kalau kes tu sepatutnya tidak berlaku.
</details>

---

# BAB 6: if let dengan Enums Kompleks 🎭

## Enum Bersarang

```rust
#[derive(Debug)]
enum Respons {
    Berjaya {
        kod:  u16,
        data: Option<String>,
    },
    Ralat {
        kod:     u16,
        mesej:   String,
        cuba_lagi: bool,
    },
    Redirect(String),
    Timeout,
}

fn tangani(resp: &Respons) {
    // if let dengan struct variant
    if let Respons::Berjaya { kod: 200, data: Some(d) } = resp {
        println!("✔ 200 OK: {}", d);
        return;
    }

    if let Respons::Berjaya { kod, data: None } = resp {
        println!("✔ Kod {} tanpa data", kod);
        return;
    }

    if let Respons::Ralat { kod, mesej, cuba_lagi: true } = resp {
        println!("✘ Error {} (boleh cuba lagi): {}", kod, mesej);
        return;
    }

    if let Respons::Ralat { kod, mesej, .. } = resp {
        println!("✘ Error {}: {}", kod, mesej);
        return;
    }

    if let Respons::Redirect(url) = resp {
        println!("→ Redirect ke {}", url);
        return;
    }

    if let Respons::Timeout = resp {
        println!("⏱ Request timeout");
    }
}

fn main() {
    tangani(&Respons::Berjaya { kod: 200, data: Some("hasil query".into()) });
    tangani(&Respons::Berjaya { kod: 204, data: None });
    tangani(&Respons::Ralat   { kod: 503, mesej: "Overloaded".into(), cuba_lagi: true });
    tangani(&Respons::Ralat   { kod: 404, mesej: "Not Found".into(),  cuba_lagi: false });
    tangani(&Respons::Redirect("https://baru.com".into()));
    tangani(&Respons::Timeout);
}
```

---

## Gabungan if let + matches!

```rust
fn main() {
    let statuses = vec![
        Some(200u16),
        Some(404),
        None,
        Some(500),
        Some(200),
        Some(301),
    ];

    // matches! — lebih ringkas untuk check pattern
    let berjaya_count = statuses.iter()
        .filter(|s| matches!(s, Some(200..=299)))
        .count();
    println!("Berjaya: {}", berjaya_count); // 2

    // if let dalam iterator
    let kod_ralat: Vec<u16> = statuses.iter()
        .filter_map(|s| {
            if let Some(kod @ 400..=599) = s {
                Some(*kod)
            } else {
                None
            }
        })
        .collect();
    println!("Kod ralat: {:?}", kod_ralat); // [404, 500]
}
```

---

# BAB 7: if let dalam Struct Methods 🔧

## Self Methods dengan if let

```rust
#[derive(Debug)]
struct Wallet {
    baki: f64,
    had:  Option<f64>, // had belanja, boleh tiada
}

impl Wallet {
    fn baru(baki: f64) -> Self {
        Wallet { baki, had: None }
    }

    fn dengan_had(mut self, had: f64) -> Self {
        self.had = Some(had);
        self
    }

    fn belanja(&mut self, jumlah: f64) -> bool {
        // if let untuk semak had
        if let Some(had) = self.had {
            if jumlah > had {
                println!("✘ Melebihi had belanja RM{:.2}", had);
                return false;
            }
        }

        if jumlah > self.baki {
            println!("✘ Baki tidak mencukupi (RM{:.2})", self.baki);
            return false;
        }

        self.baki -= jumlah;
        println!("✔ Beli RM{:.2} | Baki: RM{:.2}", jumlah, self.baki);
        true
    }

    fn tambah(&mut self, jumlah: f64) {
        self.baki += jumlah;
        println!("+ Isi RM{:.2} | Baki: RM{:.2}", jumlah, self.baki);
    }

    fn had_info(&self) {
        if let Some(had) = self.had {
            println!("Had belanja: RM{:.2}", had);
        } else {
            println!("Tiada had belanja");
        }
    }
}

fn main() {
    let mut w = Wallet::baru(100.0).dengan_had(50.0);
    w.had_info();

    w.belanja(30.0);  // OK
    w.belanja(60.0);  // Melebihi had
    w.belanja(80.0);  // Baki tidak mencukupi
    w.tambah(200.0);
    w.belanja(45.0);  // OK (baki 225, had 50)
}
```

---

# BAB 8: if let vs match — Bila Pilih Apa? ⚖️

## Panduan Pilih

```
PILIH if let BILA:
  ✔ Hanya peduli SATU pattern
  ✔ Kes lain tidak penting atau hanya else
  ✔ Nak kod yang lebih "linear" dan mudah baca
  ✔ Dalam method yang check satu kondisi

PILIH match BILA:
  ✔ Ada 3+ kes yang penting
  ✔ Perlu exhaustive check (compiler paksa semua kes)
  ✔ Tiap kes ada logik berbeza
  ✔ Kerja dengan enum yang ada banyak variant
```

```rust
#[derive(Debug)]
enum Status { Aktif, Tidak, Pending, Dibuang }

fn main() {
    let s = Status::Aktif;

    // ✔ if let — sesuai: hanya peduli Aktif
    if let Status::Aktif = s {
        println!("Aktif!");
    }

    // ✔ match — sesuai: semua status penting
    let label = match s {
        Status::Aktif   => "🟢 Aktif",
        Status::Tidak   => "🔴 Tidak aktif",
        Status::Pending => "🟡 Menunggu",
        Status::Dibuang => "⚫ Dibuang",
    };
    println!("{}", label);

    // ❌ match tidak perlu jika hanya satu kes
    // (verbose tanpa sebab)
    match s {
        Status::Aktif => println!("Aktif!"),
        _             => {},  // buang baris ini guna if let
    }

    // ✔ if let lebih baik di atas
    if let Status::Aktif = s {
        println!("Aktif!");
    }
}
```

---

## Perbandingan Kod Sebenar

```rust
// Senario: cek permission sebelum buat sesuatu

#[derive(Debug)]
enum Peranan { Admin, Editor, Pengguna, Tetamu }

struct Sesi {
    nama:    String,
    peranan: Option<Peranan>,
    aktif:   bool,
}

// Versi dengan match
fn boleh_edit_match(sesi: &Sesi) -> bool {
    if !sesi.aktif { return false; }
    match &sesi.peranan {
        Some(Peranan::Admin)  => true,
        Some(Peranan::Editor) => true,
        _                     => false,
    }
}

// Versi dengan if let
fn boleh_edit_iflet(sesi: &Sesi) -> bool {
    if !sesi.aktif { return false; }
    if let Some(Peranan::Admin | Peranan::Editor) = &sesi.peranan {
        true
    } else {
        false
    }
}

// Lebih ringkas lagi dengan matches!
fn boleh_edit_matches(sesi: &Sesi) -> bool {
    sesi.aktif && matches!(&sesi.peranan, Some(Peranan::Admin | Peranan::Editor))
}

fn main() {
    let sesi = Sesi {
        nama:    "Ali".into(),
        peranan: Some(Peranan::Editor),
        aktif:   true,
    };

    println!("match:   {}", boleh_edit_match(&sesi));   // true
    println!("if let:  {}", boleh_edit_iflet(&sesi));   // true
    println!("matches: {}", boleh_edit_matches(&sesi)); // true
}
```

---

# BAB 9: Pattern Lanjutan dengan if let 🎯

## if let dengan @ Binding

```rust
fn main() {
    let skor: Option<u32> = Some(87);

    // @ binding dalam if let
    if let Some(n @ 90..=100) = skor {
        println!("Cemerlang: {}", n);
    } else if let Some(n @ 75..=89) = skor {
        println!("Baik: {}", n);          // ← ini print
    } else if let Some(n) = skor {
        println!("Lulus: {}", n);
    } else {
        println!("Gagal");
    }

    // @ dalam enum
    #[derive(Debug)]
    enum Paket { Kecil(u32), Sederhana(u32), Besar(u32) }

    let p = Paket::Sederhana(350);
    if let Paket::Sederhana(berat @ 300..=500) = p {
        println!("Paket sederhana {}g — dalam julat!", berat);
    }
}
```

---

## if let dengan Destructure Kompleks

```rust
#[derive(Debug)]
struct Titik { x: f64, y: f64 }

#[derive(Debug)]
enum Bentuk {
    Bulatan { pusat: Titik, radius: f64 },
    Segiempat { sudut: Titik, lebar: f64, tinggi: f64 },
}

fn main() {
    let b = Bentuk::Bulatan {
        pusat:  Titik { x: 5.0, y: 5.0 },
        radius: 3.0,
    };

    // Destructure nested dalam if let
    if let Bentuk::Bulatan { pusat: Titik { x, y }, radius } = &b {
        println!("Bulatan di ({}, {}) r={}", x, y, radius);
    }

    // Destructure dengan guard
    let s = Bentuk::Segiempat {
        sudut:  Titik { x: 0.0, y: 0.0 },
        lebar:  10.0,
        tinggi: 5.0,
    };

    if let Bentuk::Segiempat { lebar, tinggi, .. } = &s
        && lebar > tinggi
    {
        println!("Segiempat melebar: {}×{}", lebar, tinggi);
    }
}
```

---

## if let dalam Async Code

```rust
use tokio::time::{sleep, Duration};

async fn ambil_data(id: u32) -> Option<String> {
    sleep(Duration::from_millis(10)).await;
    if id > 0 {
        Some(format!("Data untuk ID {}", id))
    } else {
        None
    }
}

async fn proses(id: u32) -> String {
    // if let berfungsi sama dalam async context
    if let Some(data) = ambil_data(id).await {
        format!("Proses: {}", data)
    } else {
        "Tiada data".to_string()
    }
}

#[tokio::main]
async fn main() {
    println!("{}", proses(1).await);  // Proses: Data untuk ID 1
    println!("{}", proses(0).await);  // Tiada data
}
```

---

## 🧠 Brain Teaser #5 (Advanced)

Tulis fungsi `filter_dan_transform` yang:
1. Terima `Vec<Option<&str>>`
2. Skip semua `None`
3. Skip string yang tidak boleh di-parse sebagai nombor
4. Darabkan nombor dengan 2
5. Return `Vec<i32>`

Guna kombinasi `while let`, `if let`, dan `let else`.

<details>
<summary>👀 Jawapan</summary>

```rust
fn filter_dan_transform(data: Vec<Option<&str>>) -> Vec<i32> {
    let mut hasil = Vec::new();
    let mut iter  = data.into_iter();

    while let Some(item) = iter.next() {
        // Skip None
        let Some(s) = item else { continue; };

        // Skip yang tidak boleh parse
        let Ok(n) = s.trim().parse::<i32>() else { continue; };

        hasil.push(n * 2);
    }
    hasil
}

// Cara lebih idiomatik dengan iterator:
fn filter_dan_transform_v2(data: Vec<Option<&str>>) -> Vec<i32> {
    data.into_iter()
        .filter_map(|opt| {
            let s = opt?;                    // None → skip
            s.trim().parse::<i32>().ok()     // parse gagal → skip
        })
        .map(|n| n * 2)
        .collect()
}

fn main() {
    let input = vec![Some("5"), None, Some("abc"), Some("10"), Some(" 3 "), None];
    println!("{:?}", filter_dan_transform(input.clone()));   // [10, 20, 6]
    println!("{:?}", filter_dan_transform_v2(input));        // [10, 20, 6]
}
```
</details>

---

# BAB 10: Mini Project — Sistem Log Parser 📋

```rust
use std::collections::HashMap;

// ─── Model ────────────────────────────────────────────────────

#[derive(Debug, Clone, PartialEq)]
enum Tahap { Debug, Info, Warn, Error, Fatal }

#[derive(Debug, Clone)]
struct LogEntry {
    timestamp: String,
    tahap:     Tahap,
    modul:     String,
    mesej:     String,
    kod_ralat: Option<u32>,
}

impl LogEntry {
    fn parse(baris: &str) -> Option<Self> {
        // Format: [TIMESTAMP] TAHAP modul: mesej [E:kod]
        // Contoh: [2024-01-15 10:30:00] ERROR db: sambungan gagal [E:500]

        let baris = baris.trim();

        // Skip komen dan baris kosong
        let true = !baris.is_empty() && !baris.starts_with('#') else {
            return None;
        };

        // Parse timestamp dalam [ ]
        let Some(ts_end) = baris.find(']') else { return None; };
        let timestamp    = baris[1..ts_end].to_string();
        let baki         = baris[ts_end+1..].trim();

        // Parse tahap
        let mut bahagian = baki.splitn(3, ' ');

        let Some(tahap_str) = bahagian.next() else { return None; };
        let tahap = match tahap_str {
            "DEBUG" => Tahap::Debug,
            "INFO"  => Tahap::Info,
            "WARN"  => Tahap::Warn,
            "ERROR" => Tahap::Error,
            "FATAL" => Tahap::Fatal,
            _       => return None,
        };

        let Some(modul_dan_mesej) = bahagian.next() else { return None; };
        let Some(mesej_raw)       = bahagian.next() else { return None; };

        // Buang ':' dari modul
        let modul = modul_dan_mesej.trim_end_matches(':').to_string();

        // Parse mesej dan kod ralat pilihan [E:xxx]
        let (mesej, kod_ralat) = if let Some(pos) = mesej_raw.rfind("[E:") {
            let mesej_bersih = mesej_raw[..pos].trim().to_string();
            let kod_str      = &mesej_raw[pos+3..mesej_raw.len()-1];
            let kod          = kod_str.parse::<u32>().ok();
            (mesej_bersih, kod)
        } else {
            (mesej_raw.trim().to_string(), None)
        };

        Some(LogEntry { timestamp, tahap, modul, mesej, kod_ralat })
    }

    fn adalah_ralat(&self) -> bool {
        matches!(self.tahap, Tahap::Error | Tahap::Fatal)
    }

    fn simbol(&self) -> &str {
        match self.tahap {
            Tahap::Debug => "🔵",
            Tahap::Info  => "🟢",
            Tahap::Warn  => "🟡",
            Tahap::Error => "🔴",
            Tahap::Fatal => "💀",
        }
    }
}

// ─── Analyzer ─────────────────────────────────────────────────

struct LogAnalyzer {
    entries: Vec<LogEntry>,
}

impl LogAnalyzer {
    fn muat(teks: &str) -> Self {
        let entries = teks.lines()
            .filter_map(LogEntry::parse)  // if let inline!
            .collect();
        LogAnalyzer { entries }
    }

    fn papar_semua(&self) {
        for e in &self.entries {
            if let Some(kod) = e.kod_ralat {
                println!("{} [{}] {}/{}: {} [E:{}]",
                    e.simbol(), e.timestamp, e.tahap_str(), e.modul, e.mesej, kod);
            } else {
                println!("{} [{}] {}/{}: {}",
                    e.simbol(), e.timestamp, e.tahap_str(), e.modul, e.mesej);
            }
        }
    }

    fn hanya_ralat(&self) -> Vec<&LogEntry> {
        self.entries.iter()
            .filter(|e| e.adalah_ralat())
            .collect()
    }

    fn ikut_modul(&self) -> HashMap<&str, Vec<&LogEntry>> {
        let mut peta: HashMap<&str, Vec<&LogEntry>> = HashMap::new();
        for e in &self.entries {
            peta.entry(&e.modul).or_default().push(e);
        }
        peta
    }

    fn cari_kod(&self, kod: u32) -> Vec<&LogEntry> {
        self.entries.iter()
            .filter(|e| {
                if let Some(k) = e.kod_ralat {
                    k == kod
                } else {
                    false
                }
            })
            .collect()
    }

    fn statistik(&self) {
        let mut kira: HashMap<String, u32> = HashMap::new();
        for e in &self.entries {
            *kira.entry(e.tahap_str().to_string()).or_insert(0) += 1;
        }

        println!("\n{'─'*40}");
        println!("Statistik Log:");
        for (tahap, bil) in &kira {
            println!("  {:<8}: {}", tahap, bil);
        }
        println!("  {'─'*20}");
        println!("  Jumlah: {}", self.entries.len());
    }
}

impl LogEntry {
    fn tahap_str(&self) -> &str {
        match self.tahap {
            Tahap::Debug => "DEBUG",
            Tahap::Info  => "INFO",
            Tahap::Warn  => "WARN",
            Tahap::Error => "ERROR",
            Tahap::Fatal => "FATAL",
        }
    }
}

// ─── Main ──────────────────────────────────────────────────────

fn main() {
    let log_data = r#"
# Log sistem KADA Mobile API

[2024-01-15 08:00:01] INFO  server: Pelayan dimulakan
[2024-01-15 08:00:05] INFO  db: Sambungan database berjaya
[2024-01-15 08:15:22] DEBUG api: Terima request GET /api/pekerja
[2024-01-15 08:15:23] INFO  api: Hantar 42 rekod pekerja
[2024-01-15 09:30:01] WARN  auth: Cuba log masuk gagal untuk user 'admin'
[2024-01-15 09:30:05] WARN  auth: Cuba log masuk gagal kali ke-3
[2024-01-15 10:00:00] ERROR db: Sambungan database timeout [E:500]
[2024-01-15 10:00:01] ERROR db: Retry sambungan gagal [E:503]
[2024-01-15 10:00:05] INFO  db: Sambungan dipulihkan
[2024-01-15 11:45:30] DEBUG gps: Terima koordinat (3.1412, 101.6865)
[2024-01-15 11:45:31] INFO  gps: Kehadiran direkod untuk ID 1001
[2024-01-15 14:00:00] FATAL server: Memori tidak mencukupi [E:507]
[2024-01-15 14:00:01] INFO  server: Restart automatik dimulakan
    "#;

    let analyzer = LogAnalyzer::muat(log_data);

    println!("=== SEMUA LOG ===");
    analyzer.papar_semua();

    println!("\n=== RALAT SAHAJA ===");
    let ralat = analyzer.hanya_ralat();
    for e in &ralat {
        if let Some(kod) = e.kod_ralat {
            println!("  {} {} — kod {}", e.simbol(), e.mesej, kod);
        } else {
            println!("  {} {}", e.simbol(), e.mesej);
        }
    }

    println!("\n=== CARI KOD 503 ===");
    let kod503 = analyzer.cari_kod(503);
    if kod503.is_empty() {
        println!("  Tiada log dengan kod 503");
    } else {
        for e in kod503 {
            println!("  {} [{}] {}", e.simbol(), e.timestamp, e.mesej);
        }
    }

    println!("\n=== IKUT MODUL ===");
    let ikut_modul = analyzer.ikut_modul();
    let mut modul: Vec<&&str> = ikut_modul.keys().collect();
    modul.sort();
    for m in modul {
        let entries = &ikut_modul[m];
        let ralat_count = entries.iter().filter(|e| e.adalah_ralat()).count();
        println!("  {:<10}: {} log ({} ralat)", m, entries.len(), ralat_count);
    }

    analyzer.statistik();
}
```

---

# 📋 Rujukan Pantas — if let Cheat Sheet

## Sintaks Asas

```rust
// if let
if let PATTERN = NILAI { .. }
if let PATTERN = NILAI { .. } else { .. }
if let PATTERN = NILAI { .. } else if SYARAT { .. } else { .. }

// if let chain (Rust 1.64+)
if let P1 = v1 && let P2 = v2 && SYARAT { .. }

// while let
while let PATTERN = NILAI { .. }

// let else
let PATTERN = NILAI else { /* mesti diverge */ };
```

## Pattern yang Biasa Digunakan

```rust
if let Some(x) = option { .. }
if let Ok(v)   = result { .. }
if let Err(e)  = result { .. }
if let Enum::Variant(data) = val { .. }
if let Enum::Struct { field } = val { .. }
if let Some(x @ 1..=10) = opt { .. }   // dengan @
if let [a, b, ..] = slice { .. }        // slice pattern
```

## Padanan Cepat

```
match val { Some(x) => f(x), _ => {} }
  → if let Some(x) = val { f(x) }

match val { Ok(v) => use(v), Err(e) => handle(e) }
  → if let Ok(v) = val { use(v) } else { handle_err() }

while sth { match iter.next() { Some(x) => .., None => break } }
  → while let Some(x) = iter.next() { .. }

if let Some(x) = a { if let Some(y) = b { .. } }
  → if let Some(x) = a && let Some(y) = b { .. }

if let Some(x) = val { use(x) } else { return Err(..) }
  → let Some(x) = val else { return Err(..) }; use(x)
```

## if let vs match vs let else

```
Bila guna if let:
  → Satu atau dua kes penting
  → Kes lain boleh abaikan atau else ringkas

Bila guna match:
  → Banyak variant/kes
  → Perlu exhaustive check
  → Tiap kes ada logik berbeza

Bila guna let else:
  → "Jika tidak match, keluar (return/break/panic)"
  → Nak binding yang tersedia selepas blok
  → Validation di awal fungsi
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **State Machine** — guna `while let` untuk proses event hingga Terminate
2. **Command Processor** — `while let Some(cmd) = input.next()` dengan let else validation
3. **File Iterator** — baca baris fail dengan `while let Some(Ok(baris)) = iter.next()`
4. **Retry Logic** — cuba operasi dengan `while let Err(e) = cuba()` sehingga berjaya atau maks cuba
5. **Tree Traversal** — guna `if let Some(nod) = pokok.anak` untuk traverse binary tree

---

*if let in Rust — dari `if let Some(x)` ringkas hingga let-else dan chains kompleks. Selamat belajar!* 🦀
