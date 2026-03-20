# 📦 Vec dalam Rust — Beginner to Advanced

> Panduan lengkap Vec<T>: dari konsep asas hingga pattern lanjutan.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu Vec?

```
┌─────────────────────────────────────────────────────┐
│                    Vec<i32>                         │
│                                                     │
│  Index:  [0]   [1]   [2]   [3]   [4]               │
│  Data:   [10]  [20]  [30]  [40]  [50]               │
│                                                     │
│  len      = 5  (bilangan elemen)                    │
│  capacity = 8  (ruang yang dialokasi)               │
└─────────────────────────────────────────────────────┘
```

`Vec<T>` adalah **dynamic array** — boleh tambah atau buang elemen pada masa runtime. Berbeza dengan array biasa `[T; N]` yang saiznya tetap semasa compile.

| Bahasa | Equivalent |
|--------|-----------|
| PHP | `$arr = [1, 2, 3]` |
| Python | `lst = [1, 2, 3]` |
| JavaScript | `arr = [1, 2, 3]` |
| Java | `ArrayList<Integer>` |
| C++ | `std::vector<int>` |
| **Rust** | `Vec<i32>` |

---

## Peta Pembelajaran

```
Bab 1  → Buat Vec
Bab 2  → Tambah & Buang Elemen
Bab 3  → Baca & Akses Elemen
Bab 4  → Iterate (Loop)
Bab 5  → Slice — Pandang Dalam Vec
Bab 6  → Sort & Cari
Bab 7  → Transform — map, filter, collect
Bab 8  → Vec dengan Struct
Bab 9  → Pattern Lanjutan
Bab 10 → Mini Project: Senarai Tugasan
```

---

# BAB 1: Buat Vec 🏗️

## Cara-Cara Buat Vec

```rust
fn main() {
    // Cara 1: kosong, nyatakan type
    let v: Vec<i32> = Vec::new();

    // Cara 2: guna macro vec! — paling common
    let v = vec![10, 20, 30, 40, 50];

    // Cara 3: kosong, biar Rust infer type dari push pertama
    let mut v = Vec::new();
    v.push(100_i32);  // Rust tahu type sekarang

    // Cara 4: dengan kapasiti awal — elak re-allocate
    let mut v: Vec<String> = Vec::with_capacity(10);

    // Cara 5: isi dengan nilai sama
    let zeros = vec![0; 5];         // [0, 0, 0, 0, 0]
    let hello = vec!["hai"; 3];     // ["hai", "hai", "hai"]

    // Cara 6: dari range
    let satu_hingga_lima: Vec<i32> = (1..=5).collect();
    println!("{:?}", satu_hingga_lima); // [1, 2, 3, 4, 5]

    // Cara 7: dari iterator
    let genap: Vec<i32> = (1..=10).filter(|x| x % 2 == 0).collect();
    println!("{:?}", genap); // [2, 4, 6, 8, 10]
}
```

---

## len vs capacity

```rust
fn main() {
    let mut v: Vec<i32> = Vec::with_capacity(4);

    println!("len={}, capacity={}", v.len(), v.capacity()); // 0, 4

    v.push(1);
    v.push(2);
    println!("len={}, capacity={}", v.len(), v.capacity()); // 2, 4

    v.push(3);
    v.push(4);
    v.push(5); // ← melebihi capacity! Rust auto double
    println!("len={}, capacity={}", v.len(), v.capacity()); // 5, 8

    // Paksa shrink capacity ke len
    v.shrink_to_fit();
    println!("len={}, capacity={}", v.len(), v.capacity()); // 5, 5
}
```

```
Visualisasi memory:

Sebelum push ke-5:
[1][2][3][4][ ][ ][ ][ ]
 ↑___len=4___↑  ↑cap=4↑

Selepas push ke-5 (auto grow):
[1][2][3][4][5][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ]
 ↑______len=5______↑  ↑_________cap=8_________↑
```

> 💡 **Tip:** Guna `Vec::with_capacity(n)` kalau tahu lebih kurang berapa banyak elemen — elak re-allocate berulang kali yang lambatkan program.

---

## 🧠 Brain Teaser #1

Apa output kod ni?

```rust
fn main() {
    let a = vec![1, 2, 3];
    let b = vec![0; 4];
    let c: Vec<i32> = (1..4).collect();

    println!("{}", a.len() + b.len() + c.len());
}
```

<details>
<summary>👀 Jawapan</summary>

Output: `10`

- `a.len()` = 3 (tiga elemen)
- `b.len()` = 4 (empat sifar)
- `c.len()` = 3 (1, 2, 3 — range `1..4` tidak termasuk 4)
- 3 + 4 + 3 = **10**
</details>

---

# BAB 2: Tambah & Buang Elemen ➕➖

## Tambah Elemen

