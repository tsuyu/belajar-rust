# 🦀 Idiomatic Rust — Tulis Kod Cara Rust

> "Kod yang compile bukan bermakna kod yang baik."
> Panduan menulis Rust yang betul, bersih, dan Rusty.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu "Idiomatic"?

```
Idiomatic = cara penulisan yang dianggap "betul" dan "natural"
            oleh komuniti Rust. Bukan sekadar compile — tapi
            kod yang programmer Rust lain boleh faham, jangka,
            dan appreciate.

Macam bahasa manusia:
  "Saya lapar sangat" ← betul tapi pelik
  "Saya sangat lapar" ← idiomatic BM

Dalam Rust:
  for i in 0..v.len() { println!("{}", v[i]); }  ← compile, tapi...
  for item in &v { println!("{}", item); }        ← idiomatic!

Kenapa penting?
  ✔ Kod lebih mudah dibaca oleh programmer Rust lain
  ✔ Compiler optimization lebih baik
  ✔ Kurang bug (idioms wujud sebab best practice)
  ✔ Code review lebih mudah
  ✔ Kerjasama dalam pasukan lebih lancar
```

---

## Peta Pembelajaran

```
Bahagian 1  → Naming Conventions
Bahagian 2  → Types & Data Modelling
Bahagian 3  → Iterator Patterns
Bahagian 4  → Error Handling yang Betul
Bahagian 5  → Ownership & Borrowing Idioms
Bahagian 6  → Struct & Enum Idioms
Bahagian 7  → Function Design
Bahagian 8  → Traits Idioms
Bahagian 9  → Anti-Patterns — Jangan Buat Ini
Bahagian 10 → Mini Project: Refactor ke Idiomatic
```

---

# BAHAGIAN 1: Naming Conventions 📝

## Panduan Penamaan Rust

```
Perkara          Konvensyen        Contoh
─────────────────────────────────────────────────────────────
Variables        snake_case        nama_pengguna, harga_item
Functions        snake_case        kira_purata(), tambah_item()
Methods          snake_case        .to_string(), .push_str()
Modules          snake_case        mod auth_service;
Crates           snake_case        my_crate, kada_core
Struct           PascalCase        PenggunaSistem, KonfigApp
Enum             PascalCase        StatusPesanan, JenisRalat
Enum variants    PascalCase        StatusPesanan::Baru
Trait            PascalCase        Serialize, Tambahkan
Type aliases     PascalCase        type KoordinatGps = (f64, f64);
Const            SCREAMING_SNAKE   const MAX_SAMBUNGAN: u32 = 100;
Static           SCREAMING_SNAKE   static VERSI: &str = "1.0";
Lifetime         lowercase 'a     'a, 'static, 'lifetime_panjang
Generic param    PascalCase/T      T, E, Item, Key, Val
```

```rust
// ✔ Idiomatic naming
const MAX_HAD: usize = 1000;
static MESEJ_LALAI: &str = "Selamat datang";

struct PekerjaData {
    nama_penuh: String,
    no_pekerja: String,
}

enum StatusKehadiran {
    HadirTepat,
    HadirLewat { minit: u32 },
    Tidak { sebab: String },
}

trait BolehDisimpan {
    fn simpan(&self) -> Result<(), RalatDb>;
}

fn kira_gaji_bersih(gaji_kasar: f64, kadar_cukai: f64) -> f64 {
    gaji_kasar * (1.0 - kadar_cukai)
}

// Generic dengan nama bermakna
fn cari_maksimum<T: PartialOrd>(senarai: &[T]) -> Option<&T> {
    senarai.iter().reduce(|a, b| if a > b { a } else { b })
}
```

---

## Constructor Pattern — `new()` dan Builder

```rust
// ✔ Idiomatic: associated function "new" sebagai constructor
struct Sambungan {
    host:  String,
    port:  u16,
    aktif: bool,
}

impl Sambungan {
    // Constructor tanpa argument
    pub fn new() -> Self {
        Sambungan {
            host:  "localhost".into(),
            port:  8080,
            aktif: false,
        }
    }

    // Constructor dengan argument
    pub fn dengan(host: &str, port: u16) -> Self {
        Sambungan {
            host:  host.into(),
            port,
            aktif: false,
        }
    }
}

// ✔ Idiomatic: impl Default untuk default values
impl Default for Sambungan {
    fn default() -> Self {
        Self::new()
    }
}

// ✔ Derive Default bila boleh (semua field implement Default)
#[derive(Debug, Default)]
struct Tetapan {
    debug:  bool,       // default: false
    log:    bool,       // default: false
    kadar:  f64,        // default: 0.0
    nama:   String,     // default: ""
}

fn main() {
    let s = Sambungan::new();
    let s2 = Sambungan::dengan("server.com", 5432);
    let t = Tetapan::default();
    let t2 = Tetapan { debug: true, ..Tetapan::default() };
}
```

---

## 🧠 Brain Teaser #1

Yang mana idiomatic dan kenapa?

```rust
// A
fn dapatkan_panjang(s: &String) -> usize {
    s.len()
}

// B
fn dapatkan_panjang(s: &str) -> usize {
    s.len()
}

// C
fn adalah_kosong(s: String) -> bool {
    s.is_empty()
}

// D
fn adalah_kosong(s: &str) -> bool {
    s.is_empty()
}
```

<details>
<summary>👀 Jawapan</summary>

**B dan D adalah idiomatic.**

**A — tidak idiomatic:**
- `&String` adalah terlalu spesifik. `&str` boleh terima `&String` (auto-deref) DAN string literal.
- Rule of thumb: **Input strings → guna `&str`**

