# ⚡ Async Programming dalam Rust — Beginner to Advanced

> Panduan lengkap async/await: dari konsep asas hingga concurrent patterns lanjutan.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu Async Programming?

```
SYNCHRONOUS (blocking) — satu kerja selesai dulu, baru yang lain:

  Thread 1: ──[Baca fail A]──────────────[Baca fail B]──────────────▶
                              tunggu...                   tunggu...
  Masa:      0ms             500ms        500ms           1000ms    1000ms

ASYNCHRONOUS (non-blocking) — tunggu sambil buat kerja lain:

  Task 1:   ──[Mula baca A]──[...]──────────────────[Dapat A]──────▶
                 ↓ yield                                 ↑
  Task 2:         ──[Mula baca B]──[...]──────[Dapat B]────────────▶

  Masa:      0ms              500ms (siap KEDUA-DUA!)

Async = Buat banyak kerja dalam masa yang sama, tanpa thread baru!
```

---

## Kenapa Async dalam Rust?

```
Masalah:
  Server web perlu layan 10,000 request serentak.

  Cara lama (1 thread per request):
  10,000 threads × 2MB = 20GB RAM! 💸

  Cara async:
  1 thread OS = boleh handle ribuan task async
  Setiap task hanya pakai beberapa KB! 💚

Rust async:
  ✔ Zero-cost abstractions — tiada overhead runtime
  ✔ Memory safe — compiler check semua
  ✔ No garbage collector — predictable latency
  ✔ Tokio ecosystem — battle-tested di production
```

---

## Peta Pembelajaran

```
Bab 1  → Future & async/await Asas
Bab 2  → Tokio Runtime
Bab 3  → async Functions & Tasks
Bab 4  → join! & select! — Concurrent Execution
Bab 5  → Channels — Komunikasi Antara Tasks
Bab 6  → Mutex & Shared State
Bab 7  → Async I/O — Fail & Network
Bab 8  → Error Handling dalam Async
Bab 9  → Pattern Lanjutan
Bab 10 → Mini Project: Async Web Scraper
```

---

# BAB 1: Future & async/await Asas 🌱

## Apa Itu Future?

```
Future<Output = T> adalah "janji" bahawa nilai T akan wujud pada masa depan.

Macam pesan makanan di restoran:
  - Pesan (create Future)   → dapat resit/nombor
  - Tunggu (await)          → duduk, boleh buat benda lain
  - Dapat makanan (resolve) → Future selesai, dapat nilai T

Berbeza dengan blocking:
  - Blocking: Berdiri depan dapur, halang semua orang lain
  - Async:    Duduk, bagi tempat orang lain, tunggu panggilan
```

```rust
// Cara declare async function
async fn sapa(nama: &str) -> String {
    format!("Hello, {}!", nama)
    // Rust transform ini kepada:
    // fn sapa(nama: &str) -> impl Future<Output = String>
}

// MESTI await untuk dapat nilai
#[tokio::main]
async fn main() {
    let mesej = sapa("Ali").await; // .await = tunggu future selesai
    println!("{}", mesej);         // "Hello, Ali!"

    // Tanpa .await — dapat Future, BUKAN nilai!
    let future = sapa("Siti");     // belum jalan lagi!
    let nilai  = future.await;     // baru jalan dan tunggu
    println!("{}", nilai);
}
```

---

## Future Tidak Buat Apa-Apa Tanpa .await

```rust
use tokio::time::{sleep, Duration};

async fn cetak_selepas(ms: u64, mesej: &str) {
    sleep(Duration::from_millis(ms)).await;
    println!("{}", mesej);
}

#[tokio::main]
async fn main() {
    // ⚠ INI TIDAK AKAN PRINT APA-APA!
    let _ = cetak_selepas(100, "Helo"); // create future tapi tak await!

    // ✔ Ini betul
    cetak_selepas(100, "Helo").await;
    println!("Selesai");
}
```

> 💡 **Peraturan #1:** Future dalam Rust adalah **lazy** — ia tidak lakukan apa-apa sehingga di-`await`.
> Ini berbeza dengan JavaScript `Promise` yang terus mula bila dibuat!

---

## async Block

```rust
#[tokio::main]
async fn main() {
    // async block — inline future
    let hasil = async {
        let a = 10;
        let b = 20;
        a + b  // return value dari async block
    }.await;

    println!("Hasil: {}", hasil); // 30

    // async block boleh capture variables
    let nama = "Ali";
    let sapa = async move {
        format!("Salam, {}!", nama) // capture nama dengan move
    }.await;
    println!("{}", sapa);
}
```

---

## 🧠 Brain Teaser #1

Apa output kod ini, dan mengapa?

```rust
use tokio::time::{sleep, Duration};

async fn tugas(id: u32) {
    println!("Mula tugas {}", id);
    sleep(Duration::from_millis(100)).await;
    println!("Siap tugas {}", id);
}

#[tokio::main]
async fn main() {
    tugas(1).await;
    tugas(2).await;
    println!("Semua siap!");
}
```

<details>
<summary>👀 Jawapan</summary>

Output:
```
Mula tugas 1
Siap tugas 1
Mula tugas 2
Siap tugas 2
Semua siap!
```

`.await` satu-satu bermakna **sequential** — tugas 1 mesti siap dulu, baru tugas 2 mula. Walaupun ini async, ia tidak concurrent! Untuk concurrent, guna `tokio::join!` atau `tokio::spawn`.
</details>

---

# BAB 2: Tokio Runtime ⚙️

## Setup Tokio

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
// Cara 1: #[tokio::main] macro — paling biasa
#[tokio::main]
async fn main() {
    println!("Dalam tokio runtime!");
}

