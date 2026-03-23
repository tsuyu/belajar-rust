# 📊 CSV Manipulation dalam Rust — Panduan Lengkap

> Semua tentang CSV: baca, tulis, filter, transform, aggregate,
> join, dan export. Dari asas hingga pemprosesan data besar.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Kenapa CSV?

```
CSV = Comma-Separated Values

Format paling universal untuk data tabular:
  ✔ Boleh buka dengan Excel, Google Sheets
  ✔ Semua database boleh import/export CSV
  ✔ Simple, human-readable
  ✔ Kecil (berbanding JSON/XML untuk data besar)
  ✔ Standard de facto untuk data interchange

Guna kes:
  - Import data pekerja dari HR system
  - Export laporan untuk akaun
  - Data migration antara sistem
  - Analytics dan reporting
  - ETL (Extract, Transform, Load)
```

---

## Peta Pembelajaran

```
Bahagian 1  → Setup & Baca CSV Asas
Bahagian 2  → Tulis CSV
Bahagian 3  → CSV dengan Serde
Bahagian 4  → Filter & Cari
Bahagian 5  → Transform & Modify
Bahagian 6  → Aggregate & Statistik
Bahagian 7  → Join Pelbagai CSV
Bahagian 8  → CSV Besar & Streaming
Bahagian 9  → Ralat & Data Kotor
Bahagian 10 → Mini Project: Sistem Laporan Gaji
```

---

# BAHAGIAN 1: Setup & Baca CSV Asas 📖

## Cargo.toml

```toml
[dependencies]
csv           = "1"
serde         = { version = "1", features = ["derive"] }
```

## Baca CSV Paling Mudah

```rust
use std::error::Error;
use csv::Reader;

fn main() -> Result<(), Box<dyn Error>> {
    // Contoh CSV dalam string
    let data = "nama,umur,bandar\nAli,25,KL\nSiti,30,JB\nAmin,22,KB";

    let mut pembaca = Reader::from_reader(data.as_bytes());

    // Baca header
    println!("Header: {:?}", pembaca.headers()?);

    // Iterate rekod
    for rekod in pembaca.records() {
        let rekod = rekod?;
        println!("{:?}", rekod);
        // akses field dengan indeks
        println!("Nama: {}, Umur: {}", &rekod[0], &rekod[1]);
    }

    Ok(())
}
```

---

## Baca dari Fail

```rust
use csv::Reader;
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> {
    // ── Baca dari fail ────────────────────────────────────────
    let mut pembaca = Reader::from_path("pekerja.csv")?;

    // Header
    let header = pembaca.headers()?.clone();
    println!("Kolum: {:?}", header);

    // Iterate semua rekod
    for (no, rekod) in pembaca.records().enumerate() {
        let rekod = rekod?;
        println!("Baris {}: {:?}", no + 1, rekod);
    }

    // ── Dengan konfigurasi ────────────────────────────────────
    let mut pembaca = csv::ReaderBuilder::new()
        .delimiter(b',')           // separator (default: ,)
        .has_headers(true)         // ada header (default: true)
        .flexible(false)           // semua baris sama bilangan field
        .trim(csv::Trim::All)      // trim whitespace
        .comment(Some(b'#'))       // baris bermula # = comment
        .from_path("data.csv")?;

    // ── CSV dengan delimiter lain ─────────────────────────────
    let mut tsv_pembaca = csv::ReaderBuilder::new()
        .delimiter(b'\t')          // tab-separated
        .from_path("data.tsv")?;

    let mut semicolon_pembaca = csv::ReaderBuilder::new()
        .delimiter(b';')           // semicolon (Excel European format)
        .from_path("data.csv")?;

    Ok(())
}
```

---

## Akses Field

```rust
use csv::{Reader, StringRecord};
use std::error::Error;

fn demo_akses_field() -> Result<(), Box<dyn Error>> {
    let data = "no_pekerja,nama,bahagian,gaji,aktif\n\
                KADA001,Ali Ahmad,ICT,4500.00,true\n\
                KADA002,Siti Hawa,HR,3800.00,true\n\
                KADA003,Amin Razak,Kewangan,4200.00,false";

    let mut pembaca = Reader::from_reader(data.as_bytes());

    // Simpan header untuk reference
    let header = pembaca.headers()?.clone();

    for rekod in pembaca.records() {
        let rekod = rekod?;

        // Cara 1: Akses dengan indeks
        let nama = &rekod[1];
        let gaji_str = &rekod[3];

        // Cara 2: Akses dengan nama kolum (cari indeks dulu)
        let idx_gaji = header.iter().position(|h| h == "gaji").unwrap();
        let gaji: f64 = rekod[idx_gaji].parse()?;

        // Cara 3: Iterate semua field
        for (nama_kolum, nilai) in header.iter().zip(rekod.iter()) {
            println!("  {}: {}", nama_kolum, nilai);
        }

        // Cara 4: Tukar ke HashMap
        let peta: std::collections::HashMap<&str, &str> =
            header.iter().zip(rekod.iter()).collect();
        println!("Nama: {}", peta["nama"]);

        // Parse ke type tertentu
        let gaji: f64 = rekod[3].parse()?;
        let aktif: bool = rekod[4].parse()?;
        println!("{} - RM{:.2} - {}", nama, gaji, aktif);
    }

    Ok(())
}
```

---

## 🧠 Brain Teaser #1

Apa perbezaan antara `records()` dan `byte_records()` dalam crate csv?

<details>
<summary>👀 Jawapan</summary>

```
records() → Iterator<Item = Result<StringRecord>>
  - Return StringRecord (String/&str)
  - Validate UTF-8 untuk setiap field
  - Lebih mudah digunakan
  - Sedikit lebih lambat (UTF-8 validation)
  - Guna bila data adalah teks biasa

byte_records() → Iterator<Item = Result<ByteRecord>>
  - Return ByteRecord (&[u8])
  - TIDAK validate UTF-8
  - Lebih laju
  - Perlu manual tukar ke string bila perlu
  - Guna bila:
    • Data mungkin bukan UTF-8 (encoding lain)
    • Performance kritikal (fail besar)
    • Binary data dalam CSV

Contoh:
  // records() — mudah, selamat
  for rekod in pembaca.records() {
      let nama: &str = &rekod?[0];  // sudah String
  }

  // byte_records() — laju
  for rekod in pembaca.byte_records() {
      let rekod = rekod?;
      let nama = std::str::from_utf8(&rekod[0])?;  // manual convert
  }
```
</details>

---

# BAHAGIAN 2: Tulis CSV ✍️

## Tulis CSV Asas

