# 🗂️ Queue Operations dalam Rust — Panduan Lengkap

> VecDeque, mpsc channels, BinaryHeap, dan real-world job queues.
> Dari FIFO asas hingga priority queue dan async work queues.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Kenapa Queue?

```
Queue = barisan giliran
  FIFO: First In, First Out

Guna kes:
  ✔ Task scheduling — proses kerja mengikut urutan
  ✔ Message passing — komunikasi antara thread
  ✔ Rate limiting — kawal kadar pemprosesan
  ✔ BFS (Breadth-First Search) — algoritma graf
  ✔ Print spooling — hantar tugas ke printer
  ✔ Event driven system — process events secara beratur

Jenis Queue dalam Rust:
  VecDeque<T>           → FIFO standard (single-thread)
  mpsc::channel         → Multi-producer, single-consumer
  sync::channel         → Bounded mpsc
  crossbeam-channel     → Flexible multi-producer, multi-consumer
  BinaryHeap<T>         → Priority queue (max-heap)
  tokio::mpsc           → Async queue
```

---

## Peta Pembelajaran

```
Bahagian 1  → VecDeque — Queue Asas
Bahagian 2  → Operasi Queue Lengkap
Bahagian 3  → mpsc Channel — Thread-safe Queue
Bahagian 4  → Bounded Channel
Bahagian 5  → BinaryHeap — Priority Queue
Bahagian 6  → Circular Buffer
Bahagian 7  → Async Queue dengan Tokio
Bahagian 8  → Worker Pool Pattern
Bahagian 9  → Real-world: Rate Limiter Queue
Bahagian 10 → Mini Project: KADA Job Queue
```

---

# BAHAGIAN 1: VecDeque — Queue Asas 📋

## Setup & Konsep

```rust
use std::collections::VecDeque;

fn main() {
    // VecDeque = "Vector Double-Ended Queue"
    // Boleh tambah/buang dari KEDUA-DUA hujung dengan cekap
    // Dalaman: ring buffer yang boleh berkembang

    // ── Buat VecDeque ─────────────────────────────────────────
    let mut barisan: VecDeque<String> = VecDeque::new();

    // Dengan kapasiti awal (optional, untuk performance)
    let mut barisan2: VecDeque<i32> = VecDeque::with_capacity(10);

    // Dari Vec
    let mut barisan3: VecDeque<i32> = vec![1, 2, 3].into();

    // ── Visualisasi ───────────────────────────────────────────
    // DEPAN                              BELAKANG
    //   ↓                                  ↓
    // [ Ali ] [ Siti ] [ Amin ] [ Zara ]
    //   ↑ push_front / pop_front    push_back / pop_back ↑

    // ── Operasi FIFO (Queue) ──────────────────────────────────
    let mut giliran: VecDeque<&str> = VecDeque::new();

    // Masuk barisan (dari belakang)
    giliran.push_back("Ali");
    giliran.push_back("Siti");
    giliran.push_back("Amin");

    println!("Barisan: {:?}", giliran); // ["Ali", "Siti", "Amin"]

    // Keluar barisan (dari depan) — FIFO!
    let pertama = giliran.pop_front(); // Some("Ali")
    println!("Dilayan: {:?}", pertama);
    println!("Tinggal: {:?}", giliran); // ["Siti", "Amin"]

    // ── Operasi LIFO (Stack) ──────────────────────────────────
    let mut timbunan: VecDeque<i32> = VecDeque::new();
    timbunan.push_back(1); // push ke atas
    timbunan.push_back(2);
    timbunan.push_back(3);

    let atas = timbunan.pop_back(); // Some(3) — LIFO!

    // ── Semak ─────────────────────────────────────────────────
    println!("Kosong: {}", giliran.is_empty());
    println!("Saiz:   {}", giliran.len());
    println!("Depan:  {:?}", giliran.front()); // peek, tidak buang
    println!("Belakang: {:?}", giliran.back());
}
```

---

## 🧠 Brain Teaser #1

Apakah output dan kenapa `VecDeque` lebih baik dari `Vec` untuk queue?

```rust
use std::collections::VecDeque;
use std::time::Instant;

fn main() {
    let n = 100_000;

    // Cara A: Vec sebagai queue
    let mula = Instant::now();
    let mut vec_queue: Vec<i32> = Vec::new();
    for i in 0..n { vec_queue.push(i); }
    for _ in 0..n { vec_queue.remove(0); } // pop dari depan!
    println!("Vec:     {:?}", mula.elapsed());

    // Cara B: VecDeque sebagai queue
    let mula = Instant::now();
    let mut deque: VecDeque<i32> = VecDeque::new();
    for i in 0..n { deque.push_back(i); }
    for _ in 0..n { deque.pop_front(); }
    println!("VecDeque: {:?}", mula.elapsed());
}
```

<details>
<summary>👀 Jawapan</summary>