**B — idiomatic ✔:**
- `&str` lebih flexible: boleh terima `&String`, `&str`, `String.as_str()`
- Tiada overhead allocation

**C — tidak idiomatic:**
- `String` dalam parameter = caller kena MOVE atau CLONE string
- Fungsi hanya baca string, kenapa ambil ownership?

**D — idiomatic ✔:**
- `&str` = pinjam sahaja, tiada move/clone diperlukan
- Caller kekal punya string selepas panggil fungsi

**Prinsip:** Kalau fungsi hanya BACA string → `&str`. Kalau fungsi perlu SIMPAN atau UBAH → `String`.
</details>

---

# BAHAGIAN 2: Types & Data Modelling 🏗️

## Guna Type System untuk Express Intent

```rust
// ❌ Bukan idiomatic — guna primitive untuk semua
fn hantar_emel(alamat: String, subjek: String, kandungan: String) {
    // Bila ada 3 String params, senang tersilap urutan!
    // hantar_emel(kandungan, alamat, subjek) — compiler tidak tangkap!
}

// ✔ Idiomatic — buat types yang bermakna
struct Emel(String);     // newtype untuk enforce validity
struct Subjek(String);
struct Kandungan(String);

fn hantar_emel(alamat: &Emel, subjek: &Subjek, kandungan: &Kandungan) {
    // Sekarang silap urutan = compiler error!
}

// Atau guna struct untuk group parameters
struct ParameterEmel {
    alamat:    Emel,
    subjek:    Subjek,
    kandungan: Kandungan,
}
```

---

## Enum untuk State yang Jelas

```rust
// ❌ Bukan idiomatic — guna bool/int untuk state
struct Pengguna {
    nama:        String,
    aktif:       bool,
    dah_verify:  bool,
    diblok:      bool,
    // Boleh ada state yang tidak masuk akal:
    // aktif: true, diblok: true ← ini apa maknanya?!
}

// ✔ Idiomatic — guna enum untuk state yang exclusive
#[derive(Debug, PartialEq)]
enum StatusPengguna {
    Aktif,
    BelumVerify { token: String },
    Diblok { sebab: String, sehingga: Option<String> },
    Padam,
}

struct Pengguna {
    nama:   String,
    emel:   String,
    status: StatusPengguna,  // hanya satu status pada satu masa!
}

impl Pengguna {
    fn adalah_aktif(&self) -> bool {
        matches!(self.status, StatusPengguna::Aktif)
    }

    fn boleh_login(&self) -> bool {
        self.adalah_aktif()
    }
}
```

---

## Make Illegal States Unrepresentable

```rust
// Prinsip paling penting dalam Rust type design:
// "Make illegal states unrepresentable"

// ❌ Boleh buat state yang tidak masuk akal:
struct TitikGPS {
    lat:    Option<f64>,  // boleh None?
    lon:    Option<f64>,  // boleh None?
    // Boleh ada: lat = Some(3.14), lon = None ← GPS setengah?!
}

// ✔ Sama ada ada KEDUA-DUA atau tiada langsung
enum LokasiGPS {
    Diketahui { lat: f64, lon: f64 },
    Tidak,
}

// Atau:
struct KoordinatGPS {
    lat: f64,  // mesti ada kedua-dua!
    lon: f64,
}
type LokasiOpsyen = Option<KoordinatGPS>;

// ─────────────────────────────────────────────────────────────

// ❌ Nama dalam pair boleh tidak konsisten
struct Julat {
    mula:  i32,
    tamat: i32,
    // Boleh ada mula > tamat — itu valid secara type tapi tidak masuk akal!
}

// ✔ Enforce invariant dalam constructor
struct JulatSah {
    mula:  i32,
    tamat: i32,
}

impl JulatSah {
    pub fn baru(mula: i32, tamat: i32) -> Option<Self> {
        if mula <= tamat {
            Some(JulatSah { mula, tamat })
        } else {
            None  // Julat tidak sah!
        }
    }

    pub fn panjang(&self) -> i32 {
        self.tamat - self.mula
    }
}
```

---

# BAHAGIAN 3: Iterator Patterns 🔄

## Iterator adalah Cara Rust

```rust
fn main() {
    let nombor = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // ❌ Bukan idiomatic — C-style loop
    let mut jumlah_genap = 0;
    for i in 0..nombor.len() {
        if nombor[i] % 2 == 0 {
            jumlah_genap += nombor[i];
        }
    }

    // ✔ Idiomatic — iterator chain
    let jumlah_genap: i32 = nombor.iter()
        .filter(|&&n| n % 2 == 0)
        .sum();

    println!("{}", jumlah_genap); // 30

    // ─── Transformasi biasa ────────────────────────────────────

    // Darabkan semua dengan 2
    let dua_kali: Vec<i32> = nombor.iter()
        .map(|&n| n * 2)
        .collect();

    // Filter dan transform serentak
    let kuasa_ganjil: Vec<i32> = nombor.iter()
        .filter(|&&n| n % 2 != 0)
        .map(|&n| n * n)
        .collect();

    // Kumpul ke HashMap
    use std::collections::HashMap;
    let peta: HashMap<i32, i32> = nombor.iter()
        .map(|&n| (n, n * n))
        .collect();

    // ─── Reduce / Fold ──────────────────────────────────────────

    let hasil = nombor.iter().fold(0, |acc, &n| acc + n);
    println!("Fold: {}", hasil); // 55

    let faktorial_5: u64 = (1..=5).product();
    println!("5! = {}", faktorial_5); // 120
}
```

---

## Iterator Methods yang Wajib Tahu

