# 🗺️ HashMap dalam Rust — Beginner to Advanced

> Panduan lengkap HashMap: dari konsep asas hingga pattern lanjutan.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu HashMap?

```
┌─────────────────────────────────────────────────────┐
│                    HashMap                          │
│                                                     │
│   Key          Value                                │
│  ───────      ────────                              │
│  "Ali"    →   85                                    │
│  "Siti"   →   92                                    │
│  "Amin"   →   78                                    │
│                                                     │
│  Cari "Ali" → terus dapat 85, O(1)!                │
└─────────────────────────────────────────────────────┘
```

HashMap adalah koleksi **key-value pairs** — macam kamus, dictionary Python, atau associative array PHP. Carian sangat laju kerana guna hashing algorithm.

| Bahasa | Equivalent |
|--------|-----------|
| PHP | `$arr = ["Ali" => 85]` |
| Python | `d = {"Ali": 85}` |
| JavaScript | `obj = {Ali: 85}` |
| Java | `HashMap<String, Integer>` |
| **Rust** | `HashMap<String, i32>` |

---

## Peta Pembelajaran

```
Bab 1  → Buat & Isi HashMap
Bab 2  → Baca & Cari Data
Bab 3  → Kemaskini & Padam
Bab 4  → Iterate (Loop)
Bab 5  → Entry API — pattern paling power
Bab 6  → HashMap dengan Struct
Bab 7  → Nested HashMap
Bab 8  → Custom Hash & Performance
Bab 9  → Pattern Lanjutan
Bab 10 → Mini Project: Markah Pelajar
```

---

# BAB 1: Buat & Isi HashMap 🏗️

## Import Dulu

```rust
use std::collections::HashMap;  // wajib import — tidak dalam prelude
```

> 💡 Berbeza dengan `Vec` yang automatic ada, `HashMap` perlu `use` dulu.

---

## Cara-Cara Buat HashMap

```rust
use std::collections::HashMap;

fn main() {
    // Cara 1: kosong, letak type annotation
    let mut markah: HashMap<String, u32> = HashMap::new();

    // Cara 2: kosong, biar Rust infer type dari insert pertama
    let mut markah = HashMap::new();
    markah.insert(String::from("Ali"), 85_u32);  // Rust tahu type dari sini

    // Cara 3: dengan kapasiti awal (elak re-allocate kalau tahu saiz)
    let mut markah: HashMap<String, u32> = HashMap::with_capacity(10);

    // Cara 4: dari dua Vec menggunakan zip + collect
    let nama  = vec!["Ali", "Siti", "Amin"];
    let skor  = vec![85u32, 92, 78];
    let markah: HashMap<&str, u32> = nama.into_iter()
        .zip(skor.into_iter())
        .collect();

    println!("{:?}", markah);
}
```

---

## Insert Data

```rust
use std::collections::HashMap;

fn main() {
    let mut db: HashMap<String, String> = HashMap::new();

    // insert(key, value) — kalau key dah ada, VALUE LAMA DIGANTI
    db.insert("nama".to_string(),     "Ali Ahmad".to_string());
    db.insert("jawatan".to_string(),  "Jurutera".to_string());
    db.insert("bahagian".to_string(), "ICT".to_string());

    // Insert semula — ganti nilai lama
    let nilai_lama = db.insert("nama".to_string(), "Ali bin Ahmad".to_string());
    // insert() return Option<V> — nilai lama kalau ada
    println!("Nilai lama: {:?}", nilai_lama); // Some("Ali Ahmad")

    println!("{:?}", db);
}
```

---

## 🧠 Brain Teaser #1

Berapa kali "ICT" akan muncul dalam HashMap ni?

```rust
let mut m: HashMap<&str, &str> = HashMap::new();
m.insert("bahagian", "HR");
m.insert("bahagian", "ICT");
m.insert("bahagian", "IT");
println!("{}", m["bahagian"]);
```

<details>
<summary>👀 Jawapan</summary>

Output: `IT`

HashMap hanya simpan **satu nilai per key**. Setiap `insert` dengan key yang sama **gantikan** nilai sebelumnya. Akhirnya "bahagian" = "IT".
</details>

---

# BAB 2: Baca & Cari Data 🔍

## Akses Nilai