```rust
use csv::Writer;
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> {
    // ── Tulis ke fail ─────────────────────────────────────────
    let mut penulis = Writer::from_path("output.csv")?;

    // Tulis header
    penulis.write_record(&["no_pekerja", "nama", "bahagian", "gaji"])?;

    // Tulis data
    penulis.write_record(&["KADA001", "Ali Ahmad", "ICT", "4500.00"])?;
    penulis.write_record(&["KADA002", "Siti Hawa", "HR", "3800.00"])?;

    penulis.flush()?;  // penting! pastikan semua ditulis ke disk
    println!("CSV ditulis!");

    // ── Tulis ke String (tidak ke fail) ───────────────────────
    let mut penulis = Writer::from_writer(vec![]);  // tulis ke Vec<u8>

    penulis.write_record(&["nama", "skor"])?;
    penulis.write_record(&["Ali", "85.5"])?;
    penulis.write_record(&["Siti", "92.0"])?;
    penulis.flush()?;

    let output = String::from_utf8(penulis.into_inner()?)?;
    println!("CSV:\n{}", output);

    // ── Dengan konfigurasi ────────────────────────────────────
    let mut penulis = csv::WriterBuilder::new()
        .delimiter(b',')
        .has_headers(true)
        .quote_style(csv::QuoteStyle::Necessary)  // quote bila perlu sahaja
        .from_path("data.csv")?;

    // ── Tulis Vec<String> ─────────────────────────────────────
    let rekod: Vec<String> = vec![
        "KADA003".into(),
        "Amin Razak".into(),
        "4200.00".into()
    ];
    penulis.write_record(&rekod)?;

    // ── Field dengan koma/newline (auto-quoted) ───────────────
    penulis.write_record(&[
        "KADA004",
        "Ahmad, bin Salleh",     // koma dalam nama → auto-quoted!
        "\"Pejabat\" Kemubu",   // quote dalam nilai → auto-escaped
        "3800.00"
    ])?;

    penulis.flush()?;
    std::fs::remove_file("data.csv")?;
    std::fs::remove_file("output.csv")?;

    Ok(())
}
```

---

## Append ke Fail CSV

```rust
use std::fs::OpenOptions;
use csv::Writer;
use std::error::Error;

fn tambah_rekod(
    laluan: &str,
    rekod: &[&str],
) -> Result<(), Box<dyn Error>> {
    // Semak sama ada fail baru atau sudah ada
    let fail_baru = !std::path::Path::new(laluan).exists();

    // Buka dalam mod append
    let fail = OpenOptions::new()
        .write(true)
        .create(true)
        .append(true)
        .open(laluan)?;

    let mut penulis = Writer::from_writer(fail);

    // Tulis header hanya kalau fail baru
    if fail_baru {
        penulis.write_record(&["tarikh", "nama", "tindakan"])?;
    }

    penulis.write_record(rekod)?;
    penulis.flush()?;

    Ok(())
}

fn main() -> Result<(), Box<dyn Error>> {
    let fail_log = "log_aktiviti.csv";

    tambah_rekod(fail_log, &["2024-01-15", "Ali Ahmad", "Log masuk"])?;
    tambah_rekod(fail_log, &["2024-01-15", "Siti Hawa", "Log masuk"])?;
    tambah_rekod(fail_log, &["2024-01-15", "Ali Ahmad", "Log keluar"])?;

    println!("Log dikemas kini!");

    // Baca semula untuk verify
    let mut pembaca = csv::Reader::from_path(fail_log)?;
    for rekod in pembaca.records() {
        println!("{:?}", rekod?);
    }

    std::fs::remove_file(fail_log)?;
    Ok(())
}
```

---

# BAHAGIAN 3: CSV dengan Serde 🔄

## Deserialize CSV → Struct

```rust
use csv::Reader;
use serde::{Serialize, Deserialize};
use std::error::Error;

// ── Struct biasa ──────────────────────────────────────────────
#[derive(Debug, Serialize, Deserialize, Clone)]
struct Pekerja {
    no_pekerja: String,
    nama:       String,
    bahagian:   String,
    gaji:       f64,
    aktif:      bool,
}

// ── Dengan rename dan skip ────────────────────────────────────
#[derive(Debug, Serialize, Deserialize)]
struct PekerjaAPI {
    #[serde(rename = "Employee ID")]    // nama kolum dalam CSV
    id:       String,
    #[serde(rename = "Full Name")]
    nama:     String,
    #[serde(rename = "Department")]
    bahagian: String,
    #[serde(rename = "Salary")]
    gaji:     f64,
    #[serde(skip_deserializing)]        // skip — tidak dalam CSV
    hash:     String,
}

// ── Dengan Option untuk field optional ───────────────────────
#[derive(Debug, Deserialize)]
struct PekerjaOpt {
    nama:     String,
    bahagian: String,
    gaji:     f64,
    emel:     Option<String>,           // mungkin kosong dalam CSV
    no_tel:   Option<String>,
}

fn main() -> Result<(), Box<dyn Error>> {
    let csv_data = "no_pekerja,nama,bahagian,gaji,aktif\n\
                    KADA001,Ali Ahmad,ICT,4500.00,true\n\
                    KADA002,Siti Hawa,HR,3800.00,true\n\
                    KADA003,Amin Razak,Kewangan,4200.00,false";

    let mut pembaca = Reader::from_reader(csv_data.as_bytes());

    // Deserialize terus ke struct!
    let mut pekerja_list: Vec<Pekerja> = Vec::new();
    for rekod in pembaca.deserialize() {
        let pekerja: Pekerja = rekod?;
        println!("{:#?}", pekerja);
        pekerja_list.push(pekerja);
    }

    // Atau collect sekaligus
    let mut pembaca = Reader::from_reader(csv_data.as_bytes());
    let semua: Vec<Pekerja> = pembaca.deserialize()
        .collect::<Result<Vec<_>, _>>()?;

    println!("\nJumlah: {} pekerja", semua.len());

    Ok(())
}
```

---

## Serialize Struct → CSV

```rust
use csv::Writer;
use serde::{Serialize, Deserialize};
use std::error::Error;

#[derive(Debug, Serialize, Deserialize, Clone)]
struct LaporanGaji {
    no_pekerja: String,
    nama:       String,
    gaji_asas:  f64,
    elaun:      f64,
    potongan:   f64,
    gaji_bersih: f64,
}

fn main() -> Result<(), Box<dyn Error>> {
    let laporan = vec![
        LaporanGaji {
            no_pekerja:  "KADA001".into(),
            nama:        "Ali Ahmad".into(),
            gaji_asas:   4500.00,
            elaun:       500.00,
            potongan:    300.00,
            gaji_bersih: 4700.00,
        },
        LaporanGaji {
            no_pekerja:  "KADA002".into(),
            nama:        "Siti Hawa".into(),
            gaji_asas:   3800.00,
            elaun:       300.00,
            potongan:    250.00,
            gaji_bersih: 3850.00,
        },
    ];

    // Serialize Vec<Struct> ke CSV
    let mut penulis = Writer::from_path("laporan_gaji.csv")?;

    for rekod in &laporan {
        penulis.serialize(rekod)?;  // auto-include header!
    }

    penulis.flush()?;
    println!("Laporan gaji ditulis!");

    // Baca semula untuk verify
    let mut pembaca = Reader::from_path("laporan_gaji.csv")?;
    for rekod in pembaca.deserialize::<LaporanGaji>() {
        let r = rekod?;
        println!("{}: RM{:.2}", r.nama, r.gaji_bersih);
    }

    std::fs::remove_file("laporan_gaji.csv")?;
    Ok(())
}
```

