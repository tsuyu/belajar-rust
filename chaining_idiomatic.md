# ⛓️ Idiomatic vs Bad Chaining dalam Rust

> "Chaining yang baik = satu aliran logik yang jelas.
>  Chaining yang buruk = satu baris yang perlu 10 minit untuk faham."
> Faham bila chain, bila break, dan bila Iterator > for loop.

---

## Premis

```
Rust menggalakkan chaining — especially untuk Iterator.
Tapi "boleh chain" tidak bermaksud "patut chain".

Good chain:
  data.iter()
      .filter(|x| x.aktif)
      .map(|x| x.gaji)
      .sum::<f64>()
  → Jelas: filter aktif, ambil gaji, jumlahkan

Bad chain:
  s.chars().filter(|c|c.is_alphabetic()).map(|c|if c.is_uppercase()
  {c.to_lowercase().next().unwrap()}else{c}).collect::<String>()
  → Penat baca, sukar debug, tidak boleh "step through"

Prinsip:
  1. Setiap baris satu transformasi
  2. Nama variable > chain panjang
  3. Break bila ada branching logik
  4. Readable > clever
  5. Debug-ability adalah feature
```

---

## Peta Pembelajaran

```
Bab 1  → Iterator Chaining yang Idiomatic
Bab 2  → Iterator Chaining yang Buruk
Bab 3  → Option/Result Chaining
Bab 4  → Builder Pattern Chaining
Bab 5  → Bila Perlu Break Chain
Bab 6  → Naming Intermediate Results
Bab 7  → Anti-patterns: "Clever" Code
Bab 8  → Performance vs Readability
Bab 9  → Debug Chain yang Panjang
Bab 10 → Mini Project: Refactor ke Idiomatic
```

---

# BAB 1: Iterator Chaining yang Idiomatic ✅

## Prinsip Chain yang Baik

```rust
// Prinsip:
//   ✔ Setiap .method() pada baris sendiri
//   ✔ Aliran logik jelas dari atas ke bawah
//   ✔ Nama chain menggambarkan hasil
//   ✔ Tidak lebih dari 5-7 method dalam satu chain
//   ✔ Boleh "baca seperti prosa"

struct Pekerja {
    nama:     String,
    bahagian: String,
    gaji:     f64,
    aktif:    bool,
}

fn main() {
    let pekerja = vec![
        Pekerja { nama: "Ali".into(),  bahagian: "ICT".into(),      gaji: 4500.0, aktif: true  },
        Pekerja { nama: "Siti".into(), bahagian: "HR".into(),       gaji: 3800.0, aktif: true  },
        Pekerja { nama: "Amin".into(), bahagian: "ICT".into(),      gaji: 5000.0, aktif: false },
        Pekerja { nama: "Zara".into(), bahagian: "Kewangan".into(), gaji: 4200.0, aktif: true  },
        Pekerja { nama: "Razi".into(), bahagian: "ICT".into(),      gaji: 4800.0, aktif: true  },
    ];

    // ── Contoh 1: Kira jumlah gaji ICT yang aktif ─────────────
    // Baca seperti prosa: ambil pekerja → filter aktif → filter ICT → ambil gaji → jumlah
    let jumlah_gaji_ict: f64 = pekerja.iter()
        .filter(|p| p.aktif)
        .filter(|p| p.bahagian == "ICT")
        .map(|p| p.gaji)
        .sum();

    println!("Jumlah gaji ICT aktif: RM{:.2}", jumlah_gaji_ict);

    // ── Contoh 2: Dapatkan nama pekerja aktif, sort, uppercase ─
    let nama_aktif: Vec<String> = pekerja.iter()
        .filter(|p| p.aktif)
        .map(|p| p.nama.to_uppercase())
        .collect();

    // sort berasingan dari collect (beza method)
    let mut nama_sorted = nama_aktif;
    nama_sorted.sort();

    println!("{:?}", nama_sorted);

    // ── Contoh 3: Top 3 gaji tertinggi ────────────────────────
    let mut gaji_list: Vec<f64> = pekerja.iter()
        .filter(|p| p.aktif)
        .map(|p| p.gaji)
        .collect();

    gaji_list.sort_by(|a, b| b.partial_cmp(a).unwrap());
    let top3: Vec<f64> = gaji_list.into_iter().take(3).collect();

    println!("Top 3: {:?}", top3);

    // ── Contoh 4: Group by bahagian ───────────────────────────
    use std::collections::HashMap;

    let ikut_bahagian: HashMap<&str, Vec<&str>> = pekerja.iter()
        .filter(|p| p.aktif)
        .fold(HashMap::new(), |mut map, p| {
            map.entry(&p.bahagian)
               .or_default()
               .push(&p.nama);
            map
        });

    for (bah, nama) in &ikut_bahagian {
        println!("{}: {:?}", bah, nama);
    }
}
```

---

## Idiomatic: Setiap Langkah Jelas

```rust
// ✔ IDIOMATIC — aliran atas ke bawah, satu langkah satu baris
fn statistik_gaji(pekerja: &[Pekerja]) -> (f64, f64, f64) {
    let gaji_aktif: Vec<f64> = pekerja.iter()
        .filter(|p| p.aktif)       // langkah 1: ambil yang aktif
        .map(|p| p.gaji)           // langkah 2: ambil gaji sahaja
        .collect();                // langkah 3: kumpul

    // Lebih jelas kalau kira berasingan
    let min = gaji_aktif.iter().cloned().fold(f64::INFINITY, f64::min);
    let max = gaji_aktif.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
    let purata = gaji_aktif.iter().sum::<f64>() / gaji_aktif.len() as f64;

    (min, max, purata)
}
```

