# 🧬 Clone: Bila Betul, Bila Salah, & Alternatifnya

> "Sometimes the correct Rust solution is just cloning the value and moving on."
> Panduan pragmatik: bila clone adalah jawapan yang tepat,
> bila ia adalah masalah, dan apa alternatifnya.

---

## Premis Utama

```
Rust borrow checker kadang-kala menolak kod yang "sepatutnya" berfungsi.
Response programmer baru: tambah .clone() sehingga compile.
Response programmer advance: cuba elak clone sepenuhnya.

KEDUANYA SALAH kadang-kala.

Jawapan yang betul: FIKIR dulu.

  "Adakah clone ini murah? Ya → clone dan teruskan."
  "Adakah clone ini dalam hot loop? Mungkin perlu refactor."
  "Adakah clone di sini menyelesaikan masalah sebenar? → clone."
  "Adakah clone menyembunyikan design problem? → refactor."
```

---

## Peta Panduan

```
Bab 1  → Kos Clone — Apa Yang Sebenarnya Berlaku
Bab 2  → Bila Clone ADALAH Jawapan Yang Betul
Bab 3  → Alternatif #1: Borrowing (&, &mut)
Bab 4  → Alternatif #2: Lifetime Annotation
Bab 5  → Alternatif #3: Arc<T> — Shared Ownership
Bab 6  → Alternatif #4: Cow<T> — Clone On Write
Bab 7  → Alternatif #5: Redesign Struktur Data
Bab 8  → Alternatif #6: Move Instead of Clone
Bab 9  → Decision Flowchart
Bab 10 → Mini Project: Refactor Clone ke Idiomatic
```

---

# BAB 1: Kos Clone — Apa Yang Sebenarnya Berlaku 💰

## Bukan Semua Clone Sama

```rust
// ── Clone yang MURAH (hampir free) ───────────────────────────
let n: i32 = 42;
let n2 = n.clone(); // 4 bytes copy pada stack — tidak beza dengan n2 = n

let f: f64 = 3.14;
let f2 = f.clone(); // 8 bytes — Copy type, clone = copy

let b: bool = true;
let b2 = b; // BUKAN clone — Copy, terus copy stack

// ── Clone yang SEDERHANA (kecil, OK dalam kebanyakan kes) ─────
let s: String = "hello".into();
let s2 = s.clone(); // heap allocation + 5 bytes copy → OK untuk sekali guna

// ── Clone yang MAHAL (perlu fikir dua kali) ──────────────────
let v: Vec<String> = vec!["ali".into(), "siti".into(), "amin".into()];
let v2 = v.clone();
// → Allocate Vec baru
// → Clone SETIAP String (3 × heap allocation!)
// Semakin besar Vec, semakin mahal!

// ── Clone yang SANGAT MAHAL (dalam loop = disaster) ──────────
let data: Vec<Vec<u8>> = vec![vec![0u8; 1000]; 100];
// → 100 * 1000 bytes = 100KB, plus overhead
for _ in 0..1000 {
    let _salinan = data.clone(); // 100MB allocation setiap iterasi!
}

// ── Summary: kos clone bergantung pada APA yang di-clone ──────
// Copy types (i32, f64, bool, char, &T):  clone ≈ assignment (free)
// Small String/Vec:                        OK, tapi ada allocation
// Large Vec, nested structures:            MAHAL, perlu fikir
// Arc<T>:                                 clone = atomic increment (murah!)
```

---

## 🧠 Brain Teaser #1

Yang mana lebih mahal?

```rust
// A
let a: Arc<Vec<String>> = Arc::new(vec!["panjang".repeat(1000); 100]);
let a2 = a.clone();

// B
let b: Vec<String> = vec!["panjang".repeat(1000); 100];
let b2 = b.clone();

// C
let c: i32 = 42;
let c2 = c.clone();
```

<details>
<summary>👀 Jawapan</summary>

```
C (i32.clone()):
  → 4 bytes pada stack
  → hampir free

A (Arc::clone()):
  → HANYA increment reference counter (atomic operation)
  → Data TIDAK disalin! Arc kongsi data yang sama
  → ~nanoseconds, tidak kira saiz data

B (Vec<String>.clone()):
  → Allocate Vec baru
  → Clone SETIAP String: 100 × "panjang" × 1000 chars = ~100KB allocation
  → Salin 100KB data
  → PALING MAHAL walaupun terlihat "biasa"

Dari murah ke mahal: C < A <<< B

Pelajaran:
  Arc::clone() adalah MURAH walaupun data besar!
  Arc hanya clone pointer, bukan data.
  Inilah sebab Arc ada — untuk share data tanpa clone data sebenar.
```
</details>

