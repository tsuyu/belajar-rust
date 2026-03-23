# 🧬 Clone: Bila OK, Bila Bottleneck, & Cara Refactor — Rust

> "Clone looks fine at first... but becomes a bottleneck."
> Panduan lengkap: kenapa clone mengasyikkan, kenapa ia merbahaya, dan
> bagaimana refactor dengan betul — di sinilah Rust benar-benar "clicks".

---

## Cerita Biasa Programmer Baru Rust

```
Hari 1:
  "Compiler kata value moved! Apa tu?"
  let b = a;
  println!("{}", a); // ERROR!

Hari 2:
  "Oh, letak .clone() je! Problem solved."
  let b = a.clone();
  println!("{}", a); // ✔ OK!

Hari 3-30:
  .clone() di mana-mana...
  "Rust susah tapi aku dah master clone!"

Bulan 3:
  Program jalan tapi LAMBAT.
  Profiler tunjuk 60% masa dalam allocation.
  "Wait... kenapa???"

  → Itulah masa Rust benar-benar "clicks".
```

---

## Apa Yang Akan Kita Belajar

```
Fasa 1: Clone — Asas & Bila OK
Fasa 2: Clone Bottleneck — Bila Jadi Masalah
Fasa 3: Refactor — Cara Betul Rust

Selepas ini:
  ✔ Faham kos sebenar clone()
  ✔ Kenal pasti bila clone perlu vs tidak perlu
  ✔ Refactor dengan &, &mut, lifetime, Cow<>, Arc<>
  ✔ Tulis kod Rust yang betul-betul idiomatik
```

---

# FASA 1: Clone — Asas & Bila OK 🌱

## Apa Yang clone() Sebenarnya Buat

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();

    // Apa berlaku dalam memori:
    //
    // Stack:          Heap (A):        Heap (B):
    // s1 → ptr ─────▶ h,e,l,l,o
    // s2 → ptr ─────────────────────▶ h,e,l,l,o
    //
    // clone() = allocate heap BARU + salin semua data
    // Mahal bila data besar!

    println!("{} {}", s1, s2); // kedua-dua valid
}
```

```
Kos clone():
  String("hello")    → allocate 5 bytes heap + copy
  Vec<i32>(1M items) → allocate 4MB heap + copy 4MB data!
  HashMap<K,V>       → allocate + clone semua keys + values
  Nested struct      → clone SETIAP field secara rekursif

Copy (berbanding clone):
  i32, f64, bool... → copy dalam STACK — hampir free!
  Tiada heap allocation.
```

---

## Bila clone() MEMANG OK

```rust
// ✔ OK 1: Setup awal / one-time initialization
fn main() {
    let config_asal = Konfigurasi::muat_dari_env();
    let config_thread1 = config_asal.clone(); // sekali je
    let config_thread2 = config_asal.clone(); // sekali je

    // Clone untuk hantar ke thread — wajar
    std::thread::spawn(move || guna(config_thread1));
    std::thread::spawn(move || guna(config_thread2));
}

// ✔ OK 2: Test / debugging
#[cfg(test)]
fn test_transform() {
    let asal = buat_data_ujian();
    let klon = asal.clone(); // OK dalam test — performance tidak kritikal
    let hasil = transform(asal);
    assert_ne!(hasil, klon);
}

// ✔ OK 3: Data kecil yang memang perlu owned
fn buat_pesan(nama: &str) -> String {
    let prefix = "Salam, ".to_string();
    // Kena clone kalau perlu modify
    let mut mesej = prefix.clone();
    mesej.push_str(nama);
    mesej
    // Tapi cara lebih baik: format!("Salam, {}", nama)
}

// ✔ OK 4: Terpaksa kerana API pihak ketiga
fn hantar_ke_api_luar(data: Data) {
    // API ambil ownership — terpaksa clone kalau nak guna lagi
    let salinan = data.clone();
    api_luar::hantar(data);
    simpan_log(salinan);
}

// ✔ OK 5: Nilai primitive yang "hampir free"
fn main() {
    let id: u32 = 42;
    let id2 = id; // COPY — bukan clone, tiada overhead
    // u32 implement Copy — .clone() pun sama cepat
}
```

---

## 🧠 Brain Teaser #1

Clone yang mana lebih mahal?

```rust
let a = "hello".to_string();           // A
let b = vec![1u8; 1_000_000];          // B
let c = vec!["x".to_string(); 1000];   // C
let d = 42u32;                          // D