```rust
fn main() {
    let mut v: Vec<i32> = Vec::new();

    // push — tambah di HUJUNG (O(1) amortized)
    v.push(10);
    v.push(20);
    v.push(30);
    println!("{:?}", v); // [10, 20, 30]

    // insert — tambah di KEDUDUKAN tertentu (O(n) — lambat untuk Vec besar!)
    v.insert(1, 99); // sisip 99 di index 1
    println!("{:?}", v); // [10, 99, 20, 30]

    // extend — tambah banyak elemen sekaligus
    v.extend([40, 50, 60]);
    println!("{:?}", v); // [10, 99, 20, 30, 40, 50, 60]

    // extend dari Vec lain
    let tambahan = vec![70, 80];
    v.extend(tambahan.iter());
    println!("{:?}", v);

    // append — alihkan semua elemen dari Vec lain (kosongkan sumber)
    let mut extra = vec![100, 200];
    v.append(&mut extra);
    println!("v: {:?}", v);
    println!("extra selepas append: {:?}", extra); // [] — dah kosong!
}
```

---

## Buang Elemen

```rust
fn main() {
    let mut v = vec![10, 20, 30, 40, 50];

    // pop — buang dari HUJUNG (O(1)), return Option<T>
    let terakhir = v.pop();
    println!("Pop: {:?}", terakhir); // Some(50)
    println!("{:?}", v);             // [10, 20, 30, 40]

    // remove — buang di INDEX tertentu (O(n)), return T
    let dibuang = v.remove(1); // buang index 1
    println!("Remove: {}", dibuang); // 20
    println!("{:?}", v);             // [10, 30, 40]

    // swap_remove — buang index, ISI dengan elemen TERAKHIR (O(1)!)
    // Urutan berubah tapi lebih laju dari remove
    let mut v2 = vec![10, 20, 30, 40, 50];
    let dibuang2 = v2.swap_remove(1); // buang index 1 (20), ganti dengan 50
    println!("swap_remove: {}", dibuang2); // 20
    println!("{:?}", v2);                  // [10, 50, 30, 40]

    // retain — kekalkan elemen yang MEMENUHI syarat
    let mut v3 = vec![1, 2, 3, 4, 5, 6, 7, 8];
    v3.retain(|&x| x % 2 == 0); // kekalkan nombor genap sahaja
    println!("{:?}", v3); // [2, 4, 6, 8]

    // clear — kosongkan semua
    v3.clear();
    println!("Selepas clear: {:?}", v3); // []
    println!("Len: {}", v3.len());       // 0

    // truncate — potong ke panjang tertentu
    let mut v4 = vec![1, 2, 3, 4, 5];
    v4.truncate(3);
    println!("{:?}", v4); // [1, 2, 3]

    // dedup — buang CONSECUTIVE duplicates (perlu sort dulu kalau nak buang semua duplikat)
    let mut v5 = vec![1, 1, 2, 3, 3, 3, 2, 1];
    v5.dedup();
    println!("{:?}", v5); // [1, 2, 3, 2, 1] — hanya consecutive yg dibuang

    let mut v6 = vec![1, 1, 2, 2, 3, 3];
    v6.sort();
    v6.dedup();
    println!("{:?}", v6); // [1, 2, 3] — sort dulu, baru dedup
}
```

---

## remove vs swap_remove

```
remove(1):                    swap_remove(1):
[A][B][C][D][E]               [A][B][C][D][E]
    ↑ buang B                     ↑ buang B, ganti dengan E
[A][C][D][E]                  [A][E][C][D]
 ← semua shift kiri →          tiada shifting! O(1)
 O(n) — lambat                 LAJU tapi urutan berubah
```

> 💡 Guna `swap_remove` bila urutan tidak penting — jauh lebih laju!

---

# BAB 3: Baca & Akses Elemen 🔍

## Akses Nilai

```rust
fn main() {
    let v = vec![10, 20, 30, 40, 50];

    // Cara 1: index [] — PANIC kalau out of bounds!
    println!("{}", v[0]); // 10
    println!("{}", v[4]); // 50
    // println!("{}", v[5]); // ← PANIC! index out of bounds

    // Cara 2: get() — return Option<&T>, SELAMAT
    match v.get(2) {
        Some(val) => println!("Index 2: {}", val),
        None      => println!("Index tidak wujud"),
    }

    println!("{:?}", v.get(10)); // None — tiada panic!

    // Cara 3: first() dan last()
    println!("Pertama: {:?}", v.first()); // Some(10)
    println!("Terakhir: {:?}", v.last()); // Some(50)

    let kosong: Vec<i32> = Vec::new();
    println!("Pertama kosong: {:?}", kosong.first()); // None

    // Cara 4: unwrap_or dengan default
    let val = v.get(99).copied().unwrap_or(0);
    println!("Get 99 atau 0: {}", val); // 0
}
```

---

## Cari Dalam Vec

```rust
fn main() {
    let v = vec![15, 3, 42, 8, 27, 42, 99];

    // contains — adakah nilai wujud?
    println!("Ada 42? {}", v.contains(&42)); // true
    println!("Ada 100? {}", v.contains(&100)); // false

    // position — index PERTAMA yang sepadan
    let pos = v.iter().position(|&x| x == 42);
    println!("42 pertama di index: {:?}", pos); // Some(2)

    // rposition — index TERAKHIR yang sepadan
    let rpos = v.iter().rposition(|&x| x == 42);
    println!("42 terakhir di index: {:?}", rpos); // Some(5)

    // find — nilai pertama yang memenuhi syarat
    let besar = v.iter().find(|&&x| x > 40);
    println!("Nilai > 40 pertama: {:?}", besar); // Some(42)

    // any & all
    println!("Ada > 90? {}", v.iter().any(|&x| x > 90));   // true
    println!("Semua > 0? {}", v.iter().all(|&x| x > 0));   // true
    println!("Semua > 10? {}", v.iter().all(|&x| x > 10)); // false

    // min & max
    println!("Min: {:?}", v.iter().min()); // Some(3)
    println!("Max: {:?}", v.iter().max()); // Some(99)
}
```

