# 🏗️ Struct & Enum dalam Rust — Beginner to Advanced

> Panduan lengkap: Struct, Enum, match, Option dan Result sebagai pengganti null & exception.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Gambaran Besar

```
Rust tiada class seperti Java/PHP/Python.
Sebagai gantinya:

  STRUCT = data sahaja (macam class tanpa method dulu)
  impl   = method untuk struct
  ENUM   = jenis yang ada beberapa kemungkinan bentuk

  Option<T>  = pengganti null  (Some(x) atau None)
  Result<T,E> = pengganti exception (Ok(x) atau Err(e))

PHP:                        Rust:
  class Pekerja { }           struct Pekerja { }
  null                        None
  throw Exception             return Err(...)
  nullable $x = null          Option<T>
```

---

## Peta Pembelajaran

```
Bab 1  → Struct Asas
Bab 2  → Struct Methods dengan impl
Bab 3  → Tuple Struct & Unit Struct
Bab 4  → Enum Asas
Bab 5  → Enum dengan Data
Bab 6  → match dengan Enum
Bab 7  → Option<T> — Pengganti null
Bab 8  → Result<T,E> — Pengganti Exception
Bab 9  → Gabungan Struct + Enum + Option + Result
Bab 10 → Mini Project: Sistem Pengurusan Staf
```

---

# BAB 1: Struct Asas 🏗️

## Apa Itu Struct?

```
Struct = kumpulan data yang ada nama.

Macam borang:
  ┌───────────────────────────────────┐
  │  Borang Pendaftaran Pekerja       │
  ├───────────────────────────────────┤
  │  Nama:     Ali Ahmad              │
  │  ID:       1001                   │
  │  Gaji:     4500.00                │
  │  Aktif:    Ya                     │
  └───────────────────────────────────┘

Dalam Rust:
  struct Pekerja {
      nama:   String,
      id:     u32,
      gaji:   f64,
      aktif:  bool,
  }
```

```rust
// Define struct
struct Pekerja {
    nama:     String,
    id:       u32,
    gaji:     f64,
    aktif:    bool,
}

fn main() {
    // Buat instance — semua field mesti diisi!
    let pekerja1 = Pekerja {
        nama:  String::from("Ali Ahmad"),
        id:    1001,
        gaji:  4500.0,
        aktif: true,
    };

    // Akses field dengan .
    println!("Nama: {}", pekerja1.nama);
    println!("ID:   {}", pekerja1.id);
    println!("Gaji: RM{:.2}", pekerja1.gaji);

    // ⚠ TIDAK BOLEH ubah kalau tidak mut
    // pekerja1.gaji = 5000.0; // ERROR!

    // Gunakan mut untuk struct yang boleh diubah
    let mut pekerja2 = Pekerja {
        nama:  String::from("Siti Hawa"),
        id:    1002,
        gaji:  3800.0,
        aktif: true,
    };

    pekerja2.gaji = 4200.0; // OK dengan mut
    println!("Gaji baru Siti: RM{:.2}", pekerja2.gaji);

    // Struct update syntax — ambil baki dari struct lain
    let pekerja3 = Pekerja {
        nama: String::from("Amin Razak"),
        id:   1003,
        ..pekerja2  // gaji, aktif ambil dari pekerja2
    };
    println!("{} gaji: RM{:.2}", pekerja3.nama, pekerja3.gaji); // 4200.0
}
```

---

## Debug & Display untuk Struct

```rust
// #[derive(Debug)] — auto-generate debug output {:?}
#[derive(Debug, Clone)]
struct Titik {
    x: f64,
    y: f64,
}

#[derive(Debug)]
struct Segiempat {
    sudut_kiri_atas:  Titik,
    lebar:            f64,
    tinggi:           f64,
}

fn main() {
    let t = Titik { x: 3.0, y: 4.0 };
    let s = Segiempat {
        sudut_kiri_atas: Titik { x: 0.0, y: 0.0 },
        lebar:   10.0,
        tinggi:  5.0,
    };

    println!("{:?}", t);   // Titik { x: 3.0, y: 4.0 }
    println!("{:#?}", s);  // Pretty-print dengan indent

    // Clone
    let t2 = t.clone();
    println!("Original: {:?}", t);
    println!("Klon:     {:?}", t2);
}
```

---

## 🧠 Brain Teaser #1

Apa yang akan berlaku bila kod ini dijalankan?

```rust
struct Buku {
    tajuk:   String,
    halaman: u32,
}

fn main() {
    let b1 = Buku {
        tajuk:   String::from("Rust Programming"),
        halaman: 500,
    };

    let b2 = b1; // apa jadi pada b1?

    println!("{}", b2.tajuk);
    println!("{}", b1.tajuk); // adakah ini OK?
}
```

<details>
<summary>👀 Jawapan</summary>

**Tidak compile!** — `b1.tajuk` tidak lagi valid selepas `let b2 = b1`.

Apabila kita tulis `let b2 = b1`, ownership `b1` (termasuk `String` dalam `tajuk`) **dipindah (move)** ke `b2`. `b1` tidak lagi valid.

**Penyelesaian:**
```rust
// Cara 1: Clone
let b2 = b1.clone(); // perlukan #[derive(Clone)]

// Cara 2: Borrow
let b2 = &b1; // b2 adalah reference kepada b1

// Cara 3: Guna terus tanpa assign
println!("{}", b1.tajuk); // sebelum b2
let b2 = b1;
println!("{}", b2.tajuk);
```

Ini adalah **Ownership** — salah satu konsep paling unik Rust!
</details>

---

# BAB 2: Struct Methods dengan impl 🔧

## Tambah Methods

```rust
#[derive(Debug, Clone)]
struct Bulatan {
    radius: f64,
}

// impl block — tempat letak method untuk Bulatan
impl Bulatan {
    // Associated function (tiada &self) — macam "static method"
    // Guna untuk "constructor"
    fn baru(radius: f64) -> Bulatan {
        Bulatan { radius }
    }

    // Method — ambil &self (borrow, tidak ubah)
    fn luas(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }

    fn perimeter(&self) -> f64 {
        2.0 * std::f64::consts::PI * self.radius
    }

    // Method — ambil &mut self (boleh ubah nilai)
    fn skala(&mut self, faktor: f64) {
        self.radius *= faktor;
    }

    // Method — ambil self (consume struct, return baru)
    fn dengan_radius_baru(self, radius_baru: f64) -> Bulatan {
        Bulatan { radius: radius_baru }
    }

    fn huraian(&self) -> String {
        format!("Bulatan(r={:.2}, luas={:.2})", self.radius, self.luas())
    }
}

fn main() {
    // Guna associated function (bukan new().field tapi ::baru())
    let mut b = Bulatan::baru(5.0);

    println!("{}", b.huraian());
    println!("Luas:      {:.4}", b.luas());
    println!("Perimeter: {:.4}", b.perimeter());

    b.skala(2.0); // radius jadi 10.0
    println!("Selepas skala 2x: {}", b.huraian());

    // Consume dan buat baru
    let b2 = b.dengan_radius_baru(3.0);
    // b tidak lagi boleh guna sekarang (moved)
    println!("Bulatan baru: {}", b2.huraian());
}
```