// Cara 2: Manual runtime — untuk kawalan lebih
use tokio::runtime::Runtime;

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        println!("Dalam runtime manual!");
    });
}

// Cara 3: multi-thread runtime
#[tokio::main(flavor = "multi_thread", worker_threads = 4)]
async fn main() {
    println!("4 worker threads!");
}

// Cara 4: single-thread runtime (untuk embedded / testing)
#[tokio::main(flavor = "current_thread")]
async fn main() {
    println!("Single thread runtime!");
}
```

---

## Cara Tokio Bekerja

```
Tokio Runtime:

  ┌─────────────────────────────────────────────────┐
  │                  TOKIO RUNTIME                  │
  │                                                 │
  │  Thread Pool (worker threads)                   │
  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐       │
  │  │  T1  │  │  T2  │  │  T3  │  │  T4  │       │
  │  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘       │
  │     │         │          │          │           │
  │  Task Queue (work-stealing)                     │
  │  ┌───────┬───────┬───────┬───────┬───────┐     │
  │  │Task A │Task B │Task C │Task D │Task E │     │
  │  └───────┴───────┴───────┴───────┴───────┘     │
  │                                                 │
  │  I/O Event Loop (epoll/kqueue/IOCP)             │
  └─────────────────────────────────────────────────┘

  Bila task .await (tunggu I/O):
  1. Task yield — bagi thread kepada task lain
  2. I/O event loop pantau descriptor
  3. Bila I/O siap → task masuk queue semula
  4. Worker thread ambil dan teruskan task
```

---

## tokio::spawn — Buat Task Baru

```rust
use tokio::time::{sleep, Duration};

async fn kerja(id: u32, ms: u64) -> String {
    println!("  Task {} mula", id);
    sleep(Duration::from_millis(ms)).await;
    println!("  Task {} siap", id);
    format!("Hasil task {}", id)
}

#[tokio::main]
async fn main() {
    // spawn — jalankan task secara concurrent (tidak tunggu)
    let handle1 = tokio::spawn(kerja(1, 300));
    let handle2 = tokio::spawn(kerja(2, 100));
    let handle3 = tokio::spawn(kerja(3, 200));

    // .await pada JoinHandle untuk tunggu dan dapat hasil
    let r1 = handle1.await.unwrap();
    let r2 = handle2.await.unwrap();
    let r3 = handle3.await.unwrap();

    println!("{}", r1);
    println!("{}", r2);
    println!("{}", r3);

    // Output: Task 2 siap dulu (paling cepat 100ms)
    //         Task 3 siap (200ms)
    //         Task 1 siap (300ms)
    //         — bukan mengikut urutan spawn!
}
```

---

# BAB 3: async Functions & Tasks 🔨

## Sequential vs Concurrent

```rust
use tokio::time::{sleep, Duration, Instant};

async fn muat_turun(nama: &str, ms: u64) -> String {
    println!("  ⬇ Muat turun {} dimulakan...", nama);
    sleep(Duration::from_millis(ms)).await;
    println!("  ✔ {} siap!", nama);
    format!("data:{}", nama)
}

#[tokio::main]
async fn main() {
    // ❌ SEQUENTIAL — lambat!
    let mula = Instant::now();
    let a = muat_turun("fail_A", 300).await;
    let b = muat_turun("fail_B", 200).await;
    let c = muat_turun("fail_C", 100).await;
    println!("Sequential: {}ms\n", mula.elapsed().as_millis()); // ~600ms

    // ✔ CONCURRENT — laju!
    let mula = Instant::now();
    let (a, b, c) = tokio::join!(
        muat_turun("fail_A", 300),
        muat_turun("fail_B", 200),
        muat_turun("fail_C", 100),
    );
    println!("Concurrent: {}ms", mula.elapsed().as_millis()); // ~300ms!
}
```

```
Sequential (total ~600ms):
  [──A 300ms──][──B 200ms──][──C 100ms──]

Concurrent (total ~300ms):
  [──────A 300ms──────]
  [────B 200ms────]
  [──C 100ms──]
  Semua jalan serentak, tunggu yang paling lambat!
```

---

## Perbezaan spawn vs join

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // tokio::join! — tunggu SEMUA selesai, jalankan dalam task yang sama
    let (a, b) = tokio::join!(
        async { sleep(Duration::from_millis(100)).await; "A" },
        async { sleep(Duration::from_millis(200)).await; "B" },
    );
    println!("join: {}, {}", a, b);

    // tokio::spawn — buat task BEBAS, boleh jalan di thread lain
    // Lebih overhead tapi lebih fleksibel
    let h1 = tokio::spawn(async { "spawn A" });
    let h2 = tokio::spawn(async { "spawn B" });
    println!("spawn: {}, {}", h1.await.unwrap(), h2.await.unwrap());

    // Bila guna apa:
    // join! → task berkaitan, nak semua hasil, dalam satu fungsi
    // spawn → task bebas, nak jalankan "background task", thread lain
}
```

---

## 🧠 Brain Teaser #2

Berapa lama (anggaran) masa yang diambil?

```rust
use tokio::time::{sleep, Duration, Instant};

async fn lambat(ms: u64) -> u64 { sleep(Duration::from_millis(ms)).await; ms }

#[tokio::main]
async fn main() {
    let mula = Instant::now();

    let h1 = tokio::spawn(lambat(500));
    let h2 = tokio::spawn(lambat(300));

    let a = lambat(200).await;
    let r1 = h1.await.unwrap();
    let r2 = h2.await.unwrap();

    println!("{}ms", mula.elapsed().as_millis());
}
```