```rust
use std::collections::HashMap;

fn main() {
    let mut markah: HashMap<&str, u32> = HashMap::new();
    markah.insert("Ali",  85);
    markah.insert("Siti", 92);
    markah.insert("Amin", 78);

    // Cara 1: get() — return Option<&V>, SELAMAT
    match markah.get("Ali") {
        Some(m) => println!("Ali: {}", m),
        None    => println!("Ali tiada dalam senarai"),
    }

    // Cara 2: get() + unwrap_or — bagi default kalau tiada
    let skor = markah.get("Zara").copied().unwrap_or(0);
    println!("Zara: {}", skor); // 0

    // Cara 3: [] operator — PANIC kalau key tiada!
    // Guna HANYA kalau 100% pasti key ada
    println!("Siti: {}", markah["Siti"]); // 92
    // println!("{}", markah["Zara"]);     // ← PANIC! thread 'main' panicked

    // Cara 4: if let — ringkas untuk satu kes
    if let Some(m) = markah.get("Amin") {
        println!("Amin lulus dengan markah {}", m);
    }
}
```

---

## Semak Kewujudan Key

```rust
use std::collections::HashMap;

fn main() {
    let mut kamus: HashMap<&str, &str> = HashMap::new();
    kamus.insert("rumah",  "house");
    kamus.insert("kereta", "car");

    // contains_key() — return bool
    if kamus.contains_key("rumah") {
        println!("'rumah' ada dalam kamus");
    }

    // Gabung contains_key + get
    let perkataan = "basikal";
    if kamus.contains_key(perkataan) {
        println!("{} = {}", perkataan, kamus[perkataan]);
    } else {
        println!("'{}' belum ada dalam kamus", perkataan);
    }
}
```

---

## Saiz HashMap

```rust
use std::collections::HashMap;

fn main() {
    let mut m: HashMap<i32, &str> = HashMap::new();
    m.insert(1, "satu");
    m.insert(2, "dua");
    m.insert(3, "tiga");

    println!("Bilangan entry: {}", m.len());    // 3
    println!("Kosong?        {}", m.is_empty()); // false
    println!("Kapasiti:      {}", m.capacity()); // ≥ 3
}
```

---

# BAB 3: Kemaskini & Padam ✏️

## Kemaskini Nilai

```rust
use std::collections::HashMap;

fn main() {
    let mut stok: HashMap<&str, u32> = HashMap::new();
    stok.insert("beras", 100);
    stok.insert("gula",  50);

    // Tukar terus — sama seperti insert(), ganti nilai lama
    stok.insert("beras", 80);
    println!("Beras: {}", stok["beras"]); // 80

    // Kemaskini berdasarkan nilai lama (tambah 20 unit)
    if let Some(kuantiti) = stok.get_mut("gula") {
        *kuantiti += 20;  // dereference untuk ubah nilai
    }
    println!("Gula: {}", stok["gula"]); // 70
}
```

---

## Padam Entry

```rust
use std::collections::HashMap;

fn main() {
    let mut senarai: HashMap<&str, i32> = HashMap::new();
    senarai.insert("A", 1);
    senarai.insert("B", 2);
    senarai.insert("C", 3);

    // remove() — return Option<V>
    let nilai_dipadam = senarai.remove("B");
    println!("Dipadam: {:?}", nilai_dipadam); // Some(2)

    let tiada = senarai.remove("Z");
    println!("Tiada:   {:?}", tiada); // None

    // Padam semua
    senarai.clear();
    println!("Selepas clear: {}", senarai.len()); // 0

    // retain() — padam berdasarkan syarat
    let mut nombor: HashMap<&str, i32> = [
        ("a", 10), ("b", 3), ("c", 7), ("d", 1)
    ].into_iter().collect();

    // Kekalkan hanya nilai >= 5
    nombor.retain(|_k, v| *v >= 5);
    println!("Selepas retain: {:?}", nombor); // {"a": 10, "c": 7}
}
```

---

## 🧠 Brain Teaser #2

Apa output kod ni?

```rust
use std::collections::HashMap;

fn main() {
    let mut m: HashMap<&str, i32> = HashMap::new();
    m.insert("x", 10);

    if let Some(v) = m.get_mut("x") {
        *v *= 3;
    }

    m.insert("x", m["x"] + 5);
    println!("{}", m["x"]);
}
```

<details>
<summary>👀 Jawapan</summary>

Output: `35`

1. `insert("x", 10)` → x = 10
2. `get_mut` + `*v *= 3` → x = 30
3. `m["x"] + 5` = 35, then `insert("x", 35)` → x = 35
</details>

---

# BAB 4: Iterate (Loop) 🔄

## Semua Cara Iterate

