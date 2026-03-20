# 🔵 HashSet dalam Rust — Beginner to Advanced

> Panduan lengkap HashSet<T>: dari konsep asas hingga set operations lanjutan.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu HashSet?

```
┌─────────────────────────────────────────────────────┐
│                    HashSet                          │
│                                                     │
│   { "Ali", "Siti", "Amin", "Zara" }                │
│                                                     │
│   ✔ Setiap nilai UNIK — tiada duplikat              │
│   ✔ Tiada index — bukan ordered                     │
│   ✔ Carian O(1) — sangat laju                       │
│   ✔ Sokong operasi set: union, intersection, dll.   │
└─────────────────────────────────────────────────────┘
```

HashSet adalah koleksi nilai **unik** — macam HashMap tapi **tanpa value**, key sahaja. Setiap elemen dijamin wujud sekali sahaja.

| Bahasa | Equivalent |
|--------|-----------|
| PHP | `array_unique($arr)` / SplFixedArray |
| Python | `set({1, 2, 3})` |
| JavaScript | `new Set([1, 2, 3])` |
| Java | `HashSet<String>` |
| **Rust** | `HashSet<String>` |

---

## Vec vs HashSet — Bila Guna Yang Mana?

```
┌─────────────────┬──────────────────────────────────┐
│   Guna Vec      │   Guna HashSet                   │
├─────────────────┼──────────────────────────────────┤
│ Urutan penting  │ Urutan tidak penting              │
│ Boleh duplikat  │ Mahu nilai unik sahaja            │
│ Akses by index  │ Cek kewujudan dengan cepat        │
│ Stack/Queue     │ Set operations (union, intersect) │
│ [1, 1, 2, 3]   │ {1, 2, 3}                         │
└─────────────────┴──────────────────────────────────┘
```

---

## Peta Pembelajaran

```
Bab 1  → Buat & Isi HashSet
Bab 2  → Semak & Cari
Bab 3  → Buang Elemen
Bab 4  → Iterate
Bab 5  → Set Operations — Union, Intersection, Difference
Bab 6  → HashSet dengan Struct
Bab 7  → HashSet vs BTreeSet
Bab 8  → Pattern Lanjutan
Bab 9  → Performance & Custom Hash
Bab 10 → Mini Project: Sistem Kehadiran
```

---

# BAB 1: Buat & Isi HashSet 🏗️

## Import Dulu

```rust
use std::collections::HashSet;  // wajib import
```

---

## Cara-Cara Buat HashSet

```rust
use std::collections::HashSet;

fn main() {
    // Cara 1: kosong, nyatakan type
    let mut s: HashSet<i32> = HashSet::new();

    // Cara 2: dari array guna into_iter + collect
    let s: HashSet<i32> = [1, 2, 3, 4, 5].into_iter().collect();
    println!("{:?}", s);

    // Cara 3: dari Vec guna collect
    let v = vec!["Ali", "Siti", "Amin", "Ali"]; // ada duplikat!
    let s: HashSet<&str> = v.into_iter().collect();
    println!("{:?}", s); // {"Ali", "Siti", "Amin"} — duplikat hilang!

    // Cara 4: dengan kapasiti awal
    let mut s: HashSet<String> = HashSet::with_capacity(10);

    // Cara 5: guna macro (tiada built-in, tapi boleh simulate)
    let s: HashSet<i32> = vec![1, 2, 3, 4, 5].into_iter().collect();
}
```

---

## Insert & Keunikan

```rust
use std::collections::HashSet;

fn main() {
    let mut tag: HashSet<&str> = HashSet::new();

    // insert() return bool — true kalau berjaya, false kalau dah ada
    println!("{}", tag.insert("rust"));    // true  — baru
    println!("{}", tag.insert("coding"));  // true  — baru
    println!("{}", tag.insert("rust"));    // false — dah ada!
    println!("{}", tag.insert("web"));     // true  — baru

    println!("Tags: {:?}", tag);
    println!("Bilangan: {}", tag.len()); // 3, bukan 4!

    // Insert banyak sekaligus
    let tambahan = ["linux", "coding", "open-source"]; // "coding" dah ada
    for t in &tambahan {
        let baru = tag.insert(t);
        if baru {
            println!("Ditambah: {}", t);
        } else {
            println!("Dah ada: {}", t);
        }
    }
}
```

---

## 🧠 Brain Teaser #1

Berapa elemen dalam HashSet ni?

```rust
use std::collections::HashSet;

fn main() {
    let data = vec![1, 2, 2, 3, 3, 3, 4, 4, 4, 4];
    let s: HashSet<i32> = data.into_iter().collect();
    println!("{}", s.len());
}
```

<details>
<summary>👀 Jawapan</summary>

Output: `4`

HashSet hanya simpan nilai **unik**: {1, 2, 3, 4}. Walaupun 4 muncul 4 kali dalam Vec, ia hanya disimpan sekali dalam HashSet.
</details>

---

# BAB 2: Semak & Cari 🔍

## contains — Cara Paling Penting!