```rust
fn main() {
    let data = vec![
        ("Ali",  85),
        ("Siti", 92),
        ("Amin", 78),
        ("Zara", 95),
        ("Razi", 60),
    ];

    // any / all — boolean check
    let ada_gagal = data.iter().any(|&(_, markah)| markah < 70);
    let semua_lulus = data.iter().all(|&(_, markah)| markah >= 60);
    println!("Ada gagal: {}, Semua lulus: {}", ada_gagal, semua_lulus);

    // find / position
    let cemerlang = data.iter().find(|&&(_, m)| m >= 90);
    println!("{:?}", cemerlang); // Some(("Siti", 92))

    let pos = data.iter().position(|&(nama, _)| nama == "Amin");
    println!("Amin pada: {:?}", pos); // Some(2)

    // max_by_key / min_by_key
    let terbaik = data.iter().max_by_key(|&&(_, m)| m);
    let terburuk = data.iter().min_by_key(|&&(_, m)| m);
    println!("Terbaik: {:?}", terbaik);  // Some(("Zara", 95))
    println!("Terburuk: {:?}", terburuk); // Some(("Razi", 60))

    // enumerate — tambah index
    for (i, (nama, markah)) in data.iter().enumerate() {
        println!("[{}] {}: {}", i + 1, nama, markah);
    }

    // zip — gabung dua iterator
    let gred = vec!['A', 'A', 'B', 'A', 'C'];
    for ((nama, markah), gred) in data.iter().zip(gred.iter()) {
        println!("{}: {} → {}", nama, markah, gred);
    }

    // flat_map — flatten nested
    let ayat = vec!["hello world", "foo bar baz"];
    let perkataan: Vec<&str> = ayat.iter()
        .flat_map(|s| s.split_whitespace())
        .collect();
    println!("{:?}", perkataan); // ["hello", "world", "foo", "bar", "baz"]

    // chain — gabung dua iterator
    let a = vec![1, 2, 3];
    let b = vec![4, 5, 6];
    let c: Vec<i32> = a.iter().chain(b.iter()).copied().collect();
    println!("{:?}", c); // [1, 2, 3, 4, 5, 6]

    // windows / chunks
    let nombor = vec![1, 2, 3, 4, 5];
    for w in nombor.windows(3) {
        println!("Window: {:?}", w); // [1,2,3], [2,3,4], [3,4,5]
    }

    for chunk in nombor.chunks(2) {
        println!("Chunk: {:?}", chunk); // [1,2], [3,4], [5]
    }
}
```

---

## Implement Iterator untuk Custom Type

```rust
// Implement Iterator sendiri untuk struct
struct KiraMenurun {
    nilai: u32,
}

impl KiraMenurun {
    fn baru(mula: u32) -> Self { KiraMenurun { nilai: mula } }
}

impl Iterator for KiraMenurun {
    type Item = u32;

    fn next(&mut self) -> Option<u32> {
        if self.nilai == 0 {
            None
        } else {
            let semasa = self.nilai;
            self.nilai -= 1;
            Some(semasa)
        }
    }
}

fn main() {
    // Dapat SEMUA iterator methods secara automatik!
    let kiraan = KiraMenurun::baru(5);

    // for loop
    for n in KiraMenurun::baru(5) {
        print!("{} ", n); // 5 4 3 2 1
    }
    println!();

    // collect
    let v: Vec<u32> = KiraMenurun::baru(5).collect();
    println!("{:?}", v); // [5, 4, 3, 2, 1]

    // sum, product
    let hasil: u32 = KiraMenurun::baru(5).sum();
    println!("Jumlah: {}", hasil); // 15

    // filter, map, chain — semua berfungsi!
    let genap: Vec<u32> = KiraMenurun::baru(10)
        .filter(|n| n % 2 == 0)
        .collect();
    println!("{:?}", genap); // [10, 8, 6, 4, 2]
}
```

---

## 🧠 Brain Teaser #2

Refactor ini ke gaya idiomatic:

```rust
fn statistik(data: &Vec<i32>) -> (i32, i32, f64) {
    let mut min = data[0];
    let mut max = data[0];
    let mut jumlah = 0;

    for i in 0..data.len() {
        if data[i] < min { min = data[i]; }
        if data[i] > max { max = data[i]; }
        jumlah += data[i];
    }

    let purata = jumlah as f64 / data.len() as f64;
    (min, max, purata)
}
```

<details>
<summary>👀 Jawapan</summary>

```rust
// Idiomatic version:
fn statistik(data: &[i32]) -> Option<(i32, i32, f64)> {
    // &[i32] bukan &Vec<i32> — lebih flexible, terima array dan slice
    // Return Option — handle kes kosong!

    let &min = data.iter().min()?;  // ? → return None jika kosong
    let &max = data.iter().max()?;
    let purata = data.iter().sum::<i32>() as f64 / data.len() as f64;

    Some((min, max, purata))
}

// Atau dengan fold untuk single pass:
fn statistik_v2(data: &[i32]) -> Option<(i32, i32, f64)> {
    if data.is_empty() { return None; }

    let (min, max, jumlah) = data.iter().fold(
        (data[0], data[0], 0i64),
        |(min, max, sum), &n| (min.min(n), max.max(n), sum + n as i64)
    );

    Some((min, max, jumlah as f64 / data.len() as f64))
}

fn main() {
    let data = vec![3, 1, 4, 1, 5, 9, 2, 6];
    println!("{:?}", statistik(&data)); // Some((1, 9, 3.875))
    println!("{:?}", statistik(&[]));   // None
}
```

**Perbaikan:**
1. `&[i32]` bukannya `&Vec<i32>` — lebih flexible
2. Return `Option` — handle empty slice
3. Guna `.iter().min()` dan `.max()` — lebih ringkas dan jelas
4. Tiada manual loop — iterator lebih expressive
</details>

