# 🧪 Unit Test dalam Rust — Panduan Lengkap

> Rust mempunyai test framework built-in yang sangat berkuasa.
> Dari unit test mudah hingga integration test, async test,
> dan property-based testing.

---

## Kenapa Test dalam Rust Berbeza?

```
Bahasa lain: Test adalah crate/library berasingan
  PHP:    PHPUnit (composer require --dev)
  Python: pytest (pip install)
  Java:   JUnit (maven/gradle dependency)

Rust: Test adalah BUILT-IN — tiada perlu install apa-apa!
  cargo test  ← satu arahan, semua test jalankan

Tambahan:
  ✔ #[test] attribute — tandakan fungsi sebagai test
  ✔ #[should_panic] — test yang perlu panic
  ✔ #[ignore] — skip test tertentu
  ✔ doc tests — test dalam dokumentasi!
  ✔ integration tests — dalam folder /tests
  ✔ benchmark — dalam folder /benches (nightly)
```

---

## Peta Pembelajaran

```
Bahagian 1  → Test Asas & Assertions
Bahagian 2  → Organisasi Test
Bahagian 3  → Test Module & Visibility
Bahagian 4  → Setup & Teardown
Bahagian 5  → Test dengan Error (Result)
Bahagian 6  → Async Tests
Bahagian 7  → Integration Tests
Bahagian 8  → Doc Tests
Bahagian 9  → Mocking & Test Doubles
Bahagian 10 → Mini Project: TDD KADA Service
```

---

# BAHAGIAN 1: Test Asas & Assertions 🧪

## Test Paling Mudah

```rust
// src/lib.rs atau mana-mana fail .rs

fn tambah(a: i32, b: i32) -> i32 {
    a + b
}

fn bahagi(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 { None } else { Some(a / b) }
}

// Test module — konvensyen standard Rust
#[cfg(test)]          // hanya compile semasa `cargo test`
mod tests {
    use super::*;     // import semua dari parent module

    #[test]           // tandakan sebagai test function
    fn test_tambah_asas() {
        let hasil = tambah(2, 3);
        assert_eq!(hasil, 5);
    }

    #[test]
    fn test_tambah_negatif() {
        assert_eq!(tambah(-2, 3), 1);
        assert_eq!(tambah(-2, -3), -5);
    }

    #[test]
    fn test_bahagi_normal() {
        let hasil = bahagi(10.0, 2.0);
        assert_eq!(hasil, Some(5.0));
    }

    #[test]
    fn test_bahagi_dengan_sifar() {
        let hasil = bahagi(10.0, 0.0);
        assert_eq!(hasil, None);
    }
}
```

---

## Semua Assertion Macros

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn demo_semua_assertions() {
        let a = 5;
        let b = 10;
        let v = vec![1, 2, 3];
        let nama = "Ali Ahmad";

        // ── assert! — boolean condition ───────────────────────
        assert!(a > 0);
        assert!(v.len() == 3);
        assert!(!v.is_empty());

        // Dengan mesej custom:
        assert!(a > 0, "a mestilah positif, dapat: {}", a);

        // ── assert_eq! — equal ────────────────────────────────
        assert_eq!(a, 5);
        assert_eq!(v.len(), 3);
        assert_eq!("hello".to_uppercase(), "HELLO");

        // Dengan mesej:
        assert_eq!(a, 5, "a mestilah 5, dapat: {}", a);

        // ── assert_ne! — not equal ────────────────────────────
        assert_ne!(a, b);
        assert_ne!(v.len(), 0);

        // ── Perbandingan nombor float ─────────────────────────
        // JANGAN guna assert_eq! untuk float!
        let f = 0.1 + 0.2;
        // assert_eq!(f, 0.3); // GAGAL! 0.30000000000000004

        // Cara betul:
        let epsilon = 1e-10;
        assert!((f - 0.3).abs() < epsilon, "Float tidak sama: {} != 0.3", f);

        // ── assert dengan Pattern ─────────────────────────────
        let opt: Option<i32> = Some(42);
        assert!(opt.is_some());
        assert_eq!(opt.unwrap(), 42);

        // Pattern matching dalam assert:
        assert!(matches!(opt, Some(n) if n > 0));
        assert!(matches!(v.first(), Some(&1)));

        // ── String assertions ─────────────────────────────────
        assert!(nama.contains("Ali"));
        assert!(nama.starts_with("Ali"));
        assert!(nama.ends_with("Ahmad"));
        assert_eq!(nama.len(), 9);
    }

    // ── should_panic ──────────────────────────────────────────
    #[test]
    #[should_panic]
    fn test_yang_patut_panic() {
        let v: Vec<i32> = vec![];
        let _ = v[0]; // panic! index out of bounds
    }

    #[test]
    #[should_panic(expected = "bahagi dengan sifar")]
    fn test_panic_dengan_mesej() {
        panic!("ralat: bahagi dengan sifar");
        // expected = substring yang perlu ada dalam panic message
    }

    // ── ignore ────────────────────────────────────────────────
    #[test]
    #[ignore = "Lambat — jalankan hanya bila perlu: cargo test -- --ignored"]
    fn test_yang_lambat() {
        std::thread::sleep(std::time::Duration::from_secs(5));
        assert!(true);
    }
}
```

---

## Jalankan Test

```bash
# Jalankan semua test
cargo test

# Jalankan test tertentu (nama mengandungi "tambah")
cargo test tambah

# Jalankan test dalam modul tertentu
cargo test tests::

# Jalankan test yang di-ignore
cargo test -- --ignored