let _ = a.clone();
let _ = b.clone();
let _ = c.clone();
let _ = d.clone();
```

<details>
<summary>👀 Jawapan</summary>

**Dari paling murah ke paling mahal:**

```
D. 42u32       → Copy type! clone() = salin 4 bytes stack. Hampir free.
A. "hello"     → allocate 5 bytes heap + copy. Murah.
B. vec![1u8; 1M] → allocate 1MB heap + copy 1MB. MAHAL.
C. vec!["x"; 1000] → allocate Vec + clone SETIAP String dalam vec.
                     = 1000x String allocation + copy. PALING MAHAL.
```

`C` adalah yang paling berbahaya kerana **clone rekursif** — clone Vec bermaksud clone setiap elemen, dan setiap `String` pula memerlukan heap allocation sendiri.

Kalau `c` adalah `vec![String::from("x".repeat(1000)); 1000]` — lebih teruk lagi: 1000 allocations × 1000 bytes = 1MB data!
</details>

---

# FASA 2: Clone Bottleneck — Bila Jadi Masalah 🔴

## Pattern Berbahaya #1: Clone dalam Loop

```rust
// ❌ BOTTLENECK — clone dalam hot loop
fn proses_data_buruk(rekod: Vec<String>) -> Vec<String> {
    let mut hasil = Vec::new();

    for item in &rekod {
        let item_klon = item.clone(); // ← clone setiap iterasi!
        let diproses = transform(item_klon);
        hasil.push(diproses);
    }

    hasil
}

// Berapa clone berlaku?
// rekod ada 1000 item? → 1000 clone!
// rekod ada 1M item?   → 1M clone! Setiap satu allocate heap.

// ✔ LEBIH BAIK — hantar reference
fn proses_data_baik(rekod: &[String]) -> Vec<String> {
    rekod.iter()
        .map(|item| transform_ref(item)) // pinjam, tidak clone!
        .collect()
}

fn transform(s: String) -> String { s.to_uppercase() }
fn transform_ref(s: &str) -> String { s.to_uppercase() }
```

---

## Pattern Berbahaya #2: Clone untuk "Selesaikan" Borrow Error

```rust
// ❌ CARA MALAS — clone untuk escape borrow checker
struct Pemproses {
    data: Vec<String>,
    cache: std::collections::HashMap<String, String>,
}

impl Pemproses {
    fn proses_malas(&mut self, kunci: &str) -> String {
        // Borrow checker complaint → clone untuk "selesaikan"
        let nilai = self.data.first().unwrap().clone(); // ← clone!
        self.cache.insert(kunci.to_string(), nilai.clone()); // ← clone lagi!
        nilai
    }

    // ✔ CARA BETUL — fikir semula struktur
    fn proses_betul(&mut self, kunci: &str) -> &str {
        if !self.cache.contains_key(kunci) {
            if let Some(nilai) = self.data.first() {
                self.cache.insert(kunci.to_string(), nilai.clone());
                // Clone sekali sahaja, dan hanya bila perlu masuk HashMap
            }
        }
        self.cache.get(kunci).map(|s| s.as_str()).unwrap_or("")
    }
}
```

---

## Pattern Berbahaya #3: Clone dalam Fungsi Parameter

```rust
// ❌ Clone sebelum hantar ke fungsi
fn main() {
    let nama = String::from("Ali Ahmad");
    let salam = buat_salam(nama.clone()); // ← kenapa clone?
    println!("{} → {}", nama, salam);
}

fn buat_salam(n: String) -> String { // ← ambil ownership!
    format!("Hello, {}!", n)
}

// ✔ Guna &str — lebih flexible, tiada allocation
fn main_baik() {
    let nama = String::from("Ali Ahmad");
    let salam = buat_salam_ref(&nama); // ← pinjam!
    println!("{} → {}", nama, salam);
}

fn buat_salam_ref(n: &str) -> String { // ← terima reference
    format!("Hello, {}!", n)
}
```

---

## Pattern Berbahaya #4: Clone dalam Struct

```rust
// ❌ Struct yang clone pada setiap akses
#[derive(Clone)]
struct Laporan {
    tajuk:     String,
    data:      Vec<BarisDat>,
    metadata:  Metadata,
}

