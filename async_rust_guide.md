# ⚡ Async Rust — Panduan Lengkap

> Dari mindset async hingga production systems.
> Faham bukan sekadar syntax — faham *mengapa* dan *bagaimana*.

---

## Kandungan

| # | Topik |
|---|-------|
| 1 | [The Async Mindset](#1-the-async-mindset) |
| 2 | [Futures Explained Clearly](#2-futures-explained-clearly) |
| 3 | [async dan .await](#3-async-dan-await) |
| 4 | [Executors dan Runtimes](#4-executors-dan-runtimes) |
| 5 | [Task Spawning dan Orchestration](#5-task-spawning-dan-orchestration) |
| 6 | [Blocking Pitfalls dan Performance Traps](#6-blocking-pitfalls-dan-performance-traps) |
| 7 | [Shared State, Channels, dan Synchronization](#7-shared-state-channels-dan-synchronization) |
| 8 | [Streams dan Async Data Flow](#8-streams-dan-async-data-flow) |
| 9 | [Cancellation dan Graceful Shutdown](#9-cancellation-dan-graceful-shutdown) |
| 10 | [Testing Async Rust](#10-testing-async-rust) |
| 11 | [Production-Oriented Guidance](#11-production-oriented-guidance) |

---

# 1. The Async Mindset

## Kenapa Async Wujud?

```
Masalah asas: I/O adalah LAMBAT berbanding CPU

Masa operasi (anggaran):
  CPU instruction      →  0.3 nanosaat
  L1 cache access      →  1 nanosaat
  RAM access           →  100 nanosaat
  SSD read             →  100,000 nanosaat  (0.1ms)
  Network (same DC)    →  500,000 nanosaat  (0.5ms)
  Network (internet)   →  150,000,000 ns    (150ms)
  Database query       →  1,000,000+ ns     (1ms+)

Semasa tunggu network → CPU duduk kosong.
Async = manfaatkan masa menunggu itu.
```

## Tiga Pilihan: Blocking, Thread, Async

```rust
// ── Cara 1: BLOCKING (synchronous) ───────────────────────────
// Mudah ditulis, tapi thread tersekat semasa tunggu
fn proses_blocking() {
    let data = std::fs::read_to_string("fail.txt").unwrap(); // BLOCK!
    println!("{}", data);
    // Thread tidak boleh buat apa-apa semasa baca fail
}

// ── Cara 2: THREADS (OS threads) ─────────────────────────────
// Setiap request dapat thread sendiri
// Masalah: setiap thread pakai ~2MB stack
// 10,000 request = 20GB RAM hanya untuk stack!
fn proses_dengan_thread() {
    let handle = std::thread::spawn(|| {
        let data = std::fs::read_to_string("fail.txt").unwrap();
        println!("{}", data);
    });
    handle.join().unwrap();
}

// ── Cara 3: ASYNC ─────────────────────────────────────────────
// Satu thread boleh handle ribuan request serentak
// Semasa tunggu I/O, thread buat kerja lain
async fn proses_async() {
    let data = tokio::fs::read_to_string("fail.txt").await.unwrap();
    println!("{}", data);
    // Semasa `await`, thread bebas handle request lain!
}
```

## Bila Guna Apa?

```
BLOCKING (std biasa):
  ✔ Skrip CLI, tools mudah
  ✔ CPU-bound work (computation intensif)
  ✔ Bila simplicity lebih penting dari performance
  ✔ Prototype awal
  ✗ Jangan: server dengan banyak concurrent connections

THREADS (std::thread):
  ✔ CPU-bound parallelism (rayon, compute)
  ✔ Kerja yang perlu isolation sebenar
  ✔ Legacy code integration
  ✗ Jangan: ribuan concurrent I/O operations

ASYNC (tokio, async-std):
  ✔ I/O-bound workload (network, database, file)
  ✔ Web servers, API servers
  ✔ Ribuan concurrent connections
  ✔ Microservices
  ✗ Jangan: CPU-heavy work dalam async context
  ✗ Jangan: kalau async complexity tidak berbaloi
```

## Model Mental Async

```
Synchronous:    [Task A ====blocking====] [Task B ====blocking====]
                ─────────────────────────────────────────────────→ masa

Thread per req: Thread 1: [Task A =====] (idle selama I/O)
                Thread 2: [Task B =====] (idle selama I/O)
                Thread 3: [Task C =====] (idle selama I/O)
                ─────────────────────────────────────────────────→ masa

Async:          Thread 1:  [A-work][B-work][C-work][A-work][A-done]
                             ↑         ↑         ↑
                           A tunggu  B tunggu  C tunggu I/O
                ─────────────────────────────────────────────────→ masa
                Satu thread, banyak task, tiada idle!
```

---

# 2. Futures Explained Clearly

## Apa itu Future?

```rust
// Future = "janji nilai yang akan ada pada masa hadapan"
// Ia LAZY — tidak buat apa-apa sampai di-poll

// Trait Future (dari std):
// pub trait Future {
//     type Output;
//     fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
// }

// Poll ada dua nilai:
// enum Poll<T> {
//     Ready(T),    // selesai, ada nilai
//     Pending,     // belum selesai, cuba lagi kemudian
// }
```

## Future adalah LAZY

```rust
use tokio::time::{sleep, Duration};

async fn lambat() -> String {
    sleep(Duration::from_secs(1)).await;
    "selesai".to_string()
}

fn main() {
    // ❌ SALAH FAHAM: ini TIDAK mula buat apa-apa!
    let future = lambat(); // hanya cipta Future object

    // Future tidak berjalan sampai di-await atau di-spawn!
    println!("Future dicipta, tapi belum berjalan");

    // ✔ Perlu executor untuk jalankan:
    let rt = tokio::runtime::Runtime::new().unwrap();
    let hasil = rt.block_on(lambat()); // BARU berjalan
    println!("{}", hasil);
}
```

## Bagaimana Polling Berfungsi?

```rust
// Visualisasi proses polling:
//
// Executor:  "Future A, adakah kau selesai?"
// Future A:  Poll::Pending  ← "Belum, tunggu network"
//
// Executor:  "Future B, adakah kau selesai?"
// Future B:  Poll::Pending  ← "Belum, tunggu database"
//
// [Network response tiba untuk A]
// Waker:     "Executor! Future A dah boleh dilayan!"
//
// Executor:  "Future A, adakah kau selesai?"
// Future A:  Poll::Ready("data")  ← "Ya! Ini hasilnya"

// Waker adalah kunci — ia memberitahu executor bila nak poll semula
// Tanpa waker yang betul, Future akan "hilang" dalam queue

use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

// Contoh Future mudah dari scratch
struct TungguDua {
    mula: Instant,
}

impl TungguDua {
    fn new() -> Self {
        TungguDua { mula: Instant::now() }
    }
}

impl Future for TungguDua {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if self.mula.elapsed() >= Duration::from_secs(2) {
            Poll::Ready(())
        } else {
            // Jadualkan wake selepas 2 saat
            let waker = cx.waker().clone();
            std::thread::spawn(move || {
                std::thread::sleep(Duration::from_secs(2));
                waker.wake();
            });
            Poll::Pending
        }
    }
}
```

## State Machine di Bawah Tabir

```rust
// Semasa compile, async fn ditukar kepada state machine!

async fn contoh() {
    let a = langkah_satu().await;   // titik suspend #1
    let b = langkah_dua(a).await;   // titik suspend #2
    println!("{}", b);
}

// Compiler hasilkan sesuatu seperti ini:
enum ContohStateMachine {
    Mula,
    TungguLangkahSatu { future: LangkahSatuFuture },
    TungguLangkahDua  { a: String, future: LangkahDuaFuture },
    Selesai,
}

// Setiap kali .await → simpan state semasa
// Setiap kali poll → teruskan dari state tersimpan
```

---

# 3. async dan .await

## Apa yang Sebenarnya Berlaku?

```rust
use tokio::time::{sleep, Duration};

// `async fn` = fungsi yang return impl Future
// Badan fungsi tidak berjalan terus — ia cipta state machine

async fn satu() -> u32 { 1 }

// Equivalent kepada:
// fn satu() -> impl Future<Output = u32> {
//     async { 1 }
// }

// `.await` = "poll future ini sampai Ready, dan beri nilai output"
// TETAPI: semasa Pending, yield balik ke executor

async fn guna_await() {
    // Semasa `sleep(...).await`:
    //   1. Poll Future sleep
    //   2. Dapat Pending → yield ke executor
    //   3. Executor buat kerja lain
    //   4. Timer habis → Waker dipanggil
    //   5. Executor poll semula → Ready
    //   6. Teruskan selepas await
    sleep(Duration::from_millis(100)).await;
    println!("Selepas await!");
}
```

## await hanya boleh dalam async fn

```rust
// ❌ TIDAK BOLEH dalam fungsi biasa
// fn bukan_async() {
//     let val = some_future().await; // ERROR: syntax error
// }

// ✔ MESTI dalam async fn atau async block
async fn boleh_await() {
    let val = some_future().await; // OK
}

fn dalam_sync() {
    // Guna async block kalau perlu
    let _future = async {
        let val = some_future().await; // OK dalam async block
        val
    };
    // Tapi masih perlu executor untuk jalankan!
}
```

## async Block

```rust
// async block = async fn tanpa nama
// Berguna untuk:
// 1. Future inline
// 2. Conditional async
// 3. Capture variables

async fn contoh() {
    // Async block sebagai expression
    let hasil = async {
        let a = langkah_a().await;
        let b = langkah_b(a).await;
        b + 10
    }.await;

    println!("{}", hasil);
}

// async move — sama seperti move closure
async fn dengan_move() {
    let teks = String::from("hello");

    let future = async move {
        // teks dipindah ke dalam async block
        println!("{}", teks);
    };

    future.await;
    // println!("{}", teks); // ERROR: teks sudah moved
}
```

## Return Types

```rust
// async fn sentiasa return Future
// Output type ada dalam <> atau diinfer

async fn tiada_return()              { }           // Future<Output = ()>
async fn return_i32() -> i32         { 42 }        // Future<Output = i32>
async fn return_result() -> Result<String, std::io::Error> {  // Future<Output = Result<...>>
    tokio::fs::read_to_string("fail.txt").await
}

// Await dalam rantaian
async fn rantaian() {
    // Boleh chain terus
    let panjang = tokio::fs::read_to_string("fail.txt")
        .await
        .unwrap_or_default()
        .len();

    println!("Panjang: {}", panjang);
}
```

---

# 4. Executors dan Runtimes

## Apa itu Runtime?

```
Runtime = Jentera yang jalankan futures

Komponen utama:
  1. Executor   — poll futures, uruskan task queue
  2. Reactor    — monitor I/O events (epoll/kqueue/IOCP)
  3. Timer      — urus sleep, timeout, deadline
  4. Thread pool — worker threads (untuk async I/O + blocking tasks)

Rust tiada runtime built-in!
Perlu pilih: Tokio, async-std, smol, dll.
```

## Tokio — Runtime Paling Popular

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
// ── Cara 1: #[tokio::main] — paling mudah ────────────────────
#[tokio::main]
async fn main() {
    println!("Dalam async main!");
    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
    println!("Selesai!");
}

// #[tokio::main] mengembang kepada:
// fn main() {
//     tokio::runtime::Builder::new_multi_thread()
//         .enable_all()
//         .build()
//         .unwrap()
//         .block_on(async {
//             // kod anda di sini
//         })
// }

// ── Cara 2: Runtime manual — lebih kawalan ────────────────────
fn main() {
    // Multi-thread runtime (default, guna semua CPU cores)
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        println!("Multi-thread runtime");
    });

    // Current-thread runtime (satu thread sahaja)
    let rt_single = tokio::runtime::Builder::new_current_thread()
        .enable_all()
        .build()
        .unwrap();
    rt_single.block_on(async {
        println!("Single-thread runtime");
    });
}

// ── Cara 3: Konfigurasi lanjutan ──────────────────────────────
fn main() {
    let rt = tokio::runtime::Builder::new_multi_thread()
        .worker_threads(4)                    // 4 worker threads
        .max_blocking_threads(128)            // thread pool untuk blocking
        .thread_name("worker")               // nama thread
        .on_thread_start(|| println!("Thread dimulakan"))
        .enable_io()                          // I/O driver
        .enable_time()                        // timer driver
        .build()
        .unwrap();

    rt.block_on(run());
}

async fn run() {
    println!("Berjalan dengan konfigurasi custom!");
}
```

## Bagaimana Executor Bekerja?

```
Executor loop (ringkasan):

loop {
    1. Semak task queue — ada task yang ready?
       → Ya: poll task
         → Task return Ready → selesai
         → Task return Pending → simpan semula, tunggu Waker
       → Tidak: idle

    2. Semak I/O events (epoll/kqueue)
       → Network data tiba? → wake task berkenaan
       → Timer habis?       → wake task berkenaan

    3. Ulangi
}

PENTING:
  - Executor berjalan pada thread
  - Multi-thread runtime: banyak executor thread
  - Task boleh "steal" antara thread (work stealing)
```

---

# 5. Task Spawning dan Orchestration

## spawn — Jalankan Task Serentak

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    // spawn = cipta task baru (background)
    // Task berjalan SERENTAK dengan caller
    let handle = task::spawn(async {
        println!("Task ini berjalan serentak!");
        "hasil dari task"
    });

    // Buat kerja lain semasa task berjalan
    println!("Main terus berjalan...");

    // Tunggu task selesai
    let hasil = handle.await.unwrap();
    println!("Task selesai: {}", hasil);
}
```

## join! — Tunggu Semua Serentak

```rust
use tokio::time::{sleep, Duration};

async fn dapatkan_pengguna(id: u32) -> String {
    sleep(Duration::from_millis(100)).await;
    format!("Pengguna {}", id)
}

async fn dapatkan_produk(id: u32) -> String {
    sleep(Duration::from_millis(150)).await;
    format!("Produk {}", id)
}

#[tokio::main]
async fn main() {
    // ❌ PERLAHAN: berjalan satu per satu (250ms)
    let pengguna = dapatkan_pengguna(1).await;
    let produk   = dapatkan_produk(1).await;

    // ✔ LAJU: berjalan serentak (150ms sahaja!)
    let (pengguna, produk) = tokio::join!(
        dapatkan_pengguna(1),
        dapatkan_produk(1)
    );

    println!("{} membeli {}", pengguna, produk);
}
```

## try_join! — Henti bila Ada Error

```rust
use tokio::io;

async fn operasi_a() -> Result<String, io::Error> { Ok("A".into()) }
async fn operasi_b() -> Result<String, io::Error> { Ok("B".into()) }
async fn operasi_c() -> Result<String, io::Error> {
    Err(io::Error::new(io::ErrorKind::Other, "Operasi C gagal!"))
}

#[tokio::main]
async fn main() {
    // try_join! berhenti dan return Err kalau mana-mana gagal
    match tokio::try_join!(operasi_a(), operasi_b(), operasi_c()) {
        Ok((a, b, c)) => println!("{} {} {}", a, b, c),
        Err(e)        => println!("Gagal: {}", e), // "Operasi C gagal!"
    }
}
```

## select! — Race Antara Futures

```rust
use tokio::time::{sleep, Duration};
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel::<String>();

    // Spawn task yang akan hantar mesej
    tokio::spawn(async move {
        sleep(Duration::from_millis(500)).await;
        tx.send("Mesej tiba!".into()).unwrap();
    });

    // select! — ambil yang PERTAMA selesai
    tokio::select! {
        mesej = rx => {
            println!("Terima: {}", mesej.unwrap());
        }
        _ = sleep(Duration::from_secs(1)) => {
            println!("Timeout! Tiada mesej dalam 1 saat");
        }
    }
}
```

## Spawn Banyak Task & Kumpul Hasil

```rust
use tokio::task::JoinSet;

#[tokio::main]
async fn main() {
    let mut set = JoinSet::new();

    // Spawn 10 task serentak
    for i in 0..10 {
        set.spawn(async move {
            tokio::time::sleep(tokio::time::Duration::from_millis(i * 10)).await;
            i * i // return kuasa dua
        });
    }

    // Kumpul hasil mengikut urutan selesai (bukan urutan spawn)
    let mut hasil = Vec::new();
    while let Some(res) = set.join_next().await {
        hasil.push(res.unwrap());
    }

    println!("Semua selesai: {:?}", hasil);
}
```

## Ownership dalam Async Tasks

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

// Task yang di-spawn perlu 'static + Send
// 'static = tidak ada borrowed reference yang expired
// Send = selamat hantar ke thread lain

async fn contoh_ownership() {
    let data = String::from("hello");

    // ❌ TIDAK boleh borrow ke dalam spawn
    // tokio::spawn(async {
    //     println!("{}", data); // ERROR: borrowed data may outlive task
    // });

    // ✔ MOVE ke dalam task
    tokio::spawn(async move {
        println!("{}", data); // OK: data dipindah masuk
    });

    // ✔ Guna Arc untuk share antara task
    let shared = Arc::new(Mutex::new(vec![1, 2, 3]));
    let klon = Arc::clone(&shared);

    tokio::spawn(async move {
        klon.lock().await.push(4);
    }).await.unwrap();

    println!("{:?}", shared.lock().await);
}
```

---

# 6. Blocking Pitfalls dan Performance Traps

## Bahaya Blocking dalam Async Context

```rust
use tokio::time::Duration;

// ❌ MASALAH: blocking call dalam async task
// Ini "freeze" seluruh thread executor!
async fn blocking_buruk() {
    // std::fs berjalan secara synchronous — BLOCK thread!
    let data = std::fs::read_to_string("fail.txt").unwrap();

    // std::thread::sleep juga block!
    std::thread::sleep(Duration::from_secs(5)); // JANGAN!

    // Computation berat pun block
    let hasil = (0..1_000_000_000u64).sum::<u64>(); // JANGAN dalam async!
}

// Analogi:
// Executor = pelayan restoran
// Task = meja yang perlu dilayan
// Blocking call = pelayan terpaksa masak sendiri (bukan hantar ke dapur)
//                 Semua meja lain tersekat!
```

## Penyelesaian: spawn_blocking

```rust
#[tokio::main]
async fn main() {
    // ✔ Hantar blocking work ke thread pool yang berbeza
    let data = tokio::task::spawn_blocking(|| {
        // Ini berjalan dalam thread pool khas (bukan executor thread)
        std::fs::read_to_string("fail.txt").unwrap()
    }).await.unwrap();

    println!("{}", data);

    // ✔ Guna async versi library
    let data2 = tokio::fs::read_to_string("fail.txt").await.unwrap();
    // tokio::fs = async wrapper untuk file I/O

    // ✔ Sleep guna tokio, bukan std::thread
    tokio::time::sleep(Duration::from_secs(1)).await; // OK!
    // std::thread::sleep(...) // JANGAN!
}
```

## Computation Berat

```rust
// CPU-bound work akan block executor thread!
// Penyelesaian: spawn_blocking atau rayon

#[tokio::main]
async fn main() {
    // ❌ BURUK: CPU bound dalam async task
    let hasil = compute_berat().await;

    // ✔ BAIK: pindah ke blocking thread pool
    let hasil = tokio::task::spawn_blocking(|| {
        (0..1_000_000u64).map(|n| n * n).sum::<u64>()
    }).await.unwrap();

    println!("{}", hasil);

    // ✔ JUGA BAIK: guna rayon untuk CPU parallelism
    let hasil = tokio::task::spawn_blocking(|| {
        use rayon::prelude::*;
        (0u64..1_000_000).into_par_iter().map(|n| n * n).sum::<u64>()
    }).await.unwrap();

    println!("{}", hasil);
}

async fn compute_berat() -> u64 {
    // ❌ JANGAN: ini block executor!
    (0..1_000_000u64).map(|n| n * n).sum()
}
```

## Perangkap Biasa

```rust
use tokio::time::Duration;

// ❌ PERANGKAP 1: Mutex std dalam async
use std::sync::Mutex;

async fn mutex_std_buruk() {
    let m = Mutex::new(0);
    let guard = m.lock().unwrap();
    tokio::time::sleep(Duration::from_secs(1)).await; // BAHAYA!
    // MutexGuard merentasi await point!
    // Jika thread lain panggil lock() semasa sleep → deadlock!
    drop(guard);
}

// ✔ Guna tokio::sync::Mutex untuk async context
use tokio::sync::Mutex as AsyncMutex;

async fn mutex_async_betul() {
    let m = AsyncMutex::new(0);
    {
        let mut guard = m.lock().await;
        *guard += 1;
    } // guard di-drop sebelum sleep
    tokio::time::sleep(Duration::from_secs(1)).await; // OK
}

// ❌ PERANGKAP 2: Terlalu banyak spawn
// Setiap spawn ada overhead. Jangan spawn untuk task kecil!
async fn spawn_berlebihan() {
    for i in 0..1000 {
        tokio::spawn(async move { i + 1 }); // overhead berlebihan!
    }
}

// ✔ Kumpulkan dalam satu task atau guna FuturesUnordered
use futures::stream::{FuturesUnordered, StreamExt};

async fn lebih_cekap() {
    let mut futures = FuturesUnordered::new();
    for i in 0..1000 {
        futures.push(async move { i + 1 });
    }
    while let Some(_) = futures.next().await {}
}
```

---

# 7. Shared State, Channels, dan Synchronization

## Prinsip: Prefer Message Passing

```
"Do not communicate by sharing memory;
 instead, share memory by communicating."

Bila guna Channel:
  ✔ Hantar kerja antara task
  ✔ Event notification
  ✔ Producer-consumer pattern
  ✔ Bila ownership perlu berpindah

Bila guna Shared State (Mutex/RwLock):
  ✔ Data yang dibaca/ditulis dari mana-mana sahaja
  ✔ Cache atau registry
  ✔ State yang perlu atomic update
```

## Tokio Channels

```rust
use tokio::sync::{mpsc, oneshot, broadcast, watch};

// ── mpsc: Multi-producer, Single-consumer ─────────────────────
async fn demo_mpsc() {
    let (tx, mut rx) = mpsc::channel::<String>(100); // buffer 100

    // Boleh clone tx untuk banyak producer
    let tx2 = tx.clone();

    tokio::spawn(async move {
        tx.send("Mesej dari task 1".into()).await.unwrap();
    });

    tokio::spawn(async move {
        tx2.send("Mesej dari task 2".into()).await.unwrap();
    });

    // Consumer
    while let Some(mesej) = rx.recv().await {
        println!("{}", mesej);
    }
}

// ── oneshot: Hantar SATU nilai sahaja ─────────────────────────
async fn demo_oneshot() {
    let (tx, rx) = oneshot::channel::<u32>();

    tokio::spawn(async move {
        // Buat kerja, hantar hasil
        let hasil = 42;
        tx.send(hasil).unwrap();
    });

    let nilai = rx.await.unwrap();
    println!("Dapat: {}", nilai);
}

// ── broadcast: Satu-ke-banyak ─────────────────────────────────
async fn demo_broadcast() {
    let (tx, _) = broadcast::channel::<String>(16);

    // Setiap subscriber dapat semua mesej
    for i in 0..3 {
        let mut rx = tx.subscribe();
        tokio::spawn(async move {
            while let Ok(mesej) = rx.recv().await {
                println!("Subscriber {}: {}", i, mesej);
            }
        });
    }

    tx.send("Hello semua!".into()).unwrap();
    tx.send("Mesej kedua".into()).unwrap();
    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
}

// ── watch: Nilai terkini sahaja (latest value) ────────────────
async fn demo_watch() {
    let (tx, rx) = watch::channel(0u32); // nilai awal = 0

    tokio::spawn(async move {
        for i in 1..=5 {
            tx.send(i).unwrap();
            tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
        }
    });

    let mut rx = rx;
    loop {
        rx.changed().await.unwrap(); // tunggu nilai baru
        println!("Nilai terkini: {}", *rx.borrow());
    }
}
```

## Async Mutex dan RwLock

```rust
use tokio::sync::{Mutex, RwLock};
use std::sync::Arc;
use std::collections::HashMap;

// ── Mutex untuk mutable state ─────────────────────────────────
async fn demo_mutex() {
    let kiraan = Arc::new(Mutex::new(0u32));

    let mut handles = vec![];
    for _ in 0..10 {
        let k = Arc::clone(&kiraan);
        handles.push(tokio::spawn(async move {
            let mut guard = k.lock().await;
            *guard += 1;
            // guard di-drop di sini — lock dilepas
        }));
    }

    for h in handles { h.await.unwrap(); }
    println!("Kiraan: {}", *kiraan.lock().await); // 10
}

// ── RwLock untuk read-heavy state ─────────────────────────────
async fn demo_rwlock() {
    let cache: Arc<RwLock<HashMap<String, String>>> =
        Arc::new(RwLock::new(HashMap::new()));

    // Banyak reader serentak
    let c1 = Arc::clone(&cache);
    let c2 = Arc::clone(&cache);

    tokio::join!(
        async move { let _ = c1.read().await; },
        async move { let _ = c2.read().await; }
        // Kedua-dua boleh baca serentak!
    );

    // Satu writer
    cache.write().await.insert("kunci".into(), "nilai".into());
}
```

## Pattern: Actor dengan Channel

```rust
use tokio::sync::{mpsc, oneshot};

// Actor pattern: state dilindungi dalam satu task
// Komunikasi melalui message (channel)

enum PesananAktor {
    Tambah { nilai: i32 },
    Dapatkan { balasan: oneshot::Sender<i32> },
    Henti,
}

async fn jalankan_aktor(mut rx: mpsc::Receiver<PesananAktor>) {
    let mut state = 0i32;
    while let Some(pesanan) = rx.recv().await {
        match pesanan {
            PesananAktor::Tambah { nilai } => {
                state += nilai;
            }
            PesananAktor::Dapatkan { balasan } => {
                let _ = balasan.send(state);
            }
            PesananAktor::Henti => break,
        }
    }
}

#[tokio::main]
async fn main() {
    let (tx, rx) = mpsc::channel(32);
    tokio::spawn(jalankan_aktor(rx));

    tx.send(PesananAktor::Tambah { nilai: 10 }).await.unwrap();
    tx.send(PesananAktor::Tambah { nilai: 5 }).await.unwrap();

    let (btx, brx) = oneshot::channel();
    tx.send(PesananAktor::Dapatkan { balasan: btx }).await.unwrap();
    println!("State: {}", brx.await.unwrap()); // 15

    tx.send(PesananAktor::Henti).await.unwrap();
}
```

---

# 8. Streams dan Async Data Flow

## Apa itu Stream?

```rust
// Future = satu nilai pada masa hadapan
// Stream = banyak nilai pada masa hadapan (async iterator)

// trait Stream {
//     type Item;
//     fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>)
//         -> Poll<Option<Self::Item>>;
// }

// Anggap: Iterator::next() → Option<T>
//         Stream::poll_next() → Poll<Option<T>>
```

## Guna Stream

```rust
use tokio_stream::{self as stream, StreamExt};
use tokio::time::{interval, Duration};

#[tokio::main]
async fn main() {
    // ── Stream dari iterator biasa ─────────────────────────────
    let mut s = stream::iter(vec![1, 2, 3, 4, 5]);
    while let Some(val) = s.next().await {
        println!("{}", val);
    }

    // ── Stream dengan adapter (macam Iterator) ─────────────────
    let jumlah: i32 = stream::iter(1..=10)
        .filter(|&x| async move { x % 2 == 0 })
        .map(|x| async move { x * x })
        .buffer_unordered(4) // proses 4 serentak
        .collect::<Vec<_>>()
        .await
        .into_iter()
        .sum();

    println!("Jumlah: {}", jumlah);

    // ── Stream dari channel ────────────────────────────────────
    let (tx, rx) = tokio::sync::mpsc::channel(10);

    tokio::spawn(async move {
        for i in 0..5 {
            tx.send(i).await.unwrap();
            tokio::time::sleep(Duration::from_millis(100)).await;
        }
    });

    let mut rx_stream = tokio_stream::wrappers::ReceiverStream::new(rx);
    while let Some(val) = rx_stream.next().await {
        println!("Stream: {}", val);
    }

    // ── Interval stream ────────────────────────────────────────
    let mut ticker = interval(Duration::from_millis(500));
    for _ in 0..3 {
        ticker.tick().await;
        println!("Tick!");
    }
}
```

## Proses Stream Serentak

```rust
use futures::stream::{self, StreamExt};

#[tokio::main]
async fn main() {
    let url_list = vec!["url1", "url2", "url3", "url4", "url5"];

    // buffer_unordered = proses N serentak, hasilkan ikut turutan siap
    let hasil: Vec<_> = stream::iter(url_list)
        .map(|url| async move {
            // Simulate HTTP request
            tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
            format!("Data dari {}", url)
        })
        .buffer_unordered(3) // max 3 serentak
        .collect()
        .await;

    println!("{:?}", hasil);

    // buffered = sama tapi kekalkan urutan input
    let hasil_tertib: Vec<_> = stream::iter(vec![1, 2, 3])
        .map(|n| async move { n * 2 })
        .buffered(2)
        .collect()
        .await;

    println!("{:?}", hasil_tertib); // [2, 4, 6] — dalam urutan
}
```

---

# 9. Cancellation dan Graceful Shutdown

## Cancellation dalam Async Rust

```rust
use tokio::time::{sleep, Duration};

// Apabila Future di-drop → ia di-cancel!
// Ini adalah "cancellation by drop"

async fn task_yang_boleh_cancel() {
    println!("Mula");
    sleep(Duration::from_secs(10)).await; // boleh di-cancel di sini
    println!("Selesai"); // mungkin tidak dicapai!
}

#[tokio::main]
async fn main() {
    // select! cancel future yang kalah
    tokio::select! {
        _ = task_yang_boleh_cancel() => println!("Task selesai"),
        _ = sleep(Duration::from_secs(1)) => println!("Timeout — task di-cancel!"),
    }
    // task_yang_boleh_cancel di-drop apabila timeout menang
}
```

## CancellationToken — Cancel Eksplisit

```toml
tokio-util = "0.7"
```

```rust
use tokio_util::sync::CancellationToken;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let token = CancellationToken::new();

    // Spawn beberapa task yang akan cancel bila diminta
    for i in 0..3 {
        let t = token.clone();
        tokio::spawn(async move {
            tokio::select! {
                _ = t.cancelled() => {
                    println!("Task {} di-cancel", i);
                }
                _ = buat_kerja(i) => {
                    println!("Task {} selesai", i);
                }
            }
        });
    }

    sleep(Duration::from_millis(500)).await;

    // Cancel semua task
    println!("Menghantar signal cancel...");
    token.cancel();

    sleep(Duration::from_millis(100)).await;
    println!("Selesai");
}

async fn buat_kerja(id: usize) {
    sleep(Duration::from_secs(10)).await;
    println!("Kerja {} selesai (tidak akan capai ini)", id);
}
```

## Graceful Shutdown — Signal OS

```rust
use tokio::signal;
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    let (shutdown_tx, _) = broadcast::channel::<()>(1);

    // Spawn worker tasks
    for i in 0..3 {
        let mut rx = shutdown_tx.subscribe();
        tokio::spawn(async move {
            loop {
                tokio::select! {
                    _ = rx.recv() => {
                        println!("Worker {} menerima shutdown signal", i);
                        // Buat cleanup di sini
                        break;
                    }
                    _ = buat_kerja_berterusan(i) => {}
                }
            }
            println!("Worker {} berhenti dengan bersih", i);
        });
    }

    // Tunggu signal dari OS (Ctrl+C atau SIGTERM)
    tokio::select! {
        _ = signal::ctrl_c() => {
            println!("\nCtrl+C diterima!");
        }
        _ = signal_sigterm() => {
            println!("SIGTERM diterima!");
        }
    }

    println!("Menghantar shutdown signal kepada semua worker...");
    let _ = shutdown_tx.send(());

    // Beri masa untuk cleanup
    tokio::time::sleep(tokio::time::Duration::from_secs(5)).await;
    println!("Shutdown selesai.");
}

async fn buat_kerja_berterusan(id: usize) {
    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
}

#[cfg(unix)]
async fn signal_sigterm() {
    use tokio::signal::unix::{signal, SignalKind};
    let mut stream = signal(SignalKind::terminate()).unwrap();
    stream.recv().await;
}

#[cfg(not(unix))]
async fn signal_sigterm() {
    // Windows tiada SIGTERM — tunggu selamanya
    std::future::pending::<()>().await;
}
```

## Cleanup yang Selamat Semasa Cancel

```rust
use tokio::time::{sleep, Duration};

// AWAS: Bila Future di-cancel, kod selepas .await MUNGKIN tidak berjalan
// Guna Drop trait untuk pastikan cleanup berlaku

struct Sambungan {
    id: u32,
}

impl Sambungan {
    async fn buat(id: u32) -> Self {
        println!("Membuka sambungan {}", id);
        sleep(Duration::from_millis(10)).await;
        Sambungan { id }
    }

    async fn guna(&self) {
        println!("Menggunakan sambungan {}", self.id);
        sleep(Duration::from_secs(5)).await; // boleh di-cancel di sini
        println!("Selesai guna sambungan {}", self.id); // mungkin tidak dicapai
    }
}

// Drop SENTIASA dipanggil, walaupun di-cancel!
impl Drop for Sambungan {
    fn drop(&mut self) {
        println!("Sambungan {} ditutup (Drop dipanggil)");
        // Cleanup synchronous boleh dibuat di sini
        // TAPI: tidak boleh .await dalam Drop!
    }
}

#[tokio::main]
async fn main() {
    let sambungan = Sambungan::buat(1).await;

    tokio::select! {
        _ = sambungan.guna() => println!("Selesai"),
        _ = sleep(Duration::from_millis(500)) => println!("Timeout"),
    }
    // sambungan.drop() dipanggil di sini — walaupun di-cancel!
}
```

---

# 10. Testing Async Rust

## Test Asas

```rust
// Tambah #[tokio::test] untuk async test
#[cfg(test)]
mod tests {
    use super::*;
    use tokio::time::{sleep, Duration};

    #[tokio::test]
    async fn test_mudah() {
        let hasil = fungsi_async().await;
        assert_eq!(hasil, 42);
    }

    #[tokio::test]
    async fn test_dengan_timeout() {
        // Test gagal kalau lebih 1 saat
        let hasil = tokio::time::timeout(
            Duration::from_secs(1),
            fungsi_yang_sepatutnya_pantas()
        ).await;

        assert!(hasil.is_ok(), "Test timeout!");
    }

    // Runtime current_thread untuk test yang tidak perlu multi-thread
    #[tokio::test(flavor = "current_thread")]
    async fn test_single_thread() {
        assert_eq!(1 + 1, 2);
    }

    // Multi-thread dengan bilangan thread tertentu
    #[tokio::test(flavor = "multi_thread", worker_threads = 2)]
    async fn test_multi_thread() {
        let handle1 = tokio::spawn(async { 1 });
        let handle2 = tokio::spawn(async { 2 });
        let (a, b) = tokio::join!(handle1, handle2);
        assert_eq!(a.unwrap() + b.unwrap(), 3);
    }
}

async fn fungsi_async() -> i32 { 42 }
async fn fungsi_yang_sepatutnya_pantas() -> i32 { 1 }
```

## Mock dan Dependency Injection

```rust
use async_trait::async_trait;

// Guna trait untuk dependency injection dalam async
#[async_trait]
pub trait HttpClient: Send + Sync {
    async fn get(&self, url: &str) -> Result<String, String>;
}

pub struct Servis {
    client: Box<dyn HttpClient>,
}

impl Servis {
    pub fn baru(client: Box<dyn HttpClient>) -> Self {
        Servis { client }
    }

    pub async fn dapatkan_data(&self, id: u32) -> Result<String, String> {
        let url = format!("https://api.example.com/data/{}", id);
        self.client.get(&url).await
    }
}

// Mock untuk testing
struct MockClient {
    respons: String,
}

#[async_trait]
impl HttpClient for MockClient {
    async fn get(&self, _url: &str) -> Result<String, String> {
        Ok(self.respons.clone())
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_servis_dengan_mock() {
        let mock = MockClient { respons: r#"{"id":1,"nama":"Ali"}"#.into() };
        let servis = Servis::baru(Box::new(mock));

        let hasil = servis.dapatkan_data(1).await.unwrap();
        assert!(hasil.contains("Ali"));
    }
}
```

## Test Masa (Time Manipulation)

```rust
#[cfg(test)]
mod tests {
    use tokio::time::{pause, advance, Duration, Instant};

    #[tokio::test]
    async fn test_dengan_masa_palsu() {
        // pause() hentikan masa sebenar
        tokio::time::pause();

        let mula = Instant::now();

        // advance() gerakkan masa ke hadapan tanpa tunggu sebenar
        tokio::time::advance(Duration::from_secs(60)).await;

        let berlalu = mula.elapsed();
        assert!(berlalu >= Duration::from_secs(60));
        // Test berlaku SEGERA walaupun "60 saat" berlalu!
    }

    #[tokio::test]
    async fn test_cache_expire() {
        tokio::time::pause();

        let cache = Cache::baru(Duration::from_secs(30));
        cache.simpan("kunci", "nilai");

        assert!(cache.dapatkan("kunci").is_some());

        // Simulasi masa berlalu 31 saat
        tokio::time::advance(Duration::from_secs(31)).await;

        assert!(cache.dapatkan("kunci").is_none()); // expired!
    }
}
```

## Hindari Test yang Fragile

```rust
// ❌ FRAGILE: bergantung pada timing sebenar
#[tokio::test]
async fn test_fragile() {
    tokio::spawn(async {
        tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
        // update global state
    });

    tokio::time::sleep(tokio::time::Duration::from_millis(200)).await; // harap cukup masa
    // Bergantung pada timing! Boleh gagal dalam CI yang lambat
}

// ✔ KUKUH: guna synchronization
#[tokio::test]
async fn test_kukuh() {
    let (tx, rx) = tokio::sync::oneshot::channel();

    tokio::spawn(async move {
        tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
        tx.send(()).unwrap(); // signal bila selesai
    });

    rx.await.unwrap(); // tunggu signal, bukan timing
    // Test ini sentiasa betul tanpa mengira kelajuan sistem!
}
```

---

# 11. Production-Oriented Guidance

## Structured Logging dalam Async

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

```rust
use tracing::{info, warn, error, instrument, Span};
use uuid::Uuid;

// #[instrument] tambah span untuk setiap fungsi
#[instrument(fields(request_id = %Uuid::new_v4()))]
async fn tangani_request(pengguna_id: u32) -> Result<String, String> {
    info!("Mula proses request untuk pengguna {}", pengguna_id);

    let data = ambil_data(pengguna_id).await?;

    info!("Berjaya ambil data");
    Ok(data)
}

#[instrument(skip(id), fields(id = id))]
async fn ambil_data(id: u32) -> Result<String, String> {
    // Span otomatik merekod masa
    tokio::time::sleep(tokio::time::Duration::from_millis(10)).await;
    Ok(format!("Data untuk {}", id))
}

fn setup_tracing() {
    tracing_subscriber::fmt()
        .with_env_filter("info")  // RUST_LOG=debug untuk debug
        .json()                    // JSON untuk production
        .init();
}
```

## Metrics dan Observability

```rust
// Guna tokio-metrics untuk memantau task
use tokio_metrics::{TaskMonitor, RuntimeMonitor};

async fn dengan_metrics() {
    let monitor = TaskMonitor::new();

    // Instrument task
    let handle = tokio::spawn(monitor.instrument(async {
        // kerja task
    }));

    handle.await.unwrap();

    // Baca metrics
    let metrics = monitor.cumulative();
    println!("Mean poll duration: {:?}", metrics.mean_poll_duration);
    println!("Slow polls: {}", metrics.slow_poll_count);
}
```

## Configuration Best Practices

```rust
// Konfigurasi runtime untuk production

fn bina_runtime() -> tokio::runtime::Runtime {
    tokio::runtime::Builder::new_multi_thread()
        // Bilangan thread = bilangan CPU core (biasanya)
        .worker_threads(num_cpus::get())
        // Thread pool untuk blocking tasks
        .max_blocking_threads(512)
        // Nama thread untuk debugging
        .thread_name("tokio-worker")
        // Hook untuk monitoring
        .on_thread_start(|| {
            tracing::debug!("Worker thread dimulakan");
        })
        .on_thread_stop(|| {
            tracing::debug!("Worker thread dihentikan");
        })
        .enable_all()
        .build()
        .expect("Gagal bina Tokio runtime")
}
```

## Pengendalian Panic

```rust
// Panic dalam spawn task tidak propagate secara automatik!
// Perlu tangkap sendiri

#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        panic!("Task panic!"); // ini TIDAK crash program terus
    });

    // Tangkap panic dari task
    match handle.await {
        Ok(nilai)  => println!("Task selesai: {:?}", nilai),
        Err(e) if e.is_panic() => {
            eprintln!("Task panic: {:?}", e);
            // Log, alert, restart task jika perlu
        }
        Err(e) => eprintln!("Task error: {:?}", e),
    }
}
```

## Rate Limiting dan Backpressure

```rust
use tokio::sync::Semaphore;
use std::sync::Arc;

// Semaphore untuk hadkan concurrent operations
async fn dengan_rate_limit() {
    // Maksimum 10 sambungan serentak ke DB
    let semaphore = Arc::new(Semaphore::new(10));

    let mut handles = vec![];

    for i in 0..100 {
        let sem = Arc::clone(&semaphore);
        handles.push(tokio::spawn(async move {
            let _permit = sem.acquire().await.unwrap();
            // Hanya 10 task boleh masuk sini serentak
            query_database(i).await;
            // permit di-drop di sini → slot bebas
        }));
    }

    for h in handles { h.await.unwrap(); }
}

async fn query_database(id: u32) {
    tokio::time::sleep(tokio::time::Duration::from_millis(50)).await;
    println!("Query {}", id);
}
```

## Panduan Ringkas Production

```
1. LOGGING
   ✔ Guna tracing, bukan println!
   ✔ Structured logging (JSON) untuk production
   ✔ Masukkan trace_id/request_id dalam setiap log
   ✔ Asingkan log level (ERROR untuk alert, INFO untuk audit)

2. ERROR HANDLING
   ✔ Jangan unwrap() dalam production code
   ✔ Tangkap panic dari spawn task
   ✔ Tambah context ke setiap error (anyhow::Context)
   ✔ Distinguish recoverable vs unrecoverable errors

3. PERFORMANCE
   ✔ Profile dulu, optimize kemudian
   ✔ Elak blocking dalam async context
   ✔ Guna spawn_blocking untuk CPU work
   ✔ Monitor slow polls (>10ms = masalah)
   ✔ Buffer channel yang sesuai (jangan terlalu besar/kecil)

4. RELIABILITY
   ✔ Implement graceful shutdown
   ✔ Guna timeout untuk semua I/O
   ✔ Retry dengan exponential backoff
   ✔ Circuit breaker untuk dependencies

5. TESTING
   ✔ Test async code dengan #[tokio::test]
   ✔ Guna tokio::time::pause() untuk time-based tests
   ✔ Mock external dependencies
   ✔ Test cancellation paths

6. DEBUGGING
   ✔ TOKIO_CONSOLE=1 untuk live task inspection
   ✔ Aktifkan tokio tracing: RUSTFLAGS="--cfg tokio_unstable"
   ✔ Guna tokio-console untuk visualize task states
   ✔ Log task lifecycle events
```

---

## Rujukan Pantas

### Pilih Synchronization Primitive

```
Bila data hanya perlu dibaca → Arc<T>
Bila data perlu ditulis, async context → Arc<tokio::sync::Mutex<T>>
Bila data perlu ditulis, sync context → Arc<std::sync::Mutex<T>>
Bila banyak reader, sikit writer → Arc<tokio::sync::RwLock<T>>
Hantar nilai satu kali → oneshot::channel
Hantar banyak nilai (1 producer) → mpsc::channel
Hantar banyak nilai (N producer) → mpsc::channel + tx.clone()
Broadcast ke semua → broadcast::channel
Nilai terkini sahaja → watch::channel
```

### Cargo.toml untuk Async Rust

```toml
[dependencies]
tokio        = { version = "1", features = ["full"] }
tokio-stream = "0.1"
tokio-util   = "0.7"
futures      = "0.3"
async-trait  = "0.1"
tracing      = "0.1"

[dev-dependencies]
tokio        = { version = "1", features = ["full", "test-util"] }
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

---

*Async Rust: kompleks di permukaan, kukuh di dalam.*
*Faham model mental — selebihnya adalah detail.* 🦀