```rust
use std::collections::HashMap;

fn main() {
    let mut pekerja: HashMap<&str, &str> = HashMap::new();
    pekerja.insert("Ali",  "ICT");
    pekerja.insert("Siti", "Kewangan");
    pekerja.insert("Amin", "Pengurusan");

    // Cara 1: iterate key-value (borrow)
    println!("--- Cara 1: &key, &value ---");
    for (nama, bahagian) in &pekerja {
        println!("  {} → {}", nama, bahagian);
    }

    // Cara 2: keys sahaja
    println!("--- Cara 2: Keys ---");
    for nama in pekerja.keys() {
        println!("  {}", nama);
    }

    // Cara 3: values sahaja
    println!("--- Cara 3: Values ---");
    for bahagian in pekerja.values() {
        println!("  {}", bahagian);
    }

    // Cara 4: mutable values (untuk ubah semasa iterate)
    let mut markah: HashMap<&str, u32> = [
        ("Ali", 70), ("Siti", 85), ("Amin", 60)
    ].into_iter().collect();

    // Tambah 5 markah bonus untuk semua
    for (_nama, skor) in markah.iter_mut() {
        *skor += 5;
    }
    println!("--- Cara 4: Selepas bonus ---");
    for (nama, skor) in &markah {
        println!("  {}: {}", nama, skor);
    }

    // Cara 5: consume (into_iter) — HashMap tidak boleh guna selepas ni
    let data: HashMap<&str, i32> = [("a", 1), ("b", 2)].into_iter().collect();
    for (k, v) in data {  // data dipindah (moved) ke sini
        println!("  {} = {}", k, v);
    }
    // println!("{:?}", data); // ← ERROR! data sudah moved
}
```

---

## Sort Sebelum Print

> HashMap **tidak ada order** — urutan iterate tidak dijamin. Nak sorted? Convert ke Vec dulu.

```rust
use std::collections::HashMap;

fn main() {
    let markah: HashMap<&str, u32> = [
        ("Ali", 85), ("Zara", 91), ("Amin", 78), ("Siti", 92)
    ].into_iter().collect();

    // Sort by key (nama)
    let mut sorted: Vec<(&&str, &u32)> = markah.iter().collect();
    sorted.sort_by_key(|&(k, _)| *k);

    println!("--- Sorted by nama ---");
    for (nama, skor) in &sorted {
        println!("  {}: {}", nama, skor);
    }

    // Sort by value (markah) — descending
    sorted.sort_by(|a, b| b.1.cmp(a.1));
    println!("--- Sorted by markah (tinggi → rendah) ---");
    for (nama, skor) in &sorted {
        println!("  {}: {}", nama, skor);
    }
}
```

---

# BAB 5: Entry API — Pattern Paling Power ⚡

> Entry API adalah ciri **unik dan idiomatik** Rust untuk HashMap.
> Ia selesaikan masalah "insert kalau tiada, update kalau ada" dengan elegan.

## or_insert — Insert Kalau Key Belum Ada

```rust
use std::collections::HashMap;

fn main() {
    let mut kiraan: HashMap<&str, u32> = HashMap::new();

    let teks = "ali siti ali amin siti ali";

    // Kira kekerapan perkataan
    for perkataan in teks.split_whitespace() {
        // entry(key) → Entry enum
        // or_insert(0) → kalau tiada, masukkan 0, return &mut u32
        let count = kiraan.entry(perkataan).or_insert(0);
        *count += 1;  // tambah kiraan
    }

    println!("{:?}", kiraan);
    // {"ali": 3, "siti": 2, "amin": 1}
}
```

> 💡 Pattern `entry().or_insert()` adalah **idiom standard** untuk word count, frequency map, dll.

---

## or_insert_with — Lazy Initialization

```rust
use std::collections::HashMap;

fn main() {
    let mut kumpulan: HashMap<&str, Vec<&str>> = HashMap::new();

    let data = vec![
        ("ICT",       "Ali"),
        ("Kewangan",  "Siti"),
        ("ICT",       "Amin"),
        ("Kewangan",  "Zara"),
        ("ICT",       "Razi"),
    ];

    for (bahagian, nama) in data {
        // or_insert_with(Vec::new) — hanya buat Vec baru kalau belum ada
        // lebih efficient dari or_insert(Vec::new()) yang SELALU buat Vec
        kumpulan
            .entry(bahagian)
            .or_insert_with(Vec::new)
            .push(nama);
    }

    for (bahagian, ahli) in &kumpulan {
        println!("{}: {:?}", bahagian, ahli);
    }
    // ICT:      ["Ali", "Amin", "Razi"]
    // Kewangan: ["Siti", "Zara"]
}
```

---

## or_default — Guna Default Value

```rust
use std::collections::HashMap;

fn main() {
    let mut skor: HashMap<&str, i32> = HashMap::new();

    // or_default() guna Default trait — i32::default() = 0
    *skor.entry("Ali").or_default()  += 10;
    *skor.entry("Ali").or_default()  += 5;
    *skor.entry("Siti").or_default() += 20;

    println!("{:?}", skor); // {"Ali": 15, "Siti": 20}

    // Contoh dengan Vec — Vec::default() = Vec::new()
    let mut log: HashMap<&str, Vec<String>> = HashMap::new();
    log.entry("error").or_default().push("Fail connect".to_string());
    log.entry("error").or_default().push("Timeout".to_string());
    log.entry("info").or_default().push("Server start".to_string());

    println!("{:?}", log);
}
```