---

# BAHAGIAN 4: Filter & Cari 🔍

## Filter Rekod

```rust
use csv::{Reader, Writer};
use serde::{Serialize, Deserialize};
use std::error::Error;

#[derive(Debug, Serialize, Deserialize, Clone)]
struct Pekerja {
    no_pekerja: String,
    nama:       String,
    bahagian:   String,
    gaji:       f64,
    aktif:      bool,
}

fn filter_pekerja<F>(
    laluan_masuk: &str,
    laluan_keluar: &str,
    predikat: F,
) -> Result<usize, Box<dyn Error>>
where
    F: Fn(&Pekerja) -> bool,
{
    let mut pembaca = Reader::from_path(laluan_masuk)?;
    let mut penulis = Writer::from_path(laluan_keluar)?;
    let mut kiraan = 0;

    for rekod in pembaca.deserialize::<Pekerja>() {
        let pekerja = rekod?;
        if predikat(&pekerja) {
            penulis.serialize(&pekerja)?;
            kiraan += 1;
        }
    }

    penulis.flush()?;
    Ok(kiraan)
}

fn main() -> Result<(), Box<dyn Error>> {
    let csv_data = "no_pekerja,nama,bahagian,gaji,aktif\n\
                    KADA001,Ali Ahmad,ICT,4500.00,true\n\
                    KADA002,Siti Hawa,HR,3800.00,true\n\
                    KADA003,Amin Razak,ICT,5000.00,true\n\
                    KADA004,Zara Lina,Kewangan,4200.00,false\n\
                    KADA005,Razi Mahir,HR,3500.00,false";

    // Tulis fail ujian
    std::fs::write("pekerja.csv", csv_data)?;

    // ── Filter 1: Pekerja aktif sahaja ────────────────────────
    let n = filter_pekerja(
        "pekerja.csv",
        "pekerja_aktif.csv",
        |p| p.aktif
    )?;
    println!("Pekerja aktif: {}", n);

    // ── Filter 2: Bahagian tertentu ───────────────────────────
    let n = filter_pekerja(
        "pekerja.csv",
        "pekerja_ict.csv",
        |p| p.bahagian == "ICT"
    )?;
    println!("Pekerja ICT: {}", n);

    // ── Filter 3: Gaji dalam julat ────────────────────────────
    let n = filter_pekerja(
        "pekerja.csv",
        "gaji_tinggi.csv",
        |p| p.gaji >= 4000.0 && p.gaji <= 5000.0
    )?;
    println!("Gaji 4000-5000: {}", n);

    // ── Cari rekod tertentu ───────────────────────────────────
    let mut pembaca = Reader::from_path("pekerja.csv")?;
    let hasil: Vec<Pekerja> = pembaca.deserialize::<Pekerja>()
        .filter_map(|r| r.ok())
        .filter(|p| p.nama.to_lowercase().contains("ali"))
        .collect();

    println!("\nCarian 'ali': {} keputusan", hasil.len());
    for p in &hasil {
        println!("  {} — {}", p.no_pekerja, p.nama);
    }

    // Bersihkan
    for f in &["pekerja.csv", "pekerja_aktif.csv", "pekerja_ict.csv",
               "gaji_tinggi.csv"] {
        let _ = std::fs::remove_file(f);
    }

    Ok(())
}
```

---

# BAHAGIAN 5: Transform & Modify 🔧

## Transform Data CSV

```rust
use csv::{Reader, Writer, StringRecord};
use serde::{Serialize, Deserialize};
use std::error::Error;

#[derive(Debug, Deserialize)]
struct PekerjaRaw {
    no_pekerja: String,
    nama:       String,
    bahagian:   String,
    gaji:       f64,
    tarikh_join: String,
}

#[derive(Debug, Serialize)]
struct PekerjaTransformed {
    id:           String,
    nama_penuh:   String,
    jabatan:      String,
    gaji_rm:      String,           // format dengan RM
    gaji_bersih:  f64,              // selepas potongan
    tahun_khidmat: u32,
    kategori_gaji: String,
}

fn transform_pekerja(raw: PekerjaRaw) -> PekerjaTransformed {
    // Kira tahun khidmat
    let tahun_join: u32 = raw.tarikh_join
        .split('-')
        .next()
        .and_then(|s| s.parse().ok())
        .unwrap_or(2024);
    let tahun_khidmat = 2024_u32.saturating_sub(tahun_join);

    // Kategori gaji
    let kategori = match raw.gaji as u32 {
        g if g >= 7000 => "Pengurusan Atasan",
        g if g >= 5000 => "Pengurusan",
        g if g >= 3500 => "Staf Senior",
        _              => "Staf",
    };

    // Gaji bersih (simulasi: tolak 11% KWSP + cukai)
    let gaji_bersih = raw.gaji * 0.89;

    PekerjaTransformed {
        id:           raw.no_pekerja.to_uppercase(),
        nama_penuh:   raw.nama.trim().to_string(),
        jabatan:      raw.bahagian.clone(),
        gaji_rm:      format!("RM{:,.2}", raw.gaji),
        gaji_bersih:  (gaji_bersih * 100.0).round() / 100.0,
        tahun_khidmat,
        kategori_gaji: kategori.into(),
    }
}

fn main() -> Result<(), Box<dyn Error>> {
    let data = "no_pekerja,nama,bahagian,gaji,tarikh_join\n\
                KADA001, Ali Ahmad ,ICT,4500.00,2018-03-15\n\
                KADA002,Siti Hawa,HR,3800.00,2020-07-01\n\
                KADA003,Amin Razak,ICT,5500.00,2015-01-10";

    let mut pembaca = Reader::from_reader(data.as_bytes());
    let mut penulis = Writer::from_writer(vec![]);

    for rekod in pembaca.deserialize::<PekerjaRaw>() {
        let raw = rekod?;
        let transformed = transform_pekerja(raw);
        penulis.serialize(&transformed)?;
    }

    penulis.flush()?;
    let output = String::from_utf8(penulis.into_inner()?)?;
    println!("{}", output);

    Ok(())
}
```

---

## Tambah, Buang, Susun Semula Kolum

