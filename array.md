# 📐 Arrays dalam Rust — Beginner to Advanced

> Panduan lengkap Arrays [T; N]: dari konsep asas hingga pattern lanjutan.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu Array?

```
Array [i32; 5] — saiz TETAP, type SAMA, disimpan bersebelahan:

Stack:
┌────┬────┬────┬────┬────┐
│ 10 │ 20 │ 30 │ 40 │ 50 │
└────┴────┴────┴────┴────┘
  [0]  [1]  [2]  [3]  [4]

Saiz: 5 × 4 bytes = 20 bytes (i32 = 4 bytes)
Semua dalam stack — TIADA heap allocation!
```

Array dalam Rust adalah **fixed-size collection** — saiznya **mesti diketahui semasa compile**. Ini berbeza dengan `Vec<T>` yang boleh tumbuh dan mengecil masa runtime.

---

## Array vs Vec — Bila Guna Yang Mana?

```
┌─────────────────────┬──────────────────┬───────────────────┐
│  Ciri               │  Array [T; N]    │  Vec<T>           │
├─────────────────────┼──────────────────┼───────────────────┤
│  Saiz               │  Tetap (compile) │  Dinamik (runtime)│
│  Memory             │  Stack           │  Heap             │
│  Allocation         │  Tiada!          │  Ada              │
│  Laju               │  Sangat laju     │  Laju             │
│  Boleh push/pop?    │  ✘               │  ✔                │
│  Saiz tahu awal?    │  ✔ Wajib         │  Tak perlu        │
│  Iterate            │  ✔               │  ✔                │
│  Slice              │  ✔               │  ✔                │
└─────────────────────┴──────────────────┴───────────────────┘

Guna Array bila:
  ✔ Saiz tetap, tahu semasa tulis kod
  ✔ Nak elak heap allocation (performance kritikal)
  ✔ Data kecil & tetap: hari dalam seminggu, warna RGB, grid 3×3
  ✔ Embedded / no-std environment

Guna Vec bila:
  ✔ Saiz tidak tahu semasa compile
  ✔ Perlu push/pop/resize
```

---

## Peta Pembelajaran

```
Bab 1  → Buat Array
Bab 2  → Akses & Kemaskini
Bab 3  → Iterate
Bab 4  → Slice — Jambatan Array & Vec
Bab 5  → Array Multidimensi
Bab 6  → Sort & Cari
Bab 7  → Array sebagai Function Parameter
Bab 8  → Const & Static Arrays
Bab 9  → Pattern Lanjutan
Bab 10 → Mini Project: Papan Catur Mini
```

---

# BAB 1: Buat Array 🏗️

## Semua Cara Buat Array

```rust
fn main() {
    // Cara 1: literal — type diinfer
    let a = [1, 2, 3, 4, 5];             // [i32; 5]
    let b = [1.0, 2.0, 3.0];             // [f64; 3]
    let c = [true, false, true, false];   // [bool; 4]
    let d = ["epal", "mangga", "pisang"]; // [&str; 3]

    // Cara 2: explicit type annotation
    let e: [u8; 4] = [10, 20, 30, 40];

    // Cara 3: nilai sama berulang — [nilai; bilangan]
    let zeros  = [0; 5];              // [0, 0, 0, 0, 0]
    let penuh  = [255_u8; 4];         // [255, 255, 255, 255]
    let kosong = [' '; 10];           // [' ', ' ', ..., ' ']
    let aktif  = [false; 8];          // [false; 8] — bit flags!

    println!("{:?}", zeros);   // [0, 0, 0, 0, 0]
    println!("{:?}", penuh);   // [255, 255, 255, 255]

    // Cara 4: dari range dengan collect (jadi Vec dulu)
    // Array tak boleh collect terus — perlu tahu saiz!
    let v: Vec<i32> = (1..=5).collect();
    let arr: [i32; 5] = v.try_into().unwrap();
    println!("{:?}", arr); // [1, 2, 3, 4, 5]

    // Cara 5: std::array::from_fn — buat array dari fungsi
    let kuasa_dua: [i32; 6] = std::array::from_fn(|i| (i * i) as i32);
    println!("{:?}", kuasa_dua); // [0, 1, 4, 9, 16, 25]

    let indeks: [usize; 5] = std::array::from_fn(|i| i);
    println!("{:?}", indeks); // [0, 1, 2, 3, 4]
}
```

---

## Saiz Array Adalah Sebahagian Dari Type!

```rust
fn main() {
    let a: [i32; 3] = [1, 2, 3];
    let b: [i32; 5] = [1, 2, 3, 4, 5];

    // a dan b ada TYPE BERBEZA walaupun sama-sama [i32; _]
    // Tidak boleh buat fungsi yang terima kedua-dua tanpa generik atau slice

    println!("Saiz a: {}", a.len()); // 3
    println!("Saiz b: {}", b.len()); // 5

    // Type checking pada compile time:
    // let c: [i32; 3] = [1, 2, 3, 4]; // ERROR! expected 3, found 4

    // Saiz array sebagai const generic
    fn papar<const N: usize>(arr: [i32; N]) {
        println!("Array saiz {}: {:?}", N, arr);
    }

    papar([1, 2, 3]);       // Array saiz 3: [1, 2, 3]
    papar([10, 20, 30, 40]); // Array saiz 4: [10, 20, 30, 40]
}
```