---

## and_modify — Ubah Kalau Dah Ada

```rust
use std::collections::HashMap;

fn main() {
    let mut inventori: HashMap<&str, u32> = HashMap::new();
    inventori.insert("beras", 100);

    // and_modify — jalankan closure HANYA kalau key dah ada
    // or_insert — masukkan nilai default kalau tiada
    inventori
        .entry("beras")
        .and_modify(|v| *v += 50)  // beras dah ada → tambah 50
        .or_insert(0);

    inventori
        .entry("gula")
        .and_modify(|v| *v += 50)  // gula tiada → skip and_modify
        .or_insert(30);             // masukkan 30

    println!("Beras: {}", inventori["beras"]); // 150
    println!("Gula:  {}", inventori["gula"]);  // 30
}
```

---

## 🧠 Brain Teaser #3

Apakah output program ini?

```rust
use std::collections::HashMap;

fn main() {
    let ayat = "the quick brown fox jumps over the lazy dog the fox";
    let mut kira: HashMap<&str, u32> = HashMap::new();

    for w in ayat.split_whitespace() {
        *kira.entry(w).or_insert(0) += 1;
    }

    println!("the: {}", kira["the"]);
    println!("fox: {}", kira["fox"]);
    println!("cat: {}", kira.get("cat").copied().unwrap_or(0));
}
```

<details>
<summary>👀 Jawapan</summary>

```
the: 3
fox: 2
cat: 0
```

"the" muncul 3 kali, "fox" 2 kali, "cat" langsung tiada (return default 0).
</details>

---

# BAB 6: HashMap dengan Struct 🏗️

## Value Adalah Struct

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct Pekerja {
    nama:     String,
    bahagian: String,
    gaji:     f64,
}

impl Pekerja {
    fn baru(nama: &str, bahagian: &str, gaji: f64) -> Self {
        Pekerja {
            nama:     nama.to_string(),
            bahagian: bahagian.to_string(),
            gaji,
        }
    }
}

fn main() {
    let mut direktori: HashMap<u32, Pekerja> = HashMap::new();

    direktori.insert(1001, Pekerja::baru("Ali Ahmad",  "ICT",       4500.0));
    direktori.insert(1002, Pekerja::baru("Siti Hawa",  "Kewangan",  3800.0));
    direktori.insert(1003, Pekerja::baru("Amin Razak", "Pengurusan",5200.0));

    // Cari pekerja
    if let Some(p) = direktori.get(&1002) {
        println!("Jumpa: {} dari {}", p.nama, p.bahagian);
    }

    // Kemaskini gaji
    if let Some(p) = direktori.get_mut(&1001) {
        p.gaji *= 1.10;  // naik 10%
        println!("{} gaji baru: RM{:.2}", p.nama, p.gaji);
    }

    // Filter — cari semua pekerja gaji > 4000
    println!("\nPekerja gaji > RM4000:");
    let mut hasil: Vec<&Pekerja> = direktori.values()
        .filter(|p| p.gaji > 4000.0)
        .collect();
    hasil.sort_by(|a, b| a.nama.cmp(&b.nama));
    for p in hasil {
        println!("  {} — RM{:.2}", p.nama, p.gaji);
    }
}
```

---

## Key Adalah Struct (Custom Key)

```rust
use std::collections::HashMap;

// Key mesti implement Hash + Eq
#[derive(Debug, Hash, PartialEq, Eq, Clone)]
struct KodJawatan {
    bahagian: String,
    gred:     u8,
}

fn main() {
    let mut bilangan: HashMap<KodJawatan, u32> = HashMap::new();

    let ict_41 = KodJawatan { bahagian: "ICT".into(), gred: 41 };
    let ict_44 = KodJawatan { bahagian: "ICT".into(), gred: 44 };
    let kew_41 = KodJawatan { bahagian: "Kewangan".into(), gred: 41 };

    bilangan.insert(ict_41.clone(), 5);
    bilangan.insert(ict_44.clone(), 2);
    bilangan.insert(kew_41.clone(), 3);

    println!("ICT Gred 41: {} orang", bilangan[&ict_41]);
    println!("ICT Gred 44: {} orang", bilangan[&ict_44]);
}
```

> ⚠️ **Peraturan Custom Key:** struct mestilah implement `Hash`, `PartialEq`, dan `Eq`.
> Boleh guna `#[derive(Hash, PartialEq, Eq)]` kalau semua field pun implement ketiga-tiga trait ni.