---

# BAB 2: Bila Clone ADALAH Jawapan Yang Betul ✅

## Kes 1: Setup & Initialization

```rust
use std::sync::Arc;

struct Config {
    host:     String,
    port:     u16,
    pekerja:  u32,
}

fn main() {
    let config = Arc::new(Config {
        host: "localhost".into(),
        port: 8080,
        pekerja: 4,
    });

    // ✔ Clone Arc untuk thread — ini BETUL dan murah
    for i in 0..config.pekerja {
        let cfg = Arc::clone(&config); // ← BETUL! clone Arc, bukan data
        std::thread::spawn(move || {
            println!("Thread {} mendengar pada {}:{}",
                i, cfg.host, cfg.port);
        });
    }
}
```

---

## Kes 2: API Boundary — Bila Caller Perlu Ownership

```rust
// Situasi: fungsi perlu hantar String ke API luar yang ambil ownership

struct EmailSender;

impl EmailSender {
    fn hantar(&self, kepada: String, subjek: String, badan: String) {
        // API ini ambil ownership — tidak dapat &str
        println!("Hantar ke: {}", kepada);
    }
}

fn main() {
    let penerima = "ali@email.com".to_string();
    let subjek   = "Laporan Bulanan".to_string();
    let badan    = "Kandungan laporan...".to_string();

    let pengirim = EmailSender;

    // ✔ Clone di sini adalah BETUL — kita perlu kekalkan nilai-nilai ini
    pengirim.hantar(
        penerima.clone(), // ← clone untuk API
        subjek.clone(),
        badan.clone(),
    );

    // Masih perlu nilai-nilai ini selepas hantar
    println!("Emel dihantar kepada: {}", penerima);
    println!("Subjek: {}", subjek);

    // Alternatif: refactor EmailSender untuk terima &str
    // Tapi kalau ia dari crate luar yang tidak boleh diubah → clone adalah betul!
}
```

---

## Kes 3: Test & Debugging

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_transform() {
        let asal = BuatData::baru("contoh");

        // ✔ Clone dalam test adalah FINE — performance tidak kritikal
        let klon = asal.clone();
        let hasil = transform(asal);

        // Bandingkan hasil dengan klon asal
        assert_ne!(hasil, klon);
        assert_eq!(hasil.diproses, true);
    }
}
```

---

## Kes 4: Prototype & Early Development

```rust
// Semasa prototype — CLONE DAHULU, optimize kemudian
// "Make it work, make it right, make it fast"

fn proses_data_v1(data: Vec<Item>) -> Vec<HasilItem> {
    data.iter()
        .map(|item| {
            // ✔ Clone dalam prototype — nak pastikan logik betul dulu
            let mut salinan = item.clone();
            salinan.proses();
            HasilItem::dari(salinan)
        })
        .collect()
}

// Selepas logik betul, optimize:
fn proses_data_v2(data: &[Item]) -> Vec<HasilItem> {
    data.iter()
        .map(|item| HasilItem::dari_ref(item)) // tiada clone!
        .collect()
}
```

---

## Kes 5: Small, Copy-like Data

```rust
#[derive(Clone, Copy)] // boleh jadi Copy!
struct Titik { x: f32, y: f32 }

#[derive(Clone)]
struct WarnaNama(String);

fn main() {
    // ✔ Titik: Copy type — clone adalah assignment
    let t1 = Titik { x: 1.0, y: 2.0 };
    let t2 = t1; // auto-copy, tidak ada "clone" sebenarnya
    println!("{} {}", t1.x, t2.x); // t1 masih valid!

    // ✔ WarnaNama kecil — clone adalah OK
    let merah = WarnaNama("Merah".into());
    let juga_merah = merah.clone();
    println!("{}", merah.0); // "Merah"
}
```

---

# BAB 3: Alternatif #1 — Borrowing (&, &mut) 🔗

## Paling Biasa: Guna &str Bukan String

```rust
// ❌ Clone tidak perlu — parameter ambil ownership
fn hitung_vokal(s: String) -> usize {
    s.chars().filter(|c| "aeiou".contains(*c)).count()
}

fn main() {
    let nama = "Ali Ahmad".to_string();
    let v = hitung_vokal(nama.clone()); // clone! kenapa?
    println!("{}: {} vokal", nama, v);
}

