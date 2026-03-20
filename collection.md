# 🗃️ Collections dalam Rust — Panduan Lengkap

> Panduan menyeluruh semua collections Rust: bila guna apa, perbandingan,
> dan cara pilih collection yang tepat untuk masalah anda.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Peta Besar Collections Rust

```
std::collections
│
├── Sequence (urutan penting)
│   ├── Vec<T>          — dynamic array, guna paling kerap
│   ├── VecDeque<T>     — double-ended queue, push/pop depan & belakang
│   └── LinkedList<T>   — doubly linked list (jarang diguna)
│
├── Map (key → value)
│   ├── HashMap<K,V>    — carian O(1), tiada order
│   └── BTreeMap<K,V>   — sorted by key, O(log n)
│
├── Set (nilai unik)
│   ├── HashSet<T>      — unik, tiada order, O(1)
│   └── BTreeSet<T>     — unik, sorted, O(log n)
│
└── Priority
    └── BinaryHeap<T>   — max-heap, pop = elemen terbesar

+ Primitif:
    [T; N]              — fixed-size array, stack
    &[T]                — slice, pandangan tanpa ownership
```

---

## Carta Pilihan Pantas

```
Soal diri anda:

Perlu simpan senarai nilai?
  ├── Saiz tetap (tahu semasa compile)?  → [T; N]  array
  ├── Tambah/buang belakang sahaja?      → Vec<T>
  ├── Tambah/buang depan & belakang?     → VecDeque<T>
  └── Banyak insert/delete di tengah?    → LinkedList<T>

Perlu map key → value?
  ├── Carian laju, order tak penting?    → HashMap<K,V>
  └── Perlu sorted / range query?        → BTreeMap<K,V>

Perlu koleksi nilai unik?
  ├── Carian laju, order tak penting?    → HashSet<T>
  └── Perlu sorted / range query?        → BTreeSet<T>

Perlu proses elemen terbesar/terkecil dulu?
  └── BinaryHeap<T>  (atau Reverse untuk min)
```

---

## Peta Pembelajaran

```
Bab 1  → Vec — Koleksi Paling Versatil
Bab 2  → VecDeque — Dua Hujung
Bab 3  → HashMap — Kamus Laju
Bab 4  → BTreeMap — Kamus Tersusun
Bab 5  → HashSet — Nilai Unik Laju
Bab 6  → BTreeSet — Nilai Unik Tersusun
Bab 7  → BinaryHeap — Keutamaan
Bab 8  → LinkedList — Rantai Node
Bab 9  → Perbandingan Menyeluruh
Bab 10 → Mini Project: Sistem Inventori Kedai
```

---

# BAB 1: Vec<T> — Koleksi Paling Versatil 📦

```
Vec<T>:  [ 10 | 20 | 30 | 40 | 50 ]
           ↑                     ↑
         index 0               index 4

push_back O(1)  ←─────────────────── push
pop_back  O(1)  ──────────────────→  pop
insert    O(n)  (semua shift kanan)
remove    O(n)  (semua shift kiri)
akses[i]  O(1)  terus ke alamat
```

```rust
use std::collections::*;

fn demo_vec() {
    let mut v: Vec<i32> = Vec::new();

    // Tambah
    v.push(10);
    v.push(20);
    v.push(30);
    v.insert(1, 15); // sisip di index 1 — O(n)!

    println!("Vec: {:?}", v); // [10, 15, 20, 30]

    // Baca
    println!("Indeks 2:  {}", v[2]);                    // 20
    println!("Get 10:    {:?}", v.get(10));              // None (selamat)
    println!("First:     {:?}", v.first());              // Some(10)
    println!("Last:      {:?}", v.last());               // Some(30)

    // Buang
    println!("Pop:       {:?}", v.pop());                // Some(30)
    println!("Remove[1]: {}", v.remove(1));              // 15
    v.retain(|&x| x > 10);                              // kekal > 10 sahaja
    println!("Selepas retain: {:?}", v);                 // [20]

    // Transformasi
    let v2 = vec![3, 1, 4, 1, 5, 9, 2, 6];
    let genap: Vec<i32> = v2.iter().filter(|&&x| x%2==0).copied().collect();
    let gandadua: Vec<i32> = v2.iter().map(|&x| x*2).collect();
    println!("Genap:    {:?}", genap);    // [4, 2, 6]
    println!("Ganda dua:{:?}", gandadua); // [6,2,8,2,10,18,4,12]

    // Sort
    let mut s = v2.clone();
    s.sort();
    println!("Sorted:   {:?}", s);

    // Jumlah
    let sum: i32 = v2.iter().sum();
    println!("Jumlah:   {}", sum);
}

fn main() { demo_vec(); }
```

### Bila Guna Vec?
```
✔ Default — guna Vec kecuali ada sebab spesifik lain
✔ Perlu akses by index
✔ Push/pop di belakang sahaja
✔ Iterate forward/backward
✔ Sort, filter, map
✔ Saiz tidak diketahui semasa compile
```

---

# BAB 2: VecDeque<T> — Dua Hujung 🚌

```
VecDeque<T> — ring buffer dalaman:

  ┌──────────────────────────────────┐
  │ [ _ | 30 | 40 | 50 | _ | _ ]   │  ← ring buffer
  │        ↑              ↑          │
  │       head           tail        │
  └──────────────────────────────────┘

push_front O(1)  ← tambah depan
push_back  O(1)  tambah belakang →
pop_front  O(1)  ← buang depan
pop_back   O(1)  buang belakang →
akses[i]   O(1)  masih boleh!
```