---

# BAB 7: Nested HashMap 🪆

## HashMap dalam HashMap

```rust
use std::collections::HashMap;

fn main() {
    // Struktur: Negeri → Bandar → Penduduk
    let mut populasi: HashMap<&str, HashMap<&str, u32>> = HashMap::new();

    // Tambah data Selangor
    let mut selangor = HashMap::new();
    selangor.insert("Shah Alam",    700_000);
    selangor.insert("Petaling Jaya",650_000);
    selangor.insert("Klang",        940_000);
    populasi.insert("Selangor", selangor);

    // Tambah data Johor
    let mut johor = HashMap::new();
    johor.insert("Johor Bahru", 800_000);
    johor.insert("Batu Pahat",  250_000);
    populasi.insert("Johor", johor);

    // Baca nested value
    if let Some(negeri) = populasi.get("Selangor") {
        if let Some(penduduk) = negeri.get("Shah Alam") {
            println!("Shah Alam: {} penduduk", penduduk);
        }
    }

    // Cara lebih ringkas dengan and_then
    let p = populasi
        .get("Johor")
        .and_then(|n| n.get("Johor Bahru"));
    println!("Johor Bahru: {:?}", p);

    // Entry API untuk nested insert
    populasi
        .entry("Kedah")
        .or_insert_with(HashMap::new)
        .insert("Alor Setar", 320_000);

    // Iterate nested
    for (negeri, bandar_map) in &populasi {
        let jumlah: u32 = bandar_map.values().sum();
        println!("{}: jumlah {} penduduk", negeri, jumlah);
    }
}
```

---

## 🧠 Brain Teaser #4

Bagaimana nak tambah nilai "Sungai Petani" ke dalam nested HashMap `populasi` di atas secara selamat, tanpa panic?

<details>
<summary>👀 Jawapan</summary>

```rust
populasi
    .entry("Kedah")
    .or_insert_with(HashMap::new)
    .insert("Sungai Petani", 280_000);
```

`entry().or_insert_with(HashMap::new)` pastikan inner HashMap wujud dulu sebelum `.insert()`. Kalau guna `populasi["Kedah"]["Sungai Petani"] = ...` terus, ia akan panic kalau "Kedah" belum ada.
</details>

---

# BAB 8: Custom Hash & Performance ⚙️

## Default Hasher vs Custom Hasher

```rust
use std::collections::HashMap;

fn main() {
    // Default: SipHash — selamat (tahan DoS attack), tapi bukan paling laju

    // Untuk performance kritikal (e.g. game loop, bukan web server),
    // boleh guna hasher lain seperti FxHashMap dari crate rustc-hash
    // atau AHashMap dari crate ahash

    let mut m: HashMap<u64, &str> = HashMap::new();

    // Benchmark tip: guna with_capacity kalau tahu bilangan entry
    let mut laju: HashMap<u64, &str> = HashMap::with_capacity(1000);

    for i in 0..1000_u64 {
        laju.insert(i, "data");
    }

    println!("Saiz: {}", laju.len());
    println!("Kapasiti: {}", laju.capacity());
}
```

---

## Cargo.toml untuk HashMap Laju

```toml
[dependencies]
# AHashMap — ~2x laju dari default, selamat untuk production
ahash = "0.8"

# FxHashMap — paling laju untuk integer keys, digunakan dalam rustc sendiri
rustc-hash = "1.1"
```

```rust
// Guna AHashMap
use ahash::AHashMap;

fn main() {
    let mut m: AHashMap<&str, i32> = AHashMap::new();
    m.insert("cepat", 100);
    println!("{:?}", m.get("cepat"));
}
```

---

## Elak Clone Yang Tidak Perlu

```rust
use std::collections::HashMap;

fn main() {
    // ❌ Kurang efisien — clone String untuk setiap lookup
    let mut m: HashMap<String, i32> = HashMap::new();
    m.insert("hello".to_string(), 1);
    let k = String::from("hello");
    m.get(&k);         // OK tapi k dah owned

    // ✅ Lebih efisien — guna &str untuk lookup pada HashMap<String, _>
    // HashMap<String, V> boleh dicari dengan &str terus!
    let cari: &str = "hello";
    m.get(cari);       // works! &str → &String lookup via Borrow trait

    // ✅ Kalau key tak perlu owned, guna &str terus
    let mut m2: HashMap<&str, i32> = HashMap::new();
    m2.insert("hello", 1);
    m2.insert("world", 2);
}
```

---

# BAB 9: Pattern Lanjutan 🎯

## Invert HashMap (Key ↔ Value)