impl Laporan {
    // Fungsi ini return clone SETIAP kali dipanggil!
    fn tajuk_sekarang(&self) -> String {
        self.tajuk.clone() // ← clone!
    }

    fn data_sekarang(&self) -> Vec<BarisDat> {
        self.data.clone()  // ← SANGAT mahal jika data besar!
    }
}

// ✔ Return reference — tiada allocation!
impl Laporan {
    fn tajuk(&self) -> &str {
        &self.tajuk  // ← reference ke String dalam struct
    }

    fn data(&self) -> &[BarisDat] {
        &self.data   // ← slice reference ke Vec dalam struct
    }
}

struct BarisDat { nilai: f64 }
struct Metadata { tarikh: String }
```

---

## Cara Kesan Clone Bottleneck

```bash
# 1. Guna cargo flamegraph
cargo install flamegraph
cargo flamegraph --bin nama-program

# 2. Guna criterion untuk benchmark
# Cargo.toml:
# [dev-dependencies]
# criterion = "0.5"

# 3. Guna perf (Linux)
cargo build --release
perf record ./target/release/nama-program
perf report

# 4. Guna heaptrack (untuk track allocation)
heaptrack ./target/release/nama-program

# Tanda-tanda ada masalah:
# ✗ Allocator dipanggil berkali-kali dalam loop
# ✗ Memory usage tinggi berbanding data size
# ✗ Flamegraph tunjuk masa besar dalam alloc/dealloc
# ✗ Clone muncul dalam hot path di flamegraph
```

---

## 🧠 Brain Teaser #2

Berapa kali heap allocation berlaku dalam fungsi ini?

```rust
fn proses(nama_list: Vec<String>) -> Vec<String> {
    nama_list.iter()
        .map(|n| n.clone())
        .filter(|n| n.len() > 3)
        .map(|n| n.to_uppercase())
        .collect()
}

// Panggil dengan:
let data = vec!["Ali".to_string(), "Siti".to_string(), "Amin".to_string()];
proses(data);
```

<details>
<summary>👀 Jawapan</summary>

Untuk 3 nama:
```
1. n.clone()        → 3 allocations (satu per nama)
2. n.to_uppercase() → 3 allocations (satu per nama yang lulus filter)
   Note: "Ali" (3 chars, tidak > 3) → discard selepas clone
         "Siti" (4 chars) → clone + uppercase
         "Amin" (4 chars) → clone + uppercase
3. collect()        → 1 allocation (untuk Vec baru)

Jumlah: ~7 allocations untuk 3 nama
```

**Versi lebih baik:**
```rust
fn proses_baik(nama_list: &[String]) -> Vec<String> {
    nama_list.iter()
        .filter(|n| n.len() > 3)   // filter DULU (tiada clone)
        .map(|n| n.to_uppercase())  // to_uppercase() sudah allocate
        .collect()
}
// Hanya 3 allocations: 2 uppercase + 1 Vec
// Kita elak clone() terus!
```
</details>

---

# FASA 3: Refactor — Cara Betul Rust ✅

## Senjata Refactor #1: & dan &str

```rust
// SEBELUM — clone mana-mana
fn format_nama(nama: String) -> String {
    format!("En. {}", nama)
}

fn main() {
    let nama = "Ali Ahmad".to_string();
    let formatted = format_nama(nama.clone()); // clone!
    println!("{}: {}", nama, formatted);
}

// SELEPAS — guna &str
fn format_nama(nama: &str) -> String {
    format!("En. {}", nama)
}

fn main() {
    let nama = "Ali Ahmad".to_string();
    let formatted = format_nama(&nama); // tiada clone!
    println!("{}: {}", nama, formatted);
}

// Tip: Kalau fungsi hanya BACA string, guna &str bukan String.
// &str boleh terima &String (auto-deref) DAN string literal.
fn cetak(s: &str) { println!("{}", s); }
cetak(&nama);              // ✔ dari String
cetak("literal");          // ✔ dari literal
cetak(nama.as_str());      // ✔ explicit convert
```

---

## Senjata Refactor #2: Return &str daripada String

```rust
struct Pekerja {
    nama:     String,
    bahagian: String,
}