```
Vec:      ~50-100ms (bergantung pada komputer)
VecDeque: ~1-2ms

Kenapa Vec lambat untuk queue?
  vec.remove(0) = SHIFT semua elemen ke kiri!
  100,000 remove(0) = O(n²) operasi!
  Untuk setiap pop: [S,i,t,i,A,m,i,n] → shift semua kiri satu langkah

Kenapa VecDeque laju?
  Menggunakan RING BUFFER dalaman
  pop_front() = gerak pointer depan sahaja = O(1)!
  Tiada data yang perlu dialihkan

  Ring buffer visualization:
  Kapasiti: [_, _, A, B, C, D, _, _]
                    ↑ depan      ↑ belakang
  pop_front() = gerak depan ke kanan: [_, _, _, B, C, D, _, _]
  push_back() = gerak belakang ke kanan: [_, _, _, B, C, D, E, _]

Ringkasan:
  Vec untuk pop/push di BELAKANG = OK (O(1) amortized)
  Vec untuk pop/push di DEPAN = LAMBAT (O(n))
  VecDeque untuk kedua-dua hujung = LAJU (O(1) amortized)
```
</details>

---

# BAHAGIAN 2: Operasi Queue Lengkap 🔧

## Semua Method VecDeque

```rust
use std::collections::VecDeque;

fn main() {
    let mut q: VecDeque<i32> = VecDeque::new();
    q.push_back(1);
    q.push_back(2);
    q.push_back(3);
    q.push_front(0); // tambah di depan

    // q = [0, 1, 2, 3]

    // ── Baca ─────────────────────────────────────────────────
    println!("front:  {:?}", q.front());      // Some(0) — tidak buang
    println!("back:   {:?}", q.back());       // Some(3) — tidak buang
    println!("[1]:    {:?}", q.get(1));       // Some(1) — akses by index
    println!("[99]:   {:?}", q.get(99));      // None — selamat!
    // println!("{}", q[0]);                  // direct index — PANIC kalau OOB

    // ── Buang ────────────────────────────────────────────────
    let depan = q.pop_front(); // Some(0) — buang depan
    let belakang = q.pop_back(); // Some(3) — buang belakang
    // q = [1, 2]

    // ── Semak ────────────────────────────────────────────────
    println!("len:      {}", q.len());
    println!("empty:    {}", q.is_empty());
    println!("capacity: {}", q.capacity());
    println!("contains: {}", q.contains(&2));

    // ── Modifikasi ───────────────────────────────────────────
    q.retain(|&x| x > 0);      // buang yang tidak lulus syarat
    q.clear();                   // kosongkan semua

    // ── Rotate ───────────────────────────────────────────────
    let mut r: VecDeque<i32> = vec![1, 2, 3, 4, 5].into();
    r.rotate_left(2);  // [3, 4, 5, 1, 2] — rotate kiri 2 posisi
    println!("{:?}", r);

    r.rotate_right(1); // [2, 3, 4, 5, 1] — rotate kanan 1 posisi
    println!("{:?}", r);

    // ── Insert & Remove di tengah ────────────────────────────
    let mut m: VecDeque<i32> = vec![1, 2, 4, 5].into();
    m.insert(2, 3); // insert 3 pada index 2: [1, 2, 3, 4, 5]
    m.remove(0);    // buang index 0: [2, 3, 4, 5]

    // ── Convert ke Vec ────────────────────────────────────────
    let v: Vec<i32> = m.into_iter().collect();
    println!("{:?}", v);

    // ── Iterate ──────────────────────────────────────────────
    let q2: VecDeque<i32> = vec![10, 20, 30].into();
    for &val in &q2 { print!("{} ", val); }  // pinjam
    println!();

    for val in q2.into_iter() { print!("{} ", val); } // consume
    println!();

    // ── Drain ────────────────────────────────────────────────
    let mut d: VecDeque<i32> = vec![1, 2, 3, 4, 5].into();
    let dibuang: Vec<i32> = d.drain(1..3).collect(); // buang index 1,2
    println!("Dibuang: {:?}", dibuang); // [2, 3]
    println!("Tinggal: {:?}", d);       // [1, 4, 5]
}
```

---

## BFS dengan VecDeque

```rust
use std::collections::{VecDeque, HashSet};

// BFS (Breadth-First Search) — contoh klasik Queue
fn bfs(graf: &Vec<Vec<usize>>, mula: usize, sasaran: usize) -> Option<Vec<usize>> {
    let mut barisan: VecDeque<Vec<usize>> = VecDeque::new();
    let mut dilawati: HashSet<usize> = HashSet::new();

    barisan.push_back(vec![mula]);
    dilawati.insert(mula);

    while let Some(laluan) = barisan.pop_front() {
        let nod = *laluan.last().unwrap();

        if nod == sasaran {
            return Some(laluan); // Jumpa!
        }

        for &jiran in &graf[nod] {
            if !dilawati.contains(&jiran) {
                dilawati.insert(jiran);
                let mut laluan_baru = laluan.clone();
                laluan_baru.push(jiran);
                barisan.push_back(laluan_baru);
            }
        }
    }

    None // Tidak jumpa
}

fn main() {
    // Graf: 0→1, 0→2, 1→3, 2→3, 3→4
    let graf = vec![
        vec![1, 2],    // 0 → 1, 2
        vec![3],        // 1 → 3
        vec![3],        // 2 → 3
        vec![4],        // 3 → 4
        vec![],         // 4 (tiada jiran)
    ];

    match bfs(&graf, 0, 4) {
        Some(laluan) => println!("Laluan: {:?}", laluan), // [0, 1, 3, 4] atau [0, 2, 3, 4]
        None         => println!("Tiada laluan!"),
    }
}
```

---

# BAHAGIAN 3: mpsc Channel — Thread-safe Queue 📡