# Jalankan satu thread (untuk test yang tidak thread-safe)
cargo test -- --test-threads=1

# Tunjukkan output println! dalam test (biasanya disembunyi)
cargo test -- --nocapture

# Jalankan dan tunjukkan semua
cargo test -- --nocapture --test-threads=1

# Output terperinci
cargo test -- --test-threads=1 -q  # quiet
cargo test -- --test-threads=1 -v  # verbose (show semua test)

# Test integration sahaja
cargo test --test nama_fail

# Test unit sahaja (exclude integration)
cargo test --lib
```

---

## 🧠 Brain Teaser #1

Kenapa test ini mungkin gagal walaupun logik kelihatan betul?

```rust
#[test]
fn test_float_sama() {
    let hasil = (0.1 + 0.2) * 3.0;
    let dijangka = 0.9;
    assert_eq!(hasil, dijangka);
}
```

<details>
<summary>👀 Jawapan</summary>

```
GAGAL kerana floating point precision!
(0.1 + 0.2) = 0.30000000000000004 (bukan 0.3 tepat)
(0.30000000000000004) * 3.0 = 0.9000000000000001

Cara betul:
#[test]
fn test_float_sama_betul() {
    let hasil = (0.1 + 0.2) * 3.0;
    let dijangka = 0.9;
    let epsilon = 1e-10;

    assert!(
        (hasil - dijangka).abs() < epsilon,
        "Dijangka {}, dapat {}",
        dijangka, hasil
    );
}

Atau guna crate `approx`:
use approx::assert_relative_eq;
assert_relative_eq!(hasil, dijangka, epsilon = 1e-10);

Atau guna Decimal (rust_decimal):
// Decimal tidak ada floating point issue!
use rust_decimal_macros::dec;
assert_eq!(dec!(0.1) + dec!(0.2), dec!(0.3)); // BERJAYA!