```rust
use std::collections::HashMap;

fn main() {
    let kod_ke_nama: HashMap<u32, &str> = [
        (1001, "Ali"),
        (1002, "Siti"),
        (1003, "Amin"),
    ].into_iter().collect();

    // Invert: nama → kod
    let nama_ke_kod: HashMap<&str, u32> = kod_ke_nama
        .iter()
        .map(|(&k, &v)| (v, k))
        .collect();

    println!("Ali punya kod: {}", nama_ke_kod["Ali"]);  // 1001
    println!("Nama bagi 1002: {}", kod_ke_nama[&1002]); // Siti
}
```

---

## Merge Dua HashMap

```rust
use std::collections::HashMap;

fn main() {
    let mut map_a: HashMap<&str, i32> = [("a", 1), ("b", 2)].into_iter().collect();
    let     map_b: HashMap<&str, i32> = [("b", 99), ("c", 3)].into_iter().collect();

    // Merge — nilai map_b override map_a kalau key sama
    for (k, v) in &map_b {
        map_a.insert(k, *v);
    }
    println!("Selepas merge: {:?}", map_a);
    // {"a": 1, "b": 99, "c": 3}  — "b" dari map_b menang

    // Merge — JANGAN override kalau key dah ada
    let mut base: HashMap<&str, i32> = [("a", 1), ("b", 2)].into_iter().collect();
    let extra: HashMap<&str, i32>    = [("b", 99), ("c", 3)].into_iter().collect();

    for (k, v) in extra {
        base.entry(k).or_insert(v);  // hanya masuk kalau belum ada
    }
    println!("Selepas merge (no override): {:?}", base);
    // {"a": 1, "b": 2, "c": 3}  — "b" kekal dari base
}
```

---

## Grouping / Bucketing

```rust
use std::collections::HashMap;

#[derive(Debug)]
struct Pelajar {
    nama:   &'static str,
    kelas:  &'static str,
    markah: u32,
}

fn main() {
    let pelajar = vec![
        Pelajar { nama: "Ali",   kelas: "4A", markah: 85 },
        Pelajar { nama: "Siti",  kelas: "4B", markah: 92 },
        Pelajar { nama: "Amin",  kelas: "4A", markah: 78 },
        Pelajar { nama: "Zara",  kelas: "4B", markah: 88 },
        Pelajar { nama: "Razi",  kelas: "4A", markah: 95 },
    ];

    // Group by kelas
    let mut mengikut_kelas: HashMap<&str, Vec<&Pelajar>> = HashMap::new();
    for p in &pelajar {
        mengikut_kelas
            .entry(p.kelas)
            .or_insert_with(Vec::new)
            .push(p);
    }

    // Purata markah per kelas
    for (kelas, ahli) in &mengikut_kelas {
        let jumlah: u32 = ahli.iter().map(|p| p.markah).sum();
        let purata = jumlah as f64 / ahli.len() as f64;
        println!("Kelas {}: purata {:.1}", kelas, purata);
    }
}
```

---

## Frequency Counter (Histogram)

```rust
use std::collections::HashMap;

fn histogram(data: &[i32]) -> HashMap<i32, usize> {
    let mut freq = HashMap::new();
    for &x in data {
        *freq.entry(x).or_insert(0) += 1;
    }
    freq
}

fn main() {
    let dadu = vec![1,2,3,2,1,4,5,2,3,1,1,6,2,4];
    let freq  = histogram(&dadu);

    // Print sorted
    let mut sorted: Vec<_> = freq.iter().collect();
    sorted.sort_by_key(|(&k, _)| k);

    println!("Keputusan balingan dadu:");
    for (nilai, kali) in sorted {
        let bar: String = "█".repeat(*kali);
        println!("  {} | {} ({}x)", nilai, bar, kali);
    }
}
```

---

## Cache / Memoization

```rust
use std::collections::HashMap;

// Fibonacci dengan memoization guna HashMap sebagai cache
fn fib(n: u64, cache: &mut HashMap<u64, u64>) -> u64 {
    if n <= 1 {
        return n;
    }

    // Semak cache dulu
    if let Some(&hasil) = cache.get(&n) {
        return hasil;  // dah kira sebelum ni, return terus
    }

    // Belum ada — kira dan simpan dalam cache
    let hasil = fib(n - 1, cache) + fib(n - 2, cache);
    cache.insert(n, hasil);
    hasil
}

fn main() {
    let mut cache: HashMap<u64, u64> = HashMap::new();

    for i in [10, 20, 30, 40, 50] {
        let hasil = fib(i, &mut cache);
        println!("fib({}) = {}", i, hasil);
    }

    println!("Cache size: {} entries", cache.len());
}
```