<details>
<summary>👀 Jawapan</summary>

**~500ms**

- `h1` dan `h2` di-spawn serentak (concurrent)
- `lambat(200).await` mula sambil h1 dan h2 jalan
- Selepas 200ms, lambat(200) siap
- `h1.await` — h1 masih jalan, tunggu sehingga 500ms
- `h2.await` — h2 dah siap (300ms < 500ms), terus return

Total = max(500, 300, 200) = **~500ms**

Kalau semua sequential: 500 + 300 + 200 = 1000ms!
</details>

---

# BAB 4: join! & select! — Concurrent Execution 🔀

## tokio::join! — Tunggu Semua

```rust
use tokio::time::{sleep, Duration};

async fn api_pengguna(id: u32) -> String {
    sleep(Duration::from_millis(100)).await;
    format!("Pengguna {}", id)
}

async fn api_pesanan(id: u32) -> Vec<String> {
    sleep(Duration::from_millis(150)).await;
    vec![format!("Pesanan A dari {}", id), format!("Pesanan B dari {}", id)]
}

async fn api_alamat(id: u32) -> String {
    sleep(Duration::from_millis(80)).await;
    format!("Jalan Maju, KL (user {})", id)
}

#[tokio::main]
async fn main() {
    let user_id = 42;

    // Buat 3 API call serentak — selesai dalam ~150ms (paling lambat)
    let (pengguna, pesanan, alamat) = tokio::join!(
        api_pengguna(user_id),
        api_pesanan(user_id),
        api_alamat(user_id),
    );

    println!("Pengguna: {}", pengguna);
    println!("Alamat:   {}", alamat);
    println!("Pesanan:  {:?}", pesanan);

    // join! dengan error handling — guna try_join!
    use tokio::try_join;

    async fn boleh_gagal(gagal: bool) -> Result<&'static str, &'static str> {
        if gagal { Err("Gagal!") } else { Ok("Berjaya") }
    }

    match try_join!(boleh_gagal(false), boleh_gagal(false)) {
        Ok((a, b)) => println!("try_join OK: {}, {}", a, b),
        Err(e)     => println!("try_join ERR: {}", e),
    }

    match try_join!(boleh_gagal(false), boleh_gagal(true)) {
        Ok(_)  => println!("OK"),
        Err(e) => println!("try_join gagal: {}", e), // "Gagal!"
    }
}
```

---

## tokio::select! — Ambil Yang Pertama Siap

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // select! — ambil branch PERTAMA yang selesai, cancel yang lain
    tokio::select! {
        val = async { sleep(Duration::from_millis(200)).await; "lambat" } => {
            println!("Lambat menang: {}", val);
        }
        val = async { sleep(Duration::from_millis(100)).await; "cepat" } => {
            println!("Cepat menang: {}", val); // ← ini yang print!
        }
    }

    // select! dengan timeout pattern
    use tokio::time::timeout;

    let hasil = timeout(
        Duration::from_millis(50),
        async { sleep(Duration::from_millis(200)).await; "data" }
    ).await;

    match hasil {
        Ok(data)   => println!("Dapat: {}", data),
        Err(_)     => println!("Timeout! Terlalu lambat."), // ← ini
    }

    // select! loop — proses mana-mana channel siap dulu
    let mut kiraan = 0u32;
    loop {
        tokio::select! {
            _ = sleep(Duration::from_millis(100)) => {
                kiraan += 1;
                println!("Tick #{}", kiraan);
                if kiraan >= 3 { break; }
            }
            _ = sleep(Duration::from_millis(500)) => {
                println!("Timeout keseluruhan");
                break;
            }
        }
    }
}
```

---

## join! vs select! vs spawn

```
join!(a, b, c)    → Tunggu SEMUA selesai
                    Cancel mana-mana = semua dibatalkan
                    Guna: muat data dari pelbagai sumber

try_join!(a, b)   → Tunggu SEMUA, tapi stop bila ada Error
                    Guna: semua mesti berjaya untuk terus

select!(a, b, c)  → Ambil yang PERTAMA selesai, cancel lain
                    Guna: timeout, race condition, "mana cepat menang"

spawn(a); spawn(b)→ Jalankan BEBAS, boleh await kemudian
                    Guna: background tasks, long-running jobs
```

---

# BAB 5: Channels — Komunikasi Antara Tasks 📬

## mpsc — Multi-Producer Single-Consumer

```
Producer1 ──┐
Producer2 ──┼──▶  [ Channel Buffer ]  ──▶  Consumer
Producer3 ──┘
```

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // Buat channel dengan buffer saiz 32
    let (tx, mut rx) = mpsc::channel::<String>(32);

    // Clone tx untuk beberapa producer
    let tx1 = tx.clone();
    let tx2 = tx.clone();
    drop(tx); // drop tx asal — hanya guna clone

    // Producer 1
    tokio::spawn(async move {
        for i in 1..=3 {
            let mesej = format!("P1: Mesej {}", i);
            tx1.send(mesej).await.unwrap();
            sleep(Duration::from_millis(100)).await;
        }
        println!("Producer 1 selesai");
    });

    // Producer 2
    tokio::spawn(async move {
        for i in 1..=3 {
            let mesej = format!("P2: Mesej {}", i);
            tx2.send(mesej).await.unwrap();
            sleep(Duration::from_millis(150)).await;
        }
        println!("Producer 2 selesai");
    });

    // Consumer — terima sehingga semua sender drop
    while let Some(mesej) = rx.recv().await {
        println!("Terima: {}", mesej);
    }
    println!("Channel ditutup, semua producer selesai");
}
```

