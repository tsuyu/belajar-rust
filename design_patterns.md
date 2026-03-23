# 🏛️ Design Patterns dalam Rust — Panduan Lengkap

> Rust bukan sahaja terjemah patterns klasik — ia mencipta patterns baru.
> Ownership, traits, dan type system menghasilkan patterns yang tidak wujud
> dalam Java atau PHP.

---

## Rust vs OOP Patterns

```
GOF (Gang of Four) patterns dibina untuk bahasa OOP dengan:
  - Inheritance
  - Runtime polymorphism (virtual dispatch)
  - Mutable shared state
  - Garbage collection

Rust ada:
  - Composition (tiada inheritance)
  - Trait objects (dyn Trait) DAN generics
  - Ownership + Borrowing (tiada shared mutable state secara default)
  - RAII (tiada GC)

Kesannya:
  Sesetengah patterns GOF → TIDAK RELEVAN dalam Rust
  Sesetengah patterns GOF → Terjemah dengan cara berbeza
  Patterns BARU yang hanya mungkin dalam Rust
```

---

## Peta Pembelajaran

```
Bahagian 1  → Builder Pattern
Bahagian 2  → Strategy Pattern
Bahagian 3  → Observer Pattern
Bahagian 4  → Iterator Pattern
Bahagian 5  → Newtype Pattern
Bahagian 6  → Type State Pattern
Bahagian 7  → Command Pattern
Bahagian 8  → State Machine
Bahagian 9  → Extension Trait
Bahagian 10 → Mini Project: Plugin System
```

---

# BAHAGIAN 1: Builder Pattern 🔨

## Masalah yang Diselesaikan

```
Masalah:
  Struct dengan banyak field optional
  Constructor dengan 10+ parameter → sukar baca dan guna
  Berbeza dari PHP: tiada named parameters (sebelum PHP 8)

Rust tiada:
  fn baru(nama: str, gaji: float, bahagian: str = "ICT", ...) // PHP
  new Pekerja(nama, gaji, bahagian=ICT, ...) // Python

Rust mesti:
  Guna Builder pattern untuk optional fields
```

## Cara 1: Builder Mudah

```rust
#[derive(Debug, Clone)]
pub struct RequestHttp {
    url:       String,
    kaedah:    String,
    headers:   Vec<(String, String)>,
    body:      Option<String>,
    timeout:   u64,
    ikut_redirect: bool,
}

pub struct PembinaRequest {
    url:       String,
    kaedah:    String,
    headers:   Vec<(String, String)>,
    body:      Option<String>,
    timeout:   u64,
    ikut_redirect: bool,
}

impl PembinaRequest {
    pub fn baru(url: &str) -> Self {
        PembinaRequest {
            url:           url.into(),
            kaedah:        "GET".into(),
            headers:       Vec::new(),
            body:          None,
            timeout:       30,
            ikut_redirect: true,
        }
    }

    pub fn kaedah(mut self, kaedah: &str) -> Self {
        self.kaedah = kaedah.into(); self
    }

    pub fn header(mut self, kunci: &str, nilai: &str) -> Self {
        self.headers.push((kunci.into(), nilai.into())); self
    }

    pub fn body(mut self, body: &str) -> Self {
        self.body = Some(body.into()); self
    }

    pub fn timeout(mut self, saat: u64) -> Self {
        self.timeout = saat; self
    }

    pub fn bina(self) -> Result<RequestHttp, String> {
        if self.url.is_empty() {
            return Err("URL tidak boleh kosong".into());
        }
        Ok(RequestHttp {
            url:           self.url,
            kaedah:        self.kaedah,
            headers:       self.headers,
            body:          self.body,
            timeout:       self.timeout,
            ikut_redirect: self.ikut_redirect,
        })
    }
}

fn main() {
    let req = PembinaRequest::baru("https://api.kada.gov.my/pekerja")
        .kaedah("POST")
        .header("Authorization", "Bearer token123")
        .header("Content-Type", "application/json")
        .body(r#"{"nama": "Ali"}"#)
        .timeout(60)
        .bina()
        .unwrap();

    println!("{:#?}", req);
}
```

---

## Cara 2: Type-Safe Builder (PhantomData)

```rust
use std::marker::PhantomData;

// States (zero-sized types — tiada memory overhead)
struct TiadaNama;
struct AdaNama;
struct TiadaGaji;
struct AdaGaji;

struct PembinaPekerja<N, G> {
    nama:     Option<String>,
    bahagian: String,
    gaji:     Option<f64>,
    _phantom: PhantomData<(N, G)>,
}

// Initial state — boleh set nama dan gaji
impl PembinaPekerja<TiadaNama, TiadaGaji> {
    fn baru() -> Self {
        PembinaPekerja {
            nama:     None,
            bahagian: "ICT".into(),
            gaji:     None,
            _phantom: PhantomData,
        }
    }
}

// Set nama → state bertukar ke AdaNama
impl<G> PembinaPekerja<TiadaNama, G> {
    fn nama(self, nama: &str) -> PembinaPekerja<AdaNama, G> {
        PembinaPekerja {
            nama:     Some(nama.into()),
            bahagian: self.bahagian,
            gaji:     self.gaji,
            _phantom: PhantomData,
        }
    }
}

// Set gaji → state bertukar ke AdaGaji
impl<N> PembinaPekerja<N, TiadaGaji> {
    fn gaji(self, gaji: f64) -> PembinaPekerja<N, AdaGaji> {
        PembinaPekerja {
            nama:     self.nama,
            bahagian: self.bahagian,
            gaji:     Some(gaji),
            _phantom: PhantomData,
        }
    }
}

// Bahagian boleh set pada bila-bila state
impl<N, G> PembinaPekerja<N, G> {
    fn bahagian(mut self, bahagian: &str) -> Self {
        self.bahagian = bahagian.into(); self
    }
}

// Hanya boleh bina kalau KEDUA-DUA nama dan gaji ada!
#[derive(Debug)]
struct Pekerja { nama: String, bahagian: String, gaji: f64 }

impl PembinaPekerja<AdaNama, AdaGaji> {
    fn bina(self) -> Pekerja {
        Pekerja {
            nama:     self.nama.unwrap(),
            bahagian: self.bahagian,
            gaji:     self.gaji.unwrap(),
        }
    }
}

fn main() {
    // ✔ Berjaya — ada nama dan gaji
    let p = PembinaPekerja::baru()
        .nama("Ali Ahmad")
        .gaji(4500.0)
        .bahagian("ICT")
        .bina();

    println!("{:#?}", p);

    // ✗ ERROR — tiada gaji, tidak boleh .bina()
    // PembinaPekerja::baru()
    //     .nama("Siti")
    //     .bina(); // COMPILE ERROR! method tidak ada untuk state ini
}
```

