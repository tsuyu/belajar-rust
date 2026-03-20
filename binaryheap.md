# 🏔️ BinaryHeap dalam Rust — Beginner to Advanced

> Panduan lengkap BinaryHeap<T>: dari konsep asas hingga priority queue lanjutan.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu BinaryHeap?

```
BinaryHeap dalam Rust adalah MAX-HEAP:
Elemen TERBESAR sentiasa berada di atas (root).

Visualisasi sebagai pokok:
              100
            /     \
          85        90
         /  \      /  \
        70   80  60    75
       / \
      50  65

Simpan dalam Vec (level order):
[100, 85, 90, 70, 80, 60, 75, 50, 65]
  0    1   2   3   4   5   6   7   8

Hubungan parent-child:
  Parent(i) = (i-1) / 2
  LeftChild(i)  = 2*i + 1
  RightChild(i) = 2*i + 2
```

BinaryHeap adalah struktur data yang sentiasa pastikan **elemen terbesar boleh diakses dalam O(1)**. Ia sebenarnya disimpan sebagai Vec di belakang tabir!

---

## Bila Guna BinaryHeap?

```
✔ Priority Queue — proses tugas mengikut keutamaan
✔ Scheduling — job dengan priority tertinggi diproses dulu
✔ Dijkstra / A* — carian laluan terpendek
✔ Top-K elements — cari K elemen terbesar dengan cepat
✔ Heap Sort — sort dalam O(n log n)
✔ Merge K sorted lists — gabung senarai dengan cekap

✘ Jangan guna bila:
  - Perlu akses by index
  - Perlu iterate dalam order
  - Perlu cari elemen tertentu O(1)
```

---

## Peta Pembelajaran

```
Bab 1  → Buat & Isi BinaryHeap
Bab 2  → Peek, Pop & Push
Bab 3  → Iterate & Convert
Bab 4  → Min-Heap — Cara Buat
Bab 5  → BinaryHeap dengan Struct
Bab 6  → Custom Ordering dengan Ord
Bab 7  → Priority Queue Aplikasi
Bab 8  → Heap Sort
Bab 9  → Pattern Lanjutan
Bab 10 → Mini Project: Task Scheduler
```

---

# BAB 1: Buat & Isi BinaryHeap 🏗️

## Import & Asas

```rust
use std::collections::BinaryHeap;

fn main() {
    // Cara 1: kosong
    let mut heap: BinaryHeap<i32> = BinaryHeap::new();

    // Cara 2: dengan kapasiti
    let mut heap: BinaryHeap<i32> = BinaryHeap::with_capacity(10);

    // Cara 3: dari Vec — O(n) lebih laju dari push satu-satu!
    let heap: BinaryHeap<i32> = vec![3, 1, 4, 1, 5, 9, 2, 6].into_iter().collect();
    println!("{:?}", heap); // pelbagai order tapi root = 9

    // Cara 4: dari array
    let heap: BinaryHeap<i32> = [10, 30, 20, 50, 40].into_iter().collect();
    println!("Max: {:?}", heap.peek()); // Some(50)
}
```

---

## Push & Sifting

```rust
use std::collections::BinaryHeap;

fn main() {
    let mut heap = BinaryHeap::new();

    // push — O(log n), auto-sift untuk kekal heap property
    heap.push(5);
    println!("Push 5:  max={:?}", heap.peek()); // Some(5)

    heap.push(10);
    println!("Push 10: max={:?}", heap.peek()); // Some(10)

    heap.push(3);
    println!("Push 3:  max={:?}", heap.peek()); // Some(10)

    heap.push(15);
    println!("Push 15: max={:?}", heap.peek()); // Some(15)

    heap.push(7);
    println!("Push 7:  max={:?}", heap.peek()); // Some(15)

    println!("\nSemua elemen (bukan sorted!): {:?}", heap);
    // [15, 10, 3, 5, 7] atau susunan heap lain — bukan sorted!
    println!("Len: {}", heap.len());
    println!("Kapasiti: {}", heap.capacity());
}
```

> ⚠️ **Penting:** BinaryHeap **bukan sorted list**! Hanya dijamin elemen **pertama (max)** sahaja.
> Kalau nak semua dalam order, guna `pop()` berulang kali atau convert ke sorted Vec.

---

## 🧠 Brain Teaser #1

Selepas operasi ini, apakah nilai `heap.peek()`?

```rust
use std::collections::BinaryHeap;

fn main() {
    let mut heap = BinaryHeap::new();
    heap.push(42);
    heap.push(17);
    heap.push(99);
    heap.push(8);
    heap.push(55);
    heap.pop();
    heap.push(100);
    heap.pop();
    println!("{:?}", heap.peek());
}
```

<details>
<summary>👀 Jawapan</summary>

Output: `Some(55)`

1. Push: `{42, 17, 99, 8, 55}` → max = 99
2. `pop()` → buang 99 → `{55, 17, 42, 8}` → max = 55
3. Push 100 → `{100, 17, 42, 8, 55}` → max = 100
4. `pop()` → buang 100 → `{55, 17, 42, 8}` → max = **55**
</details>

---