---

## 🧠 Brain Teaser #2

Apa bezanya dua kod ni? Mana yang lebih selamat?

```rust
// Kod A
let v = vec![1, 2, 3];
let x = v[5];

// Kod B
let v = vec![1, 2, 3];
let x = v.get(5);
```

<details>
<summary>👀 Jawapan</summary>

**Kod A** — PANIC! `v[5]` akan crash program dengan `index out of bounds` kerana Vec hanya ada 3 elemen (index 0, 1, 2).

**Kod B** — SELAMAT. `v.get(5)` return `None` tanpa crash. Kita boleh handle dengan `match`, `if let`, atau `unwrap_or`.

**Pilih `get()` bila tidak pasti index sah atau tidak.**
</details>

---

# BAB 4: Iterate (Loop) 🔄

## Semua Cara Iterate

```rust
fn main() {
    let v = vec![10, 20, 30, 40, 50];

    // Cara 1: borrow — paling common untuk baca sahaja
    println!("--- Borrow ---");
    for x in &v {
        print!("{} ", x); // x adalah &i32
    }
    println!();

    // Cara 2: mutable borrow — untuk ubah nilai
    let mut v2 = vec![1, 2, 3, 4, 5];
    for x in &mut v2 {
        *x *= 10; // dereference untuk ubah nilai
    }
    println!("Selepas *10: {:?}", v2); // [10, 20, 30, 40, 50]

    // Cara 3: consume — ambil ownership (Vec tidak boleh guna lepas ni)
    let v3 = vec![100, 200, 300];
    for x in v3 {  // v3 moved ke loop
        print!("{} ", x);
    }
    println!();
    // println!("{:?}", v3); // ← ERROR! v3 sudah moved

    // Cara 4: dengan index guna enumerate
    let buah = vec!["epal", "mangga", "pisang"];
    for (i, nama) in buah.iter().enumerate() {
        println!("  [{}] {}", i, nama);
    }

    // Cara 5: zip — iterate dua Vec serentak
    let nama  = vec!["Ali", "Siti", "Amin"];
    let markah = vec![85, 92, 78];
    for (n, m) in nama.iter().zip(markah.iter()) {
        println!("  {} → {}", n, m);
    }

    // Cara 6: rev — iterate terbalik
    let v4 = vec![1, 2, 3, 4, 5];
    for x in v4.iter().rev() {
        print!("{} ", x); // 5 4 3 2 1
    }
    println!();

    // Cara 7: step_by
    for x in (0..10).step_by(2) {
        print!("{} ", x); // 0 2 4 6 8
    }
    println!();
}
```

---

## Iterator Adapters Yang Berguna

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // take — ambil n elemen pertama
    let tiga: Vec<_> = v.iter().take(3).collect();
    println!("take(3): {:?}", tiga); // [1, 2, 3]

    // skip — langkau n elemen pertama
    let skip3: Vec<_> = v.iter().skip(3).collect();
    println!("skip(3): {:?}", skip3); // [4, 5, 6, 7, 8, 9, 10]

    // take + skip = pagination!
    let halaman = 2;
    let saiz = 3;
    let page: Vec<_> = v.iter().skip((halaman - 1) * saiz).take(saiz).collect();
    println!("Halaman {}: {:?}", halaman, page); // [4, 5, 6]

    // chain — gabung dua iterator
    let a = vec![1, 2, 3];
    let b = vec![4, 5, 6];
    let gabung: Vec<_> = a.iter().chain(b.iter()).collect();
    println!("chain: {:?}", gabung); // [1, 2, 3, 4, 5, 6]

    // flatten — Vec dalam Vec → Vec flat
    let nested = vec![vec![1, 2], vec![3, 4], vec![5, 6]];
    let flat: Vec<_> = nested.into_iter().flatten().collect();
    println!("flatten: {:?}", flat); // [1, 2, 3, 4, 5, 6]

    // windows — sliding window
    let w = vec![1, 2, 3, 4, 5];
    for window in w.windows(3) {
        println!("window: {:?}", window);
    }
    // [1,2,3], [2,3,4], [3,4,5]

    // chunks — bahagi kepada kumpulan saiz n
    for chunk in w.chunks(2) {
        println!("chunk: {:?}", chunk);
    }
    // [1,2], [3,4], [5]
}
```

---

# BAB 5: Slice — Pandang Dalam Vec 🔭

## Apa Itu Slice?

```
Vec<i32>:   [10][20][30][40][50]
              ↑               ↑
              0               4

Slice &v[1..4]:  [20][30][40]
                   ↑       ↑
                   1       3 (index 4 tidak termasuk)