---

# BAHAGIAN 4: Error Handling yang Betul ⚠️

## ? Operator adalah Standard

```rust
use std::fs;
use std::num::ParseIntError;

// ❌ Bukan idiomatic — unwrap dalam production code
fn baca_konfigurasi(path: &str) -> u16 {
    let kandungan = fs::read_to_string(path).unwrap(); // panic!
    kandungan.trim().parse::<u16>().unwrap()           // panic!
}

// ✔ Idiomatic — propagate errors dengan ?
fn baca_konfigurasi(path: &str) -> Result<u16, Box<dyn std::error::Error>> {
    let kandungan = fs::read_to_string(path)?;
    let port = kandungan.trim().parse::<u16>()?;
    Ok(port)
}

// ✔ Lebih baik — custom error type
#[derive(Debug, thiserror::Error)]
enum KonfigError {
    #[error("Gagal baca fail: {0}")]
    Fail(#[from] std::io::Error),
    #[error("Port tidak sah: {0}")]
    ParsePort(#[from] ParseIntError),
    #[error("Port luar julat: {port} (had: 1-65535)")]
    LuarJulat { port: u16 },
}

fn baca_konfigurasi_baik(path: &str) -> Result<u16, KonfigError> {
    let kandungan = fs::read_to_string(path)?;   // auto: io::Error → KonfigError::Fail
    let port: u16 = kandungan.trim().parse()?;   // auto: ParseIntError → KonfigError::ParsePort

    if port == 0 {
        return Err(KonfigError::LuarJulat { port });
    }

    Ok(port)
}
```

---

## Guna anyhow untuk Application Code

```rust
// ✔ Idiomatic dalam application code (bukan library)
use anyhow::{Context, Result, bail, ensure};

fn proses_data(path: &str) -> Result<Vec<i32>> {
    let kandungan = std::fs::read_to_string(path)
        .with_context(|| format!("Gagal baca '{}'", path))?;

    let nombor: Vec<i32> = kandungan
        .lines()
        .filter(|l| !l.is_empty())
        .map(|l| l.trim().parse::<i32>()
             .with_context(|| format!("Gagal parse baris: '{}'", l)))
        .collect::<Result<_>>()?; // collect Vec<Result<T,E>> → Result<Vec<T>,E>

    ensure!(!nombor.is_empty(), "Fail tidak ada nombor");
    ensure!(nombor.iter().all(|&n| n > 0), "Semua nombor mesti positif");

    Ok(nombor)
}

fn main() -> Result<()> {
    let hasil = proses_data("data.txt")?;
    println!("Nombor: {:?}", hasil);
    Ok(())
}
```

---

## Match Exhaustive dengan Enum Error

```rust
#[derive(Debug)]
enum ApiError {
    TidakDijumpai(String),
    TidakDibenarkan,
    RalatServer(String),
    Rangkaian(String),
}

fn tangani_error(e: ApiError) {
    match e {
        // ✔ Idiomatic — handle setiap kes explicitly
        ApiError::TidakDijumpai(resource) => {
            println!("404: {} tidak dijumpai", resource);
        }
        ApiError::TidakDibenarkan => {
            println!("401: Sila log masuk");
        }
        ApiError::RalatServer(mesej) => {
            eprintln!("500: Ralat server — {}", mesej);
        }
        ApiError::Rangkaian(mesej) => {
            eprintln!("Rangkaian: {}", mesej);
            // boleh cuba lagi
        }
    }
    // Compiler akan ERROR jika ada kes yang tertinggal!
}
```

---

# BAHAGIAN 5: Ownership & Borrowing Idioms 🔗

## Prefer Borrowing Atas Clone

```rust
// ❌ Clone yang tidak perlu
fn papar_nama(pengguna: Pengguna) {  // ambil ownership!
    println!("{}", pengguna.nama);
}

fn main() {
    let p = Pengguna { nama: "Ali".into() };
    papar_nama(p.clone()); // clone!
    papar_nama(p.clone()); // clone lagi!
    println!("{}", p.nama); // p masih valid
}

// ✔ Idiomatic — borrow
fn papar_nama(pengguna: &Pengguna) { // pinjam sahaja
    println!("{}", pengguna.nama);
}

fn main() {
    let p = Pengguna { nama: "Ali".into() };
    papar_nama(&p);  // tiada clone!
    papar_nama(&p);  // tiada clone!
    println!("{}", p.nama); // p masih valid
}

// ─────────────────────────────────────────────────────────────

// ❌ &String dalam parameter (terlalu restrictive)
fn hitung_vokal_buruk(s: &String) -> usize {
    s.chars().filter(|c| "aeiou".contains(*c)).count()
}

// ✔ &str lebih flexible
fn hitung_vokal(s: &str) -> usize {
    s.chars().filter(|c| "aeiou".contains(*c)).count()
}

// &str boleh terima &String, string literal, slice
let nama = String::from("Ali Ahmad");
hitung_vokal(&nama);         // ✔ dari String
hitung_vokal("Siti Hawa");   // ✔ literal
hitung_vokal(&nama[0..3]);   // ✔ slice
```

---

## Into / From untuk Flexible Parameters