---

## 🧠 Brain Teaser #1

Apakah kelebihan Type-Safe Builder berbanding Builder biasa?

<details>
<summary>👀 Jawapan</summary>

```
Builder biasa:
  struct PembinaBiasa { nama: Option<String>, gaji: Option<f64> }
  fn bina(self) -> Result<Pekerja, String> {
      let nama = self.nama.ok_or("Nama diperlukan")?;
      let gaji = self.gaji.ok_or("Gaji diperlukan")?;
      Ok(Pekerja { nama, gaji })
  }

  Masalah: Error hanya diketahui pada RUNTIME:
  PembinaBiasa::baru().bina() // Err("Nama diperlukan") — RUNTIME!

Type-Safe Builder (PhantomData):
  Masalah: Error pada COMPILE TIME!
  PembinaPekerja::baru().bina() // ERROR! method .bina() tidak wujud!
                                 // untuk state TiadaNama, TiadaGaji

  Tidak mungkin guna builder dengan cara salah —
  compiler akan menolak.

Kos: Lebih verbose, lebih banyak boilerplate
Manfaat: TIADA runtime error untuk missing fields
         IDE akan suggest field mana yang perlu diisi
         API lebih jelas (state menolong pemahaman)

Sesuai untuk: API yang digunakan ramai, library public, critical code
Tidak perlu untuk: internal code, prototype, data sederhana
```
</details>

---

# BAHAGIAN 2: Strategy Pattern ♟️

## Trait sebagai Strategy

```rust
// Masalah: Algorithm yang boleh ditukar ganti pada runtime
// Contoh: Sort algorithm, compression, validation, pricing

trait StrategiHarga {
    fn kira(&self, harga_asas: f64, kuantiti: u32) -> f64;
    fn nama(&self) -> &str;
}

// Strategi 1: Harga penuh
struct HargaPenuh;
impl StrategiHarga for HargaPenuh {
    fn kira(&self, harga_asas: f64, kuantiti: u32) -> f64 {
        harga_asas * kuantiti as f64
    }
    fn nama(&self) -> &str { "Harga Penuh" }
}

// Strategi 2: Diskaun pukal
struct StrategiDiskaun {
    peratus_diskaun: f64,
    kuantiti_min: u32,
}

impl StrategiHarga for StrategiDiskaun {
    fn kira(&self, harga_asas: f64, kuantiti: u32) -> f64 {
        let jumlah = harga_asas * kuantiti as f64;
        if kuantiti >= self.kuantiti_min {
            jumlah * (1.0 - self.peratus_diskaun / 100.0)
        } else {
            jumlah
        }
    }
    fn nama(&self) -> &str { "Diskaun Pukal" }
}

// Strategi 3: Harga berperingkat
struct HargaBerperingkat {
    peringkat: Vec<(u32, f64)>, // (kuantiti_min, harga_per_unit)
}

impl StrategiHarga for HargaBerperingkat {
    fn kira(&self, harga_asas: f64, kuantiti: u32) -> f64 {
        let harga = self.peringkat.iter()
            .rev()
            .find(|(min, _)| kuantiti >= *min)
            .map(|(_, h)| *h)
            .unwrap_or(harga_asas);
        harga * kuantiti as f64
    }
    fn nama(&self) -> &str { "Harga Berperingkat" }
}

// Context yang menggunakan strategi
struct SistemPesanan {
    strategi: Box<dyn StrategiHarga>,
}

impl SistemPesanan {
    fn baru(strategi: Box<dyn StrategiHarga>) -> Self {
        SistemPesanan { strategi }
    }

    fn tukar_strategi(&mut self, strategi: Box<dyn StrategiHarga>) {
        println!("Strategi ditukar ke: {}", strategi.nama());
        self.strategi = strategi;
    }

    fn kira_jumlah(&self, harga: f64, kuantiti: u32) -> f64 {
        let jumlah = self.strategi.kira(harga, kuantiti);
        println!("[{}] {} unit @ RM{:.2} = RM{:.2}",
            self.strategi.nama(), kuantiti, harga, jumlah);
        jumlah
    }
}

// Dengan Generics (static dispatch — lebih laju, tiada runtime overhead)
struct PengiraHarga<S: StrategiHarga> {
    strategi: S,
}

impl<S: StrategiHarga> PengiraHarga<S> {
    fn kira(&self, harga: f64, kuantiti: u32) -> f64 {
        self.strategi.kira(harga, kuantiti)
    }
}

fn main() {
    let mut sistem = SistemPesanan::baru(Box::new(HargaPenuh));
    sistem.kira_jumlah(10.0, 5);   // RM50.00

    sistem.tukar_strategi(Box::new(StrategiDiskaun {
        peratus_diskaun: 20.0,
        kuantiti_min: 10,
    }));
    sistem.kira_jumlah(10.0, 5);   // RM50.00 (kurang dari min)
    sistem.kira_jumlah(10.0, 15);  // RM120.00 (ada diskaun 20%)

    sistem.tukar_strategi(Box::new(HargaBerperingkat {
        peringkat: vec![(1, 10.0), (10, 8.0), (100, 6.0)],
    }));
    sistem.kira_jumlah(10.0, 50);  // RM400.00 (50 × RM8)
    sistem.kira_jumlah(10.0, 200); // RM1200.00 (200 × RM6)
}
```