## std::sync::mpsc

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    // mpsc = multi-producer, single-consumer
    // tx = transmitter (pengirim) — boleh clone!
    // rx = receiver (penerima) — hanya satu

    let (tx, rx) = mpsc::channel::<String>();

    // ── Satu producer, satu consumer ─────────────────────────
    let tx1 = tx.clone();
    thread::spawn(move || {
        for i in 0..5 {
            let mesej = format!("Mesej #{}", i);
            tx1.send(mesej).unwrap();
            thread::sleep(Duration::from_millis(50));
        }
        // tx1 di-drop di sini → channel ditutup!
    });

    // Tutup tx original (kalau tidak, rx.recv() tidak akan selesai)
    drop(tx);

    // ── Terima mesej ─────────────────────────────────────────
    // Cara 1: recv() — BLOCK sampai ada mesej atau semua tx drop
    while let Ok(mesej) = rx.recv() {
        println!("Terima: {}", mesej);
    }
    println!("Channel ditutup, selesai!");

    // ── Multiple producers ────────────────────────────────────
    let (tx2, rx2) = mpsc::channel::<(usize, String)>();

    for i in 0..3 {
        let tx_klon = tx2.clone();
        thread::spawn(move || {
            let data = format!("Data dari thread {}", i);
            tx_klon.send((i, data)).unwrap();
        });
    }

    // Drop original transmitter
    drop(tx2);

    // Terima dari semua thread
    let mut hasil = Vec::new();
    while let Ok(item) = rx2.recv() {
        hasil.push(item);
    }
    hasil.sort_by_key(|(i, _)| *i); // sort ikut thread id
    for (i, data) in hasil {
        println!("Thread {}: {}", i, data);
    }
}
```

---

## try_recv vs recv vs iter

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel::<i32>();

    thread::spawn(move || {
        for i in 0..5 {
            tx.send(i).unwrap();
            thread::sleep(std::time::Duration::from_millis(100));
        }
    });

    // ── recv() — BLOCKING ─────────────────────────────────────
    // Block thread semasa sampai mesej ada
    let nilai = rx.recv().unwrap(); // tunggu mesej pertama
    println!("recv: {}", nilai);

    // ── try_recv() — NON-BLOCKING ─────────────────────────────
    // Return Err kalau tiada mesej (tidak block)
    thread::sleep(std::time::Duration::from_millis(200));
    loop {
        match rx.try_recv() {
            Ok(v)                                  => println!("try_recv: {}", v),
            Err(mpsc::TryRecvError::Empty)         => { println!("Kosong, tunggu..."); break; }
            Err(mpsc::TryRecvError::Disconnected)  => { println!("Disconnected!"); break; }
        }
    }

    // ── recv_timeout() — TIMEOUT ──────────────────────────────
    match rx.recv_timeout(std::time::Duration::from_millis(500)) {
        Ok(v)                                   => println!("Timeout recv: {}", v),
        Err(mpsc::RecvTimeoutError::Timeout)    => println!("Timeout!"),
        Err(mpsc::RecvTimeoutError::Disconnected) => println!("Disconnected!"),
    }

    // ── iter() — Iterator ─────────────────────────────────────
    // Iterate sampai semua tx di-drop
    // (berfungsi macam while let Ok(v) = rx.recv())
    for v in rx.iter() {
        println!("iter: {}", v);
    }

    println!("Channel habis!");
}
```

---

# BAHAGIAN 4: Bounded Channel (Sync) 🚦