Pelajaran: Jangan guna == untuk float dalam test!
```
</details>

---

# BAHAGIAN 2: Organisasi Test 📁

## Konvensyen Fail & Modul

```
projek/
├── src/
│   ├── lib.rs              ← unit tests di sini (#[cfg(test)] mod tests)
│   ├── main.rs             ← unit tests juga boleh di sini
│   ├── pekerja.rs          ← unit tests dalam fail yang sama
│   ├── kehadiran.rs
│   └── utils.rs
├── tests/                  ← integration tests (akses crate seperti pengguna)
│   ├── test_pekerja.rs
│   ├── test_kehadiran.rs
│   └── common/
│       └── mod.rs          ← shared test helpers
└── benches/                ← benchmarks (nightly sahaja)
    └── bench_main.rs
```

## Unit Test dalam Fail yang Sama

```rust
// src/pekerja.rs

#[derive(Debug, Clone, PartialEq)]
pub struct Pekerja {
    pub id:       u32,
    pub nama:     String,
    pub gaji:     f64,
    pub aktif:    bool,
}

impl Pekerja {
    pub fn baru(id: u32, nama: &str, gaji: f64) -> Self {
        Pekerja { id, nama: nama.into(), gaji, aktif: true }
    }

    pub fn gaji_dengan_naik(&self, peratus: f64) -> f64 {
        self.gaji * (1.0 + peratus / 100.0)
    }

    pub fn adalah_layak_bonus(&self) -> bool {
        self.aktif && self.gaji < 5000.0
    }
}

// Unit tests untuk Pekerja — DALAM fail yang sama
#[cfg(test)]
mod tests {
    use super::*;

    // ── Helper untuk buat data test ──────────────────────────
    fn pekerja_contoh() -> Pekerja {
        Pekerja::baru(1, "Ali Ahmad", 4500.0)
    }

    // ── Test asas ─────────────────────────────────────────────
    #[test]
    fn test_buat_pekerja() {
        let p = pekerja_contoh();
        assert_eq!(p.id, 1);
        assert_eq!(p.nama, "Ali Ahmad");
        assert_eq!(p.gaji, 4500.0);
        assert!(p.aktif);
    }

    #[test]
    fn test_gaji_naik_10_peratus() {
        let p = pekerja_contoh();
        let gaji_baru = p.gaji_dengan_naik(10.0);
        assert_eq!(gaji_baru, 4950.0);
    }

    #[test]
    fn test_gaji_naik_sifar_peratus() {
        let p = pekerja_contoh();
        assert_eq!(p.gaji_dengan_naik(0.0), p.gaji);
    }

    #[test]
    fn test_layak_bonus_aktif_gaji_rendah() {
        let p = pekerja_contoh(); // gaji 4500, aktif
        assert!(p.adalah_layak_bonus());
    }

    #[test]
    fn test_tidak_layak_bonus_tidak_aktif() {
        let mut p = pekerja_contoh();
        p.aktif = false;
        assert!(!p.adalah_layak_bonus());
    }

    #[test]
    fn test_tidak_layak_bonus_gaji_tinggi() {
        let p = Pekerja::baru(2, "Ketua", 6000.0);
        assert!(!p.adalah_layak_bonus());
    }

    // ── Test edge cases ───────────────────────────────────────
    #[test]
    fn test_nama_kosong() {
        let p = Pekerja::baru(1, "", 4500.0);
        assert_eq!(p.nama, "");
    }

    #[test]
    fn test_gaji_sifar() {
        let p = Pekerja::baru(1, "Pelatih", 0.0);
        assert_eq!(p.gaji_dengan_naik(10.0), 0.0);
    }
}
```

---

# BAHAGIAN 3: Test Helpers & Fixtures 🛠️

## DRY dalam Test Code

```rust
// src/lib.rs

#[derive(Debug, Clone, PartialEq)]
pub struct Transaksi {
    pub id:       u32,
    pub amaun:    f64,
    pub kategori: String,
    pub diluluskan: bool,
}

impl Transaksi {
    pub fn baru(id: u32, amaun: f64, kategori: &str) -> Self {
        Transaksi { id, amaun, kategori: kategori.into(), diluluskan: false }
    }

    pub fn luluskan(&mut self) {
        self.diluluskan = true;
    }

    pub fn adalah_besar(&self) -> bool {
        self.amaun > 10000.0
    }
}

pub fn kira_jumlah_diluluskan(transaksi: &[Transaksi]) -> f64 {
    transaksi.iter()
        .filter(|t| t.diluluskan)
        .map(|t| t.amaun)
        .sum()
}

#[cfg(test)]
mod tests {
    use super::*;

    // ── Test Fixtures ─────────────────────────────────────────
    // Fungsi helper yang buat data test yang konsisten
    fn buat_transaksi_kecil() -> Transaksi {
        Transaksi::baru(1, 500.0, "Perjalanan")
    }

    fn buat_transaksi_besar() -> Transaksi {
        Transaksi::baru(2, 15000.0, "Peralatan")
    }

    fn buat_senarai_transaksi() -> Vec<Transaksi> {
        vec![
            { let mut t = Transaksi::baru(1, 500.0, "A"); t.luluskan(); t },
            Transaksi::baru(2, 1000.0, "B"),  // tidak diluluskan
            { let mut t = Transaksi::baru(3, 2500.0, "C"); t.luluskan(); t },
        ]
    }

    // ── Tests menggunakan fixtures ────────────────────────────
    #[test]
    fn test_transaksi_kecil_bukan_besar() {
        let t = buat_transaksi_kecil();
        assert!(!t.adalah_besar());
    }

    #[test]
    fn test_transaksi_besar_adalah_besar() {
        let t = buat_transaksi_besar();
        assert!(t.adalah_besar());
    }

    #[test]
    fn test_jumlah_diluluskan() {
        let senarai = buat_senarai_transaksi();
        let jumlah = kira_jumlah_diluluskan(&senarai);
        assert_eq!(jumlah, 3000.0); // 500 + 2500
    }

    #[test]
    fn test_luluskan_transaksi() {
        let mut t = buat_transaksi_kecil();
        assert!(!t.diluluskan);
        t.luluskan();
        assert!(t.diluluskan);
    }

    // ── Parametric tests (manual) ─────────────────────────────
    // Rust belum ada parametric test built-in, tapi boleh guna loop
    #[test]
    fn test_pelbagai_amaun_besar() {
        let kes = [
            (10001.0, true),
            (10000.0, false), // tepat 10000 = tidak besar
            (9999.0,  false),
            (0.0,     false),
            (f64::MAX, true),
        ];

        for (amaun, dijangka) in &kes {
            let t = Transaksi::baru(1, *amaun, "Test");
            assert_eq!(
                t.adalah_besar(), *dijangka,
                "Untuk amaun {}, dijangka adalah_besar={}", amaun, dijangka
            );
        }
    }
}
```

---

# BAHAGIAN 4: Setup & Teardown 🔄

## Inisialisasi dan Pembersihan

```rust
// Rust tiada @Before/@After seperti JUnit
// Tapi ada pattern-pattern yang berfungsi:

use std::sync::Once;
use std::sync::Mutex;

// ── Cara 1: Setup dalam setiap test ───────────────────────────
#[cfg(test)]
mod tests {
    use super::*;

    fn setup() -> Vec<Pekerja> {
        // Setup data untuk test
        vec![
            Pekerja::baru(1, "Ali", 4500.0),
            Pekerja::baru(2, "Siti", 3800.0),
        ]
    }

    fn teardown(data: &mut Vec<Pekerja>) {
        // Cleanup (kalau ada state global, fail, DB dll)
        data.clear();
    }

    #[test]
    fn test_dengan_setup() {
        let mut data = setup();     // setup

        // Test
        assert_eq!(data.len(), 2);
        data.push(Pekerja::baru(3, "Amin", 5000.0));
        assert_eq!(data.len(), 3);

        teardown(&mut data);        // teardown
        assert!(data.is_empty());
    }

    // ── Cara 2: RAII untuk auto-cleanup ──────────────────────
    struct TestEnv {
        data: Vec<Pekerja>,
        fail_temp: Option<std::path::PathBuf>,
    }

    impl TestEnv {
        fn baru() -> Self {
            // Buat fail sementara untuk test
            let laluan = std::path::PathBuf::from("/tmp/test_pekerja.csv");
            std::fs::write(&laluan, "id,nama,gaji\n").unwrap();

            TestEnv {
                data: vec![Pekerja::baru(1, "Test", 4000.0)],
                fail_temp: Some(laluan),
            }
        }
    }

    impl Drop for TestEnv {
        fn drop(&mut self) {
            // Auto-cleanup bila TestEnv keluar scope!
            if let Some(laluan) = &self.fail_temp {
                let _ = std::fs::remove_file(laluan);
            }
            println!("TestEnv dibersihkan!");
        }
    }

    #[test]
    fn test_dengan_raii_env() {
        let env = TestEnv::baru();
        // fail_temp wujud di sini

        assert_eq!(env.data.len(), 1);

        // env di-drop di sini → Drop dipanggil → fail_temp dipadam!
    }

    // ── Cara 3: Once untuk global setup ──────────────────────
    static INISIALISASI: Once = Once::new();

    fn inisialisasi_sekali() {
        INISIALISASI.call_once(|| {
            // Ini hanya dijalankan SEKALI walaupun banyak test panggil
            println!("Global setup...");
            // Init logging, database connection, dll
        });
    }

    #[test]
    fn test_a_dengan_global_setup() {
        inisialisasi_sekali();
        // ...
    }

    #[test]
    fn test_b_dengan_global_setup() {
        inisialisasi_sekali(); // tidak buat apa - sudah dijalankan
        // ...
    }
}
```

---

# BAHAGIAN 5: Test dengan Error (Result) 🚨

## Test yang Return Result

```rust
use std::num::ParseIntError;

fn parse_id(s: &str) -> Result<u32, ParseIntError> {
    s.parse::<u32>()
}

fn baca_gaji(s: &str) -> Result<f64, String> {
    let gaji: f64 = s.parse().map_err(|_| format!("'{}' bukan nombor sah", s))?;
    if gaji < 0.0 { return Err("Gaji tidak boleh negatif".into()); }
    Ok(gaji)
}

#[cfg(test)]
mod tests {
    use super::*;

    // ── Test yang return Result ────────────────────────────────
    // Kalau return Err, test GAGAL dengan mesej yang jelas
    #[test]
    fn test_parse_id_sah() -> Result<(), ParseIntError> {
        let id = parse_id("42")?;
        assert_eq!(id, 42);
        Ok(())
    }

    #[test]
    fn test_parse_id_tidak_sah() {
        let hasil = parse_id("abc");
        assert!(hasil.is_err());
    }

    // ── Cara check specific error ─────────────────────────────
    #[test]
    fn test_gaji_negatif() {
        let hasil = baca_gaji("-100");
        assert_eq!(hasil, Err("Gaji tidak boleh negatif".into()));
    }

    #[test]
    fn test_gaji_bukan_nombor() {
        let hasil = baca_gaji("abc");
        assert!(hasil.is_err());
        let e = hasil.unwrap_err();
        assert!(e.contains("bukan nombor sah"), "Mesej ralat: {}", e);
    }

    // ── unwrap_err dan expect_err ─────────────────────────────
    #[test]
    fn test_dengan_unwrap_err() {
        let ralat = baca_gaji("xyz").unwrap_err();
        assert!(ralat.contains("xyz"));
    }

    // ── Test untuk thiserror errors ──────────────────────────
    #[derive(Debug, PartialEq, thiserror::Error)]
    enum AppError {
        #[error("Tidak dijumpai: {0}")]
        TidakJumpai(String),
        #[error("Nilai tidak sah: {0}")]
        TidakSah(String),
    }

    fn cari_pekerja(id: u32) -> Result<String, AppError> {
        match id {
            1 => Ok("Ali".into()),
            _ => Err(AppError::TidakJumpai(format!("ID {}", id))),
        }
    }

    #[test]
    fn test_cari_berjaya() -> Result<(), AppError> {
        let nama = cari_pekerja(1)?;
        assert_eq!(nama, "Ali");
        Ok(())
    }

    #[test]
    fn test_cari_tidak_jumpa() {
        let hasil = cari_pekerja(999);
        assert!(matches!(
            hasil,
            Err(AppError::TidakJumpai(m)) if m.contains("999")
        ));
    }
}
```

---

# BAHAGIAN 6: Async Tests ⚡

```toml
[dev-dependencies]
tokio = { version = "1", features = ["full", "test-util"] }
```

```rust
// Fungsi async yang perlu di-test
async fn dapatkan_data(id: u32) -> Result<String, String> {
    tokio::time::sleep(tokio::time::Duration::from_millis(10)).await;
    if id == 0 { return Err("ID tidak sah".into()); }
    Ok(format!("Data untuk ID {}", id))
}

async fn proses_semua(ids: Vec<u32>) -> Vec<Result<String, String>> {
    let mut hasil = Vec::new();
    for id in ids {
        hasil.push(dapatkan_data(id).await);
    }
    hasil
}

#[cfg(test)]
mod tests {
    use super::*;

    // ── Async test dengan #[tokio::test] ──────────────────────
    #[tokio::test]
    async fn test_async_berjaya() {
        let hasil = dapatkan_data(1).await;
        assert_eq!(hasil, Ok("Data untuk ID 1".into()));
    }

    #[tokio::test]
    async fn test_async_gagal() {
        let hasil = dapatkan_data(0).await;
        assert!(hasil.is_err());
        assert_eq!(hasil.unwrap_err(), "ID tidak sah");
    }

    // ── Async test yang return Result ─────────────────────────
    #[tokio::test]
    async fn test_async_result() -> Result<(), String> {
        let hasil = dapatkan_data(42).await?;
        assert!(hasil.contains("42"));
        Ok(())
    }

    // ── Test timeout ──────────────────────────────────────────
    #[tokio::test(flavor = "current_thread")]  // single-threaded runtime
    async fn test_dengan_timeout() {
        let hasil = tokio::time::timeout(
            tokio::time::Duration::from_millis(100),
            dapatkan_data(1)
        ).await;

        assert!(hasil.is_ok(), "Test timeout!");
        assert_eq!(hasil.unwrap(), Ok("Data untuk ID 1".into()));
    }

    // ── Test concurrent ───────────────────────────────────────
    #[tokio::test]
    async fn test_proses_serentak() {
        let ids = vec![1, 2, 3, 4, 5];
        let hasil = proses_semua(ids).await;
        assert_eq!(hasil.len(), 5);
        assert!(hasil.iter().all(|r| r.is_ok()));
    }

    // ── Guna tokio::time::pause untuk test masa ───────────────
    #[tokio::test]
    async fn test_dengan_masa_palsu() {
        tokio::time::pause(); // hentikan masa sebenar

        let mula = tokio::time::Instant::now();

        tokio::time::advance(tokio::time::Duration::from_secs(5)).await;

        let berlalu = mula.elapsed();
        assert!(berlalu >= tokio::time::Duration::from_secs(5));
        // Berlaku "seketika" walaupun ada 5 saat!
    }
}
```

---

# BAHAGIAN 7: Integration Tests 🔗

## Test yang Akses Crate Seperti Pengguna

```rust
// tests/test_sistem_pekerja.rs
// (fail dalam folder /tests - bukan dalam src/)

// Import crate yang ditest
use nama_crate::{Pekerja, SistemPekerja, KonfigSistem};

// Helper untuk integration test
mod common;  // tests/common/mod.rs

// ── Integration test ──────────────────────────────────────────

#[test]
fn test_workflow_pekerja_penuh() {
    // Setup
    let konfig = KonfigSistem::default();
    let mut sistem = SistemPekerja::baru(konfig);

    // Buat pekerja
    let id = sistem.tambah_pekerja("Ali Ahmad", 4500.0).unwrap();
    assert!(id > 0);

    // Cari pekerja
    let p = sistem.cari(id).unwrap();
    assert_eq!(p.nama, "Ali Ahmad");

    // Kemaskini
    sistem.kemaskini_gaji(id, 5000.0).unwrap();
    let p_kemas = sistem.cari(id).unwrap();
    assert_eq!(p_kemas.gaji, 5000.0);

    // Padam
    sistem.padam(id).unwrap();
    assert!(sistem.cari(id).is_none());
}

// tests/common/mod.rs
pub fn buat_sistem_test() -> SistemPekerja {
    let konfig = KonfigSistem {
        pangkalan_data: ":memory:".into(), // SQLite in-memory
        ..Default::default()
    };
    SistemPekerja::baru(konfig)
}

pub fn isi_data_test(sistem: &mut SistemPekerja) {
    sistem.tambah_pekerja("Ali", 4500.0).unwrap();
    sistem.tambah_pekerja("Siti", 3800.0).unwrap();
    sistem.tambah_pekerja("Amin", 5000.0).unwrap();
}
```

---

# BAHAGIAN 8: Doc Tests 📖

## Test dalam Dokumentasi

```rust
/// Kira jumlah elemen dalam slice.
///
/// # Examples
///
/// ```
/// let v = vec![1, 2, 3, 4, 5];
/// assert_eq!(nama_crate::kira_jumlah(&v), 15);
/// ```
///
/// Slice kosong:
/// ```
/// let kosong: Vec<i32> = vec![];
/// assert_eq!(nama_crate::kira_jumlah(&kosong), 0);
/// ```
pub fn kira_jumlah(v: &[i32]) -> i32 {
    v.iter().sum()
}

/// Cari nilai dalam slice.
///
/// Returns `Some(index)` kalau dijumpai, `None` kalau tidak.
///
/// # Examples
///
/// ```
/// let v = vec!["ali", "siti", "amin"];
/// assert_eq!(nama_crate::cari(&v, "siti"), Some(1));
/// assert_eq!(nama_crate::cari(&v, "zara"), None);
/// ```
///
/// # Panics
///
/// Tidak akan panic dalam keadaan biasa.
///
/// # Doc test yang menunjukkan error (should panic):
///
/// ```should_panic
/// let v: Vec<i32> = vec![];
/// let _ = v[0]; // Index out of bounds!
/// ```
///
/// ```compile_fail
/// let x: i32 = "bukan nombor"; // Ini tidak akan compile
/// ```
///
/// ```no_run
/// // Ini hanya compile, tidak dijalankan (guna untuk I/O)
/// let kandungan = std::fs::read_to_string("fail.txt").unwrap();
/// ```
pub fn cari<T: PartialEq>(v: &[T], cari: &T) -> Option<usize> {
    v.iter().position(|x| x == cari)
}
```

```bash
# Jalankan doc tests sahaja
cargo test --doc

# Jalankan semua (unit + integration + doc)
cargo test
```

---

# BAHAGIAN 9: Mocking & Test Doubles 🎭

## Mocking tanpa Crate

```rust
// Rust tidak ada mocking framework built-in
// Cara paling idiomatic: gunakan TRAIT

// ── Trait untuk dependency ────────────────────────────────────
pub trait PenyimpanPekerja {
    fn simpan(&mut self, pekerja: &Pekerja) -> Result<u32, String>;
    fn cari(&self, id: u32) -> Option<Pekerja>;
    fn padam(&mut self, id: u32) -> bool;
}

// ── Implementasi sebenar ──────────────────────────────────────
pub struct DbPekerja {
    pub pool: sqlx::MySqlPool,
}

impl PenyimpanPekerja for DbPekerja {
    fn simpan(&mut self, pekerja: &Pekerja) -> Result<u32, String> {
        // Query DB sebenar...
        todo!()
    }
    fn cari(&self, id: u32) -> Option<Pekerja> { todo!() }
    fn padam(&mut self, id: u32) -> bool { todo!() }
}

// ── Service yang bergantung pada trait ──────────────────────
pub struct PekerjaService<S: PenyimpanPekerja> {
    pub simpanan: S,
}

impl<S: PenyimpanPekerja> PekerjaService<S> {
    pub fn baru(simpanan: S) -> Self { PekerjaService { simpanan } }

    pub fn daftar_pekerja_baru(&mut self, nama: &str, gaji: f64) -> Result<Pekerja, String> {
        if nama.trim().is_empty() {
            return Err("Nama tidak boleh kosong".into());
        }
        if gaji < 0.0 {
            return Err("Gaji tidak boleh negatif".into());
        }
        let p = Pekerja::baru(0, nama, gaji);
        let id = self.simpanan.simpan(&p)?;
        Ok(Pekerja { id, ..p })
    }
}

// ── Test double (mock) ────────────────────────────────────────
#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::HashMap;

    // In-memory implementation untuk test
    struct MockPenyimpan {
        data:     HashMap<u32, Pekerja>,
        id_seter: u32,
        // Boleh tambah untuk track calls
        simpan_dipanggil: u32,
    }

    impl MockPenyimpan {
        fn baru() -> Self {
            MockPenyimpan {
                data:             HashMap::new(),
                id_seter:         0,
                simpan_dipanggil: 0,
            }
        }
    }

    impl PenyimpanPekerja for MockPenyimpan {
        fn simpan(&mut self, pekerja: &Pekerja) -> Result<u32, String> {
            self.id_seter += 1;
            self.simpan_dipanggil += 1;
            let id = self.id_seter;
            self.data.insert(id, Pekerja { id, ..pekerja.clone() });
            Ok(id)
        }

        fn cari(&self, id: u32) -> Option<Pekerja> {
            self.data.get(&id).cloned()
        }

        fn padam(&mut self, id: u32) -> bool {
            self.data.remove(&id).is_some()
        }
    }

    // Fail mock untuk test error handling
    struct MockPenyimpanGagal;

    impl PenyimpanPekerja for MockPenyimpanGagal {
        fn simpan(&mut self, _: &Pekerja) -> Result<u32, String> {
            Err("Pangkalan data tidak bersambung".into())
        }
        fn cari(&self, _: u32) -> Option<Pekerja> { None }
        fn padam(&mut self, _: u32) -> bool { false }
    }

    // ── Tests menggunakan mock ─────────────────────────────────
    #[test]
    fn test_daftar_pekerja_berjaya() {
        let mock = MockPenyimpan::baru();
        let mut service = PekerjaService::baru(mock);

        let hasil = service.daftar_pekerja_baru("Ali Ahmad", 4500.0);
        assert!(hasil.is_ok());
        let p = hasil.unwrap();
        assert_eq!(p.nama, "Ali Ahmad");
        assert_eq!(p.gaji, 4500.0);
        assert!(p.id > 0);
    }

    #[test]
    fn test_daftar_nama_kosong_gagal() {
        let mut service = PekerjaService::baru(MockPenyimpan::baru());
        let hasil = service.daftar_pekerja_baru("", 4500.0);
        assert!(hasil.is_err());
        assert_eq!(hasil.unwrap_err(), "Nama tidak boleh kosong");
    }

    #[test]
    fn test_daftar_gaji_negatif_gagal() {
        let mut service = PekerjaService::baru(MockPenyimpan::baru());
        let hasil = service.daftar_pekerja_baru("Ali", -100.0);
        assert!(hasil.is_err());
    }

    #[test]
    fn test_daftar_pangkalan_data_gagal() {
        let mut service = PekerjaService::baru(MockPenyimpanGagal);
        let hasil = service.daftar_pekerja_baru("Ali", 4500.0);
        assert!(hasil.is_err());
        assert!(hasil.unwrap_err().contains("tidak bersambung"));
    }

    #[test]
    fn test_simpan_dipanggil_sekali() {
        let mock = MockPenyimpan::baru();
        let mut service = PekerjaService::baru(mock);
        service.daftar_pekerja_baru("Ali", 4500.0).unwrap();

        // Akses mock untuk verify call count
        assert_eq!(service.simpanan.simpan_dipanggil, 1);
    }
}
```

---

## Dengan Crate `mockall`

```toml
[dev-dependencies]
mockall = "0.12"
```

```rust
use mockall::{automock, predicate::*};