---

## oneshot — Satu Nilai Sahaja

```rust
use tokio::sync::oneshot;

async fn buat_kiraan() -> u32 {
    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
    42 // "jawapan kepada segala-galanya"
}

#[tokio::main]
async fn main() {
    // oneshot — hantar SATU nilai dari satu task ke satu task lain
    let (tx, rx) = oneshot::channel::<u32>();

    tokio::spawn(async move {
        let hasil = buat_kiraan().await;
        tx.send(hasil).unwrap(); // hantar sekali
    });

    match rx.await {
        Ok(nilai) => println!("Dapat: {}", nilai), // 42
        Err(_)    => println!("Sender di-drop!"),
    }
}
```

---

## broadcast — Satu ke Semua

```rust
use tokio::sync::broadcast;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // broadcast — satu sender, banyak receiver
    let (tx, _) = broadcast::channel::<String>(16);

    // Buat 3 subscriber
    for i in 1..=3 {
        let mut rx = tx.subscribe();
        tokio::spawn(async move {
            while let Ok(msg) = rx.recv().await {
                println!("  Subscriber {}: terima '{}'", i, msg);
            }
        });
    }

    // Broadcast mesej
    sleep(Duration::from_millis(10)).await; // bagi masa subscriber ready
    tx.send("Salam semua!".to_string()).unwrap();
    tx.send("Mesej kedua".to_string()).unwrap();

    sleep(Duration::from_millis(100)).await;
    // Semua 3 subscriber dapat kedua-dua mesej!
}
```

---

## watch — Track Nilai Terkini

```rust
use tokio::sync::watch;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // watch — subscriber selalu dapat NILAI TERKINI (bukan semua nilai)
    let (tx, rx) = watch::channel("status: starting".to_string());

    // Buat watcher
    let mut rx_clone = rx.clone();
    tokio::spawn(async move {
        loop {
            // changed() — tunggu sehingga nilai berubah
            if rx_clone.changed().await.is_err() { break; }
            println!("  Status berubah: {}", *rx_clone.borrow());
        }
    });

    sleep(Duration::from_millis(50)).await;
    tx.send("status: running".to_string()).unwrap();
    sleep(Duration::from_millis(50)).await;
    tx.send("status: stopping".to_string()).unwrap();
    sleep(Duration::from_millis(50)).await;
    tx.send("status: stopped".to_string()).unwrap();
    sleep(Duration::from_millis(50)).await;
}
```

### Pilih Channel Yang Mana?

```
mpsc (multi-producer, single-consumer):
  → Hantar banyak item dari banyak task ke satu consumer
  → Queue, work dispatch, logging

oneshot:
  → Hantar SATU hasil dari task ke caller
  → Request-response pattern, "future-like" manual

broadcast:
  → Satu mesej ke SEMUA subscriber
  → Event notification, pub/sub

watch:
  → Pantau NILAI TERKINI, subscriber boleh miss intermediate values
  → Config updates, status monitoring
```

---

## 🧠 Brain Teaser #3

Apakah perbezaan antara `mpsc::channel` dan `broadcast::channel`?

```rust
// Senario: 3 subscriber, sender hantar "Hello"
// Guna mpsc vs broadcast — siapa dapat mesej?
```

<details>
<summary>👀 Jawapan</summary>

```
mpsc (multi-producer, single-consumer):
  → Hanya SATU consumer boleh ada
  → "Hello" pergi ke SATU receiver sahaja
  → Macam beli online — satu pembeli dapat barang

broadcast:
  → SEMUA subscriber dapat salinan mesej
  → "Hello" pergi ke subscriber 1, 2, DAN 3
  → Macam siaran radio — semua pendengar dengar sama

mpsc boleh ada BANYAK sender (clone tx):
  Sender1 ──┐
  Sender2 ──┼──▶ [buffer] ──▶ Satu Receiver
  Sender3 ──┘

broadcast ada SATU sender, BANYAK receiver:
  Satu Sender ──▶ [buffer] ──▶ Receiver 1
                           ──▶ Receiver 2
                           ──▶ Receiver 3
```
</details>

---

# BAB 6: Mutex & Shared State 🔒

## Async Mutex

```rust
use tokio::sync::Mutex;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    // Arc<Mutex<T>> — shared mutable state antara tasks
    let kaunter = Arc::new(Mutex::new(0u32));

    let mut handles = vec![];

    // Spawn 10 tasks, semua tambah kaunter
    for i in 0..10 {
        let kaunter_clone = Arc::clone(&kaunter);
        let handle = tokio::spawn(async move {
            let mut guard = kaunter_clone.lock().await; // ambil lock
            *guard += 1;
            println!("  Task {} → kaunter = {}", i, *guard);
            // guard di-drop di sini → lock dilepaskan
        });
        handles.push(handle);
    }

    // Tunggu semua tasks
    for h in handles {
        h.await.unwrap();
    }

    println!("Nilai akhir: {}", *kaunter.lock().await); // 10
}
```

---

## std::sync::Mutex vs tokio::sync::Mutex

```rust
use std::sync::Mutex as StdMutex;
use tokio::sync::Mutex as TokioMutex;

// std::sync::Mutex — untuk operasi CEPAT (tidak ada .await dalam lock)
fn guna_std_mutex() {
    let m = StdMutex::new(0);
    let mut g = m.lock().unwrap(); // blocking lock
    *g += 1;
    // Jangan .await ketika memegang lock ini!
}

// tokio::sync::Mutex — untuk operasi yang ada .await dalam lock
async fn guna_tokio_mutex() {
    let m = TokioMutex::new(0);
    let mut g = m.lock().await; // async lock
    // BOLEH .await di sini — yield kepada task lain tanpa deadlock
    tokio::time::sleep(tokio::time::Duration::from_millis(10)).await;
    *g += 1;
}
```