```

Slice `&[T]` adalah **pandangan (view)** ke data Vec tanpa copy — tiada ownership, tiada allocate.

```rust
fn jumlah(data: &[i32]) -> i32 {    // terima slice, bukan Vec
    data.iter().sum()
}

fn main() {
    let v = vec![10, 20, 30, 40, 50];

    // Buat slice dari Vec
    let semua: &[i32]   = &v;           // semua elemen
    let tiga:  &[i32]   = &v[1..4];     // index 1,2,3 → [20,30,40]
    let awal:  &[i32]   = &v[..3];      // index 0,1,2 → [10,20,30]
    let akhir: &[i32]   = &v[2..];      // index 2 hingga akhir → [30,40,50]
    let sama:  &[i32]   = &v[1..=3];    // inclusive → [20,30,40]

    println!("{:?}", tiga);  // [20, 30, 40]
    println!("{:?}", awal);  // [10, 20, 30]
    println!("{:?}", akhir); // [30, 40, 50]

    // Fungsi yang terima slice boleh terima Vec atau array!
    println!("Jumlah Vec:   {}", jumlah(&v));
    println!("Jumlah slice: {}", jumlah(&v[1..4]));

    let arr = [1, 2, 3, 4, 5];
    println!("Jumlah array: {}", jumlah(&arr)); // array pun OK!
}
```

---

## split_at & split Slice

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5, 6];

    // split_at — bahagi kepada dua
    let (kiri, kanan) = v.split_at(3);
    println!("Kiri:  {:?}", kiri);  // [1, 2, 3]
    println!("Kanan: {:?}", kanan); // [4, 5, 6]

    // split — bahagi berdasarkan syarat
    let data = vec![1, 0, 2, 0, 3, 0, 4];
    let bahagian: Vec<&[i32]> = data.split(|&x| x == 0).collect();
    for b in &bahagian {
        println!("{:?}", b);
    }
    // [1], [2], [3], [4]
}
```

---

## 🧠 Brain Teaser #3

Kenapa fungsi ni lebih baik terima `&[T]` dari `&Vec<T>`?

```rust
// A: terima &Vec<T>
fn cetak_a(v: &Vec<i32>) {
    for x in v { println!("{}", x); }
}

// B: terima &[T]
fn cetak_b(v: &[i32]) {
    for x in v { println!("{}", x); }
}
```

<details>
<summary>👀 Jawapan</summary>

Fungsi **B** lebih fleksibel kerana `&[T]` boleh menerima:
- `&Vec<i32>` — auto coerce
- `&[i32; 5]` — array
- `&v[1..4]` — slice dari Vec
- Apa sahaja yang boleh jadi slice

Fungsi A hanya boleh terima `&Vec<i32>`.

**Peraturan Rust:** Bila parameter adalah untuk baca sahaja, lebih suka `&[T]` dari `&Vec<T>`.
</details>

---

# BAB 6: Sort & Cari 🔢

## Sort Pelbagai Cara

```rust
fn main() {
    // Sort nombor
    let mut v = vec![5, 2, 8, 1, 9, 3];
    v.sort();
    println!("Ascending:  {:?}", v); // [1, 2, 3, 5, 8, 9]

    v.sort_by(|a, b| b.cmp(a)); // descending
    println!("Descending: {:?}", v); // [9, 8, 5, 3, 2, 1]

    // Sort float (guna partial_cmp)
    let mut f = vec![3.1, 1.4, 2.7, 0.5];
    f.sort_by(|a, b| a.partial_cmp(b).unwrap());
    println!("Float sort: {:?}", f); // [0.5, 1.4, 2.7, 3.1]

    // Sort string
    let mut nama = vec!["Zara", "Ali", "Siti", "Amin"];
    nama.sort();
    println!("Sort nama: {:?}", nama); // ["Ali", "Amin", "Siti", "Zara"]

    // sort_by_key — sort mengikut field struct
    #[derive(Debug)]
    struct Pelajar { nama: &'static str, markah: u32 }

    let mut pelajar = vec![
        Pelajar { nama: "Ali",  markah: 85 },
        Pelajar { nama: "Siti", markah: 92 },
        Pelajar { nama: "Amin", markah: 78 },
    ];

    // Sort by markah (ascending)
    pelajar.sort_by_key(|p| p.markah);
    for p in &pelajar {
        println!("  {} — {}", p.nama, p.markah);
    }

    // Sort by markah (descending) — guna Reverse
    use std::cmp::Reverse;
    pelajar.sort_by_key(|p| Reverse(p.markah));
    for p in &pelajar {
        println!("  {} — {}", p.nama, p.markah);
    }
}
```

---

## Binary Search (Carian Laju)