```rust
use std::collections::HashSet;

fn main() {
    let akses: HashSet<&str> = ["admin", "editor", "viewer"].into_iter().collect();

    // contains() — O(1), sangat laju!
    if akses.contains("admin") {
        println!("Akses admin dibenarkan ✔");
    }

    if !akses.contains("superuser") {
        println!("Akses superuser ditolak ✘");
    }

    // Berbeza dengan Vec.contains() yang O(n)!
    let v = vec!["admin", "editor", "viewer"];
    // v.contains(&"admin") — kena scan satu-satu, O(n)
    // s.contains("admin")  — terus jumpa, O(1)

    println!("\nSemak beberapa peranan:");
    for peranan in &["admin", "viewer", "root", "editor"] {
        let ada = if akses.contains(peranan) { "✔" } else { "✘" };
        println!("  {} {}", ada, peranan);
    }
}
```

---

## get — Dapat Reference Kepada Elemen

```rust
use std::collections::HashSet;

fn main() {
    let nama: HashSet<String> = vec![
        "Ali".to_string(),
        "Siti".to_string(),
        "Amin".to_string(),
    ].into_iter().collect();

    // get() — return Option<&T> (reference kepada elemen dalam set)
    // Berguna bila nak dapat owned reference, bukan sekadar semak kewujudan
    match nama.get("Ali") {
        Some(n) => println!("Jumpa: {}", n),
        None    => println!("Tiada"),
    }

    println!("{:?}", nama.get("Zara")); // None
}
```

---

## Saiz & Kosong

```rust
use std::collections::HashSet;

fn main() {
    let mut s: HashSet<i32> = HashSet::new();

    println!("Kosong? {}", s.is_empty()); // true
    println!("Saiz:   {}", s.len());      // 0

    s.insert(1);
    s.insert(2);
    s.insert(3);

    println!("Kosong? {}", s.is_empty()); // false
    println!("Saiz:   {}", s.len());      // 3
    println!("Kapasiti: {}", s.capacity()); // ≥ 3
}
```

---

## 🧠 Brain Teaser #2

Kenapa HashSet lebih sesuai dari Vec untuk semak akses pengguna?

```rust
// Sistem dengan 1,000,000 pengguna aktif
let aktif_vec: Vec<u64>     = /* ... 1 juta ID ... */;
let aktif_set: HashSet<u64> = /* ... 1 juta ID ... */;

// Bila pengguna login, semak sama ada ID mereka aktif
let id_pengguna = 123456_u64;
aktif_vec.contains(&id_pengguna); // ???
aktif_set.contains(&id_pengguna); // ???
```

<details>
<summary>👀 Jawapan</summary>

- `Vec.contains()` — **O(n)**: kena scan satu-satu hingga jumpa atau habis. Dengan 1 juta elemen, boleh scan sehingga 1 juta kali!
- `HashSet.contains()` — **O(1)**: terus hash ID dan pergi ke lokasi memori yang betul. Dengan 1 juta elemen pun, masa hampir sama!

Untuk lookup yang kerap, HashSet jauh lebih laju.
</details>

---

# BAB 3: Buang Elemen 🗑️

```rust
use std::collections::HashSet;

fn main() {
    let mut s: HashSet<i32> = [1, 2, 3, 4, 5].into_iter().collect();

    // remove() — buang elemen, return bool
    let berjaya = s.remove(&3);
    println!("Remove 3: {}", berjaya); // true
    println!("Remove 9: {}", s.remove(&9)); // false — tiada 9

    println!("{:?}", s); // {1, 2, 4, 5}

    // take() — buang DAN return nilai (ambil ownership)
    let diambil = s.take(&4);
    println!("Take 4: {:?}", diambil); // Some(4)
    println!("Take 9: {:?}", s.take(&9)); // None

    println!("{:?}", s); // {1, 2, 5}

    // retain() — kekalkan hanya yang memenuhi syarat
    let mut nombor: HashSet<i32> = (1..=10).collect();
    nombor.retain(|&x| x % 3 == 0); // kekalkan gandaan 3 sahaja
    println!("Gandaan 3: {:?}", nombor); // {3, 6, 9}

    // clear() — kosongkan semua
    nombor.clear();
    println!("Selepas clear: {:?}", nombor); // {}
    println!("Len: {}", nombor.len()); // 0
}
```

---

# BAB 4: Iterate 🔄