### Panduan Mutex

```
std::sync::Mutex  → Lock untuk operasi singkat, tiada .await dalam lock scope
tokio::sync::Mutex → Lock yang perlu .await dalam scope (lebih overhead)
RwLock            → Banyak reader OK serentak, hanya satu writer
```

---

## RwLock — Read-Write Lock

```rust
use tokio::sync::RwLock;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    // RwLock: banyak reader serentak, hanya satu writer
    let data = Arc::new(RwLock::new(vec![1, 2, 3]));

    // Banyak reader — boleh concurrent!
    let mut readers = vec![];
    for i in 0..3 {
        let d = Arc::clone(&data);
        readers.push(tokio::spawn(async move {
            let guard = d.read().await; // shared read lock
            println!("  Reader {}: {:?}", i, *guard);
        }));
    }

    for r in readers { r.await.unwrap(); }

    // Writer — exclusive
    {
        let mut guard = data.write().await;
        guard.push(4);
        println!("  Writer tambah 4");
    }

    println!("Data akhir: {:?}", *data.read().await); // [1,2,3,4]
}
```

---

# BAB 7: Async I/O — Fail & Network 🌐

## Baca/Tulis Fail Secara Async

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
use tokio::fs;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Tulis fail
    fs::write("test.txt", "Hello, Async Rust!\nBaris kedua\n").await?;
    println!("Fail ditulis");

    // Baca fail (semua sekaligus)
    let kandungan = fs::read_to_string("test.txt").await?;
    println!("Kandungan:\n{}", kandungan);

    // Baca fail dengan File handle (untuk fail besar)
    let mut fail = fs::File::open("test.txt").await?;
    let mut buffer = Vec::new();
    fail.read_to_end(&mut buffer).await?;
    println!("Bytes: {}", buffer.len());

    // Tulis dengan File handle
    let mut output = fs::File::create("output.txt").await?;
    output.write_all(b"Ditulis dengan async!").await?;
    output.flush().await?;

    // Operasi fail lain
    let metadata = fs::metadata("test.txt").await?;
    println!("Saiz fail: {} bytes", metadata.len());

    fs::remove_file("test.txt").await?;
    fs::remove_file("output.txt").await?;
    println!("Fail dipadam");

    Ok(())
}
```

---

## TCP Server & Client

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{AsyncReadExt, AsyncWriteExt};

// Server
async fn jalankan_server() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("Server mendengar pada port 8080");

    loop {
        let (mut socket, addr) = listener.accept().await?;
        println!("Sambungan baru dari {}", addr);

        // Spawn task untuk setiap connection
        tokio::spawn(async move {
            let mut buf = vec![0; 1024];

            loop {
                match socket.read(&mut buf).await {
                    Ok(0) => {
                        println!("{} putuskan sambungan", addr);
                        break;
                    }
                    Ok(n) => {
                        // Echo balik apa yang diterima
                        let data = &buf[..n];
                        println!("  Terima dari {}: {:?}",
                            addr, String::from_utf8_lossy(data));
                        socket.write_all(data).await.unwrap();
                    }
                    Err(e) => {
                        eprintln!("Error dari {}: {}", addr, e);
                        break;
                    }
                }
            }
        });
    }
}

// Client
async fn jalankan_client() -> Result<(), Box<dyn std::error::Error>> {
    let mut stream = TcpStream::connect("127.0.0.1:8080").await?;
    println!("Disambung ke server");

    stream.write_all(b"Hello, server!").await?;

    let mut buf = vec![0; 1024];
    let n = stream.read(&mut buf).await?;
    println!("Terima echo: {}", String::from_utf8_lossy(&buf[..n]));

    Ok(())
}
```

---

## HTTP Request dengan reqwest

```toml
[dependencies]
tokio  = { version = "1", features = ["full"] }
reqwest = { version = "0.11", features = ["json"] }
serde  = { version = "1", features = ["derive"] }
```

```rust
use reqwest;
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Post {
    id:    u32,
    title: String,
    body:  String,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = reqwest::Client::new();

    // GET request
    let post: Post = client
        .get("https://jsonplaceholder.typicode.com/posts/1")
        .send()
        .await?
        .json()
        .await?;
    println!("Post: {}", post.title);

    // Buat banyak request serentak
    let urls: Vec<String> = (1..=5)
        .map(|i| format!("https://jsonplaceholder.typicode.com/posts/{}", i))
        .collect();

    let futures: Vec<_> = urls.iter()
        .map(|url| {
            let client = client.clone();
            let url = url.clone();
            tokio::spawn(async move {
                client.get(&url).send().await?.json::<Post>().await
            })
        })
        .collect();

    for future in futures {
        match future.await? {
            Ok(post)  => println!("  ID {}: {}", post.id, post.title),
            Err(e)    => eprintln!("  Error: {}", e),
        }
    }

    Ok(())
}
```

---

# BAB 8: Error Handling dalam Async ⚠️

## ? Operator dalam Async

```rust
use tokio::fs;

// async function yang return Result
async fn baca_dan_proses(path: &str) -> Result<usize, std::io::Error> {
    let kandungan = fs::read_to_string(path).await?; // ? bekerja dalam async!
    let baris = kandungan.lines().count();
    Ok(baris)
}

#[tokio::main]
async fn main() {
    match baca_dan_proses("ada.txt").await {
        Ok(n)  => println!("Fail ada {} baris", n),
        Err(e) => println!("Error: {}", e),
    }
}
```