```rust
use csv::{Reader, Writer, ByteRecord};
use std::error::Error;

// Pilih kolum tertentu sahaja (projection)
fn pilih_kolum(
    laluan_masuk: &str,
    kolum_pilih:  &[&str],
    laluan_keluar: &str,
) -> Result<(), Box<dyn Error>> {
    let mut pembaca = Reader::from_path(laluan_masuk)?;
    let header = pembaca.headers()?.clone();

    // Cari indeks kolum yang dikehendaki
    let indeks: Vec<usize> = kolum_pilih.iter()
        .filter_map(|k| header.iter().position(|h| h == *k))
        .collect();

    let mut penulis = Writer::from_path(laluan_keluar)?;

    // Tulis header baru
    penulis.write_record(kolum_pilih)?;

    for rekod in pembaca.records() {
        let rekod = rekod?;
        let baris: Vec<&str> = indeks.iter()
            .map(|&i| &rekod[i])
            .collect();
        penulis.write_record(&baris)?;
    }

    penulis.flush()?;
    Ok(())
}

// Tambah kolum baru
fn tambah_kolum<F>(
    laluan_masuk:  &str,
    nama_kolum:    &str,
    nilai_fn:      F,
    laluan_keluar: &str,
) -> Result<(), Box<dyn Error>>
where
    F: Fn(&csv::StringRecord) -> String,
{
    let mut pembaca = Reader::from_path(laluan_masuk)?;
    let header = pembaca.headers()?.clone();
    let mut penulis = Writer::from_path(laluan_keluar)?;

    // Header baru dengan kolum tambahan
    let mut header_baru: Vec<String> = header.iter().map(|s| s.to_string()).collect();
    header_baru.push(nama_kolum.to_string());
    penulis.write_record(&header_baru)?;

    for rekod in pembaca.records() {
        let rekod = rekod?;
        let nilai_baru = nilai_fn(&rekod);
        let mut baris: Vec<String> = rekod.iter().map(|s| s.to_string()).collect();
        baris.push(nilai_baru);
        penulis.write_record(&baris)?;
    }

    penulis.flush()?;
    Ok(())
}

fn main() -> Result<(), Box<dyn Error>> {
    let data = "no_pekerja,nama,bahagian,gaji,aktif\n\
                KADA001,Ali Ahmad,ICT,4500.00,true\n\
                KADA002,Siti Hawa,HR,3800.00,true";

    std::fs::write("input.csv", data)?;

    // Pilih kolum tertentu sahaja
    pilih_kolum("input.csv", &["no_pekerja", "nama", "gaji"], "output_pilih.csv")?;

    // Tambah kolum gaji_bersih
    tambah_kolum(
        "input.csv",
        "gaji_bersih",
        |rekod| {
            let gaji: f64 = rekod[3].parse().unwrap_or(0.0);
            format!("{:.2}", gaji * 0.89)
        },
        "output_tambah.csv",
    )?;

    // Papar hasil
    println!("Kolum terpilih:");
    let mut r = Reader::from_path("output_pilih.csv")?;
    for rekod in r.records() { println!("  {:?}", rekod?); }

    println!("\nDengan kolum baru:");
    let mut r = Reader::from_path("output_tambah.csv")?;
    for rekod in r.records() { println!("  {:?}", rekod?); }

    // Bersihkan
    for f in &["input.csv", "output_pilih.csv", "output_tambah.csv"] {
        let _ = std::fs::remove_file(f);
    }

    Ok(())
}
```

---

# BAHAGIAN 6: Aggregate & Statistik 📈

## Kira Statistik dari CSV

```rust
use csv::Reader;
use serde::Deserialize;
use std::collections::HashMap;
use std::error::Error;

#[derive(Debug, Deserialize)]
struct Pekerja {
    no_pekerja: String,
    nama:       String,
    bahagian:   String,
    gaji:       f64,
    aktif:      bool,
}

fn main() -> Result<(), Box<dyn Error>> {
    let data = "no_pekerja,nama,bahagian,gaji,aktif\n\
                KADA001,Ali Ahmad,ICT,4500.00,true\n\
                KADA002,Siti Hawa,HR,3800.00,true\n\
                KADA003,Amin Razak,ICT,5000.00,true\n\
                KADA004,Zara Lina,Kewangan,4200.00,false\n\
                KADA005,Razi Mahir,HR,3500.00,true\n\
                KADA006,Nora Hashim,ICT,6000.00,true\n\
                KADA007,Hadi Jamil,Kewangan,4800.00,true";

    let mut pembaca = Reader::from_reader(data.as_bytes());
    let pekerja_list: Vec<Pekerja> = pembaca.deserialize()
        .collect::<Result<Vec<_>, _>>()?;

    // ── Aggregate asas ────────────────────────────────────────
    let jumlah = pekerja_list.len();
    let jumlah_aktif = pekerja_list.iter().filter(|p| p.aktif).count();

    let gaji_jumlah: f64 = pekerja_list.iter().map(|p| p.gaji).sum();
    let gaji_purata = gaji_jumlah / jumlah as f64;
    let gaji_max = pekerja_list.iter().map(|p| p.gaji).fold(f64::NEG_INFINITY, f64::max);
    let gaji_min = pekerja_list.iter().map(|p| p.gaji).fold(f64::INFINITY, f64::min);

    println!("=== Statistik Pekerja ===");
    println!("Jumlah pekerja:  {}", jumlah);
    println!("Pekerja aktif:   {}", jumlah_aktif);
    println!("Gaji purata:     RM{:.2}", gaji_purata);
    println!("Gaji tertinggi:  RM{:.2}", gaji_max);
    println!("Gaji terendah:   RM{:.2}", gaji_min);
    println!("Jumlah gaji:     RM{:.2}", gaji_jumlah);

    // ── Group by bahagian ─────────────────────────────────────
    let mut ikut_bahagian: HashMap<&str, Vec<&Pekerja>> = HashMap::new();
    for p in &pekerja_list {
        ikut_bahagian.entry(&p.bahagian).or_default().push(p);
    }

    println!("\n=== Mengikut Bahagian ===");
    let mut bahagian_list: Vec<&str> = ikut_bahagian.keys().copied().collect();
    bahagian_list.sort();

    for bahagian in &bahagian_list {
        let kumpulan = &ikut_bahagian[bahagian];
        let jumlah_bah = kumpulan.len();
        let gaji_bah: f64 = kumpulan.iter().map(|p| p.gaji).sum();
        let purata_bah = gaji_bah / jumlah_bah as f64;

        println!("  {:<12} {:>3} orang  RM{:>8.2} purata  RM{:>10.2} jumlah",
            bahagian, jumlah_bah, purata_bah, gaji_bah);
    }

    // ── Distribusi gaji ───────────────────────────────────────
    println!("\n=== Distribusi Gaji ===");
    let julat = [
        (0.0_f64, 3500.0, "< RM3,500"),
        (3500.0, 4500.0,  "RM3,500 - RM4,500"),
        (4500.0, 6000.0,  "RM4,500 - RM6,000"),
        (6000.0, f64::MAX,"≥ RM6,000"),
    ];

    for (min, max, label) in &julat {
        let kiraan = pekerja_list.iter()
            .filter(|p| p.gaji >= *min && p.gaji < *max)
            .count();
        let bar = "█".repeat(kiraan * 5);
        println!("  {:20} {:>2} {}", label, kiraan, bar);
    }

    // ── Peratus per bahagian ──────────────────────────────────
    println!("\n=== Peratus Bahagian ===");
    for bahagian in &bahagian_list {
        let kiraan = ikut_bahagian[bahagian].len();
        let peratus = kiraan as f64 / jumlah as f64 * 100.0;
        let bar = "▓".repeat((peratus / 5.0) as usize);
        println!("  {:12} {:>5.1}% {}", bahagian, peratus, bar);
    }

    Ok(())
}
```