```rust
use std::collections::VecDeque;

fn demo_vecdeque() {
    let mut dq: VecDeque<i32> = VecDeque::new();

    // push kedua-dua hujung — O(1) untuk kedua-duanya!
    dq.push_back(20);
    dq.push_back(30);
    dq.push_front(10);
    dq.push_front(5);

    println!("VecDeque: {:?}", dq); // [5, 10, 20, 30]

    // pop kedua-dua hujung
    println!("Pop front: {:?}", dq.pop_front()); // Some(5)
    println!("Pop back:  {:?}", dq.pop_back());  // Some(30)
    println!("Baki: {:?}", dq);                  // [10, 20]

    // Masih boleh akses by index!
    println!("Index 0: {:?}", dq.get(0)); // Some(10)

    // rotate
    let mut dq2: VecDeque<i32> = (1..=5).collect();
    dq2.rotate_left(2);
    println!("Rotate left 2: {:?}", dq2); // [3, 4, 5, 1, 2]

    // Convert ke Vec
    let v: Vec<i32> = dq2.into_iter().collect();
    println!("Jadi Vec: {:?}", v);

    // Convert dari Vec
    let dq3: VecDeque<i32> = vec![1, 2, 3].into();
    println!("Dari Vec: {:?}", dq3);
}

fn main() { demo_vecdeque(); }
```

### Vec vs VecDeque
```
┌───────────────────┬──────────┬──────────────┐
│  Operasi          │  Vec     │  VecDeque    │
├───────────────────┼──────────┼──────────────┤
│  push_back        │  O(1)    │  O(1)        │
│  pop_back         │  O(1)    │  O(1)        │
│  push_front       │  O(n)❌  │  O(1)✔       │
│  pop_front        │  O(n)❌  │  O(1)✔       │
│  akses [i]        │  O(1)    │  O(1)        │
│  Memory layout    │ Contiguous│ Ring buffer  │
└───────────────────┴──────────┴──────────────┘
Guna VecDeque bila perlu push/pop di KEDUA-DUA hujung.
```

---

# BAB 3: HashMap<K,V> — Kamus Laju 🗺️

```
HashMap<K, V>:

  "Ali"   ──hash──▶  bucket[42]  ──▶  ("Ali",   85)
  "Siti"  ──hash──▶  bucket[17]  ──▶  ("Siti",  92)
  "Amin"  ──hash──▶  bucket[93]  ──▶  ("Amin",  78)

  Carian: O(1) — hash key → terus ke bucket!
  (Vec carian: O(n) — scan satu-satu)
```

```rust
use std::collections::HashMap;

fn demo_hashmap() {
    let mut markah: HashMap<&str, u32> = HashMap::new();

    // Insert
    markah.insert("Ali",  85);
    markah.insert("Siti", 92);
    markah.insert("Amin", 78);

    // Baca
    println!("Ali:  {:?}", markah.get("Ali"));  // Some(85)
    println!("Zara: {:?}", markah.get("Zara")); // None

    // Contains
    println!("Ada Siti? {}", markah.contains_key("Siti")); // true

    // Entry API — paling power!
    markah.entry("Zara").or_insert(70);         // insert kalau tiada
    *markah.entry("Ali").or_insert(0) += 5;     // tambah 5 ke Ali

    // Iterate
    let mut sorted: Vec<_> = markah.iter().collect();
    sorted.sort_by_key(|&(k, _)| *k);
    for (nama, skor) in sorted {
        println!("  {} → {}", nama, skor);
    }

    // Kira kekerapan — classic pattern
    let teks = "ali siti ali amin siti ali zara";
    let mut kira: HashMap<&str, u32> = HashMap::new();
    for perkataan in teks.split_whitespace() {
        *kira.entry(perkataan).or_insert(0) += 1;
    }
    println!("Kira: {:?}", kira);

    // Remove
    markah.remove("Amin");
    println!("Selepas remove: {:?}", markah.keys().collect::<Vec<_>>());
}

fn main() { demo_hashmap(); }
```

### Entry API — Ringkasan
```rust
map.entry(k).or_insert(v)           // insert v kalau tiada
map.entry(k).or_insert_with(f)      // insert hasil f() kalau tiada (lazy)
map.entry(k).or_default()           // insert Default::default() kalau tiada
map.entry(k).and_modify(|v| ...)    // ubah kalau ada
map.entry(k).and_modify(...).or_insert(v) // ubah kalau ada, insert kalau tiada
```

---

# BAB 4: BTreeMap<K,V> — Kamus Tersusun 🌲

```
BTreeMap — disimpan dalam B-tree:
  Keys sentiasa SORTED!

  10 ──▶ "sepuluh"
  20 ──▶ "dua puluh"
  30 ──▶ "tiga puluh"
  40 ──▶ "empat puluh"
     ↑
  sentiasa ascending
```

```rust
use std::collections::BTreeMap;

fn demo_btreemap() {
    let mut harga: BTreeMap<String, f64> = BTreeMap::new();

    harga.insert("Epal".into(),   3.50);
    harga.insert("Durian".into(), 25.00);
    harga.insert("Mangga".into(), 8.00);
    harga.insert("Pisang".into(), 2.50);
    harga.insert("Betik".into(),  5.00);

    // Iterate SENTIASA dalam key order (sorted)!
    println!("Senarai harga (sorted):");
    for (buah, hg) in &harga {
        println!("  {:<10} RM{:.2}", buah, hg);
    }
    // Betik, Durian, Epal, Mangga, Pisang — alphabetical!

    // Range query — HANYA BTreeMap boleh ini!
    println!("\nHarga antara Epal dan Pisang:");
    for (buah, hg) in harga.range("Epal".to_string()..="Pisang".to_string()) {
        println!("  {} → RM{:.2}", buah, hg);
    }

    // first_key_value / last_key_value
    println!("\nPaling awal: {:?}", harga.first_key_value()); // ("Betik", 5.0)
    println!("Paling akhir: {:?}", harga.last_key_value());   // ("Pisang", 2.5)

    // Operasi sama seperti HashMap
    harga.insert("Ciku".into(), 4.00);
    harga.remove("Durian");
    println!("Buah berdaftar: {}", harga.len());
}

fn main() { demo_btreemap(); }
```