#[automock]
pub trait EmailSender {
    fn hantar(&self, kepada: &str, subjek: &str, badan: &str) -> Result<(), String>;
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_dengan_mockall() {
        let mut mock = MockEmailSender::new();

        // Setup expectation
        mock.expect_hantar()
            .with(eq("ali@email.com"), always(), always())
            .times(1)  // mesti dipanggil TEPAT 1 kali
            .returning(|_, _, _| Ok(()));

        // Guna mock
        let hasil = mock.hantar("ali@email.com", "Test", "Badan test");
        assert!(hasil.is_ok());

        // mockall auto-verify expectations bila di-drop
    }
}
```

---

# BAHAGIAN 10: Mini Project — TDD KADA Service 🏗️

## Test-Driven Development

```rust
// Tulis TEST dulu, kemudian kod

// src/kehadiran_service.rs

use std::collections::HashMap;
use chrono::{NaiveDate, NaiveDateTime, Utc};

#[derive(Debug, Clone, PartialEq)]
pub struct RekodKehadiran {
    pub id:           u32,
    pub pekerja_id:   u32,
    pub tarikh:       NaiveDate,
    pub masa_masuk:   NaiveDateTime,
    pub masa_keluar:  Option<NaiveDateTime>,
}

impl RekodKehadiran {
    pub fn jam_bekerja(&self) -> Option<f64> {
        let keluar = self.masa_keluar?;
        let minit = (keluar - self.masa_masuk).num_minutes();
        Some(minit as f64 / 60.0)
    }