---

## Method Chaining — Builder Pattern

```rust
#[derive(Debug)]
struct PembuatSQL {
    jadual:  String,
    medan:   Vec<String>,
    syarat:  Vec<String>,
    had:     Option<u32>,
}

impl PembuatSQL {
    fn baru(jadual: &str) -> Self {
        PembuatSQL {
            jadual:  jadual.to_string(),
            medan:   Vec::new(),
            syarat:  Vec::new(),
            had:     None,
        }
    }

    // Setiap method return &mut Self untuk chaining
    fn pilih(mut self, medan: &str) -> Self {
        self.medan.push(medan.to_string());
        self
    }

    fn di_mana(mut self, syarat: &str) -> Self {
        self.syarat.push(syarat.to_string());
        self
    }

    fn had(mut self, n: u32) -> Self {
        self.had = Some(n);
        self
    }

    fn bina(&self) -> String {
        let medan = if self.medan.is_empty() {
            "*".to_string()
        } else {
            self.medan.join(", ")
        };

        let mut sql = format!("SELECT {} FROM {}", medan, self.jadual);

        if !self.syarat.is_empty() {
            sql.push_str(" WHERE ");
            sql.push_str(&self.syarat.join(" AND "));
        }

        if let Some(n) = self.had {
            sql.push_str(&format!(" LIMIT {}", n));
        }

        sql
    }
}

fn main() {
    // Method chaining!
    let sql = PembuatSQL::baru("pekerja")
        .pilih("nama")
        .pilih("gaji")
        .di_mana("aktif = true")
        .di_mana("gaji > 3000")
        .had(10)
        .bina();

    println!("{}", sql);
    // SELECT nama, gaji FROM pekerja WHERE aktif = true AND gaji > 3000 LIMIT 10
}
```

---

# BAB 3: Tuple Struct & Unit Struct 📦

## Tuple Struct — Struct dengan Posisi

```rust
// Tuple struct — seperti tuple tapi ada nama type
struct Meter(f64);
struct Kilogram(f64);
struct Warna(u8, u8, u8); // RGB

impl Meter {
    fn ke_km(&self) -> f64 { self.0 / 1000.0 }
    fn tambah(&self, lain: &Meter) -> Meter { Meter(self.0 + lain.0) }
}

impl Kilogram {
    fn ke_gram(&self) -> f64 { self.0 * 1000.0 }
}

impl Warna {
    fn html(&self) -> String {
        format!("#{:02X}{:02X}{:02X}", self.0, self.1, self.2)
    }
}

fn main() {
    let jarak  = Meter(1500.0);
    let berat   = Kilogram(75.0);
    let merah   = Warna(255, 0, 0);
    let biru    = Warna(0, 0, 255);

    println!("{}m = {}km", jarak.0, jarak.ke_km());
    println!("{}kg = {}g", berat.0, berat.ke_gram());
    println!("Merah: {}", merah.html()); // #FF0000
    println!("Biru:  {}", biru.html());  // #0000FF

    // Destructure tuple struct
    let Meter(nilai)         = jarak;
    let Warna(r, g, b)       = merah;
    println!("Jarak: {}m, R={} G={} B={}", nilai, r, g, b);

    // Type safety — tidak boleh campur!
    // Meter(5.0) + Kilogram(3.0); // ERROR! type berbeza
    let j1 = Meter(100.0);
    let j2 = Meter(200.0);
    let j3 = j1.tambah(&j2);
    println!("{}m + {}m = {}m", j1.0, j2.0, j3.0);
}
```

---

## Unit Struct — Struct Tanpa Data

```rust
// Unit struct — tiada data, berguna untuk traits
struct Pencatat;
struct PencatatSenyap;

trait Log {
    fn log(&self, mesej: &str);
}

impl Log for Pencatat {
    fn log(&self, mesej: &str) {
        println!("[LOG] {}", mesej);
    }
}

impl Log for PencatatSenyap {
    fn log(&self, _mesej: &str) {
        // Tidak buat apa-apa!
    }
}

fn buat_kerja(logger: &dyn Log) {
    logger.log("Mula kerja");
    // buat kerja...
    logger.log("Kerja selesai");
}

fn main() {
    let log = Pencatat;
    let senyap = PencatatSenyap;

    println!("=== Dengan log ===");
    buat_kerja(&log);

    println!("=== Tanpa log ===");
    buat_kerja(&senyap);
}
```

---

# BAB 4: Enum Asas 🎭

## Apa Itu Enum?

```
Enum = "salah satu daripada beberapa kemungkinan"

Macam status lampu isyarat:
  ┌─────────────────────────────────────┐
  │  enum LampuIsyarat {                │
  │      Merah,    // BERHENTI          │
  │      Kuning,   // BERHATI-HATI      │
  │      Hijau,    // PERGI             │
  │  }                                  │
  └─────────────────────────────────────┘

  Lampu HANYA boleh ada SATU keadaan pada satu masa.
  Tidak mungkin Merah DAN Hijau serentak!
```

```rust
// Define enum
#[derive(Debug, PartialEq)]
enum Hari {
    Isnin,
    Selasa,
    Rabu,
    Khamis,
    Jumaat,
    Sabtu,
    Ahad,
}

impl Hari {
    fn adalah_hujung_minggu(&self) -> bool {
        matches!(self, Hari::Sabtu | Hari::Ahad)
    }

    fn nama(&self) -> &str {
        match self {
            Hari::Isnin  => "Isnin",
            Hari::Selasa => "Selasa",
            Hari::Rabu   => "Rabu",
            Hari::Khamis => "Khamis",
            Hari::Jumaat => "Jumaat",
            Hari::Sabtu  => "Sabtu",
            Hari::Ahad   => "Ahad",
        }
    }
}

fn main() {
    let hari_ini = Hari::Rabu;
    println!("Hari: {}", hari_ini.nama());
    println!("Hujung minggu? {}", hari_ini.adalah_hujung_minggu());

    // Comparison
    if hari_ini == Hari::Jumaat {
        println!("Yeay! Hari Jumaat!");
    } else {
        println!("Bukan hari Jumaat lagi...");
    }

    // Dalam Vec
    let jadual = vec![Hari::Isnin, Hari::Rabu, Hari::Jumaat];
    for h in &jadual {
        println!("{}", h.nama());
    }
}
```

---

## 🧠 Brain Teaser #2

Tulis enum `ArahKompas` dengan 4 nilai (Utara, Selatan, Timur, Barat) dan fungsi `bertentangan` yang return arah bertentangan.

<details>
<summary>👀 Jawapan</summary>