```rust
use std::collections::HashSet;

fn main() {
    let buah: HashSet<&str> = ["epal", "mangga", "pisang", "durian"]
        .into_iter()
        .collect();

    // Cara 1: borrow — paling biasa
    println!("--- Iterate biasa ---");
    for b in &buah {
        print!("{} ", b);
    }
    println!();

    // Cara 2: into_iter — consume HashSet
    let nombor: HashSet<i32> = [10, 20, 30].into_iter().collect();
    for n in nombor {  // nombor moved
        print!("{} ", n);
    }
    println!();

    // Cara 3: sorted (HashSet tiada order — sort dulu untuk output konsisten)
    let mut sorted: Vec<&&str> = buah.iter().collect();
    sorted.sort();
    println!("Sorted: {:?}", sorted);

    // Cara 4: dengan filter
    let angka: HashSet<i32> = (1..=20).collect();
    let genap_besar: Vec<i32> = angka.iter()
        .filter(|&&x| x % 2 == 0 && x > 10)
        .copied()
        .collect();
    let mut gb = genap_besar;
    gb.sort();
    println!("Genap > 10: {:?}", gb); // [12, 14, 16, 18, 20]

    // Cara 5: transform dan collect ke HashSet baru
    let huruf_besar: HashSet<String> = buah.iter()
        .map(|s| s.to_uppercase())
        .collect();
    println!("Huruf besar: {:?}", huruf_besar);
}
```

> ⚠️ **Ingat:** HashSet **tidak ada order** — iterate tidak dijamin sama setiap kali. Sort dulu kalau nak output yang konsisten!

---

# BAB 5: Set Operations — Union, Intersection, Difference ⊕

> Ini kelebihan **utama** HashSet berbanding Vec — operasi set matematik!

```
Set A: { 1, 2, 3, 4, 5 }
Set B: { 3, 4, 5, 6, 7 }

Union (A ∪ B):        { 1, 2, 3, 4, 5, 6, 7 }  — semua
Intersection (A ∩ B): { 3, 4, 5 }               — yang ada dalam KEDUA-DUA
Difference (A - B):   { 1, 2 }                  — ada dalam A, TIADA dalam B
Sym. Diff (A △ B):    { 1, 2, 6, 7 }            — ada dalam satu tapi bukan kedua-dua
```

## Semua Set Operations

```rust
use std::collections::HashSet;

fn main() {
    let a: HashSet<i32> = [1, 2, 3, 4, 5].into_iter().collect();
    let b: HashSet<i32> = [3, 4, 5, 6, 7].into_iter().collect();

    // UNION (A ∪ B) — semua elemen dari kedua-dua set
    let union: HashSet<i32> = a.union(&b).copied().collect();
    let mut u: Vec<i32> = union.into_iter().collect(); u.sort();
    println!("Union:        {:?}", u); // [1, 2, 3, 4, 5, 6, 7]

    // INTERSECTION (A ∩ B) — elemen yang ada dalam KEDUA-DUA set
    let inter: HashSet<i32> = a.intersection(&b).copied().collect();
    let mut i: Vec<i32> = inter.into_iter().collect(); i.sort();
    println!("Intersection: {:?}", i); // [3, 4, 5]

    // DIFFERENCE (A - B) — ada dalam A tapi TIADA dalam B
    let diff: HashSet<i32> = a.difference(&b).copied().collect();
    let mut d: Vec<i32> = diff.into_iter().collect(); d.sort();
    println!("A - B:        {:?}", d); // [1, 2]

    // DIFFERENCE (B - A) — ada dalam B tapi TIADA dalam A
    let diff2: HashSet<i32> = b.difference(&a).copied().collect();
    let mut d2: Vec<i32> = diff2.into_iter().collect(); d2.sort();
    println!("B - A:        {:?}", d2); // [6, 7]

    // SYMMETRIC DIFFERENCE (A △ B) — ada dalam satu SAHAJA, bukan kedua-dua
    let sym: HashSet<i32> = a.symmetric_difference(&b).copied().collect();
    let mut s: Vec<i32> = sym.into_iter().collect(); s.sort();
    println!("Sym. Diff:    {:?}", s); // [1, 2, 6, 7]
}
```

---

## Semak Hubungan Antara Set

```rust
use std::collections::HashSet;

fn main() {
    let semua:    HashSet<i32> = (1..=10).collect();
    let genap:    HashSet<i32> = [2, 4, 6, 8, 10].into_iter().collect();
    let kecil:    HashSet<i32> = [1, 2, 3].into_iter().collect();
    let besar:    HashSet<i32> = [100, 200].into_iter().collect();

    // is_subset — adakah semua elemen A ada dalam B?
    println!("genap ⊆ semua?  {}", genap.is_subset(&semua));   // true
    println!("kecil ⊆ genap?  {}", kecil.is_subset(&genap));   // false

    // is_superset — adakah semua elemen B ada dalam A?
    println!("semua ⊇ genap?  {}", semua.is_superset(&genap)); // true

    // is_disjoint — tiada elemen yang sama langsung?
    println!("genap ∩ besar = ∅? {}", genap.is_disjoint(&besar)); // true
    println!("genap ∩ kecil = ∅? {}", genap.is_disjoint(&kecil)); // false (ada 2)

    // Contoh praktikal: semak permission
    let required: HashSet<&str> = ["read", "write"].into_iter().collect();
    let user_perm: HashSet<&str> = ["read", "write", "admin"].into_iter().collect();
    let guest_perm: HashSet<&str> = ["read"].into_iter().collect();

    println!("\nUser boleh akses?  {}", required.is_subset(&user_perm));  // true
    println!("Guest boleh akses? {}", required.is_subset(&guest_perm)); // false
}
```