---

## Sorted & Top-N

```rust
use serde::Deserialize;
use std::error::Error;

#[derive(Debug, Deserialize, Clone)]
struct Pekerja {
    no_pekerja: String,
    nama:       String,
    bahagian:   String,
    gaji:       f64,
}

fn main() -> Result<(), Box<dyn Error>> {
    let data = "no_pekerja,nama,bahagian,gaji\n\
                KADA001,Ali Ahmad,ICT,4500.00\n\
                KADA002,Siti Hawa,HR,3800.00\n\
                KADA003,Amin Razak,ICT,5000.00\n\
                KADA004,Zara Lina,Kewangan,4200.00\n\
                KADA005,Razi Mahir,HR,3500.00";

    let mut pembaca = csv::Reader::from_reader(data.as_bytes());
    let mut pekerja: Vec<Pekerja> = pembaca.deserialize()
        .collect::<Result<Vec<_>, _>>()?;

    // ── Sort mengikut gaji (descending) ───────────────────────
    pekerja.sort_by(|a, b| b.gaji.partial_cmp(&a.gaji).unwrap());
    println!("Top 3 gaji tertinggi:");
    for (i, p) in pekerja.iter().take(3).enumerate() {
        println!("  {}. {} — RM{:.2}", i + 1, p.nama, p.gaji);
    }

    // ── Sort mengikut nama (ascending) ────────────────────────
    pekerja.sort_by(|a, b| a.nama.cmp(&b.nama));
    println!("\nMengikut nama (A-Z):");
    for p in &pekerja { println!("  {}", p.nama); }

    // ── Sort pelbagai kriteria ─────────────────────────────────
    pekerja.sort_by(|a, b| {
        a.bahagian.cmp(&b.bahagian)
            .then(b.gaji.partial_cmp(&a.gaji).unwrap())
    });
    println!("\nMengikut bahagian kemudian gaji:");
    for p in &pekerja {
        println!("  {:12} {} — RM{:.2}", p.bahagian, p.nama, p.gaji);
    }

    Ok(())
}
```

---

# BAHAGIAN 7: Join Pelbagai CSV 🔗

## Inner Join

```rust
use csv::Reader;
use serde::Deserialize;
use std::collections::HashMap;
use std::error::Error;

#[derive(Debug, Deserialize, Clone)]
struct Pekerja {
    id:       u32,
    nama:     String,
    jabatan_id: u32,
}

#[derive(Debug, Deserialize, Clone)]
struct Jabatan {
    id:   u32,
    nama: String,
    kod:  String,
}

#[derive(Debug)]
struct PekerjaJabatan {
    id_pekerja:   u32,
    nama_pekerja: String,
    jabatan:      String,
    kod_jabatan:  String,
}

fn main() -> Result<(), Box<dyn Error>> {
    let csv_pekerja = "id,nama,jabatan_id\n\
                       1,Ali Ahmad,10\n\
                       2,Siti Hawa,20\n\
                       3,Amin Razak,10\n\
                       4,Zara Lina,30";

    let csv_jabatan = "id,nama,kod\n\
                       10,ICT,ICT001\n\
                       20,HR,HR001\n\
                       30,Kewangan,KEW001";

    // Baca jabatan ke HashMap (lookup table)
    let mut pembaca_jab = Reader::from_reader(csv_jabatan.as_bytes());
    let jabatan_map: HashMap<u32, Jabatan> = pembaca_jab
        .deserialize::<Jabatan>()
        .filter_map(|r| r.ok())
        .map(|j| (j.id, j))
        .collect();

    // Join dengan pekerja
    let mut pembaca_pek = Reader::from_reader(csv_pekerja.as_bytes());
    let mut hasil = Vec::new();

    for rekod in pembaca_pek.deserialize::<Pekerja>() {
        let pekerja = rekod?;
        // Inner join — skip kalau jabatan tidak jumpa
        if let Some(jabatan) = jabatan_map.get(&pekerja.jabatan_id) {
            hasil.push(PekerjaJabatan {
                id_pekerja:   pekerja.id,
                nama_pekerja: pekerja.nama,
                jabatan:      jabatan.nama.clone(),
                kod_jabatan:  jabatan.kod.clone(),
            });
        }
    }

    println!("Hasil JOIN:");
    for h in &hasil {
        println!("  {} ({}) — {} [{}]",
            h.nama_pekerja, h.id_pekerja,
            h.jabatan, h.kod_jabatan);
    }

    Ok(())
}
```

---

## Merge CSV (UNION)

```rust
use csv::{Reader, Writer};
use std::error::Error;
use std::collections::HashSet;

// Gabung dua CSV — buang duplikat berdasarkan kolum kunci
fn union_csv(
    fail1:          &str,
    fail2:          &str,
    laluan_keluar:  &str,
    kolum_kunci:    usize,    // indeks kolum kunci untuk dedup
) -> Result<usize, Box<dyn Error>> {
    let mut penulis = Writer::from_path(laluan_keluar)?;
    let mut kunci_set: HashSet<String> = HashSet::new();
    let mut kiraan = 0;
    let mut header_ditulis = false;

    for laluan in &[fail1, fail2] {
        let mut pembaca = Reader::from_path(laluan)?;

        // Tulis header dari fail pertama sahaja
        if !header_ditulis {
            let header = pembaca.headers()?.clone();
            penulis.write_record(&header)?;
            header_ditulis = true;
        } else {
            // Skip header fail kedua
            let _ = pembaca.headers()?;
        }

        for rekod in pembaca.records() {
            let rekod = rekod?;
            let kunci = rekod[kolum_kunci].to_string();

            if kunci_set.insert(kunci) {
                // Baru — belum ada, tulis
                penulis.write_record(&rekod)?;
                kiraan += 1;
            }
            // Duplikat — skip
        }
    }

    penulis.flush()?;
    Ok(kiraan)
}

fn main() -> Result<(), Box<dyn Error>> {
    let data1 = "id,nama,gaji\n1,Ali,4500\n2,Siti,3800";
    let data2 = "id,nama,gaji\n2,Siti,3800\n3,Amin,4200";  // Siti adalah duplikat

    std::fs::write("csv1.csv", data1)?;
    std::fs::write("csv2.csv", data2)?;

    let kiraan = union_csv("csv1.csv", "csv2.csv", "gabung.csv", 0)?;
    println!("Rekod unik: {}", kiraan);  // 3 (Siti dikira sekali)

    let mut r = Reader::from_path("gabung.csv")?;
    for rekod in r.records() { println!("{:?}", rekod?); }

    for f in &["csv1.csv", "csv2.csv", "gabung.csv"] {
        let _ = std::fs::remove_file(f);
    }

    Ok(())
}
```

