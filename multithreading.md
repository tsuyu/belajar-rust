# 🧵 Multithreading dalam Rust — Beginner to Advanced

> Panduan lengkap multithreading: dari thread asas hingga thread pool dan parallel patterns.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu Multithreading?

```
Single Thread — satu kerja pada satu masa:

  Thread 1: ──[A]──[B]──[C]──[D]──[E]──▶
  Masa:       1s   2s   3s   4s   5s    = 5 saat

Multi Thread — banyak kerja serentak:

  Thread 1: ──[A]──[B]──▶
  Thread 2: ──[C]──[D]──▶   Semua siap dalam ~2 saat!
  Thread 3: ──[E]────────▶

Guna:
  ✔ CPU-intensive tasks (compute, encode, compress)
  ✔ Parallel data processing
  ✔ Background tasks
  ✔ Kepantasan pada multi-core CPU
```

---

## Kenapa Multithreading Rust Istimewa?

```
Masalah multithreading dalam bahasa lain:
  ❌ Data Race  — dua thread tulis ke data sama serentak
  ❌ Deadlock   — thread A tunggu B, B tunggu A → stuck!
  ❌ Use-after-free — akses data selepas thread lain buang

Rust selesaikan ini di COMPILE TIME:
  ✔ Ownership sistem → tiada data race
  ✔ Send + Sync traits → compiler check thread safety
  ✔ Mutex/RwLock → akses selamat
  ✔ Channel → komunikasi selamat

"Fearless Concurrency" — berani tulis concurrent code!
```

---

## Peta Pembelajaran

```
Bab 1  → std::thread Asas
Bab 2  → JoinHandle & Return Values
Bab 3  → Move Closures & Ownership
Bab 4  → Arc & Shared State
Bab 5  → Mutex — Thread-Safe Mutability
Bab 6  → RwLock — Many Readers, One Writer
Bab 7  → Channels — Komunikasi Antara Thread
Bab 8  → Condvar & Barrier
Bab 9  → Thread Pool & Rayon
Bab 10 → Mini Project: Parallel File Processor
```

---

# BAB 1: std::thread Asas 🧵

## Spawn Thread Pertama

```rust
use std::thread;
use std::time::Duration;

fn main() {
    // Spawn thread baru
    let handle = thread::spawn(|| {
        for i in 1..=5 {
            println!("  Thread anak: {}", i);
            thread::sleep(Duration::from_millis(50));
        }
    });

    // Thread utama terus jalan
    for i in 1..=3 {
        println!("Thread utama: {}", i);
        thread::sleep(Duration::from_millis(80));
    }

    // Tunggu thread anak selesai
    handle.join().unwrap();
    println!("Semua selesai!");
}
```

```
Output (campur-aduk, tidak dijamin order):
  Thread utama: 1
  Thread anak: 1
  Thread anak: 2
  Thread utama: 2
  Thread anak: 3
  Thread utama: 3
  Thread anak: 4
  Thread anak: 5
  Semua selesai!
```

---

## Thread tanpa join — Bahaya!

```rust
use std::thread;
use std::time::Duration;

fn main() {
    // ⚠ TANPA join — thread mungkin tidak selesai!
    thread::spawn(|| {
        thread::sleep(Duration::from_millis(500));
        println!("Thread anak selesai"); // Mungkin tidak print!
    });

    println!("Main selesai"); // Main keluar → thread anak dibunuh!
}

// Output yang mungkin:
//   Main selesai
//   (thread anak dibunuh sebelum sempat print)
```

> ⚠ **Peraturan:** Selalu `join()` thread yang di-spawn kecuali ada sebab khusus tidak perlu!

---

## Spawn Banyak Thread

```rust
use std::thread;

fn main() {
    let mut handles = vec![];

    // Spawn 5 thread serentak
    for i in 0..5 {
        let handle = thread::spawn(move || {
            println!("Thread {} mula", i);
            // Simulate kerja
            let result = i * i;
            println!("Thread {} siap: {} × {} = {}", i, i, i, result);
            result // return nilai
        });
        handles.push(handle);
    }

    // Kumpul semua hasil
    let hasil: Vec<u32> = handles
        .into_iter()
        .map(|h| h.join().unwrap())
        .collect();

    println!("Semua hasil: {:?}", hasil);
    println!("Jumlah: {}", hasil.iter().sum::<u32>());
}
```

---

## Thread Config — Builder Pattern

```rust
use std::thread;

fn main() {
    // Kustomkan thread dengan Builder
    let builder = thread::Builder::new()
        .name("pekerja-utama".to_string())  // nama thread
        .stack_size(4 * 1024 * 1024);       // 4MB stack (default ~8MB)

    let handle = builder.spawn(|| {
        let nama = thread::current().name().unwrap_or("?").to_string();
        println!("Thread '{}' berjalan", nama);

        // Dapatkan info thread semasa
        println!("Thread ID: {:?}", thread::current().id());
    }).unwrap();

    handle.join().unwrap();

    // Bilangan logical CPU
    println!("Bilangan CPU: {}", thread::available_parallelism().unwrap());
}
```

---

## 🧠 Brain Teaser #1

Apa output kod ini, dan mengapa urutan mungkin berbeza setiap kali?

```rust
use std::thread;

fn main() {
    let h1 = thread::spawn(|| println!("A"));
    let h2 = thread::spawn(|| println!("B"));
    let h3 = thread::spawn(|| println!("C"));

    h1.join().unwrap();
    h2.join().unwrap();
    h3.join().unwrap();

    println!("D");
}
```

<details>
<summary>👀 Jawapan</summary>

Output dijamin:
- `D` sentiasa **terakhir** — hanya print selepas h1, h2, h3 semua di-join
- `A`, `B`, `C` dalam **mana-mana urutan** — bergantung pada OS scheduler

Kemungkinan output:
```
A B C D   atau   B A C D   atau   C A B D   dll.
```

Walaupun kita `join` dalam urutan h1→h2→h3, thread-thread ini sudah di-spawn serentak. OS memutuskan mana yang jalan dulu. `join` hanya tunggu thread tersebut selesai — ia tidak paksa thread jalan dalam urutan tertentu.
</details>

---

# BAB 2: JoinHandle & Return Values ↩️

## Tangani Hasil Thread