```rust
#[derive(Debug, PartialEq)]
enum ArahKompas { Utara, Selatan, Timur, Barat }

impl ArahKompas {
    fn bertentangan(&self) -> ArahKompas {
        match self {
            ArahKompas::Utara  => ArahKompas::Selatan,
            ArahKompas::Selatan => ArahKompas::Utara,
            ArahKompas::Timur  => ArahKompas::Barat,
            ArahKompas::Barat  => ArahKompas::Timur,
        }
    }
}

fn main() {
    let arah = ArahKompas::Utara;
    let lawan = arah.bertentangan();
    println!("{:?} ↔ {:?}", arah, lawan); // Utara ↔ Selatan
}
```
</details>

---

# BAB 5: Enum dengan Data 💾

## Enum Boleh Simpan Data!

```
Ini yang membezakan Rust enum dari bahasa lain!
Setiap variant boleh ada data berbeza.

enum Mesej {
    Berhenti,                    // tiada data
    Gerak { x: i32, y: i32 },   // struct-like data
    Tulis(String),               // tuple-like data
    TukarWarna(u8, u8, u8),      // tiga nilai
}
```

```rust
#[derive(Debug)]
enum Mesej {
    Berhenti,
    Gerak { x: i32, y: i32 },
    Tulis(String),
    TukarWarna(u8, u8, u8),
}

impl Mesej {
    fn proses(&self) {
        match self {
            Mesej::Berhenti              => println!("⏹ Berhenti!"),
            Mesej::Gerak { x, y }        => println!("→ Gerak ke ({}, {})", x, y),
            Mesej::Tulis(teks)           => println!("✏ Tulis: {}", teks),
            Mesej::TukarWarna(r, g, b)   => println!("🎨 Warna: RGB({},{},{})", r, g, b),
        }
    }
}

fn main() {
    let mesej_list = vec![
        Mesej::Gerak { x: 10, y: 20 },
        Mesej::Tulis(String::from("Hello, Rust!")),
        Mesej::TukarWarna(255, 128, 0),
        Mesej::Berhenti,
    ];

    for m in &mesej_list {
        m.proses();
    }
}
```

---

## Enum untuk State Machine

```rust
#[derive(Debug, PartialEq)]
enum StatusPesanan {
    Baru,
    Disahkan { masa_sahkan: String },
    Diproses,
    Dihantar { no_tracking: String },
    Selesai,
    Dibatalkan { sebab: String },
}

impl StatusPesanan {
    fn boleh_batal(&self) -> bool {
        matches!(self,
            StatusPesanan::Baru |
            StatusPesanan::Disahkan { .. } |
            StatusPesanan::Diproses
        )
    }

    fn label(&self) -> &str {
        match self {
            StatusPesanan::Baru              => "🆕 Baru",
            StatusPesanan::Disahkan { .. }   => "✅ Disahkan",
            StatusPesanan::Diproses          => "⚙️  Diproses",
            StatusPesanan::Dihantar { .. }   => "🚚 Dihantar",
            StatusPesanan::Selesai           => "🎉 Selesai",
            StatusPesanan::Dibatalkan { .. } => "❌ Dibatalkan",
        }
    }
}

fn main() {
    let mut status = StatusPesanan::Baru;
    println!("{} — boleh batal: {}", status.label(), status.boleh_batal());

    status = StatusPesanan::Disahkan {
        masa_sahkan: "2024-01-15 10:30".to_string()
    };
    println!("{} — boleh batal: {}", status.label(), status.boleh_batal());

    status = StatusPesanan::Dihantar {
        no_tracking: "MY123456".to_string()
    };

    if let StatusPesanan::Dihantar { no_tracking } = &status {
        println!("{} — Track: {}", status.label(), no_tracking);
    }

    println!("Boleh batal lagi? {}", status.boleh_batal()); // false
}
```

---

# BAB 6: match dengan Enum 🎯

## match — Lebih Berkuasa dari switch

```rust
#[derive(Debug)]
enum Bentuk {
    Bulatan(f64),
    Segiempat { lebar: f64, tinggi: f64 },
    Segitiga { tapak: f64, tinggi: f64 },
}

fn kira_luas(b: &Bentuk) -> f64 {
    match b {
        Bentuk::Bulatan(r)                      => std::f64::consts::PI * r * r,
        Bentuk::Segiempat { lebar, tinggi }      => lebar * tinggi,
        Bentuk::Segitiga  { tapak, tinggi }      => 0.5 * tapak * tinggi,
    }
}

fn main() {
    let bentuk = vec![
        Bentuk::Bulatan(5.0),
        Bentuk::Segiempat { lebar: 4.0, tinggi: 6.0 },
        Bentuk::Segitiga  { tapak: 3.0, tinggi: 8.0 },
    ];

    for b in &bentuk {
        println!("{:?} → luas = {:.2}", b, kira_luas(b));
    }

    // match dengan guard (if)
    let r = 7.0f64;
    let kategori = match r {
        x if x <= 0.0  => "tidak sah",
        x if x <= 5.0  => "kecil",
        x if x <= 10.0 => "sederhana",
        _              => "besar",
    };
    println!("Radius {} → {}", r, kategori);

    // match adalah expression — ada nilai!
    let b = Bentuk::Bulatan(3.0);
    let label = match &b {
        Bentuk::Bulatan(_)      => "bulat",
        Bentuk::Segiempat { .. }=> "bersegi",
        Bentuk::Segitiga  { .. }=> "tiga sisi",
    };
    println!("Bentuk: {}", label);
}
```

---

## match Exhaustive — Compiler Jaga!

```rust
#[derive(Debug)]
enum Musim { Sejuk, Panas, Hujan }

fn pakaian(m: &Musim) -> &str {
    match m {
        Musim::Sejuk  => "Baju tebal",
        Musim::Panas  => "Baju nipis",
        Musim::Hujan  => "Payung + jaket",
        // Cuba comment mana-mana baris ↑ — compiler ERROR!
        // "non-exhaustive patterns"
    }
}

fn main() {
    for musim in &[Musim::Sejuk, Musim::Panas, Musim::Hujan] {
        println!("{:?}: {}", musim, pakaian(musim));
    }

    // match dengan | (OR pattern)
    let nilai = 4;
    match nilai {
        1 | 2        => println!("Satu atau dua"),
        3 | 4 | 5    => println!("Tiga, empat, atau lima"),
        6..=10       => println!("Enam hingga sepuluh"),
        _            => println!("Lain-lain"),
    }

    // match dengan @ binding
    let skor = 85u32;
    match skor {
        n @ 90..=100 => println!("A — skor: {}", n),
        n @ 75..=89  => println!("B — skor: {}", n), // ← ini match!
        n @ 60..=74  => println!("C — skor: {}", n),
        n            => println!("Gagal — skor: {}", n),
    }
}
```

---

## 🧠 Brain Teaser #3

Apa output kod ini? Dan kenapa `_` diperlukan?