---

## Contoh Dunia Sebenar: Analisis Kursus Pelajar

```rust
use std::collections::HashSet;

fn main() {
    let ali_kursus:  HashSet<&str> = ["Math", "Physics", "CS", "English"].into_iter().collect();
    let siti_kursus: HashSet<&str> = ["CS", "English", "Biology", "Chemistry"].into_iter().collect();

    // Kursus yang diambil KEDUA-DUA
    let sama: Vec<&&str> = {
        let mut v: Vec<_> = ali_kursus.intersection(&siti_kursus).collect();
        v.sort();
        v
    };
    println!("Kursus sama:         {:?}", sama);

    // Kursus yang Ali ada tapi Siti tiada
    let ali_sahaja: Vec<&&str> = {
        let mut v: Vec<_> = ali_kursus.difference(&siti_kursus).collect();
        v.sort();
        v
    };
    println!("Ali sahaja:          {:?}", ali_sahaja);

    // Gabungan semua kursus
    let semua: Vec<&&str> = {
        let mut v: Vec<_> = ali_kursus.union(&siti_kursus).collect();
        v.sort();
        v
    };
    println!("Semua kursus:        {:?}", semua);

    // Kursus yang hanya diambil SATU orang
    let unik: Vec<&&str> = {
        let mut v: Vec<_> = ali_kursus.symmetric_difference(&siti_kursus).collect();
        v.sort();
        v
    };
    println!("Unik (satu org saja):{:?}", unik);
}
```

---

## 🧠 Brain Teaser #3

Ada 3 set ini:

```rust
let a: HashSet<i32> = [1, 2, 3, 4].into_iter().collect();
let b: HashSet<i32> = [3, 4, 5, 6].into_iter().collect();
let c: HashSet<i32> = [5, 6, 7, 8].into_iter().collect();
```

Tanpa run kod, apakah hasil `a ∩ b`, `b ∩ c`, dan `a ∩ c`?

<details>
<summary>👀 Jawapan</summary>

```
a ∩ b = {3, 4}   — ada dalam kedua-dua a dan b
b ∩ c = {5, 6}   — ada dalam kedua-dua b dan c
a ∩ c = {}       — tiada yang sama! a dan c disjoint
```

Kod:
```rust
let ab: HashSet<_> = a.intersection(&b).collect(); // {3,4}
let bc: HashSet<_> = b.intersection(&c).collect(); // {5,6}
let ac: HashSet<_> = a.intersection(&c).collect(); // {}
println!("a ∩ c disjoint: {}", a.is_disjoint(&c)); // true
```
</details>

---

# BAB 6: HashSet dengan Struct 🏗️

## Struct Dalam HashSet — Mesti Implement Hash + Eq

```rust
use std::collections::HashSet;

// Wajib: Hash, PartialEq, Eq
// Optional tapi useful: Debug, Clone
#[derive(Debug, Hash, PartialEq, Eq, Clone)]
struct PelajarID {
    matrik: String,
    tahun:  u32,
}

impl PelajarID {
    fn baru(matrik: &str, tahun: u32) -> Self {
        PelajarID { matrik: matrik.into(), tahun }
    }
}

fn main() {
    let mut hadir: HashSet<PelajarID> = HashSet::new();

    hadir.insert(PelajarID::baru("A001", 2024));
    hadir.insert(PelajarID::baru("A002", 2024));
    hadir.insert(PelajarID::baru("A001", 2024)); // duplikat — diabaikan!
    hadir.insert(PelajarID::baru("A003", 2023));

    println!("Jumlah hadir unik: {}", hadir.len()); // 3

    let semak = PelajarID::baru("A002", 2024);
    println!("A002 hadir? {}", hadir.contains(&semak)); // true
}
```

---

## Bila Struct Lebih Kompleks — Implement Manual

```rust
use std::collections::HashSet;
use std::hash::{Hash, Hasher};

#[derive(Debug, Clone)]
struct Pekerja {
    id:       u32,
    nama:     String,
    bahagian: String,  // kita TIDAK nak hash field ini
}

// Hash hanya berdasarkan `id` — dua Pekerja dengan id sama = sama
impl Hash for Pekerja {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.id.hash(state);
    }
}

// Eq juga hanya berdasarkan `id`
impl PartialEq for Pekerja {
    fn eq(&self, other: &Self) -> bool {
        self.id == other.id
    }
}

impl Eq for Pekerja {}

fn main() {
    let mut set: HashSet<Pekerja> = HashSet::new();

    set.insert(Pekerja { id: 1, nama: "Ali".into(),   bahagian: "ICT".into() });
    set.insert(Pekerja { id: 2, nama: "Siti".into(),  bahagian: "HR".into() });
    // ID sama = dianggap sama, walaupun nama berbeza!
    set.insert(Pekerja { id: 1, nama: "Ali Ahmad".into(), bahagian: "ICT".into() });

    println!("Jumlah unik: {}", set.len()); // 2, bukan 3!

    for p in &set {
        println!("  ID {}: {}", p.id, p.nama);
    }
}
```