```rust
fn main() {
    let mut v = vec![10, 20, 30, 40, 50];
    v.sort(); // binary_search WAJIB sorted dulu!

    match v.binary_search(&30) {
        Ok(i)  => println!("Jumpa 30 di index {}", i),  // Ok(2)
        Err(i) => println!("Tiada 30, patut di index {}", i),
    }

    match v.binary_search(&25) {
        Ok(i)  => println!("Jumpa 25 di index {}", i),
        Err(i) => println!("Tiada 25, insertion point: {}", i), // Err(2)
    }

    // binary_search_by — dengan custom comparator
    #[derive(Debug)]
    struct Item { id: u32, nama: &'static str }

    let items = vec![
        Item { id: 1, nama: "Epal" },
        Item { id: 3, nama: "Mangga" },
        Item { id: 5, nama: "Pisang" },
    ];

    let cari_id = 3;
    let result = items.binary_search_by(|item| item.id.cmp(&cari_id));
    if let Ok(i) = result {
        println!("Jumpa: {:?}", items[i].nama); // "Mangga"
    }
}
```

---

# BAB 7: Transform — map, filter, collect 🔀

## Iterator Chains — Cara Idiomatic Rust

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // map — transform setiap elemen
    let kuasa_dua: Vec<i32> = v.iter()
        .map(|&x| x * x)
        .collect();
    println!("Kuasa dua: {:?}", kuasa_dua);
    // [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

    // filter — pilih elemen yang memenuhi syarat
    let genap: Vec<&i32> = v.iter()
        .filter(|&&x| x % 2 == 0)
        .collect();
    println!("Genap: {:?}", genap);
    // [2, 4, 6, 8, 10]

    // filter + map dalam satu langkah
    let genap_kuasa: Vec<i32> = v.iter()
        .filter(|&&x| x % 2 == 0)
        .map(|&x| x * x)
        .collect();
    println!("Genap kuasa dua: {:?}", genap_kuasa);
    // [4, 16, 36, 64, 100]

    // filter_map — filter DAN transform serentak
    let string_nombor = vec!["1", "dua", "3", "empat", "5"];
    let nombor: Vec<i32> = string_nombor.iter()
        .filter_map(|s| s.parse::<i32>().ok())
        .collect();
    println!("Parse berjaya: {:?}", nombor); // [1, 3, 5]

    // fold — kumpul jadi satu nilai (macam reduce)
    let jumlah = v.iter().fold(0, |acc, &x| acc + x);
    println!("Jumlah: {}", jumlah); // 55

    // flat_map — map + flatten
    let ayat = vec!["hello world", "foo bar baz"];
    let perkataan: Vec<&str> = ayat.iter()
        .flat_map(|s| s.split_whitespace())
        .collect();
    println!("Perkataan: {:?}", perkataan);
    // ["hello", "world", "foo", "bar", "baz"]

    // sum & product
    let jumlah2: i32 = v.iter().sum();
    let hasil: i32 = vec![1, 2, 3, 4, 5].iter().product();
    println!("Sum: {}, Product: {}", jumlah2, hasil); // 55, 120
}
```

---

## collect ke Pelbagai Jenis

```rust
use std::collections::{HashMap, HashSet};

fn main() {
    let v = vec![1, 2, 3, 2, 1, 4];

    // collect ke Vec
    let v2: Vec<i32> = v.iter().copied().collect();

    // collect ke HashSet — buang duplikat!
    let unik: HashSet<i32> = v.iter().copied().collect();
    println!("Unik: {:?}", unik); // {1, 2, 3, 4}

    // collect ke HashMap
    let pasangan = vec![("Ali", 85), ("Siti", 92)];
    let map: HashMap<&str, i32> = pasangan.into_iter().collect();
    println!("{:?}", map);

    // collect String dari Vec<char>
    let huruf = vec!['R', 'u', 's', 't'];
    let perkataan: String = huruf.iter().collect();
    println!("{}", perkataan); // Rust

    // join — Vec<String> → String
    let nama = vec!["Ali", "Siti", "Amin"];
    let gabung = nama.join(", ");
    println!("{}", gabung); // Ali, Siti, Amin
}
```

---

## 🧠 Brain Teaser #4

Tulis satu baris kod (tanpa `for` loop) untuk hasilkan Vec yang mengandungi kuasa dua semua nombor ganjil dari 1 hingga 20.

Expected: `[1, 9, 25, 49, 81, 121, 169, 225, 289, 361]`

<details>
<summary>👀 Jawapan</summary>

```rust
let hasil: Vec<i32> = (1..=20)
    .filter(|x| x % 2 != 0)
    .map(|x| x * x)
    .collect();
println!("{:?}", hasil);
```

Atau lebih ringkas dengan `step_by`:
```rust
let hasil: Vec<i32> = (1..=20).step_by(2).map(|x| x * x).collect();
```
</details>

---

# BAB 8: Vec dengan Struct 🏗️

## Vec Sebagai Container Struct

```rust
#[derive(Debug, Clone)]
struct Pekerja {
    id:       u32,
    nama:     String,
    bahagian: String,
    gaji:     f64,
}

impl Pekerja {
    fn baru(id: u32, nama: &str, bahagian: &str, gaji: f64) -> Self {
        Pekerja { id, nama: nama.into(), bahagian: bahagian.into(), gaji }
    }
}