// ❌ SEBELUM — return String (allocate + clone)
impl Pekerja {
    fn nama_buruk(&self) -> String {
        self.nama.clone() // allocation tidak perlu!
    }
}

// ✔ SELEPAS — return &str (borrow dari struct)
impl Pekerja {
    fn nama(&self) -> &str {
        &self.nama  // reference ke data dalam struct
    }

    fn bahagian(&self) -> &str {
        &self.bahagian
    }

    // Kalau return type perlu owned, baru guna String
    fn nama_dengan_jabatan(&self) -> String {
        format!("{} ({})", self.nama, self.bahagian)
        // format! allocate — tapi ini memang perlu string baru
    }
}

fn main() {
    let p = Pekerja {
        nama:     "Ali Ahmad".into(),
        bahagian: "ICT".into(),
    };

    let n: &str = p.nama();       // tiada allocation!
    let b: &str = p.bahagian();   // tiada allocation!
    let np: String = p.nama_dengan_jabatan(); // allocation (perlu)

    println!("{} dari {}", n, b);
    println!("{}", np);
}
```

---

## Senjata Refactor #3: Slice &[T] daripada Vec\<T\>

```rust
// ❌ SEBELUM — parameter Vec = ambil ownership atau clone
fn jumlah_buruk(data: Vec<i32>) -> i32 {
    data.iter().sum()
}

fn purata_buruk(data: Vec<i32>) -> f64 {
    data.iter().sum::<i32>() as f64 / data.len() as f64
}

fn main() {
    let v = vec![1, 2, 3, 4, 5];
    println!("{}", jumlah_buruk(v.clone())); // clone!
    println!("{}", purata_buruk(v.clone())); // clone lagi!
    // v mungkin dah moved kalau tanpa clone
}

// ✔ SELEPAS — &[T] berfungsi untuk Vec DAN array
fn jumlah(data: &[i32]) -> i32 {
    data.iter().sum()
}

fn purata(data: &[i32]) -> f64 {
    data.iter().sum::<i32>() as f64 / data.len() as f64
}

fn max_nilai(data: &[i32]) -> Option<i32> {
    data.iter().copied().reduce(i32::max)
}

fn main() {
    let v = vec![1, 2, 3, 4, 5];
    let arr = [10, 20, 30, 40, 50];

    println!("{}", jumlah(&v));         // tiada clone, tiada move
    println!("{:.1}", purata(&v));      // tiada clone
    println!("{:?}", max_nilai(&v));    // tiada clone
    println!("{}", jumlah(&arr));       // sama fungsi, array pun ok!
    println!("{}", jumlah(&v[1..4]));   // slice sebahagian
}
```

---

## Senjata Refactor #4: Cow\<'a, str\> — Clone On Write

```rust
use std::borrow::Cow;

// Cow = "Clone Only When You Need To Write"
//
// Cow<'a, str> ada dua keadaan:
//   Borrowed(&'a str) → pinjam, tiada allocation
//   Owned(String)     → milik sendiri, ada allocation
//
// Allocate HANYA bila perlu ubah!

fn bersihkan_input(input: &str) -> Cow<str> {
    if input.contains("  ") {
        // Perlu ubah → return Owned (ada allocation)
        Cow::Owned(input.replace("  ", " "))
    } else {
        // Tidak perlu ubah → return Borrowed (tiada allocation!)
        Cow::Borrowed(input)
    }
}

fn tambah_prefix<'a>(s: &'a str, perlu_prefix: bool) -> Cow<'a, str> {
    if perlu_prefix {
        Cow::Owned(format!("KADA: {}", s))  // allocation hanya bila perlu
    } else {
        Cow::Borrowed(s)  // tiada allocation!
    }
}

fn main() {
    let bersih   = "hello world";
    let kotor    = "hello  world";  // ada double space

    let hasil1 = bersihkan_input(bersih); // Borrowed — tiada alloc!
    let hasil2 = bersihkan_input(kotor);  // Owned — ada alloc (perlu)

    println!("{}", hasil1); // "hello world"
    println!("{}", hasil2); // "hello world" (sudah bersih)

    // Cow boleh guna seperti &str
    fn panjang(s: &str) -> usize { s.len() }
    println!("{}", panjang(&hasil1)); // deref Cow → &str
    println!("{}", panjang(&hasil2));

    // Cow<[T]> untuk Vec/slice
    fn proses_senarai(data: Cow<[i32]>) -> i32 {
        data.iter().sum()
    }

    let v = vec![1, 2, 3, 4, 5];
    let hasil = proses_senarai(Cow::Borrowed(&v)); // tiada clone!
    println!("Jumlah: {}", hasil);
}
```

---

## Senjata Refactor #5: Arc\<T\> untuk Share Tanpa Clone

```rust
use std::sync::Arc;