```rust
enum Syiling { Sen1, Sen5, Sen10, Sen20, Sen50, RM1 }

fn nilai_sen(s: &Syiling) -> u32 {
    match s {
        Syiling::Sen1  => 1,
        Syiling::Sen5  => 5,
        Syiling::Sen10 => 10,
        Syiling::Sen20 => 20,
        Syiling::Sen50 => 50,
        Syiling::RM1   => 100,
    }
}

fn main() {
    let duit = vec![Syiling::RM1, Syiling::Sen50, Syiling::Sen20];
    let jumlah: u32 = duit.iter().map(nilai_sen).sum();
    println!("Jumlah: {} sen", jumlah);
}
```

<details>
<summary>👀 Jawapan</summary>

Output: `Jumlah: 170 sen`

100 + 50 + 20 = 170 sen.

`_` dalam `match` adalah wildcard — tangkap semua kes yang belum di-handle. Dalam kod ini, `_` TIDAK diperlukan kerana kita sudah handle SEMUA 6 variant. Kalau tinggalkan satu (contoh, `Syiling::Sen50`), compiler akan ERROR: "non-exhaustive patterns: `Sen50` not covered".

Inilah kehebatan Rust — bila tambah variant baru ke enum, **semua match akan gagal compile** sehingga kita update! Zero silent bugs.
</details>

---

# BAB 7: Option<T> — Pengganti null 🔍

## Masalah dengan null

```
Bahasa lain:
  String nama = null;      // silently boleh jadi null
  nama.length();           // NullPointerException! 💥
  // Tidak tahu fungsi boleh return null atau tidak

Rust:
  let nama: Option<String> = None;
  nama.len();              // ERROR semasa compile!
  // Compiler PAKSA kita check sama ada ada nilai atau tidak
```

## Option<T> Asas

```rust
fn main() {
    // Option<T> adalah enum dalam std:
    // enum Option<T> { Some(T), None }

    let ada:   Option<i32>    = Some(42);
    let tiada: Option<i32>    = None;
    let nama:  Option<String> = Some(String::from("Ali"));

    // Cara 1: match (paling eksplisit)
    match ada {
        Some(n) => println!("Ada nilai: {}", n),
        None    => println!("Tiada nilai"),
    }

    // Cara 2: if let (lebih ringkas untuk satu kes)
    if let Some(n) = ada {
        println!("Nilai: {}", n);
    }

    // Cara 3: unwrap_or (nilai default)
    println!("{}", tiada.unwrap_or(0));           // 0
    println!("{}", ada.unwrap_or(0));             // 42

    // Cara 4: map (transform nilai dalam Some)
    let dua_kali = ada.map(|n| n * 2);
    println!("{:?}", dua_kali); // Some(84)

    let x: Option<i32> = None;
    println!("{:?}", x.map(|n| n * 2)); // None (tidak berubah)

    // Cara 5: ? dalam fungsi yang return Option
    fn ambil_pertama(v: &[i32]) -> Option<i32> {
        let n = v.first()?;  // None jika kosong
        Some(n * 10)
    }

    println!("{:?}", ambil_pertama(&[1, 2, 3])); // Some(10)
    println!("{:?}", ambil_pertama(&[]));         // None
}
```

---

## Option dalam Struct

```rust
#[derive(Debug)]
struct Pengguna {
    nama:      String,
    emel:      String,
    no_telefon: Option<String>,  // mungkin ada, mungkin tidak
    alamat:    Option<String>,
}

impl Pengguna {
    fn baru(nama: &str, emel: &str) -> Self {
        Pengguna {
            nama:       nama.to_string(),
            emel:       emel.to_string(),
            no_telefon: None,  // tiada dulu
            alamat:     None,
        }
    }

    fn dengan_telefon(mut self, no: &str) -> Self {
        self.no_telefon = Some(no.to_string());
        self
    }

    fn dengan_alamat(mut self, alamat: &str) -> Self {
        self.alamat = Some(alamat.to_string());
        self
    }

    fn boleh_hubungi(&self) -> bool {
        self.no_telefon.is_some() || !self.emel.is_empty()
    }

    fn maklumat_hubungan(&self) -> String {
        let telefon = self.no_telefon
            .as_deref()         // Option<String> → Option<&str>
            .unwrap_or("tiada");
        format!("Emel: {}, Tel: {}", self.emel, telefon)
    }
}

fn main() {
    let u1 = Pengguna::baru("Ali Ahmad", "ali@email.com")
        .dengan_telefon("012-345-6789");

    let u2 = Pengguna::baru("Siti Hawa", "siti@email.com");

    println!("{:?}", u1);
    println!("Hubungi: {}", u1.maklumat_hubungan());
    println!("Hubungi: {}", u2.maklumat_hubungan());
    println!("u2 boleh dihubungi? {}", u2.boleh_hubungi()); // true (ada emel)
}
```

---

## Option Methods Berguna

```rust
fn main() {
    let ada: Option<i32>    = Some(10);
    let tiada: Option<i32>  = None;

    // is_some() / is_none()
    println!("{} {}", ada.is_some(), tiada.is_some()); // true false

    // map — transform Some
    println!("{:?}", ada.map(|n| n * 3));     // Some(30)
    println!("{:?}", tiada.map(|n| n * 3));   // None

    // and_then — chain (flatMap)
    fn kuasa_dua_positif(n: i32) -> Option<i32> {
        if n > 0 { Some(n * n) } else { None }
    }
    println!("{:?}", ada.and_then(kuasa_dua_positif));       // Some(100)
    println!("{:?}", Some(-5).and_then(kuasa_dua_positif));  // None

    // or / or_else — nilai alternatif
    println!("{:?}", tiada.or(Some(99)));         // Some(99)
    println!("{:?}", ada.or(Some(99)));           // Some(10) — ada kekal

    // filter — None kalau syarat gagal
    println!("{:?}", ada.filter(|&n| n > 5));    // Some(10)
    println!("{:?}", ada.filter(|&n| n > 50));   // None

    // unwrap_or_else — compute lazily
    println!("{}", tiada.unwrap_or_else(|| {
        println!("(mengira nilai default...)");
        42
    }));

    // flatten — Option<Option<T>> → Option<T>
    let nested: Option<Option<i32>> = Some(Some(7));
    println!("{:?}", nested.flatten());           // Some(7)

    // zip — gabung dua Option
    let a = Some(1);
    let b = Some("hello");
    println!("{:?}", a.zip(b));                  // Some((1, "hello"))
    println!("{:?}", a.zip(tiada));              // None
}
```

---

## 🧠 Brain Teaser #4

Tulis fungsi `cari_pekerja` yang terima `&[Pekerja]` dan `id: u32`, return `Option<&Pekerja>`. Test dengan ID yang ada dan tidak ada.

<details>
<summary>👀 Jawapan</summary>