---

# BAHAGIAN 3: Observer Pattern 👁️

## Event System dengan Traits

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

// Event types
#[derive(Debug, Clone)]
pub enum Acara {
    PekerjaLog { id: u32, nama: String },
    PekerjaKeluar { id: u32 },
    GajiDikemaskini { id: u32, gaji_baru: f64 },
    AmalaanBaru { id: u32, jenis: String },
}

// Observer trait
pub trait Pemerhati: Send + Sync {
    fn nama(&self) -> &str;
    fn tangani(&self, acara: &Acara);
}

// Event bus
pub struct BusAcara {
    pemerhati: Mutex<Vec<Box<dyn Pemerhati>>>,
}

impl BusAcara {
    pub fn baru() -> Arc<Self> {
        Arc::new(BusAcara { pemerhati: Mutex::new(Vec::new()) })
    }

    pub fn daftar(&self, pemerhati: Box<dyn Pemerhati>) {
        self.pemerhati.lock().unwrap().push(pemerhati);
    }

    pub fn siar(&self, acara: Acara) {
        let list = self.pemerhati.lock().unwrap();
        for p in list.iter() {
            p.tangani(&acara);
        }
    }
}

// Concrete observers
struct PemerhatiLog;
impl Pemerhati for PemerhatiLog {
    fn nama(&self) -> &str { "Log" }
    fn tangani(&self, acara: &Acara) {
        println!("[LOG] {:?}", acara);
    }
}

struct PemerhatiEmel {
    emel_admin: String,
}
impl Pemerhati for PemerhatiEmel {
    fn nama(&self) -> &str { "Emel" }
    fn tangani(&self, acara: &Acara) {
        match acara {
            Acara::PekerjaLog { nama, .. } =>
                println!("[EMEL] → {} Selamat datang, {}!", self.emel_admin, nama),
            Acara::GajiDikemaskini { id, gaji_baru } =>
                println!("[EMEL] → {} Gaji pekerja {} = RM{:.2}", self.emel_admin, id, gaji_baru),
            _ => {}
        }
    }
}

struct PemerhatiAudit {
    rekod: Mutex<Vec<String>>,
}
impl PemerhatiAudit {
    fn baru() -> Self { PemerhatiAudit { rekod: Mutex::new(Vec::new()) } }
    fn senarai(&self) -> Vec<String> { self.rekod.lock().unwrap().clone() }
}
impl Pemerhati for PemerhatiAudit {
    fn nama(&self) -> &str { "Audit" }
    fn tangani(&self, acara: &Acara) {
        let entri = format!("{:?}", acara);
        self.rekod.lock().unwrap().push(entri);
    }
}

fn main() {
    let bas = BusAcara::baru();
    let audit = Arc::new(PemerhatiAudit::baru());

    bas.daftar(Box::new(PemerhatiLog));
    bas.daftar(Box::new(PemerhatiEmel { emel_admin: "admin@kada.gov.my".into() }));

    // Perhatian: PemerhatiAudit perlu Arc supaya boleh akses rekod
    // Guna closure sebagai adapter
    let audit_klon = Arc::clone(&audit);
    struct AuditAdapter(Arc<PemerhatiAudit>);
    impl Pemerhati for AuditAdapter {
        fn nama(&self) -> &str { "AuditAdapter" }
        fn tangani(&self, acara: &Acara) { self.0.tangani(acara); }
    }
    bas.daftar(Box::new(AuditAdapter(audit_klon)));

    // Siar events
    bas.siar(Acara::PekerjaLog { id: 1, nama: "Ali Ahmad".into() });
    bas.siar(Acara::GajiDikemaskini { id: 1, gaji_baru: 5000.0 });
    bas.siar(Acara::PekerjaKeluar { id: 2 });

    println!("\nAudit ({} rekod):", audit.senarai().len());
    for r in audit.senarai() { println!("  {}", r); }
}
```

---

# BAHAGIAN 4: Iterator Pattern 🔁

## Custom Iterator

```rust
// Iterator bukan sekadar pattern — ia adalah trait terbina dalam Rust!
// Tapi menulis custom iterator sendiri adalah pattern yang berguna

struct KiraanFib {
    semasa: u64,
    akan_datang: u64,
    had: u64,
}

impl KiraanFib {
    fn baru(had: u64) -> Self {
        KiraanFib { semasa: 0, akan_datang: 1, had }
    }
}

impl Iterator for KiraanFib {
    type Item = u64;

    fn next(&mut self) -> Option<u64> {
        if self.semasa > self.had { return None; }
        let nilai = self.semasa;
        (self.semasa, self.akan_datang) = (self.akan_datang, self.semasa + self.akan_datang);
        Some(nilai)
    }
}