// Arc = Atomic Reference Counting
// Kongsi data TANPA clone data itu sendiri!
// Hanya pointer (8 bytes) yang diklon, bukan data.

#[derive(Debug)]
struct KonfigBesar {
    data:   Vec<String>,   // mungkin besar
    rules:  Vec<String>,   // banyak rules
    index:  std::collections::HashMap<String, usize>,
}

// ❌ SEBELUM — clone seluruh config
fn proses_dengan_config_buruk(config: KonfigBesar, item: &str) -> String {
    // config masuk sini, terpaksa clone kalau nak guna lagi
    format!("{} dengan {} rules", item, config.rules.len())
}

fn main_buruk() {
    let config = KonfigBesar {
        data:  vec!["a".into(); 10000],
        rules: vec!["rule".into(); 1000],
        index: std::collections::HashMap::new(),
    };

    // Nak guna config dalam banyak tempat → terpaksa clone!
    let r1 = proses_dengan_config_buruk(config.clone(), "item1"); // clone 11MB?
    let r2 = proses_dengan_config_buruk(config.clone(), "item2"); // clone lagi!
}

// ✔ SELEPAS — Arc<T> untuk share
fn proses_dengan_arc(config: Arc<KonfigBesar>, item: &str) -> String {
    format!("{} dengan {} rules", item, config.rules.len())
}

fn main_baik() {
    let config = Arc::new(KonfigBesar {
        data:  vec!["a".into(); 10000],
        rules: vec!["rule".into(); 1000],
        index: std::collections::HashMap::new(),
    });

    // Arc::clone hanya copy POINTER (8 bytes), bukan data!
    let r1 = proses_dengan_arc(Arc::clone(&config), "item1");
    let r2 = proses_dengan_arc(Arc::clone(&config), "item2");

    println!("{}", r1);
    println!("{}", r2);
    // Data KonfigBesar hanya ada SATU salinan dalam memori!
}
```

---

## Senjata Refactor #6: Entry API — Elak Clone dalam HashMap

```rust
use std::collections::HashMap;

fn main() {
    let mut kiraan: HashMap<String, u32> = HashMap::new();
    let perkataan = vec!["hello", "world", "hello", "rust", "hello"];

    // ❌ BURUK — clone key setiap kali
    for kata in &perkataan {
        if kiraan.contains_key(*kata) {
            *kiraan.get_mut(*kata).unwrap() += 1;
        } else {
            kiraan.insert(kata.to_string(), 1); // insert baru
        }
        // contains_key + get_mut = dua lookup!
    }

    // ✔ BAIK — entry API, tiada double lookup, tiada clone yang sia-sia
    let mut kiraan2: HashMap<String, u32> = HashMap::new();
    for kata in &perkataan {
        *kiraan2.entry(kata.to_string()).or_insert(0) += 1;
        // Hanya allocate String kalau key belum ada!
    }

    // ✔ LEBIH BAIK — elak clone key bila cari
    let mut kiraan3: HashMap<&str, u32> = HashMap::new();
    for kata in &perkataan {
        *kiraan3.entry(kata).or_insert(0) += 1;
        // Key adalah &str — tiada allocation langsung!
    }

    println!("{:?}", kiraan3);
}
```

---

## Senjata Refactor #7: Pindah Logik, Bukan Data

```rust
// ❌ Clone data untuk bawa ke tempat lain
fn analisis_buruk(data: Vec<f64>) -> (f64, f64) {
    let min = data.clone().into_iter().reduce(f64::min).unwrap_or(0.0);
    let max = data.clone().into_iter().reduce(f64::max).unwrap_or(0.0);
    (min, max)
}

// ✔ Iterate data sekali, kira semua dalam satu pass
fn analisis_baik(data: &[f64]) -> Option<(f64, f64)> {
    if data.is_empty() { return None; }

    let mut min = data[0];
    let mut max = data[0];

    for &nilai in &data[1..] {
        if nilai < min { min = nilai; }
        if nilai > max { max = nilai; }
    }

    Some((min, max))
}