```rust
#[derive(Debug)]
struct Pekerja {
    id:   u32,
    nama: String,
}

fn cari_pekerja<'a>(senarai: &'a [Pekerja], id: u32) -> Option<&'a Pekerja> {
    senarai.iter().find(|p| p.id == id)
}

fn main() {
    let pekerja = vec![
        Pekerja { id: 1, nama: "Ali".into() },
        Pekerja { id: 2, nama: "Siti".into() },
        Pekerja { id: 3, nama: "Amin".into() },
    ];

    // Cara 1: match
    match cari_pekerja(&pekerja, 2) {
        Some(p) => println!("Jumpa: {:?}", p),
        None    => println!("Tidak jumpa"),
    }

    // Cara 2: if let
    if let Some(p) = cari_pekerja(&pekerja, 1) {
        println!("ID 1: {}", p.nama);
    }

    // Cara 3: unwrap_or_else
    println!("{}", cari_pekerja(&pekerja, 9)
        .map(|p| p.nama.as_str())
        .unwrap_or("(tidak dijumpai)"));
}
```
</details>

---

# BAB 8: Result<T, E> — Pengganti Exception ⚠️

## Masalah dengan Exception

```
Bahasa lain:
  try {
      String isi = bacaFail("config.txt");  // boleh throw!
      int n = Integer.parseInt(isi);        // boleh throw!
  } catch (Exception e) {
      // mudah lupa catch
      // tidak tahu fungsi mana yang throw
  }

Rust:
  let isi = baca_fail("config.txt")?;  // Result<String, io::Error>
  let n = isi.trim().parse::<i32>()?;  // Result<i32, ParseError>
  // Compiler PAKSA handle!
  // Jelas fungsi mana yang boleh gagal
```

## Result<T, E> Asas

```rust
use std::num::ParseIntError;

// Fungsi yang return Result
fn bahagi(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("Tidak boleh bahagi dengan sifar"))
    } else {
        Ok(a / b)
    }
}

fn parse_dan_kira(input: &str) -> Result<i32, ParseIntError> {
    let n = input.trim().parse::<i32>()?;  // ? = return Err kalau gagal
    Ok(n * n)
}

fn main() {
    // match untuk handle Result
    match bahagi(10.0, 2.0) {
        Ok(hasil)  => println!("10 ÷ 2 = {}", hasil),
        Err(e)     => println!("Error: {}", e),
    }

    match bahagi(10.0, 0.0) {
        Ok(hasil)  => println!("Hasil: {}", hasil),
        Err(e)     => println!("Error: {}", e), // ← ini print
    }

    // if let untuk kes Ok sahaja
    if let Ok(hasil) = bahagi(15.0, 3.0) {
        println!("15 ÷ 3 = {}", hasil);
    }

    // Parse string
    println!("{:?}", parse_dan_kira("5"));   // Ok(25)
    println!("{:?}", parse_dan_kira("abc")); // Err(ParseIntError)
    println!("{:?}", parse_dan_kira("10"));  // Ok(100)

    // unwrap_or untuk nilai default
    println!("{}", parse_dan_kira("abc").unwrap_or(-1)); // -1
}
```

---

## Result dalam Kehidupan Sebenar

```rust
use std::fs;
use std::io;

#[derive(Debug)]
enum AppError {
    FailTidakWujud(io::Error),
    ParseGagal(String),
    NilaiTidakSah(String),
}

impl std::fmt::Display for AppError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            AppError::FailTidakWujud(e)  => write!(f, "Fail error: {}", e),
            AppError::ParseGagal(s)      => write!(f, "Parse gagal: {}", s),
            AppError::NilaiTidakSah(s)   => write!(f, "Nilai tidak sah: {}", s),
        }
    }
}

impl From<io::Error> for AppError {
    fn from(e: io::Error) -> Self { AppError::FailTidakWujud(e) }
}

fn muat_konfigurasi(path: &str) -> Result<u32, AppError> {
    // ? akan auto-convert io::Error → AppError::FailTidakWujud
    let kandungan = fs::read_to_string(path)?;

    let nilai: u32 = kandungan.trim()
        .parse()
        .map_err(|_| AppError::ParseGagal(kandungan.trim().to_string()))?;

    if nilai > 65535 {
        return Err(AppError::NilaiTidakSah(
            format!("{} melebihi had 65535", nilai)
        ));
    }

    Ok(nilai)
}

fn main() {
    // Test pelbagai kes
    let test_cases = vec![
        ("wujud_betul.txt",    "Fail betul"),
        ("tidak_wujud.txt",    "Fail tidak wujud"),
        ("nilai_salah.txt",    "Nilai bukan nombor"),
    ];

    // Simulasi dengan membuat fail sementara
    fs::write("wujud_betul.txt", "8080").ok();
    fs::write("nilai_salah.txt", "bukan_nombor").ok();

    for (path, _label) in &test_cases {
        match muat_konfigurasi(path) {
            Ok(port)                          =>
                println!("✔ Port: {}", port),
            Err(AppError::FailTidakWujud(_))  =>
                println!("✘ Fail '{}' tidak wujud", path),
            Err(AppError::ParseGagal(s))      =>
                println!("✘ '{}' bukan nombor sah", s),
            Err(AppError::NilaiTidakSah(s))   =>
                println!("✘ {}", s),
        }
    }

    // Cleanup
    fs::remove_file("wujud_betul.txt").ok();
    fs::remove_file("nilai_salah.txt").ok();
}
```

---

## Option vs Result — Bila Guna Mana?

```
Option<T>:
  → Nilai MUNGKIN ada atau tidak ada
  → Tiada maklumat kenapa tidak ada
  → Contoh: cari dalam senarai, ambil elemen pertama,
            field yang optional dalam struct

Result<T, E>:
  → Operasi MUNGKIN berjaya atau gagal
  → Ada maklumat tentang KENAPA gagal
  → Contoh: I/O, parse, network, database,
            apa-apa yang boleh gagal dengan sebab

Mudah ingat:
  "Ada atau tiada?"       → Option<T>
  "Berjaya atau gagal?"   → Result<T, E>
```

---

## 🧠 Brain Teaser #5

Tukar kod PHP ini ke Rust dengan Option dan Result:

```php
function bahagi_dan_akar($a, $b) {
    if ($b == 0) {
        return null;  // error!
    }
    $hasil = $a / $b;
    if ($hasil < 0) {
        return null;  // tidak boleh akar negatif
    }
    return sqrt($hasil);
}

$x = bahagi_dan_akar(16, 4);  // 2.0
$y = bahagi_dan_akar(16, 0);  // null
$z = bahagi_dan_akar(-4, 1);  // null
```

<details>
<summary>👀 Jawapan</summary>