> ⚠️ **Peraturan Penting:** Kalau `a == b`, maka `hash(a) == hash(b)` mesti benar. Kalau implement `Hash` dan `Eq` secara manual, pastikan konsisten!

---

# BAB 7: HashSet vs BTreeSet ⚖️

```
HashSet                     BTreeSet
─────────────────────       ─────────────────────
O(1) lookup                 O(log n) lookup
Tiada order                 SORTED selalu
Guna hashing                Guna B-tree
Lebih laju                  Lebih perlahan
Tiada range queries         Boleh range queries!
```

```rust
use std::collections::{HashSet, BTreeSet};

fn main() {
    let data = vec![5, 3, 8, 1, 9, 2, 7, 4, 6];

    // HashSet — tiada order
    let h: HashSet<i32> = data.iter().copied().collect();
    println!("HashSet: {:?}", h); // rawak: {5, 3, 8, 1, ...}

    // BTreeSet — sentiasa sorted!
    let b: BTreeSet<i32> = data.iter().copied().collect();
    println!("BTreeSet: {:?}", b); // [1, 2, 3, 4, 5, 6, 7, 8, 9]

    // BTreeSet boleh range queries
    let lima_ke_lapan: Vec<&i32> = b.range(5..=8).collect();
    println!("Range 5..=8: {:?}", lima_ke_lapan); // [5, 6, 7, 8]

    // BTreeSet boleh first() dan last()
    println!("Min: {:?}", b.first()); // Some(1)
    println!("Max: {:?}", b.last());  // Some(9)

    // Semua set operations sama untuk kedua-dua!
    let b2: BTreeSet<i32> = [3, 4, 5, 6, 7].into_iter().collect();
    let inter: BTreeSet<_> = b.intersection(&b2).copied().collect();
    println!("Intersection: {:?}", inter); // {3, 4, 5, 6, 7}
}
```

## Panduan Pilih

```
Perlu lookup laju sahaja?          → HashSet
Perlu sorted output?               → BTreeSet
Perlu range query (5..10)?         → BTreeSet
Perlu min/max dengan cepat?        → BTreeSet
Data besar, lookup kritikal?       → HashSet
```

---

# BAB 8: Pattern Lanjutan 🎯

## Buang Duplikat Dari Vec (Kekal Order)

```rust
use std::collections::HashSet;

fn buang_duplikat<T: Eq + std::hash::Hash + Clone>(v: Vec<T>) -> Vec<T> {
    let mut seen = HashSet::new();
    v.into_iter()
        .filter(|x| seen.insert(x.clone()))
        .collect()
}

fn main() {
    let v = vec![3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5];
    println!("Asal:  {:?}", v);
    println!("Unik:  {:?}", buang_duplikat(v));
    // Unik: [3, 1, 4, 5, 9, 2, 6]  — order dikekal!

    let kata = vec!["ali", "siti", "ali", "amin", "siti"];
    println!("Unik kata: {:?}", buang_duplikat(kata));
    // ["ali", "siti", "amin"]
}
```

---

## Cari Duplikat Dalam Vec

```rust
use std::collections::HashSet;

fn cari_duplikat<T: Eq + std::hash::Hash + Clone>(v: &[T]) -> Vec<T> {
    let mut seen    = HashSet::new();
    let mut duplikat = HashSet::new();

    for item in v {
        if !seen.insert(item) {
            duplikat.insert(item.clone());
        }
    }

    duplikat.into_iter().collect()
}

fn main() {
    let v = vec![1, 2, 3, 2, 4, 3, 5, 1];
    let mut dup = cari_duplikat(&v);
    dup.sort();
    println!("Duplikat: {:?}", dup); // [1, 2, 3]

    let nama = vec!["Ali", "Siti", "Ali", "Amin", "Siti"];
    let mut dup_nama = cari_duplikat(&nama);
    dup_nama.sort();
    println!("Nama duplikat: {:?}", dup_nama); // ["Ali", "Siti"]
}
```

---

## Two-Sum Problem — HashSet Trick Klasik

```rust
use std::collections::HashSet;

// Cari sama ada ada dua nombor dalam Vec yang jumlahnya = sasaran
fn ada_dua_sum(nombor: &[i32], sasaran: i32) -> Option<(i32, i32)> {
    let mut seen: HashSet<i32> = HashSet::new();

    for &n in nombor {
        let pasangan = sasaran - n;
        if seen.contains(&pasangan) {
            return Some((pasangan, n));
        }
        seen.insert(n);
    }
    None
}

fn main() {
    let v = vec![2, 7, 11, 15, 3, 6];

    match ada_dua_sum(&v, 9) {
        Some((a, b)) => println!("{} + {} = 9 ✔", a, b), // 2 + 7 = 9
        None         => println!("Tiada pasangan"),
    }

    match ada_dua_sum(&v, 100) {
        Some((a, b)) => println!("{} + {} = 100", a, b),
        None         => println!("Tiada pasangan untuk 100 ✔"),
    }
}
```