    pub fn adalah_lewat(&self) -> bool {
        // Lewat kalau masuk selepas 8:30 pagi
        self.masa_masuk.time().hour() > 8
            || (self.masa_masuk.time().hour() == 8
                && self.masa_masuk.time().minute() > 30)
    }
}

pub trait KehadiranRepository {
    fn simpan(&mut self, rekod: RekodKehadiran) -> Result<u32, String>;
    fn cari_oleh_pekerja(&self, pekerja_id: u32, tarikh: NaiveDate) -> Option<&RekodKehadiran>;
    fn senarai_oleh_tarikh(&self, tarikh: NaiveDate) -> Vec<&RekodKehadiran>;
    fn kemaskini_masa_keluar(&mut self, id: u32, masa: NaiveDateTime) -> Result<(), String>;
}

pub struct KehadiranService<R: KehadiranRepository> {
    repo:    R,
    id_seter: u32,
}

impl<R: KehadiranRepository> KehadiranService<R> {
    pub fn baru(repo: R) -> Self {
        KehadiranService { repo, id_seter: 0 }
    }

    pub fn daftar_masuk(
        &mut self,
        pekerja_id:   u32,
        masa_masuk:   NaiveDateTime,
    ) -> Result<RekodKehadiran, String> {
        let tarikh = masa_masuk.date();

        // Semak dah masuk hari ini
        if self.repo.cari_oleh_pekerja(pekerja_id, tarikh).is_some() {
            return Err(format!("Pekerja {} sudah daftar masuk hari ini", pekerja_id));
        }

        self.id_seter += 1;
        let rekod = RekodKehadiran {
            id:          self.id_seter,
            pekerja_id,
            tarikh,
            masa_masuk,
            masa_keluar: None,
        };

        self.repo.simpan(rekod.clone())?;
        Ok(rekod)
    }