---

## 🧠 Brain Teaser #1

Kod manakah yang akan compile?

```rust
// A
let a: [i32; 3] = [1, 2, 3, 4];

// B
let b = [0; 5];
let c: [i32; 5] = b;

// C
let d: [i32; 3] = [42; 3];

// D
let e: [u8; 3] = [1, 2, 256];
```

<details>
<summary>👀 Jawapan</summary>

- **A** — ❌ ERROR: `[1,2,3,4]` ada 4 elemen tapi type kata 3.
- **B** — ✔ OK: `b` adalah `[i32; 5]`, sama type dengan `c`.
- **C** — ✔ OK: `[42; 3]` = `[42, 42, 42]` — tiga elemen, type `[i32; 3]`.
- **D** — ❌ ERROR: `256` melebihi `u8::MAX` (255). Compile error!
</details>

---

# BAB 2: Akses & Kemaskini ✏️

## Akses Elemen

```rust
fn main() {
    let warna = ["merah", "hijau", "biru"];

    // Cara 1: index [] — PANIC kalau out of bounds
    println!("{}", warna[0]); // "merah"
    println!("{}", warna[2]); // "biru"
    // println!("{}", warna[3]); // ← PANIC! index out of bounds

    // Cara 2: get() — return Option<&T>, SELAMAT
    match warna.get(1) {
        Some(w) => println!("Warna 1: {}", w),
        None    => println!("Tiada"),
    }
    println!("{:?}", warna.get(10)); // None — tiada panic!

    // Cara 3: first() dan last()
    println!("Pertama: {:?}", warna.first()); // Some("merah")
    println!("Terakhir: {:?}", warna.last());  // Some("biru")

    // Akses dua elemen serentak (mutable) — split_at_mut
    let mut data = [1, 2, 3, 4, 5];
    let (kiri, kanan) = data.split_at_mut(2);
    kiri[0]  = 10;
    kanan[0] = 30;
    println!("{:?}", data); // [10, 2, 30, 4, 5]
}
```

---

## Kemaskini Array

```rust
fn main() {
    // Array mesti `mut` untuk kemaskini
    let mut suhu = [28.5_f64, 30.1, 27.8, 29.3, 31.0];

    // Kemaskini satu elemen
    suhu[2] = 28.0;
    println!("{:?}", suhu);

    // Kemaskini semua elemen
    for s in &mut suhu {
        *s += 1.0; // naikkan semua 1 darjah
    }
    println!("{:?}", suhu);

    // Swap dua elemen
    suhu.swap(0, 4);
    println!("Selepas swap 0 & 4: {:?}", suhu);

    // Fill semua dengan nilai sama
    suhu.fill(25.0);
    println!("Selepas fill: {:?}", suhu); // semua 25.0

    // fill_with — isi dengan fungsi
    let mut counter = [0i32; 5];
    let mut n = 0;
    counter.fill_with(|| { n += 1; n });
    println!("fill_with: {:?}", counter); // [1, 2, 3, 4, 5]
}
```

---

## Saiz & Semak Kosong

```rust
fn main() {
    let a = [1, 2, 3, 4, 5];
    let b: [i32; 0] = [];

    println!("a.len(): {}", a.len());       // 5
    println!("b.len(): {}", b.len());       // 0
    println!("a kosong? {}", a.is_empty()); // false
    println!("b kosong? {}", b.is_empty()); // true

    // contains
    println!("Ada 3? {}", a.contains(&3));  // true
    println!("Ada 9? {}", a.contains(&9));  // false
}
```

---

# BAB 3: Iterate 🔄

## Semua Cara Iterate

```rust
fn main() {
    let buah = ["epal", "mangga", "pisang", "durian"];

    // Cara 1: for loop dengan borrow — paling biasa
    for b in &buah {
        print!("{} ", b); // b adalah &&str
    }
    println!();

    // Cara 2: for loop consume — ambil ownership
    let nombor = [10, 20, 30];
    for n in nombor {     // Copy types boleh iterate terus
        print!("{} ", n); // n adalah i32
    }
    println!();

    // Cara 3: iter() — explicit iterator
    for b in buah.iter() {
        print!("{} ", b);
    }
    println!();

    // Cara 4: iter_mut() — mutable iterate
    let mut nilai = [1, 2, 3, 4, 5];
    for v in nilai.iter_mut() {
        *v *= 2;
    }
    println!("Selepas *2: {:?}", nilai); // [2, 4, 6, 8, 10]

    // Cara 5: enumerate — dengan index
    let hari = ["Isnin", "Selasa", "Rabu", "Khamis", "Jumaat"];
    for (i, h) in hari.iter().enumerate() {
        println!("  Hari {}: {}", i + 1, h);
    }

    // Cara 6: rev — terbalik
    for b in buah.iter().rev() {
        print!("{} ", b); // durian pisang mangga epal
    }
    println!();

    // Cara 7: windows — sliding window
    let data = [1, 2, 3, 4, 5];
    for w in data.windows(3) {
        println!("  window: {:?}", w); // [1,2,3], [2,3,4], [3,4,5]
    }

    // Cara 8: chunks — kumpulan saiz n
    for c in data.chunks(2) {
        println!("  chunk: {:?}", c); // [1,2], [3,4], [5]
    }
}
```