```rust
// Cara 1: Option (bila tidak perlu tahu kenapa gagal)
fn bahagi_dan_akar_option(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 { return None; }
    let hasil = a / b;
    if hasil < 0.0 { return None; }
    Some(hasil.sqrt())
}

// Cara 2: Result (bila perlu tahu KENAPA gagal)
#[derive(Debug)]
enum KiraError { BahagiSifar, NilainegatifTidakBolehAkar }

fn bahagi_dan_akar_result(a: f64, b: f64) -> Result<f64, KiraError> {
    if b == 0.0 { return Err(KiraError::BahagiSifar); }
    let hasil = a / b;
    if hasil < 0.0 { return Err(KiraError::NilainegatifTidakBolehAkar); }
    Ok(hasil.sqrt())
}

fn main() {
    // Option version
    println!("{:?}", bahagi_dan_akar_option(16.0, 4.0));  // Some(2.0)
    println!("{:?}", bahagi_dan_akar_option(16.0, 0.0));  // None
    println!("{:?}", bahagi_dan_akar_option(-4.0, 1.0));  // None

    // Result version
    println!("{:?}", bahagi_dan_akar_result(16.0, 4.0));  // Ok(2.0)
    println!("{:?}", bahagi_dan_akar_result(16.0, 0.0));  // Err(BahagiSifar)
    println!("{:?}", bahagi_dan_akar_result(-4.0, 1.0));  // Err(NilainegatifTidakBolehAkar)
}
```

Pilih **Result** bila pengguna perlu tahu KENAPA gagal untuk handle dengan betul.
Pilih **Option** bila "ada atau tidak ada" sudah cukup maklumat.
</details>

---

# BAB 9: Gabungan Struct + Enum + Option + Result 🔗

## Pattern Lengkap

```rust
use std::collections::HashMap;

// ─── Types ────────────────────────────────────────────────────

#[derive(Debug, Clone, PartialEq)]
enum Peranan {
    Admin,
    Editor,
    Pengguna,
    Tetamu,
}

#[derive(Debug, Clone, PartialEq)]
enum StatusAkaun {
    Aktif,
    Tidak { sebab: String },
    BekuSementara { hingga: String },
}

#[derive(Debug, Clone)]
struct Pengguna {
    id:       u32,
    nama:     String,
    emel:     String,
    peranan:  Peranan,
    status:   StatusAkaun,
    no_tel:   Option<String>,  // optional
}

#[derive(Debug)]
enum DaftarError {
    EmelDuplikat(String),
    NamaTerlalPendek(String),
    EmelTidakSah(String),
}

impl std::fmt::Display for DaftarError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            DaftarError::EmelDuplikat(e)    => write!(f, "Emel '{}' sudah wujud", e),
            DaftarError::NamaTerlalPendek(n) => write!(f, "Nama '{}' terlalu pendek (min 3)", n),
            DaftarError::EmelTidakSah(e)    => write!(f, "Emel '{}' tidak sah", e),
        }
    }
}

// ─── Repository ───────────────────────────────────────────────

struct PenggunаRepo {
    data:       HashMap<u32, Pengguna>,
    id_seterus: u32,
}

impl PenggunаRepo {
    fn baru() -> Self {
        PenggunаRepo { data: HashMap::new(), id_seterus: 1 }
    }

    fn daftar(
        &mut self,
        nama: &str,
        emel: &str,
        peranan: Peranan,
    ) -> Result<&Pengguna, DaftarError> {
        // Validasi
        if nama.len() < 3 {
            return Err(DaftarError::NamaTerlalPendek(nama.into()));
        }
        if !emel.contains('@') {
            return Err(DaftarError::EmelTidakSah(emel.into()));
        }
        if self.data.values().any(|p| p.emel == emel) {
            return Err(DaftarError::EmelDuplikat(emel.into()));
        }

        let id = self.id_seterus;
        self.id_seterus += 1;

        let pengguna = Pengguna {
            id,
            nama:    nama.into(),
            emel:    emel.into(),
            peranan,
            status:  StatusAkaun::Aktif,
            no_tel:  None,
        };

        self.data.insert(id, pengguna);
        Ok(self.data.get(&id).unwrap())
    }

    fn cari_mengikut_id(&self, id: u32) -> Option<&Pengguna> {
        self.data.get(&id)
    }

    fn cari_mengikut_emel(&self, emel: &str) -> Option<&Pengguna> {
        self.data.values().find(|p| p.emel == emel)
    }

    fn beku(&mut self, id: u32, hingga: &str) -> Result<(), String> {
        match self.data.get_mut(&id) {
            Some(p) => {
                p.status = StatusAkaun::BekuSementara { hingga: hingga.into() };
                Ok(())
            }
            None => Err(format!("Pengguna ID {} tidak wujud", id)),
        }
    }

    fn semua_aktif(&self) -> Vec<&Pengguna> {
        self.data.values()
            .filter(|p| p.status == StatusAkaun::Aktif)
            .collect()
    }
}

// ─── Main ──────────────────────────────────────────────────────

fn main() {
    let mut repo = PenggunаRepo::baru();

    println!("=== Daftar Pengguna ===");

    // Pendaftaran berjaya
    let kes_daftar = vec![
        ("Ali Ahmad",  "ali@email.com",   Peranan::Admin),
        ("Siti Hawa",  "siti@email.com",  Peranan::Editor),
        ("Amin Razak", "amin@email.com",  Peranan::Pengguna),
        ("AB",         "ab@email.com",    Peranan::Pengguna),   // nama pendek
        ("Zara Lina",  "emel-tidak-sah",  Peranan::Pengguna),  // emel tak sah
        ("Ali Copy",   "ali@email.com",   Peranan::Pengguna),   // emel duplikat
    ];

    for (nama, emel, peranan) in kes_daftar {
        match repo.daftar(nama, emel, peranan) {
            Ok(p)  => println!("  ✔ Daftar berjaya: ID={} {}", p.id, p.nama),
            Err(e) => println!("  ✘ Gagal daftar '{}': {}", nama, e),
        }
    }

    println!("\n=== Cari Pengguna ===");

    // Cari dengan Option
    match repo.cari_mengikut_id(2) {
        Some(p) => println!("  ID 2: {} ({})", p.nama, p.emel),
        None    => println!("  ID 2: tidak wujud"),
    }

    if let Some(p) = repo.cari_mengikut_emel("amin@email.com") {
        println!("  Emel amin@: {} — {:?}", p.nama, p.peranan);
    }

    println!("  ID 99: {:?}", repo.cari_mengikut_id(99)); // None

    println!("\n=== Beku Akaun ===");

    match repo.beku(2, "2024-12-31") {
        Ok(()) => println!("  ✔ Akaun ID 2 dibekukan"),
        Err(e) => println!("  ✘ {}", e),
    }

    match repo.beku(99, "2024-12-31") {
        Ok(())  => println!("  ✔ OK"),
        Err(e) => println!("  ✘ {}", e),
    }

    println!("\n=== Pengguna Aktif ===");

    let aktif = repo.semua_aktif();
    println!("  Jumlah aktif: {}", aktif.len());
    for p in aktif {
        let tel = p.no_tel.as_deref().unwrap_or("tiada");
        println!("  {} ({:?}) tel:{}", p.nama, p.peranan, tel);
    }
}
```