---

## Anagram Checker

```rust
use std::collections::HashSet;

fn adalah_anagram(a: &str, b: &str) -> bool {
    // Dua string adalah anagram jika mengandungi huruf yang sama
    if a.len() != b.len() { return false; }

    // Cara 1: guna sort
    let mut ca: Vec<char> = a.to_lowercase().chars().collect();
    let mut cb: Vec<char> = b.to_lowercase().chars().collect();
    ca.sort();
    cb.sort();
    ca == cb
}

fn huruf_sama(a: &str, b: &str) -> HashSet<char> {
    // Huruf yang ada dalam KEDUA-DUA string
    let set_a: HashSet<char> = a.to_lowercase().chars().collect();
    let set_b: HashSet<char> = b.to_lowercase().chars().collect();
    set_a.intersection(&set_b).copied().collect()
}

fn main() {
    println!("listen / silent anagram? {}", adalah_anagram("listen", "silent")); // true
    println!("hello  / world  anagram? {}", adalah_anagram("hello", "world"));   // false

    let sama = huruf_sama("Rust", "Turn");
    let mut v: Vec<char> = sama.into_iter().collect(); v.sort();
    println!("Huruf sama 'Rust' & 'Turn': {:?}", v); // ['r', 't', 'u']
}
```

---

## Graph — Jiran Menggunakan HashSet

```rust
use std::collections::{HashMap, HashSet};

struct Graf {
    tepi: HashMap<&'static str, HashSet<&'static str>>,
}

impl Graf {
    fn baru() -> Self {
        Graf { tepi: HashMap::new() }
    }

    fn tambah_tepi(&mut self, dari: &'static str, ke: &'static str) {
        self.tepi.entry(dari).or_insert_with(HashSet::new).insert(ke);
        self.tepi.entry(ke).or_insert_with(HashSet::new).insert(dari);
    }

    fn jiran(&self, nod: &str) -> HashSet<&&str> {
        self.tepi.get(nod)
            .map(|s| s.iter().collect())
            .unwrap_or_default()
    }

    fn jiran_bersama(&self, a: &str, b: &str) -> HashSet<&'static str> {
        match (self.tepi.get(a), self.tepi.get(b)) {
            (Some(ja), Some(jb)) => ja.intersection(jb).copied().collect(),
            _ => HashSet::new(),
        }
    }
}

fn main() {
    let mut g = Graf::baru();
    g.tambah_tepi("Ali",   "Siti");
    g.tambah_tepi("Ali",   "Amin");
    g.tambah_tepi("Siti",  "Amin");
    g.tambah_tepi("Siti",  "Zara");
    g.tambah_tepi("Amin",  "Zara");

    let jiran_ali = g.jiran("Ali");
    let mut j: Vec<_> = jiran_ali.into_iter().collect(); j.sort();
    println!("Jiran Ali:   {:?}", j);   // ["Amin", "Siti"]

    let mut jb = g.jiran_bersama("Ali", "Zara").into_iter().collect::<Vec<_>>();
    jb.sort();
    println!("Jiran bersama Ali & Zara: {:?}", jb); // ["Amin", "Siti"]
}
```

---

## 🧠 Brain Teaser #4

Tulis fungsi yang terima dua `Vec<String>` dan return tiga benda:
1. Elemen yang ada dalam **kedua-dua** Vec
2. Elemen yang ada dalam **pertama sahaja**
3. Elemen yang ada dalam **kedua sahaja**

<details>
<summary>👀 Jawapan</summary>

```rust
use std::collections::HashSet;

fn analisis_dua_senarai(
    a: Vec<String>,
    b: Vec<String>,
) -> (Vec<String>, Vec<String>, Vec<String>) {
    let set_a: HashSet<String> = a.into_iter().collect();
    let set_b: HashSet<String> = b.into_iter().collect();

    let mut sama: Vec<String> = set_a.intersection(&set_b)
        .cloned().collect();
    let mut a_sahaja: Vec<String> = set_a.difference(&set_b)
        .cloned().collect();
    let mut b_sahaja: Vec<String> = set_b.difference(&set_a)
        .cloned().collect();

    sama.sort(); a_sahaja.sort(); b_sahaja.sort();
    (sama, a_sahaja, b_sahaja)
}

fn main() {
    let a = vec!["Ali".into(), "Siti".into(), "Amin".into()];
    let b = vec!["Siti".into(), "Amin".into(), "Zara".into()];

    let (sama, a_sahaja, b_sahaja) = analisis_dua_senarai(a, b);
    println!("Sama:    {:?}", sama);     // ["Amin", "Siti"]
    println!("A sahaja:{:?}", a_sahaja); // ["Ali"]
    println!("B sahaja:{:?}", b_sahaja); // ["Zara"]
}
```
</details>

---

# BAB 9: Performance & Custom Hash ⚙️

## Default Hasher — SipHash