```rust
use std::thread;

fn main() {
    // Thread yang return nilai
    let handle = thread::spawn(|| {
        let mut jumlah = 0u64;
        for i in 1..=1_000_000u64 {
            jumlah += i;
        }
        jumlah  // return nilai dari thread
    });

    // Buat kerja lain ketika thread jalan...
    println!("Mengira... sila tunggu");

    // Dapatkan hasil
    let jumlah = handle.join().unwrap();
    println!("Jumlah 1..1,000,000 = {}", jumlah); // 500000500000
}
```

---

## Tangani Panic dalam Thread

```rust
use std::thread;

fn main() {
    // Thread yang mungkin panic
    let handle = thread::spawn(|| {
        println!("Thread mula");
        panic!("Sesuatu rosak dalam thread!");
    });

    // join() return Result — Err jika thread panic
    match handle.join() {
        Ok(val)  => println!("Thread berjaya: {:?}", val),
        Err(e)   => {
            // e adalah Box<dyn Any> — payload panic
            if let Some(s) = e.downcast_ref::<&str>() {
                println!("Thread panic dengan mesej: {}", s);
            } else if let Some(s) = e.downcast_ref::<String>() {
                println!("Thread panic: {}", s);
            } else {
                println!("Thread panic dengan nilai tidak dikenali");
            }
        }
    }

    println!("Program terus jalan walaupun thread panic!");
}
```

> 💡 **Penting:** Panic dalam child thread **tidak mematikan** main thread! Main thread terus jalan.
> Hanya bila **main thread** panic, semua thread lain dibunuh.

---

## Kumpul Hasil dari Banyak Thread

```rust
use std::thread;

fn kira_prima(mula: u64, tamat: u64) -> Vec<u64> {
    (mula..=tamat)
        .filter(|&n| {
            if n < 2 { return false; }
            if n == 2 { return true; }
            if n % 2 == 0 { return false; }
            !(3..=(n as f64).sqrt() as u64)
                .step_by(2)
                .any(|i| n % i == 0)
        })
        .collect()
}

fn main() {
    let had = 100_000u64;
    let n_thread = 4;
    let saiz_bahagian = had / n_thread;

    let mut handles = vec![];

    for i in 0..n_thread {
        let mula = i * saiz_bahagian + 1;
        let tamat = if i == n_thread - 1 { had } else { (i + 1) * saiz_bahagian };

        let handle = thread::spawn(move || {
            println!("Thread {} mengira prima {}..{}", i, mula, tamat);
            kira_prima(mula, tamat)
        });
        handles.push(handle);
    }

    // Kumpul dan gabung semua hasil
    let mut semua_prima: Vec<u64> = handles
        .into_iter()
        .flat_map(|h| h.join().unwrap())
        .collect();

    semua_prima.sort();
    println!("Jumlah prima sehingga {}: {}", had, semua_prima.len());
    println!("10 prima pertama: {:?}", &semua_prima[..10]);
    println!("10 prima terakhir: {:?}", &semua_prima[semua_prima.len()-10..]);
}
```

---

# BAB 3: Move Closures & Ownership 🏠

## Masalah: Thread Perlu Own Data

```rust
use std::thread;

fn main() {
    let mesej = String::from("Hello dari thread!");

    // ❌ TIDAK COMPILE — thread mungkin hidup lebih lama dari mesej
    // let handle = thread::spawn(|| {
    //     println!("{}", mesej); // ERROR: may outlive borrowed value
    // });

    // ✔ Guna 'move' — pindah ownership ke thread
    let handle = thread::spawn(move || {
        println!("{}", mesej); // mesej kini dimiliki thread ini
    });

    // println!("{}", mesej); // ← ERROR jika uncomment! mesej dah moved

    handle.join().unwrap();
}
```

---

## move dengan Pelbagai Types

```rust
use std::thread;

fn main() {
    let nombor = 42i32;            // Copy type — di-copy, bukan moved
    let teks   = String::from("hello"); // non-Copy — di-moved
    let vec    = vec![1, 2, 3];    // non-Copy — di-moved

    let handle = thread::spawn(move || {
        println!("nombor: {}", nombor); // copy
        println!("teks:   {}", teks);   // moved
        println!("vec:    {:?}", vec);  // moved
        nombor + vec.iter().sum::<i32>()
    });

    // nombor masih boleh guna (Copy)
    println!("Nombor asal: {}", nombor);

    // teks dan vec tidak boleh guna lagi (moved ke thread)
    // println!("{}", teks); // ← ERROR!

    let hasil = handle.join().unwrap();
    println!("Hasil: {}", hasil); // 42 + 6 = 48
}
```

---

## Clone untuk Berkongsi Data

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3, 4, 5];

    // Jika perlu data dalam thread DAN dalam main:
    let data_klon = data.clone(); // clone sebelum move!

    let handle = thread::spawn(move || {
        let jumlah: i32 = data_klon.iter().sum();
        println!("Thread: jumlah = {}", jumlah);
        jumlah
    });

    // Main masih ada data asal
    println!("Main: {:?}", data);

    handle.join().unwrap();
}
```

---

## 🧠 Brain Teaser #2

Mengapa kod ini tidak compile?

```rust
use std::thread;

fn main() {
    let mut v = vec![1, 2, 3];

    let h1 = thread::spawn(move || {
        v.push(4);
        println!("{:?}", v);
    });

    let h2 = thread::spawn(move || {
        v.push(5);           // ← masalah di sini
        println!("{:?}", v);
    });

    h1.join().unwrap();
    h2.join().unwrap();
}
```

<details>
<summary>👀 Jawapan</summary>

**Error:** `v` sudah di-move ke thread h1 (closure pertama) — tidak boleh di-move lagi ke h2!

```
error[E0382]: use of moved value: `v`
  --> h2 closure: v.push(5)
```

**Penyelesaian bergantung pada keperluan:**

**1. Kalau tidak perlu serentak** — jalankan sequential:
```rust
let h1 = thread::spawn(move || { v.push(4); v });
let v = h1.join().unwrap(); // ambil v balik
let h2 = thread::spawn(move || { v.push(5); v });
let v = h2.join().unwrap();
```

**2. Kalau perlu serentak + share** — guna `Arc<Mutex<Vec<i32>>>`:
```rust
use std::sync::{Arc, Mutex};
let v = Arc::new(Mutex::new(vec![1, 2, 3]));
let v1 = Arc::clone(&v);
let v2 = Arc::clone(&v);
thread::spawn(move || { v1.lock().unwrap().push(4); });
thread::spawn(move || { v2.lock().unwrap().push(5); });
```
</details>

---

# BAB 4: Arc — Shared Ownership Merentasi Thread 🔗

## Arc = Atomic Reference Counting

```
Rc<T>  → banyak owner, SATU thread sahaja
Arc<T> → banyak owner, BANYAK thread (thread-safe!)