---

## Iterator Methods pada Array

```rust
fn main() {
    let markah = [85, 92, 78, 95, 60, 88, 73];

    // map + collect — transform kepada Vec
    let bergred: Vec<&str> = markah.iter()
        .map(|&m| if m >= 90 { "A" } else if m >= 75 { "B" } else { "C" })
        .collect();
    println!("Gred: {:?}", bergred);

    // filter
    let lulus: Vec<&i32> = markah.iter().filter(|&&m| m >= 75).collect();
    println!("Lulus: {:?}", lulus);

    // sum, product
    let jumlah: i32 = markah.iter().sum();
    println!("Jumlah: {}", jumlah);

    // min, max
    println!("Min: {:?}", markah.iter().min()); // Some(60)
    println!("Max: {:?}", markah.iter().max()); // Some(95)

    // any, all
    println!("Ada >= 90? {}", markah.iter().any(|&m| m >= 90));
    println!("Semua > 50? {}", markah.iter().all(|&m| m > 50));

    // fold
    let purata = markah.iter().fold(0, |acc, &m| acc + m) / markah.len() as i32;
    println!("Purata: {}", purata);

    // position
    let pos = markah.iter().position(|&m| m == 95);
    println!("95 ada di index: {:?}", pos); // Some(3)
}
```

---

## 🧠 Brain Teaser #2

Apa output kod ini?

```rust
fn main() {
    let a = [1, 2, 3, 4, 5];
    let hasil: i32 = a.iter()
        .filter(|&&x| x % 2 != 0)
        .map(|&x| x * x)
        .sum();
    println!("{}", hasil);
}
```

<details>
<summary>👀 Jawapan</summary>

Output: `35`

1. `filter` — elemen ganjil: `[1, 3, 5]`
2. `map` — kuasa dua: `[1, 9, 25]`
3. `sum` — 1 + 9 + 25 = **35**
</details>

---

# BAB 4: Slice — Jambatan Array & Vec 🌉

## Apa Itu Slice?

```
Array  [i32; 5]:  [10][20][30][40][50]
                    ↑               ↑
                    0               4

Slice &[i32]:     [20][30][40]
                    ↑       ↑
                    1       3  (pandangan sahaja, tiada copy!)
```

Slice `&[T]` adalah **pandangan** kepada sekumpulan data bersebelahan. Ia boleh point ke:
- Array: `&arr[1..4]`
- Vec: `&vec[1..4]`
- Slice lain: `&slice[0..2]`

```rust
fn jumlah(data: &[i32]) -> i32 {
    data.iter().sum()
}

fn main() {
    let arr = [10, 20, 30, 40, 50];
    let vec = vec![10, 20, 30, 40, 50];

    // Fungsi yang terima &[T] boleh guna dengan array DAN Vec!
    println!("Array: {}", jumlah(&arr));       // 150
    println!("Vec:   {}", jumlah(&vec));       // 150
    println!("Slice: {}", jumlah(&arr[1..4])); // 90

    // Semua jenis slice dari array
    let semua  = &arr[..];   // semua elemen
    let tiga   = &arr[1..4]; // index 1,2,3
    let awal   = &arr[..3];  // index 0,1,2
    let akhir  = &arr[2..];  // index 2 hingga akhir
    let inklusif = &arr[1..=3]; // index 1,2,3

    println!("{:?}", tiga);     // [20, 30, 40]
    println!("{:?}", awal);     // [10, 20, 30]
    println!("{:?}", akhir);    // [30, 40, 50]
    println!("{:?}", inklusif); // [20, 30, 40]
}
```

---

## Mutable Slice

```rust
fn gandakan(data: &mut [i32]) {
    for x in data {
        *x *= 2;
    }
}

fn main() {
    let mut arr = [1, 2, 3, 4, 5];

    gandakan(&mut arr[1..4]); // gandakan hanya index 1,2,3
    println!("{:?}", arr);    // [1, 4, 6, 8, 5]

    gandakan(&mut arr);       // gandakan semua
    println!("{:?}", arr);    // [2, 8, 12, 16, 10]
}
```

---

## Peraturan &[T] vs &Vec<T> vs &[T; N]

```rust
// Hierarki coercion:
// &[T; N]  →  dapat coerce ke  →  &[T]
// &Vec<T>  →  dapat coerce ke  →  &[T]

fn proses(data: &[i32]) {  // terima semua!
    println!("{:?}", data);
}

fn main() {
    let arr: [i32; 3]  = [1, 2, 3];
    let vec: Vec<i32>  = vec![4, 5, 6];
    let sli: &[i32]    = &[7, 8, 9];

    proses(&arr); // [i32; 3] → &[i32] ✔
    proses(&vec); // Vec<i32> → &[i32] ✔
    proses(sli);  // &[i32]           ✔
    proses(&arr[1..]);  // slice of arr ✔
}
```

> 💡 **Peraturan Emas:** Parameter fungsi yang baca sahaja → guna `&[T]` bukan `&Vec<T>` atau `&[T; N]`. Lebih fleksibel!

---

# BAB 5: Array Multidimensi 🔢