## Kawal Bilangan Mesej Dalam Queue

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    // sync_channel(n) = bounded channel dengan kapasiti n
    // Pengirim BLOCK bila queue penuh!
    // Berguna untuk backpressure!

    let (tx, rx) = mpsc::sync_channel::<i32>(3); // max 3 mesej dalam queue

    let producer = thread::spawn(move || {
        for i in 0..7 {
            println!("Hantar #{}", i);
            tx.send(i).unwrap(); // BLOCK kalau queue penuh (>3)
            println!("Dihantar #{}", i);
        }
    });

    // Consumer lambat — simulasi processing
    thread::sleep(std::time::Duration::from_millis(500));
    while let Ok(v) = rx.try_recv() {
        println!("Proses: {}", v);
        thread::sleep(std::time::Duration::from_millis(100));
    }

    producer.join().unwrap();

    // ── Sync channel untuk rate limiting ──────────────────────
    // Producer yang laju diperlambatkan oleh consumer yang lambat
    // Ini adalah BACKPRESSURE!

    let (tx2, rx2) = mpsc::sync_channel::<String>(5);

    // Fast producer
    let prod = thread::spawn(move || {
        for i in 0..20 {
            let data = format!("Kerja #{}", i);
            match tx2.try_send(data) {
                Ok(_)                        => println!("  ✔ Dimasukkan #{}", i),
                Err(mpsc::TrySendError::Full(_))   => println!("  ✗ Queue penuh! #{} diabaikan", i),
                Err(mpsc::TrySendError::Disconnected(_)) => break,
            }
        }
    });

    // Slow consumer
    for v in rx2.iter().take(20) {
        println!("Memproses: {}", v);
        thread::sleep(std::time::Duration::from_millis(50));
    }

    prod.join().unwrap();
}
```

---

# BAHAGIAN 5: BinaryHeap — Priority Queue 🏆

## Queue dengan Keutamaan

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

fn main() {
    // BinaryHeap = MAX-HEAP secara default
    // Elemen TERBESAR keluar dulu!

    let mut timbunan: BinaryHeap<i32> = BinaryHeap::new();
    timbunan.push(3);
    timbunan.push(1);
    timbunan.push(4);
    timbunan.push(1);
    timbunan.push(5);

    // Pop keluar dari terbesar ke terkecil
    while let Some(val) = timbunan.pop() {
        print!("{} ", val); // 5 4 3 1 1
    }
    println!();

    // ── Min-heap menggunakan Reverse ──────────────────────────
    let mut min_heap: BinaryHeap<Reverse<i32>> = BinaryHeap::new();
    min_heap.push(Reverse(3));
    min_heap.push(Reverse(1));
    min_heap.push(Reverse(4));

    while let Some(Reverse(val)) = min_heap.pop() {
        print!("{} ", val); // 1 3 4
    }
    println!();

    // ── Custom Ord untuk struct ────────────────────────────────
    #[derive(Debug, Eq, PartialEq)]
    struct Tugasan {
        keutamaan: u32,  // LEBIH BESAR = lebih utama
        nama:      String,
    }

    impl Ord for Tugasan {
        fn cmp(&self, lain: &Self) -> std::cmp::Ordering {
            self.keutamaan.cmp(&lain.keutamaan) // max-heap by keutamaan
                .then(self.nama.cmp(&lain.nama)) // tie-break by nama
        }
    }

    impl PartialOrd for Tugasan {
        fn partial_cmp(&self, lain: &Self) -> Option<std::cmp::Ordering> {
            Some(self.cmp(lain))
        }
    }

    let mut antrian: BinaryHeap<Tugasan> = BinaryHeap::new();
    antrian.push(Tugasan { keutamaan: 3, nama: "Laporan".into() });
    antrian.push(Tugasan { keutamaan: 1, nama: "Emel".into() });
    antrian.push(Tugasan { keutamaan: 5, nama: "Mesyuarat".into() });
    antrian.push(Tugasan { keutamaan: 2, nama: "Data entry".into() });

    println!("\nTugasan mengikut keutamaan:");
    while let Some(t) = antrian.pop() {
        println!("  [{}] {}", t.keutamaan, t.nama);
    }
    // [5] Mesyuarat
    // [3] Laporan
    // [2] Data entry
    // [1] Emel
}
```

---

## Priority Queue dengan Masa Tamat

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;
use std::time::{Duration, Instant};

#[derive(Debug, Eq, PartialEq)]
struct TugasanBerMasa {
    tamat_pada: Reverse<Instant>, // Reverse = yang PALING AWAL tamat keluar dulu
    keutamaan:  u32,
    kerja:      String,
}

impl Ord for TugasanBerMasa {
    fn cmp(&self, lain: &Self) -> std::cmp::Ordering {
        self.tamat_pada.cmp(&lain.tamat_pada)
            .then(self.keutamaan.cmp(&lain.keutamaan))
    }
}

impl PartialOrd for TugasanBerMasa {
    fn partial_cmp(&self, lain: &Self) -> Option<std::cmp::Ordering> {
        Some(self.cmp(lain))
    }
}

fn main() {
    let mut antrian: BinaryHeap<TugasanBerMasa> = BinaryHeap::new();
    let sekarang = Instant::now();

    antrian.push(TugasanBerMasa {
        tamat_pada: Reverse(sekarang + Duration::from_secs(30)),
        keutamaan:  5,
        kerja:      "Urgent report".into(),
    });
    antrian.push(TugasanBerMasa {
        tamat_pada: Reverse(sekarang + Duration::from_secs(10)),
        keutamaan:  3,
        kerja:      "Quick email".into(),
    });
    antrian.push(TugasanBerMasa {
        tamat_pada: Reverse(sekarang + Duration::from_secs(60)),
        keutamaan:  1,
        kerja:      "Low priority task".into(),
    });

    // Yang paling awal tamat keluar dulu
    while let Some(t) = antrian.pop() {
        println!("Proses: {} (keutamaan: {})", t.kerja, t.keutamaan);
    }
}
```

---

# BAHAGIAN 6: Circular Buffer 🔄

## Ring Buffer Manual

```rust
struct RingBuffer<T> {
    data:     Vec<Option<T>>,
    depan:    usize,
    belakang: usize,
    saiz:     usize,
    kapasiti: usize,
}

impl<T: std::fmt::Debug + Clone> RingBuffer<T> {
    fn baru(kapasiti: usize) -> Self {
        RingBuffer {
            data:     vec![None; kapasiti + 1],
            depan:    0,
            belakang: 0,
            saiz:     0,
            kapasiti,
        }
    }

    fn penuh(&self) -> bool { self.saiz == self.kapasiti }
    fn kosong(&self) -> bool { self.saiz == 0 }

    fn masuk(&mut self, nilai: T) -> Result<(), &'static str> {
        if self.penuh() { return Err("Buffer penuh!"); }
        self.data[self.belakang] = Some(nilai);
        self.belakang = (self.belakang + 1) % (self.kapasiti + 1);
        self.saiz += 1;
        Ok(())
    }

    fn keluar(&mut self) -> Option<T> {
        if self.kosong() { return None; }
        let nilai = self.data[self.depan].take();
        self.depan = (self.depan + 1) % (self.kapasiti + 1);
        self.saiz -= 1;
        nilai
    }

    fn tengok_depan(&self) -> Option<&T> {
        if self.kosong() { None } else { self.data[self.depan].as_ref() }
    }
}