---

## 🧠 Brain Teaser #5 (Advanced)

Tulis fungsi yang terima `Vec<&str>` dan return HashMap di mana:
- Key = panjang perkataan
- Value = Vec perkataan dengan panjang tersebut

Contoh input: `["cat", "dog", "rust", "go", "fn", "map"]`
Expected output: `{2: ["go", "fn"], 3: ["cat", "dog", "map"], 4: ["rust"]}`

<details>
<summary>👀 Jawapan</summary>

```rust
use std::collections::HashMap;

fn kumpul_ikut_panjang<'a>(perkataan: &[&'a str]) -> HashMap<usize, Vec<&'a str>> {
    let mut hasil: HashMap<usize, Vec<&str>> = HashMap::new();
    for &p in perkataan {
        hasil.entry(p.len()).or_insert_with(Vec::new).push(p);
    }
    hasil
}

fn main() {
    let input = vec!["cat", "dog", "rust", "go", "fn", "map"];
    let kumpul = kumpul_ikut_panjang(&input);
    let mut sorted: Vec<_> = kumpul.iter().collect();
    sorted.sort_by_key(|(&k, _)| k);
    for (panjang, senarai) in sorted {
        println!("{}: {:?}", panjang, senarai);
    }
}
```
</details>

---

# BAB 10: Mini Project — Sistem Markah Pelajar 📊

## Bina sistem lengkap guna HashMap!

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct Rekod {
    markah:   Vec<u32>,
    kehadiran: u32,  // hari hadir
}

impl Rekod {
    fn baru() -> Self {
        Rekod { markah: Vec::new(), kehadiran: 0 }
    }

    fn purata(&self) -> f64 {
        if self.markah.is_empty() {
            return 0.0;
        }
        self.markah.iter().sum::<u32>() as f64 / self.markah.len() as f64
    }

    fn gred(&self) -> &str {
        match self.purata() as u32 {
            90..=100 => "A+",
            80..=89  => "A",
            70..=79  => "B",
            60..=69  => "C",
            50..=59  => "D",
            _        => "Gagal",
        }
    }
}

struct SistemMarkah {
    data: HashMap<String, Rekod>,
}

impl SistemMarkah {
    fn baru() -> Self {
        SistemMarkah { data: HashMap::new() }
    }

    fn daftar(&mut self, nama: &str) {
        self.data.entry(nama.to_string()).or_insert_with(Rekod::baru);
        println!("✔ {} didaftarkan", nama);
    }

    fn tambah_markah(&mut self, nama: &str, markah: u32) {
        match self.data.get_mut(nama) {
            Some(rekod) => {
                rekod.markah.push(markah);
                println!("✔ Markah {} ditambah untuk {}", markah, nama);
            }
            None => println!("✘ '{}' tidak dijumpai. Daftar dulu!", nama),
        }
    }

    fn rekod_hadir(&mut self, nama: &str) {
        if let Some(rekod) = self.data.get_mut(nama) {
            rekod.kehadiran += 1;
        }
    }

    fn laporan(&self) {
        println!("\n{}", "═".repeat(55));
        println!("{:^55}", "LAPORAN AKHIR PELAJAR");
        println!("{}", "═".repeat(55));
        println!("{:<15} {:>8} {:>6} {:>10} {:>8}",
            "Nama", "Purata", "Gred", "Kehadiran", "Markah");
        println!("{}", "─".repeat(55));

        let mut senarai: Vec<(&String, &Rekod)> = self.data.iter().collect();
        senarai.sort_by(|a, b| {
            b.1.purata().partial_cmp(&a.1.purata()).unwrap()
        });

        for (nama, rekod) in &senarai {
            println!("{:<15} {:>8.1} {:>6} {:>10} {:>8?}",
                nama, rekod.purata(), rekod.gred(),
                rekod.kehadiran, rekod.markah);
        }

        println!("{}", "═".repeat(55));

        // Statistik kelas
        let semua_purata: Vec<f64> = self.data.values()
            .map(|r| r.purata())
            .collect();

        let jumlah: f64 = semua_purata.iter().sum();
        let purata_kelas = jumlah / semua_purata.len() as f64;
        let tertinggi = semua_purata.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
        let terendah  = semua_purata.iter().cloned().fold(f64::INFINITY, f64::min);

        println!("Purata Kelas : {:.1}", purata_kelas);
        println!("Markah Tinggi: {:.1}", tertinggi);
        println!("Markah Rendah: {:.1}", terendah);

        // Taburan gred
        let mut taburan: HashMap<&str, usize> = HashMap::new();
        for rekod in self.data.values() {
            *taburan.entry(rekod.gred()).or_insert(0) += 1;
        }
        println!("\nTaburan Gred: {:?}", taburan);
    }
}