---

## 🧠 Brain Teaser #1

Yang mana lebih baik? Kenapa?

```rust
// Versi A
fn v1(data: &[i32]) -> Vec<i32> {
    data.iter().filter(|&&x| x > 0).filter(|&&x| x % 2 == 0).map(|&x| x * x).take(5).collect()
}

// Versi B
fn v2(data: &[i32]) -> Vec<i32> {
    data.iter()
        .filter(|&&x| x > 0)
        .filter(|&&x| x % 2 == 0)
        .map(|&x| x * x)
        .take(5)
        .collect()
}

// Versi C
fn v3(data: &[i32]) -> Vec<i32> {
    let positif = data.iter().filter(|&&x| x > 0);
    let genap   = positif.filter(|&&x| x % 2 == 0);
    let kuasa2  = genap.map(|&x| x * x);
    kuasa2.take(5).collect()
}
```

<details>
<summary>👀 Jawapan</summary>

```
A: ❌ BURUK — satu baris panjang
   - Tidak boleh "scan" logik dengan cepat
   - Perlu scroll horizontal
   - Sukar tambah/buang step

B: ✔ BAIK — sama dengan A tapi lebih readable
   - Satu step satu baris
   - Boleh comment setiap langkah
   - Mudah tambah/buang step

C: ✔ BAIK JUGA — intermediate names yang bermakna
   - "positif", "genap", "kuasa2" menjelaskan intent
   - Lebih verbose tapi sangat jelas
   - Sesuai bila logic kompleks

Mana yang TERBAIK? Bergantung konteks:
  - Operasi mudah → B (chain yang terbaca)
  - Logik kompleks → C (nama intermediate yang bermakna)
  - Performance kritikal → B (compiler optimise chain sama)
  - Debugging → C (boleh inspect setiap peringkat)

TIDAK ADA perbezaan performance antara B dan C!
Rust compiler optimise kedua-duanya sama.
```
</details>

---

# BAB 2: Iterator Chaining yang Buruk ❌

## Apa yang Menjadikan Chain "Buruk"

```rust
// ── BURUK 1: Terlalu panjang dalam satu baris ─────────────────

// ❌ Ini adalah satu baris — siapa boleh baca?
let hasil = data.iter().filter(|x| x.aktif).map(|x| &x.nama).filter(|n| !n.is_empty()).map(|n| n.trim()).collect::<Vec<_>>();

// ✔ Sama, tapi readable
let hasil: Vec<&str> = data.iter()
    .filter(|x| x.aktif)
    .map(|x| x.nama.as_str())
    .filter(|n| !n.is_empty())
    .map(|n| n.trim())
    .collect();

// ── BURUK 2: Logik tersembunyi dalam closure panjang ──────────

// ❌ Closure yang terlalu panjang dalam chain
let hasil = pekerja.iter()
    .map(|p| { let g = p.gaji * if p.bahagian == "ICT" { 1.15 } else if p.bahagian == "HR" { 1.10 } else { 1.05 }; (p.nama.clone(), g) })
    .collect::<Vec<_>>();

// ✔ Extract logic ke fungsi atau variable
fn multiplier_bahagian(bahagian: &str) -> f64 {
    match bahagian {
        "ICT"      => 1.15,
        "HR"       => 1.10,
        _          => 1.05,
    }
}

let hasil: Vec<(String, f64)> = pekerja.iter()
    .map(|p| {
        let gaji_baru = p.gaji * multiplier_bahagian(&p.bahagian);
        (p.nama.clone(), gaji_baru)
    })
    .collect();

// ── BURUK 3: Side effect dalam chain ─────────────────────────

// ❌ Jangan guna map() untuk side effects!
let _ = pekerja.iter()
    .map(|p| {
        println!("{}", p.nama); // side effect dalam map!
        p
    })
    .collect::<Vec<_>>();

// ✔ Guna for_each() untuk side effects
pekerja.iter()
    .filter(|p| p.aktif)
    .for_each(|p| println!("{}", p.nama));

// ✔ Atau for loop biasa
for p in pekerja.iter().filter(|p| p.aktif) {
    println!("{}", p.nama);
}

// ── BURUK 4: Unwrap tersembunyi dalam chain ───────────────────

// ❌ unwrap() tersembunyi — panic tidak nampak!
let hasil: Vec<i32> = teks_list.iter()
    .map(|s| s.parse::<i32>().unwrap()) // kalau gagal parse → panic!
    .collect();

// ✔ Handle error dengan betul
let hasil: Result<Vec<i32>, _> = teks_list.iter()
    .map(|s| s.parse::<i32>())
    .collect();

// ✔ Atau skip yang gagal
let hasil: Vec<i32> = teks_list.iter()
    .filter_map(|s| s.parse::<i32>().ok())
    .collect();
```

---

## Buruk: Nested Chain