// Iterator untuk halaman dalam database
struct HalamanDB<T> {
    halaman_semasa: usize,
    saiz_halaman:   usize,
    data:           Vec<T>,
    selesai:        bool,
}

impl<T: Clone> HalamanDB<T> {
    fn baru(data: Vec<T>, saiz: usize) -> Self {
        HalamanDB { halaman_semasa: 0, saiz_halaman: saiz, data, selesai: false }
    }
}

impl<T: Clone> Iterator for HalamanDB<T> {
    type Item = Vec<T>;

    fn next(&mut self) -> Option<Vec<T>> {
        if self.selesai { return None; }
        let mula = self.halaman_semasa * self.saiz_halaman;
        if mula >= self.data.len() { return None; }
        let akhir = (mula + self.saiz_halaman).min(self.data.len());
        self.halaman_semasa += 1;
        if akhir >= self.data.len() { self.selesai = true; }
        Some(self.data[mula..akhir].to_vec())
    }
}

fn main() {
    // Fibonacci iterator
    let fib: Vec<u64> = KiraanFib::baru(100).collect();
    println!("Fibonacci: {:?}", fib);

    // Guna semua kuasa Iterator trait!
    let jumlah: u64 = KiraanFib::baru(100).sum();
    let genap: Vec<u64> = KiraanFib::baru(100).filter(|n| n % 2 == 0).collect();
    println!("Jumlah: {}", jumlah);
    println!("Genap: {:?}", genap);

    // Pagination
    let data: Vec<String> = (1..=25).map(|i| format!("Item {}", i)).collect();
    let halaman = HalamanDB::baru(data, 10);

    for (no, hal) in halaman.enumerate() {
        println!("Halaman {}: {} item — [{}...{}]",
            no + 1, hal.len(), hal.first().unwrap(), hal.last().unwrap());
    }
}
```

---

# BAHAGIAN 5: Newtype Pattern 🔖

## Wrapper untuk Type Safety

```rust
// Masalah: String atau f64 boleh bermakna apa sahaja
// Penyelesaian: Bungkus dalam newtype untuk keselamatan type

// Tanpa Newtype — semua "sama"
fn hantar_emel(kepada: String, dari: String, subjek: String) {
    // Senang tersilap parameter!
    println!("Kepada: {}", kepada);
}
// hantar_emel("Subjek", "kepada@email.com", "dari@email.com") — SILAP tapi compile!

// Dengan Newtype — type safety!
#[derive(Debug, Clone, PartialEq)]
struct AlamatEmel(String);

#[derive(Debug, Clone)]
struct Subjek(String);

impl AlamatEmel {
    fn baru(emel: &str) -> Result<Self, String> {
        if emel.contains('@') && emel.contains('.') {
            Ok(AlamatEmel(emel.into()))
        } else {
            Err(format!("'{}' bukan format emel sah", emel))
        }
    }
    fn nilai(&self) -> &str { &self.0 }
}

fn hantar_emel_selamat(kepada: AlamatEmel, dari: AlamatEmel, subjek: Subjek) {
    println!("Kepada:  {}", kepada.nilai());
    println!("Dari:    {}", dari.nilai());
    println!("Subjek:  {}", subjek.0);
}

// Newtype untuk unit ukuran — elak kesilapan (km vs m)
#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
struct Kilometer(f64);

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
struct Meter(f64);

impl From<Meter> for Kilometer {
    fn from(m: Meter) -> Self { Kilometer(m.0 / 1000.0) }
}

impl From<Kilometer> for Meter {
    fn from(k: Kilometer) -> Self { Meter(k.0 * 1000.0) }
}

// Newtype untuk ID yang berbeza
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct PekerjaId(u32);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct PesananId(u32);

fn cari_pekerja(id: PekerjaId) -> Option<String> {
    if id.0 == 1 { Some("Ali".into()) } else { None }
}

fn main() {
    // Type-safe email
    let kepada = AlamatEmel::baru("ali@email.com").unwrap();
    let dari   = AlamatEmel::baru("admin@email.com").unwrap();
    hantar_emel_selamat(kepada, dari, Subjek("Test".into()));

    // AlamatEmel::baru("bukan-emel"); // Err!

    // Unit conversion yang selamat
    let jarak_km = Kilometer(5.0);
    let jarak_m: Meter = jarak_km.into();
    println!("{:?} = {:?}", jarak_km, jarak_m);

    // Tidak boleh campur ID
    let id_pekerja = PekerjaId(1);
    // let id_pesanan = PesananId(1);
    // cari_pekerja(id_pesanan); // COMPILE ERROR!
    cari_pekerja(id_pekerja);
}
```

---

# BAHAGIAN 6: Type State Pattern 🔄

## State Transitions yang Selamat

```rust
use std::marker::PhantomData;

// Keadaan sebagai zero-sized types
struct Draf;
struct Semak;
struct Terbit;

// Dokumen dengan state
struct Dokumen<S> {
    id:       u32,
    tajuk:    String,
    kandungan: String,
    _keadaan: PhantomData<S>,
}

// Operasi pada Draf
impl Dokumen<Draf> {
    fn baru(id: u32, tajuk: &str) -> Self {
        Dokumen {
            id, tajuk: tajuk.into(), kandungan: String::new(), _keadaan: PhantomData,
        }
    }

    fn tulis(&mut self, kandungan: &str) {
        self.kandungan.push_str(kandungan);
    }

    fn hantar_semak(self) -> Dokumen<Semak> {
        println!("'{}' dihantar untuk semakan", self.tajuk);
        Dokumen { id: self.id, tajuk: self.tajuk, kandungan: self.kandungan, _keadaan: PhantomData }
    }
}