fn main() {
    let mut sistem = SistemMarkah::baru();

    // Daftar pelajar
    for nama in &["Ali Ahmad", "Siti Hawa", "Amin Razak", "Zara Lina", "Razi Malik"] {
        sistem.daftar(nama);
    }

    // Tambah markah
    sistem.tambah_markah("Ali Ahmad",   85);
    sistem.tambah_markah("Ali Ahmad",   90);
    sistem.tambah_markah("Ali Ahmad",   88);
    sistem.tambah_markah("Siti Hawa",   95);
    sistem.tambah_markah("Siti Hawa",   92);
    sistem.tambah_markah("Amin Razak",  70);
    sistem.tambah_markah("Amin Razak",  65);
    sistem.tambah_markah("Zara Lina",   78);
    sistem.tambah_markah("Zara Lina",   82);
    sistem.tambah_markah("Razi Malik",  55);
    sistem.tambah_markah("Razi Malik",  60);
    sistem.tambah_markah("TIADA",       50); // test nama tak wujud

    // Rekod kehadiran
    for nama in &["Ali Ahmad", "Siti Hawa", "Amin Razak"] {
        for _ in 0..20 { sistem.rekod_hadir(nama); }
    }
    for _ in 0..18 { sistem.rekod_hadir("Zara Lina"); }
    for _ in 0..15 { sistem.rekod_hadir("Razi Malik"); }

    // Cetak laporan
    sistem.laporan();
}
```

**Output contoh:**
```
═══════════════════════════════════════════════════════
              LAPORAN AKHIR PELAJAR
═══════════════════════════════════════════════════════
Nama             Purata   Gred  Kehadiran   Markah
───────────────────────────────────────────────────────
Siti Hawa          93.5     A+         20  [95, 92]
Ali Ahmad          87.7      A         20  [85, 90, 88]
Zara Lina          80.0      A         18  [78, 82]
Amin Razak         67.5      C         20  [70, 65]
Razi Malik         57.5      D         15  [55, 60]
═══════════════════════════════════════════════════════
Purata Kelas : 77.2
Markah Tinggi: 93.5
Markah Rendah: 57.5

Taburan Gred: {"A+": 1, "A": 2, "C": 1, "D": 1}
```

---

# 📋 Rujukan Pantas — HashMap Cheat Sheet

## Methods Paling Penting

| Method | Kegunaan | Return |
|--------|----------|--------|
| `insert(k, v)` | Masukkan / ganti | `Option<V>` (nilai lama) |
| `get(&k)` | Baca nilai | `Option<&V>` |
| `get_mut(&k)` | Baca nilai (mutable) | `Option<&mut V>` |
| `contains_key(&k)` | Semak kewujudan | `bool` |
| `remove(&k)` | Padam | `Option<V>` |
| `entry(k)` | Entry API | `Entry<K, V>` |
| `len()` | Bilangan entry | `usize` |
| `is_empty()` | Semak kosong | `bool` |
| `clear()` | Padam semua | `()` |
| `keys()` | Iterator keys | `Keys<K, V>` |
| `values()` | Iterator values | `Values<K, V>` |
| `iter()` | Iterator `(&K, &V)` | `Iter<K, V>` |
| `iter_mut()` | Iterator `(&K, &mut V)` | `IterMut<K, V>` |
| `retain(\|k,v\| ...)` | Padam mengikut syarat | `()` |

## Entry API

| Method | Kegunaan |
|--------|----------|
| `.or_insert(v)` | Insert `v` kalau key tiada |
| `.or_insert_with(\|\| v)` | Insert dari closure kalau key tiada (lazy) |
| `.or_default()` | Insert `Default::default()` kalau key tiada |
| `.and_modify(\|v\| ...)` | Modify kalau key dah ada |

## Buat HashMap Dari Data

```rust
// Dari array of tuples
let m: HashMap<&str, i32> = [("a", 1), ("b", 2)].into_iter().collect();

// Dari dua Vec
let m: HashMap<_, _> = keys.into_iter().zip(vals.into_iter()).collect();

// Kosong dengan kapasiti
let m: HashMap<K, V> = HashMap::with_capacity(n);
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **Kamus Dwibahasa** — input perkataan Melayu, dapat English (guna HashMap)
2. **Skor Game** — leaderboard dengan top 5 pemain (HashMap + sort)
3. **Inventory System** — tambah/tolak stok, alert kalau bawah minimum
4. **Analisis Teks** — baca string, kira kekerapan setiap huruf

---

*HashMap in Rust — dari `HashMap::new()` hingga pattern lanjutan. Selamat belajar!* 🦀