Arc menggunakan atomic operations untuk kira reference — lebih
lambat dari Rc, tapi selamat untuk multi-thread.
```

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    // Data yang dikongsi semua thread
    let konfigurasi = Arc::new(vec![
        ("host", "localhost"),
        ("port", "8080"),
        ("db",   "kada_db"),
    ]);

    let mut handles = vec![];

    for i in 0..4 {
        // Clone Arc — tambah reference count (bukan clone data!)
        let config = Arc::clone(&konfigurasi);

        let handle = thread::spawn(move || {
            println!("Thread {}: host = {}", i, config[0].1);
            println!("Thread {}: port = {}", i, config[1].1);
        });
        handles.push(handle);
    }

    for h in handles { h.join().unwrap(); }

    // Arc masih ada selepas semua thread selesai
    println!("Arc count: {}", Arc::strong_count(&konfigurasi)); // 1
    println!("Config masih ada: {:?}", konfigurasi[0]);
}
```

---

## Arc dengan Struct

```rust
use std::sync::Arc;
use std::thread;

#[derive(Debug)]
struct Konfigurasi {
    host:     String,
    port:     u16,
    max_conn: u32,
}

impl Konfigurasi {
    fn url(&self) -> String {
        format!("{}:{}", self.host, self.port)
    }
}

fn main() {
    let config = Arc::new(Konfigurasi {
        host:     "localhost".into(),
        port:     5432,
        max_conn: 100,
    });

    let mut handles = vec![];

    for worker_id in 0..3 {
        let cfg = Arc::clone(&config);
        handles.push(thread::spawn(move || {
            println!("Worker {} sambung ke {}", worker_id, cfg.url());
        }));
    }

    for h in handles { h.join().unwrap(); }
}
```

> ⚠ **Arc hanya immutable!** Untuk ubah data yang dikongsi, perlukan `Arc<Mutex<T>>` atau `Arc<RwLock<T>>`.

---

# BAB 5: Mutex — Thread-Safe Mutability 🔒

## Apa Itu Mutex?

```
Mutex = Mutual Exclusion

  Thread 1: ──[kunci]──[ubah data]──[buka]──▶
  Thread 2: ──────────────[tunggu]──[kunci]──[ubah]──[buka]──▶
  Thread 3: ──────────────────────────────────[tunggu]──...

  Hanya SATU thread boleh pegang lock pada satu masa.
  Yang lain tunggu giliran.

Rust Mutex:
  .lock()   → dapat MutexGuard (auto unlock bila di-drop)
  .try_lock() → cuba lock, return Err jika sudah locked
```

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    // Arc<Mutex<T>> — pattern standard untuk shared mutable state
    let kaunter = Arc::new(Mutex::new(0u32));
    let mut handles = vec![];

    for _ in 0..10 {
        let k = Arc::clone(&kaunter);
        handles.push(thread::spawn(move || {
            let mut guard = k.lock().unwrap(); // ambil lock → block jika dah locked
            *guard += 1;
            println!("  Tambah → {}", *guard);
            // guard di-drop di sini → lock dilepaskan AUTO!
        }));
    }

    for h in handles { h.join().unwrap(); }

    println!("Nilai akhir: {}", *kaunter.lock().unwrap()); // 10
}
```

---

## Mutex Poisoning

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(vec![1, 2, 3]));
    let data_klon = Arc::clone(&data);

    // Thread yang panic semasa memegang lock → Mutex "poisoned"!
    let handle = thread::spawn(move || {
        let _guard = data_klon.lock().unwrap();
        panic!("Panic semasa pegang lock!");
        // _guard di-drop semasa unwind → Mutex jadi poisoned
    });

    let _ = handle.join(); // Tangani panic

    // Cuba guna Mutex yang poisoned
    match data.lock() {
        Ok(guard)  => println!("OK: {:?}", *guard),
        Err(poisoned) => {
            println!("Mutex poisoned! Tapi data masih ada.");
            // Boleh recover dengan into_inner()
            let guard = poisoned.into_inner();
            println!("Data recover: {:?}", *guard);
        }
    }
}
```

---

## Deadlock — Bahaya!

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let m1 = Arc::new(Mutex::new(1));
    let m2 = Arc::new(Mutex::new(2));

    let m1a = Arc::clone(&m1);
    let m2a = Arc::clone(&m2);

    // Thread A: ambil m1 dulu, kemudian m2
    let ha = thread::spawn(move || {
        let _g1 = m1a.lock().unwrap();
        println!("Thread A dapat m1");
        thread::sleep(std::time::Duration::from_millis(10));
        let _g2 = m2a.lock().unwrap(); // tunggu m2...
        println!("Thread A dapat m2");
    });

    let m1b = Arc::clone(&m1);
    let m2b = Arc::clone(&m2);

    // Thread B: ambil m2 dulu, kemudian m1 → DEADLOCK!
    let hb = thread::spawn(move || {
        let _g2 = m2b.lock().unwrap();
        println!("Thread B dapat m2");
        thread::sleep(std::time::Duration::from_millis(10));
        let _g1 = m1b.lock().unwrap(); // tunggu m1... tapi Thread A pegang m1!
        println!("Thread B dapat m1");
    });

    // Program akan STUCK di sini — deadlock!
    // ha.join().unwrap();
    // hb.join().unwrap();

    println!("(Elak join untuk demonstrasi — program akan hang jika join)");
}

// Cara elak deadlock:
// 1. Sentiasa lock dalam URUTAN yang sama (semua thread lock m1 dulu, kemudian m2)
// 2. Guna try_lock() dengan timeout
// 3. Kurangkan scope lock (lepaskan lock ASAP)
// 4. Reka semula supaya kurang lock diperlukan
```

---

## try_lock — Non-Blocking Lock

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

fn main() {
    let data = Arc::new(Mutex::new(0u32));
    let data2 = Arc::clone(&data);

    // Thread yang pegang lock lama
    let h = thread::spawn(move || {
        let _guard = data2.lock().unwrap();
        thread::sleep(Duration::from_millis(200));
        // lock dilepas selepas 200ms
    });

    thread::sleep(Duration::from_millis(50)); // bagi masa thread ambil lock

    // Cuba lock tanpa block
    match data.try_lock() {
        Ok(guard)  => println!("Dapat lock: {}", *guard),
        Err(_)     => println!("Lock sedang dipegang — cuba lagi nanti"),
    }

    h.join().unwrap();

    // Sekarang lock bebas
    match data.try_lock() {
        Ok(guard) => println!("Sekarang dapat: {}", *guard),
        Err(_)    => println!("Masih locked?"),
    }
}
```