    pub fn daftar_keluar(
        &mut self,
        pekerja_id:   u32,
        masa_keluar:  NaiveDateTime,
    ) -> Result<f64, String> {
        let tarikh = masa_keluar.date();

        let rekod = self.repo.cari_oleh_pekerja(pekerja_id, tarikh)
            .ok_or_else(|| format!("Pekerja {} belum daftar masuk hari ini", pekerja_id))?;

        if rekod.masa_keluar.is_some() {
            return Err(format!("Pekerja {} sudah daftar keluar", pekerja_id));
        }

        if masa_keluar <= rekod.masa_masuk {
            return Err("Masa keluar mesti selepas masa masuk".into());
        }

        let id = rekod.id;
        self.repo.kemaskini_masa_keluar(id, masa_keluar)?;

        let minit = (masa_keluar - rekod.masa_masuk).num_minutes();
        Ok(minit as f64 / 60.0)
    }

    pub fn laporan_hari(&self, tarikh: NaiveDate) -> LaporanHarian {
        let semua = self.repo.senarai_oleh_tarikh(tarikh);
        let jumlah = semua.len();
        let lewat = semua.iter().filter(|r| r.adalah_lewat()).count();
        let sudah_keluar = semua.iter().filter(|r| r.masa_keluar.is_some()).count();

        LaporanHarian { tarikh, jumlah, lewat, sudah_keluar }
    }
}