---

# BAB 10: Mini Project — Sistem Pengurusan Staf 👥

```rust
use std::collections::HashMap;

// ─── Enums ────────────────────────────────────────────────────

#[derive(Debug, Clone, PartialEq)]
enum Bahagian {
    ICT,
    Kewangan,
    Pengurusan,
    Pertanian,
    HR,
}

impl Bahagian {
    fn nama(&self) -> &str {
        match self {
            Bahagian::ICT        => "ICT",
            Bahagian::Kewangan   => "Kewangan",
            Bahagian::Pengurusan => "Pengurusan",
            Bahagian::Pertanian  => "Pertanian",
            Bahagian::HR         => "HR",
        }
    }
}

#[derive(Debug, Clone, PartialEq)]
enum GredGaji {
    Gred19,
    Gred22,
    Gred29,
    Gred41,
    Gred44,
    Gred48,
}

impl GredGaji {
    fn gaji_pokok(&self) -> f64 {
        match self {
            GredGaji::Gred19 => 1200.0,
            GredGaji::Gred22 => 1500.0,
            GredGaji::Gred29 => 2000.0,
            GredGaji::Gred41 => 3000.0,
            GredGaji::Gred44 => 4000.0,
            GredGaji::Gred48 => 6000.0,
        }
    }
}

#[derive(Debug, Clone, PartialEq)]
enum StatusKhidmat {
    Tetap,
    Kontrak { tamat: String },
    Percubaan { tempoh_bulan: u8 },
    Bersara,
    Berhenti { tarikh: String },
}

// ─── Struct ───────────────────────────────────────────────────

#[derive(Debug, Clone)]
struct Staf {
    no_pekerja: String,
    nama:       String,
    bahagian:   Bahagian,
    gred:       GredGaji,
    status:     StatusKhidmat,
    emel:       Option<String>,
    supervisor: Option<String>, // no_pekerja supervisor
}

impl Staf {
    fn baru(
        no: &str,
        nama: &str,
        bahagian: Bahagian,
        gred: GredGaji,
    ) -> Self {
        Staf {
            no_pekerja: no.into(),
            nama:       nama.into(),
            bahagian,
            gred,
            status:     StatusKhidmat::Tetap,
            emel:       None,
            supervisor: None,
        }
    }

    fn dengan_emel(mut self, emel: &str) -> Self {
        self.emel = Some(emel.into());
        self
    }

    fn dengan_supervisor(mut self, no: &str) -> Self {
        self.supervisor = Some(no.into());
        self
    }

    fn dengan_status(mut self, status: StatusKhidmat) -> Self {
        self.status = status;
        self
    }

    fn gaji_bersih(&self) -> f64 {
        let pokok = self.gred.gaji_pokok();
        // COLA 2%
        let cola = pokok * 0.02;
        // Elaun perumahan bergantung gred
        let perumahan = match &self.gred {
            GredGaji::Gred41 | GredGaji::Gred44 | GredGaji::Gred48 => 300.0,
            _ => 150.0,
        };
        pokok + cola + perumahan
    }

    fn adalah_aktif(&self) -> bool {
        !matches!(&self.status,
            StatusKhidmat::Bersara | StatusKhidmat::Berhenti { .. }
        )
    }

    fn ringkasan(&self) -> String {
        let status_str = match &self.status {
            StatusKhidmat::Tetap                  => "Tetap".to_string(),
            StatusKhidmat::Kontrak { tamat }       => format!("Kontrak hingga {}", tamat),
            StatusKhidmat::Percubaan { tempoh_bulan } => format!("Percubaan {}bln", tempoh_bulan),
            StatusKhidmat::Bersara                => "Bersara".to_string(),
            StatusKhidmat::Berhenti { tarikh }     => format!("Berhenti {}", tarikh),
        };
        format!(
            "[{}] {} | {} | Gred:{:?} | RM{:.2} | {}",
            self.no_pekerja,
            self.nama,
            self.bahagian.nama(),
            self.gred,
            self.gaji_bersih(),
            status_str
        )
    }
}

// ─── Sistem ───────────────────────────────────────────────────

#[derive(Debug)]
enum StafError {
    StafTidakWujud(String),
    NoPekerjaaDuplikat(String),
    TidakBolehUbah { sebab: String },
}

impl std::fmt::Display for StafError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            StafError::StafTidakWujud(no)        => write!(f, "Staf '{}' tidak dijumpai", no),
            StafError::NoPekerjaaDuplikat(no)    => write!(f, "No pekerja '{}' sudah wujud", no),
            StafError::TidakBolehUbah { sebab } => write!(f, "Tidak boleh ubah: {}", sebab),
        }
    }
}

struct SistemStaf {
    staf: HashMap<String, Staf>,
}

impl SistemStaf {
    fn baru() -> Self {
        SistemStaf { staf: HashMap::new() }
    }

    fn tambah(&mut self, s: Staf) -> Result<(), StafError> {
        if self.staf.contains_key(&s.no_pekerja) {
            return Err(StafError::NoPekerjaaDuplikat(s.no_pekerja.clone()));
        }
        println!("  ✔ Tambah staf: {}", s.nama);
        self.staf.insert(s.no_pekerja.clone(), s);
        Ok(())
    }

    fn cari(&self, no: &str) -> Option<&Staf> {
        self.staf.get(no)
    }

    fn naik_gred(&mut self, no: &str, gred_baru: GredGaji) -> Result<(), StafError> {
        match self.staf.get_mut(no) {
            None => Err(StafError::StafTidakWujud(no.into())),
            Some(s) if !s.adalah_aktif() => Err(StafError::TidakBolehUbah {
                sebab: "Staf tidak aktif".into(),
            }),
            Some(s) => {
                println!("  ✔ {} naik gred: {:?} → {:?}", s.nama, s.gred, gred_baru);
                s.gred = gred_baru;
                Ok(())
            }
        }
    }

    fn senarai_mengikut_bahagian(&self) -> HashMap<&str, Vec<&Staf>> {
        let mut hasil: HashMap<&str, Vec<&Staf>> = HashMap::new();
        for s in self.staf.values() {
            hasil.entry(s.bahagian.nama()).or_default().push(s);
        }
        hasil
    }

    fn jumlah_gaji_bulanan(&self) -> f64 {
        self.staf.values()
            .filter(|s| s.adalah_aktif())
            .map(|s| s.gaji_bersih())
            .sum()
    }

    fn laporan(&self) {
        println!("\n{'═'*65}");
        println!("{:^65}", "LAPORAN SISTEM PENGURUSAN STAF KADA");
        println!("{'═'*65}");

        let mengikut_bahagian = self.senarai_mengikut_bahagian();
        let mut bahagian: Vec<&&str> = mengikut_bahagian.keys().collect();
        bahagian.sort();

        for bah in bahagian {
            let senarai = &mengikut_bahagian[bah];
            println!("\n  📁 Bahagian {} ({} staf):", bah, senarai.len());
            let mut aktif: Vec<&&Staf> = senarai.iter()
                .filter(|s| s.adalah_aktif())
                .collect();
            aktif.sort_by(|a, b| a.nama.cmp(&b.nama));

            for s in aktif {
                println!("    {}", s.ringkasan());
                if let Some(ref emel) = s.emel {
                    println!("       📧 {}", emel);
                }
                if let Some(ref sv) = s.supervisor {
                    if let Some(sv_staf) = self.cari(sv) {
                        println!("       👤 Penyelia: {}", sv_staf.nama);
                    }
                }
            }
        }

        println!("\n{'─'*65}");
        println!("  Jumlah staf aktif: {}",
            self.staf.values().filter(|s| s.adalah_aktif()).count());
        println!("  Jumlah gaji bulanan: RM{:.2}", self.jumlah_gaji_bulanan());
        println!("{'═'*65}");
    }
}

// ─── Main ──────────────────────────────────────────────────────

fn main() {
    let mut sistem = SistemStaf::baru();

    println!("=== Tambah Staf ===");

    let staf_list = vec![
        Staf::baru("KADA001", "Ali Ahmad",     Bahagian::ICT,        GredGaji::Gred44)
            .dengan_emel("ali@kada.gov.my")
            .dengan_status(StatusKhidmat::Tetap),

        Staf::baru("KADA002", "Siti Hawa",     Bahagian::ICT,        GredGaji::Gred41)
            .dengan_emel("siti@kada.gov.my")
            .dengan_supervisor("KADA001"),

        Staf::baru("KADA003", "Amin Razak",    Bahagian::Kewangan,   GredGaji::Gred44)
            .dengan_emel("amin@kada.gov.my"),

        Staf::baru("KADA004", "Zara Lina",     Bahagian::HR,         GredGaji::Gred29)
            .dengan_status(StatusKhidmat::Kontrak {
                tamat: "2025-06-30".into()
            }),

        Staf::baru("KADA005", "Razi Malik",    Bahagian::Pertanian,  GredGaji::Gred22)
            .dengan_status(StatusKhidmat::Percubaan { tempoh_bulan: 3 }),

        Staf::baru("KADA006", "Nora Hashim",   Bahagian::Pengurusan, GredGaji::Gred48)
            .dengan_emel("nora@kada.gov.my"),
    ];

    for s in staf_list {
        if let Err(e) = sistem.tambah(s) {
            println!("  ✘ {}", e);
        }
    }

    // Cuba tambah no pekerja yang sama
    let duplikat = Staf::baru("KADA001", "Penipu", Bahagian::ICT, GredGaji::Gred19);
    match sistem.tambah(duplikat) {
        Ok(())  => println!("  ✔ OK"),
        Err(e) => println!("  ✘ {}", e),
    }

    println!("\n=== Naik Gred ===");

    match sistem.naik_gred("KADA002", GredGaji::Gred44) {
        Ok(())  => {},
        Err(e) => println!("  ✘ {}", e),
    }

    match sistem.naik_gred("KADA999", GredGaji::Gred41) {
        Ok(())  => {},
        Err(e) => println!("  ✘ {}", e),
    }

    println!("\n=== Cari Staf ===");

    match sistem.cari("KADA003") {
        Some(s) => println!("  Jumpa: {}", s.ringkasan()),
        None    => println!("  Tidak jumpa"),
    }

    println!("  KADA999: {:?}", sistem.cari("KADA999").map(|s| &s.nama));

    sistem.laporan();
}
```