### HashMap vs BTreeMap
```
┌──────────────────────┬─────────────┬─────────────┐
│  Ciri                │  HashMap    │  BTreeMap   │
├──────────────────────┼─────────────┼─────────────┤
│  Lookup              │  O(1) ✔✔   │  O(log n)   │
│  Insert              │  O(1) ✔✔   │  O(log n)   │
│  Sorted iteration    │  ✘          │  ✔ ✔        │
│  Range query         │  ✘          │  ✔ ✔        │
│  Min / Max key       │  ✘          │  O(log n)✔  │
│  Memory              │  Lebih      │  Lebih      │
│  Cache friendly      │  OK         │  Kurang     │
└──────────────────────┴─────────────┴─────────────┘
Pilih BTreeMap bila: perlu sorted, range query, min/max key.
Pilih HashMap bila:  perlu laju, order tak penting.
```

---

## 🧠 Brain Teaser #1

Kenapa output ini berbeza?

```rust
use std::collections::{HashMap, BTreeMap};

fn main() {
    let data = [("c", 3), ("a", 1), ("b", 2)];

    let hm: HashMap<&str, i32>  = data.into_iter().collect();
    let bm: BTreeMap<&str, i32> = data.into_iter().collect();

    let hm_keys: Vec<&&str> = hm.keys().collect();
    let bm_keys: Vec<&&str> = bm.keys().collect();

    // hm_keys → ???
    // bm_keys → ???
}
```

<details>
<summary>👀 Jawapan</summary>

```
hm_keys → tidak dijamin: mungkin ["c", "a", "b"] atau sebarang order
bm_keys → SENTIASA:      ["a", "b", "c"]  ← alphabetical ascending
```

`HashMap` tidak ada order — urutan bergantung kepada hash function dan bucket placement.
`BTreeMap` sentiasa sorted — iterate sentiasa dalam ascending key order.

**Implikasi praktikal:**
- Guna `BTreeMap` bila output perlu deterministik / sorted
- Guna `HashMap` bila perlu laju dan order tak penting
</details>

---

# BAB 5: HashSet<T> — Nilai Unik Laju 🔵

```
HashSet<T> — macam HashMap tapi KEY sahaja, tiada value:

  { "Ali", "Siti", "Amin", "Zara" }

  contains("Ali")  → O(1)  ← sangat laju!
  insert("Ali")    → false ← dah ada, diabaikan
  insert("Razi")   → true  ← baru ditambah

  Set Operations:
  A ∪ B = union        → semua
  A ∩ B = intersection → yang sama
  A - B = difference   → dalam A, tiada dalam B
```

```rust
use std::collections::HashSet;

fn demo_hashset() {
    let mut akses: HashSet<&str> = HashSet::new();

    // Insert — return bool
    println!("{}", akses.insert("admin"));   // true
    println!("{}", akses.insert("editor"));  // true
    println!("{}", akses.insert("admin"));   // false! dah ada

    // Contains — O(1)
    println!("Ada admin? {}", akses.contains("admin")); // true
    println!("Ada root?  {}", akses.contains("root"));  // false

    // Set operations
    let a: HashSet<i32> = [1, 2, 3, 4, 5].into_iter().collect();
    let b: HashSet<i32> = [3, 4, 5, 6, 7].into_iter().collect();

    let union:  Vec<i32> = { let mut v: Vec<_> = a.union(&b).copied().collect(); v.sort(); v };
    let inter:  Vec<i32> = { let mut v: Vec<_> = a.intersection(&b).copied().collect(); v.sort(); v };
    let diff_a: Vec<i32> = { let mut v: Vec<_> = a.difference(&b).copied().collect(); v.sort(); v };
    let sym:    Vec<i32> = { let mut v: Vec<_> = a.symmetric_difference(&b).copied().collect(); v.sort(); v };

    println!("A ∪ B  = {:?}", union);  // [1,2,3,4,5,6,7]
    println!("A ∩ B  = {:?}", inter);  // [3,4,5]
    println!("A - B  = {:?}", diff_a); // [1,2]
    println!("A △ B  = {:?}", sym);    // [1,2,6,7]

    // Buang duplikat dari Vec
    let v = vec![1, 2, 2, 3, 3, 3, 4];
    let unik: HashSet<i32> = v.into_iter().collect();
    println!("Unik: {:?}", unik); // {1,2,3,4}

    // Subset check
    let pentadbir: HashSet<&str> = ["admin", "superuser"].into_iter().collect();
    let user_role: HashSet<&str> = ["admin", "editor", "superuser"].into_iter().collect();
    println!("Pentadbir ⊆ user? {}", pentadbir.is_subset(&user_role)); // true
}

fn main() { demo_hashset(); }
```

---

# BAB 6: BTreeSet<T> — Nilai Unik Tersusun 🌳

```
BTreeSet — macam HashSet tapi SORTED:

  { 1, 3, 5, 7, 9 }  ← sentiasa ascending

  range(3..=7) → [3, 5, 7]  ← boleh range query!
  first()      → Some(1)
  last()        → Some(9)
```