---

# BAHAGIAN 8: CSV Besar & Streaming 🌊

## Proses Fail Besar Tanpa Load ke Memory

```rust
use csv::{Reader, Writer};
use serde::{Serialize, Deserialize};
use std::error::Error;
use std::io::BufReader;
use std::fs::File;

#[derive(Deserialize)]
struct TransaksiBesar {
    id:       u64,
    amaun:    f64,
    kategori: String,
    status:   String,
}

#[derive(Serialize)]
struct RingkasanKategori {
    kategori:      String,
    jumlah_rekod:  u64,
    jumlah_amaun:  f64,
    amaun_purata:  f64,
}

fn proses_streaming(
    laluan_masuk:  &str,
    laluan_keluar: &str,
) -> Result<(), Box<dyn Error>> {
    // Reader dengan buffer besar untuk fail besar
    let fail = File::open(laluan_masuk)?;
    let buf = BufReader::with_capacity(1024 * 1024, fail);  // 1MB buffer

    let mut pembaca = csv::ReaderBuilder::new()
        .buffer_capacity(1024 * 1024)
        .from_reader(buf);

    let mut kategori_data: std::collections::HashMap<String, (u64, f64)> =
        std::collections::HashMap::new();

    let mut kiraan_rekod = 0u64;

    // Proses SATU rekod pada satu masa — tiada load semua ke memory!
    for rekod in pembaca.deserialize::<TransaksiBesar>() {
        let t = rekod?;
        kiraan_rekod += 1;

        if t.status == "selesai" {
            let entry = kategori_data.entry(t.kategori).or_insert((0, 0.0));
            entry.0 += 1;
            entry.1 += t.amaun;
        }

        // Laporan kemajuan setiap 100,000 rekod
        if kiraan_rekod % 100_000 == 0 {
            print!("\rDiproses: {} rekod", kiraan_rekod);
            std::io::Write::flush(&mut std::io::stdout())?;
        }
    }

    println!("\nSelesai: {} rekod diproses", kiraan_rekod);

    // Tulis ringkasan
    let mut penulis = Writer::from_path(laluan_keluar)?;
    let mut ringkasan: Vec<RingkasanKategori> = kategori_data
        .into_iter()
        .map(|(kategori, (kiraan, jumlah))| RingkasanKategori {
            kategori,
            jumlah_rekod: kiraan,
            jumlah_amaun: jumlah,
            amaun_purata: jumlah / kiraan as f64,
        })
        .collect();

    ringkasan.sort_by(|a, b| b.jumlah_amaun.partial_cmp(&a.jumlah_amaun).unwrap());

    for r in &ringkasan {
        penulis.serialize(r)?;
    }

    penulis.flush()?;
    Ok(())
}

// Proses CSV selari dengan rayon
fn proses_selari(laluan: &str) -> Result<f64, Box<dyn Error>> {
    let kandungan = std::fs::read_to_string(laluan)?;
    let baris: Vec<&str> = kandungan.lines().skip(1).collect();  // skip header

    use rayon::prelude::*;
    let jumlah: f64 = baris.par_iter()
        .filter_map(|baris| {
            let bahagian: Vec<&str> = baris.split(',').collect();
            bahagian.get(1)?.parse::<f64>().ok()
        })
        .sum();

    Ok(jumlah)
}

fn main() -> Result<(), Box<dyn Error>> {
    // Demo dengan fail kecil
    println!("Pemprosesan streaming CSV besar...");
    Ok(())
}
```

---

# BAHAGIAN 9: Ralat & Data Kotor 🧹

## Handle Data Tidak Seragam

```rust
use csv::{Reader, ReaderBuilder, Trim};
use serde::Deserialize;
use std::error::Error;

#[derive(Debug, Deserialize)]
struct RekodKotor {
    id:       Option<String>,    // mungkin kosong
    nama:     Option<String>,
    gaji:     Option<String>,    // mungkin bukan nombor
    umur:     Option<String>,    // mungkin negatif atau sangat besar
}

#[derive(Debug)]
struct RekodBersih {
    id:   u32,
    nama: String,
    gaji: f64,
    umur: u8,
}

#[derive(Debug)]
enum RalatData {
    IdTiada,
    NamaTiada,
    GajiTidakSah(String),
    UmurTidakSah(String),
    UmurLuarJulat(u32),
}

fn bersihkan_rekod(kotor: RekodKotor) -> Result<RekodBersih, RalatData> {
    let id_str = kotor.id.filter(|s| !s.trim().is_empty())
        .ok_or(RalatData::IdTiada)?;
    let id: u32 = id_str.trim().parse()
        .map_err(|_| RalatData::GajiTidakSah(id_str.clone()))?;

    let nama = kotor.nama
        .map(|n| n.trim().to_string())
        .filter(|n| !n.is_empty())
        .ok_or(RalatData::NamaTiada)?;

    let gaji_str = kotor.gaji.unwrap_or_default();
    let gaji: f64 = gaji_str.trim()
        .replace(',', "")  // buang koma (1,500.00 → 1500.00)
        .replace("RM", "")
        .trim()
        .parse()
        .map_err(|_| RalatData::GajiTidakSah(gaji_str.clone()))?;

    let umur_str = kotor.umur.unwrap_or_default();
    let umur_u32: u32 = umur_str.trim().parse()
        .map_err(|_| RalatData::UmurTidakSah(umur_str.clone()))?;

    if umur_u32 < 18 || umur_u32 > 70 {
        return Err(RalatData::UmurLuarJulat(umur_u32));
    }

    Ok(RekodBersih {
        id,
        nama,
        gaji,
        umur: umur_u32 as u8,
    })
}

fn main() -> Result<(), Box<dyn Error>> {
    // Data dengan pelbagai masalah
    let data_kotor = "id,nama,gaji,umur\n\
                      1,Ali Ahmad,4500.00,28\n\
                      2, ,3800.00,30\n\
                      3,Amin,TIGA RIBU,22\n\
                      4,Zara,4200.00,200\n\
                      5,Razi,\"RM 3,500.00\",35\n\
                      ,Hadi,4800.00,40";

    let mut pembaca = ReaderBuilder::new()
        .trim(Trim::All)           // trim semua whitespace
        .flexible(true)            // biar field tidak sama bilangan
        .from_reader(data_kotor.as_bytes());

    let mut bersih = Vec::new();
    let mut ralat = Vec::new();

    for (no, rekod) in pembaca.deserialize::<RekodKotor>().enumerate() {
        match rekod {
            Ok(kotor) => {
                match bersihkan_rekod(kotor) {
                    Ok(r)  => { println!("✔ Baris {}: {:?}", no + 2, r); bersih.push(r); }
                    Err(e) => { println!("✗ Baris {}: {:?}", no + 2, e); ralat.push((no + 2, e)); }
                }
            }
            Err(e) => {
                println!("✗ Baris {}: CSV parse error: {}", no + 2, e);
            }
        }
    }

    println!("\nRingkasan:");
    println!("  Berjaya: {} rekod", bersih.len());
    println!("  Ralat:   {} rekod", ralat.len());

    Ok(())
}
```