```rust
// ✔ Idiomatic — guna Into<T> untuk flexible input
struct Mesej {
    kandungan: String,
}

impl Mesej {
    // ❌ Terlalu restrictive:
    // pub fn baru(kandungan: String) -> Self { ... }

    // ✔ Terima &str, String, apa-apa yang boleh jadi String
    pub fn baru(kandungan: impl Into<String>) -> Self {
        Mesej { kandungan: kandungan.into() }
    }
}

// Sekarang boleh panggil dengan pelbagai cara:
Mesej::baru("Hello!");             // &str → String
Mesej::baru(String::from("Hi!"));  // String → String
Mesej::baru(format!("Halo {}!", "dunia")); // String → String

// ─────────────────────────────────────────────────────────────

// AsRef<str> untuk read-only string params
fn cetak_panjang(s: impl AsRef<str>) {
    let s = s.as_ref();
    println!("'{}' ada {} aksara", s, s.len());
}

cetak_panjang("literal");                    // ✔
cetak_panjang(String::from("owned"));        // ✔
cetak_panjang(&String::from("reference"));   // ✔
```

---

## Entry API untuk HashMap

```rust
use std::collections::HashMap;

fn main() {
    let teks = "hello world hello rust hello world";

    // ❌ Bukan idiomatic — dua lookup
    let mut kiraan: HashMap<&str, u32> = HashMap::new();
    for kata in teks.split_whitespace() {
        if kiraan.contains_key(kata) {
            *kiraan.get_mut(kata).unwrap() += 1;
        } else {
            kiraan.insert(kata, 1);
        }
    }

    // ✔ Idiomatic — entry API, satu lookup
    let mut kiraan2: HashMap<&str, u32> = HashMap::new();
    for kata in teks.split_whitespace() {
        *kiraan2.entry(kata).or_insert(0) += 1;
    }

    // ✔ Lebih cantik dengan or_default
    let mut kiraan3: HashMap<&str, u32> = HashMap::new();
    for kata in teks.split_whitespace() {
        *kiraan3.entry(kata).or_default() += 1;
    }

    println!("{:?}", kiraan3);
    // {"hello": 3, "world": 2, "rust": 1}

    // Entry dengan modify yang complex
    let mut data: HashMap<String, Vec<i32>> = HashMap::new();
    data.entry("senarai".to_string())
        .or_insert_with(Vec::new)
        .push(42);
    data.entry("senarai".to_string())
        .or_default()
        .push(99);
}
```

---

# BAHAGIAN 6: Struct & Enum Idioms 🏛️

## Builder Pattern yang Betul

```rust
// ✔ Idiomatic Builder Pattern
#[derive(Debug)]
struct PermintaanHttp {
    url:      String,
    kaedah:   String,
    header:   Vec<(String, String)>,
    badan:    Option<String>,
    tamat:    u32,
}

struct HttpBuilder {
    url:    String,
    kaedah: String,
    header: Vec<(String, String)>,
    badan:  Option<String>,
    tamat:  u32,
}

impl HttpBuilder {
    pub fn baru(url: impl Into<String>) -> Self {
        HttpBuilder {
            url:    url.into(),
            kaedah: "GET".into(),
            header: Vec::new(),
            badan:  None,
            tamat:  30,
        }
    }

    // Method chaining — return Self untuk chain
    pub fn kaedah(mut self, k: impl Into<String>) -> Self {
        self.kaedah = k.into();
        self
    }

    pub fn header(mut self, kunci: impl Into<String>, nilai: impl Into<String>) -> Self {
        self.header.push((kunci.into(), nilai.into()));
        self
    }

    pub fn badan(mut self, b: impl Into<String>) -> Self {
        self.badan = Some(b.into());
        self
    }

    pub fn tamat(mut self, saat: u32) -> Self {
        self.tamat = saat;
        self
    }

    pub fn bina(self) -> PermintaanHttp {
        PermintaanHttp {
            url:    self.url,
            kaedah: self.kaedah,
            header: self.header,
            badan:  self.badan,
            tamat:  self.tamat,
        }
    }
}

fn main() {
    let req = HttpBuilder::baru("https://api.kada.gov.my/pekerja")
        .kaedah("POST")
        .header("Content-Type", "application/json")
        .header("Authorization", "Bearer token123")
        .badan(r#"{"nama":"Ali","id":1001}"#)
        .tamat(60)
        .bina();

    println!("{:?}", req);
}
```

---

## Guna #[derive] Secara Bijak

```rust
// ✔ Idiomatic — derive standard traits bila sesuai

#[derive(
    Debug,      // untuk {:?} printing dan debugging
    Clone,      // untuk .clone()
    PartialEq,  // untuk ==
    Eq,         // untuk HashMap key (butuh PartialEq + Eq)
    Hash,       // untuk HashMap key (butuh Eq + Hash)
)]
struct PengecamPekerja {
    id:          u32,
    no_pekerja:  String,
}

// Copy HANYA untuk types kecil yang stack-only
#[derive(Debug, Clone, Copy, PartialEq)]
struct Koordinat {
    lat: f64,
    lon: f64,
}

// Default untuk konstruksi dengan nilai lalai
#[derive(Debug, Default)]
struct Statistik {
    kiraan:  u32,   // 0
    jumlah:  f64,   // 0.0
    minimum: f64,   // 0.0 (perlu Override!)
}

// Serde untuk serialization
#[derive(Debug, serde::Serialize, serde::Deserialize)]
#[serde(rename_all = "camelCase")]
struct DataApi {
    nama_penuh:  String,
    no_telefon:  Option<String>,
    tarikh_daftar: String,
}
```

---

# BAHAGIAN 7: Function Design 🔧

## Small, Focused Functions