---

# BAB 6: RwLock — Many Readers, One Writer 📖

## Bila RwLock Lebih Baik dari Mutex

```
Mutex:  Hanya SATU thread boleh akses pada satu masa
        (sama ada baca ATAU tulis)

RwLock: BANYAK thread boleh BACA serentak
        HANYA satu thread TULIS (eksklusif)

Bila RwLock lebih baik:
  ✔ Baca lebih kerap dari tulis (read-heavy workload)
  ✔ Contoh: cache, konfigurasi, direktori
```

```rust
use std::sync::{Arc, RwLock};
use std::thread;
use std::time::Duration;

fn main() {
    let data = Arc::new(RwLock::new(vec!["Ali", "Siti", "Amin"]));
    let mut handles = vec![];

    // Spawn 5 reader thread
    for i in 0..5 {
        let d = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            let guard = d.read().unwrap(); // shared read lock
            println!("Reader {}: {:?}", i, *guard);
            thread::sleep(Duration::from_millis(50));
            // Banyak reader boleh hold lock serentak!
        }));
    }

    // Spawn 1 writer thread
    let d = Arc::clone(&data);
    handles.push(thread::spawn(move || {
        thread::sleep(Duration::from_millis(10)); // bagi reader mula dulu
        let mut guard = d.write().unwrap(); // exclusive write lock
        guard.push("Zara");
        println!("Writer tambah Zara");
        // Tunggu semua reader lepas lock dulu!
    }));

    for h in handles { h.join().unwrap(); }

    println!("Akhir: {:?}", *data.read().unwrap());
}
```

---

## RwLock Pattern — Cache

```rust
use std::sync::{Arc, RwLock};
use std::collections::HashMap;
use std::thread;

struct Cache {
    data: RwLock<HashMap<String, String>>,
}

impl Cache {
    fn baru() -> Arc<Self> {
        Arc::new(Cache {
            data: RwLock::new(HashMap::new()),
        })
    }

    fn dapatkan(&self, kunci: &str) -> Option<String> {
        let guard = self.data.read().unwrap(); // shared read
        guard.get(kunci).cloned()
    }

    fn tetapkan(&self, kunci: &str, nilai: &str) {
        let mut guard = self.data.write().unwrap(); // exclusive write
        guard.insert(kunci.to_string(), nilai.to_string());
    }
}

fn main() {
    let cache = Cache::baru();
    let mut handles = vec![];

    // Isi cache
    let c = Arc::clone(&cache);
    handles.push(thread::spawn(move || {
        c.tetapkan("user:1", "Ali Ahmad");
        c.tetapkan("user:2", "Siti Hawa");
        println!("Cache diisi");
    }));

    handles[0].join().unwrap();
    handles.clear();

    // Banyak reader serentak
    for i in 1..=3 {
        let c = Arc::clone(&cache);
        let kunci = format!("user:{}", i);
        handles.push(thread::spawn(move || {
            match c.dapatkan(&kunci) {
                Some(v) => println!("Thread {}: {} = {}", i, kunci, v),
                None    => println!("Thread {}: {} tidak ada", i, kunci),
            }
        }));
    }

    for h in handles { h.join().unwrap(); }
}
```

---

## 🧠 Brain Teaser #3

Bila nak guna `Mutex<T>` dan bila `RwLock<T>`?

```
Senario:
A. Konfigurasi app yang dibaca setiap request, diubah setiap restart
B. Kaunter yang perlu ditambah setiap request
C. Log buffer yang banyak thread tulis
D. User database yang lebih banyak query dari insert
```

<details>
<summary>👀 Jawapan</summary>

```
A. Konfigurasi → RwLock ✔
   Banyak baca (setiap request), jarang tulis (restart sahaja)
   Reader boleh serentak → prestasi lebih baik

B. Kaunter → Mutex ✔ atau AtomicU32 (lebih baik)
   Setiap operasi adalah tulis → RwLock tiada kelebihan
   Atomic types lebih laju untuk operasi mudah

C. Log buffer → Mutex ✔
   Tulis kerap, baca jarang → RwLock tiada kelebihan
   Mutex lebih mudah dan overhead lebih rendah

D. User database → RwLock ✔
   Read-heavy (query) → banyak reader serentak
   Write-jarang (insert) → writer tunggu reader siap

Panduan ringkas:
  Tulis kerap / sama banyak dengan baca → Mutex
  Baca lebih kerap dari tulis → RwLock
  Operasi atom mudah (increment, compare-swap) → Atomic types
```
</details>

---

# BAB 7: Channels — Komunikasi Antara Thread 📬

## mpsc — Multi-Producer Single-Consumer

```
Konsep: Hantar mesej, bukan share data (message passing)

  Thread 1 ──tx──┐
  Thread 2 ──tx──┼──▶  [Channel Buffer]  ──rx──▶  Receiver
  Thread 3 ──tx──┘

"Do not communicate by sharing memory;
 share memory by communicating." — Go proverb, valid untuk Rust juga
```

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    // Buat channel
    let (tx, rx) = mpsc::channel::<String>();

    // Clone tx untuk beberapa producer
    let tx1 = tx.clone();
    let tx2 = tx.clone();
    drop(tx); // buang tx asal, hanya guna clone

    // Producer 1
    thread::spawn(move || {
        for i in 1..=3 {
            let mesej = format!("P1 mesej {}", i);
            tx1.send(mesej).unwrap();
            thread::sleep(Duration::from_millis(100));
        }
    });

    // Producer 2
    thread::spawn(move || {
        for i in 1..=3 {
            let mesej = format!("P2 mesej {}", i);
            tx2.send(mesej).unwrap();
            thread::sleep(Duration::from_millis(150));
        }
    });

    // Consumer — terima sehingga semua tx drop
    for mesej in rx {
        println!("Terima: {}", mesej);
    }
    println!("Semua mesej diterima!");
}
```

---

## Bounded Channel — Backpressure

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    // sync_channel(n) — buffer terhad, sender block jika penuh!
    let (tx, rx) = mpsc::sync_channel::<i32>(3); // buffer 3 sahaja

    let handle = thread::spawn(move || {
        for i in 1..=6 {
            println!("Hantar {}", i);
            tx.send(i).unwrap(); // block jika buffer penuh!
            println!("Hantar {} berjaya", i);
        }
    });

    thread::sleep(std::time::Duration::from_millis(200));

    for v in rx {
        println!("  Terima: {}", v);
        thread::sleep(std::time::Duration::from_millis(100));
    }

    handle.join().unwrap();
}
```