// Operasi pada Semak
impl Dokumen<Semak> {
    fn luluskan(self) -> Dokumen<Terbit> {
        println!("'{}' diluluskan untuk terbit", self.tajuk);
        Dokumen { id: self.id, tajuk: self.tajuk, kandungan: self.kandungan, _keadaan: PhantomData }
    }

    fn tolak(self) -> Dokumen<Draf> {
        println!("'{}' dikembalikan ke draf", self.tajuk);
        Dokumen { id: self.id, tajuk: self.tajuk, kandungan: self.kandungan, _keadaan: PhantomData }
    }
}

// Operasi pada Terbit
impl Dokumen<Terbit> {
    fn papar(&self) -> String {
        format!("📄 {} ({})\n{}", self.tajuk, self.id, self.kandungan)
    }

    fn arkib(self) {
        println!("'{}' diarkibkan", self.tajuk);
        // Dokumen habis di sini — tidak boleh guna lagi
    }
}

fn main() {
    let mut draf = Dokumen::<Draf>::baru(1, "Dasar KADA 2024");
    draf.tulis("Kandungan dasar yang penting...");

    // Tidak boleh terbit terus — mesti semak dulu!
    // draf.luluskan(); // COMPILE ERROR! method tiada untuk Draf

    let semak = draf.hantar_semak();

    // Tidak boleh tulis semasa dalam semakan!
    // semak.tulis("..."); // COMPILE ERROR! method tiada untuk Semak

    let terbit = semak.luluskan();
    println!("{}", terbit.papar());

    terbit.arkib();
    // terbit.papar(); // COMPILE ERROR! moved!
}
```

---

# BAHAGIAN 7: Command Pattern 📋

## Operasi sebagai Objek

```rust
use std::collections::VecDeque;

// Command trait
trait Arahan: std::fmt::Debug {
    fn laksana(&mut self, teks: &mut String);
    fn batal(&mut self, teks: &mut String);
}

// Concrete commands
#[derive(Debug)]
struct TambahTeks {
    teks_ditambah: String,
}

impl Arahan for TambahTeks {
    fn laksana(&mut self, teks: &mut String) {
        teks.push_str(&self.teks_ditambah);
    }
    fn batal(&mut self, teks: &mut String) {
        let panjang = teks.len() - self.teks_ditambah.len();
        teks.truncate(panjang);
    }
}

#[derive(Debug)]
struct PadamKarakter {
    dipadamkan: Option<char>,
}

impl Arahan for PadamKarakter {
    fn laksana(&mut self, teks: &mut String) {
        self.dipadamkan = teks.pop();
    }
    fn batal(&mut self, teks: &mut String) {
        if let Some(c) = self.dipadamkan {
            teks.push(c);
        }
    }
}

#[derive(Debug)]
struct GantiTeks {
    cari: String,
    ganti: String,
    berlaku: bool,
}

impl Arahan for GantiTeks {
    fn laksana(&mut self, teks: &mut String) {
        if teks.contains(&self.cari) {
            *teks = teks.replace(&self.cari, &self.ganti);
            self.berlaku = true;
        }
    }
    fn batal(&mut self, teks: &mut String) {
        if self.berlaku {
            *teks = teks.replace(&self.ganti, &self.cari);
        }
    }
}

// Editor dengan history (undo/redo)
struct Editor {
    kandungan: String,
    sejarah:   Vec<Box<dyn Arahan>>,
    dibatal:   Vec<Box<dyn Arahan>>,
}

impl Editor {
    fn baru() -> Self {
        Editor { kandungan: String::new(), sejarah: Vec::new(), dibatal: Vec::new() }
    }

    fn laksana(&mut self, mut arahan: Box<dyn Arahan>) {
        arahan.laksana(&mut self.kandungan);
        self.sejarah.push(arahan);
        self.dibatal.clear(); // clear redo stack
    }

    fn batal(&mut self) -> bool {
        if let Some(mut arahan) = self.sejarah.pop() {
            arahan.batal(&mut self.kandungan);
            self.dibatal.push(arahan);
            true
        } else { false }
    }

    fn buat_semula(&mut self) -> bool {
        if let Some(mut arahan) = self.dibatal.pop() {
            arahan.laksana(&mut self.kandungan);
            self.sejarah.push(arahan);
            true
        } else { false }
    }

    fn kandungan(&self) -> &str { &self.kandungan }
}

fn main() {
    let mut editor = Editor::baru();

    editor.laksana(Box::new(TambahTeks { teks_ditambah: "Hello, ".into() }));
    editor.laksana(Box::new(TambahTeks { teks_ditambah: "World!".into() }));
    println!("{}", editor.kandungan()); // "Hello, World!"

    editor.laksana(Box::new(GantiTeks {
        cari: "World".into(), ganti: "Rust".into(), berlaku: false,
    }));
    println!("{}", editor.kandungan()); // "Hello, Rust!"

    editor.batal();
    println!("{}", editor.kandungan()); // "Hello, World!"

    editor.buat_semula();
    println!("{}", editor.kandungan()); // "Hello, Rust!"
}
```

---

# BAHAGIAN 8: State Machine 🔀

## Enum-based State Machine

```rust
// Rust idiom: guna enum untuk state machine
// Lebih idiomatic daripada GOF State pattern!

#[derive(Debug, Clone, PartialEq)]
enum KeadaanPesanan {
    Baru,
    Dibayar { rujukan: String },
    Diproses { pengendali: String },
    Dihantar { nombor_penjejak: String },
    Sampai,
    Dibatal { sebab: String },
}

#[derive(Debug)]
struct Pesanan {
    id:       u32,
    butiran:  String,
    keadaan:  KeadaanPesanan,
}