```rust
// ❌ Bukan idiomatic — fungsi buat terlalu banyak perkara
fn proses_semua(data: &[String]) -> Result<HashMap<String, Vec<i32>>, String> {
    let mut hasil = HashMap::new();
    for item in data {
        let bahagian: Vec<&str> = item.split(':').collect();
        if bahagian.len() != 2 { return Err(format!("Format salah: {}", item)); }
        let kunci = bahagian[0].trim().to_string();
        let nilai: i32 = bahagian[1].trim().parse().map_err(|e| e.to_string())?;
        hasil.entry(kunci).or_insert_with(Vec::new).push(nilai);
    }
    Ok(hasil)
}

// ✔ Idiomatic — pecah kepada fungsi kecil dengan tanggungjawab jelas
fn parse_baris(baris: &str) -> Result<(&str, i32), String> {
    let (kunci, nilai_str) = baris.split_once(':')
        .ok_or_else(|| format!("Tiada ':' dalam baris: '{}'", baris))?;
    let nilai = nilai_str.trim()
        .parse::<i32>()
        .map_err(|e| format!("Nilai tidak sah '{}': {}", nilai_str.trim(), e))?;
    Ok((kunci.trim(), nilai))
}

fn kumpul_ke_map(
    baris_senarai: &[String]
) -> Result<HashMap<String, Vec<i32>>, String> {
    let mut hasil = HashMap::new();
    for baris in baris_senarai {
        let (kunci, nilai) = parse_baris(baris)?;
        hasil.entry(kunci.to_string()).or_default().push(nilai);
    }
    Ok(hasil)
}
```

---

## Return Type Conventions

```rust
// ✔ Return &str atau &T kalau boleh (tiada allocation)
impl Rekod {
    // Ambil reference ke data dalam struct
    pub fn nama(&self) -> &str { &self.nama }
    pub fn data(&self) -> &[u8] { &self.data }

    // Kalau perlu owned → baru return String/Vec
    pub fn nama_besar(&self) -> String { self.nama.to_uppercase() }

    // Optional data → Option<&T>
    pub fn metadata(&self) -> Option<&str> {
        self.metadata.as_deref()
    }

    // Mutable access → Option<&mut T>
    pub fn metadata_mut(&mut self) -> Option<&mut String> {
        self.metadata.as_mut()
    }
}

// ✔ Boolean predicates — nama dengan is_, has_, can_
impl Pengguna {
    pub fn is_aktif(&self) -> bool     { ... }
    pub fn has_emel(&self) -> bool     { self.emel.is_some() }
    pub fn can_edit(&self) -> bool     { ... }
    pub fn is_admin(&self) -> bool     { ... }
}
```

---

# BAHAGIAN 8: Traits Idioms 🎭

## Implement Display dan Debug

```rust
use std::fmt;

#[derive(Debug)]  // Debug dengan derive — untuk developer
struct Pekerja {
    nama: String,
    id:   u32,
}

// Display — untuk pengguna
impl fmt::Display for Pekerja {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{} (ID: {})", self.nama, self.id)
    }
}

// ✔ Guna std::error::Error untuk custom errors
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self { /* ... */ }
    }
}
impl std::error::Error for AppError {}

// ✔ Guna From untuk error conversion (enable ?)
impl From<std::io::Error> for AppError {
    fn from(e: std::io::Error) -> Self {
        AppError::Io(e)
    }
}
```

---

## Extension Traits — Tambah Methods ke External Types

```rust
// ✔ Idiomatic — buat extension traits untuk external types

trait StrExt {
    fn potong(&self, maks: usize) -> &str;
    fn adalah_nombor(&self) -> bool;
    fn ke_title_case(&self) -> String;
}

impl StrExt for str {
    fn potong(&self, maks: usize) -> &str {
        if self.len() <= maks {
            self
        } else {
            &self[..maks]
        }
    }

    fn adalah_nombor(&self) -> bool {
        !self.is_empty() && self.chars().all(|c| c.is_ascii_digit())
    }

    fn ke_title_case(&self) -> String {
        self.split_whitespace()
            .map(|kata| {
                let mut c = kata.chars();
                match c.next() {
                    None    => String::new(),
                    Some(f) => f.to_uppercase().to_string() + c.as_str(),
                }
            })
            .collect::<Vec<_>>()
            .join(" ")
    }
}

fn main() {
    println!("{}", "hello world ini panjang".potong(10)); // "hello worl"
    println!("{}", "12345".adalah_nombor());               // true
    println!("{}", "hello world".ke_title_case());         // "Hello World"
}
```

---

# BAHAGIAN 9: Anti-Patterns — Jangan Buat Ini 🚫

## Anti-Pattern #1: Unwrap Mana-Mana

```rust
// ❌ JANGAN — unwrap dalam production code
fn dapatkan_pengguna(id: u32) -> Pengguna {
    db.query(id).unwrap()        // panic bila tiada record!
    // Bila panic? Mana-mana masa tanpa mesej berguna.
}

// ✔ Handle properly
fn dapatkan_pengguna(id: u32) -> Result<Pengguna, DbError> {
    db.query(id)?.ok_or(DbError::TidakJumpai(id))
}

// ✔ Atau kalau betul-betul mesti ada (dengan justifikasi)
fn nama_fail_config() -> &'static str {
    // JUSTIFIKASI: compile-time constant, pasti valid
    std::env::var("CONFIG_FILE")
        .expect("CONFIG_FILE env var mesti ditetapkan")
        .leak() // intentional leak untuk 'static
}
```

---

## Anti-Pattern #2: Clone Mana-Mana

```rust
// ❌ Clone yang tidak perlu
fn jumlah_buruk(data: Vec<i32>) -> i32 {
    data.iter().sum()
}
fn main() {
    let v = vec![1, 2, 3];
    let j = jumlah_buruk(v.clone()); // clone!
    println!("{:?}", v); // kena clone sebab v dah moved!
}

// ✔ Guna slice
fn jumlah(data: &[i32]) -> i32 {
    data.iter().sum()
}
fn main() {
    let v = vec![1, 2, 3];
    let j = jumlah(&v);  // tiada clone!
    println!("{:?}", v); // v masih valid
}
```