// ✔ Guna iterator tanpa clone
fn statistik(data: &[f64]) -> Option<(f64, f64, f64)> {
    if data.is_empty() { return None; }

    let min = data.iter().cloned().reduce(f64::min)?;
    let max = data.iter().cloned().reduce(f64::max)?;
    let purata = data.iter().sum::<f64>() / data.len() as f64;
    // .cloned() = copy f64 dari reference — f64 adalah Copy, cepat!

    Some((min, max, purata))
}
```

---

## 🧠 Brain Teaser #3

Refactor fungsi ini tanpa guna .clone():

```rust
// Ini adalah kod yang ada terlalu banyak clone
fn proses_rekod(rekod: Vec<Rekod>) -> HashMap<String, Vec<String>> {
    let mut hasil: HashMap<String, Vec<String>> = HashMap::new();

    for r in &rekod {
        let kategori = r.kategori.clone();  // clone 1
        let nama     = r.nama.clone();      // clone 2

        hasil.entry(kategori)
             .or_insert_with(Vec::new)
             .push(nama);
    }

    hasil
}

struct Rekod {
    nama:     String,
    kategori: String,
}
```

<details>
<summary>👀 Jawapan</summary>

```rust
// Cara 1: Masih ada clone tapi lebih sedikit (unavoidable untuk HashMap<String, ..>)
fn proses_rekod_v1(rekod: &[Rekod]) -> HashMap<&str, Vec<&str>> {
    let mut hasil: HashMap<&str, Vec<&str>> = HashMap::new();

    for r in rekod {
        hasil.entry(&r.kategori)   // &str — tiada clone!
             .or_default()
             .push(&r.nama);       // &str — tiada clone!
    }

    hasil
    // Return HashMap<&str, Vec<&str>> — data masih dalam rekod
    // lifetime terikat kepada rekod
}

// Cara 2: Kalau perlu owned HashMap, clone hanya sekali bila insert
fn proses_rekod_v2(rekod: &[Rekod]) -> HashMap<String, Vec<String>> {
    let mut hasil: HashMap<String, Vec<String>> = HashMap::new();

    for r in rekod {
        hasil.entry(r.kategori.clone()) // clone kategori hanya bila key baru
             .or_default()
             .push(r.nama.clone());     // clone nama sekali je
        // entry() tidak clone kalau key sudah ada!
    }

    hasil
}

// Cara 3: Consume Vec (kalau tak perlu rekod asal)
fn proses_rekod_v3(rekod: Vec<Rekod>) -> HashMap<String, Vec<String>> {
    let mut hasil: HashMap<String, Vec<String>> = HashMap::new();

    for r in rekod {  // consume — move setiap Rekod
        hasil.entry(r.kategori) // MOVE! tiada clone
             .or_default()
             .push(r.nama);     // MOVE! tiada clone
    }

    hasil
}

// Cara 3 adalah ZERO clone untuk content — semua di-move!
```
</details>

---

## Refactor Real-World: Sebelum & Selepas Penuh

```rust
use std::collections::HashMap;

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// VERSI 1: Clone mana-mana (beginner code)
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

#[derive(Clone, Debug)]
struct PelajarV1 {
    nama:     String,
    markah:   Vec<f64>,
    bahagian: String,
}

struct SistemV1 {
    pelajar: Vec<PelajarV1>,
}

impl SistemV1 {
    fn cari_nama(&self, nama: &str) -> Option<PelajarV1> {
        self.pelajar
            .iter()
            .find(|p| p.nama == nama)
            .map(|p| p.clone()) // ← clone kerana return owned
    }

    fn nama_semua(&self) -> Vec<String> {
        self.pelajar
            .iter()
            .map(|p| p.nama.clone()) // ← clone dalam loop!
            .collect()
    }

    fn ikut_bahagian(&self) -> HashMap<String, Vec<String>> {
        let mut map = HashMap::new();
        for p in &self.pelajar {
            map.entry(p.bahagian.clone())     // ← clone key
               .or_default()
               .push(p.nama.clone());          // ← clone value
        }
        map
    }