#[derive(Debug)]
enum TransisiPesanan {
    Bayar { rujukan: String },
    Proses { pengendali: String },
    Hantar { penjejak: String },
    TandaSampai,
    Batal { sebab: String },
}

#[derive(Debug)]
enum RalatTransisi {
    TransisiTidakSah {
        dari: String,
        ke:   String,
    },
}

impl Pesanan {
    fn baru(id: u32, butiran: &str) -> Self {
        Pesanan { id, butiran: butiran.into(), keadaan: KeadaanPesanan::Baru }
    }

    fn transisi(&mut self, t: TransisiPesanan) -> Result<(), RalatTransisi> {
        let keadaan_baru = match (&self.keadaan, t) {
            // Dari Baru
            (KeadaanPesanan::Baru, TransisiPesanan::Bayar { rujukan }) =>
                KeadaanPesanan::Dibayar { rujukan },
            (KeadaanPesanan::Baru, TransisiPesanan::Batal { sebab }) =>
                KeadaanPesanan::Dibatal { sebab },

            // Dari Dibayar
            (KeadaanPesanan::Dibayar { .. }, TransisiPesanan::Proses { pengendali }) =>
                KeadaanPesanan::Diproses { pengendali },
            (KeadaanPesanan::Dibayar { .. }, TransisiPesanan::Batal { sebab }) =>
                KeadaanPesanan::Dibatal { sebab },

            // Dari Diproses
            (KeadaanPesanan::Diproses { .. }, TransisiPesanan::Hantar { penjejak }) =>
                KeadaanPesanan::Dihantar { nombor_penjejak: penjejak },

            // Dari Dihantar
            (KeadaanPesanan::Dihantar { .. }, TransisiPesanan::TandaSampai) =>
                KeadaanPesanan::Sampai,

            // Semua transisi lain tidak sah
            (dari, transisi) => {
                return Err(RalatTransisi::TransisiTidakSah {
                    dari: format!("{:?}", dari),
                    ke:   format!("{:?}", transisi),
                });
            }
        };

        println!("[Pesanan #{}] {:?} → {:?}", self.id, self.keadaan, keadaan_baru);
        self.keadaan = keadaan_baru;
        Ok(())
    }

    fn boleh_dibatal(&self) -> bool {
        matches!(self.keadaan, KeadaanPesanan::Baru | KeadaanPesanan::Dibayar { .. })
    }
}

fn main() {
    let mut p = Pesanan::baru(1, "Laptop KADA");

    p.transisi(TransisiPesanan::Bayar { rujukan: "PAY-123".into() }).unwrap();
    p.transisi(TransisiPesanan::Proses { pengendali: "Ali".into() }).unwrap();
    p.transisi(TransisiPesanan::Hantar { penjejak: "TRK-456".into() }).unwrap();
    p.transisi(TransisiPesanan::TandaSampai).unwrap();

    // Transisi tidak sah
    match p.transisi(TransisiPesanan::Batal { sebab: "Tidak mahu".into() }) {
        Err(RalatTransisi::TransisiTidakSah { dari, ke }) =>
            println!("✗ Tidak boleh dari {} dengan {:?}", dari, ke),
        _ => {}
    }
}
```

---

# BAHAGIAN 9: Extension Trait 🔌

## Tambah Method ke Type Sedia Ada

```rust
// Masalah: Nak tambah method ke Vec, String, i32 dari stdlib
// Rust tidak boleh guna inheritance, tapi ada extension traits!

// Extension trait untuk Vec
pub trait VecExtensions<T> {
    fn bahagi_sama(&self, n: usize) -> Vec<Vec<&T>>;
    fn kedua_dua<F: Fn(&T) -> bool>(&self, predikat: F) -> bool;
    fn tiada<F: Fn(&T) -> bool>(&self, predikat: F) -> bool;
    fn statistik(&self) -> Option<(&T, &T)> where T: Ord;
}

impl<T> VecExtensions<T> for Vec<T> {
    fn bahagi_sama(&self, n: usize) -> Vec<Vec<&T>> {
        self.chunks(n).map(|c| c.iter().collect()).collect()
    }

    fn kedua_dua<F: Fn(&T) -> bool>(&self, predikat: F) -> bool {
        self.iter().all(predikat)
    }

    fn tiada<F: Fn(&T) -> bool>(&self, predikat: F) -> bool {
        !self.iter().any(predikat)
    }

    fn statistik(&self) -> Option<(&T, &T)> where T: Ord {
        if self.is_empty() { return None; }
        let min = self.iter().min().unwrap();
        let max = self.iter().max().unwrap();
        Some((min, max))
    }
}

// Extension trait untuk String
pub trait StringExtensions {
    fn adalah_nombor(&self) -> bool;
    fn potong_kiri(&self, n: usize) -> &str;
    fn potong_kanan(&self, n: usize) -> &str;
    fn perkataan(&self) -> Vec<&str>;
    fn mask_emel(&self) -> String;
}

impl StringExtensions for str {
    fn adalah_nombor(&self) -> bool {
        !self.is_empty() && self.chars().all(|c| c.is_ascii_digit())
    }

    fn potong_kiri(&self, n: usize) -> &str {
        let n = n.min(self.len());
        &self[..n]
    }

    fn potong_kanan(&self, n: usize) -> &str {
        let panjang = self.len();
        let mula = panjang.saturating_sub(n);
        &self[mula..]
    }

    fn perkataan(&self) -> Vec<&str> {
        self.split_whitespace().collect()
    }