fn main() {
    let mut pekerja: Vec<Pekerja> = vec![
        Pekerja::baru(1, "Ali Ahmad",  "ICT",       4500.0),
        Pekerja::baru(2, "Siti Hawa",  "Kewangan",  3800.0),
        Pekerja::baru(3, "Amin Razak", "ICT",       5200.0),
        Pekerja::baru(4, "Zara Lina",  "HR",        4100.0),
        Pekerja::baru(5, "Razi Malik", "Kewangan",  3600.0),
    ];

    // Filter: cari pekerja ICT sahaja
    let ict: Vec<&Pekerja> = pekerja.iter()
        .filter(|p| p.bahagian == "ICT")
        .collect();
    println!("Pekerja ICT:");
    for p in &ict {
        println!("  {} — RM{:.0}", p.nama, p.gaji);
    }

    // Sort: susun ikut gaji (tinggi ke rendah)
    pekerja.sort_by(|a, b| b.gaji.partial_cmp(&a.gaji).unwrap());
    println!("\nTop gaji:");
    for p in pekerja.iter().take(3) {
        println!("  {} — RM{:.0}", p.nama, p.gaji);
    }

    // Aggregate: jumlah gaji per bahagian
    use std::collections::HashMap;
    let mut gaji_bahagian: HashMap<&str, f64> = HashMap::new();
    for p in &pekerja {
        *gaji_bahagian.entry(&p.bahagian).or_insert(0.0) += p.gaji;
    }
    println!("\nJumlah gaji per bahagian:");
    for (bahagian, jumlah) in &gaji_bahagian {
        println!("  {}: RM{:.0}", bahagian, jumlah);
    }

    // Kemaskini: naik gaji 10% untuk ICT
    for p in pekerja.iter_mut() {
        if p.bahagian == "ICT" {
            p.gaji *= 1.10;
        }
    }

    // Cari: guna find untuk satu pekerja
    if let Some(p) = pekerja.iter().find(|p| p.id == 2) {
        println!("\nJumpa: {} dari {}", p.nama, p.bahagian);
    }

    // Padam: buang pekerja dengan gaji < 4000
    pekerja.retain(|p| p.gaji >= 4000.0);
    println!("\nSelepas retain (gaji ≥ 4000): {} pekerja", pekerja.len());
}
```

---

# BAB 9: Pattern Lanjutan 🎯

## Partition — Bahagi Kepada Dua Vec

```rust
fn main() {
    let nombor = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // partition — bahagi berdasarkan syarat → (Vec_true, Vec_false)
    let (genap, ganjil): (Vec<i32>, Vec<i32>) = nombor
        .into_iter()
        .partition(|&x| x % 2 == 0);

    println!("Genap: {:?}", genap);  // [2, 4, 6, 8, 10]
    println!("Ganjil: {:?}", ganjil); // [1, 3, 5, 7, 9]
}
```

---

## Dedup & Unik

```rust
fn main() {
    // Buang duplikat sambil kekal order
    fn unik<T: Eq + std::hash::Hash + Clone>(v: Vec<T>) -> Vec<T> {
        use std::collections::HashSet;
        let mut seen = HashSet::new();
        v.into_iter().filter(|x| seen.insert(x.clone())).collect()
    }

    let v = vec![3, 1, 4, 1, 5, 9, 2, 6, 5, 3];
    println!("Unik (kekal order): {:?}", unik(v)); // [3, 1, 4, 5, 9, 2, 6]

    // Cara lain guna sort + dedup (order berubah)
    let mut v2 = vec![3, 1, 4, 1, 5, 9, 2, 6, 5, 3];
    v2.sort();
    v2.dedup();
    println!("Unik (sorted):      {:?}", v2); // [1, 2, 3, 4, 5, 6, 9]
}
```

---

## Rotate & Reverse

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5];

    // reverse — terbalikkan in-place
    v.reverse();
    println!("Reverse: {:?}", v); // [5, 4, 3, 2, 1]

    // rotate_left — pusing ke kiri
    let mut v2 = vec![1, 2, 3, 4, 5];
    v2.rotate_left(2);
    println!("Rotate left 2:  {:?}", v2); // [3, 4, 5, 1, 2]

    // rotate_right — pusing ke kanan
    let mut v3 = vec![1, 2, 3, 4, 5];
    v3.rotate_right(2);
    println!("Rotate right 2: {:?}", v3); // [4, 5, 1, 2, 3]
}
```

---

## Vec Sebagai Stack dan Queue

```rust
fn main() {
    // Vec sebagai STACK (LIFO — Last In, First Out)
    let mut stack: Vec<i32> = Vec::new();
    stack.push(1);  // push ke atas
    stack.push(2);
    stack.push(3);
    println!("Stack: {:?}", stack);      // [1, 2, 3]
    println!("Pop: {:?}", stack.pop());  // Some(3)
    println!("Pop: {:?}", stack.pop());  // Some(2)
    println!("Stack: {:?}", stack);      // [1]

    // Vec sebagai QUEUE (FIFO — First In, First Out)
    // ⚠️ Guna VecDeque untuk queue yang lebih efisien
    let mut queue: Vec<i32> = Vec::new();
    queue.push(1);         // enqueue
    queue.push(2);
    queue.push(3);
    let depan = queue.remove(0); // dequeue — O(n)! guna VecDeque untuk O(1)
    println!("Dequeue: {}", depan); // 1
    println!("Queue: {:?}", queue); // [2, 3]

    // Untuk queue yang betul, guna VecDeque
    use std::collections::VecDeque;
    let mut vd: VecDeque<i32> = VecDeque::new();
    vd.push_back(1);   // enqueue belakang
    vd.push_back(2);
    vd.push_front(0);  // boleh tambah depan juga!
    println!("VecDeque front: {:?}", vd.pop_front()); // Some(0) — O(1)
}
```