## 2D Array (Matrix)

```rust
fn main() {
    // 2D array: [baris][lajur]
    let matrix: [[i32; 3]; 3] = [
        [1, 2, 3],
        [4, 5, 6],
        [7, 8, 9],
    ];

    // Akses: matrix[baris][lajur]
    println!("Tengah: {}", matrix[1][1]); // 5
    println!("Baris 0: {:?}", matrix[0]); // [1, 2, 3]

    // Iterate 2D
    for (i, baris) in matrix.iter().enumerate() {
        for (j, &val) in baris.iter().enumerate() {
            print!("{:3}", val);
        }
        println!();
    }

    // Transpose matrix
    let mut transpos = [[0i32; 3]; 3];
    for i in 0..3 {
        for j in 0..3 {
            transpos[j][i] = matrix[i][j];
        }
    }
    println!("Transpos:");
    for baris in &transpos {
        println!("  {:?}", baris);
    }
}
```

---

## 2D Array — Operasi Matematik

```rust
fn tambah_matrix(a: &[[i32; 3]; 3], b: &[[i32; 3]; 3]) -> [[i32; 3]; 3] {
    std::array::from_fn(|i| std::array::from_fn(|j| a[i][j] + b[i][j]))
}

fn darab_matrix(a: &[[i32; 3]; 3], b: &[[i32; 3]; 3]) -> [[i32; 3]; 3] {
    let mut hasil = [[0i32; 3]; 3];
    for i in 0..3 {
        for j in 0..3 {
            for k in 0..3 {
                hasil[i][j] += a[i][k] * b[k][j];
            }
        }
    }
    hasil
}

fn main() {
    let a = [[1, 2, 3], [4, 5, 6], [7, 8, 9]];
    let b = [[9, 8, 7], [6, 5, 4], [3, 2, 1]];

    let tambah = tambah_matrix(&a, &b);
    println!("A + B:");
    for r in &tambah { println!("  {:?}", r); }

    let darab = darab_matrix(&a, &b);
    println!("A × B:");
    for r in &darab { println!("  {:?}", r); }
}
```

---

## 3D Array

```rust
fn main() {
    // 3D: [lapisan][baris][lajur]
    let kubus: [[[u8; 2]; 2]; 2] = [
        [[1, 2], [3, 4]],   // lapisan 0
        [[5, 6], [7, 8]],   // lapisan 1
    ];

    println!("Lapisan 1, Baris 0, Lajur 1: {}", kubus[1][0][1]); // 6

    // Iterate 3D
    for (l, lapisan) in kubus.iter().enumerate() {
        for (b, baris) in lapisan.iter().enumerate() {
            for (k, &val) in baris.iter().enumerate() {
                print!("[{},{},{}]={} ", l, b, k, val);
            }
        }
        println!();
    }
}
```

---

## 🧠 Brain Teaser #3

Tuliskan fungsi yang check sama ada matrix 3×3 adalah **symmetric** (sama dengan transpose-nya).

```rust
fn adalah_simetri(m: &[[i32; 3]; 3]) -> bool {
    // tulis kod di sini
}
```

<details>
<summary>👀 Jawapan</summary>

```rust
fn adalah_simetri(m: &[[i32; 3]; 3]) -> bool {
    for i in 0..3 {
        for j in 0..3 {
            if m[i][j] != m[j][i] {
                return false;
            }
        }
    }
    true
}

fn main() {
    let simetri = [[1, 2, 3], [2, 5, 6], [3, 6, 9]];
    let bukan   = [[1, 2, 3], [4, 5, 6], [7, 8, 9]];

    println!("Simetri? {}", adalah_simetri(&simetri)); // true
    println!("Bukan?   {}", adalah_simetri(&bukan));   // false
}
```

Matrix simetri: `m[i][j] == m[j][i]` untuk semua i, j.
</details>

---

# BAB 6: Sort & Cari 🔢

## Sort Array

```rust
fn main() {
    // Sort integer
    let mut v = [5, 2, 8, 1, 9, 3, 7, 4, 6];
    v.sort();
    println!("Ascending:  {:?}", v); // [1,2,3,4,5,6,7,8,9]

    v.sort_by(|a, b| b.cmp(a)); // descending
    println!("Descending: {:?}", v);

    // Sort float — guna partial_cmp
    let mut f = [3.14, 1.41, 2.72, 0.57];
    f.sort_by(|a, b| a.partial_cmp(b).unwrap());
    println!("Float sort: {:?}", f);

    // sort_by_key
    let mut kata = ["pisang", "epal", "durian", "mangga"];
    kata.sort_by_key(|k| k.len()); // sort by panjang
    println!("Sort by len: {:?}", kata);

    // sort_unstable — laju, tapi tidak stable
    let mut u = [3, 1, 4, 1, 5, 9, 2, 6];
    u.sort_unstable();
    println!("sort_unstable: {:?}", u);

    // Semak sama ada sorted
    println!("Sorted? {}", u.windows(2).all(|w| w[0] <= w[1])); // true
}
```

---

## Binary Search