    fn mask_emel(&self) -> String {
        if let Some(pos) = self.find('@') {
            let (nama, domain) = self.split_at(pos);
            if nama.len() <= 2 {
                return format!("{}***{}", &nama[..1], domain);
            }
            format!("{}***{}", &nama[..2], domain)
        } else {
            "*".repeat(self.len())
        }
    }
}

// Extension untuk i32
pub trait IntExtensions {
    fn dalam_julat(&self, min: i32, max: i32) -> bool;
    fn cerita(&self) -> String;
}

impl IntExtensions for i32 {
    fn dalam_julat(&self, min: i32, max: i32) -> bool {
        *self >= min && *self <= max
    }

    fn cerita(&self) -> String {
        match *self {
            0          => "sifar".into(),
            1..=9      => ["satu","dua","tiga","empat","lima","enam","tujuh","lapan","sembilan"][*self as usize - 1].into(),
            10         => "sepuluh".into(),
            _          => format!("{}", self),
        }
    }
}

fn main() {
    let v = vec![3, 1, 4, 1, 5, 9, 2, 6];

    // Vec extensions
    let bahagian = v.bahagi_sama(3);
    println!("Bahagian: {:?}", bahagian);

    if let Some((min, max)) = v.statistik() {
        println!("Min: {}, Max: {}", min, max);
    }

    println!("Semua positif: {}", v.kedua_dua(|&n| n > 0));
    println!("Tiada negatif: {}", v.tiada(|&n| n < 0));

    // String extensions
    let emel = "ali.ahmad@email.com";
    println!("{}", emel.mask_emel()); // "al***@email.com"

    let nombor = "12345";
    println!("Adalah nombor: {}", nombor.adalah_nombor()); // true

    let ayat = "Ini adalah ayat";
    println!("Perkataan: {:?}", ayat.perkataan());
    println!("Kiri 3: '{}'", ayat.potong_kiri(3));
    println!("Kanan 4: '{}'", ayat.potong_kanan(4));

    // Int extensions
    println!("5 dalam julat 1-10: {}", 5i32.dalam_julat(1, 10));
    println!("{}", 7i32.cerita()); // "tujuh"
}
```

---

# BAHAGIAN 10: Mini Project — Plugin System 🔌

```rust
use std::collections::HashMap;
use std::sync::Arc;

// ─── Plugin API ─────────────────────────────────────────────

pub trait Plugin: Send + Sync {
    fn nama(&self) -> &str;
    fn versi(&self) -> &str;
    fn perihal(&self) -> &str;
    fn boleh_tangani(&self, jenis: &str) -> bool;
    fn proses(&self, input: &str, tetapan: &HashMap<String, String>) -> Result<String, String>;
}

// ─── Registry ────────────────────────────────────────────────

pub struct RegistriPlugin {
    plugins: HashMap<String, Arc<dyn Plugin>>,
}

impl RegistriPlugin {
    pub fn baru() -> Self {
        RegistriPlugin { plugins: HashMap::new() }
    }

    pub fn daftar(&mut self, plugin: Arc<dyn Plugin>) {
        println!("Plugin '{}' v{} didaftarkan", plugin.nama(), plugin.versi());
        self.plugins.insert(plugin.nama().to_string(), plugin);
    }

    pub fn dapatkan(&self, nama: &str) -> Option<&Arc<dyn Plugin>> {
        self.plugins.get(nama)
    }

    pub fn cari_untuk(&self, jenis: &str) -> Vec<&Arc<dyn Plugin>> {
        self.plugins.values()
            .filter(|p| p.boleh_tangani(jenis))
            .collect()
    }

    pub fn proses_dengan(
        &self,
        nama_plugin: &str,
        input: &str,
        tetapan: &HashMap<String, String>,
    ) -> Result<String, String> {
        self.dapatkan(nama_plugin)
            .ok_or_else(|| format!("Plugin '{}' tidak dijumpai", nama_plugin))?
            .proses(input, tetapan)
    }

    pub fn senarai(&self) -> Vec<String> {
        let mut list: Vec<String> = self.plugins.values()
            .map(|p| format!("{} v{} — {}", p.nama(), p.versi(), p.perihal()))
            .collect();
        list.sort();
        list
    }
}

// ─── Concrete Plugins ─────────────────────────────────────────

struct PluginBesarkan;
impl Plugin for PluginBesarkan {
    fn nama(&self) -> &str { "besarkan" }
    fn versi(&self) -> &str { "1.0.0" }
    fn perihal(&self) -> &str { "Tukar teks ke huruf besar" }
    fn boleh_tangani(&self, jenis: &str) -> bool { jenis == "teks" }
    fn proses(&self, input: &str, _: &HashMap<String, String>) -> Result<String, String> {
        Ok(input.to_uppercase())
    }
}

struct PluginPotong;
impl Plugin for PluginPotong {
    fn nama(&self) -> &str { "potong" }
    fn versi(&self) -> &str { "1.2.0" }
    fn perihal(&self) -> &str { "Potong teks kepada panjang tertentu" }
    fn boleh_tangani(&self, jenis: &str) -> bool { jenis == "teks" }
    fn proses(&self, input: &str, tetapan: &HashMap<String, String>) -> Result<String, String> {
        let panjang: usize = tetapan.get("panjang")
            .and_then(|s| s.parse().ok())
            .unwrap_or(50);
        if input.len() <= panjang {
            Ok(input.to_string())
        } else {
            Ok(format!("{}...", &input[..panjang]))
        }
    }
}