```rust
// ❌ BURUK — chain dalam chain dalam chain
let hasil = pekerja.iter()
    .map(|p| {
        p.rekod.iter()
            .filter(|r| r.tahun == 2024)
            .map(|r| r.nilai)
            .sum::<f64>()
            / p.rekod.iter()
                .filter(|r| r.tahun == 2024)
                .count() as f64
    })
    .collect::<Vec<f64>>();

// ✔ BAIK — extract ke fungsi
fn purata_rekod_2024(rekod: &[Rekod]) -> f64 {
    let rekod_2024: Vec<f64> = rekod.iter()
        .filter(|r| r.tahun == 2024)
        .map(|r| r.nilai)
        .collect();

    if rekod_2024.is_empty() {
        return 0.0;
    }

    rekod_2024.iter().sum::<f64>() / rekod_2024.len() as f64
}

let hasil: Vec<f64> = pekerja.iter()
    .map(|p| purata_rekod_2024(&p.rekod))
    .collect();
```

---

# BAB 3: Option/Result Chaining ✨

## Idiomatic Option Chain

```rust
use std::collections::HashMap;

struct DB {
    pekerja: HashMap<u32, Pekerja>,
}

impl DB {
    fn cari(&self, id: u32) -> Option<&Pekerja> {
        self.pekerja.get(&id)
    }
}

fn main() {
    let db = DB { pekerja: HashMap::new() };

    // ── Idiomatic: ? dalam fungsi ────────────────────────────
    fn dapatkan_emel_domain(db: &DB, id: u32) -> Option<&str> {
        let pekerja = db.cari(id)?;                    // langkah 1: cari pekerja
        let emel    = pekerja.emel.as_deref()?;        // langkah 2: ambil emel
        let domain  = emel.split('@').nth(1)?;          // langkah 3: ambil domain
        Some(domain)
    }

    // ── Idiomatic: and_then chain ────────────────────────────
    // Lebih ringkas bila tidak perlu intermediate names
    let domain = db.cari(1)
        .and_then(|p| p.emel.as_deref())
        .and_then(|e| e.split('@').nth(1));

    // ── Bila guna ? vs and_then? ─────────────────────────────
    // ? dalam fungsi = boleh ada logik antara setiap step
    // and_then = transformasi mudah tanpa logik kompleks

    // ? sesuai bila:
    fn proses(db: &DB, id: u32) -> Option<String> {
        let p = db.cari(id)?;
        if !p.aktif { return None; }  // logik tambahan!
        let e = p.emel.as_deref()?;
        Some(e.to_uppercase())
    }

    // and_then sesuai bila:
    let e = db.cari(1)
        .filter(|p| p.aktif)         // filter sebagai step
        .and_then(|p| p.emel.as_deref())
        .map(|e| e.to_uppercase());  // transform akhir
}
```

---

## Option Chain yang Buruk

```rust
// ❌ BURUK 1: Terlalu banyak transform dalam satu chain
let hasil = pekerja.emel.clone().and_then(|e| if e.contains('@') { Some(e) } else { None }).map(|e| { let parts: Vec<&str> = e.split('@').collect(); parts[1].to_string() }).map(|d| d.to_uppercase()).unwrap_or_else(|| "TIADA".to_string());

// ✔ BAIK: Pecah kepada fungsi atau langkah
fn dapatkan_domain_emel(emel: &Option<String>) -> String {
    emel.as_deref()
        .filter(|e| e.contains('@'))
        .and_then(|e| e.split('@').nth(1))
        .map(|d| d.to_uppercase())
        .unwrap_or_else(|| "TIADA".into())
}

// ❌ BURUK 2: Mix option dan result dalam cara yang mengelirukan
let hasil = std::fs::read_to_string("fail.txt")
    .ok()                    // Result → Option (buang error!)
    .and_then(|s| s.parse::<i32>().ok())  // buang parse error!
    .map(|n| n * 2);

// Masalah: silently drop errors — kenapa fail? tidak tahu!

// ✔ BAIK: Kekalkan error information
fn baca_dan_parse(laluan: &str) -> Result<i32, Box<dyn std::error::Error>> {
    let kandungan = std::fs::read_to_string(laluan)?;  // io::Error
    let nombor: i32 = kandungan.trim().parse()?;        // ParseIntError
    Ok(nombor * 2)
}
```

---

## Result Chaining yang Idiomatic

```rust
use std::io;
use serde::Deserialize;

#[derive(Deserialize)]
struct Konfig { host: String, port: u16 }

fn muat_konfig(laluan: &str) -> Result<Konfig, Box<dyn std::error::Error>> {
    // ── ? operator chain ─────────────────────────────────────
    // Setiap ? adalah satu kemungkinan gagal yang jelas
    let kandungan = std::fs::read_to_string(laluan)?;
    let konfig: Konfig = serde_json::from_str(&kandungan)?;

    // Validate
    if konfig.port < 1024 {
        return Err(format!("Port {} tidak dibenarkan (< 1024)", konfig.port).into());
    }

    Ok(konfig)
}

// ── Chain dengan context ──────────────────────────────────────
use anyhow::Context;

fn muat_konfig_dengan_context(laluan: &str) -> anyhow::Result<Konfig> {
    let kandungan = std::fs::read_to_string(laluan)
        .with_context(|| format!("Gagal baca '{}'", laluan))?;

    let konfig: Konfig = serde_json::from_str(&kandungan)
        .with_context(|| format!("Format JSON tidak sah dalam '{}'", laluan))?;

    Ok(konfig)
}
```

---

# BAB 4: Builder Pattern Chaining 🏗️

## Builder yang Idiomatic