---

## Custom Error dalam Async

```rust
use thiserror::Error;
use tokio::fs;

#[derive(Debug, Error)]
enum AppError {
    #[error("Fail error: {0}")]
    Io(#[from] std::io::Error),

    #[error("Parse error: {0}")]
    Parse(#[from] std::num::ParseIntError),

    #[error("Nilai terlalu besar: {0}")]
    NilaiTerlaluBesar(u32),
}

async fn proses_fail(path: &str) -> Result<u32, AppError> {
    let kandungan = fs::read_to_string(path).await?;      // io::Error → AppError::Io
    let nombor: u32 = kandungan.trim().parse()?;           // ParseIntError → AppError::Parse

    if nombor > 1000 {
        return Err(AppError::NilaiTerlaluBesar(nombor));   // custom error
    }

    Ok(nombor * 2)
}

#[tokio::main]
async fn main() {
    match proses_fail("nombor.txt").await {
        Ok(n)                          => println!("Hasil: {}", n),
        Err(AppError::Io(e))           => println!("Fail error: {}", e),
        Err(AppError::Parse(e))        => println!("Bukan nombor: {}", e),
        Err(AppError::NilaiTerlaluBesar(n)) => println!("{} terlalu besar!", n),
    }
}
```

---

## JoinHandle Error Handling

```rust
use tokio::task::JoinError;

#[tokio::main]
async fn main() {
    // Task yang mungkin panic
    let handle = tokio::spawn(async {
        panic!("Task ini panic!");
    });

    match handle.await {
        Ok(_)  => println!("Task OK"),
        Err(e) if e.is_panic() => println!("Task panic: {:?}", e),
        Err(e) if e.is_cancelled() => println!("Task dibatal"),
        Err(e) => println!("Error lain: {:?}", e),
    }

    // Task yang return Result
    let handle2 = tokio::spawn(async {
        Ok::<i32, &str>(42)
    });

    match handle2.await {
        Ok(Ok(val))   => println!("Dapat: {}", val),  // nested Result!
        Ok(Err(e))    => println!("Task error: {}", e),
        Err(join_err) => println!("Join error: {:?}", join_err),
    }
}
```

---

## 🧠 Brain Teaser #4

Apakah masalah dengan kod ini?

```rust
use std::sync::Mutex;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let m = Mutex::new(0);

    tokio::spawn(async move {
        let mut guard = m.lock().unwrap();
        sleep(Duration::from_millis(100)).await; // ← masalah di sini!
        *guard += 1;
    }).await.unwrap();
}
```

<details>
<summary>👀 Jawapan</summary>

**Masalah:** `std::sync::Mutex` lock dipegang merentasi `.await`!

Bila `sleep().await` dipanggil, task yield kepada runtime. Tetapi `guard` (Mutex lock) masih dipegang. Kalau task lain cuba lock Mutex yang sama, ia akan **deadlock** atau **block thread** kerana Mutex std adalah blocking.

**Penyelesaian 1:** Guna `tokio::sync::Mutex`
```rust
use tokio::sync::Mutex;
let m = Mutex::new(0);
let mut guard = m.lock().await;  // async lock
sleep(...).await;                // OK — yield tapi lock masih "async"
*guard += 1;
```

**Penyelesaian 2:** Lepaskan lock sebelum `.await`
```rust
{
    let mut guard = m.lock().unwrap();
    *guard += 1;
} // guard di-drop di sini, lock dilepaskan
sleep(...).await; // OK — lock dah bebas
```

**Peraturan:** Jangan pegang `std::sync::Mutex` merentasi `.await`!
</details>

---

# BAB 9: Pattern Lanjutan 🎯

## Semaphore — Hadkan Concurrency

```rust
use tokio::sync::Semaphore;
use std::sync::Arc;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // Hadkan kepada 3 request serentak sahaja
    let semaphore = Arc::new(Semaphore::new(3));

    let mut handles = vec![];

    for i in 0..10 {
        let sem = Arc::clone(&semaphore);
        let handle = tokio::spawn(async move {
            let _permit = sem.acquire().await.unwrap(); // tunggu permit
            println!("  Task {} mula (permit diambil)", i);
            sleep(Duration::from_millis(200)).await;
            println!("  Task {} siap (permit dilepas)", i);
            // _permit di-drop → permit kembali ke semaphore
        });
        handles.push(handle);
    }

    for h in handles { h.await.unwrap(); }
    println!("Semua task selesai!");
}
```

---

## Retry dengan Backoff

```rust
use tokio::time::{sleep, Duration};

async fn operasi_yang_mungkin_gagal(percubaan: u32) -> Result<String, String> {
    if percubaan < 3 {
        Err(format!("Gagal percubaan {}", percubaan))
    } else {
        Ok("Berjaya!".to_string())
    }
}

async fn dengan_retry<F, Fut, T, E>(
    mut fungsi: F,
    maks_cuba: u32,
) -> Result<T, E>
where
    F: FnMut(u32) -> Fut,
    Fut: std::future::Future<Output = Result<T, E>>,
    E: std::fmt::Debug,
{
    let mut cuba = 0;
    loop {
        match fungsi(cuba).await {
            Ok(val) => return Ok(val),
            Err(e) if cuba < maks_cuba => {
                let tunggu = Duration::from_millis(100 * 2u64.pow(cuba));
                println!("  Gagal: {:?}. Cuba semula dalam {:?}...", e, tunggu);
                sleep(tunggu).await;
                cuba += 1;
            }
            Err(e) => return Err(e),
        }
    }
}

#[tokio::main]
async fn main() {
    match dengan_retry(operasi_yang_mungkin_gagal, 5).await {
        Ok(v)  => println!("Hasil: {}", v),  // "Berjaya!"
        Err(e) => println!("Gagal semua: {:?}", e),
    }
}
```