// ✔ Guna &str — tidak perlu clone langsung
fn hitung_vokal(s: &str) -> usize {
    s.chars().filter(|c| "aeiou".contains(*c)).count()
}

fn main() {
    let nama = "Ali Ahmad".to_string();
    let v = hitung_vokal(&nama); // tiada clone!
    println!("{}: {} vokal", nama, v);

    // Boleh panggil dengan &str terus
    let v2 = hitung_vokal("Siti Hawa");
    // Dan dengan String via auto-deref
    let v3 = hitung_vokal(&nama);
}

// ── Panduan: String parameter ─────────────────────────────────
//
// fn f(s: String)    → ambil ownership, caller mesti clone atau move
// fn f(s: &str)      → pinjam, boleh terima &String dan &str
// fn f(s: &String)   → terlalu spesifik, guna &str!
// fn f(s: impl Into<String>) → flexible, ambil ownership bila perlu
```

---

## &[T] Bukan Vec\<T\>

```rust
// ❌ Vec parameter — paksa caller clone atau move
fn jumlah(v: Vec<i32>) -> i32 { v.iter().sum() }

fn main() {
    let data = vec![1, 2, 3, 4, 5];
    let j1 = jumlah(data.clone()); // clone!
    let j2 = jumlah(data.clone()); // clone lagi!
    println!("{} {}", j1, j2);
}

// ✔ &[T] — flexible, tidak perlu clone
fn jumlah(v: &[i32]) -> i32 { v.iter().sum() }

fn main() {
    let data = vec![1, 2, 3, 4, 5];
    let arr = [1, 2, 3, 4, 5];

    let j1 = jumlah(&data);    // Vec<i32> → &[i32] auto
    let j2 = jumlah(&data);    // Boleh guna berkali-kali!
    let j3 = jumlah(&arr);     // Array pun OK!
    let j4 = jumlah(&data[1..]); // Slice pun OK!

    println!("{} {} {} {}", j1, j2, j3, j4);
}
```

---

## Borrow dalam Struct

```rust
// ❌ Struct simpan String — setiap penggunaan perlu clone
struct LaporanBuruk {
    tajuk:    String,
    penulis:  String,
    kandungan: String,
}

fn cetak_laporan(laporan: &LaporanBuruk) {
    let t = laporan.tajuk.clone(); // kenapa clone?
    println!("Laporan: {}", t);
}

// ✔ Guna &str dalam struct (dengan lifetime)
struct Laporan<'a> {
    tajuk:    &'a str,
    penulis:  &'a str,
    kandungan: &'a str,
}

fn cetak_laporan(laporan: &Laporan) {
    println!("Laporan: {}", laporan.tajuk); // tiada clone!
}

fn main() {
    let tajuk = "Laporan Q1 2024".to_string();
    let l = Laporan {
        tajuk:    &tajuk,
        penulis:  "Ali Ahmad",
        kandungan: "...",
    };
    cetak_laporan(&l);
}
```

---

# BAB 4: Alternatif #2 — Lifetime Annotation ⏳

## Bila Borrow Perlu Lifetime

```rust
// Situasi: fungsi return reference yang hidup selama input

// ❌ Terpaksa clone kerana tidak tahu dari mana reference datang
fn terpanjang_clone(s1: &str, s2: &str) -> String {
    if s1.len() >= s2.len() {
        s1.to_string() // clone!
    } else {
        s2.to_string() // clone!
    }
}

// ✔ Guna lifetime — return reference ke salah satu input
fn terpanjang<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() >= s2.len() { s1 } else { s2 }
    // tiada clone! Return reference terus
}

fn main() {
    let s1 = String::from("panjang sekali");
    let hasil;
    {
        let s2 = String::from("pendek");
        hasil = terpanjang(&s1, &s2);
        println!("Terpanjang: {}", hasil); // OK di dalam scope s2
    }
    // println!("{}", hasil); // ERROR! s2 sudah drop
}
```

---

## Struct dengan Lifetime

```rust
// Bila struct perlu pinjam data (bukan simpan salinan)

struct PemilihToken<'a> {
    teks:    &'a str,
    kedudukan: usize,
}

impl<'a> PemilihToken<'a> {
    fn baru(teks: &'a str) -> Self {
        PemilihToken { teks, kedudukan: 0 }
    }