fn main() {
    let mut rb: RingBuffer<i32> = RingBuffer::baru(3);

    rb.masuk(1).unwrap();
    rb.masuk(2).unwrap();
    rb.masuk(3).unwrap();
    assert!(rb.masuk(4).is_err()); // penuh!

    println!("Depan: {:?}", rb.tengok_depan()); // Some(1)
    println!("{:?}", rb.keluar()); // Some(1)
    println!("{:?}", rb.keluar()); // Some(2)

    rb.masuk(4).unwrap(); // sekarang ada ruang
    println!("{:?}", rb.keluar()); // Some(3)
    println!("{:?}", rb.keluar()); // Some(4)
    println!("{:?}", rb.keluar()); // None
}
```

---

# BAHAGIAN 7: Async Queue dengan Tokio ⚡

## tokio::sync::mpsc

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // ── Unbounded channel (tidak ada had) ─────────────────────
    let (tx, mut rx) = mpsc::unbounded_channel::<String>();

    // Producer tasks
    for i in 0..3 {
        let tx_klon = tx.clone();
        tokio::spawn(async move {
            for j in 0..3 {
                let mesej = format!("Task {}-{}", i, j);
                tx_klon.send(mesej).unwrap();
                sleep(Duration::from_millis(10)).await;
            }
        });
    }

    drop(tx); // tutup semua sender asli

    // Consumer
    while let Some(mesej) = rx.recv().await {
        println!("Terima: {}", mesej);
    }

    // ── Bounded channel (ada had) ─────────────────────────────
    let (tx2, mut rx2) = mpsc::channel::<i32>(5); // max 5 dalam queue

    tokio::spawn(async move {
        for i in 0..10 {
            // .send() adalah async — akan .await bila penuh
            tx2.send(i).await.unwrap();
            println!("Hantar: {}", i);
        }
    });

    for _ in 0..10 {
        if let Some(v) = rx2.recv().await {
            println!("Proses: {}", v);
            sleep(Duration::from_millis(20)).await; // lambat sikit
        }
    }

    // ── Broadcast — satu pengirim, banyak penerima ────────────
    let (tx3, _) = tokio::sync::broadcast::channel::<String>(16);

    for i in 0..3 {
        let mut rx_i = tx3.subscribe();
        tokio::spawn(async move {
            while let Ok(mesej) = rx_i.recv().await {
                println!("Penerima {}: {}", i, mesej);
            }
        });
    }

    tx3.send("Hello semua!".into()).unwrap();
    tx3.send("Mesej broadcast".into()).unwrap();

    sleep(Duration::from_millis(100)).await;
}
```

---

## 🧠 Brain Teaser #2

Apakah perbezaan antara `mpsc::channel` (std) dan `tokio::sync::mpsc::channel`?

<details>
<summary>👀 Jawapan</summary>

```
std::sync::mpsc:
  ✔ Synchronous — boleh guna dalam non-async code
  ✔ recv() BLOCK thread semasa (tidak boleh .await)
  ✔ try_recv() non-blocking (return Err kalau kosong)
  ✗ Tidak boleh guna dalam async context (block executor thread!)
  ✗ Tidak ada timeout built-in (perlu crossbeam atau thread spawn)

tokio::sync::mpsc:
  ✔ Asynchronous — .await tidak block thread, yield ke executor
  ✔ recv().await — cekap, tidak block executor thread
  ✔ try_recv() juga ada (non-blocking)
  ✔ timeout dengan tokio::time::timeout
  ✔ send().await boleh tunggu kalau queue penuh (backpressure)
  ✗ Hanya boleh guna dalam async context (dengan tokio runtime)

Bila guna mana:
  Synchronous code       → std::sync::mpsc
  Async code (Tokio)     → tokio::sync::mpsc
  Multiple receivers     → tokio::sync::broadcast
  Oneshot (sekali)       → tokio::sync::oneshot

Analogi:
  std::mpsc    = telefon biasa (sambungan langsung, kena tunggu)
  tokio::mpsc  = WhatsApp (hantar dan buat kerja lain, dapat notifikasi bila reply)
```
</details>

---

# BAHAGIAN 8: Worker Pool Pattern 👷

## Thread Pool dengan Queue

```rust
use std::sync::{Arc, Mutex};
use std::sync::mpsc;
use std::thread;

type Kerja = Box<dyn FnOnce() + Send + 'static>;

struct KoleksiThread {
    pekerja:  Vec<thread::JoinHandle<()>>,
    pengirim: mpsc::Sender<Option<Kerja>>,
}

impl KoleksiThread {
    fn baru(bilangan: usize) -> Self {
        let (tx, rx) = mpsc::channel::<Option<Kerja>>();
        let rx = Arc::new(Mutex::new(rx));
        let mut pekerja = Vec::new();

        for id in 0..bilangan {
            let rx_klon = Arc::clone(&rx);
            let handle = thread::spawn(move || {
                loop {
                    let kerja = rx_klon.lock().unwrap().recv().unwrap();
                    match kerja {
                        Some(f) => {
                            println!("Worker {} memproses kerja", id);
                            f();
                        }
                        None => {
                            println!("Worker {} berhenti", id);
                            break;
                        }
                    }
                }
            });
            pekerja.push(handle);
        }

        KoleksiThread { pekerja, pengirim: tx }
    }

    fn hantar<F>(&self, f: F)
    where F: FnOnce() + Send + 'static {
        self.pengirim.send(Some(Box::new(f))).unwrap();
    }
}

impl Drop for KoleksiThread {
    fn drop(&mut self) {
        // Hantar signal berhenti kepada setiap worker
        for _ in &self.pekerja {
            self.pengirim.send(None).unwrap();
        }
        // Tunggu semua worker selesai
        for worker in self.pekerja.drain(..) {
            worker.join().unwrap();
        }
    }
}

fn main() {
    let pool = KoleksiThread::baru(4);

    for i in 0..8 {
        pool.hantar(move || {
            println!("Kerja {} diproses oleh thread {:?}", i, thread::current().id());
            thread::sleep(std::time::Duration::from_millis(50));
        });
    }

    // pool di-drop di sini → Drop dipanggil → tunggu semua kerja selesai
    println!("Semua kerja dihantar!");
}
```