```rust
// Builder pattern = antara contoh chaining terbaik dalam Rust
// Setiap method return self, boleh chain

#[derive(Debug)]
struct KonfigServer {
    host:      String,
    port:      u16,
    pekerja:   usize,
    timeout:   u64,
    ssl:       bool,
    log_level: String,
}

struct PembinaKonfig {
    konfig: KonfigServer,
}

impl PembinaKonfig {
    fn baru() -> Self {
        PembinaKonfig {
            konfig: KonfigServer {
                host:      "localhost".into(),
                port:      8080,
                pekerja:   4,
                timeout:   30,
                ssl:       false,
                log_level: "info".into(),
            }
        }
    }

    // Setiap method: transform self, return Self
    fn host(mut self, host: &str) -> Self {
        self.konfig.host = host.into(); self
    }
    fn port(mut self, port: u16) -> Self {
        self.konfig.port = port; self
    }
    fn pekerja(mut self, n: usize) -> Self {
        self.konfig.pekerja = n; self
    }
    fn timeout(mut self, saat: u64) -> Self {
        self.konfig.timeout = saat; self
    }
    fn ssl(mut self) -> Self {
        self.konfig.ssl = true; self
    }
    fn log_level(mut self, level: &str) -> Self {
        self.konfig.log_level = level.into(); self
    }

    fn bina(self) -> Result<KonfigServer, &'static str> {
        if self.konfig.port < 1024 && !self.konfig.ssl {
            return Err("Port < 1024 perlu SSL");
        }
        Ok(self.konfig)
    }
}

fn main() {
    // ✔ Builder chain yang idiomatic — dibaca seperti konfigurasi
    let konfig = PembinaKonfig::baru()
        .host("0.0.0.0")
        .port(8443)
        .pekerja(8)
        .timeout(60)
        .ssl()
        .log_level("warn")
        .bina()
        .unwrap();

    println!("{:#?}", konfig);

    // ❌ Builder yang buruk — semuanya dalam satu call
    // Konfig::new("0.0.0.0", 8443, 8, 60, true, "warn") → 6 parameter!
    // Tidak tahu parameter mana apa tanpa tengok signature
}
```

---

# BAB 5: Bila Perlu Break Chain 🔀

## Tanda-tanda Chain Perlu Dipecah

```rust
struct Rekod { tahun: u32, nilai: f64 }

fn main() {
    let pekerja: Vec<PekerjaData> = vec![]; // simplified

    // ── Tanda 1: Perlu semak intermediate result ──────────────

    // ❌ BURUK — kalau rekod_2024 kosong, akan panic (div by zero)
    let purata = pekerja.iter()
        .flat_map(|p| p.rekod.iter())
        .filter(|r| r.tahun == 2024)
        .map(|r| r.nilai)
        .sum::<f64>()
        / pekerja.iter() // kira dua kali!
            .flat_map(|p| p.rekod.iter())
            .filter(|r| r.tahun == 2024)
            .count() as f64;

    // ✔ BAIK — break untuk intermediate check
    let rekod_2024: Vec<f64> = pekerja.iter()
        .flat_map(|p| p.rekod.iter())
        .filter(|r| r.tahun == 2024)
        .map(|r| r.nilai)
        .collect();

    let purata = if rekod_2024.is_empty() {
        0.0
    } else {
        rekod_2024.iter().sum::<f64>() / rekod_2024.len() as f64
    };

    // ── Tanda 2: Logik branching dalam chain ──────────────────

    // ❌ BURUK — branching dalam map adalah tanda break diperlukan
    let hasil: Vec<_> = data.iter()
        .map(|item| {
            if item.kategori == "A" {
                // logik panjang untuk A
                proses_a(item)
            } else if item.kategori == "B" {
                // logik panjang untuk B
                proses_b(item)
            } else {
                // logik default
                proses_lain(item)
            }
        })
        .collect();

    // ✔ BAIK — extract fungsi
    fn proses_item(item: &Item) -> Hasil {
        match item.kategori.as_str() {
            "A" => proses_a(item),
            "B" => proses_b(item),
            _   => proses_lain(item),
        }
    }

    let hasil: Vec<_> = data.iter()
        .map(proses_item) // clean!
        .collect();

    // ── Tanda 3: Debug sukar ──────────────────────────────────

    // ❌ BURUK — kalau ada bug, di mana?
    let result = api_client.dapatkan_data(url)
        .await?
        .json::<Vec<Item>>()?
        .into_iter()
        .filter(|i| i.valid)
        .map(|i| Transform::apply(i))
        .filter(|r| r.score > threshold)
        .take(limit)
        .collect::<Vec<_>>();

    // ✔ BAIK — kalau ada error, tahu di mana
    let respons   = api_client.dapatkan_data(url).await?;
    let item_list = respons.json::<Vec<Item>>()?;
    let disaring  = item_list.iter()
        .filter(|i| i.valid)
        .map(|i| Transform::apply(i))
        .filter(|r| r.score > threshold)
        .take(limit)
        .collect::<Vec<_>>();
}
```

---

## 🧠 Brain Teaser #2

Kenapa versi A lebih baik dari B walaupun B lebih pendek?