    fn seterusnya(&mut self) -> Option<&'a str> {
        if self.kedudukan >= self.teks.len() {
            return None;
        }
        let mula = self.kedudukan;
        // Cari token seterusnya...
        let akhir = self.teks[mula..]
            .find(|c: char| c.is_whitespace())
            .map(|i| mula + i)
            .unwrap_or(self.teks.len());
        self.kedudukan = akhir + 1;
        Some(&self.teks[mula..akhir]) // reference ke data asal, tiada clone!
    }
}

fn main() {
    let ayat = String::from("Ali Ahmad bekerja di KADA");
    let mut pemilih = PemilihToken::baru(&ayat);

    while let Some(token) = pemilih.seterusnya() {
        println!("Token: '{}'", token); // tiada clone sepanjang proses!
    }
}
```

---

# BAB 5: Alternatif #3 — Arc\<T\> — Shared Ownership 🔄

## Arc untuk Data Kongsi Tanpa Clone Data

```rust
use std::sync::Arc;

// ❌ Clone data besar untuk setiap thread
fn dengan_clone() {
    let data: Vec<u8> = vec![0u8; 1_000_000]; // 1MB

    for i in 0..4 {
        let salinan = data.clone(); // clone 1MB setiap kali!
        std::thread::spawn(move || {
            println!("Thread {}: {} bytes", i, salinan.len());
        });
    }
}

// ✔ Arc — share data tanpa clone
fn dengan_arc() {
    let data: Arc<Vec<u8>> = Arc::new(vec![0u8; 1_000_000]); // 1MB

    for i in 0..4 {
        let d = Arc::clone(&data); // hanya clone POINTER (atomic increment)
        std::thread::spawn(move || {
            println!("Thread {}: {} bytes", i, d.len()); // baca data kongsi
        });
    }
    // Data 1MB hanya SATU salinan dalam memory!
}

// ── Arc dalam App State ───────────────────────────────────────
#[derive(Clone)] // Clone hanya clone Arc pointers, bukan data!
struct AppState {
    konfigurasi: Arc<Konfigurasi>,      // shared, immutable
    senarai_pengguna: Arc<Vec<String>>, // shared, baca sahaja
}

struct Konfigurasi { host: String, port: u16 }

fn main() {
    let state = AppState {
        konfigurasi: Arc::new(Konfigurasi {
            host: "localhost".into(),
            port: 8080,
        }),
        senarai_pengguna: Arc::new(vec!["ali".into(), "siti".into()]),
    };

    // AppState::clone() = clone Arc pointers sahaja = murah!
    let state2 = state.clone();
    let state3 = state.clone();

    println!("{}:{}", state.konfigurasi.host, state.konfigurasi.port);
    // Semua state shares data yang sama dalam memory
}
```

---

## Arc + Mutex untuk Mutable Shared Data

```rust
use std::sync::{Arc, Mutex};

// Bila data perlu berubah dari banyak thread
struct KiraanKongsi {
    nilai: Arc<Mutex<u64>>,
}

impl Clone for KiraanKongsi {
    fn clone(&self) -> Self {
        // Clone Arc (murah), bukan Mutex content (data tidak clone)
        KiraanKongsi { nilai: Arc::clone(&self.nilai) }
    }
}

impl KiraanKongsi {
    fn baru() -> Self { KiraanKongsi { nilai: Arc::new(Mutex::new(0)) } }
    fn tambah(&self) { *self.nilai.lock().unwrap() += 1; }
    fn baca(&self) -> u64 { *self.nilai.lock().unwrap() }
}
```

---

# BAB 6: Alternatif #4 — Cow\<T\> — Clone On Write 🐄

## Clone HANYA Bila Perlu Mengubah

```rust
use std::borrow::Cow;

// Cow<'a, str> boleh jadi:
// - Borrowed(&'a str) → borrow, tiada allocation
// - Owned(String)     → milik sendiri, ada allocation

// ── Guna: Fungsi yang mungkin atau tidak perlu ubah ──────────

fn bersihkan(input: &str) -> Cow<str> {
    if input.contains("  ") {
        // Perlu ubah → Owned (allocation berlaku)
        Cow::Owned(input.replace("  ", " "))
    } else {
        // Tidak perlu ubah → Borrowed (tiada allocation!)
        Cow::Borrowed(input)
    }
}