---

# BAHAGIAN 9: Real-world — Rate Limiter Queue 🚦

## Token Bucket dengan Queue

```rust
use std::collections::VecDeque;
use std::time::{Duration, Instant};
use std::sync::{Arc, Mutex};
use std::thread;

struct TokenBucket {
    token:       u32,          // token semasa
    kapasiti:    u32,          // token maksimum
    kadar:       f64,          // token per saat
    masa_terakhir: Instant,
}

impl TokenBucket {
    fn baru(kapasiti: u32, kadar_per_saat: f64) -> Self {
        TokenBucket {
            token: kapasiti,
            kapasiti,
            kadar: kadar_per_saat,
            masa_terakhir: Instant::now(),
        }
    }

    fn minta_token(&mut self) -> bool {
        // Tambah token berdasarkan masa berlalu
        let berlalu = self.masa_terakhir.elapsed().as_secs_f64();
        let token_baru = (berlalu * self.kadar).floor() as u32;

        if token_baru > 0 {
            self.token = (self.token + token_baru).min(self.kapasiti);
            self.masa_terakhir = Instant::now();
        }

        if self.token > 0 {
            self.token -= 1;
            true
        } else {
            false
        }
    }
}

struct AntrianKadar<T> {
    barisan:  VecDeque<T>,
    had:      Arc<Mutex<TokenBucket>>,
}

impl<T: std::fmt::Debug> AntrianKadar<T> {
    fn baru(kapasiti_antrian: usize, kadar_per_saat: f64, burst: u32) -> Self {
        AntrianKadar {
            barisan: VecDeque::with_capacity(kapasiti_antrian),
            had:     Arc::new(Mutex::new(TokenBucket::baru(burst, kadar_per_saat))),
        }
    }

    fn masukkan(&mut self, item: T) -> bool {
        if self.barisan.len() < self.barisan.capacity() {
            self.barisan.push_back(item);
            true
        } else {
            eprintln!("Antrian penuh! {:?} diabaikan", item);
            false
        }
    }

    fn proses_satu(&mut self) -> Option<T> {
        if self.had.lock().unwrap().minta_token() {
            self.barisan.pop_front()
        } else {
            None // kadar had dilampaui
        }
    }
}

fn main() {
    // 5 requests per saat, burst sehingga 10
    let mut antrian = AntrianKadar::new(100, 5.0, 10);

    // Masukkan banyak request
    for i in 0..15 {
        antrian.masukkan(format!("Request #{}", i));
    }

    println!("Antrian: {} item", antrian.barisan.len());

    // Proses dengan kadar had
    let mut diproses = 0;
    loop {
        match antrian.proses_satu() {
            Some(item) => {
                println!("✔ Diproses: {}", item);
                diproses += 1;
                thread::sleep(Duration::from_millis(100));
            }
            None if antrian.barisan.is_empty() => {
                println!("Semua selesai! {} diproses", diproses);
                break;
            }
            None => {
                println!("Had kadar - tunggu...");
                thread::sleep(Duration::from_millis(200));
            }
        }
    }
}
```

---

# BAHAGIAN 10: Mini Project — KADA Job Queue 🏗️

```toml
[dependencies]
tokio    = { version = "1", features = ["full"] }
serde    = { version = "1", features = ["derive"] }
uuid     = { version = "1", features = ["v4"] }
chrono   = "0.4"
```