#[derive(Debug, PartialEq)]
pub struct LaporanHarian {
    pub tarikh:        NaiveDate,
    pub jumlah:        usize,
    pub lewat:         usize,
    pub sudah_keluar:  usize,
}

// ─── Tests ─────────────────────────────────────────────────────

#[cfg(test)]
mod tests {
    use super::*;
    use chrono::NaiveTime;

    // ── Mock Repository ───────────────────────────────────────
    #[derive(Default)]
    struct MockRepo {
        data:    HashMap<u32, RekodKehadiran>,
        id_seter: u32,
    }

    impl KehadiranRepository for MockRepo {
        fn simpan(&mut self, rekod: RekodKehadiran) -> Result<u32, String> {
            let id = rekod.id;
            self.data.insert(id, rekod);
            Ok(id)
        }

        fn cari_oleh_pekerja(&self, pekerja_id: u32, tarikh: NaiveDate) -> Option<&RekodKehadiran> {
            self.data.values()
                .find(|r| r.pekerja_id == pekerja_id && r.tarikh == tarikh)
        }

        fn senarai_oleh_tarikh(&self, tarikh: NaiveDate) -> Vec<&RekodKehadiran> {
            self.data.values()
                .filter(|r| r.tarikh == tarikh)
                .collect()
        }

        fn kemaskini_masa_keluar(&mut self, id: u32, masa: NaiveDateTime) -> Result<(), String> {
            self.data.get_mut(&id)
                .map(|r| r.masa_keluar = Some(masa))
                .ok_or_else(|| "Rekod tidak dijumpai".into())
        }
    }

    fn buat_service() -> KehadiranService<MockRepo> {
        KehadiranService::baru(MockRepo::default())
    }

    fn dt(tarikh: &str, jam: u32, minit: u32) -> NaiveDateTime {
        NaiveDateTime::new(
            NaiveDate::parse_from_str(tarikh, "%Y-%m-%d").unwrap(),
            NaiveTime::from_hms_opt(jam, minit, 0).unwrap(),
        )
    }

    fn tarikh(s: &str) -> NaiveDate {
        NaiveDate::parse_from_str(s, "%Y-%m-%d").unwrap()
    }

    // ── Unit Tests ────────────────────────────────────────────