---

## Anti-Pattern #3: Index Loop

```rust
// ❌ C-style index loop
for i in 0..v.len() {
    println!("{}", v[i]); // bounds check setiap akses!
}

// ✔ Iterator
for item in &v {
    println!("{}", item);
}

// ✔ Kalau perlu index juga
for (i, item) in v.iter().enumerate() {
    println!("[{}] {}", i, item);
}

// ❌ Manual accumulation
let mut total = 0;
for &n in &v { total += n; }

// ✔ Sum/fold
let total: i32 = v.iter().sum();
```

---

## Anti-Pattern #4: Nested Match yang Dalam

```rust
// ❌ Pyramid of doom
fn proses(opt: Option<Option<i32>>) -> String {
    match opt {
        Some(inner) => {
            match inner {
                Some(n) => {
                    if n > 0 {
                        format!("positif: {}", n)
                    } else {
                        "tidak positif".to_string()
                    }
                }
                None => "inner tiada".to_string(),
            }
        }
        None => "outer tiada".to_string(),
    }
}

// ✔ Flatten dan chain
fn proses(opt: Option<Option<i32>>) -> String {
    match opt.flatten() {
        Some(n) if n > 0 => format!("positif: {}", n),
        Some(_)          => "tidak positif".to_string(),
        None             => "tiada nilai".to_string(),
    }
}
```

---

## Anti-Pattern #5: Return Early Dari Happy Path

```rust
// ❌ Return Ok() scattered everywhere
fn validate(data: &FormData) -> Result<(), Vec<String>> {
    let mut ralat = Vec::new();

    if data.nama.is_empty() {
        ralat.push("Nama kosong".to_string());
    }
    if !data.emel.contains('@') {
        ralat.push("Emel tidak sah".to_string());
    }
    if data.umur < 18 {
        ralat.push("Perlu 18+".to_string());
    }

    if ralat.is_empty() {
        Ok(())
    } else {
        Err(ralat)
    }
}

// ✔ Collect validation errors
fn validate_idiomatic(data: &FormData) -> Result<(), Vec<String>> {
    let ralat: Vec<String> = [
        data.nama.is_empty().then_some("Nama kosong"),
        (!data.emel.contains('@')).then_some("Emel tidak sah"),
        (data.umur < 18).then_some("Perlu 18+"),
    ]
    .into_iter()
    .flatten()
    .map(str::to_string)
    .collect();

    if ralat.is_empty() { Ok(()) } else { Err(ralat) }
}
```

---

# BAHAGIAN 10: Mini Project — Refactor ke Idiomatic 🔄