---

# 📋 Rujukan Pantas — Struct & Enum Cheat Sheet

## Struct

```rust
// Define
struct Nama { field: Type, .. }

// Buat
let s = Nama { field: val, .. };
let mut s = Nama { .. };  // mutable

// Update syntax
let s2 = Nama { field: val, ..s1 };  // baki dari s1

// Tuple struct
struct Wrapper(Type);
let Wrapper(val) = wrapper;  // destructure

// Methods
impl Nama {
    fn baru(args) -> Self { .. }    // constructor
    fn method(&self) { .. }         // borrow
    fn ubah(&mut self) { .. }       // mutable borrow
    fn consume(self) -> T { .. }    // take ownership
}
```

## Enum

```rust
// Define
enum Nama {
    UnitVariant,
    TupleVariant(Type1, Type2),
    StructVariant { field: Type },
}

// Buat
let e = Nama::UnitVariant;
let e = Nama::TupleVariant(val1, val2);
let e = Nama::StructVariant { field: val };

// match (MESTI exhaustive)
match e {
    Nama::UnitVariant           => ..,
    Nama::TupleVariant(a, b)    => ..,
    Nama::StructVariant { field }=> ..,
}
```

## Option<T>

```rust
Some(nilai)             // ada nilai
None                    // tiada nilai

opt.is_some()           // bool
opt.is_none()           // bool
opt.unwrap()            // T atau PANIC
opt.unwrap_or(default)  // T atau default
opt.map(|v| ...)        // transform Some
opt.and_then(|v| ...)   // chain Option
opt.filter(|v| ...)     // None kalau syarat gagal
opt?                    // return None kalau None (dalam fungsi)

if let Some(v) = opt { .. }
```

## Result<T, E>

```rust
Ok(nilai)               // berjaya
Err(ralat)              // gagal

res.is_ok()             // bool
res.is_err()            // bool
res.unwrap()            // T atau PANIC
res.expect("msg")       // T atau PANIC dengan mesej
res.unwrap_or(default)  // T atau default
res.map(|v| ...)        // transform Ok
res.map_err(|e| ...)    // transform Err
res?                    // return Err kalau Err (dalam fungsi)

match res { Ok(v) => .., Err(e) => .. }
if let Ok(v) = res { .. }
```

## Bila Guna Apa

```
Struct   → Kumpulkan data berkaitan
Enum     → Nilai yang ada beberapa kemungkinan bentuk
Option   → "Ada atau tiada?" (ganti null)
Result   → "Berjaya atau gagal?" (ganti exception)
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **Kalkulator Postfix** — guna `enum Token` untuk `Number(f64)`, `Op(char)`, parse dan evaluate
2. **Mini Bank** — `struct Akaun`, `enum TransaksiType`, `Result` untuk operasi
3. **Todo App** — `struct Todo`, `enum StatusTodo`, `Option` untuk tarikh siap
4. **Inventory Sistem** — `struct Produk`, `enum Kategori`, `Result` untuk operasi CRUD
5. **Pokok Binary** — `enum Pokok { Daun(i32), Nod(Box<Pokok>, i32, Box<Pokok>) }`

---

*Struct & Enum in Rust — dari struct asas hingga sistem pengurusan staf lengkap. Selamat belajar!* 🦀