---

## Channel untuk Work Distribution

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let n_pekerja = 4;
    let (tx_kerja, rx_kerja) = mpsc::channel::<Option<u64>>();
    let (tx_hasil, rx_hasil) = mpsc::channel::<(u64, u64)>();

    // Spawn worker threads
    for id in 0..n_pekerja {
        let rx = {
            // Setiap worker dapat receiver yang sama
            // Guna Arc<Mutex<Receiver>> untuk share receiver
            use std::sync::{Arc, Mutex};
            // Simplified: guna channel berasingan per worker
            let (tx2, rx2) = mpsc::channel::<Option<u64>>();
            (tx2, rx2)
        };
        // (Simplified demo — production guna rayon atau thread pool)
    }

    // Pattern lebih mudah: hantar kerja, tunggu hasil
    let (tx_k, rx_k) = mpsc::channel::<u64>();
    let (tx_h, rx_h) = mpsc::channel::<(u64, bool)>();

    // Spawn thread untuk semak nombor perdana
    for _ in 0..n_pekerja {
        let rx = rx_k.clone(); // ← TIDAK boleh clone standard Receiver
        // Gunakan Arc<Mutex<Receiver>> untuk share
    }

    // Cara betul dengan Arc<Mutex<Receiver>>:
    use std::sync::{Arc, Mutex};

    let (tx_tugas, rx_tugas) = mpsc::channel::<Option<u64>>();
    let rx_tugas = Arc::new(Mutex::new(rx_tugas));
    let (tx_r, rx_r) = mpsc::channel::<u64>();

    // Worker threads
    let mut handles = vec![];
    for id in 0..n_pekerja {
        let rx = Arc::clone(&rx_tugas);
        let tx = tx_r.clone();
        handles.push(thread::spawn(move || {
            loop {
                let tugas = rx.lock().unwrap().recv();
                match tugas {
                    Ok(Some(n)) => {
                        if adalah_perdana(n) {
                            tx.send(n).unwrap();
                        }
                    }
                    _ => break, // None atau error → berhenti
                }
            }
        }));
    }
    drop(tx_r);

    // Hantar tugas
    for n in 1..=100u64 {
        tx_tugas.send(Some(n)).unwrap();
    }
    // Hantar signal berhenti
    for _ in 0..n_pekerja {
        tx_tugas.send(None).unwrap();
    }
    drop(tx_tugas);

    // Kumpul hasil
    let mut prima: Vec<u64> = rx_r.iter().collect();
    prima.sort();
    println!("Prima 1..100: {:?}", prima);

    for h in handles { h.join().unwrap(); }
}

fn adalah_perdana(n: u64) -> bool {
    if n < 2 { return false; }
    if n == 2 { return true; }
    if n % 2 == 0 { return false; }
    !(3..=(n as f64).sqrt() as u64).step_by(2).any(|i| n % i == 0)
}
```

---

# BAB 8: Condvar & Barrier 🚧

## Condvar — Conditional Variable

```rust
use std::sync::{Arc, Mutex, Condvar};
use std::thread;

fn main() {
    // Condvar: tunggu sehingga syarat dipenuhi
    let pasangan = Arc::new((Mutex::new(false), Condvar::new()));
    let pasangan2 = Arc::clone(&pasangan);

    // Thread pekerja — tunggu isyarat
    thread::spawn(move || {
        let (kunci, cvar) = &*pasangan2;
        let mut bersedia = kunci.lock().unwrap();

        // Tunggu sehingga bersedia == true
        while !*bersedia {
            bersedia = cvar.wait(bersedia).unwrap();
        }
        println!("Pekerja: isyarat diterima, mula bekerja!");
    });

    // Main thread — bagi isyarat selepas beberapa masa
    thread::sleep(std::time::Duration::from_millis(100));

    let (kunci, cvar) = &*pasangan;
    let mut bersedia = kunci.lock().unwrap();
    *bersedia = true;
    cvar.notify_one(); // bangunkan satu thread yang menunggu
    println!("Main: isyarat dihantar!");
    drop(bersedia);

    thread::sleep(std::time::Duration::from_millis(50));
}
```

---

## Barrier — Tunggu Semua Thread

```rust
use std::sync::{Arc, Barrier};
use std::thread;
use std::time::Duration;

fn main() {
    let n_thread = 5;
    // Barrier: semua thread mesti sampai checkpoint sebelum terus
    let barrier = Arc::new(Barrier::new(n_thread));
    let mut handles = vec![];

    for i in 0..n_thread {
        let b = Arc::clone(&barrier);
        handles.push(thread::spawn(move || {
            // Fasa 1: setiap thread buat kerja berbeza
            let masa_kerja = (i + 1) as u64 * 50;
            thread::sleep(Duration::from_millis(masa_kerja));
            println!("Thread {} selesai fasa 1 ({}ms)", i, masa_kerja);

            // Tunggu SEMUA thread selesai fasa 1
            let result = b.wait();
            if result.is_leader() {
                println!("→ Semua thread selesai fasa 1! Mula fasa 2.");
            }

            // Fasa 2: semua mula serentak
            println!("Thread {} mula fasa 2", i);
        }));
    }

    for h in handles { h.join().unwrap(); }
    println!("Semua selesai!");
}
```

---

## AtomicUsize — Lock-Free Counter

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    // Atomic — operasi thread-safe TANPA lock!
    let kaunter = Arc::new(AtomicUsize::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let k = Arc::clone(&kaunter);
        handles.push(thread::spawn(move || {
            // fetch_add — tambah dan return nilai LAMA
            let sebelum = k.fetch_add(1, Ordering::SeqCst);
            println!("  Sebelum: {}", sebelum);
        }));
    }

    for h in handles { h.join().unwrap(); }

    println!("Kaunter akhir: {}", kaunter.load(Ordering::SeqCst)); // 10

    // Perbandingan Mutex vs Atomic untuk kaunter
    // Atomic: lebih laju, untuk operasi mudah (increment, decrement)
    // Mutex:  lebih fleksibel, untuk operasi kompleks
}
```