---

# BAHAGIAN 10: Mini Project — Sistem Laporan Gaji 💰

```rust
use csv::{Reader, Writer, ReaderBuilder, Trim};
use serde::{Serialize, Deserialize};
use std::collections::HashMap;
use std::error::Error;
use std::io::Write;

// ─── Structs ──────────────────────────────────────────────────

#[derive(Debug, Deserialize, Clone)]
struct PekerjaMaster {
    no_pekerja: String,
    nama:       String,
    bahagian:   String,
    jawatan:    String,
    gaji_asas:  f64,
    tarikh_join: String,
}

#[derive(Debug, Deserialize, Clone)]
struct DataKehadiran {
    no_pekerja:  String,
    hari_hadir:  u32,
    hari_kerja:  u32,
    ot_jam:      f64,
}

#[derive(Debug, Serialize)]
struct LaporanGajiBulanan {
    no_pekerja:   String,
    nama:         String,
    bahagian:     String,
    gaji_asas:    f64,
    elaun_ot:     f64,
    elaun_kehadiran: f64,
    potongan_cuti: f64,
    potongan_kwsp: f64,
    potongan_socso: f64,
    gaji_kasar:   f64,
    gaji_bersih:  f64,
    status_bayar: String,
}

#[derive(Debug, Serialize)]
struct RingkasanBahagian {
    bahagian:     String,
    jumlah_staf:  usize,
    jumlah_gaji:  f64,
    gaji_purata:  f64,
    gaji_tertinggi: f64,
    gaji_terendah:  f64,
}

// ─── Pengiraan ────────────────────────────────────────────────

fn kira_laporan_gaji(
    pekerja: &PekerjaMaster,
    kehadiran: &DataKehadiran,
) -> LaporanGajiBulanan {
    let kadar_harian = pekerja.gaji_asas / kehadiran.hari_kerja as f64;

    // Elaun overtime (1.5x kadar sejam, andai 8 jam sehari)
    let kadar_sejam = pekerja.gaji_asas / (kehadiran.hari_kerja as f64 * 8.0);
    let elaun_ot = kehadiran.ot_jam * kadar_sejam * 1.5;

    // Elaun kehadiran penuh (RM200 kalau hadir semua hari)
    let elaun_kehadiran = if kehadiran.hari_hadir == kehadiran.hari_kerja {
        200.0
    } else {
        0.0
    };

    // Potongan cuti tanpa gaji
    let hari_cuti = kehadiran.hari_kerja.saturating_sub(kehadiran.hari_hadir) as f64;
    let potongan_cuti = hari_cuti * kadar_harian;

    let gaji_kasar = pekerja.gaji_asas + elaun_ot + elaun_kehadiran;

    // Potongan wajib
    let potongan_kwsp  = gaji_kasar * 0.11;   // 11% KWSP pekerja
    let potongan_socso = gaji_kasar.min(4000.0) * 0.005; // 0.5% SOCSO

    let gaji_bersih = gaji_kasar - potongan_cuti - potongan_kwsp - potongan_socso;

    LaporanGajiBulanan {
        no_pekerja:      pekerja.no_pekerja.clone(),
        nama:            pekerja.nama.clone(),
        bahagian:        pekerja.bahagian.clone(),
        gaji_asas:       (pekerja.gaji_asas * 100.0).round() / 100.0,
        elaun_ot:        (elaun_ot * 100.0).round() / 100.0,
        elaun_kehadiran,
        potongan_cuti:   (potongan_cuti * 100.0).round() / 100.0,
        potongan_kwsp:   (potongan_kwsp * 100.0).round() / 100.0,
        potongan_socso:  (potongan_socso * 100.0).round() / 100.0,
        gaji_kasar:      (gaji_kasar * 100.0).round() / 100.0,
        gaji_bersih:     (gaji_bersih * 100.0).round() / 100.0,
        status_bayar:    "Belum Bayar".into(),
    }
}

// ─── Main ──────────────────────────────────────────────────────

fn main() -> Result<(), Box<dyn Error>> {
    // Data master pekerja
    let csv_master = "no_pekerja,nama,bahagian,jawatan,gaji_asas,tarikh_join\n\
        KADA001,Ali Ahmad,ICT,Pengaturcara,4500.00,2018-03-15\n\
        KADA002,Siti Hawa,HR,Pegawai HR,3800.00,2020-07-01\n\
        KADA003,Amin Razak,ICT,Penganalisis,5500.00,2015-01-10\n\
        KADA004,Zara Lina,Kewangan,Akauntan,4200.00,2019-06-20\n\
        KADA005,Razi Mahir,HR,Penolong HR,3500.00,2022-02-14";

    // Data kehadiran bulan ini
    let csv_kehadiran = "no_pekerja,hari_hadir,hari_kerja,ot_jam\n\
        KADA001,22,22,5.0\n\
        KADA002,20,22,0.0\n\
        KADA003,22,22,8.5\n\
        KADA004,19,22,2.0\n\
        KADA005,22,22,0.0";

    println!("{'═'*70}");
    println!("{:^70}", "SISTEM LAPORAN GAJI KADA — JAN 2024");
    println!("{'═'*70}");

    // ── 1. Baca kedua-dua CSV ──────────────────────────────────
    let mut r_master = ReaderBuilder::new().trim(Trim::All)
        .from_reader(csv_master.as_bytes());
    let pekerja_list: Vec<PekerjaMaster> = r_master.deserialize()
        .collect::<Result<_, _>>()?;

    let mut r_hadir = ReaderBuilder::new().trim(Trim::All)
        .from_reader(csv_kehadiran.as_bytes());
    let hadir_map: HashMap<String, DataKehadiran> = r_hadir
        .deserialize::<DataKehadiran>()
        .filter_map(|r| r.ok())
        .map(|k| (k.no_pekerja.clone(), k))
        .collect();

    // ── 2. Kira laporan gaji setiap pekerja ───────────────────
    let mut laporan_list: Vec<LaporanGajiBulanan> = Vec::new();

    for pekerja in &pekerja_list {
        if let Some(kehadiran) = hadir_map.get(&pekerja.no_pekerja) {
            let laporan = kira_laporan_gaji(pekerja, kehadiran);
            laporan_list.push(laporan);
        }
    }

    // ── 3. Papar laporan ──────────────────────────────────────
    println!("\n{:<10} {:<16} {:<12} {:>10} {:>10} {:>10}",
        "No", "Nama", "Bahagian", "Gaji Asas", "Gaji Kasar", "Gaji Bersih");
    println!("{}", "-".repeat(70));

    let mut jumlah_bersih = 0.0f64;
    for l in &laporan_list {
        println!("{:<10} {:<16} {:<12} RM{:>8.2} RM{:>8.2} RM{:>8.2}",
            l.no_pekerja, l.nama, l.bahagian,
            l.gaji_asas, l.gaji_kasar, l.gaji_bersih);
        jumlah_bersih += l.gaji_bersih;
    }

    println!("{}", "=".repeat(70));
    println!("{:>52} RM{:>8.2}", "JUMLAH PERLU DIBAYAR:", jumlah_bersih);

    // ── 4. Ringkasan mengikut bahagian ────────────────────────
    println!("\n=== Ringkasan Mengikut Bahagian ===");
    let mut ikut_bahagian: HashMap<&str, Vec<&LaporanGajiBulanan>> = HashMap::new();
    for l in &laporan_list {
        ikut_bahagian.entry(&l.bahagian).or_default().push(l);
    }

    let mut ringkasan_list: Vec<RingkasanBahagian> = ikut_bahagian.iter()
        .map(|(&bahagian, senarai)| {
            let jumlah: f64 = senarai.iter().map(|l| l.gaji_bersih).sum();
            let purata = jumlah / senarai.len() as f64;
            let tertinggi = senarai.iter().map(|l| l.gaji_bersih).fold(f64::NEG_INFINITY, f64::max);
            let terendah  = senarai.iter().map(|l| l.gaji_bersih).fold(f64::INFINITY, f64::min);

            RingkasanBahagian {
                bahagian:       bahagian.into(),
                jumlah_staf:    senarai.len(),
                jumlah_gaji:    (jumlah * 100.0).round() / 100.0,
                gaji_purata:    (purata * 100.0).round() / 100.0,
                gaji_tertinggi: (tertinggi * 100.0).round() / 100.0,
                gaji_terendah:  (terendah * 100.0).round() / 100.0,
            }
        })
        .collect();

    ringkasan_list.sort_by(|a, b| b.jumlah_gaji.partial_cmp(&a.jumlah_gaji).unwrap());

    for r in &ringkasan_list {
        println!("  {:<12} {:>2} staf  RM{:>10.2} jumlah  RM{:>8.2} purata",
            r.bahagian, r.jumlah_staf, r.jumlah_gaji, r.gaji_purata);
    }

    // ── 5. Export ke CSV ─────────────────────────────────────
    let mut penulis_laporan = Writer::from_path("laporan_gaji_jan2024.csv")?;
    for l in &laporan_list {
        penulis_laporan.serialize(l)?;
    }
    penulis_laporan.flush()?;

    let mut penulis_ring = Writer::from_path("ringkasan_bahagian.csv")?;
    for r in &ringkasan_list {
        penulis_ring.serialize(r)?;
    }
    penulis_ring.flush()?;

    println!("\n✔ Laporan diekspot:");
    println!("  - laporan_gaji_jan2024.csv");
    println!("  - ringkasan_bahagian.csv");

    // Bersihkan
    let _ = std::fs::remove_file("laporan_gaji_jan2024.csv");
    let _ = std::fs::remove_file("ringkasan_bahagian.csv");

    println!("{'═'*70}");
    Ok(())
}
```