---

## Task Cancellation

```rust
use tokio::time::{sleep, Duration};
use tokio_util::sync::CancellationToken;

#[tokio::main]
async fn main() {
    let token = CancellationToken::new();
    let token_clone = token.clone();

    // Task yang boleh dibatal
    let handle = tokio::spawn(async move {
        loop {
            tokio::select! {
                _ = token_clone.cancelled() => {
                    println!("  Task dibatal!");
                    break;
                }
                _ = sleep(Duration::from_millis(100)) => {
                    println!("  Task berjalan...");
                }
            }
        }
    });

    // Biarkan task jalan 350ms
    sleep(Duration::from_millis(350)).await;

    // Batal task
    println!("Membatalkan task...");
    token.cancel();

    handle.await.unwrap();
    println!("Selesai");
}
```

---

## Stream — Async Iterator

```rust
use tokio_stream::{self as stream, StreamExt};
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // Stream dari iterator biasa
    let mut s = stream::iter(vec![1, 2, 3, 4, 5]);
    while let Some(val) = s.next().await {
        println!("Nilai: {}", val);
    }

    // interval stream — emit setiap interval
    use tokio::time::interval;
    let mut ticker = tokio_stream::wrappers::IntervalStream::new(
        interval(Duration::from_millis(100))
    );

    let mut kiraan = 0;
    while let Some(_) = ticker.next().await {
        kiraan += 1;
        println!("Tick {}", kiraan);
        if kiraan >= 5 { break; }
    }

    // Stream operations — macam iterator!
    let hasil: Vec<i32> = stream::iter(1..=10)
        .filter(|&x| x % 2 == 0)
        .map(|x| x * x)
        .collect()
        .await;
    println!("Hasil stream: {:?}", hasil); // [4,16,36,64,100]
}
```

---

# BAB 10: Mini Project — Async Web Scraper 🕷️

```toml
[dependencies]
tokio   = { version = "1", features = ["full"] }
reqwest = { version = "0.11", features = ["json"] }
serde   = { version = "1", features = ["derive"] }
thiserror = "1"
```

```rust
use reqwest::Client;
use serde::Deserialize;
use std::sync::Arc;
use tokio::sync::{Mutex, Semaphore};
use tokio::time::{sleep, Duration, Instant};
use thiserror::Error;

// ─── Error Types ──────────────────────────────────────

#[derive(Debug, Error)]
enum ScraperError {
    #[error("HTTP error: {0}")]
    Http(#[from] reqwest::Error),
    #[error("Timeout selepas {0}ms")]
    Timeout(u64),
    #[error("Status {0}: {1}")]
    StatusKod(u16, String),
}

// ─── Data Models ──────────────────────────────────────

#[derive(Debug, Deserialize, Clone)]
struct Post {
    id:     u32,
    title:  String,
    #[serde(rename = "userId")]
    user_id: u32,
}

#[derive(Debug, Deserialize, Clone)]
struct User {
    id:    u32,
    name:  String,
    email: String,
}

#[derive(Debug)]
struct HasilScrape {
    post:  Post,
    user:  User,
    masa:  u128, // ms
}

// ─── Scraper ──────────────────────────────────────────

struct Scraper {
    client:    Client,
    semaphore: Arc<Semaphore>,       // hadkan concurrent request
    cache:     Arc<Mutex<std::collections::HashMap<u32, User>>>,
    statistik: Arc<Mutex<Statistik>>,
}

#[derive(Debug, Default)]
struct Statistik {
    berjaya: u32,
    gagal:   u32,
    jumlah_ms: u128,
}

impl Scraper {
    fn baru(maks_concurrent: usize) -> Self {
        Scraper {
            client:    Client::builder()
                .timeout(Duration::from_secs(10))
                .build()
                .unwrap(),
            semaphore: Arc::new(Semaphore::new(maks_concurrent)),
            cache:     Arc::new(Mutex::new(std::collections::HashMap::new())),
            statistik: Arc::new(Mutex::new(Statistik::default())),
        }
    }

    async fn ambil_post(&self, id: u32) -> Result<Post, ScraperError> {
        let url = format!(
            "https://jsonplaceholder.typicode.com/posts/{}", id
        );
        let resp = self.client.get(&url).send().await?;

        if !resp.status().is_success() {
            return Err(ScraperError::StatusKod(
                resp.status().as_u16(),
                resp.status().to_string(),
            ));
        }

        Ok(resp.json::<Post>().await?)
    }

    async fn ambil_user(&self, id: u32) -> Result<User, ScraperError> {
        // Semak cache dulu
        {
            let cache = self.cache.lock().await;
            if let Some(user) = cache.get(&id) {
                return Ok(user.clone());
            }
        }

        let url = format!(
            "https://jsonplaceholder.typicode.com/users/{}", id
        );
        let user: User = self.client.get(&url)
            .send().await?
            .json().await?;

        // Simpan dalam cache
        self.cache.lock().await.insert(id, user.clone());
        Ok(user)
    }

    async fn scrape_satu(&self, post_id: u32) -> Result<HasilScrape, ScraperError> {
        let _permit = self.semaphore.acquire().await.unwrap();
        let mula    = Instant::now();

        // Ambil post dan user serentak
        let (post, _) = tokio::join!(
            self.ambil_post(post_id),
            sleep(Duration::from_millis(10)), // simulasi delay
        );
        let post = post?;

        let user = self.ambil_user(post.user_id).await?;
        let masa = mula.elapsed().as_millis();

        // Kemaskini statistik
        let mut stat = self.statistik.lock().await;
        stat.berjaya  += 1;
        stat.jumlah_ms += masa;

        Ok(HasilScrape { post, user, masa })
    }

    async fn scrape_banyak(&self, ids: Vec<u32>) -> Vec<Result<HasilScrape, ScraperError>> {
        let handles: Vec<_> = ids.into_iter()
            .map(|id| {
                let scraper = self as *const Scraper;
                tokio::spawn(async move {
                    // SAFETY: scraper hidup sepanjang fungsi ini
                    unsafe { (*scraper).scrape_satu(id).await }
                })
            })
            .collect();

        let mut hasil = vec![];
        for handle in handles {
            match handle.await {
                Ok(r)  => hasil.push(r),
                Err(e) => {
                    let mut stat = self.statistik.lock().await;
                    stat.gagal += 1;
                    eprintln!("Join error: {:?}", e);
                }
            }
        }
        hasil
    }

    async fn laporan(&self) {
        let stat = self.statistik.lock().await;
        let cache = self.cache.lock().await;

        println!("\n{'═'*50}");
        println!("{:^50}", "LAPORAN SCRAPER");
        println!("{'═'*50}");
        println!("Berjaya:     {}", stat.berjaya);
        println!("Gagal:       {}", stat.gagal);
        println!("Cache hit:   {} users dalam cache", cache.len());
        if stat.berjaya > 0 {
            println!("Purata masa: {}ms per request",
                stat.jumlah_ms / stat.berjaya as u128);
        }
    }
}

// ─── Main ─────────────────────────────────────────────

#[tokio::main]
async fn main() {
    println!("🕷️  Async Web Scraper");
    println!("{'─'*40}");

    let scraper = Scraper::baru(5); // maks 5 concurrent requests
    let mula    = Instant::now();

    // Scrape 10 posts serentak
    let post_ids: Vec<u32> = (1..=10).collect();
    println!("Mula scrape {} posts dengan maks 5 concurrent...\n", post_ids.len());

    let semua_hasil = scraper.scrape_banyak(post_ids).await;

    println!("\n{'─'*40}");
    println!("Keputusan:");

    let mut berjaya = vec![];
    let mut gagal   = vec![];

    for hasil in semua_hasil {
        match hasil {
            Ok(h)  => berjaya.push(h),
            Err(e) => gagal.push(e),
        }
    }

    // Sort by post id
    berjaya.sort_by_key(|h| h.post.id);

    for h in &berjaya {
        println!("  [{}] {} — oleh {} ({}ms)",
            h.post.id,
            &h.post.title[..h.post.title.len().min(40)],
            h.user.name,
            h.masa);
    }

    if !gagal.is_empty() {
        println!("\nGagal:");
        for e in &gagal {
            println!("  ❌ {}", e);
        }
    }

    println!("\nMasa total: {}ms", mula.elapsed().as_millis());
    scraper.laporan().await;
}
```