```rust
fn main() {
    let mut arr = [10, 20, 30, 40, 50, 60, 70];
    // Wajib sorted dulu!

    match arr.binary_search(&40) {
        Ok(i)  => println!("Jumpa 40 di index {}", i),  // Ok(3)
        Err(i) => println!("Tiada 40, patut di {}", i),
    }

    match arr.binary_search(&45) {
        Ok(i)  => println!("Jumpa 45 di index {}", i),
        Err(i) => println!("Tiada 45, insertion point: {}", i), // Err(4)
    }

    // binary_search_by — custom comparator
    let data = ["amin", "ali", "siti", "zara"];
    let mut sorted_data = data;
    sorted_data.sort();

    match sorted_data.binary_search_by(|s| s.cmp(&"siti")) {
        Ok(i)  => println!("Jumpa 'siti' di index {}", i),
        Err(i) => println!("Tiada 'siti'"),
    }

    // binary_search_by_key — search by field
    #[derive(Debug)]
    struct Item { id: u32, nama: &'static str }
    let items = [
        Item { id: 1, nama: "Epal" },
        Item { id: 3, nama: "Mangga" },
        Item { id: 5, nama: "Pisang" },
    ];

    match items.binary_search_by_key(&3, |item| item.id) {
        Ok(i)  => println!("Jumpa id=3: {}", items[i].nama), // Mangga
        Err(_) => println!("Tiada"),
    }
}
```

---

# BAB 7: Array sebagai Function Parameter 🔧

## Passing Arrays

```rust
// ❌ Kurang fleksibel — hanya terima [i32; 5] sahaja
fn jumlah_5(arr: [i32; 5]) -> i32 {
    arr.iter().sum()
}

// ✔ Lebih fleksibel — terima array apa-apa saiz
fn jumlah_generic<const N: usize>(arr: [i32; N]) -> i32 {
    arr.iter().sum()
}

// ✔✔ Paling fleksibel — terima array, Vec, slice apa-apa
fn jumlah_slice(arr: &[i32]) -> i32 {
    arr.iter().sum()
}

fn main() {
    let a3 = [1, 2, 3];
    let a5 = [1, 2, 3, 4, 5];

    // jumlah_5(&a3); // ← ERROR! salah saiz
    println!("{}", jumlah_generic(a3));    // 6
    println!("{}", jumlah_generic(a5));    // 15
    println!("{}", jumlah_slice(&a3));     // 6
    println!("{}", jumlah_slice(&a5));     // 15
    println!("{}", jumlah_slice(&a5[2..])); // 12
}
```

---

## Return Array dari Fungsi

```rust
// Return array — saiz mesti diketahui
fn buat_grid(nilai: i32) -> [[i32; 3]; 3] {
    [[nilai; 3]; 3]
}

// Return array dengan const generic
fn buat_array<const N: usize>(mula: i32) -> [i32; N] {
    std::array::from_fn(|i| mula + i as i32)
}

// Return Vec kalau saiz tidak tetap
fn buat_vec_dinamik(n: usize) -> Vec<i32> {
    (0..n as i32).collect()
}

fn main() {
    let grid = buat_grid(0);
    for baris in &grid {
        println!("{:?}", baris);
    }

    let arr5: [i32; 5] = buat_array(10);
    println!("{:?}", arr5); // [10, 11, 12, 13, 14]

    let arr3: [i32; 3] = buat_array(1);
    println!("{:?}", arr3); // [1, 2, 3]
}
```

---

## Const Generic — Fungsi Universal

```rust
// Fungsi yang berfungsi untuk apa-apa saiz array
fn cari_maks<T: PartialOrd, const N: usize>(arr: &[T; N]) -> Option<&T> {
    arr.iter().reduce(|a, b| if a > b { a } else { b })
}

fn purata<const N: usize>(arr: &[f64; N]) -> f64 {
    arr.iter().sum::<f64>() / N as f64
}

fn zip_tambah<const N: usize>(a: [i32; N], b: [i32; N]) -> [i32; N] {
    std::array::from_fn(|i| a[i] + b[i])
}

fn main() {
    println!("{:?}", cari_maks(&[3, 1, 4, 1, 5]));     // Some(5)
    println!("{:?}", cari_maks(&["epal", "zara", "ali"])); // Some("zara")

    println!("{:.2}", purata(&[85.0, 92.0, 78.0]));     // 85.00
    println!("{:.2}", purata(&[1.0, 2.0, 3.0, 4.0]));   // 2.50

    println!("{:?}", zip_tambah([1, 2, 3], [10, 20, 30])); // [11, 22, 33]
}
```

---

# BAB 8: Const & Static Arrays 🔒

## const — Array Compile Time

```rust
// const array — dikira semasa compile, boleh guna di mana-mana
const HARI_SEMINGGU: [&str; 7] = [
    "Ahad", "Isnin", "Selasa", "Rabu",
    "Khamis", "Jumaat", "Sabtu",
];

const BULAN: [&str; 12] = [
    "Januari", "Februari", "Mac", "April",
    "Mei", "Jun", "Julai", "Ogos",
    "September", "Oktober", "November", "Disember",
];

const FIBONACCI: [u64; 10] = [0, 1, 1, 2, 3, 5, 8, 13, 21, 34];

const PI_DIGIT: [u8; 10] = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3];

fn main() {
    println!("Hari ke-3: {}", HARI_SEMINGGU[3]); // Rabu
    println!("Bulan ke-6: {}", BULAN[5]);          // Jun
    println!("Fibonacci: {:?}", FIBONACCI);
    println!("Pi: 3.{}{}{}{}{}{}{}{}{}", 
        PI_DIGIT[1], PI_DIGIT[2], PI_DIGIT[3], PI_DIGIT[4],
        PI_DIGIT[5], PI_DIGIT[6], PI_DIGIT[7], PI_DIGIT[8]);
}
```