# BAB 2: Peek, Pop & Saiz 👀

## Semua Operations Asas

```rust
use std::collections::BinaryHeap;

fn main() {
    let mut heap: BinaryHeap<i32> = [30, 10, 50, 20, 40].into_iter().collect();

    // peek — tengok max TANPA buang, O(1)
    println!("Max (peek): {:?}", heap.peek()); // Some(50)
    println!("Len selepas peek: {}", heap.len()); // 5 — tidak berubah!

    // pop — ambil max DAN buangnya, O(log n)
    println!("Pop: {:?}", heap.pop()); // Some(50)
    println!("Pop: {:?}", heap.pop()); // Some(40)
    println!("Pop: {:?}", heap.pop()); // Some(30)
    println!("Len selepas 3x pop: {}", heap.len()); // 2

    // pop hingga kosong
    while let Some(val) = heap.pop() {
        println!("  {}", val); // 20, 10 — dalam descending order!
    }

    // pop dari heap kosong — tiada panic!
    println!("Pop kosong: {:?}", heap.pop()); // None

    // Saiz
    println!("Kosong? {}", heap.is_empty()); // true

    // push selepas kosong — OK
    heap.push(99);
    println!("Selepas push: {:?}", heap.peek()); // Some(99)
}
```

---

## Pop Dalam Order = Heap Sort!

```rust
use std::collections::BinaryHeap;

fn main() {
    let data = vec![5, 3, 8, 1, 9, 2, 7, 4, 6];
    let mut heap: BinaryHeap<i32> = data.into_iter().collect();

    // Pop satu-satu → dapat descending order
    print!("Descending: ");
    while let Some(val) = heap.pop() {
        print!("{} ", val);
    }
    println!(); // 9 8 7 6 5 4 3 2 1

    // Nak ascending? Kumpul dulu, reverse
    let data2 = vec![5, 3, 8, 1, 9, 2, 7, 4, 6];
    let mut heap2: BinaryHeap<i32> = data2.into_iter().collect();
    let mut sorted: Vec<i32> = Vec::new();
    while let Some(val) = heap2.pop() {
        sorted.push(val);
    }
    sorted.reverse();
    println!("Ascending:  {:?}", sorted); // [1, 2, 3, 4, 5, 6, 7, 8, 9]
}
```

---

## peek_mut — Ubah Max In-Place

```rust
use std::collections::BinaryHeap;

fn main() {
    let mut heap = BinaryHeap::new();
    heap.push(5);
    heap.push(10);
    heap.push(3);

    println!("Sebelum: {:?}", heap.peek()); // Some(10)

    // peek_mut — dapat mutable reference ke max
    // Bila PeekMut di-drop, heap re-sift secara automatik
    if let Some(mut top) = heap.peek_mut() {
        *top = 1; // ubah max (10) kepada 1 — heap akan re-sift!
    }

    println!("Selepas ubah: {:?}", heap.peek()); // Some(5) — 5 jadi max baru!
    println!("Heap: {:?}", heap);
}
```

---

# BAB 3: Iterate & Convert 🔄

## Iterate (Tanpa Order Dijamin)

```rust
use std::collections::BinaryHeap;

fn main() {
    let heap: BinaryHeap<i32> = [5, 3, 8, 1, 9, 2].into_iter().collect();

    // Iterate — BUKAN dalam sorted order!
    println!("Iterate (tidak sorted):");
    for val in &heap {
        print!("{} ", val); // 9 5 8 1 3 2 atau susunan heap lain
    }
    println!();

    // Kalau nak sorted, convert ke sorted Vec
    let mut sorted: Vec<i32> = heap.clone().into_sorted_vec();
    println!("into_sorted_vec (ascending): {:?}", sorted); // [1, 2, 3, 5, 8, 9]

    // into_vec — ambil underlying Vec (BUKAN sorted)
    let vec_tidak_sorted: Vec<i32> = heap.into_vec();
    println!("into_vec (tidak sorted): {:?}", vec_tidak_sorted);
}
```

---

## Convert Pelbagai Arah

```rust
use std::collections::BinaryHeap;

fn main() {
    // Vec → BinaryHeap
    let v = vec![3, 1, 4, 1, 5, 9, 2, 6];
    let heap: BinaryHeap<i32> = v.into_iter().collect();
    println!("Max: {:?}", heap.peek()); // Some(9)

    // BinaryHeap → Vec (tidak sorted)
    let v2: Vec<i32> = heap.into_vec();
    println!("Vec: {:?}", v2);

    // BinaryHeap → sorted Vec (ascending)
    let heap2: BinaryHeap<i32> = [5, 2, 8, 1, 9].into_iter().collect();
    let sorted = heap2.into_sorted_vec();
    println!("Sorted: {:?}", sorted); // [1, 2, 5, 8, 9]

    // Dari range
    let heap3: BinaryHeap<i32> = (1..=10).collect();
    println!("Max dari 1..=10: {:?}", heap3.peek()); // Some(10)

    // drain — ambil semua sambil kosongkan heap
    let mut heap4: BinaryHeap<i32> = [5, 3, 8].into_iter().collect();
    let drained: Vec<i32> = heap4.drain().collect();
    println!("Drained: {:?}", drained); // semua elemen, tidak sorted
    println!("Kosong: {}", heap4.is_empty()); // true
}
```