---

# BAB 9: Thread Pool & Rayon 🏊

## Thread Pool Manual

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::collections::VecDeque;

type Kerja = Box<dyn FnOnce() + Send + 'static>;

struct ThreadPool {
    pekerja:   Vec<thread::JoinHandle<()>>,
    tx:        std::sync::mpsc::Sender<Option<Kerja>>,
}

impl ThreadPool {
    fn baru(saiz: usize) -> Self {
        let (tx, rx) = std::sync::mpsc::channel::<Option<Kerja>>();
        let rx = Arc::new(Mutex::new(rx));
        let mut pekerja = vec![];

        for id in 0..saiz {
            let rx = Arc::clone(&rx);
            pekerja.push(thread::spawn(move || {
                loop {
                    let kerja = rx.lock().unwrap().recv();
                    match kerja {
                        Ok(Some(f)) => {
                            println!("  Pekerja {} laksana tugas", id);
                            f();
                        }
                        _ => {
                            println!("  Pekerja {} berhenti", id);
                            break;
                        }
                    }
                }
            }));
        }

        ThreadPool { pekerja, tx }
    }

    fn laksana<F>(&self, f: F)
    where F: FnOnce() + Send + 'static
    {
        self.tx.send(Some(Box::new(f))).unwrap();
    }

    fn tutup(self) {
        for _ in &self.pekerja {
            self.tx.send(None).unwrap(); // hantar signal berhenti
        }
        for h in self.pekerja {
            h.join().unwrap();
        }
    }
}

fn main() {
    let pool = ThreadPool::baru(3);

    for i in 0..8 {
        pool.laksana(move || {
            println!("Tugas {} sedang dijalankan oleh thread: {:?}",
                i, thread::current().id());
            thread::sleep(std::time::Duration::from_millis(100));
        });
    }

    pool.tutup();
    println!("Semua tugas selesai!");
}
```

---

## Rayon — Data Parallelism Mudah

```toml
# Cargo.toml
[dependencies]
rayon = "1.8"
```

```rust
use rayon::prelude::*;
use std::time::Instant;

fn main() {
    let data: Vec<u64> = (1..=10_000_000).collect();

    // Sequential
    let mula = Instant::now();
    let jumlah_seq: u64 = data.iter().sum();
    println!("Sequential: {}ms = {}", mula.elapsed().as_millis(), jumlah_seq);

    // Parallel dengan Rayon — hanya tambah 'par_'!
    let mula = Instant::now();
    let jumlah_par: u64 = data.par_iter().sum();
    println!("Parallel:   {}ms = {}", mula.elapsed().as_millis(), jumlah_par);

    // par_iter — parallel iterator
    let nombor: Vec<u64> = (1..=1_000_000u64).collect();

    // Filter parallel
    let genap: Vec<u64> = nombor.par_iter()
        .filter(|&&x| x % 2 == 0)
        .copied()
        .collect();
    println!("Genap count: {}", genap.len());

    // Map parallel
    let kuasa: Vec<u64> = nombor.par_iter()
        .map(|&x| x * x)
        .collect();
    println!("Kuasa dua pertama: {:?}", &kuasa[..5]);

    // par_chunks — proses dalam kumpulan
    let mula = Instant::now();
    let hasil: Vec<u64> = data.par_chunks(1000)
        .map(|chunk| chunk.iter().sum::<u64>())
        .collect();
    let jumlah: u64 = hasil.iter().sum();
    println!("Par chunks: {}ms = {}", mula.elapsed().as_millis(), jumlah);
}
```

---

## Rayon — Parallel Sort & Find

```rust
use rayon::prelude::*;