```rust
// Data
let data: Vec<(String, i32)> = vec![
    ("ali".into(), 85),
    ("siti".into(), 92),
    ("amin".into(), 78),
    ("".into(), 0),     // data kotor
];

// Versi A (lebih panjang)
fn proses_a(data: &[(String, i32)]) -> Vec<String> {
    let bersih: Vec<&(String, i32)> = data.iter()
        .filter(|(nama, _)| !nama.is_empty())
        .collect();

    let lulus: Vec<&(String, i32)> = bersih.iter()
        .copied()
        .filter(|(_, skor)| *skor >= 80)
        .collect();

    lulus.iter()
        .map(|(nama, skor)| format!("{}: {}", nama.to_uppercase(), skor))
        .collect()
}

// Versi B (lebih pendek)
fn proses_b(data: &[(String, i32)]) -> Vec<String> {
    data.iter()
        .filter(|(n, s)| !n.is_empty() && *s >= 80)
        .map(|(n, s)| format!("{}: {}", n.to_uppercase(), s))
        .collect()
}
```

<details>
<summary>👀 Jawapan</summary>

```
SEBENARNYA B lebih baik dalam kes ini!

Kenapa?
  B lebih mudah dibaca — "filter yang tidak kosong dan lulus, kemudian format"
  A memperkenalkan intermediate Vec yang tidak diperlukan
  A mempunyai overhead .collect() dua kali

TAPI ada kes di mana A lebih baik:
  1. Kalau nak inspect "bersih" atau "lulus" secara berasingan (debugging)
  2. Kalau "bersih" atau "lulus" perlu digunakan di tempat lain
  3. Kalau logik filter sangat kompleks dan perlu nama yang bermakna

Panduan:
  - Boleh combine dua filter → combine (lebih ringkas)
  - Logik berbeza (cleanse vs business rule) → pisah
  - Perlu inspect intermediate → pisah dengan nama bermakna

Dalam contoh ini:
  B: filter(kosong && lulus) — dua syarat yang related, OK digabung
  Tapi kalau ada 5 syarat berbeza:
    .filter(|x| !x.kosong && x.skor >= 80 && x.bahagian == "ICT"
               && x.tahun == 2024 && !x.cuti)
    → Pecah kepada fungsi! Terlalu kompleks untuk satu filter.
```
</details>

---

# BAB 6: Naming Intermediate Results 🏷️

## Nama yang Bermakna vs Variable Sementara

```rust
fn main() {
    let data: Vec<Pekerja> = vec![];

    // ── Nama yang BERMAKNA ────────────────────────────────────

    // ✔ Nama menjelaskan APAKAH data ini
    let pekerja_aktif: Vec<&Pekerja> = data.iter()
        .filter(|p| p.aktif)
        .collect();

    let gaji_ict: Vec<f64> = pekerja_aktif.iter()
        .filter(|p| p.bahagian == "ICT")
        .map(|p| p.gaji)
        .collect();

    let jumlah_gaji_ict: f64 = gaji_ict.iter().sum();

    println!("Jumlah: {}", jumlah_gaji_ict);

    // ── BERBANDING nama yang tidak bermakna ───────────────────

    // ❌ temp, x, y, hasil — tidak memberitahu apa-apa
    let temp: Vec<&Pekerja> = data.iter()
        .filter(|p| p.aktif)
        .collect();

    let x: Vec<f64> = temp.iter()
        .filter(|p| p.bahagian == "ICT")
        .map(|p| p.gaji)
        .collect();

    let hasil: f64 = x.iter().sum();

    // ── Nama untuk complex transform ──────────────────────────

    // ✔ Nama chain akhir menjelaskan intent
    let laporan_gaji_bahagian: std::collections::HashMap<&str, f64> = data.iter()
        .filter(|p| p.aktif)
        .fold(std::collections::HashMap::new(), |mut map, p| {
            *map.entry(p.bahagian.as_str()).or_insert(0.0) += p.gaji;
            map
        });

    // ── Bila TIDAK perlu nama intermediate ───────────────────

    // ✔ Kalau digunakan terus, tidak perlu nama
    println!("Bil. aktif: {}", data.iter().filter(|p| p.aktif).count());

    // Kalau digunakan lebih dari sekali → baru perlu nama
    let aktif_count = data.iter().filter(|p| p.aktif).count();
    if aktif_count > 10 {
        println!("{} orang aktif (ramai!)", aktif_count);
    } else {
        println!("{} orang aktif", aktif_count);
    }
}
```

---

# BAB 7: Anti-patterns — "Clever" Code 🤓

## Clever vs Clear