```rust
use std::collections::HashSet;

fn main() {
    // Default guna SipHash-1-3 — selamat dari HashDoS attack
    // Sesuai untuk: web server, data dari user, production code

    let s: HashSet<i32> = (0..1_000_000).collect();
    println!("1 juta elemen, contains(500000): {}", s.contains(&500000)); // O(1)!
}
```

## Custom Hasher Untuk Performance

```toml
# Cargo.toml
[dependencies]
ahash = "0.8"  # ~2x lebih laju dari SipHash untuk integer
```

```rust
use ahash::AHashSet;

fn main() {
    // AHashSet — laju, sesuai untuk data dalaman (bukan dari user terus)
    let mut s: AHashSet<i32> = AHashSet::new();
    for i in 0..1000 {
        s.insert(i);
    }
    println!("AHashSet size: {}", s.len());
    println!("Contains 500: {}", s.contains(&500));
}
```

## with_capacity — Elak Re-hash

```rust
use std::collections::HashSet;

fn main() {
    // ❌ Tanpa capacity — boleh re-hash beberapa kali
    let mut s1: HashSet<i32> = HashSet::new();
    for i in 0..10000 { s1.insert(i); }

    // ✅ Dengan capacity — allocate sekali, lebih laju
    let mut s2: HashSet<i32> = HashSet::with_capacity(10000);
    for i in 0..10000 { s2.insert(i); }

    println!("Kapasiti s2: {}", s2.capacity()); // ≥ 10000
}
```

---

# BAB 10: Mini Project — Sistem Kehadiran 📋

```rust
use std::collections::{HashMap, HashSet};

struct SistemKehadiran {
    // Nama kelas → set pelajar yang hadir
    kehadiran: HashMap<String, HashSet<String>>,
    // Semua pelajar berdaftar
    pelajar_berdaftar: HashSet<String>,
}

impl SistemKehadiran {
    fn baru() -> Self {
        SistemKehadiran {
            kehadiran: HashMap::new(),
            pelajar_berdaftar: HashSet::new(),
        }
    }

    fn daftar_pelajar(&mut self, nama: &str) {
        let baru = self.pelajar_berdaftar.insert(nama.to_string());
        if baru {
            println!("✔ {} didaftarkan", nama);
        } else {
            println!("⚠ {} sudah berdaftar", nama);
        }
    }

    fn rekod_hadir(&mut self, kelas: &str, nama: &str) {
        if !self.pelajar_berdaftar.contains(nama) {
            println!("✘ {} tidak berdaftar!", nama);
            return;
        }

        let hadir = self.kehadiran
            .entry(kelas.to_string())
            .or_insert_with(HashSet::new)
            .insert(nama.to_string());

        if hadir {
            println!("✔ {} hadir kelas {}", nama, kelas);
        } else {
            println!("⚠ {} dah rekod hadir kelas {} hari ini", nama, kelas);
        }
    }

    fn laporan_kelas(&self, kelas: &str) {
        println!("\n═══ Laporan Kelas: {} ═══", kelas);

        let hadir = match self.kehadiran.get(kelas) {
            Some(s) => s,
            None => {
                println!("  Tiada rekod untuk kelas ini.");
                return;
            }
        };

        // Pelajar yang hadir (sorted)
        let mut hadir_sorted: Vec<&String> = hadir.iter().collect();
        hadir_sorted.sort();
        println!("Hadir ({}):", hadir_sorted.len());
        for p in &hadir_sorted {
            println!("  ✅ {}", p);
        }

        // Pelajar yang TIDAK hadir (difference)
        let tidak_hadir: HashSet<&String> = self.pelajar_berdaftar
            .iter()
            .filter(|p| !hadir.contains(*p))
            .collect();

        let mut tidak_hadir_sorted: Vec<&&String> = tidak_hadir.iter().collect();
        tidak_hadir_sorted.sort();
        println!("Tidak hadir ({}):", tidak_hadir_sorted.len());
        for p in &tidak_hadir_sorted {
            println!("  ⬜ {}", p);
        }

        // Peratus kehadiran
        let total    = self.pelajar_berdaftar.len();
        let peratus  = hadir.len() as f64 / total as f64 * 100.0;
        println!("Kehadiran: {:.1}% ({}/{})", peratus, hadir.len(), total);
    }

    fn pelajar_hadir_semua_kelas(&self) -> Vec<String> {
        // Pelajar yang hadir SEMUA kelas yang ada
        if self.kehadiran.is_empty() {
            return vec![];
        }

        let mut iter = self.kehadiran.values();
        let mut hadir_semua = iter.next().unwrap().clone();

        for kelas_hadir in iter {
            hadir_semua = hadir_semua
                .intersection(kelas_hadir)
                .cloned()
                .collect();
        }

        let mut hasil: Vec<String> = hadir_semua.into_iter().collect();
        hasil.sort();
        hasil
    }

    fn laporan_keseluruhan(&self) {
        println!("\n═══════════════════════════════════");
        println!("       LAPORAN KESELURUHAN         ");
        println!("═══════════════════════════════════");
        println!("Pelajar berdaftar: {}", self.pelajar_berdaftar.len());
        println!("Bilangan kelas:    {}", self.kehadiran.len());

        let rajin = self.pelajar_hadir_semua_kelas();
        println!("\nPelajar hadir SEMUA kelas:");
        if rajin.is_empty() {
            println!("  (tiada)");
        } else {
            for p in &rajin {
                println!("  🏆 {}", p);
            }
        }
    }
}

fn main() {
    let mut sistem = SistemKehadiran::baru();

    // Daftar pelajar
    for nama in &["Ali", "Siti", "Amin", "Zara", "Razi"] {
        sistem.daftar_pelajar(nama);
    }
    sistem.daftar_pelajar("Ali"); // cuba daftar semula

    println!();

    // Rekod kehadiran kelas Matematik
    println!("--- Kelas Matematik ---");
    sistem.rekod_hadir("Matematik", "Ali");
    sistem.rekod_hadir("Matematik", "Siti");
    sistem.rekod_hadir("Matematik", "Amin");
    sistem.rekod_hadir("Matematik", "Ali");   // cuba rekod dua kali
    sistem.rekod_hadir("Matematik", "Hanif"); // pelajar tidak berdaftar

    // Rekod kehadiran kelas Sains
    println!("\n--- Kelas Sains ---");
    sistem.rekod_hadir("Sains", "Ali");
    sistem.rekod_hadir("Sains", "Siti");
    sistem.rekod_hadir("Sains", "Zara");
    sistem.rekod_hadir("Sains", "Razi");

    // Laporan
    sistem.laporan_kelas("Matematik");
    sistem.laporan_kelas("Sains");
    sistem.laporan_keseluruhan();
}
```