fn main() {
    let s1 = "hello world";        // tiada double space
    let s2 = "hello  world";       // ada double space

    let r1 = bersihkan(s1); // Cow::Borrowed → tiada allocation!
    let r2 = bersihkan(s2); // Cow::Owned   → allocation berlaku

    println!("{}", r1); // hello world
    println!("{}", r2); // hello world (cleaned)

    // Guna Cow seperti &str
    fn papar(s: &str) { println!("{}", s); }
    papar(&r1); // auto-deref Cow → &str
    papar(&r2);
}

// ── Cow<[T]> untuk Vec ────────────────────────────────────────

fn tambah_kalau_perlu(data: &[i32], tambah: i32) -> Cow<[i32]> {
    if data.contains(&tambah) {
        Cow::Borrowed(data)    // sudah ada, tidak perlu clone!
    } else {
        let mut baru = data.to_vec(); // clone hanya bila perlu
        baru.push(tambah);
        Cow::Owned(baru)
    }
}

fn main() {
    let data = vec![1, 2, 3, 4, 5];

    let r1 = tambah_kalau_perlu(&data, 3); // Borrowed — 3 sudah ada
    let r2 = tambah_kalau_perlu(&data, 6); // Owned — perlu tambah 6

    println!("{:?}", r1); // [1, 2, 3, 4, 5]
    println!("{:?}", r2); // [1, 2, 3, 4, 5, 6]
}
```

---

## Cow dalam Struct

```rust
use std::borrow::Cow;

// Struct yang boleh borrow ATAU own data
struct Rekod<'a> {
    nama: Cow<'a, str>,
    data: Cow<'a, [u8]>,
}

impl<'a> Rekod<'a> {
    // Buat dengan borrow (tiada allocation)
    fn dari_ref(nama: &'a str, data: &'a [u8]) -> Self {
        Rekod {
            nama: Cow::Borrowed(nama),
            data: Cow::Borrowed(data),
        }
    }

    // Buat dengan owned data (ada allocation)
    fn dengan_owned(nama: String, data: Vec<u8>) -> Self {
        Rekod {
            nama: Cow::Owned(nama),
            data: Cow::Owned(data),
        }
    }

    // Ubah nama — akan clone jika masih borrowed
    fn tetapkan_nama(&mut self, nama_baru: &str) {
        self.nama = Cow::Owned(nama_baru.to_string());
    }
}

fn main() {
    let nama = "Ali Ahmad";
    let data = vec![1u8, 2, 3];

    let mut r = Rekod::dari_ref(nama, &data); // tiada allocation
    println!("{}", r.nama); // borrow

    r.tetapkan_nama("Ali Ahmad Bin Omar"); // baru clone
    println!("{}", r.nama); // owned
}
```

---

# BAB 7: Alternatif #5 — Redesign Struktur Data 🏗️

## Elak Clone dengan Split Ownership

```rust
// ❌ Masalah: dua struct perlu data yang sama
struct A {
    nama: String,
    data: Vec<u8>,
}

struct B {
    nama:       String,   // sama dengan A.nama — perlu clone!
    metadata:   String,
}

fn proses(a: &A) -> B {
    B {
        nama:     a.nama.clone(), // ← clone diperlukan
        metadata: "processed".into(),
    }
}

// ✔ Penyelesaian: Guna ID/reference, bukan clone data
struct DataStore {
    data: Vec<ItemData>,  // data sebenar disimpan di sini sahaja
}

struct ItemData {
    nama: String,
    data: Vec<u8>,
}

struct ItemView {
    id:       usize,     // index ke DataStore
    metadata: String,
}

impl DataStore {
    fn dapatkan(&self, id: usize) -> &ItemData {
        &self.data[id]
    }

    fn proses(&self, id: usize) -> ItemView {
        ItemView {
            id,                     // hanya simpan ID, tiada clone data!
            metadata: "processed".into(),
        }
    }
}
```

---

## Extract Shared Data ke Bahagian Berasingan

```rust
// ❌ Setiap Pekerja simpan salinan konfigurasi
struct PekerjaLama {
    nama:      String,
    host_api:  String,  // sama untuk semua pekerja!
    port_api:  u16,     // sama untuk semua pekerja!
    timeout:   u32,     // sama untuk semua pekerja!
}

// ✔ Kongsi konfigurasi melalui Arc
use std::sync::Arc;

struct KonfigApp {
    host_api: String,
    port_api: u16,
    timeout:  u32,
}

struct Pekerja {
    nama:    String,
    konfig:  Arc<KonfigApp>,  // share konfigurasi, tiada clone data!
}