```rust
// ══════════════════════════════════════════════════════════════
// ANTI-PATTERN 1: One-liner yang terlalu padat
// ══════════════════════════════════════════════════════════════

let data = vec!["1", "2", "abc", "4", "5"];

// ❌ Terlalu clever — sukar baca
let r:Vec<_>=data.iter().filter_map(|s|s.parse::<i32>().ok().filter(|&n|n>0&&n%2==0)).map(|n|format!("{}²={}",n,n*n)).collect();

// ✔ Jelas dan boleh dibaca
let nombor_genap_positif: Vec<String> = data.iter()
    .filter_map(|s| s.parse::<i32>().ok())  // parse, skip yang gagal
    .filter(|&n| n > 0 && n % 2 == 0)       // positif DAN genap
    .map(|n| format!("{}² = {}", n, n * n)) // format
    .collect();

// ══════════════════════════════════════════════════════════════
// ANTI-PATTERN 2: Guna iterator untuk sesuatu yang lebih jelas dengan loop
// ══════════════════════════════════════════════════════════════

// ❌ Iterator untuk sesuatu yang lebih natural sebagai loop
let mut kiraan = 0;
(0..100)
    .filter(|n| n % 3 == 0 || n % 5 == 0)
    .for_each(|_| kiraan += 1);

// ✔ for loop biasa lebih jelas
for n in 0..100 {
    if n % 3 == 0 || n % 5 == 0 {
        kiraan += 1;
    }
}

// ✔ Atau guna count() kalau memang nak count
let kiraan = (0..100)
    .filter(|n| n % 3 == 0 || n % 5 == 0)
    .count();

// ══════════════════════════════════════════════════════════════
// ANTI-PATTERN 3: .clone() semasa iterate
// ══════════════════════════════════════════════════════════════

let strings = vec!["hello".to_string(), "world".to_string()];

// ❌ Clone yang tidak perlu
let besar: Vec<String> = strings.iter()
    .map(|s| s.clone().to_uppercase()) // clone kemudian uppercase
    .collect();

// ✔ Guna .to_uppercase() terus (ia return String baru)
let besar: Vec<String> = strings.iter()
    .map(|s| s.to_uppercase()) // &String → String baru, tiada clone
    .collect();

// ══════════════════════════════════════════════════════════════
// ANTI-PATTERN 4: flatten() yang mengelirukan
// ══════════════════════════════════════════════════════════════

let data: Vec<Vec<i32>> = vec![vec![1,2], vec![3,4], vec![5,6]];

// ❌ Chained flat_map yang mengelirukan
let r: Vec<i32> = data.iter()
    .flat_map(|v| v.iter().flat_map(|&n| std::iter::once(n).chain(std::iter::once(n * 2))))
    .collect();

// ✔ Lebih jelas dengan fungsi
fn kembangkan(n: i32) -> [i32; 2] { [n, n * 2] }

let r: Vec<i32> = data.iter()
    .flatten()                    // [[1,2],[3,4]] → [1,2,3,4,5,6]
    .flat_map(|&n| kembangkan(n)) // setiap n → [n, n*2]
    .collect();

// ══════════════════════════════════════════════════════════════
// ANTI-PATTERN 5: Collect yang tidak perlu
// ══════════════════════════════════════════════════════════════

let nums = vec![1, 2, 3, 4, 5];

// ❌ Collect dua kali
let doubled: Vec<i32> = nums.iter()
    .map(|&n| n * 2)
    .collect::<Vec<_>>() // collect pertama
    .iter()
    .filter(|&&n| n > 4)
    .cloned()
    .collect(); // collect kedua

// ✔ Satu chain, satu collect
let doubled: Vec<i32> = nums.iter()
    .map(|&n| n * 2)
    .filter(|&n| n > 4)
    .collect();
```

---

# BAB 8: Performance vs Readability 📊

## Bila Performance Penting

```rust
// Rust iterator chains LAZY — tidak ada overhead untuk chain!
// .filter().map().filter() tidak create intermediate Vec!
// Hanya execute bila .collect() atau consume dipanggil.

use std::time::Instant;

fn benchmark() {
    let data: Vec<i32> = (0..1_000_000).collect();

    // ── Cara 1: for loop biasa ────────────────────────────────
    let mula = Instant::now();
    let mut hasil1 = Vec::new();
    for &n in &data {
        if n % 2 == 0 {
            hasil1.push(n * n);
        }
    }
    println!("For loop: {:?}", mula.elapsed());

    // ── Cara 2: Iterator chain ────────────────────────────────
    let mula = Instant::now();
    let hasil2: Vec<i32> = data.iter()
        .filter(|&&n| n % 2 == 0)
        .map(|&n| n * n)
        .collect();
    println!("Iterator: {:?}", mula.elapsed());

    // Biasanya: hampir sama! Iterator chain di-optimise oleh compiler.
    // Dalam sesetengah kes iterator lebih LAJU (SIMD vectorization).

    // ── Bila for loop LEBIH LAJU ─────────────────────────────
    // 1. Bila perlu break awal (iterator .take() berbeza semantik)
    // 2. Bila ada complex mutable state
    // 3. Bila ada early return dengan condition kompleks

    // ── Bila iterator LEBIH LAJU ─────────────────────────────
    // 1. Compiler boleh SIMD vectorize
    // 2. Fewer bounds checks (compiler prove)
    // 3. Parallel dengan rayon (par_iter())
}

// ── Rayon: parallel iterator ──────────────────────────────────
// use rayon::prelude::*;
//
// let hasil: Vec<i32> = data.par_iter() // parallel!
//     .filter(|&&n| n % 2 == 0)
//     .map(|&n| n * n)
//     .collect();
//
// Guna pada large data (> ~10,000 items) untuk keuntungan performance!
```

---

# BAB 9: Debug Chain yang Panjang 🔍

## Teknik Debug Iterator Chain