```rust
use std::collections::BTreeSet;

fn demo_btreeset() {
    let mut s: BTreeSet<i32> = BTreeSet::new();

    for x in [5, 3, 8, 1, 9, 2, 7, 4, 6] {
        s.insert(x);
    }

    // Iterate SENTIASA sorted
    println!("BTreeSet: {:?}", s); // {1,2,3,4,5,6,7,8,9}

    // first / last
    println!("Min: {:?}", s.first()); // Some(1)
    println!("Max: {:?}", s.last());  // Some(9)

    // Range query — UNIK kepada BTreeSet!
    let dalam_range: Vec<&i32> = s.range(3..=7).collect();
    println!("Range 3..=7: {:?}", dalam_range); // [3,4,5,6,7]

    // Semua set operations sama seperti HashSet
    let a: BTreeSet<i32> = [1,2,3,4,5].into_iter().collect();
    let b: BTreeSet<i32> = [3,4,5,6,7].into_iter().collect();
    let union: BTreeSet<i32> = a.union(&b).copied().collect();
    println!("Union (sorted): {:?}", union); // {1,2,3,4,5,6,7}

    // split_off — pecah set
    let mut s2: BTreeSet<i32> = (1..=10).collect();
    let atas = s2.split_off(&6); // 6 dan ke atas pindah ke atas
    println!("Bawah: {:?}", s2);  // {1,2,3,4,5}
    println!("Atas:  {:?}", atas); // {6,7,8,9,10}
}

fn main() { demo_btreeset(); }
```

### HashSet vs BTreeSet
```
┌──────────────────────┬──────────────┬──────────────┐
│  Ciri                │  HashSet     │  BTreeSet    │
├──────────────────────┼──────────────┼──────────────┤
│  contains            │  O(1) ✔✔    │  O(log n)    │
│  insert              │  O(1) ✔✔    │  O(log n)    │
│  Sorted iteration    │  ✘           │  ✔ ✔         │
│  Range query         │  ✘           │  ✔ ✔         │
│  first / last        │  ✘           │  ✔ ✔         │
│  Set operations      │  ✔           │  ✔           │
└──────────────────────┴──────────────┴──────────────┘
```

---

# BAB 7: BinaryHeap<T> — Keutamaan 🏔️

```
BinaryHeap — MAX-HEAP:

              100             ← pop() sentiasa ambil ini
            /     \
          85        90
         /  \      /  \
        70   80  60    75

  push O(log n) — sisip & sift up
  pop  O(log n) — ambil max & sift down
  peek O(1)     — tengok max sahaja
```

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