```rust
use tokio::sync::mpsc;
use std::sync::Arc;
use tokio::sync::RwLock;
use std::collections::HashMap;
use uuid::Uuid;
use chrono::Utc;
use serde::{Serialize, Deserialize};

// ─── Jenis Kerja ───────────────────────────────────────────────

#[derive(Debug, Clone, Serialize, Deserialize)]
enum JenisKerja {
    HantarEmel     { kepada: String, subjek: String },
    JanaPDF        { laporan_id: u32, format: String },
    SegerakData    { sumber: String, destinasi: String },
    HantarNotifikasi { pekerja_id: u32, mesej: String },
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
enum StatusKerja {
    Menunggu,
    Memproses,
    Selesai,
    Gagal { sebab: String },
}

#[derive(Debug, Clone, Serialize)]
struct Kerja {
    id:          String,
    jenis:       JenisKerja,
    keutamaan:   u8,          // 1-10 (10 = paling utama)
    dicipta_pada: String,
    status:      StatusKerja,
    cuba_ke:     u8,
    max_cuba:    u8,
}

impl Kerja {
    fn baru(jenis: JenisKerja, keutamaan: u8) -> Self {
        Kerja {
            id:           Uuid::new_v4().to_string(),
            jenis,
            keutamaan,
            dicipta_pada: Utc::now().to_rfc3339(),
            status:       StatusKerja::Menunggu,
            cuba_ke:      0,
            max_cuba:     3,
        }
    }
}

// ─── Queue Engine ──────────────────────────────────────────────

#[derive(Clone)]
struct QueueEngine {
    pengirim:     mpsc::Sender<Kerja>,
    rekod:        Arc<RwLock<HashMap<String, Kerja>>>,
}

impl QueueEngine {
    fn baru(bilangan_pekerja: usize) -> (Self, tokio::task::JoinHandle<()>) {
        let (tx, rx) = mpsc::channel::<Kerja>(100);
        let rekod    = Arc::new(RwLock::new(HashMap::new()));
        let rekod_klon = Arc::clone(&rekod);

        let engine = QueueEngine {
            pengirim: tx,
            rekod:    Arc::clone(&rekod),
        };

        // Mulakan worker pool
        let handle = tokio::spawn(async move {
            proses_antrian(rx, rekod_klon, bilangan_pekerja).await;
        });

        (engine, handle)
    }

    async fn hantar(&self, kerja: Kerja) -> Result<String, String> {
        let id = kerja.id.clone();

        // Simpan dalam rekod
        self.rekod.write().await.insert(id.clone(), kerja.clone());

        // Hantar ke antrian
        self.pengirim.send(kerja).await
            .map_err(|e| format!("Gagal hantar ke antrian: {}", e))?;

        Ok(id)
    }

    async fn status(&self, id: &str) -> Option<StatusKerja> {
        self.rekod.read().await
            .get(id)
            .map(|k| k.status.clone())
    }

    async fn senarai(&self) -> Vec<Kerja> {
        self.rekod.read().await.values().cloned().collect()
    }
}

// ─── Worker Pool ───────────────────────────────────────────────

async fn proses_antrian(
    mut rx:   mpsc::Receiver<Kerja>,
    rekod:    Arc<RwLock<HashMap<String, Kerja>>>,
    pekerja:  usize,
) {
    // Semaphore untuk had bilangan kerja serentak
    let semaphore = Arc::new(tokio::sync::Semaphore::new(pekerja));

    while let Some(mut kerja) = rx.recv().await {
        let sem = Arc::clone(&semaphore);
        let rekod_klon = Arc::clone(&rekod);

        tokio::spawn(async move {
            let _permit = sem.acquire().await.unwrap(); // had concurrent tasks

            // Update status → Memproses
            rekod_klon.write().await
                .entry(kerja.id.clone())
                .and_modify(|k| k.status = StatusKerja::Memproses);

            println!("[{}] Mula proses: {:?}", kerja.id, kerja.jenis);

            // Proses kerja
            let hasil = laksana_kerja(&kerja.jenis).await;

            // Update status akhir
            let status_akhir = match hasil {
                Ok(_) => {
                    println!("[{}] ✔ Selesai", kerja.id);
                    StatusKerja::Selesai
                }
                Err(e) => {
                    kerja.cuba_ke += 1;
                    if kerja.cuba_ke < kerja.max_cuba {
                        println!("[{}] ✗ Gagal (cuba {}/{}): {}", kerja.id, kerja.cuba_ke, kerja.max_cuba, e);
                        StatusKerja::Menunggu // akan dicuba semula
                    } else {
                        println!("[{}] ✗ GAGAL MUKTAMAD: {}", kerja.id, e);
                        StatusKerja::Gagal { sebab: e }
                    }
                }
            };

            rekod_klon.write().await
                .entry(kerja.id.clone())
                .and_modify(|k| {
                    k.status  = status_akhir;
                    k.cuba_ke = kerja.cuba_ke;
                });
        });
    }
}

async fn laksana_kerja(jenis: &JenisKerja) -> Result<(), String> {
    match jenis {
        JenisKerja::HantarEmel { kepada, subjek } => {
            tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
            println!("  → Emel dihantar ke {}: {}", kepada, subjek);
            Ok(())
        }
        JenisKerja::JanaPDF { laporan_id, format } => {
            tokio::time::sleep(tokio::time::Duration::from_millis(200)).await;
            println!("  → PDF laporan {} dijanakan ({})", laporan_id, format);
            Ok(())
        }
        JenisKerja::SegerakData { sumber, destinasi } => {
            tokio::time::sleep(tokio::time::Duration::from_millis(300)).await;
            // Simulasi kegagalan rawak
            if rand_bool() {
                return Err("Sambungan gagal semasa segerak".into());
            }
            println!("  → Data disegerak dari {} ke {}", sumber, destinasi);
            Ok(())
        }
        JenisKerja::HantarNotifikasi { pekerja_id, mesej } => {
            tokio::time::sleep(tokio::time::Duration::from_millis(50)).await;
            println!("  → Notifikasi ke pekerja {}: {}", pekerja_id, mesej);
            Ok(())
        }
    }
}

fn rand_bool() -> bool {
    use std::time::{SystemTime, UNIX_EPOCH};
    SystemTime::now().duration_since(UNIX_EPOCH).unwrap().subsec_nanos() % 3 == 0
}

// ─── Demo ──────────────────────────────────────────────────────

#[tokio::main]
async fn main() {
    println!("{'═'*55}");
    println!("{:^55}", "KADA Job Queue System");
    println!("{'═'*55}\n");

    // Mulakan queue dengan 3 worker serentak
    let (queue, _handle) = QueueEngine::baru(3);

    // Hantar pelbagai jenis kerja
    let mut ids = Vec::new();

    ids.push(queue.hantar(Kerja::baru(
        JenisKerja::HantarEmel {
            kepada: "ali@kada.gov.my".into(),
            subjek: "Laporan Gaji Januari".into(),
        }, 5
    )).await.unwrap());

    ids.push(queue.hantar(Kerja::baru(
        JenisKerja::JanaPDF { laporan_id: 101, format: "A4".into() }, 8
    )).await.unwrap());

    ids.push(queue.hantar(Kerja::baru(
        JenisKerja::SegerakData {
            sumber:     "sistem_lama".into(),
            destinasi:  "sistem_baru".into(),
        }, 3
    )).await.unwrap());

    ids.push(queue.hantar(Kerja::baru(
        JenisKerja::HantarNotifikasi { pekerja_id: 1001, mesej: "Gaji telah dikreditkan".into() }, 9
    )).await.unwrap());

    ids.push(queue.hantar(Kerja::baru(
        JenisKerja::HantarEmel {
            kepada: "ketua@kada.gov.my".into(),
            subjek: "Ringkasan Bulanan".into(),
        }, 6
    )).await.unwrap());

    println!("{} kerja dihantar ke antrian\n", ids.len());

    // Tunggu kerja selesai
    tokio::time::sleep(tokio::time::Duration::from_secs(2)).await;

    // Laporan status
    println!("\n{'─'*55}");
    println!("{:^55}", "Status Akhir");
    println!("{'─'*55}");

    let semua = queue.senarai().await;
    let mut selesai = 0;
    let mut gagal = 0;
    let mut proses = 0;

    for kerja in &semua {
        let ikon = match &kerja.status {
            StatusKerja::Selesai    => { selesai += 1; "✔" }
            StatusKerja::Gagal {..} => { gagal += 1;  "✗" }
            StatusKerja::Memproses  => { proses += 1; "⟳" }
            StatusKerja::Menunggu   => "⌛",
        };
        println!("{} {:8} Keutamaan:{} Cuba:{}/{}",
            ikon, &kerja.id[..8],
            kerja.keutamaan, kerja.cuba_ke, kerja.max_cuba);
    }

    println!("\nRingkasan: {} selesai, {} gagal, {} masih proses",
        selesai, gagal, proses);
    println!("{'═'*55}");
}
```