fn main() {
    let mut data: Vec<i32> = (0..1_000_000).rev().collect();

    // Parallel sort
    let mula = std::time::Instant::now();
    data.par_sort();
    println!("Par sort: {}ms", mula.elapsed().as_millis());

    // par_sort_unstable — lebih laju tapi tidak stable
    let mut data2: Vec<i32> = (0..1_000_000).rev().collect();
    data2.par_sort_unstable();

    // any/all parallel
    let ada_negatif = data.par_iter().any(|&x| x < 0);
    let semua_positif = data.par_iter().all(|&x| x >= 0);
    println!("Ada negatif: {}", ada_negatif);     // false
    println!("Semua >= 0: {}", semua_positif);   // true

    // find_first parallel
    let pertama_besar = data.par_iter().find_first(|&&x| x > 999_990);
    println!("Pertama > 999990: {:?}", pertama_besar);

    // Reduce parallel
    let jumlah = data.par_iter().copied().reduce(|| 0, |a, b| a + b);
    println!("Jumlah: {}", jumlah);
}
```

---

# BAB 10: Mini Project — Parallel File Processor 📁

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::collections::HashMap;
use std::time::Instant;

// ─── Data Types ───────────────────────────────────────────────

#[derive(Debug, Clone)]
struct DokumenSimulasi {
    id:      u32,
    tajuk:   String,
    kandungan: String,
}

#[derive(Debug, Default, Clone)]
struct StatsDokumen {
    jumlah_patah_kata: usize,
    jumlah_aksara:     usize,
    jumlah_baris:      usize,
    perkataan_biasa:   HashMap<String, usize>,
}

#[derive(Debug)]
struct HasilProses {
    id:    u32,
    tajuk: String,
    stats: StatsDokumen,
    masa_ms: u128,
}

// ─── Proses Dokumen ───────────────────────────────────────────

fn proses_dokumen(dok: &DokumenSimulasi) -> HasilProses {
    let mula = Instant::now();
    let mut stats = StatsDokumen::default();

    stats.jumlah_aksara = dok.kandungan.len();
    stats.jumlah_baris  = dok.kandungan.lines().count();

    // Kira perkataan
    for perkataan in dok.kandungan.split_whitespace() {
        let bersih: String = perkataan.chars()
            .filter(|c| c.is_alphabetic())
            .map(|c| c.to_lowercase().next().unwrap())
            .collect();

        if !bersih.is_empty() {
            stats.jumlah_patah_kata += 1;
            *stats.perkataan_biasa.entry(bersih).or_insert(0) += 1;
        }
    }

    // Simulasi pemprosesan (kira-kira 1-5ms per dokumen)
    let simulasi: u64 = (dok.id % 5 + 1) as u64;
    thread::sleep(std::time::Duration::from_millis(simulasi));

    HasilProses {
        id:    dok.id,
        tajuk: dok.tajuk.clone(),
        stats,
        masa_ms: mula.elapsed().as_millis(),
    }
}

// ─── Parallel Processor ───────────────────────────────────────

struct PemProsesParallel {
    n_thread: usize,
}

impl PemProsesParallel {
    fn baru(n_thread: usize) -> Self {
        PemProsesParallel { n_thread }
    }

    fn proses(&self, dokumen: Vec<DokumenSimulasi>) -> Vec<HasilProses> {
        use std::sync::mpsc;

        let (tx_tugas, rx_tugas) = mpsc::channel::<DokumenSimulasi>();
        let rx_tugas = Arc::new(Mutex::new(rx_tugas));
        let (tx_hasil, rx_hasil) = mpsc::channel::<HasilProses>();

        // Spawn worker threads
        let mut handles = vec![];
        for id_thread in 0..self.n_thread {
            let rx = Arc::clone(&rx_tugas);
            let tx = tx_hasil.clone();

            handles.push(thread::Builder::new()
                .name(format!("pekerja-{}", id_thread))
                .spawn(move || {
                    let nama = thread::current().name().unwrap().to_string();
                    let mut kiraan = 0;

                    loop {
                        let dok = {
                            let guard = rx.lock().unwrap();
                            guard.recv()
                        };

                        match dok {
                            Ok(d) => {
                                println!("  {} memproses: {}", nama, d.tajuk);
                                let hasil = proses_dokumen(&d);
                                kiraan += 1;
                                tx.send(hasil).unwrap();
                            }
                            Err(_) => {
                                println!("  {} selesai ({} dok)", nama, kiraan);
                                break;
                            }
                        }
                    }
                })
                .unwrap()
            );
        }

        drop(tx_hasil); // Drop original tx

        // Hantar semua tugas
        let jumlah = dokumen.len();
        for dok in dokumen {
            tx_tugas.send(dok).unwrap();
        }
        drop(tx_tugas); // Signal berhenti ke semua worker

        // Tunggu semua selesai
        for h in handles { h.join().unwrap(); }

        // Kumpul hasil
        let mut hasil: Vec<HasilProses> = rx_hasil.iter().collect();
        hasil.sort_by_key(|h| h.id);
        hasil
    }
}

// ─── Laporan ──────────────────────────────────────────────────

fn cetak_laporan(hasil: &[HasilProses]) {
    println!("\n{'═'*55}");
    println!("{:^55}", "LAPORAN PEMPROSESAN DOKUMEN");
    println!("{'═'*55}");
    println!("{:<5} {:<20} {:>8} {:>8} {:>8}",
        "ID", "Tajuk", "Kata", "Aksara", "ms");
    println!("{'─'*55}");

    let mut jumlah_kata   = 0usize;
    let mut jumlah_aksara = 0usize;
    let mut jumlah_ms     = 0u128;
    let mut semua_perkataan: HashMap<String, usize> = HashMap::new();

    for h in hasil {
        println!("{:<5} {:<20} {:>8} {:>8} {:>8}",
            h.id,
            &h.tajuk[..h.tajuk.len().min(20)],
            h.stats.jumlah_patah_kata,
            h.stats.jumlah_aksara,
            h.masa_ms);

        jumlah_kata   += h.stats.jumlah_patah_kata;
        jumlah_aksara += h.stats.jumlah_aksara;
        jumlah_ms     += h.masa_ms;

        for (k, &v) in &h.stats.perkataan_biasa {
            *semua_perkataan.entry(k.clone()).or_insert(0) += v;
        }
    }

    println!("{'─'*55}");
    println!("{:<5} {:<20} {:>8} {:>8} {:>8}",
        "JUM", "", jumlah_kata, jumlah_aksara, jumlah_ms);

    // Top 10 perkataan
    let mut sorted: Vec<(&String, &usize)> = semua_perkataan.iter().collect();
    sorted.sort_by(|a, b| b.1.cmp(a.1));

    println!("\nTop 10 Perkataan Paling Biasa:");
    for (kata, kali) in sorted.iter().take(10) {
        println!("  {:<15} {}x", kata, kali);
    }
    println!("{'═'*55}");
}

// ─── Main ──────────────────────────────────────────────────────

fn main() {
    // Jana dokumen simulasi
    let dokumen: Vec<DokumenSimulasi> = vec![
        DokumenSimulasi {
            id: 1,
            tajuk: "Laporan Tahunan KADA".into(),
            kandungan: "KADA adalah agensi kerajaan yang bertanggungjawab \
                membangunkan sektor pertanian padi di kawasan Kemubu. \
                Program pembangunan pertanian telah berjaya meningkatkan \
                hasil padi petani. Petani-petani tempatan mendapat manfaat \
                daripada program bantuan benih dan baja yang disediakan.".into(),
        },
        DokumenSimulasi {
            id: 2,
            tajuk: "Panduan Penanaman Padi".into(),
            kandungan: "Penanaman padi memerlukan persediaan tanah yang rapi. \
                Petani perlu memastikan sistem pengairan yang baik. \
                Benih padi berkualiti tinggi perlu dipilih untuk hasil yang \
                maksimum. Pembajaan perlu dilakukan mengikut jadual yang \
                ditetapkan oleh pakar pertanian KADA.".into(),
        },
        DokumenSimulasi {
            id: 3,
            tajuk: "Pekeliling ICT 2024".into(),
            kandungan: "Semua kakitangan dikehendaki menggunakan sistem \
                KADA Mobile App untuk merekodkan kehadiran harian. \
                Sistem GPS digunakan untuk pengesahan lokasi. \
                Gambar selfie diperlukan untuk pengesahan identiti pekerja. \
                ICT Unit bertanggungjawab memberikan sokongan teknikal.".into(),
        },
        DokumenSimulasi {
            id: 4,
            tajuk: "Kajian Alam Sekitar".into(),
            kandungan: "Kawasan pertanian Kemubu mempunyai sumber air yang \
                mencukupi dari Sungai Kelantan. Alam sekitar kawasan ini \
                perlu dijaga untuk kesinambungan aktiviti pertanian. \
                Kajian mendapati kualiti air sungai masih berada pada \
                tahap yang selamat untuk pertanian.".into(),
        },
        DokumenSimulasi {
            id: 5,
            tajuk: "Statistik Pengeluaran 2024".into(),
            kandungan: "Pengeluaran padi musim pertama mencapai 5.2 tan \
                setiap hektar. Jumlah keluasan kawasan tanaman ialah \
                dua puluh ribu hektar. Hasil keseluruhan melebihi sasaran \
                yang ditetapkan dalam rancangan pembangunan pertanian.".into(),
        },
        DokumenSimulasi {
            id: 6,
            tajuk: "Program Latihan Petani".into(),
            kandungan: "Program latihan pertanian dianjurkan untuk petani \
                baru dan berpengalaman. Latihan merangkumi teknik penanaman \
                moden dan penggunaan jentera pertanian. Petani yang mengikuti \
                latihan menunjukkan peningkatan hasil padi yang ketara.".into(),
        },
        DokumenSimulasi {
            id: 7,
            tajuk: "Laporan Kewangan Q3".into(),
            kandungan: "Peruntukan kewangan untuk program bantuan petani \
                telah digunakan sepenuhnya dalam tempoh yang ditetapkan. \
                Subsidi baja dan benih telah disalurkan kepada petani \
                yang berdaftar. Akaun kewangan disediakan mengikut \
                prosedur kewangan kerajaan Malaysia.".into(),
        },
        DokumenSimulasi {
            id: 8,
            tajuk: "Rancangan Strategik 2025".into(),
            kandungan: "KADA merancang untuk meningkatkan hasil pengeluaran \
                padi melalui penggunaan teknologi pertanian pintar. \
                Program digitalisasi pertanian akan dilaksanakan secara \
                berperingkat. Kerjasama dengan universiti tempatan dalam \
                penyelidikan pertanian akan dipertingkatkan pada tahun 2025.".into(),
        },
    ];

    println!("🔄 Memproses {} dokumen dengan {} thread...\n",
        dokumen.len(), 3);

    // Sequential timing
    let mula_seq = Instant::now();
    let _hasil_seq: Vec<HasilProses> = dokumen.iter()
        .map(proses_dokumen)
        .collect();
    let masa_seq = mula_seq.elapsed().as_millis();

    // Parallel timing
    let processor = PemProsesParallel::baru(3);
    let mula_par  = Instant::now();
    let hasil_par = processor.proses(dokumen);
    let masa_par  = mula_par.elapsed().as_millis();

    cetak_laporan(&hasil_par);

    println!("\n{'─'*40}");
    println!("Perbandingan Prestasi:");
    println!("  Sequential: {}ms", masa_seq);
    println!("  Parallel:   {}ms ({} thread)", masa_par, 3);
    if masa_seq > 0 {
        println!("  Speedup:    {:.2}x", masa_seq as f64 / masa_par as f64);
    }
}
```