    #[test]
    fn test_daftar_masuk_berjaya() {
        let mut svc = buat_service();
        let masa = dt("2024-01-15", 8, 0);

        let rekod = svc.daftar_masuk(1, masa).unwrap();

        assert_eq!(rekod.pekerja_id, 1);
        assert_eq!(rekod.masa_masuk, masa);
        assert!(rekod.masa_keluar.is_none());
        assert!(!rekod.adalah_lewat()); // 8:00 = tepat masa
    }

    #[test]
    fn test_daftar_masuk_lewat() {
        let mut svc = buat_service();
        let masa = dt("2024-01-15", 9, 15); // 9:15 AM = lewat

        let rekod = svc.daftar_masuk(1, masa).unwrap();
        assert!(rekod.adalah_lewat());
    }

    #[test]
    fn test_daftar_masuk_dua_kali_gagal() {
        let mut svc = buat_service();
        svc.daftar_masuk(1, dt("2024-01-15", 8, 0)).unwrap();

        let hasil = svc.daftar_masuk(1, dt("2024-01-15", 10, 0));
        assert!(hasil.is_err());
        assert!(hasil.unwrap_err().contains("sudah daftar masuk"));
    }

    #[test]
    fn test_daftar_keluar_berjaya() {
        let mut svc = buat_service();
        svc.daftar_masuk(1, dt("2024-01-15", 8, 0)).unwrap();

        let jam = svc.daftar_keluar(1, dt("2024-01-15", 17, 0)).unwrap();
        assert_eq!(jam, 9.0); // 9 jam bekerja
    }

    #[test]
    fn test_daftar_keluar_tanpa_masuk_gagal() {
        let mut svc = buat_service();
        let hasil = svc.daftar_keluar(99, dt("2024-01-15", 17, 0));
        assert!(hasil.is_err());
        assert!(hasil.unwrap_err().contains("belum daftar masuk"));
    }

    #[test]
    fn test_daftar_keluar_sebelum_masuk_gagal() {
        let mut svc = buat_service();
        svc.daftar_masuk(1, dt("2024-01-15", 8, 0)).unwrap();

        let hasil = svc.daftar_keluar(1, dt("2024-01-15", 7, 0));
        assert!(hasil.is_err());
        assert!(hasil.unwrap_err().contains("selepas masa masuk"));
    }

    #[test]
    fn test_laporan_harian() {
        let mut svc = buat_service();
        let tarikh_test = tarikh("2024-01-15");

        svc.daftar_masuk(1, dt("2024-01-15", 8, 0)).unwrap();
        svc.daftar_masuk(2, dt("2024-01-15", 9, 30)).unwrap(); // lewat!
        svc.daftar_masuk(3, dt("2024-01-15", 7, 45)).unwrap(); // awal
        svc.daftar_keluar(1, dt("2024-01-15", 17, 0)).unwrap();

        let laporan = svc.laporan_hari(tarikh_test);
        assert_eq!(laporan.jumlah, 3);
        assert_eq!(laporan.lewat, 1);        // pekerja 2
        assert_eq!(laporan.sudah_keluar, 1); // pekerja 1 sahaja
    }

    // ── Edge cases ────────────────────────────────────────────

    #[test]
    fn test_jam_kerja_tepat_8_jam() {
        let mut svc = buat_service();
        svc.daftar_masuk(1, dt("2024-01-15", 8, 0)).unwrap();
        let jam = svc.daftar_keluar(1, dt("2024-01-15", 16, 0)).unwrap();
        assert_eq!(jam, 8.0);
    }

    #[test]
    fn test_pekerja_lain_boleh_masuk_hari_sama() {
        let mut svc = buat_service();
        svc.daftar_masuk(1, dt("2024-01-15", 8, 0)).unwrap();

        // Pekerja 2 boleh masuk — bukan pekerja 1!
        let hasil = svc.daftar_masuk(2, dt("2024-01-15", 8, 30));
        assert!(hasil.is_ok());
    }
}
```

---

# 📋 Rujukan Pantas — Test Cheat Sheet

## Attributes

```rust
#[test]                           // test function
#[cfg(test)]                      // compile hanya dalam test mode
#[should_panic]                   // test perlu panic
#[should_panic(expected = "msg")] // panic dengan pesan tertentu
#[ignore]                         // skip test
#[ignore = "sebab"]               // skip dengan nota
#[tokio::test]                    // async test
```

## Assertion Macros

```rust
assert!(expr)                     // expr mesti true
assert!(expr, "msg {}", val)      // dengan mesej custom
assert_eq!(a, b)                  // a == b
assert_eq!(a, b, "msg")
assert_ne!(a, b)                  // a != b
assert!(matches!(opt, Some(n) if n > 0))  // pattern matching
```

## Jalankan Test

```bash
cargo test                        # semua test
cargo test nama                   # filter nama
cargo test -- --nocapture         # tunjuk println!
cargo test -- --ignored           # jalankan yang di-ignore
cargo test -- --test-threads=1    # satu thread
cargo test --doc                  # doc tests sahaja
cargo test --test fail_test       # integration test sahaja
cargo test --lib                  # unit test sahaja
```

## Best Practices

```
✔ Satu test = satu behavior yang ditest
✔ Nama test: test_[fungsi]_[scenario]_[hasil dijangka]
✔ Arrange → Act → Assert
✔ Test edge cases: kosong, sifar, negatif, sangat besar
✔ Guna fixtures untuk mengelak repetition
✔ Test happy path DAN error path
✔ Mock dependencies untuk unit test yang benar-benar unit
✔ Integration test untuk workflow penuh
✔ Doc test untuk contoh dalam dokumentasi
```

---

*Test yang baik adalah dokumentasi yang boleh dijalankan.*
*Rust compiler adalah test pertama — test adalah kedua.* 🦀