---

# 📋 Rujukan Pantas — Queue Cheat Sheet

## Pilih Queue Yang Betul

```
Keperluan                          Guna
──────────────────────────────────────────────────────
FIFO, single-thread               VecDeque<T>
FIFO, multi-thread (mpsc)         std::sync::mpsc
FIFO, async mpsc                  tokio::sync::mpsc
Bounded queue (backpressure)      mpsc::sync_channel(n)
Broadcast (1 → banyak)            tokio::sync::broadcast
Oneshot (hantar sekali)           tokio::sync::oneshot
Priority (yang utama dulu)        BinaryHeap<T>
Min-heap                          BinaryHeap<Reverse<T>>
Fixed-size circular               VecDeque dengan kapasiti
BFS / traversal                   VecDeque<T>
```

## Operasi VecDeque

```rust
// FIFO
q.push_back(item)    // enqueue
q.pop_front()        // dequeue → Option<T>
q.front()            // peek → Option<&T>

// LIFO (stack)
q.push_back(item)    // push
q.pop_back()         // pop → Option<T>

// Semak
q.len()              // bilangan item
q.is_empty()         // kosong?
q.contains(&item)    // ada?
q.get(i)             // by index → Option<&T>

// Modifikasi
q.retain(|x| ...)    // filter in-place
q.clear()            // kosongkan
q.rotate_left(n)     // rotate
```

## Operasi Channel

```rust
// Buat
let (tx, rx) = mpsc::channel();           // unbounded
let (tx, rx) = mpsc::sync_channel(5);     // bounded=5
let (tx, rx) = mpsc::unbounded_channel(); // tokio async

// Hantar
tx.send(item).unwrap()                    // sync, panic kalau gagal
tx.try_send(item)?                        // sync non-blocking
tx.send(item).await?                      // async

// Terima
rx.recv().unwrap()                        // sync, block
rx.try_recv()                             // sync, non-block
rx.recv().await                           // async
for item in rx.iter() { ... }            // iterate sampai habis
```

---

## 🏆 Cabaran Akhir

Cuba implement salah satu:

1. **Printer Queue** — simulate printer dengan antrian dokumen, priority untuk "urgent"
2. **Message Broker** — topic-based pub/sub system dengan multiple subscribers
3. **Dedup Queue** — queue yang buang duplikat sebelum masuk queue
4. **Delayed Queue** — item hanya boleh diproses selepas delay tertentu
5. **Dead Letter Queue** — item yang gagal berulang kali dihantar ke DLQ untuk semakan

---

*Queue = kawalan aliran data yang tersusun.*
*VecDeque untuk single-thread, mpsc untuk multi-thread, tokio untuk async.* 🦀