---

# 📋 Rujukan Pantas — Async Cheat Sheet

## Keywords & Concepts

```
async fn f() → {...}   — declare async function
.await                 — tunggu future selesai
tokio::spawn(fut)      — jalankan future sebagai task bebas
tokio::join!(a, b)     — tunggu SEMUA concurrent
tokio::select!(a, b)   — ambil yang PERTAMA siap
try_join!(a, b)        — tunggu semua, cancel bila ada Err
```

## Channels

```
mpsc::channel(n)       — multi-producer, single-consumer, buffered
oneshot::channel()     — hantar satu nilai
broadcast::channel(n)  — satu sender, banyak receiver
watch::channel(val)    — pantau nilai terkini
```

## Synchronization

```
Mutex<T>               — exclusive access, tokio version untuk .await dalam lock
RwLock<T>              — banyak reader, satu writer
Semaphore::new(n)      — hadkan concurrent access
Arc<T>                 — shared ownership merentasi tasks
```

## Cargo.toml Dependencies

```toml
tokio      = { version = "1", features = ["full"] }
tokio-stream = "0.1"
reqwest    = { version = "0.11", features = ["json"] }
thiserror  = "1"
```

## Peraturan Penting

```
✔ Selalu .await pada Future — ia lazy, tak jalan tanpa await
✔ Guna tokio::sync::Mutex bila ada .await dalam lock scope
✔ Guna tokio::join! untuk concurrent, bukan sequential .await
✔ Arc<T> untuk share data antara tasks
✔ Clone tx untuk mpsc multi-producer
✘ Jangan pegang std::Mutex merentasi .await
✘ Jangan block thread dengan std::thread::sleep — guna tokio::time::sleep
✘ Jangan buat heavy CPU work dalam async fn — guna spawn_blocking
```

## spawn vs join! vs select!

```
tokio::spawn → Task bebas, background, boleh di-cancel
tokio::join! → Semua mesti siap, dalam task yang sama
tokio::select! → Race antara futures, ambil yang pertama
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **Async Download Manager** — muat turun banyak fail serentak, progress bar, retry
2. **Chat Server** — TCP server dengan broadcast channel, banyak client serentak
3. **Rate Limiter** — Semaphore + token bucket untuk hadkan API calls per saat
4. **Async Job Queue** — mpsc channel, worker pool, priority queue dengan BinaryHeap
5. **Health Checker** — check banyak URL serentak, laporan status, alerting

---

*Async Programming in Rust — dari `async fn` hingga concurrent patterns production-ready.* 🦀