---

## static — Array Global

```rust
// static — hidup sepanjang program, ada address tetap dalam memory
static WARNA_HTML: [(&str, &str); 5] = [
    ("merah",  "#FF0000"),
    ("hijau",  "#00FF00"),
    ("biru",   "#0000FF"),
    ("putih",  "#FFFFFF"),
    ("hitam",  "#000000"),
];

static LOOKUP_TABLE: [u32; 256] = {
    // Guna const block untuk isi table
    let mut t = [0u32; 256];
    // Nota: loop dalam const context mempunyai had
    // Untuk complex init, guna lazy_static atau once_cell
    t
};

fn cari_warna(nama: &str) -> Option<&'static str> {
    WARNA_HTML.iter()
        .find(|(n, _)| *n == nama)
        .map(|(_, hex)| *hex)
}

fn main() {
    println!("Merah: {:?}", cari_warna("merah")); // Some("#FF0000")
    println!("Ungu:  {:?}", cari_warna("ungu"));  // None

    for (nama, hex) in &WARNA_HTML {
        println!("  {} = {}", nama, hex);
    }
}
```

---

## 🧠 Brain Teaser #4

Apakah beza `const`, `static`, dan `let` untuk array?

<details>
<summary>👀 Jawapan</summary>

| | `const` | `static` | `let` |
|--|--|--|--|
| **Scope** | Global / local | Global | Local (function) |
| **Lifetime** | Compile-time | Program lifetime | Scope |
| **Address** | Mungkin inline | Address tetap | Stack |
| **Mutable** | ❌ Tidak boleh | `static mut` (unsafe) | `let mut` ✔ |
| **Type** | Wajib anotasi | Wajib anotasi | Boleh infer |
| **Guna bila** | Nilai compile-time | Global shared data | Data biasa |

```rust
const  MAX_SIZE: usize = 100;          // compile-time constant
static BUFFER: [u8; 100] = [0; 100];  // global, address tetap
let    arr = [1, 2, 3];               // local, stack
```

`const` biasanya di-**inline** oleh compiler — tiada address yang dijamin.
`static` ada **address tetap** — boleh ambil reference `&BUFFER`.
</details>

---

# BAB 9: Pattern Lanjutan 🎯

## Array sebagai Bit Flags

```rust
fn main() {
    // Guna array bool sebagai bit flags
    const BACA:    usize = 0;
    const TULIS:   usize = 1;
    const LAKSANA: usize = 2;

    let mut kebenaran = [false; 3];
    kebenaran[BACA]    = true;
    kebenaran[TULIS]   = true;
    kebenaran[LAKSANA] = false;

    println!("Boleh baca?    {}", kebenaran[BACA]);
    println!("Boleh tulis?   {}", kebenaran[TULIS]);
    println!("Boleh laksana? {}", kebenaran[LAKSANA]);

    // Semak semua
    let semua_dibenar = kebenaran.iter().all(|&k| k);
    println!("Semua dibenar? {}", semua_dibenar);
}
```

---

## Ring Buffer (Circular Array)

```rust
struct RingBuffer<T, const N: usize> {
    data:  [Option<T>; N],
    kepala: usize,   // mula baca
    ekor:   usize,   // mula tulis
    penuh:  bool,
}

impl<T: Copy + std::fmt::Debug, const N: usize> RingBuffer<T, N> {
    fn baru() -> Self {
        RingBuffer {
            data:   [None; N],
            kepala: 0,
            ekor:   0,
            penuh:  false,
        }
    }

    fn tulis(&mut self, val: T) -> bool {
        if self.penuh { return false; } // penuh!
        self.data[self.ekor] = Some(val);
        self.ekor = (self.ekor + 1) % N;
        self.penuh = self.ekor == self.kepala;
        true
    }

    fn baca(&mut self) -> Option<T> {
        if !self.penuh && self.kepala == self.ekor {
            return None; // kosong
        }
        let val = self.data[self.kepala].take();
        self.kepala = (self.kepala + 1) % N;
        self.penuh = false;
        val
    }

    fn is_empty(&self) -> bool {
        !self.penuh && self.kepala == self.ekor
    }

    fn is_full(&self) -> bool { self.penuh }
}

fn main() {
    let mut rb: RingBuffer<i32, 4> = RingBuffer::baru();

    println!("Tulis 10: {}", rb.tulis(10)); // true
    println!("Tulis 20: {}", rb.tulis(20)); // true
    println!("Tulis 30: {}", rb.tulis(30)); // true
    println!("Tulis 40: {}", rb.tulis(40)); // true
    println!("Tulis 50: {}", rb.tulis(50)); // false — penuh!

    println!("Baca: {:?}", rb.baca()); // Some(10)
    println!("Baca: {:?}", rb.baca()); // Some(20)

    println!("Tulis 50 semula: {}", rb.tulis(50)); // true — ada ruang
    println!("Baca: {:?}", rb.baca()); // Some(30)
    println!("Baca: {:?}", rb.baca()); // Some(40)
    println!("Baca: {:?}", rb.baca()); // Some(50)
    println!("Baca kosong: {:?}", rb.baca()); // None
}
```