```rust
// Masalah: kalau ada bug dalam chain, di mana?

fn main() {
    let data = vec![1, -2, 3, -4, 5, 0, 6];

    // ── Teknik 1: .inspect() untuk debug ─────────────────────
    // .inspect() = "peek" tanpa mengubah chain
    let hasil: Vec<i32> = data.iter()
        .inspect(|&&n| eprintln!("[1] masuk: {}", n))
        .filter(|&&n| n > 0)
        .inspect(|&&n| eprintln!("[2] selepas filter positif: {}", n))
        .map(|&n| n * 2)
        .inspect(|&n| eprintln!("[3] selepas ×2: {}", n))
        .filter(|&n| n != 0)
        .inspect(|&n| eprintln!("[4] selepas filter 0: {}", n))
        .collect();

    println!("Hasil: {:?}", hasil);

    // Output (ke stderr):
    // [1] masuk: 1
    // [2] selepas filter positif: 1
    // [3] selepas ×2: 2
    // [4] selepas filter 0: 2
    // [1] masuk: -2
    // ... (yang negatif tidak lulus filter)
    // dll.

    // ── Teknik 2: Materialisasi untuk debug ───────────────────
    let selepas_filter_positif: Vec<i32> = data.iter()
        .filter(|&&n| n > 0)
        .cloned()
        .collect();
    eprintln!("Debug: {:?}", selepas_filter_positif);

    let selepas_double = selepas_filter_positif.iter()
        .map(|&n| n * 2)
        .collect::<Vec<_>>();
    eprintln!("Debug: {:?}", selepas_double);

    let akhir: Vec<i32> = selepas_double.into_iter()
        .filter(|&n| n != 0)
        .collect();
    eprintln!("Akhir: {:?}", akhir);

    // ── Teknik 3: dbg! macro ─────────────────────────────────
    let hasil: Vec<i32> = data.iter()
        .filter(|&&n| n > 0)
        .map(|&n| n * 2)
        .collect();

    dbg!(&hasil); // print nama variable + nilai + lokasi baris
    // [src/main.rs:45] &hasil = [2, 6, 10, 12]
}
```

---

# BAB 10: Mini Project — Refactor ke Idiomatic 🔧

```rust
// ── SEBELUM: "Baru belajar Rust" ──────────────────────────────

struct PekerjaRaw {
    nama:      String,
    bahagian:  String,
    gaji:      String,  // data kotor — string
    aktif:     String,  // "true"/"false" sebagai string
    tag:       String,  // "rust,python,java" — comma separated
}

struct PekerjaBersih {
    nama:      String,
    bahagian:  String,
    gaji:      f64,
    aktif:     bool,
    tag:       Vec<String>,
}

// ❌ BURUK — cara lama, tidak idiomatic
fn proses_buruk(data: Vec<PekerjaRaw>) -> Vec<PekerjaBersih> {
    let mut hasil = Vec::new();
    for p in &data {
        let nama = p.nama.trim().to_string();
        if nama.is_empty() {
            continue;
        }
        let gaji = match p.gaji.trim().parse::<f64>() {
            Ok(g) => g,
            Err(_) => continue,
        };
        if gaji < 0.0 {
            continue;
        }
        let aktif = p.aktif.trim() == "true";
        if !aktif {
            continue;
        }
        let tag: Vec<String> = p.tag.split(',')
            .map(|t| t.trim().to_string())
            .filter(|t| !t.is_empty())
            .collect();
        hasil.push(PekerjaBersih {
            nama,
            bahagian: p.bahagian.trim().to_string(),
            gaji,
            aktif,
            tag,
        });
    }
    hasil
}

// ✔ BAIK — idiomatic Rust

// Helper function untuk parse pekerja
fn parse_pekerja(raw: &PekerjaRaw) -> Option<PekerjaBersih> {
    let nama = raw.nama.trim();
    if nama.is_empty() { return None; }

    let gaji: f64 = raw.gaji.trim().parse().ok()?;
    if gaji < 0.0 { return None; }

    let aktif = raw.aktif.trim() == "true";
    if !aktif { return None; }

    let tag: Vec<String> = raw.tag
        .split(',')
        .map(|t| t.trim().to_string())
        .filter(|t| !t.is_empty())
        .collect();

    Some(PekerjaBersih {
        nama:     nama.to_string(),
        bahagian: raw.bahagian.trim().to_string(),
        gaji,
        aktif,
        tag,
    })
}

fn proses_baik(data: &[PekerjaRaw]) -> Vec<PekerjaBersih> {
    data.iter()
        .filter_map(parse_pekerja) // skip None
        .collect()
}

// ── ANALISIS LANJUTAN ──────────────────────────────────────────

struct LaporanBahagian {
    bahagian:        String,
    jumlah_pekerja:  usize,
    jumlah_gaji:     f64,
    gaji_purata:     f64,
    tag_popular:     Vec<String>,
}

fn jana_laporan(pekerja: &[PekerjaBersih]) -> Vec<LaporanBahagian> {
    use std::collections::HashMap;

    // Kumpul mengikut bahagian
    let mut ikut_bahagian: HashMap<&str, Vec<&PekerjaBersih>> = HashMap::new();
    for p in pekerja {
        ikut_bahagian.entry(&p.bahagian).or_default().push(p);
    }

    // Jana laporan untuk setiap bahagian
    let mut laporan: Vec<LaporanBahagian> = ikut_bahagian
        .iter()
        .map(|(&bahagian, senarai)| {
            let jumlah_gaji: f64 = senarai.iter().map(|p| p.gaji).sum();
            let jumlah = senarai.len();
            let purata = jumlah_gaji / jumlah as f64;

            // Tag yang paling popular dalam bahagian ini
            let mut tag_kira: HashMap<&str, usize> = HashMap::new();
            for p in senarai.iter() {
                for t in &p.tag {
                    *tag_kira.entry(t.as_str()).or_insert(0) += 1;
                }
            }
            let mut tag_sorted: Vec<(&str, usize)> = tag_kira.into_iter().collect();
            tag_sorted.sort_by(|a, b| b.1.cmp(&a.1));
            let tag_popular: Vec<String> = tag_sorted.iter()
                .take(3)
                .map(|(t, _)| t.to_string())
                .collect();

            LaporanBahagian {
                bahagian:       bahagian.to_string(),
                jumlah_pekerja: jumlah,
                jumlah_gaji,
                gaji_purata:    purata,
                tag_popular,
            }
        })
        .collect();

    // Sort mengikut jumlah gaji descending
    laporan.sort_by(|a, b| b.jumlah_gaji.partial_cmp(&a.jumlah_gaji).unwrap());
    laporan
}

fn main() {
    let data_raw = vec![
        PekerjaRaw { nama: "Ali Ahmad".into(), bahagian: "ICT".into(),
            gaji: "4500.00".into(), aktif: "true".into(), tag: "rust,python".into() },
        PekerjaRaw { nama: "Siti Hawa".into(), bahagian: "HR".into(),
            gaji: "3800.00".into(), aktif: "true".into(), tag: "excel,word".into() },
        PekerjaRaw { nama: "  ".into(), bahagian: "ICT".into(),
            gaji: "5000.00".into(), aktif: "true".into(), tag: "java".into() }, // nama kosong
        PekerjaRaw { nama: "Amin Razak".into(), bahagian: "ICT".into(),
            gaji: "abc".into(), aktif: "true".into(), tag: "go,rust".into() },  // gaji tidak sah
        PekerjaRaw { nama: "Zara Lina".into(), bahagian: "Kewangan".into(),
            gaji: "4200.00".into(), aktif: "false".into(), tag: "excel".into() }, // tidak aktif
        PekerjaRaw { nama: "Razi Mahir".into(), bahagian: "ICT".into(),
            gaji: "4800.00".into(), aktif: "true".into(), tag: "rust,java,python".into() },
    ];

    let pekerja = proses_baik(&data_raw);
    println!("Pekerja bersih: {}", pekerja.len()); // 3 (skip 3 yang tidak sah)

    let laporan = jana_laporan(&pekerja);

    println!("\n=== Laporan Bahagian ===");
    for l in &laporan {
        println!("\n{}", l.bahagian);
        println!("  Pekerja:    {}", l.jumlah_pekerja);
        println!("  Gaji total: RM{:.2}", l.jumlah_gaji);
        println!("  Gaji purata: RM{:.2}", l.gaji_purata);
        println!("  Tag popular: {:?}", l.tag_popular);
    }
}
```