fn main() {
    let konfig = Arc::new(KonfigApp {
        host_api: "api.kada.gov.my".into(),
        port_api: 443,
        timeout:  30,
    });

    let pekerja: Vec<Pekerja> = vec!["Ali", "Siti", "Amin"]
        .into_iter()
        .map(|nama| Pekerja {
            nama:   nama.into(),
            konfig: Arc::clone(&konfig), // murah — hanya clone pointer
        })
        .collect();

    for p in &pekerja {
        println!("{} → {}:{}", p.nama, p.konfig.host_api, p.konfig.port_api);
    }
    // KonfigApp data hanya ADA SATU salinan dalam memory!
}
```

---

# BAB 8: Alternatif #6 — Move Instead of Clone ➡️

## Move Ownership Daripada Clone

```rust
// ❌ Clone untuk "bagi" data ke fungsi
fn proses(data: Vec<u8>) {
    println!("Data: {} bytes", data.len());
}

fn main() {
    let data = vec![0u8; 1000];
    proses(data.clone()); // clone!
    proses(data.clone()); // clone lagi!
    println!("{}", data.len()); // kita masih perlukan data
}

// ✔ Fikir: adakah kita BETUL-BETUL perlu data selepas panggil fungsi?

// Kes A: Ya, perlu kekalkan data → guna &[u8]
fn proses(data: &[u8]) { println!("{} bytes", data.len()); }

// Kes B: Tidak perlu selepas ini → move terus
fn proses(data: Vec<u8>) { println!("{} bytes", data.len()); }

fn main_b() {
    let data = vec![0u8; 1000];
    proses(data); // MOVE — tiada clone!
    // data tidak boleh digunakan lagi — tapi kita tidak perlukan!
}
```

---

## Builder Pattern — Move dalam Chain

```rust
// Builder pattern guna move untuk elak clone
struct PembinaPermintaan {
    url:      String,
    headers:  Vec<(String, String)>,
    body:     Option<String>,
}

impl PembinaPermintaan {
    fn baru(url: &str) -> Self {
        PembinaPermintaan {
            url:     url.into(),
            headers: Vec::new(),
            body:    None,
        }
    }

    // Setiap method consume self dan return self baru
    // Tiada clone diperlukan!
    fn header(mut self, k: &str, v: &str) -> Self {
        self.headers.push((k.into(), v.into()));
        self // move self
    }

    fn body(mut self, b: &str) -> Self {
        self.body = Some(b.into());
        self
    }

    fn bina(self) -> Permintaan {
        Permintaan { url: self.url, headers: self.headers, body: self.body }
    }
}

struct Permintaan { url: String, headers: Vec<(String, String)>, body: Option<String> }

fn main() {
    // Move chain — tiada clone!
    let req = PembinaPermintaan::baru("https://api.kada.gov.my")
        .header("Authorization", "Bearer token")
        .header("Content-Type", "application/json")
        .body(r#"{"id": 1}"#)
        .bina();
}
```

---

# BAB 9: Decision Flowchart 🗺️

## Bila Jumpa "Cannot Move / Borrow"

```
Compiler kata tidak boleh borrow/move? Tanya:

1. FUNGSI/METHOD apa yang panggil?
   └─ Ambil &str? → hantar &string (bukan .clone())
   └─ Ambil &[T]? → hantar &vec (bukan .clone())
   └─ Ambil T (owned)? → teruskan...

2. Adakah kita perlukan nilai asal SELEPAS panggil?
   └─ TIDAK → MOVE terus (bukan clone)
   └─ YA    → teruskan...

3. Berapa mahal clone ini?
   └─ Copy type (i32, bool, f64)?    → clone/copy BEBAS
   └─ Small String ("hello")?        → clone OK, sekali sahaja
   └─ Large Vec/HashMap/nested?      → teruskan...

4. Adakah data dikongsi antara beberapa pemilik?
   └─ YA, multi-thread → guna Arc<T> (clone pointer, bukan data)
   └─ YA, single-thread → guna Rc<T> atau restructure

5. Adakah fungsi kadang-kala ubah, kadang-kala tidak?
   └─ YA → guna Cow<T>

6. Boleh ubah fungsi signature untuk terima reference?
   └─ fn f(s: String) → fn f(s: &str)
   └─ fn f(v: Vec<T>) → fn f(v: &[T])

7. Boleh tambah lifetime parameter?
   └─ struct S { x: String } → struct S<'a> { x: &'a str }