---

# 📋 Rujukan Pantas — Multithreading Cheat Sheet

## Asas

```rust
// Spawn thread
let h = thread::spawn(|| { .. });
let h = thread::spawn(move || { .. }); // move ownership ke thread

// Tunggu selesai
h.join().unwrap();          // panic jika thread panic
h.join().expect("msg");     // sama, mesej custom

// Info thread
thread::current().id()      // ID thread
thread::current().name()    // Option<&str>
thread::sleep(Duration)     // tidurkan thread
thread::yield_now()         // bagi thread lain peluang
thread::available_parallelism() // bilangan CPU
```

## Shared State

```rust
// Immutable share
Arc::new(data)
Arc::clone(&arc)

// Mutable share (single writer)
Arc::new(Mutex::new(data))
mutex.lock().unwrap()       // block sehingga dapat lock
mutex.try_lock()            // Result, tidak block

// Many readers, one writer
Arc::new(RwLock::new(data))
rwlock.read().unwrap()      // shared read
rwlock.write().unwrap()     // exclusive write

// Lock-free counter
Arc::new(AtomicUsize::new(0))
atomic.fetch_add(1, Ordering::SeqCst)
atomic.load(Ordering::SeqCst)
```

## Channels

```rust
// Unbounded channel
let (tx, rx) = mpsc::channel::<T>();
tx.send(val).unwrap();
rx.recv().unwrap();         // block
rx.try_recv()               // Result, tidak block
for val in rx { .. }        // drain semua

// Bounded channel (backpressure)
let (tx, rx) = mpsc::sync_channel::<T>(n);
// tx.send() block jika buffer penuh!

// Multi-producer
let tx2 = tx.clone();
```

## Synchronization Primitives

```rust
Barrier::new(n)             // tunggu n thread
barrier.wait()              // .is_leader() untuk satu thread buat sesuatu

Condvar::new()              // conditional wait
condvar.wait(guard)         // tunggu isyarat
condvar.notify_one()        // bangunkan satu thread
condvar.notify_all()        // bangunkan semua thread
```

## Rayon (Parallel Iterator)

```rust
// Guna rayon = "1" dalam Cargo.toml
use rayon::prelude::*;

data.par_iter()             // parallel borrow
data.par_iter_mut()         // parallel mutable
data.into_par_iter()        // parallel consume
data.par_chunks(n)          // parallel chunks

.par_iter().sum()           // parallel sum
.par_iter().map(f).collect()
.par_sort()                 // parallel sort
.par_sort_unstable()        // lebih laju, tidak stable
```

## Panduan Pilih

```
Shared immutable data:      Arc<T>
Shared mutable, simple op:  Arc<AtomicT>
Shared mutable, complex:    Arc<Mutex<T>>
Read-heavy shared mutable:  Arc<RwLock<T>>
Message passing:            mpsc::channel
Data parallelism:           Rayon par_iter
CPU-bound parallel:         Rayon
I/O-bound concurrent:       Tokio async (lihat async.md)
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **MapReduce** — bahagi data, proses parallel (map), kumpul hasil (reduce)
2. **Producer-Consumer Queue** — thread pool dengan priority queue menggunakan `BinaryHeap`
3. **Parallel Merge Sort** — rekursif bahagi, sort parallel, merge
4. **Web Crawler** — banyak thread fetch URL, shared `HashSet` untuk visited URLs
5. **Matrix Multiplication** — parallel row-by-row menggunakan Rayon

---

*Multithreading in Rust — dari `thread::spawn` hingga Rayon parallel iterators. Selamat belajar!* 🦀