fn demo_binaryheap() {
    // MAX-HEAP (default)
    let mut heap: BinaryHeap<i32> = BinaryHeap::new();
    for x in [3, 1, 4, 1, 5, 9, 2, 6] {
        heap.push(x);
    }

    println!("Max: {:?}", heap.peek()); // Some(9)

    // Pop satu-satu → descending order
    print!("Pop order: ");
    while let Some(v) = heap.pop() {
        print!("{} ", v); // 9 6 5 4 3 2 1 1
    }
    println!();

    // MIN-HEAP guna Reverse<T>
    let mut min_heap: BinaryHeap<Reverse<i32>> = BinaryHeap::new();
    for x in [3, 1, 4, 1, 5, 9, 2, 6] {
        min_heap.push(Reverse(x));
    }

    print!("Min-heap pop: ");
    while let Some(Reverse(v)) = min_heap.pop() {
        print!("{} ", v); // 1 1 2 3 4 5 6 9
    }
    println!();

    // Priority Queue — struct dengan Ord
    use std::cmp::Ordering;
    #[derive(Eq, PartialEq)]
    struct Tugasan { keutamaan: u8, nama: &'static str }
    impl Ord for Tugasan {
        fn cmp(&self, other: &Self) -> Ordering {
            self.keutamaan.cmp(&other.keutamaan)
        }
    }
    impl PartialOrd for Tugasan {
        fn partial_cmp(&self, other: &Self) -> Option<Ordering> { Some(self.cmp(other)) }
    }

    let mut pq: BinaryHeap<Tugasan> = BinaryHeap::new();
    pq.push(Tugasan { keutamaan: 3, nama: "Laporan" });
    pq.push(Tugasan { keutamaan: 5, nama: "Server down!" });
    pq.push(Tugasan { keutamaan: 1, nama: "Mesyuarat" });

    while let Some(t) = pq.pop() {
        println!("  [P{}] {}", t.keutamaan, t.nama);
    }
    // [P5] Server down!
    // [P3] Laporan
    // [P1] Mesyuarat
}

fn main() { demo_binaryheap(); }
```

---

# BAB 8: LinkedList<T> — Rantai Node 🔗

```
LinkedList — tiap node ada pointer ke seterusnya:

  ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
  │  10  │───▶│  20  │───▶│  30  │───▶│  40  │───▶ None
  └──────┘    └──────┘    └──────┘    └──────┘
    head                                tail

  push_front O(1)  ← tambah depan
  push_back  O(1)  tambah belakang →
  pop_front  O(1)  ← buang depan
  pop_back   O(1)  buang belakang →
  akses[i]   O(n)  ← perlu traverse!
```

```rust
use std::collections::LinkedList;

fn demo_linkedlist() {
    let mut ll: LinkedList<i32> = LinkedList::new();

    ll.push_back(20);
    ll.push_back(30);
    ll.push_front(10);
    ll.push_front(5);

    println!("LinkedList: {:?}", ll); // [5, 10, 20, 30]
    println!("Front: {:?}", ll.front()); // Some(5)
    println!("Back:  {:?}", ll.back());  // Some(30)

    // Pop kedua-dua hujung
    println!("Pop front: {:?}", ll.pop_front()); // Some(5)
    println!("Pop back:  {:?}", ll.pop_back());  // Some(30)

    // append — gabung dua list O(1)
    let mut a: LinkedList<i32> = [1,2,3].into_iter().collect();
    let mut b: LinkedList<i32> = [4,5,6].into_iter().collect();
    a.append(&mut b);
    println!("Selepas append: {:?}", a); // [1,2,3,4,5,6]
    println!("b kosong: {}", b.is_empty()); // true

    // split_off — pecah di index
    let ekor = a.split_off(3);
    println!("Kepala: {:?}", a);    // [1,2,3]
    println!("Ekor:   {:?}", ekor); // [4,5,6]
}

fn main() { demo_linkedlist(); }
```

### Bila Guna LinkedList?
```
✔ Perlu split / append O(1) yang kerap
✔ Perlu insert / delete di cursor position
✘ Akses by index → O(n), guna Vec
✘ Kebanyakan kes → VecDeque lebih laju!
```

---

## 🧠 Brain Teaser #2

Padankan collection dengan kegunaan yang sesuai:

```
Collection          Kegunaan
──────────────────  ──────────────────────────────────────
A. Vec<T>           1. Leaderboard skor — perlu sorted output
B. VecDeque<T>      2. Cek username tersedia — lookup O(1)
C. HashMap<K,V>     3. Senarai pesanan restoran — FIFO queue
D. BTreeMap<K,V>    4. Senarai beli belah — tambah/buang akhir
E. HashSet<T>       5. Rekod suhu harian — lookup by tarikh, sorted
F. BinaryHeap<T>    6. Giliran kecemasan hospital — keutamaan
```

<details>
<summary>👀 Jawapan</summary>

```
A. Vec<T>        → 4. Senarai beli belah
B. VecDeque<T>   → 3. Senarai pesanan (FIFO — push_back, pop_front)
C. HashMap<K,V>  → 2. Cek username (O(1) contains)
D. BTreeMap<K,V> → 5. Rekod suhu by tarikh (sorted + range query)
E. HashSet<T>    → 1. Leaderboard? Bukan — pakai BTreeSet!
                      Sebenarnya: cek duplikat / membership
F. BinaryHeap<T> → 6. Giliran kecemasan (pop = keutamaan tertinggi)
```

Nota: Leaderboard lebih sesuai `BTreeMap<Skor, Nama>` (sorted) atau `Vec` (sort dahulu).
</details>

---

# BAB 9: Perbandingan Menyeluruh 📊

## Big-O Semua Collections

```
┌──────────────────┬──────────┬──────────┬──────────┬──────────┬────────────┐
│  Collection      │  Access  │  Search  │  Insert  │  Delete  │  Space     │
├──────────────────┼──────────┼──────────┼──────────┼──────────┼────────────┤
│  [T; N] array    │  O(1)    │  O(n)    │  N/A     │  N/A     │  O(n)      │
│  Vec<T>          │  O(1)    │  O(n)    │  O(1)*   │  O(n)    │  O(n)      │
│  VecDeque<T>     │  O(1)    │  O(n)    │  O(1)*   │  O(1)**  │  O(n)      │
│  LinkedList<T>   │  O(n)    │  O(n)    │  O(1)*** │  O(1)*** │  O(n)      │
│  HashMap<K,V>    │  N/A     │  O(1)    │  O(1)    │  O(1)    │  O(n)      │
│  BTreeMap<K,V>   │  N/A     │  O(log n)│  O(log n)│  O(log n)│  O(n)      │
│  HashSet<T>      │  N/A     │  O(1)    │  O(1)    │  O(1)    │  O(n)      │
│  BTreeSet<T>     │  N/A     │  O(log n)│  O(log n)│  O(log n)│  O(n)      │
│  BinaryHeap<T>   │  O(1)*** │  O(n)    │  O(log n)│  O(log n)│  O(n)      │
└──────────────────┴──────────┴──────────┴──────────┴──────────┴────────────┘

*   amortized (purata), boleh O(n) bila resize
**  di hujung sahaja; insert/delete tengah O(n)
*** di hujung/depan sahaja; atau bila ada pointer
*** BinaryHeap: O(1) access ke max sahaja
```

---

## Perbandingan Memory

```
[T; N]  — Stack sahaja, tiada overhead, paling compact
Vec<T>  — Stack (pointer+len+cap) + Heap (data)
          overhead: 24 bytes (pointer=8, len=8, cap=8)
HashMap — Stack (ptr+len+cap+hasher) + Heap (buckets)
          overhead: lebih besar, ada empty buckets untuk reduce collision
BTree*  — Stack + Heap (tree nodes)
          setiap node ada: data + pointers ke children
```

---

## Pattern Kegunaan Biasa

```rust
use std::collections::*;

fn contoh_pattern() {
    // 1. WORD COUNT — HashMap classic
    let teks = "rust rust adalah bahasa rust yang selamat";
    let mut kira: HashMap<&str, usize> = HashMap::new();
    for w in teks.split_whitespace() {
        *kira.entry(w).or_insert(0) += 1;
    }

    // 2. GROUPING — HashMap<K, Vec<V>>
    let data = vec![("ICT","Ali"), ("HR","Siti"), ("ICT","Amin"), ("HR","Zara")];
    let mut kumpul: HashMap<&str, Vec<&str>> = HashMap::new();
    for (bahagian, nama) in data {
        kumpul.entry(bahagian).or_insert_with(Vec::new).push(nama);
    }

    // 3. DEDUP kekal order — HashSet sebagai seen set
    let v = vec![3,1,4,1,5,9,2,6,5,3];
    let mut seen = HashSet::new();
    let unik: Vec<i32> = v.into_iter().filter(|x| seen.insert(*x)).collect();
    println!("Unik: {:?}", unik); // [3,1,4,5,9,2,6]

    // 4. TOP-K — BinaryHeap saiz K
    use std::cmp::Reverse;
    let data2 = [5,3,8,1,9,2,7,4,6];
    let k = 3;
    let mut heap: BinaryHeap<Reverse<i32>> = BinaryHeap::new();
    for &x in &data2 {
        heap.push(Reverse(x));
        if heap.len() > k { heap.pop(); }
    }
    let mut top_k: Vec<i32> = heap.into_iter().map(|Reverse(x)| x).collect();
    top_k.sort_by(|a,b| b.cmp(a));
    println!("Top-{}: {:?}", k, top_k); // [9,8,7]

    // 5. SORTED UNIQUE — BTreeSet
    let v2 = vec![5,3,8,1,9,2,7,4,6,3,1,5];
    let sorted_unik: BTreeSet<i32> = v2.into_iter().collect();
    println!("Sorted unik: {:?}", sorted_unik); // {1,2,3,4,5,6,7,8,9}

    // 6. RANGE QUERY — BTreeMap
    let mut suhu: BTreeMap<&str, f64> = BTreeMap::new();
    suhu.insert("2024-01-01", 28.5);
    suhu.insert("2024-01-02", 29.1);
    suhu.insert("2024-01-03", 27.8);
    suhu.insert("2024-01-04", 30.2);
    let jan_2_3: Vec<_> = suhu.range("2024-01-02"..="2024-01-03").collect();
    println!("Jan 2-3: {:?}", jan_2_3);

    // 7. FREQUENCY SORT — HashMap + Vec + sort
    let mut freq: HashMap<char, usize> = HashMap::new();
    for c in "hello world".chars() {
        *freq.entry(c).or_insert(0) += 1;
    }
    let mut freq_vec: Vec<(char, usize)> = freq.into_iter().collect();
    freq_vec.sort_by(|a, b| b.1.cmp(&a.1));
    println!("Freq sort: {:?}", &freq_vec[..3]); // [('l',3),('o',2),(' ',1)]
}

fn main() { contoh_pattern(); }
```

---

## Convert Antara Collections

```rust
use std::collections::*;

fn main() {
    let v = vec![3, 1, 4, 1, 5, 9, 2, 6, 5, 3];

    // Vec → HashSet (buang duplikat)
    let hs: HashSet<i32> = v.iter().copied().collect();
    println!("HashSet: {:?}", hs);

    // Vec → BTreeSet (buang duplikat + sorted)
    let bs: BTreeSet<i32> = v.iter().copied().collect();
    println!("BTreeSet: {:?}", bs);

    // Vec → BinaryHeap
    let bh: BinaryHeap<i32> = v.iter().copied().collect();
    println!("Max: {:?}", bh.peek()); // Some(9)

    // Vec → VecDeque
    let vd: VecDeque<i32> = v.iter().copied().collect();
    println!("VecDeque front: {:?}", vd.front());

    // HashSet → Vec (tidak sorted)
    let v2: Vec<i32> = hs.into_iter().collect();

    // BTreeSet → Vec (sorted!)
    let v3: Vec<i32> = bs.into_iter().collect();
    println!("Sorted Vec: {:?}", v3);

    // Pasangan → HashMap
    let pairs = vec![("a",1),("b",2),("c",3)];
    let hm: HashMap<&str, i32> = pairs.into_iter().collect();

    // HashMap → BTreeMap (untuk sorted output)
    let bm: BTreeMap<&str, i32> = hm.into_iter().collect();
    println!("BTreeMap: {:?}", bm); // sorted by key
}
```

---

## 🧠 Brain Teaser #3

Collection mana yang paling sesuai untuk setiap senario ini?

```
1. Simpan 1 juta IP address pelawat unik laman web —
   semak sama ada IP dah melawat sebelum ini.

2. Leaderboard game — papar top 10 skor terkini,
   tambah skor baru bila pemain tamat.

3. Cache HTTP — simpan URL → response body,
   had 100 entries, buang yang paling lama tidak diakses.

4. Jadual waktu penerbangan hari ini —
   cari penerbangan dalam tempoh 2 jam akan datang.

5. Undo/Redo dalam text editor —
   tambah operasi, undo buang yang terbaru.
```

<details>
<summary>👀 Jawapan</summary>

```
1. HashSet<IpAddr>
   — O(1) insert & contains, memory efficient untuk banyak data

2. BinaryHeap<(Skor, Nama)> + Vec untuk display
   — push O(log n), ambil top-10 dengan pop 10 kali

3. HashMap<Url, Response> + LinkedList untuk LRU order
   — HashMap O(1) lookup, LinkedList track access order
   — Atau guna crate: lru

4. BTreeMap<DateTime, Penerbangan>
   — range query: map.range(sekarang..sekarang+2jam)
   — Sorted by time secara automatik!

5. Vec<Operasi> sebagai stack
   — push = tambah operasi
   — pop = undo (ambil yang terbaru)
   — Simpan dua Vec: undo_stack + redo_stack
```
</details>

---

# BAB 10: Mini Project — Sistem Inventori Kedai 🏪

> Projek ini **gabungkan semua collections** dalam satu sistem realistik.

```rust
use std::collections::{HashMap, BTreeMap, HashSet, BinaryHeap};
use std::cmp::Ordering;

// ─── Model Data ───────────────────────────────────────────────

#[derive(Debug, Clone)]
struct Produk {
    id:       u32,
    nama:     String,
    kategori: String,
    harga:    f64,
    stok:     u32,
}

impl Produk {
    fn baru(id: u32, nama: &str, kategori: &str, harga: f64, stok: u32) -> Self {
        Produk { id, nama: nama.into(), kategori: kategori.into(), harga, stok }
    }
}

#[derive(Debug, Eq, PartialEq)]
struct ProdukStokRendah {
    stok:    u32,
    id:      u32,
    nama:    String,
}

impl Ord for ProdukStokRendah {
    fn cmp(&self, other: &Self) -> Ordering {
        // Min-heap: stok paling rendah dulu
        other.stok.cmp(&self.stok).then(self.id.cmp(&other.id))
    }
}

impl PartialOrd for ProdukStokRendah {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> { Some(self.cmp(other)) }
}

// ─── Sistem Inventori ─────────────────────────────────────────

struct Inventori {
    // HashMap untuk lookup laju by ID
    produk: HashMap<u32, Produk>,

    // BTreeMap untuk senarai tersusun by nama
    ikut_nama: BTreeMap<String, u32>, // nama → id

    // HashMap untuk grouping by kategori
    ikut_kategori: HashMap<String, HashSet<u32>>, // kategori → set of ids

    // BinaryHeap untuk track stok rendah
    amaran_stok: BinaryHeap<ProdukStokRendah>,

    // HashSet untuk track produk yang dah habis
    kehabisan: HashSet<u32>,

    id_seterusnya: u32,
}

impl Inventori {
    fn baru() -> Self {
        Inventori {
            produk:         HashMap::new(),
            ikut_nama:      BTreeMap::new(),
            ikut_kategori:  HashMap::new(),
            amaran_stok:    BinaryHeap::new(),
            kehabisan:      HashSet::new(),
            id_seterusnya:  1,
        }
    }

    fn tambah_produk(&mut self, nama: &str, kategori: &str, harga: f64, stok: u32) -> u32 {
        let id = self.id_seterusnya;
        self.id_seterusnya += 1;

        let produk = Produk::baru(id, nama, kategori, harga, stok);

        // Update semua collections
        self.ikut_nama.insert(nama.to_string(), id);
        self.ikut_kategori
            .entry(kategori.to_string())
            .or_insert_with(HashSet::new)
            .insert(id);

        if stok == 0 {
            self.kehabisan.insert(id);
        } else if stok < 5 {
            self.amaran_stok.push(ProdukStokRendah {
                stok, id, nama: nama.to_string()
            });
        }

        self.produk.insert(id, produk);
        println!("  ✔ Ditambah: {} (ID:{}, Stok:{})", nama, id, stok);
        id
    }

    fn jual(&mut self, id: u32, kuantiti: u32) -> Result<f64, String> {
        let produk = self.produk.get_mut(&id)
            .ok_or_else(|| format!("Produk ID {} tidak dijumpai", id))?;

        if produk.stok < kuantiti {
            return Err(format!("Stok tidak cukup! Ada: {}, Minta: {}", produk.stok, kuantiti));
        }

        produk.stok -= kuantiti;
        let jumlah = produk.harga * kuantiti as f64;

        if produk.stok == 0 {
            self.kehabisan.insert(id);
            println!("  ⚠ HABIS: {}", produk.nama);
        } else if produk.stok < 5 {
            self.amaran_stok.push(ProdukStokRendah {
                stok: produk.stok, id,
                nama: produk.nama.clone()
            });
        }

        println!("  ✔ Dijual {}x {} = RM{:.2}", kuantiti, produk.nama, jumlah);
        Ok(jumlah)
    }

    fn tambah_stok(&mut self, id: u32, kuantiti: u32) -> Result<(), String> {
        let produk = self.produk.get_mut(&id)
            .ok_or_else(|| format!("Produk ID {} tidak dijumpai", id))?;

        produk.stok += kuantiti;
        self.kehabisan.remove(&id);
        println!("  ✔ Stok {} ditambah {}. Baki: {}", produk.nama, kuantiti, produk.stok);
        Ok(())
    }

    fn cari_nama(&self, nama: &str) -> Option<&Produk> {
        self.ikut_nama.get(nama)
            .and_then(|&id| self.produk.get(&id))
    }

    fn senarai_kategori(&self, kategori: &str) -> Vec<&Produk> {
        // BTreeMap untuk return sorted by nama
        let mut hasil = BTreeMap::new();
        if let Some(ids) = self.ikut_kategori.get(kategori) {
            for &id in ids {
                if let Some(p) = self.produk.get(&id) {
                    hasil.insert(&p.nama, p);
                }
            }
        }
        hasil.into_values().collect()
    }

    fn laporan_amaran_stok(&self, had: usize) {
        println!("\n  ⚠ Produk Stok Rendah (≤5):");
        // Ambil dari BinaryHeap sementara (clone)
        let mut temp = self.amaran_stok.clone();
        let mut cetak = 0;
        let mut dah_cetak = HashSet::new();
        while let Some(p) = temp.pop() {
            if cetak >= had { break; }
            if dah_cetak.contains(&p.id) { continue; }
            // Dapatkan stok terkini
            if let Some(produk) = self.produk.get(&p.id) {
                if produk.stok < 5 {
                    println!("    🔴 {} — Stok: {}", produk.nama, produk.stok);
                    dah_cetak.insert(p.id);
                    cetak += 1;
                }
            }
        }
    }

    fn laporan_keseluruhan(&self) {
        println!("\n{'═'*50}");
        println!("{:^50}", "LAPORAN INVENTORI");
        println!("{'═'*50}");
        println!("Jumlah produk:    {}", self.produk.len());
        println!("Produk kehabisan: {}", self.kehabisan.len());

        // Nilai inventori total — iterate BTreeMap (sorted by nama)
        let nilai_total: f64 = self.produk.values()
            .map(|p| p.harga * p.stok as f64)
            .sum();
        println!("Nilai inventori:  RM{:.2}", nilai_total);

        // Senarai by kategori (BTreeMap untuk sorted output)
        let mut kategori_sorted: BTreeMap<&str, usize> = BTreeMap::new();
        for (kat, ids) in &self.ikut_kategori {
            kategori_sorted.insert(kat, ids.len());
        }

        println!("\nProduk per kategori:");
        for (kat, bil) in &kategori_sorted {
            println!("  {:<15} {} produk", kat, bil);
        }

        if !self.kehabisan.is_empty() {
            println!("\nProduk kehabisan:");
            let mut habis: Vec<&str> = self.kehabisan.iter()
                .filter_map(|id| self.produk.get(id))
                .map(|p| p.nama.as_str())
                .collect();
            habis.sort();
            for nama in habis {
                println!("  ❌ {}", nama);
            }
        }

        self.laporan_amaran_stok(5);
    }
}

fn main() {
    let mut kedai = Inventori::baru();

    println!("═══ TAMBAH PRODUK ═══");
    kedai.tambah_produk("Beras 5kg",    "Makanan",    12.90, 50);
    kedai.tambah_produk("Minyak 1L",    "Makanan",     6.50, 30);
    kedai.tambah_produk("Gula 1kg",     "Makanan",     3.20, 0 ); // kehabisan!
    kedai.tambah_produk("Sabun Mandi",  "Kebersihan",  2.50, 4 ); // stok rendah
    kedai.tambah_produk("Syampu",       "Kebersihan",  8.90, 15);
    kedai.tambah_produk("Pensil 2B",    "Alat Tulis",  1.20, 3 ); // stok rendah
    kedai.tambah_produk("Buku Nota",    "Alat Tulis",  4.50, 20);
    kedai.tambah_produk("Pemadam",      "Alat Tulis",  0.80, 2 ); // stok rendah
    kedai.tambah_produk("Air Mineral",  "Minuman",     1.50, 100);
    kedai.tambah_produk("Kopi O",       "Minuman",     5.50, 25);

    println!("\n═══ TRANSAKSI JUALAN ═══");
    let _ = kedai.jual(1, 5);   // beras
    let _ = kedai.jual(4, 3);   // sabun — stok jadi 1
    let _ = kedai.jual(3, 1);   // gula — kehabisan!
    let _ = kedai.jual(9, 200); // air mineral — error stok tak cukup

    println!("\n═══ TAMBAH STOK ═══");
    let _ = kedai.tambah_stok(3, 20); // restok gula
    let _ = kedai.tambah_stok(4, 10); // restok sabun

    println!("\n═══ CARI PRODUK ═══");
    if let Some(p) = kedai.cari_nama("Beras 5kg") {
        println!("  Jumpa: {} — RM{:.2}, Stok: {}", p.nama, p.harga, p.stok);
    }
    println!("  Cari 'Rotan': {:?}", kedai.cari_nama("Rotan").map(|p| &p.nama));

    println!("\n═══ SENARAI KATEGORI: Alat Tulis ═══");
    for p in kedai.senarai_kategori("Alat Tulis") {
        println!("  {} — RM{:.2} (Stok: {})", p.nama, p.harga, p.stok);
    }

    kedai.laporan_keseluruhan();
}
```

---

# 📋 Rujukan Pantas — Collections Cheat Sheet

## Pilih Collection

```
Nak apa?                                  Guna
────────────────────────────────────────  ─────────────────
Senarai dinamik, akses by index           Vec<T>
Push/pop depan & belakang, akses index    VecDeque<T>
Insert/delete di tengah dengan iterator  LinkedList<T>
Saiz tetap, stack memory, no alloc        [T; N]
Key→value, carian O(1)                    HashMap<K,V>
Key→value, sorted, range query            BTreeMap<K,V>
Nilai unik, cek ada/tiada O(1)            HashSet<T>
Nilai unik, sorted, range query           BTreeSet<T>
Ambil elemen terbesar/terkecil dulu       BinaryHeap<T>
```

## Kompleksiti Ringkas

```
              Access   Search   Insert   Delete
[T; N]         O(1)     O(n)    —        —
Vec            O(1)     O(n)    O(1)*    O(n)
VecDeque       O(1)     O(n)    O(1)*    O(1)**
LinkedList     O(n)     O(n)    O(1)***  O(1)***
HashMap        —        O(1)    O(1)     O(1)
BTreeMap       —      O(log n)  O(log n) O(log n)
HashSet        —        O(1)    O(1)     O(1)
BTreeSet       —      O(log n)  O(log n) O(log n)
BinaryHeap   O(1)****   O(n)    O(log n) O(log n)

*    amortized
**   di hujung; tengah O(n)
***  bila ada pointer/cursor
**** max sahaja
```

## Convert Ringkas

```rust
// Ke Vec
let v: Vec<T>           = collection.into_iter().collect();

// Ke HashSet (buang duplikat)
let hs: HashSet<T>      = vec.into_iter().collect();

// Ke BTreeSet (buang duplikat + sorted)
let bs: BTreeSet<T>     = vec.into_iter().collect();

// Vec pasangan ke HashMap
let hm: HashMap<K,V>    = pairs.into_iter().collect();

// HashMap ke BTreeMap (sorted)
let bm: BTreeMap<K,V>   = hashmap.into_iter().collect();

// Ke BinaryHeap
let bh: BinaryHeap<T>   = vec.into_iter().collect();

// Ke VecDeque
let vd: VecDeque<T>     = vec.into();
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu projek yang gabungkan beberapa collections:

1. **Social Network** — HashMap user, HashSet followers, BTreeMap trending topics
2. **Airline Booking** — BTreeMap penerbangan by masa, HashMap seat reservations
3. **Text Search Engine** — HashMap word→Vec<position>, BTreeSet sorted results
4. **Job Queue System** — BinaryHeap priority queue, HashMap job status tracking
5. **File System Mini** — BTreeMap directory tree, HashSet permissions

---

*Collections in Rust — pilih yang tepat, tulis kod yang laju dan selamat.* 🦀