**Output contoh:**
```
═══ Laporan Kelas: Matematik ═══
Hadir (3):
  ✅ Ali
  ✅ Amin
  ✅ Siti
Tidak hadir (2):
  ⬜ Razi
  ⬜ Zara
Kehadiran: 60.0% (3/5)

═══ Laporan Kelas: Sains ═══
Hadir (4):
  ✅ Ali
  ✅ Razi
  ✅ Siti
  ✅ Zara
Tidak hadir (1):
  ⬜ Amin
Kehadiran: 80.0% (4/5)

═══════════════════════════════════
       LAPORAN KESELURUHAN
═══════════════════════════════════
Pelajar berdaftar: 5
Bilangan kelas:    2

Pelajar hadir SEMUA kelas:
  🏆 Ali
  🏆 Siti
```

---

# 📋 Rujukan Pantas — HashSet Cheat Sheet

## Methods Penting

| Method | Kegunaan | Return |
|--------|----------|--------|
| `insert(v)` | Masukkan nilai | `bool` (true=baru) |
| `contains(&v)` | Semak kewujudan | `bool` |
| `remove(&v)` | Padam nilai | `bool` |
| `take(&v)` | Padam & ambil nilai | `Option<T>` |
| `get(&v)` | Dapat reference | `Option<&T>` |
| `len()` | Bilangan elemen | `usize` |
| `is_empty()` | Semak kosong | `bool` |
| `clear()` | Kosongkan semua | `()` |
| `retain(\|x\| ...)` | Padam ikut syarat | `()` |

## Set Operations

| Method | Operasi | Nota |
|--------|---------|------|
| `.union(&b)` | A ∪ B | Semua elemen |
| `.intersection(&b)` | A ∩ B | Yang sama sahaja |
| `.difference(&b)` | A − B | Dalam A, tiada dalam B |
| `.symmetric_difference(&b)` | A △ B | Dalam satu sahaja |
| `.is_subset(&b)` | A ⊆ B | Semua A ada dalam B |
| `.is_superset(&b)` | A ⊇ B | Semua B ada dalam A |
| `.is_disjoint(&b)` | A ∩ B = ∅ | Tiada yang sama |

## Buat HashSet Dari Data

```rust
// Dari array
let s: HashSet<i32> = [1, 2, 3].into_iter().collect();

// Dari Vec (buang duplikat!)
let s: HashSet<i32> = vec.into_iter().collect();

// Kosong dengan kapasiti
let s: HashSet<T> = HashSet::with_capacity(n);

// Dari range
let s: HashSet<i32> = (1..=10).collect();
```

## HashSet vs BTreeSet Ringkas

```
Perlu laju (O(1))?    → HashSet
Perlu sorted?         → BTreeSet
Perlu range query?    → BTreeSet
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **Spell Checker** — bina HashSet dari kamus, semak setiap perkataan dalam ayat
2. **Friend Suggester** — guna set intersection untuk cadang "kawan punya kawan"
3. **Tag Filter** — sistem artikel dengan tags, cari artikel yang ada tag tertentu
4. **Unique Visitor Counter** — rekod IP unik yang melawat setiap halaman web

---

*HashSet in Rust — dari `contains()` hingga set operations penuh. Selamat belajar!* 🦀