struct PluginHitungKata;
impl Plugin for PluginHitungKata {
    fn nama(&self) -> &str { "hitung-kata" }
    fn versi(&self) -> &str { "1.0.0" }
    fn perihal(&self) -> &str { "Hitung bilangan kata dalam teks" }
    fn boleh_tangani(&self, jenis: &str) -> bool { matches!(jenis, "teks" | "analitik") }
    fn proses(&self, input: &str, _: &HashMap<String, String>) -> Result<String, String> {
        let kata = input.split_whitespace().count();
        let aksara = input.chars().count();
        let baris = input.lines().count();
        Ok(format!("Kata: {}, Aksara: {}, Baris: {}", kata, aksara, baris))
    }
}

struct PluginJSONFormat;
impl Plugin for PluginJSONFormat {
    fn nama(&self) -> &str { "json-format" }
    fn versi(&self) -> &str { "2.0.1" }
    fn perihal(&self) -> &str { "Format dan sahkan JSON" }
    fn boleh_tangani(&self, jenis: &str) -> bool { jenis == "json" }
    fn proses(&self, input: &str, tetapan: &HashMap<String, String>) -> Result<String, String> {
        let nilai: serde_json::Value = serde_json::from_str(input)
            .map_err(|e| format!("JSON tidak sah: {}", e))?;
        let cantik = tetapan.get("cantik").map_or(false, |v| v == "true");
        if cantik {
            Ok(serde_json::to_string_pretty(&nilai).unwrap())
        } else {
            Ok(serde_json::to_string(&nilai).unwrap())
        }
    }
}

// ─── Pipeline ────────────────────────────────────────────────

struct PipelinePlugin<'a> {
    registry: &'a RegistriPlugin,
    langkah:  Vec<(String, HashMap<String, String>)>,
}

impl<'a> PipelinePlugin<'a> {
    fn baru(registry: &'a RegistriPlugin) -> Self {
        PipelinePlugin { registry, langkah: Vec::new() }
    }

    fn tambah(mut self, nama: &str, tetapan: HashMap<String, String>) -> Self {
        self.langkah.push((nama.into(), tetapan));
        self
    }

    fn laksana(&self, input: &str) -> Result<String, String> {
        let mut data = input.to_string();
        for (nama, tetapan) in &self.langkah {
            println!("  → Plugin '{}'", nama);
            data = self.registry.proses_dengan(nama, &data, tetapan)?;
        }
        Ok(data)
    }
}

// ─── Demo ──────────────────────────────────────────────────

fn main() {
    println!("{'═'*55}");
    println!("{:^55}", "KADA Plugin System");
    println!("{'═'*55}\n");

    let mut registry = RegistriPlugin::baru();

    registry.daftar(Arc::new(PluginBesarkan));
    registry.daftar(Arc::new(PluginPotong));
    registry.daftar(Arc::new(PluginHitungKata));
    registry.daftar(Arc::new(PluginJSONFormat));

    println!("\n📦 Plugin tersedia:");
    for p in registry.senarai() { println!("  {}", p); }

    // Guna plugin tunggal
    println!("\n🔧 Guna plugin tunggal:");
    let hasil = registry.proses_dengan(
        "besarkan", "hello dunia", &HashMap::new()
    ).unwrap();
    println!("  Hasil: {}", hasil);

    // Cari plugin yang boleh tangani "teks"
    println!("\n🔍 Plugin untuk 'teks':");
    for p in registry.cari_untuk("teks") {
        println!("  - {}", p.nama());
    }

    // Pipeline
    println!("\n⛓️ Pipeline:");
    let pipeline = PipelinePlugin::baru(&registry)
        .tambah("besarkan", HashMap::new())
        .tambah("potong", [("panjang".into(), "20".into())].into());

    let output = pipeline.laksana("Ini adalah teks yang agak panjang untuk diproses").unwrap();
    println!("  Output: {}", output);

    // JSON plugin
    println!("\n📄 JSON formatting:");
    let json_tetapan: HashMap<String, String> = [("cantik".into(), "true".into())].into();
    let output_json = registry.proses_dengan(
        "json-format",
        r#"{"nama":"Ali","gaji":4500}"#,
        &json_tetapan
    ).unwrap();
    println!("{}", output_json);
}
```

---

# 📋 Rujukan Pantas — Design Patterns

## Pattern yang Paling Berguna dalam Rust

```
Pattern               Cara Rust               Guna Bila
───────────────────────────────────────────────────────────────────
Builder               struct + impl + chain   Banyak optional fields
Strategy              trait + dyn/generic     Algorithm tukar-ganti
Observer/Event        trait + Vec<Box<dyn>>   Loose coupling events
Iterator              impl Iterator           Custom traversal
Newtype               struct Nama(T)          Type safety, unit
Type State            PhantomData states      Enforce workflow
Command               trait + Vec history     Undo/redo
State Machine         enum + match            Complex state
Extension Trait       trait + blanket impl    Tambah method ke type
Plugin System         trait + HashMap         Runtime plugin
RAII                  impl Drop               Resource cleanup
```

## Pattern yang TIDAK PERLU dalam Rust

```
Singleton:
  Guna lazy_static! atau OnceLock<T>
  (tiada perlu class-based singleton)

Prototype:
  Guna #[derive(Clone)]
  (tiada perlu clone() manual)

Factory Method:
  Guna associated functions fn baru() -> Self
  (tidak perlu inheritance hierarchy)

Adapter:
  Guna impl Trait for ExistingType
  (wrapper newtype + impl)

Facade:
  Guna module + pub re-export
  (mod system is the facade)
```

---

*Design patterns dalam Rust bukan terjemahan Java.*
*Ownership, traits, dan type system menghasilkan patterns yang lebih selamat.*
*Learn the Rust way, not the OOP way.* 🦀