---

## Transpose Vec<Vec<T>> (Matrix)

```rust
fn transpose<T: Clone>(matrix: Vec<Vec<T>>) -> Vec<Vec<T>> {
    if matrix.is_empty() { return vec![]; }
    let baris = matrix.len();
    let lajur = matrix[0].len();

    (0..lajur)
        .map(|j| (0..baris).map(|i| matrix[i][j].clone()).collect())
        .collect()
}

fn main() {
    let matrix = vec![
        vec![1, 2, 3],
        vec![4, 5, 6],
        vec![7, 8, 9],
    ];

    println!("Asal:");
    for baris in &matrix {
        println!("  {:?}", baris);
    }

    let t = transpose(matrix);
    println!("Selepas transpose:");
    for baris in &t {
        println!("  {:?}", baris);
    }
}
```

---

## 🧠 Brain Teaser #5 (Advanced)

Bina fungsi `chunk_by_sum` yang terima `Vec<u32>` dan `limit: u32`, kembalikan `Vec<Vec<u32>>` di mana setiap sub-Vec jumlahnya tidak melebihi `limit`.

Contoh: `chunk_by_sum(vec![3, 1, 4, 1, 5, 9, 2, 6], 10)` → `[[3,1,4,1],[5],[9],[2,6]]`

<details>
<summary>👀 Jawapan</summary>

```rust
fn chunk_by_sum(data: Vec<u32>, limit: u32) -> Vec<Vec<u32>> {
    let mut hasil: Vec<Vec<u32>> = Vec::new();
    let mut semasa: Vec<u32>     = Vec::new();
    let mut jumlah: u32          = 0;

    for nilai in data {
        if jumlah + nilai > limit && !semasa.is_empty() {
            hasil.push(semasa.clone());
            semasa.clear();
            jumlah = 0;
        }
        semasa.push(nilai);
        jumlah += nilai;
    }

    if !semasa.is_empty() {
        hasil.push(semasa);
    }

    hasil
}

fn main() {
    let data = vec![3, 1, 4, 1, 5, 9, 2, 6];
    let chunks = chunk_by_sum(data, 10);
    println!("{:?}", chunks);
    // [[3, 1, 4, 1], [5], [9], [2, 6]]
}
```
</details>

---

# BAB 10: Mini Project — Senarai Tugasan CLI ✅

```rust
use std::io::{self, Write};

#[derive(Debug, Clone)]
struct Tugasan {
    id:       u32,
    tajuk:    String,
    selesai:  bool,
    keutamaan: u8,  // 1=rendah, 2=sederhana, 3=tinggi
}

impl Tugasan {
    fn baru(id: u32, tajuk: &str, keutamaan: u8) -> Self {
        Tugasan { id, tajuk: tajuk.into(), selesai: false, keutamaan }
    }

    fn papar(&self) {
        let status = if self.selesai { "✅" } else { "⬜" };
        let tahap  = match self.keutamaan {
            3 => "🔴 Tinggi",
            2 => "🟡 Sederhana",
            _ => "🟢 Rendah",
        };
        println!("  {} [{}] {} ({})", status, self.id, self.tajuk, tahap);
    }
}

struct SenaraiTugasan {
    tugasan: Vec<Tugasan>,
    id_seterusnya: u32,
}

impl SenaraiTugasan {
    fn baru() -> Self {
        SenaraiTugasan { tugasan: Vec::new(), id_seterusnya: 1 }
    }

    fn tambah(&mut self, tajuk: &str, keutamaan: u8) {
        let id = self.id_seterusnya;
        self.tugasan.push(Tugasan::baru(id, tajuk, keutamaan));
        self.id_seterusnya += 1;
        println!("✔ Ditambah: \"{}\"", tajuk);
    }

    fn selesai(&mut self, id: u32) {
        match self.tugasan.iter_mut().find(|t| t.id == id) {
            Some(t) => {
                t.selesai = true;
                println!("✅ Selesai: \"{}\"", t.tajuk);
            }
            None => println!("❌ ID {} tidak dijumpai", id),
        }
    }

    fn padam(&mut self, id: u32) {
        let sebelum = self.tugasan.len();
        self.tugasan.retain(|t| t.id != id);
        if self.tugasan.len() < sebelum {
            println!("🗑️  ID {} dipadam", id);
        } else {
            println!("❌ ID {} tidak dijumpai", id);
        }
    }

    fn papar_semua(&self) {
        if self.tugasan.is_empty() {
            println!("  (tiada tugasan)");
            return;
        }

        // Sort by keutamaan (tinggi → rendah), then by id
        let mut dipapar: Vec<&Tugasan> = self.tugasan.iter().collect();
        dipapar.sort_by(|a, b| {
            b.keutamaan.cmp(&a.keutamaan).then(a.id.cmp(&b.id))
        });

        println!("\n  --- Semua Tugasan ---");
        for t in &dipapar {
            t.papar();
        }

        // Statistik
        let selesai = self.tugasan.iter().filter(|t| t.selesai).count();
        let jumlah  = self.tugasan.len();
        println!("  Selesai: {}/{}", selesai, jumlah);
    }

    fn tapis(&self, selesai_sahaja: bool) {
        let label = if selesai_sahaja { "Selesai" } else { "Belum selesai" };
        let hasil: Vec<&Tugasan> = self.tugasan.iter()
            .filter(|t| t.selesai == selesai_sahaja)
            .collect();

        println!("\n  --- {} ---", label);
        if hasil.is_empty() {
            println!("  (tiada)");
        } else {
            for t in hasil { t.papar(); }
        }
    }
}

fn main() {
    let mut senarai = SenaraiTugasan::baru();

    println!("📋 Senarai Tugasan");
    println!("==================");
    println!("Arahan: tambah <tajuk> [1/2/3] | selesai <id> | padam <id>");
    println!("        papar | selesai_sahaja | belum_selesai | keluar\n");

    loop {
        print!("> ");
        io::stdout().flush().unwrap();

        let mut input = String::new();
        io::stdin().read_line(&mut input).unwrap();
        let input = input.trim();
        let bahagian: Vec<&str> = input.splitn(3, ' ').collect();

        match bahagian[0] {
            "tambah" if bahagian.len() >= 2 => {
                let tajuk = bahagian[1];
                // keutamaan optional, default = 2
                let keutamaan: u8 = bahagian.get(2)
                    .and_then(|s| s.parse().ok())
                    .unwrap_or(2)
                    .min(3).max(1);
                senarai.tambah(tajuk, keutamaan);
            }
            "selesai" if bahagian.len() >= 2 => {
                if let Ok(id) = bahagian[1].parse::<u32>() {
                    senarai.selesai(id);
                }
            }
            "padam" if bahagian.len() >= 2 => {
                if let Ok(id) = bahagian[1].parse::<u32>() {
                    senarai.padam(id);
                }
            }
            "papar"          => senarai.papar_semua(),
            "selesai_sahaja" => senarai.tapis(true),
            "belum_selesai"  => senarai.tapis(false),
            "keluar"         => {
                println!("Terima kasih! 🦀");
                break;
            }
            _ => println!("Arahan tidak dikenali."),
        }
    }
}
```