```rust
// ─── VERSI ASAL (non-idiomatic) ──────────────────────────────
// Tugas: Baca fail CSV markah pelajar, kira statistik

use std::collections::HashMap;

fn baca_csv_buruk(path: &String) -> Vec<Vec<String>> {
    let kandungan = std::fs::read_to_string(path).unwrap();
    let mut hasil = Vec::new();
    for baris in kandungan.lines() {
        let mut fields = Vec::new();
        for field in baris.split(',') {
            fields.push(field.to_string());
        }
        hasil.push(fields);
    }
    hasil
}

fn kira_purata_buruk(markah: &Vec<Vec<String>>) -> HashMap<String, f64> {
    let mut purata: HashMap<String, f64> = HashMap::new();
    for i in 0..markah.len() {
        let nama = markah[i][0].clone();
        let mut jumlah: f64 = 0.0;
        let mut kiraan = 0;
        for j in 1..markah[i].len() {
            let m: f64 = markah[i][j].parse().unwrap();
            jumlah += m;
            kiraan += 1;
        }
        purata.insert(nama, jumlah / kiraan as f64);
    }
    purata
}

// ─── VERSI REFACTOR (idiomatic) ───────────────────────────────

use thiserror::Error;

#[derive(Debug, Error)]
enum CsvError {
    #[error("Gagal baca fail: {0}")]
    Io(#[from] std::io::Error),
    #[error("Format baris tidak sah pada baris {baris_no}: '{kandungan}'")]
    FormatTidakSah { baris_no: usize, kandungan: String },
    #[error("Markah tidak sah '{nilai}' pada baris {baris_no}")]
    MarkahTidakSah { nilai: String, baris_no: usize },
}

#[derive(Debug, Clone)]
struct RekodMarkah {
    nama:   String,
    markah: Vec<f64>,
}

impl RekodMarkah {
    fn purata(&self) -> Option<f64> {
        if self.markah.is_empty() {
            None
        } else {
            Some(self.markah.iter().sum::<f64>() / self.markah.len() as f64)
        }
    }

    fn tertinggi(&self) -> Option<f64> {
        self.markah.iter().cloned().reduce(f64::max)
    }

    fn terendah(&self) -> Option<f64> {
        self.markah.iter().cloned().reduce(f64::min)
    }

    fn gred(&self) -> char {
        match self.purata().unwrap_or(0.0) as u32 {
            90..=100 => 'A',
            75..=89  => 'B',
            60..=74  => 'C',
            50..=59  => 'D',
            _        => 'E',
        }
    }
}

// Parse satu baris CSV
fn parse_baris(baris: &str, no: usize) -> Result<RekodMarkah, CsvError> {
    let mut fields = baris.split(',');

    let nama = fields
        .next()
        .filter(|s| !s.is_empty())
        .ok_or_else(|| CsvError::FormatTidakSah {
            baris_no: no,
            kandungan: baris.to_string(),
        })?
        .trim()
        .to_string();

    let markah: Result<Vec<f64>, _> = fields
        .map(|s| {
            s.trim().parse::<f64>().map_err(|_| CsvError::MarkahTidakSah {
                nilai:    s.trim().to_string(),
                baris_no: no,
            })
        })
        .collect();

    Ok(RekodMarkah { nama, markah: markah? })
}

// Baca dan parse CSV
fn baca_csv(path: &str) -> Result<Vec<RekodMarkah>, CsvError> {
    let kandungan = std::fs::read_to_string(path)?;

    kandungan
        .lines()
        .enumerate()
        .filter(|(_, baris)| !baris.trim().is_empty() && !baris.starts_with('#'))
        .map(|(no, baris)| parse_baris(baris, no + 1))
        .collect()
}

fn cetak_laporan(rekod: &[RekodMarkah]) {
    println!("\n{'═'*55}");
    println!("{:^55}", "LAPORAN MARKAH PELAJAR");
    println!("{'═'*55}");
    println!("{:<20} {:>8} {:>8} {:>8} {:>5}",
        "Nama", "Purata", "Tinggi", "Rendah", "Gred");
    println!("{'─'*55}");

    for r in rekod {
        println!("{:<20} {:>8.1} {:>8.1} {:>8.1} {:>5}",
            r.nama,
            r.purata().unwrap_or(0.0),
            r.tertinggi().unwrap_or(0.0),
            r.terendah().unwrap_or(0.0),
            r.gred()
        );
    }

    println!("{'─'*55}");

    // Statistik keseluruhan menggunakan iterator
    if let (Some(terbaik), Some(terburuk)) = (
        rekod.iter().max_by(|a, b|
            a.purata().unwrap_or(0.0)
                .partial_cmp(&b.purata().unwrap_or(0.0))
                .unwrap()
        ),
        rekod.iter().min_by(|a, b|
            a.purata().unwrap_or(0.0)
                .partial_cmp(&b.purata().unwrap_or(0.0))
                .unwrap()
        ),
    ) {
        println!("Terbaik:  {} ({:.1})", terbaik.nama, terbaik.purata().unwrap_or(0.0));
        println!("Terburuk: {} ({:.1})", terburuk.nama, terburuk.purata().unwrap_or(0.0));
    }

    let purata_kelas: f64 = rekod.iter()
        .filter_map(|r| r.purata())
        .sum::<f64>()
        / rekod.len() as f64;

    println!("Purata kelas: {:.1}", purata_kelas);
    println!("{'═'*55}");
}

fn main() {
    // Buat data test
    let csv = "Ali Ahmad,85,90,78,92,88\n\
               Siti Hawa,92,88,95,90,87\n\
               Amin Razak,70,75,68,72,80\n\
               Zara Lina,60,65,58,70,62\n\
               # Razi - absent\n\
               Nora Hashim,95,98,92,97,96";

    std::fs::write("markah.csv", csv).unwrap();

    match baca_csv("markah.csv") {
        Ok(rekod) => cetak_laporan(&rekod),
        Err(e)    => eprintln!("Ralat: {}", e),
    }

    std::fs::remove_file("markah.csv").unwrap();
}
```

---

# 📋 Rujukan Pantas — Idiomatic Rust Checklist

## ✅ DO (Buat Ini)

```
Naming:
  ✔ snake_case untuk variables, functions, modules
  ✔ PascalCase untuk types (struct, enum, trait)
  ✔ SCREAMING_SNAKE untuk const dan static

Types:
  ✔ &str bukan &String untuk parameter baca-sahaja
  ✔ &[T] bukan &Vec<T> untuk slice parameter
  ✔ impl Into<T> untuk flexible owned parameters
  ✔ Option<T> untuk nilai yang mungkin tiada
  ✔ Result<T, E> untuk operasi yang mungkin gagal
  ✔ Enum untuk state yang mutually exclusive

Iterators:
  ✔ .iter(), .map(), .filter(), .collect()
  ✔ .enumerate() bila perlu index
  ✔ .any(), .all(), .find() untuk boolean queries
  ✔ .sum(), .product(), .min(), .max() untuk aggregation

Errors:
  ✔ ? operator untuk propagate
  ✔ thiserror untuk custom error types
  ✔ anyhow untuk application code
  ✔ Custom error enum untuk library code

Structs:
  ✔ impl Default untuk default values
  ✔ Builder pattern untuk complex construction
  ✔ #[derive] standard traits (Debug, Clone, PartialEq)
```

## ❌ DON'T (Jangan Buat Ini)

```
  ✘ .unwrap() dalam production code
  ✘ .clone() yang tidak perlu
  ✘ for i in 0..v.len() { ... v[i] ... }
  ✘ &String dalam function parameter (guna &str)
  ✘ &Vec<T> dalam function parameter (guna &[T])
  ✘ Nested match yang dalam (flatten dulu)
  ✘ Mutable global state (guna Atomic/Mutex)
  ✘ pub fields dalam struct tanpa justifikasi
  ✘ panic! dalam library code
  ✘ Box<dyn Error> dalam library API (guna custom enum)
```

---

## 🏆 Cabaran Akhir

Cuba refactor projek anda sendiri menggunakan checklist ini:

1. **Audit unwrap()** — ganti semua dengan proper error handling
2. **Audit clone()** — ganti dengan &T atau &[T] bila boleh
3. **Audit loops** — convert ke iterator chains
4. **Audit function signatures** — &String → &str, &Vec → &[T]
5. **Audit state representation** — guna enum bukan bool flags

---

*Idiomatic Rust bukan sekadar style — ia adalah cara Rust direka untuk digunakan.*
*Ikut idioms, dan compiler, safety, dan performance akan ikut sama.* 🦀