---

# 📋 Rujukan Pantas — Chaining Cheat Sheet

## Rules Idiomatic Chain

```
✔ Satu method satu baris
✔ Nama chain/variable yang menjelaskan hasil
✔ Tidak lebih ~7 method dalam satu chain
✔ Break bila ada: branching logik, intermediate check, complex logic
✔ Extract closure panjang ke named function
✔ for_each() untuk side effects, BUKAN map()
✔ filter_map() daripada filter().map()

❌ Satu baris yang horizontal scroll
❌ Closure >3 baris dalam chain
❌ Nested chain (chain dalam chain)
❌ .clone() semata-mata untuk escape borrow
❌ .collect() tengah-tengah tanpa sebab
❌ unwrap() tersembunyi dalam chain
```

## Iterator Methods Paling Berguna

```rust
.map(|x| ...)           // transform setiap element
.filter(|x| ...)        // buang yang tidak lulus
.filter_map(|x| ...)    // transform + filter (return Option)
.flat_map(|x| ...)      // transform + flatten
.flatten()              // [[a,b],[c]] → [a,b,c]
.take(n)                // ambil n pertama
.skip(n)                // skip n pertama
.take_while(|x| ...)    // ambil sehingga syarat gagal
.skip_while(|x| ...)    // skip sehingga syarat gagal
.enumerate()            // (0,x), (1,x), ...
.zip(other)             // [(a,b), (c,d), ...]
.chain(other)           // sambung dua iterator
.peekable()             // boleh peek element seterusnya
.fold(init, |acc,x|...) // reduce dengan accumulator
.any(|x| ...)           // ada satu yang lulus?
.all(|x| ...)           // semua lulus?
.find(|x| ...)          // Option<&T> pertama yang lulus
.position(|x| ...)      // Option<usize> kedudukan
.count()                // bilangan element
.sum()                  // jumlah (perlu Sum trait)
.min() / .max()         // minimum / maximum
.inspect(|x| ...)       // debug tanpa consume
.collect::<Vec<_>>()    // kumpul ke collection
.for_each(|x| ...)      // side effect, return ()
```

## Panduan Pilih: Loop vs Iterator

```
Guna ITERATOR bila:
  ✔ Transform + filter + collect
  ✔ Functional pipeline yang jelas
  ✔ Nak guna parallel (rayon)
  ✔ Return value dari setiap element

Guna FOR LOOP bila:
  ✔ Complex mutable state
  ✔ Early break dengan complex condition
  ✔ Side effects sahaja (println, push ke Vec luar)
  ✔ Logik yang sukar diexpress sebagai chain
  ✔ Perlu mut reference ke beberapa item
```

---

*Chain yang baik dibaca seperti prosa.*
*Kalau perlu 10 minit untuk faham satu baris — pecahkan.*
*Cleverness yang tidak boleh di-maintain bukan cleverness.* 🦀