    fn purata_markah(&self, nama: &str) -> Option<f64> {
        let p = self.cari_nama(nama)?; // ← clone pelajar!
        if p.markah.is_empty() { return None; }
        Some(p.markah.clone().iter().sum::<f64>() // ← clone Vec<f64>!
             / p.markah.len() as f64)
    }
}

// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// VERSI 2: Refactor dengan references
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

#[derive(Debug)]
struct PelajarV2 {
    nama:     String,
    markah:   Vec<f64>,
    bahagian: String,
}

struct SistemV2 {
    pelajar: Vec<PelajarV2>,
}

impl SistemV2 {
    // Return &Pelajar — tiada clone!
    fn cari_nama(&self, nama: &str) -> Option<&PelajarV2> {
        self.pelajar.iter().find(|p| p.nama == nama)
    }

    // Return &str — tiada clone!
    fn nama_semua(&self) -> Vec<&str> {
        self.pelajar.iter().map(|p| p.nama.as_str()).collect()
    }

    // Key dan value adalah &str — tiada clone!
    fn ikut_bahagian(&self) -> HashMap<&str, Vec<&str>> {
        let mut map: HashMap<&str, Vec<&str>> = HashMap::new();
        for p in &self.pelajar {
            map.entry(&p.bahagian)  // &str — tiada clone
               .or_default()
               .push(&p.nama);      // &str — tiada clone
        }
        map
    }

    // Guna &Pelajar — tiada clone!
    fn purata_markah(&self, nama: &str) -> Option<f64> {
        let p = self.cari_nama(nama)?; // &PelajarV2 — tiada clone!
        if p.markah.is_empty() { return None; }
        // f64 adalah Copy — iter().sum() tiada allocation
        Some(p.markah.iter().sum::<f64>() / p.markah.len() as f64)
    }

    // Kalau betul-betul perlu owned data (contoh: send to thread)
    fn eksport_nama_owned(&self) -> Vec<String> {
        self.pelajar.iter().map(|p| p.nama.clone()).collect()
        // Clone here is INTENTIONAL — caller needs owned Strings
    }
}

fn main() {
    let sistem = SistemV2 {
        pelajar: vec![
            PelajarV2 { nama: "Ali".into(), markah: vec![85.0, 90.0], bahagian: "ICT".into() },
            PelajarV2 { nama: "Siti".into(), markah: vec![78.0, 92.0], bahagian: "HR".into() },
        ],
    };

    // Tiada clone berlaku dalam semua ini:
    if let Some(p) = sistem.cari_nama("Ali") {
        println!("Jumpa: {}", p.nama);
    }

    for nama in sistem.nama_semua() {
        println!("- {}", nama);
    }

    if let Some(purata) = sistem.purata_markah("Siti") {
        println!("Purata Siti: {:.1}", purata);
    }
}
```

---

## Panduan Keputusan — Clone atau Tidak?

```
Soal diri sendiri:

1. "Adakah fungsi ini PERLU mengubah data?"
   TIDAK → &T atau &[T]   (immutable reference)
   YA    → &mut T          (mutable reference)
   YA + PERLU OWNED → clone (terakhir!)

2. "Adakah return value perlu hidup lebih lama dari input?"
   TIDAK → return &T (reference dengan lifetime)
   YA    → terpaksa return owned T (clone atau move)

3. "Adakah data ini dikongsi antara thread?"
   YA + immutable → Arc<T>          (shared ownership)
   YA + mutable   → Arc<Mutex<T>>   (shared + mutable)

4. "Adakah transformasi diperlukan KADANG-KADANG sahaja?"
   YA → Cow<T>  (clone only when write)

5. "Adakah ini dalam hot loop / performance-critical path?"
   YA → Elak clone SEPENUHNYA. Guna &, iterator, Cow.
   TIDAK → Clone mungkin OK (readability > micro-optimisation)
```

---

## Checklist Refactor Clone

```
Sebelum commit kod yang ada .clone():

□ Adakah clone ini dalam loop? → guna iterator dengan &
□ Adakah clone ini untuk "escape" borrow error? → fikir semula design
□ Boleh parameter jadi &str instead of String?
□ Boleh parameter jadi &[T] instead of Vec<T>?
□ Boleh return &str instead of String?
□ Boleh return &[T] instead of Vec<T>?
□ Adakah ini untuk share data? → guna Arc<T>
□ Adakah clone conditional? → pertimbangkan Cow<T>
□ Adakah ini untuk HashMap key? → guna entry API dengan &T
□ Clone yang tinggal — adakah ia genuinely perlu? → tulis komen kenapa
```

---

## Ringkasan: Dari Clone ke Idiomatik Rust

```
EVOLUTION PROGRAMMER RUST:

Tahap 1 — Clone Everything (beginner):
  let b = a.clone();
  fungsi(s.clone(), v.clone());
  // "Ia compile! 🎉"

Tahap 2 — Sedar Ada Masalah:
  // "Kenapa program saya lambat?"
  // Profiler: 70% masa dalam alloc/dealloc
  // "Oh... .clone()..."

Tahap 3 — Belajar References:
  fungsi(&s, &v);
  fn f(s: &str) → ...
  fn f(v: &[T]) → ...

Tahap 4 — Master Lifetimes & Cow:
  fn f<'a>(s: &'a str) -> &'a str
  fn f(s: &str) -> Cow<str>

Tahap 5 — Idiomatik Rust (pakar):
  Clone hanya bila genuinely perlu.
  Borrow checker jadi kawan, bukan musuh.
  Code adalah cepat DAN selamat.
  → INI dia saat Rust "clicks"!
```

---

## 🏆 Cabaran Akhir

Cuba refactor projek ini step by step:

```rust
// Misi: Refactor ini tanpa ubah behaviour,
//       kurangkan clone sebanyak mungkin.

#[derive(Clone)]
struct Inventori {
    nama:   String,
    item:   Vec<String>,
    tanda:  Vec<String>,
}

fn cari_inventori(senarai: &Vec<Inventori>, nama: String) -> Option<Inventori> {
    senarai.iter()
           .find(|i| i.nama == nama)
           .map(|i| i.clone())
}

fn nama_semua(senarai: &Vec<Inventori>) -> Vec<String> {
    senarai.iter().map(|i| i.nama.clone()).collect()
}

fn dengan_tanda(senarai: &Vec<Inventori>, tanda: String) -> Vec<Inventori> {
    senarai.iter()
           .filter(|i| i.tanda.contains(&tanda))
           .map(|i| i.clone())
           .collect()
}

fn jumlah_item(senarai: &Vec<Inventori>) -> HashMap<String, usize> {
    let mut m = HashMap::new();
    for i in senarai {
        m.insert(i.nama.clone(), i.item.len());
    }
    m
}
```

<details>
<summary>👀 Jawapan Refactor</summary>

```rust
// Versi refactor — tiada clone yang sia-sia

struct Inventori {
    nama:  String,
    item:  Vec<String>,
    tanda: Vec<String>,
}

// &Inventori — tiada clone!
fn cari_inventori<'a>(senarai: &'a [Inventori], nama: &str) -> Option<&'a Inventori> {
    senarai.iter().find(|i| i.nama == nama)
}

// Vec<&str> — tiada clone!
fn nama_semua<'a>(senarai: &'a [Inventori]) -> Vec<&'a str> {
    senarai.iter().map(|i| i.nama.as_str()).collect()
}

// Vec<&Inventori> — tiada clone!
fn dengan_tanda<'a>(senarai: &'a [Inventori], tanda: &str) -> Vec<&'a Inventori> {
    senarai.iter()
           .filter(|i| i.tanda.iter().any(|t| t == tanda))
           .collect()
}

// HashMap<&str, usize> — tiada clone!
fn jumlah_item<'a>(senarai: &'a [Inventori]) -> HashMap<&'a str, usize> {
    senarai.iter()
           .map(|i| (i.nama.as_str(), i.item.len()))
           .collect()
}

// Penggunaan:
fn main() {
    let senarai = vec![
        Inventori {
            nama:  "Gudang A".into(),
            item:  vec!["beras".into(), "gula".into()],
            tanda: vec!["makanan".into()],
        },
    ];

    if let Some(inv) = cari_inventori(&senarai, "Gudang A") {
        println!("Jumpa: {}", inv.nama);
    }

    for nama in nama_semua(&senarai) {
        println!("- {}", nama); // &str
    }
}
```
</details>

---

*Clone adalah permulaan. Elak clone yang tidak perlu adalah Rust yang sebenar.* 🦀