---

## 🧠 Brain Teaser #2

Apakah perbezaan antara `into_vec()` dan `into_sorted_vec()`?

```rust
let heap: BinaryHeap<i32> = [5, 3, 8, 1, 9].into_iter().collect();
let a = heap.clone().into_vec();
let b = heap.into_sorted_vec();
```

<details>
<summary>👀 Jawapan</summary>

| | `into_vec()` | `into_sorted_vec()` |
|--|--|--|
| **Order** | Heap order (tidak sorted) | Ascending (sorted!) |
| **Masa** | O(1) — ambil Vec terus | O(n log n) — pop semua |
| **Guna** | Bila tidak perlu order | Bila nak sorted result |

`into_vec()` hanya "mendedahkan" underlying Vec heap — ia dalam heap order, bukan sorted.
`into_sorted_vec()` pop semua elemen (O(n log n)) dan return dalam ascending order.

```rust
// a mungkin: [9, 5, 8, 1, 3] — heap order
// b pastikan: [1, 3, 5, 8, 9] — ascending
```
</details>

---

# BAB 4: Min-Heap — Cara Buat ⬇️

> Rust BinaryHeap adalah **MAX-heap** sahaja.
> Untuk MIN-heap, ada beberapa teknik.

## Teknik 1: Reverse (Std Library)

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

fn main() {
    let mut min_heap: BinaryHeap<Reverse<i32>> = BinaryHeap::new();

    // Wrap dengan Reverse<T> untuk invert ordering
    min_heap.push(Reverse(5));
    min_heap.push(Reverse(10));
    min_heap.push(Reverse(3));
    min_heap.push(Reverse(8));
    min_heap.push(Reverse(1));

    // peek → elemen TERKECIL di atas
    println!("Min: {:?}", min_heap.peek()); // Some(Reverse(1))

    // Unwrap nilai dari Reverse
    while let Some(Reverse(val)) = min_heap.pop() {
        print!("{} ", val); // 1 3 5 8 10 — ascending!
    }
    println!();
}
```

---

## Teknik 2: Negasi untuk Integer

```rust
use std::collections::BinaryHeap;

fn main() {
    let mut min_heap: BinaryHeap<i32> = BinaryHeap::new();
    let data = [5, 10, 3, 8, 1, 7];

    // Masukkan sebagai negatif
    for &x in &data {
        min_heap.push(-x);
    }

    // Pop → dapat negatif terbesar = nilai terkecil asal
    while let Some(neg) = min_heap.pop() {
        print!("{} ", -neg); // 1 3 5 7 8 10
    }
    println!();
}
```

---

## Teknik 3: Wrapper Struct

```rust
use std::collections::BinaryHeap;
use std::cmp::Ordering;

#[derive(Debug, Eq, PartialEq)]
struct MinItem<T: Ord>(T);

impl<T: Ord> Ord for MinItem<T> {
    fn cmp(&self, other: &Self) -> Ordering {
        other.0.cmp(&self.0) // terbalik!
    }
}