---

## Lookup Table — Optimasi Dengan Array

```rust
// Guna array sebagai lookup table untuk operasi mahal
// Contoh: precompute semua kuasa dua 0-255

const KUASA_DUA: [u32; 256] = {
    let mut t = [0u32; 256];
    let mut i = 0usize;
    while i < 256 {
        t[i] = (i * i) as u32;
        i += 1;
    }
    t
};

// Precompute uppercase ASCII
const UPPER_MAP: [u8; 128] = {
    let mut t = [0u8; 128];
    let mut i = 0usize;
    while i < 128 {
        t[i] = if i >= b'a' as usize && i <= b'z' as usize {
            (i - 32) as u8
        } else {
            i as u8
        };
        i += 1;
    }
    t
};

fn uppercase_laju(s: &str) -> String {
    s.chars()
        .map(|c| {
            let code = c as usize;
            if code < 128 { UPPER_MAP[code] as char } else { c }
        })
        .collect()
}

fn main() {
    // O(1) lookup berbanding O(n) computation
    for i in [5usize, 12, 100, 255] {
        println!("{}² = {}", i, KUASA_DUA[i]);
    }

    println!("{}", uppercase_laju("hello, rust!")); // HELLO, RUST!
}
```

---

## Flatten & Unflatten 2D

```rust
fn flatten<const R: usize, const C: usize>(
    matrix: [[i32; C]; R]
) -> Vec<i32> {
    matrix.iter().flatten().copied().collect()
}

fn unflatten<const R: usize, const C: usize>(
    v: &[i32]
) -> [[i32; C]; R] {
    std::array::from_fn(|i| {
        std::array::from_fn(|j| v[i * C + j])
    })
}

fn main() {
    let matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]];
    let flat = flatten(matrix);
    println!("Flat: {:?}", flat); // [1,2,3,4,5,6,7,8,9]

    let kembali: [[i32; 3]; 3] = unflatten(&flat);
    println!("Kembali 2D:");
    for baris in &kembali {
        println!("  {:?}", baris);
    }
}
```

---

## 🧠 Brain Teaser #5 (Advanced)

Bina fungsi `rotate_90` yang pusingkan matrix 3×3 sebanyak 90 darjah ke kanan.

```
Input:          Output:
1  2  3         7  4  1
4  5  6    →    8  5  2
7  8  9         9  6  3
```

<details>
<summary>👀 Jawapan</summary>

```rust
fn rotate_90(m: &[[i32; 3]; 3]) -> [[i32; 3]; 3] {
    std::array::from_fn(|i| {
        std::array::from_fn(|j| m[2 - j][i])
    })
}

fn main() {
    let m = [[1, 2, 3], [4, 5, 6], [7, 8, 9]];

    println!("Asal:");
    for b in &m { println!("  {:?}", b); }

    let r = rotate_90(&m);
    println!("Rotate 90°:");
    for b in &r { println!("  {:?}", b); }
    // [7, 4, 1]
    // [8, 5, 2]
    // [9, 6, 3]
}
```

Formula: `result[i][j] = original[n-1-j][i]`
di mana n = saiz matrix (3 untuk 3×3).
</details>

---

# BAB 10: Mini Project — Papan Catur Mini ♟️

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum Keping {
    Kosong,
    Pion(Warna),
    Ratu(Warna),
    Raja(Warna),
}

#[derive(Debug, Clone, Copy, PartialEq)]
enum Warna { Putih, Hitam }

impl Keping {
    fn simbol(&self) -> &str {
        match self {
            Keping::Kosong       => "·",
            Keping::Pion(Warna::Putih)  => "♙",
            Keping::Pion(Warna::Hitam)  => "♟",
            Keping::Ratu(Warna::Putih)  => "♕",
            Keping::Ratu(Warna::Hitam)  => "♛",
            Keping::Raja(Warna::Putih)  => "♔",
            Keping::Raja(Warna::Hitam)  => "♚",
        }
    }
}

struct PapanMini {
    grid: [[Keping; 5]; 5], // papan 5×5
}

impl PapanMini {
    fn baru() -> Self {
        use Keping::*;
        use Warna::*;
        PapanMini {
            grid: [
                [Ratu(Hitam),  Pion(Hitam), Kosong, Pion(Hitam), Raja(Hitam)],
                [Pion(Hitam),  Pion(Hitam), Kosong, Pion(Hitam), Pion(Hitam)],
                [Kosong,       Kosong,      Kosong, Kosong,      Kosong     ],
                [Pion(Putih),  Pion(Putih), Kosong, Pion(Putih), Pion(Putih)],
                [Raja(Putih),  Pion(Putih), Kosong, Pion(Putih), Ratu(Putih)],
            ],
        }
    }

    fn papar(&self) {
        println!("  a b c d e");
        println!("  ---------");
        for (i, baris) in self.grid.iter().enumerate() {
            print!("{} ", 5 - i);
            for keping in baris {
                print!("{} ", keping.simbol());
            }
            println!("| {}", 5 - i);
        }
        println!("  ---------");
        println!("  a b c d e");
    }