---

# 📋 Rujukan Pantas — Vec Cheat Sheet

## Methods Paling Penting

| Method | Kegunaan | Big-O |
|--------|----------|-------|
| `push(v)` | Tambah di hujung | O(1) amortized |
| `pop()` | Buang dari hujung | O(1) |
| `insert(i, v)` | Sisip di index i | O(n) |
| `remove(i)` | Buang di index i | O(n) |
| `swap_remove(i)` | Buang di index i (guna elemen terakhir) | O(1) |
| `get(i)` | Akses selamat | O(1) |
| `len()` | Bilangan elemen | O(1) |
| `is_empty()` | Semak kosong | O(1) |
| `contains(&v)` | Semak kewujudan | O(n) |
| `sort()` | Sort ascending | O(n log n) |
| `binary_search(&v)` | Cari dalam sorted Vec | O(log n) |
| `retain(\|x\| ...)` | Padam berdasarkan syarat | O(n) |
| `dedup()` | Buang consecutive duplikat | O(n) |
| `reverse()` | Terbalikkan in-place | O(n) |
| `extend(iter)` | Tambah dari iterator | O(k) |
| `append(&mut v)` | Alih semua dari Vec lain | O(1) |
| `truncate(n)` | Potong ke panjang n | O(1) |
| `clear()` | Kosongkan semua | O(n) |
| `split_at(i)` | Bahagi kepada dua slice | O(1) |
| `windows(n)` | Sliding window | — |
| `chunks(n)` | Bahagi kepada kumpulan | — |

## Bila Guna Apa

```
Nak tambah/buang hujung sahaja?   → Vec (push/pop)
Nak tambah/buang depan juga?      → VecDeque
Nak buang cepat (urutan ok berubah)? → swap_remove
Nak carian laju?                  → sort() + binary_search()
Nak unik sahaja?                  → HashSet
Nak key-value?                    → HashMap
```

## Iterator Chain Formula

```rust
vec.iter()          // atau .iter_mut() / .into_iter()
   .filter(|x| ...) // tapis
   .map(|x| ...)    // transform
   .take(n)         // hadkan bilangan
   .collect()        // kumpul semula
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **Matrix Kalkulator** — tambah, darab dua `Vec<Vec<f64>>`
2. **Sliding Window Maximum** — cari nilai max dalam setiap window saiz k
3. **Run-Length Encoding** — `[1,1,2,3,3,3]` → `[(1,2),(2,1),(3,3)]`
4. **Merge Sort** — implementasi merge sort menggunakan Vec
5. **Top-N Filter** — terima Vec<Pekerja>, return N pekerja gaji tertinggi

---

*Vec in Rust — dari `vec![1,2,3]` hingga iterator chains lanjutan. Selamat belajar!* 🦀