---

# 📋 Rujukan Pantas — CSV Cheat Sheet

## Setup

```toml
csv   = "1"
serde = { version = "1", features = ["derive"] }
```

## Baca CSV

```rust
// Mudah
let mut r = Reader::from_path("fail.csv")?;
let mut r = Reader::from_reader(data.as_bytes());

// Konfigurasi
let mut r = ReaderBuilder::new()
    .delimiter(b',')       // atau b';', b'\t'
    .has_headers(true)
    .trim(Trim::All)
    .flexible(true)        // field tidak sama bilangan
    .from_path("fail.csv")?;

// Iterate
for rekod in r.records() {        // StringRecord
    let rekod = rekod?;
    let nilai = &rekod[0];
}

for rekod in r.deserialize::<MyStruct>() { // serde
    let item: MyStruct = rekod?;
}

// Header
let header = r.headers()?.clone();
```

## Tulis CSV

```rust
let mut w = Writer::from_path("output.csv")?;
let mut w = Writer::from_writer(vec![]);   // ke Vec<u8>

// Tulis
w.write_record(&["a", "b", "c"])?;
w.write_record(&rekod)?;
w.serialize(&struct_item)?;    // dengan serde

w.flush()?;  // PENTING!

// Keluar String (untuk from_writer(vec![]))
let output = String::from_utf8(w.into_inner()?)?;
```

## Serde Attributes

```rust
#[derive(Deserialize, Serialize)]
struct Data {
    #[serde(rename = "Column Name")]    // nama dalam CSV
    field: String,
    #[serde(skip_deserializing)]        // tidak dalam CSV
    computed: f64,
}
```

## Pattern Penting

```rust
// Baca semua ke Vec
let semua: Vec<T> = r.deserialize().collect::<Result<_, _>>()?;

// Filter semasa baca
let aktif: Vec<T> = r.deserialize::<T>()
    .filter_map(|r| r.ok())
    .filter(|x| x.aktif)
    .collect();

// Group by
let mut kumpulan: HashMap<String, Vec<T>> = HashMap::new();
for item in senarai {
    kumpulan.entry(item.kunci.clone()).or_default().push(item);
}

// Sort
senarai.sort_by(|a, b| b.nilai.partial_cmp(&a.nilai).unwrap());

// Statistik
let purata = senarai.iter().map(|x| x.nilai).sum::<f64>() / senarai.len() as f64;
```

---

## 🏆 Cabaran Akhir

Cuba implement salah satu:

1. **Deduplikator CSV** — cari dan buang baris duplikat berdasarkan kolum kunci
2. **CSV Diff** — bandingkan dua versi CSV, tunjukkan baris yang berubah, ditambah, atau dipadam
3. **CSV Validator** — validate data CSV terhadap rules (required, type, range, regex)
4. **CSV ke JSON** — converter dengan type detection automatik (nombor, boolean, tarikh)
5. **Laporan Pivot** — gabungkan data dari pelbagai CSV, jana pivot table

---

*CSV dalam Rust — laju, selamat, dan type-checked.*
*Serde memastikan data CSV dan struct Rust sentiasa konsisten.* 🦀