    fn gerak(&mut self, dari: (usize, usize), ke: (usize, usize)) -> bool {
        let (br, lj) = dari;
        let (kr, kl) = ke;

        if br >= 5 || lj >= 5 || kr >= 5 || kl >= 5 {
            println!("❌ Koordinat luar papan!");
            return false;
        }

        if self.grid[br][lj] == Keping::Kosong {
            println!("❌ Tiada keping di {:?}", dari);
            return false;
        }

        let keping = self.grid[br][lj];
        self.grid[kr][kl] = keping;
        self.grid[br][lj] = Keping::Kosong;
        println!("✔ {} bergerak dari {:?} ke {:?}",
            keping.simbol(), dari, ke);
        true
    }

    fn kira_keping(&self) -> (usize, usize) {
        let mut putih = 0;
        let mut hitam = 0;
        for baris in &self.grid {
            for &k in baris {
                match k {
                    Keping::Kosong => {}
                    Keping::Pion(Warna::Putih)
                    | Keping::Ratu(Warna::Putih)
                    | Keping::Raja(Warna::Putih) => putih += 1,
                    _ => hitam += 1,
                }
            }
        }
        (putih, hitam)
    }

    fn cari_keping(&self, cari: Keping) -> Vec<(usize, usize)> {
        let mut lokasi = Vec::new();
        for (i, baris) in self.grid.iter().enumerate() {
            for (j, &k) in baris.iter().enumerate() {
                if k == cari {
                    lokasi.push((i, j));
                }
            }
        }
        lokasi
    }
}

fn main() {
    let mut papan = PapanMini::baru();

    println!("=== Papan Permulaan ===");
    papan.papar();

    let (p, h) = papan.kira_keping();
    println!("\nKeping: Putih={}, Hitam={}", p, h);

    // Beberapa gerakan
    println!("\n=== Gerakan ===");
    papan.gerak((3, 2), (2, 2)); // Pion putih maju
    papan.gerak((1, 2), (2, 2)); // Pion hitam tangkap!
    papan.gerak((0, 0), (2, 0)); // Ratu hitam maju

    println!("\n=== Selepas Gerakan ===");
    papan.papar();

    let (p2, h2) = papan.kira_keping();
    println!("\nKeping: Putih={}, Hitam={}", p2, h2);

    // Cari lokasi raja
    let raja_putih = papan.cari_keping(Keping::Raja(Warna::Putih));
    let raja_hitam = papan.cari_keping(Keping::Raja(Warna::Hitam));
    println!("Raja Putih di: {:?}", raja_putih);
    println!("Raja Hitam di: {:?}", raja_hitam);
}
```

---

# 📋 Rujukan Pantas — Array Cheat Sheet

## Buat Array

```rust
let a = [1, 2, 3];               // literal
let b: [i32; 3] = [1, 2, 3];    // dengan type
let c = [0; 5];                   // [0,0,0,0,0]
let d: [i32; 5] = std::array::from_fn(|i| i as i32); // [0,1,2,3,4]
```

## Methods Penting

| Method | Kegunaan | Big-O |
|--------|----------|-------|
| `arr[i]` | Akses (panic jika out) | O(1) |
| `arr.get(i)` | Akses selamat | O(1) |
| `arr.len()` | Saiz | O(1) |
| `arr.is_empty()` | Semak kosong | O(1) |
| `arr.contains(&v)` | Semak kewujudan | O(n) |
| `arr.iter()` | Iterator | — |
| `arr.iter_mut()` | Iterator mutable | — |
| `arr.sort()` | Sort ascending | O(n log n) |
| `arr.binary_search(&v)` | Cari (perlu sorted) | O(log n) |
| `arr.swap(i, j)` | Tukar dua elemen | O(1) |
| `arr.reverse()` | Terbalik in-place | O(n) |
| `arr.fill(v)` | Isi semua dengan v | O(n) |
| `arr.windows(n)` | Sliding window | — |
| `arr.chunks(n)` | Kumpulan saiz n | — |
| `arr.split_at(i)` | Bahagi dua | O(1) |

## Array vs Slice vs Vec

```
[T; N]    — saiz tetap, compile-time, stack
&[T]      — pandangan (view), tiada ownership
&mut [T]  — pandangan mutable
Vec<T>    — dinamik, heap, boleh grow
```

## Const Generic Pattern

```rust
fn fungsi<const N: usize>(arr: [T; N]) { ... }
fn fungsi_slice(arr: &[T]) { ... }  // lebih fleksibel
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **Conway's Game of Life** — grid 2D dengan `[[bool; 20]; 20]`
2. **Sudoku Validator** — semak grid 9×9 mengikut peraturan Sudoku
3. **Image Filter** — apply blur/sharpen pada pixel array `[[u8; 3]; N]`
4. **Magic Square** — generate magic square 3×3 di mana semua baris/lajur/diagonal = sama
5. **Maze Solver** — BFS/DFS pada grid 2D array untuk cari laluan

---

*Arrays in Rust — dari `[T; N]` hingga const generics dan ring buffer. Selamat belajar!* 🦀