8. Masih perlu clone selepas semua di atas?
   └─ YA → CLONE DAN TERUSKAN. Ini adalah jawapan yang betul!
```

---

## Jadual Rujukan Pantas

```
SITUASI                          PILIHAN TERBAIK
────────────────────────────────────────────────────────────────
Read-only access                 &T atau &str atau &[T]
Fungsi yang mungkin/tidak ubah  Cow<T>
Shared immutable data, threads  Arc<T>
Shared mutable data, threads    Arc<Mutex<T>>
Shared, single thread           Rc<T> atau Rc<RefCell<T>>
Fungsi ambil String/Vec owner   Move (jika tidak perlu lagi)
Small, Copy types                Clone atau Copy terus
Test code                        Clone bebas
Setup/init code                  Clone OK
Prototype                        Clone dulu, optimize kemudian
External API yang ambil owned    Clone atau redesign
Performance critical path        Profile dulu, optimize kemudian
```

---

# BAB 10: Mini Project — Refactor Clone ke Idiomatic 🔧

```rust
// ── VERSI 1: Clone everywhere (beginner) ──────────────────────

use std::collections::HashMap;

#[derive(Clone, Debug)]
struct Pekerja {
    id:       u32,
    nama:     String,
    bahagian: String,
    tag:      Vec<String>,
}

struct SistemV1 {
    pekerja: Vec<Pekerja>,
}

impl SistemV1 {
    fn cari_nama(&self, nama: &str) -> Option<Pekerja> {
        self.pekerja.iter()
            .find(|p| p.nama == nama)
            .map(|p| p.clone()) // ← clone untuk return owned
    }

    fn nama_semua(&self) -> Vec<String> {
        self.pekerja.iter()
            .map(|p| p.nama.clone()) // ← clone dalam loop!
            .collect()
    }

    fn ikut_bahagian(&self) -> HashMap<String, Vec<String>> {
        let mut peta = HashMap::new();
        for p in &self.pekerja {
            peta.entry(p.bahagian.clone())     // ← clone key
                .or_default()
                .push(p.nama.clone());          // ← clone value
        }
        peta
    }

    fn purata_tag_per_pekerja(&self) -> f64 {
        let jumlah_tag: usize = self.pekerja.iter()
            .map(|p| p.tag.clone().len()) // ← clone Vec untuk .len()!
            .sum();
        jumlah_tag as f64 / self.pekerja.len() as f64
    }
}

// ── VERSI 2: Idiomatic (tiada clone yang tidak perlu) ─────────

struct SistemV2 {
    pekerja: Vec<Pekerja>,
}

impl SistemV2 {
    // Return &Pekerja — tiada clone!
    fn cari_nama(&self, nama: &str) -> Option<&Pekerja> {
        self.pekerja.iter().find(|p| p.nama == nama)
        // Caller boleh clone kalau mereka perlukan owned
    }

    // Return &str — tiada clone!
    fn nama_semua(&self) -> Vec<&str> {
        self.pekerja.iter().map(|p| p.nama.as_str()).collect()
    }

    // Key dan value sebagai &str — tiada clone!
    fn ikut_bahagian(&self) -> HashMap<&str, Vec<&str>> {
        let mut peta: HashMap<&str, Vec<&str>> = HashMap::new();
        for p in &self.pekerja {
            peta.entry(&p.bahagian)  // &str — tiada clone
                .or_default()
                .push(&p.nama);      // &str — tiada clone
        }
        peta
    }

    // .len() pada reference — tiada clone!
    fn purata_tag_per_pekerja(&self) -> f64 {
        let jumlah_tag: usize = self.pekerja.iter()
            .map(|p| p.tag.len()) // &Vec.len() — tiada clone!
            .sum();
        jumlah_tag as f64 / self.pekerja.len() as f64
    }

    // Kalau betul-betul perlu owned → clone secara eksplisit di satu tempat
    fn eksport(&self) -> Vec<Pekerja> {
        self.pekerja.clone() // ← clone SATU KALI, jelas dan intentional
    }
}

// ── VERSI 3: Arc untuk sharing ────────────────────────────────

use std::sync::Arc;

struct SistemV3 {
    pekerja: Vec<Arc<Pekerja>>,
}

impl SistemV3 {
    // Return Arc<Pekerja> — clone pointer sahaja, bukan data!
    fn cari_nama(&self, nama: &str) -> Option<Arc<Pekerja>> {
        self.pekerja.iter()
            .find(|p| p.nama == nama)
            .map(|p| Arc::clone(p)) // murah — clone pointer sahaja
    }