impl<T: Ord> PartialOrd for MinItem<T> {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

fn main() {
    let mut min_heap: BinaryHeap<MinItem<i32>> = BinaryHeap::new();

    for x in [5, 1, 8, 3, 9, 2] {
        min_heap.push(MinItem(x));
    }

    println!("Min: {:?}", min_heap.peek()); // Some(MinItem(1))

    while let Some(MinItem(val)) = min_heap.pop() {
        print!("{} ", val); // 1 2 3 5 8 9
    }
    println!();
}
```

---

## Perbandingan Teknik Min-Heap

```
Teknik          Kebaikan              Keburukan
─────────────── ─────────────────── ──────────────────────
Reverse<T>      Idiomatik, selamat   Perlu unwrap Reverse
Negasi (-x)     Ringkas              Integer sahaja
Wrapper Struct  Fleksibel, bersih    Lebih verbose
```

> 💡 **Pilihan terbaik:** Guna `Reverse<T>` — ia idiomatik Rust dan selamat untuk semua type.

---

# BAB 5: BinaryHeap dengan Struct 🏗️

## Struct Dalam Heap — Mesti Implement Ord

```rust
use std::collections::BinaryHeap;
use std::cmp::Ordering;

#[derive(Debug, Eq, PartialEq, Clone)]
struct Tugasan {
    keutamaan: u32,
    nama:      String,
    id:        u32,
}

impl Tugasan {
    fn baru(id: u32, nama: &str, keutamaan: u32) -> Self {
        Tugasan { id, nama: nama.into(), keutamaan }
    }
}

// Impl Ord — sort by keutamaan (tinggi = diproses dulu)
impl Ord for Tugasan {
    fn cmp(&self, other: &Self) -> Ordering {
        self.keutamaan.cmp(&other.keutamaan)
            .then(self.id.cmp(&other.id)) // tie-break: id lebih kecil dulu
    }
}

impl PartialOrd for Tugasan {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

fn main() {
    let mut giliran: BinaryHeap<Tugasan> = BinaryHeap::new();

    giliran.push(Tugasan::baru(3, "Hantar laporan", 2));
    giliran.push(Tugasan::baru(1, "Server down!",   5)); // kritikal
    giliran.push(Tugasan::baru(4, "Kemaskini docs", 1));
    giliran.push(Tugasan::baru(2, "Review PR",      3));
    giliran.push(Tugasan::baru(5, "Baiki bug login",5)); // sama keutamaan dgn #1

    println!("=== Proses Tugasan (keutamaan tinggi dulu) ===");
    while let Some(t) = giliran.pop() {
        println!("  [P{}] {} (ID: {})", t.keutamaan, t.nama, t.id);
    }
}
```

**Output:**
```
=== Proses Tugasan (keutamaan tinggi dulu) ===
  [P5] Server down! (ID: 1)
  [P5] Baiki bug login (ID: 5)
  [P3] Review PR (ID: 2)
  [P2] Hantar laporan (ID: 3)
  [P1] Kemaskini docs (ID: 4)
```

---

## 🧠 Brain Teaser #3

Apakah urutan output jika kita tukar `Ord` implementation kepada ini?

```rust
impl Ord for Tugasan {
    fn cmp(&self, other: &Self) -> Ordering {
        other.keutamaan.cmp(&self.keutamaan) // terbalik dari contoh atas
    }
}
```

<details>
<summary>👀 Jawapan</summary>

Dengan `other.cmp(&self)` (terbalik), heap jadi **MIN-heap berdasarkan keutamaan** — keutamaan **RENDAH** diproses dulu!

Output:
```
[P1] Kemaskini docs
[P2] Hantar laporan
[P3] Review PR
[P5] Server down!    ← keutamaan tinggi diproses LAST!
[P5] Baiki bug login
```

Ini berguna bila nombor rendah = keutamaan tinggi (contoh: Priority 1 = paling kritikal).
</details>

---

# BAB 6: Custom Ordering dengan Ord 🎛️

## Struct Multi-Field Ordering

```rust
use std::collections::BinaryHeap;
use std::cmp::Ordering;

#[derive(Debug, Eq, PartialEq, Clone)]
struct Pesakit {
    nama:          String,
    kecemasan:     u8,   // 1=biasa, 2=segera, 3=kritikal
    masa_tunggu:   u32,  // minit menunggu
}

impl Ord for Pesakit {
    fn cmp(&self, other: &Self) -> Ordering {
        // Kecemasan tinggi dulu
        // Kalau sama kecemasan, yang tunggu lama dulu
        self.kecemasan.cmp(&other.kecemasan)
            .then(self.masa_tunggu.cmp(&other.masa_tunggu))
    }
}

impl PartialOrd for Pesakit {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

fn main() {
    let mut wad: BinaryHeap<Pesakit> = BinaryHeap::new();

    wad.push(Pesakit { nama: "Ali".into(),   kecemasan: 1, masa_tunggu: 30 });
    wad.push(Pesakit { nama: "Siti".into(),  kecemasan: 3, masa_tunggu: 5  }); // kritikal
    wad.push(Pesakit { nama: "Amin".into(),  kecemasan: 2, masa_tunggu: 15 });
    wad.push(Pesakit { nama: "Zara".into(),  kecemasan: 3, masa_tunggu: 10 }); // kritikal
    wad.push(Pesakit { nama: "Razi".into(),  kecemasan: 1, masa_tunggu: 45 });

    println!("=== Giliran Rawatan ===");
    while let Some(p) = wad.pop() {
        let tahap = match p.kecemasan {
            3 => "🔴 KRITIKAL",
            2 => "🟡 SEGERA",
            _ => "🟢 BIASA",
        };
        println!("  {} {} (tunggu {} min)", tahap, p.nama, p.masa_tunggu);
    }
}
```

---

# BAB 7: Priority Queue Aplikasi 🎯

## Dijkstra's Shortest Path

```rust
use std::collections::{BinaryHeap, HashMap};
use std::cmp::Reverse;

// State untuk Dijkstra
#[derive(Debug, Eq, PartialEq)]
struct State {
    kos:  u32,
    nod:  usize,
}

impl Ord for State {
    fn cmp(&self, other: &Self) -> std::cmp::Ordering {
        other.kos.cmp(&self.kos) // min-heap by kos
    }
}

impl PartialOrd for State {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        Some(self.cmp(other))
    }
}

fn dijkstra(graf: &Vec<Vec<(usize, u32)>>, mula: usize) -> Vec<u32> {
    let n = graf.len();
    let mut jarak = vec![u32::MAX; n];
    jarak[mula] = 0;

    let mut heap = BinaryHeap::new();
    heap.push(State { kos: 0, nod: mula });

    while let Some(State { kos, nod }) = heap.pop() {
        if kos > jarak[nod] { continue; } // dah ada laluan lebih pendek

        for &(jiran, berat) in &graf[nod] {
            let kos_baru = kos + berat;
            if kos_baru < jarak[jiran] {
                jarak[jiran] = kos_baru;
                heap.push(State { kos: kos_baru, nod: jiran });
            }
        }
    }
    jarak
}

fn main() {
    // Graf: (nod_tuju, berat)
    // Nod: 0=A, 1=B, 2=C, 3=D, 4=E
    let graf = vec![
        vec![(1, 4), (2, 1)],         // A → B(4), A → C(1)
        vec![(3, 1)],                  // B → D(1)
        vec![(1, 2), (3, 5)],         // C → B(2), C → D(5)
        vec![(4, 3)],                  // D → E(3)
        vec![],                        // E (destinasi)
    ];

    let jarak = dijkstra(&graf, 0); // dari nod 0 (A)
    let nod = ["A", "B", "C", "D", "E"];

    println!("Jarak terpendek dari A:");
    for (i, &j) in jarak.iter().enumerate() {
        if j == u32::MAX {
            println!("  A → {}: Tidak boleh sampai", nod[i]);
        } else {
            println!("  A → {}: {}", nod[i], j);
        }
    }
    // A → A: 0
    // A → B: 3  (A→C→B = 1+2)
    // A → C: 1
    // A → D: 4  (A→C→B→D = 1+2+1)
    // A → E: 7  (A→C→B→D→E = 1+2+1+3)
}
```

---

## Top-K Elements

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

// Cari K elemen TERKECIL — guna max-heap saiz K
fn top_k_terkecil(data: &[i32], k: usize) -> Vec<i32> {
    let mut heap: BinaryHeap<i32> = BinaryHeap::new(); // max-heap

    for &x in data {
        heap.push(x);
        if heap.len() > k {
            heap.pop(); // buang yang terbesar — kekal K terkecil
        }
    }

    let mut hasil: Vec<i32> = heap.into_iter().collect();
    hasil.sort();
    hasil
}

// Cari K elemen TERBESAR — guna min-heap saiz K
fn top_k_terbesar(data: &[i32], k: usize) -> Vec<i32> {
    let mut heap: BinaryHeap<Reverse<i32>> = BinaryHeap::new(); // min-heap

    for &x in data {
        heap.push(Reverse(x));
        if heap.len() > k {
            heap.pop(); // buang yang terkecil — kekal K terbesar
        }
    }

    let mut hasil: Vec<i32> = heap.into_iter().map(|Reverse(x)| x).collect();
    hasil.sort_by(|a, b| b.cmp(a)); // descending
    hasil
}

fn main() {
    let data = vec![3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5];
    println!("Data: {:?}", data);

    println!("Top 3 terkecil: {:?}", top_k_terkecil(&data, 3)); // [1, 1, 2]
    println!("Top 3 terbesar: {:?}", top_k_terbesar(&data, 3)); // [9, 6, 5]
}
```

---

## Merge K Sorted Lists

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

fn merge_k_sorted(lists: Vec<Vec<i32>>) -> Vec<i32> {
    // (nilai, index_list, index_elemen)
    let mut heap: BinaryHeap<Reverse<(i32, usize, usize)>> = BinaryHeap::new();

    // Masukkan elemen pertama setiap list
    for (i, list) in lists.iter().enumerate() {
        if !list.is_empty() {
            heap.push(Reverse((list[0], i, 0)));
        }
    }

    let mut hasil = Vec::new();

    while let Some(Reverse((val, list_idx, elem_idx))) = heap.pop() {
        hasil.push(val);
        let next_idx = elem_idx + 1;
        if next_idx < lists[list_idx].len() {
            heap.push(Reverse((lists[list_idx][next_idx], list_idx, next_idx)));
        }
    }
    hasil
}

fn main() {
    let lists = vec![
        vec![1, 4, 7],
        vec![2, 5, 8],
        vec![3, 6, 9],
    ];

    let merged = merge_k_sorted(lists);
    println!("Merged: {:?}", merged); // [1, 2, 3, 4, 5, 6, 7, 8, 9]

    // Cuba dengan saiz berbeza
    let lists2 = vec![
        vec![1, 10, 100],
        vec![5, 50],
        vec![2, 20, 200, 2000],
    ];
    let merged2 = merge_k_sorted(lists2);
    println!("Merged2: {:?}", merged2);
    // [1, 2, 5, 10, 20, 50, 100, 200, 2000]
}
```

---

## 🧠 Brain Teaser #4

Kenapa kita guna **max-heap saiz K** untuk cari K elemen terkecil (bukan min-heap)?

```rust
// Mencari 3 terkecil dari stream data
let data = [5, 3, 8, 1, 9, 2, 7];
// Jawapan yang dijangka: [1, 2, 3]
```

<details>
<summary>👀 Jawapan</summary>

Strategi: Kekalkan max-heap dengan **saiz K** sahaja.

- Bila elemen baru **lebih kecil** dari max dalam heap → pop max, push elemen baru
- Ini kekal K elemen terkecil dalam heap
- Root (max) sentiasa = **K-th smallest** element

Kalau guna min-heap → perlu simpan SEMUA elemen, baru pop K kali → O(n log n) space
Guna max-heap saiz K → hanya O(K) space, sesuai untuk data stream!

```
data = [5, 3, 8, 1, 9, 2, 7], k = 3

Push 5 → heap=[5]
Push 3 → heap=[5,3]
Push 8 → heap=[8,5,3]     (saiz=3, penuh)
Push 1 → 1<8, pop 8, push 1 → heap=[5,3,1]
Push 9 → 9>5(max), skip   → heap=[5,3,1]
Push 2 → 2<5, pop 5, push 2 → heap=[3,2,1]
Push 7 → 7>3(max), skip   → heap=[3,2,1]

Hasil: [1, 2, 3] ✔
```
</details>

---

# BAB 8: Heap Sort 🔢

```rust
use std::collections::BinaryHeap;

fn heap_sort_ascending(data: Vec<i32>) -> Vec<i32> {
    let mut heap: BinaryHeap<i32> = data.into_iter().collect();
    let mut hasil = Vec::with_capacity(heap.len());
    while let Some(val) = heap.pop() {
        hasil.push(val);
    }
    hasil.reverse(); // pop bagi descending, reverse untuk ascending
    hasil
}

fn heap_sort_descending(data: Vec<i32>) -> Vec<i32> {
    let mut heap: BinaryHeap<i32> = data.into_iter().collect();
    let mut hasil = Vec::with_capacity(heap.len());
    while let Some(val) = heap.pop() {
        hasil.push(val); // pop terus bagi descending
    }
    hasil
}

// Guna into_sorted_vec — lebih ringkas!
fn heap_sort_v2(data: Vec<i32>) -> Vec<i32> {
    let heap: BinaryHeap<i32> = data.into_iter().collect();
    heap.into_sorted_vec() // ascending terus!
}

fn main() {
    let data = vec![64, 34, 25, 12, 22, 11, 90];
    println!("Asal:       {:?}", data);
    println!("Ascending:  {:?}", heap_sort_ascending(data.clone()));
    println!("Descending: {:?}", heap_sort_descending(data.clone()));
    println!("v2 (asc):   {:?}", heap_sort_v2(data));
}
```

### Kompleksiti Heap Sort

```
┌─────────────────┬─────────────┐
│  Kes            │  Masa       │
├─────────────────┼─────────────┤
│  Best case      │  O(n log n) │
│  Average case   │  O(n log n) │
│  Worst case     │  O(n log n) │
│  Space          │  O(n)       │
└─────────────────┴─────────────┘

Berbanding:
  QuickSort  → O(n log n) avg, O(n²) worst
  MergeSort  → O(n log n) guaranteed, O(n) space
  HeapSort   → O(n log n) guaranteed, O(n) space (atau O(1) in-place)
```

---

# BAB 9: Pattern Lanjutan 🎯

## Running Median (Dua Heap)

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

// Median dalam data stream guna dua heap:
// max_heap: separuh bawah (elemen kecil)
// min_heap: separuh atas (elemen besar)
struct RunningMedian {
    max_heap: BinaryHeap<i32>,          // separuh kecil — max di atas
    min_heap: BinaryHeap<Reverse<i32>>, // separuh besar — min di atas
}

impl RunningMedian {
    fn baru() -> Self {
        RunningMedian {
            max_heap: BinaryHeap::new(),
            min_heap: BinaryHeap::new(),
        }
    }

    fn tambah(&mut self, num: i32) {
        // Masukkan ke max_heap (separuh kecil) dulu
        self.max_heap.push(num);

        // Pastikan semua dalam max_heap ≤ semua dalam min_heap
        if let (Some(&top_max), Some(&Reverse(top_min))) =
            (self.max_heap.peek(), self.min_heap.peek()) {
            if top_max > top_min {
                let val = self.max_heap.pop().unwrap();
                self.min_heap.push(Reverse(val));
            }
        }

        // Seimbangkan saiz — perbezaan max 1
        if self.max_heap.len() > self.min_heap.len() + 1 {
            let val = self.max_heap.pop().unwrap();
            self.min_heap.push(Reverse(val));
        } else if self.min_heap.len() > self.max_heap.len() {
            let Reverse(val) = self.min_heap.pop().unwrap();
            self.max_heap.push(val);
        }
    }

    fn median(&self) -> f64 {
        if self.max_heap.len() == self.min_heap.len() {
            // Bilangan genap — purata dua tengah
            let top_max = *self.max_heap.peek().unwrap() as f64;
            let Reverse(top_min) = *self.min_heap.peek().unwrap();
            (top_max + top_min as f64) / 2.0
        } else {
            // Bilangan ganjil — max_heap ada lebih satu
            *self.max_heap.peek().unwrap() as f64
        }
    }
}

fn main() {
    let mut rm = RunningMedian::baru();
    let stream = [5, 15, 1, 3, 8, 7, 9, 2];

    println!("Stream  | Median");
    println!("--------|--------");
    for &num in &stream {
        rm.tambah(num);
        println!("  {:3}   |  {:.1}", num, rm.median());
    }
}
```

---

## Sliding Window Maximum

```rust
use std::collections::BinaryHeap;

fn sliding_window_max(data: &[i32], k: usize) -> Vec<i32> {
    let mut hasil = Vec::new();
    // (nilai, index) — max-heap
    let mut heap: BinaryHeap<(i32, usize)> = BinaryHeap::new();

    for (i, &val) in data.iter().enumerate() {
        heap.push((val, i));

        // Buang elemen yang dah keluar dari window
        while let Some(&(_, idx)) = heap.peek() {
            if idx + k <= i {
                heap.pop();
            } else {
                break;
            }
        }

        // Tambah ke hasil bila window penuh
        if i + 1 >= k {
            hasil.push(heap.peek().unwrap().0);
        }
    }
    hasil
}

fn main() {
    let data = [1, 3, -1, -3, 5, 3, 6, 7];
    let k = 3;
    println!("Data:   {:?}", data);
    println!("k = {}", k);
    println!("Window max: {:?}", sliding_window_max(&data, k));
    // [3, 3, 5, 5, 6, 7]
}
```

---

## 🧠 Brain Teaser #5 (Advanced)

Terangkan bagaimana **Running Median** dengan dua heap berfungsi. Kenapa perlu DUA heap?

<details>
<summary>👀 Jawapan</summary>

```
Data stream: [5, 15, 1, 3, 8]

Idea: Bahagi data kepada dua separuh
  - max_heap (kiri)  = elemen kecil, root = terbesar dalam separuh kiri
  - min_heap (kanan) = elemen besar, root = terkecil dalam separuh kanan

Invariant:
  1. max(max_heap) ≤ min(min_heap)
  2. |len(max_heap) - len(min_heap)| ≤ 1

Selepas tambah [5, 15, 1, 3, 8]:

  max_heap: [5, 3, 1]  ← separuh kecil, root=5
  min_heap: [8, 15]    ← separuh besar, root=8

  Median = (5 + 8) / 2 = 6.5  (bilangan genap)

Kenapa dua heap?
  - Satu heap sahaja: untuk dapat median perlu O(n/2) pops
  - Dua heap: median sentiasa O(1) — ambil root kedua-dua heap!
  - Insert/rebalance: O(log n)
```
</details>

---

# BAB 10: Mini Project — Task Scheduler ⚙️

```rust
use std::collections::BinaryHeap;
use std::cmp::Ordering;

#[derive(Debug, Clone, Eq, PartialEq)]
struct Tugasan {
    id:           u32,
    nama:         String,
    keutamaan:    u8,   // 1-10, 10 = paling kritikal
    anggaran_ms:  u32,  // anggaran masa siapkan (ms)
    kategori:     Kategori,
}

#[derive(Debug, Clone, Eq, PartialEq)]
enum Kategori {
    Kritikal,
    Tinggi,
    Sederhana,
    Rendah,
}

impl Kategori {
    fn nilai(&self) -> u8 {
        match self {
            Kategori::Kritikal  => 4,
            Kategori::Tinggi    => 3,
            Kategori::Sederhana => 2,
            Kategori::Rendah    => 1,
        }
    }

    fn label(&self) -> &str {
        match self {
            Kategori::Kritikal  => "🔴 KRITIKAL",
            Kategori::Tinggi    => "🟠 TINGGI",
            Kategori::Sederhana => "🟡 SEDERHANA",
            Kategori::Rendah    => "🟢 RENDAH",
        }
    }
}

impl Ord for Tugasan {
    fn cmp(&self, other: &Self) -> Ordering {
        // 1. Kategori lebih tinggi dulu
        // 2. Keutamaan lebih tinggi dulu
        // 3. Anggaran masa lebih rendah dulu (tugasan cepat siap dulu)
        self.kategori.nilai().cmp(&other.kategori.nilai())
            .then(self.keutamaan.cmp(&other.keutamaan))
            .then(other.anggaran_ms.cmp(&self.anggaran_ms))
    }
}

impl PartialOrd for Tugasan {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

struct TaskScheduler {
    giliran:     BinaryHeap<Tugasan>,
    selesai:     Vec<Tugasan>,
    masa_total:  u32,
}

impl TaskScheduler {
    fn baru() -> Self {
        TaskScheduler {
            giliran:    BinaryHeap::new(),
            selesai:    Vec::new(),
            masa_total: 0,
        }
    }

    fn tambah_tugasan(&mut self, t: Tugasan) {
        println!("  ➕ Ditambah: [{}] {} (P{})",
            t.kategori.label(), t.nama, t.keutamaan);
        self.giliran.push(t);
    }

    fn proses_satu(&mut self) -> Option<&Tugasan> {
        if let Some(t) = self.giliran.pop() {
            self.masa_total += t.anggaran_ms;
            self.selesai.push(t);
            self.selesai.last()
        } else {
            None
        }
    }

    fn proses_semua(&mut self) {
        println!("\n{'='*50}");
        println!("       MEMPROSES SEMUA TUGASAN");
        println!("{'='*50}");
        let mut urutan = 1;
        while let Some(t) = self.giliran.pop() {
            println!("  #{} {} [{}] '{}' (~{}ms)",
                urutan,
                t.kategori.label(),
                t.keutamaan,
                t.nama,
                t.anggaran_ms);
            self.masa_total += t.anggaran_ms;
            self.selesai.push(t);
            urutan += 1;
        }
    }

    fn laporan(&self) {
        println!("\n{'='*50}");
        println!("         LAPORAN AKHIR");
        println!("{'='*50}");
        println!("Tugasan selesai: {}", self.selesai.len());
        println!("Masa total:      {}ms ({:.2}s)",
            self.masa_total, self.masa_total as f64 / 1000.0);

        // Kira tugasan per kategori
        let mut kira = [0u32; 4];
        for t in &self.selesai {
            kira[(t.kategori.nilai() - 1) as usize] += 1;
        }
        println!("\nRingkasan kategori:");
        println!("  🔴 Kritikal:  {}", kira[3]);
        println!("  🟠 Tinggi:    {}", kira[2]);
        println!("  🟡 Sederhana: {}", kira[1]);
        println!("  🟢 Rendah:    {}", kira[0]);
    }
}

fn main() {
    let mut scheduler = TaskScheduler::baru();

    println!("=== Menambah Tugasan ===");
    scheduler.tambah_tugasan(Tugasan {
        id: 1, nama: "Baiki crash production".into(),
        keutamaan: 10, anggaran_ms: 300,
        kategori: Kategori::Kritikal,
    });
    scheduler.tambah_tugasan(Tugasan {
        id: 2, nama: "Kemaskini dokumentasi".into(),
        keutamaan: 3, anggaran_ms: 120,
        kategori: Kategori::Rendah,
    });
    scheduler.tambah_tugasan(Tugasan {
        id: 3, nama: "Review pull request".into(),
        keutamaan: 6, anggaran_ms: 45,
        kategori: Kategori::Sederhana,
    });
    scheduler.tambah_tugasan(Tugasan {
        id: 4, nama: "Deploy ke staging".into(),
        keutamaan: 7, anggaran_ms: 200,
        kategori: Kategori::Tinggi,
    });
    scheduler.tambah_tugasan(Tugasan {
        id: 5, nama: "Security patch kritikal".into(),
        keutamaan: 9, anggaran_ms: 150,
        kategori: Kategori::Kritikal,
    });
    scheduler.tambah_tugasan(Tugasan {
        id: 6, nama: "Tulis unit tests".into(),
        keutamaan: 5, anggaran_ms: 90,
        kategori: Kategori::Sederhana,
    });

    println!("\nTugasan dalam giliran: {}", scheduler.giliran.len());

    scheduler.proses_semua();
    scheduler.laporan();
}
```

---

# 📋 Rujukan Pantas — BinaryHeap Cheat Sheet

## Methods Penting

| Method | Kegunaan | Big-O |
|--------|----------|-------|
| `push(v)` | Masukkan elemen | O(log n) |
| `pop()` | Ambil & buang max | O(log n) |
| `peek()` | Tengok max | O(1) |
| `peek_mut()` | Ubah max in-place | O(log n) |
| `len()` | Bilangan elemen | O(1) |
| `is_empty()` | Semak kosong | O(1) |
| `into_sorted_vec()` | Ambil sorted ascending | O(n log n) |
| `into_vec()` | Ambil Vec (tidak sorted) | O(1) |
| `drain()` | Ambil semua, kosongkan | O(n) |
| `clear()` | Kosongkan | O(n) |

## Buat Min-Heap (Ringkasan)

```rust
// Cara 1: Reverse<T> — DISYORKAN
use std::cmp::Reverse;
let mut min_heap: BinaryHeap<Reverse<i32>> = BinaryHeap::new();
min_heap.push(Reverse(5));
let min = min_heap.pop(); // Some(Reverse(1))

// Cara 2: Negatif (integer sahaja)
let mut min_heap: BinaryHeap<i32> = BinaryHeap::new();
min_heap.push(-5);
let min = -min_heap.pop().unwrap();
```

## Custom Ordering Ringkasan

```rust
// Wajib implement semua empat:
#[derive(Eq, PartialEq)]
struct MyStruct { ... }

impl Ord for MyStruct {
    fn cmp(&self, other: &Self) -> Ordering {
        // tentukan urutan di sini
        // self.cmp(&other) → max-heap biasa
        // other.cmp(&self) → min-heap
    }
}

impl PartialOrd for MyStruct {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other)) // delegate ke Ord
    }
}
```

## Algoritma Yang Guna BinaryHeap

```
Priority Queue      → push + pop mengikut keutamaan
Dijkstra            → min-heap, pop jarak terpendek
Top-K smallest      → max-heap saiz K
Top-K largest       → min-heap saiz K
Heap Sort           → push semua, pop semua
Running Median      → dua heap (max + min)
Sliding Window Max  → max-heap + buang elemen lama
Merge K sorted      → min-heap dengan index tracking
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **A\* Pathfinding** — guna BinaryHeap untuk priority queue dalam grid 2D
2. **Event Simulator** — simulate kejadian berdasarkan timestamp (min-heap by time)
3. **Huffman Encoding** — bina pokok Huffman guna min-heap untuk compression
4. **K-Way Merge** — merge K sorted files menggunakan BinaryHeap
5. **CPU Scheduler** — simulate round-robin + priority scheduling

---

*BinaryHeap in Rust — dari `push/pop` hingga Dijkstra dan Running Median. Selamat belajar!* 🦀