    // Boleh beri salinan ke thread lain tanpa clone data
    fn hantar_ke_thread(&self, nama: &str) {
        if let Some(p) = self.cari_nama(nama) {
            // p adalah Arc<Pekerja> — boleh move ke thread
            std::thread::spawn(move || {
                println!("Thread proses: {}", p.nama);
                // p.data tidak di-clone! Hanya pointer dikongsi
            });
        }
    }
}

// ── Demo: Bandingkan ──────────────────────────────────────────

fn main() {
    let data = vec![
        Pekerja {
            id: 1, nama: "Ali Ahmad".into(),
            bahagian: "ICT".into(),
            tag: vec!["rust".into(), "python".into()],
        },
        Pekerja {
            id: 2, nama: "Siti Hawa".into(),
            bahagian: "HR".into(),
            tag: vec!["excel".into()],
        },
        Pekerja {
            id: 3, nama: "Amin Razak".into(),
            bahagian: "ICT".into(),
            tag: vec!["rust".into(), "go".into(), "java".into()],
        },
    ];

    println!("=== Versi Idiomatic ===\n");
    let sistem = SistemV2 { pekerja: data.clone() }; // clone sekali untuk demo

    // cari_nama() — return reference
    if let Some(p) = sistem.cari_nama("Ali Ahmad") {
        println!("Jumpa: {} (bahagian: {})", p.nama, p.bahagian);
        // Caller decide sama ada nak clone atau tidak
        let salinan = p.clone(); // kalau perlu owned, clone di sini
    }

    // nama_semua() — Vec<&str>
    let nama: Vec<&str> = sistem.nama_semua();
    println!("Nama: {:?}", nama);

    // ikut_bahagian() — tiada clone
    let ikut_bah = sistem.ikut_bahagian();
    for (bah, pekerja) in &ikut_bah {
        println!("{}: {:?}", bah, pekerja);
    }

    // Statistik
    println!("Purata tag: {:.1}", sistem.purata_tag_per_pekerja());

    println!("\n=== Perbandingan ===");
    println!("V1: Clone dalam setiap operasi = banyak allocation");
    println!("V2: Reference — tiada clone yang tidak perlu");
    println!("V3: Arc — kongsi data tanpa clone data");
}
```

---

# 📋 Rujukan Pantas

## Clone vs Alternatif

```
BILA CLONE ADALAH JAWAPAN BETUL:
  ✔ Copy types (i32, bool) — hampir free
  ✔ Arc::clone() — hanya pointer (murah!)
  ✔ Setup/init code (sekali sahaja)
  ✔ Test code
  ✔ Prototype (optimize kemudian)
  ✔ API luar yang ambil owned, kita masih perlukan nilai
  ✔ Bila alternatives terlalu kompleks untuk manfaat kecil

ALTERNATIVES DAN BILA GUNA:
  &str / &[T]  → fungsi hanya baca string/slice
  &T           → fungsi hanya baca sebarang data
  Lifetime     → return reference yang hidup selama input
  Arc<T>       → kongsi data antara banyak pemilik/thread
  Cow<T>       → kadang borrow, kadang ubah
  Move         → bila tidak perlu nilai asal lagi
  Redesign     → bila clone symptom of design problem

RED FLAGS (clone menyembunyikan masalah):
  ✗ Clone dalam hot loop (1000+ iterasi)
  ✗ Clone data besar (>1KB) berkali-kali
  ✗ Clone untuk "escape" borrow error dalam struct design
  ✗ Clone kerana parameter salah type (&String bukan &str)
```

## Cara Baca Error Compiler

```
"cannot move out of `x` because it is borrowed"
  → Fikir: perlu data selepas borrow tamat? → borrow lebih lama
  → Atau: clone hanya bahagian yang perlu

"cannot borrow `x` as mutable more than once"
  → Redesign akses pattern
  → Atau: split menjadi dua bahagian berasingan

"value moved here ... use of moved value"
  → Fikir: perlu nilai asal? → guna &T
  → Tidak perlu? → biarkan move, tiada masalah
  → Perlu berkongsi? → Arc<T>
```

---

*Clone adalah alat, bukan musuh.*
*Guna bila sesuai, elak bila ada alternatif yang lebih jelas.*
*Profile sebelum optimize — jangan micro-optimize prematurely.* 🦀
