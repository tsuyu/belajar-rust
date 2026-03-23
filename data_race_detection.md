# 🔍 Strategi Detect Data Race dalam Rust — Panduan Lengkap

> "Rust menjanjikan tiada data race — tapi bagaimana?"
> Faham garanti compiler, alat pengesanan, pattern selamat,
> dan cara debug bila sesuatu tidak kena.

---

## Apa itu Data Race?

```
Data Race berlaku bila:
  1. Dua atau lebih thread SERENTAK akses memori yang SAMA
  2. Sekurang-kurangnya SATU akses adalah TULIS
  3. Tiada SYNCHRONIZATION

Kesan:
  - Output yang tidak konsisten
  - Program crash tanpa sebab yang jelas
  - Bug yang hanya muncul pada timing tertentu
  - Sangat sukar untuk direproduce dan debug

Contoh dalam C (undefined behavior!):
  int kiraan = 0;
  // Thread 1: kiraan++
  // Thread 2: kiraan++
  // Hasil mungkin 1, bukan 2!
  // Sebab: read-modify-write bukan atomic
```

---

## Rust vs Bahasa Lain

```
C/C++:  Programmmer bertanggungjawab sepenuhnya
        Compiler tidak bagi amaran
        ThreadSanitizer perlu diaktif secara manual
        Data race = undefined behavior = apa sahaja boleh berlaku

Java:   Ada thread safety primitif (synchronized, volatile)
        Tapi programmer boleh lupa guna
        Java Memory Model complicated
        Tool: FindBugs, PMD, Checkstyle

Python: GIL (Global Interpreter Lock) elak data race di CPython
        Tapi GIL boleh dilepas (numpy, C extensions)
        Tiada garanti di PyPy

Rust:   COMPILER TANGKAP data race pada compile time!
        Send + Sync traits enforce thread safety
        Ownership + Borrowing = tiada dua mut access serentak
        Masih ada: logical races, deadlock, dan livelock
```

---

## Peta Pembelajaran

```
Bab 1  → Garanti Compiler: Send & Sync
Bab 2  → Borrow Checker — Pencegah Pertama
Bab 3  → Kes yang Compiler Tidak Tangkap
Bab 4  → Atomic Operations — Lock-free
Bab 5  → Mutex & RwLock — Dengan Mutex
Bab 6  → Channel — Komunikasi Selamat
Bab 7  → ThreadSanitizer (TSan)
Bab 8  → Miri — Undefined Behavior Detector
Bab 9  → Loom — Model Checking untuk Concurrency
Bab 10 → Mini Project: Audit Data Race
```

---

# BAB 1: Garanti Compiler — Send & Sync 🛡️

## Auto Traits yang Menjamin Thread Safety

```rust
// Send: selamat untuk HANTAR ke thread lain
// Sync: selamat untuk SHARE (&T) antara thread

// Rust auto-derive Send + Sync berdasarkan field types
// Tiada apa yang perlu tulis!

use std::thread;

// ── Apa yang BOLEH dihantar ke thread ────────────────────────

// i32, f64, bool, String, Vec<T> — semua Send!
let nombor = 42i32;
let teks   = String::from("hello");

thread::spawn(move || {
    println!("{} {}", nombor, teks); // OK! Send
});

// Arc<T> — Send kalau T: Send + Sync
use std::sync::Arc;
let dikongsi = Arc::new(vec![1, 2, 3]);
let klon = Arc::clone(&dikongsi);
thread::spawn(move || {
    println!("{:?}", klon); // OK!
});

// ── Apa yang TIDAK BOLEH ──────────────────────────────────────

// Rc<T> — BUKAN Send (reference counting tidak atomic)
use std::rc::Rc;
let rc = Rc::new(42);
// thread::spawn(move || println!("{}", rc));
// ERROR: `Rc<i32>` cannot be sent between threads safely

// RefCell<T> — BUKAN Sync (runtime borrow check tidak thread-safe)
use std::cell::RefCell;
let rf = Arc::new(RefCell::new(0));
// let klon = Arc::clone(&rf);
// thread::spawn(move || { *klon.borrow_mut() += 1; });
// ERROR: `RefCell<i32>` cannot be shared between threads safely

// *mut T — BUKAN Send (raw pointer)
let ptr: *mut i32 = Box::into_raw(Box::new(42));
// thread::spawn(move || unsafe { *ptr += 1; });
// ERROR: `*mut i32` cannot be sent between threads safely

// ── Implikasi: compile error = pengesahan ────────────────────
println!("Kalau compile → tiada data race (berkaitan ownership)!");
```

---

## Buat Type yang Send + Sync

```rust
use std::sync::{Arc, Mutex};

// Struct biasa — auto Send + Sync kalau semua field Send + Sync
#[derive(Debug)]
struct DataSelamat {
    nilai: i32,        // Send + Sync ✔
    nama:  String,     // Send + Sync ✔
}
// DataSelamat auto: Send + Sync ✔

// Struct dengan Mutex — Send + Sync
struct KiraanSelamat {
    nilai: Mutex<i32>, // Mutex<T>: Send + Sync kalau T: Send
}
// KiraanSelamat auto: Send + Sync ✔

// Bila perlu implement manual (UNSAFE!)
// Hanya bila kita PASTI ia selamat tapi compiler tidak tahu
struct PetunjukKhas(*mut i32);

// SANGAT BAHAYA — pastikan benar-benar selamat sebelum ini!
// unsafe impl Send for PetunjukKhas {}
// unsafe impl Sync for PetunjukKhas {}
```

---

## 🧠 Brain Teaser #1

Kenapa `Arc<Mutex<T>>` selamat tetapi `Rc<RefCell<T>>` tidak?

```rust
// Kedua-dua boleh "share mutable data"
// Tapi hanya satu selamat untuk thread

use std::rc::Rc;
use std::cell::RefCell;
use std::sync::{Arc, Mutex};

// A: Rc<RefCell<T>>
let a = Rc::new(RefCell::new(0));

// B: Arc<Mutex<T>>
let b = Arc::new(Mutex::new(0));

// Mana yang selamat untuk multi-thread?
```

<details>
<summary>👀 Jawapan</summary>

```
Arc<Mutex<T>> = SELAMAT untuk multi-thread:

  Arc = Atomic Reference Counted
    - Increment/decrement reference count menggunakan ATOMIC operations
    - Atomic = thread-safe pada hardware level
    - Selamat untuk clone dan hantar ke thread lain

  Mutex = Mutual Exclusion
    - Hanya SATU thread boleh pegang lock pada satu masa
    - Thread lain BLOCK sehingga lock dilepas
    - OS menjamin exclusivity

  Gabungan: satu memory location, dilindungi oleh lock yang dijamin oleh OS

Rc<RefCell<T>> = TIDAK SELAMAT untuk multi-thread:

  Rc = Reference Counted (bukan atomic!)
    - Increment/decrement count adalah OPERASI BIASA
    - Dua thread boleh increment serentak → race condition pada count!
    - Rc::clone() tidak atomic → data race!

  RefCell = Runtime Borrow Check
    - Simpan "borrow flag" untuk track siapa sedang borrow
    - Flag ini diakses tanpa synchronization!
    - Dua thread boleh borrow serentak → data race pada flag!

Analogi:
  Arc<Mutex<T>> = Bank vault dengan keselamatan bank (ada teller, only one at a time)
  Rc<RefCell<T>> = Drawer bilik tidur (tiada kunci, siapa-siapa boleh buka serentak)

Rust compiler TAHU perbezaan ini:
  Rc<T>: impl !Send, !Sync (explicitly not thread-safe)
  Arc<T>: impl Send, impl Sync (thread-safe!)
  RefCell<T>: impl !Sync (not shareable between threads)
  Mutex<T>: impl Sync (shareable and safe)
```
</details>

---

# BAB 2: Borrow Checker — Pencegah Pertama 🔒

## Kes yang Compiler Tangkap

```rust
use std::thread;

fn main() {
    let mut data = vec![1, 2, 3];

    // ── Data race: tulis semasa read ─────────────────────────
    let rujukan = &data[0];      // immutable borrow
    // data.push(4);             // ERROR: cannot borrow `data` as mutable
                                 // because it is also borrowed as immutable
    println!("{}", rujukan);

    // ── Data race: dua thread akses ──────────────────────────
    let mut kiraan = 0;

    // Ini tidak compile!
    // let t1 = thread::spawn(|| kiraan += 1); // borrow kiraan
    // let t2 = thread::spawn(|| kiraan += 1); // borrow lagi — ERROR!
    // ERROR: closure may outlive the current function,
    //        but it borrows `kiraan`, which is owned by the current function

    // Penyelesaian: move ownership
    // Atau guna Arc<Mutex<T>>

    // ── Move semantics elak sharing ──────────────────────────
    let s1 = String::from("hello");
    let s2 = String::from("world");

    let t1 = thread::spawn(move || {
        println!("Thread 1: {}", s1); // s1 MOVE ke thread 1
    });

    let t2 = thread::spawn(move || {
        println!("Thread 2: {}", s2); // s2 MOVE ke thread 2
    });

    // println!("{}", s1); // ERROR! s1 sudah move
    // println!("{}", s2); // ERROR! s2 sudah move

    t1.join().unwrap();
    t2.join().unwrap();

    // ── Lifetime elak dangling reference ─────────────────────
    let teks;
    {
        let s = String::from("sementara");
        // teks = &s; // ERROR! `s` tidak hidup cukup lama
    }
    // println!("{}", teks); // Kalau boleh, s sudah drop!
}
```

---

# BAB 3: Kes yang Compiler Tidak Tangkap ⚠️

## Logical Race dan Kes Kompleks

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::collections::HashMap;

// ── Kes 1: Check-Then-Act Race ────────────────────────────────
// Compiler puas hati, tapi ada race condition logik!

struct DaftarAkaun {
    baki: Arc<Mutex<HashMap<String, f64>>>,
}

impl DaftarAkaun {
    fn pemindahan_tidak_selamat(&self, dari: &str, ke: &str, jumlah: f64) {
        // MASALAH: Dua operasi berasingan — race boleh berlaku antaranya!
        let baki_dari = {
            let baki = self.baki.lock().unwrap();
            *baki.get(dari).unwrap_or(&0.0)
        }; // ← lock dilepas di sini!

        if baki_dari >= jumlah {
            // ANTARA check dan withdraw, thread lain boleh withdraw dulu!
            let mut baki = self.baki.lock().unwrap();
            let entry = baki.entry(dari.into()).or_insert(0.0);
            *entry -= jumlah; // Mungkin jadi negatif!

            let entry_ke = baki.entry(ke.into()).or_insert(0.0);
            *entry_ke += jumlah;
        }
    }

    fn pemindahan_selamat(&self, dari: &str, ke: &str, jumlah: f64) -> Result<(), String> {
        // Pegang lock sepanjang operasi!
        let mut baki = self.baki.lock().unwrap();

        let baki_dari = *baki.get(dari).unwrap_or(&0.0);
        if baki_dari < jumlah {
            return Err("Baki tidak mencukupi".into());
        }

        *baki.entry(dari.into()).or_insert(0.0) -= jumlah;
        *baki.entry(ke.into()).or_insert(0.0)   += jumlah;
        Ok(())
    }
}

// ── Kes 2: Deadlock ───────────────────────────────────────────
// Compile OK, tapi program hang!

fn deadlock_contoh() {
    let mutex_a = Arc::new(Mutex::new(()));
    let mutex_b = Arc::new(Mutex::new(()));

    let (a1, b1) = (Arc::clone(&mutex_a), Arc::clone(&mutex_b));
    let t1 = thread::spawn(move || {
        let _ga = a1.lock().unwrap();         // pegang A
        thread::sleep(std::time::Duration::from_millis(10));
        let _gb = b1.lock().unwrap();         // tunggu B ← DEADLOCK!
    });

    let (a2, b2) = (Arc::clone(&mutex_a), Arc::clone(&mutex_b));
    let t2 = thread::spawn(move || {
        let _gb = b2.lock().unwrap();         // pegang B
        thread::sleep(std::time::Duration::from_millis(10));
        let _ga = a2.lock().unwrap();         // tunggu A ← DEADLOCK!
    });

    // Ini tidak akan selesai!
    t1.join().unwrap();
    t2.join().unwrap();
}

// ── Kes 3: Livelock ───────────────────────────────────────────
// Dua thread sentiasa "mengalah" kepada satu sama lain
// Program berjalan tapi tiada kemajuan

// ── Kes 4: Starvation ─────────────────────────────────────────
// Satu thread tidak pernah dapat akses kerana thread lain sentiasa
// dapat lock dahulu
```

---

# BAB 4: Atomic Operations — Lock-free 🔬

## Guna Atomic untuk Counter

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicU64, AtomicBool, Ordering};
use std::thread;

fn main() {
    // ── AtomicU64 — counter tanpa Mutex ──────────────────────
    let kiraan = Arc::new(AtomicU64::new(0));

    let mut handles = Vec::new();
    for _ in 0..10 {
        let k = Arc::clone(&kiraan);
        handles.push(thread::spawn(move || {
            for _ in 0..1000 {
                // fetch_add adalah ATOMIC — tidak ada race!
                k.fetch_add(1, Ordering::Relaxed);
            }
        }));
    }

    for h in handles { h.join().unwrap(); }
    println!("Kiraan: {}", kiraan.load(Ordering::Relaxed)); // SELALU 10000

    // Bandingkan dengan versi tidak selamat (C-like):
    // static mut KIRAAN: u64 = 0;
    // unsafe { KIRAAN += 1; } // DATA RACE! Undefined behavior!

    // ── Ordering — Memory Ordering ────────────────────────────
    // Relaxed:  Tiada synchronization, hanya atomicity (paling laju)
    // Acquire:  Operasi selepas tidak boleh gerak sebelum ini
    // Release:  Operasi sebelum tidak boleh gerak selepas ini
    // AcqRel:   Gabungan Acquire + Release
    // SeqCst:   Paling ketat, semua thread nampak sama order (paling lambat)

    // ── AtomicBool — flag tanpa Mutex ─────────────────────────
    let berjalan = Arc::new(AtomicBool::new(true));

    let bendera = Arc::clone(&berjalan);
    let pekerja = thread::spawn(move || {
        while bendera.load(Ordering::Acquire) {
            // Buat kerja...
            thread::sleep(std::time::Duration::from_millis(10));
        }
        println!("Pekerja berhenti!");
    });

    thread::sleep(std::time::Duration::from_millis(50));
    berjalan.store(false, Ordering::Release); // signal berhenti

    pekerja.join().unwrap();

    // ── Compare-and-Swap (CAS) ────────────────────────────────
    let nilai = Arc::new(AtomicU64::new(0));
    let v = Arc::clone(&nilai);

    thread::spawn(move || {
        // Hanya set kepada 100 kalau nilai semasa adalah 0
        match v.compare_exchange(0, 100, Ordering::SeqCst, Ordering::Relaxed) {
            Ok(lama) => println!("CAS berjaya: {} → 100", lama),
            Err(semasa) => println!("CAS gagal: nilai semasa = {}", semasa),
        }
    }).join().unwrap();
}
```

---

## Struktur Lock-free dengan Atomic

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};

// Stack lock-free yang mudah (untuk ilustrasi)
// NOTA: Dalam production, guna crate `crossbeam` yang sudah diuji!

struct KiraanBerfasa {
    dibuat:   AtomicUsize,
    diproses: AtomicUsize,
    gagal:    AtomicUsize,
}

impl KiraanBerfasa {
    fn baru() -> Arc<Self> {
        Arc::new(KiraanBerfasa {
            dibuat:   AtomicUsize::new(0),
            diproses: AtomicUsize::new(0),
            gagal:    AtomicUsize::new(0),
        })
    }

    fn rekod_dibuat(&self)   { self.dibuat.fetch_add(1, Ordering::Relaxed); }
    fn rekod_diproses(&self) { self.diproses.fetch_add(1, Ordering::Relaxed); }
    fn rekod_gagal(&self)    { self.gagal.fetch_add(1, Ordering::Relaxed); }

    fn laporan(&self) {
        println!("Dibuat:   {}", self.dibuat.load(Ordering::Relaxed));
        println!("Diproses: {}", self.diproses.load(Ordering::Relaxed));
        println!("Gagal:    {}", self.gagal.load(Ordering::Relaxed));
    }
}

fn main() {
    let kiraan = KiraanBerfasa::baru();
    let mut handles = Vec::new();

    for i in 0..5 {
        let k = Arc::clone(&kiraan);
        handles.push(std::thread::spawn(move || {
            for j in 0..100 {
                k.rekod_dibuat();
                if (i + j) % 7 == 0 {
                    k.rekod_gagal();
                } else {
                    k.rekod_diproses();
                }
            }
        }));
    }

    for h in handles { h.join().unwrap(); }
    kiraan.laporan();
}
```

---

# BAB 5: Mutex & RwLock — Cara Selamat 🔐

## Pattern Mutex yang Betul

```rust
use std::sync::{Arc, Mutex, RwLock, MutexGuard};
use std::thread;
use std::time::Duration;

// ── Elak Deadlock: Lock dalam urutan yang konsisten ──────────

struct DuaMutex {
    a: Mutex<i32>,
    b: Mutex<i32>,
}

impl DuaMutex {
    fn operasi_selamat(&self) -> i32 {
        // SELALU lock dalam urutan yang sama: A dulu, kemudian B
        let a = self.a.lock().unwrap();
        let b = self.b.lock().unwrap();
        *a + *b
    }
    // Jangan ada fungsi yang lock B dulu kemudian A!
}

// ── Lock scope yang minimum ───────────────────────────────────

struct Cache {
    data: Mutex<std::collections::HashMap<u32, String>>,
}

impl Cache {
    // ❌ BURUK: Lock terlalu lama
    fn proses_dan_simpan_buruk(&self, id: u32) {
        let mut data = self.data.lock().unwrap();
        // Operasi mahal DALAM lock!
        let hasil = std::thread::sleep(Duration::from_millis(100));
        // Ini block semua thread lain selama 100ms!
        data.insert(id, "nilai".into());
    }

    // ✔ BAIK: Lock sesingkat mungkin
    fn proses_dan_simpan_baik(&self, id: u32) {
        // Proses SEBELUM ambil lock
        let hasil = {
            // lakukan pengiraan tanpa lock
            std::thread::sleep(Duration::from_millis(100));
            format!("nilai_{}", id)
        };

        // Lock hanya untuk insert (sangat pantas)
        let mut data = self.data.lock().unwrap();
        data.insert(id, hasil);
        // Lock dilepas terus
    }

    // ✔ Baca tanpa lock kalau boleh (guna RwLock)
    fn baca(&self, id: u32) -> Option<String> {
        self.data.lock().unwrap().get(&id).cloned()
        // Lock dilepas selepas .cloned()
    }
}

// ── RwLock untuk read-heavy workload ──────────────────────────

struct KonfigApp {
    data: RwLock<std::collections::HashMap<String, String>>,
}

impl KonfigApp {
    fn baca(&self, kunci: &str) -> Option<String> {
        // BANYAK thread boleh baca serentak!
        self.data.read().unwrap().get(kunci).cloned()
    }

    fn tulis(&self, kunci: &str, nilai: &str) {
        // Hanya SATU thread boleh tulis
        self.data.write().unwrap().insert(kunci.into(), nilai.into());
    }
}

// ── Poison Mutex ─────────────────────────────────────────────
// Bila thread panic semasa pegang lock → Mutex "poisoned"

fn kendalikan_poison() {
    let m = Arc::new(Mutex::new(0i32));
    let m2 = Arc::clone(&m);

    let _ = thread::spawn(move || {
        let _guard = m2.lock().unwrap();
        panic!("Panic dalam lock!"); // Mutex jadi poisoned!
    }).join();

    // lock() akan return Err pada poisoned mutex
    match m.lock() {
        Ok(v)  => println!("OK: {}", *v),
        Err(e) => {
            println!("Mutex poisoned! Cuba recover...");
            let v = e.into_inner(); // boleh recover dengan into_inner()
            println!("Nilai: {}", *v);
        }
    }
}
```

---

# BAB 6: Channel — Komunikasi Selamat 📡

## Hantar Data, Bukan Share Data

```rust
use std::sync::mpsc;
use std::thread;

// Prinsip: "Do not communicate by sharing memory;
//           instead, share memory by communicating."
//           — Go proverb, tapi berlaku untuk Rust juga

fn main() {
    // ── Fan-out: satu producer, banyak consumer ───────────────
    let (tx, rx) = mpsc::channel::<i32>();
    let rx = std::sync::Arc::new(std::sync::Mutex::new(rx));

    for id in 0..4 {
        let rx = std::sync::Arc::clone(&rx);
        thread::spawn(move || {
            loop {
                let nilai = rx.lock().unwrap().recv();
                match nilai {
                    Ok(v)  => println!("Worker {}: proses {}", id, v),
                    Err(_) => { println!("Worker {} berhenti", id); break; }
                }
            }
        });
    }

    for i in 0..20 {
        tx.send(i).unwrap();
    }
    drop(tx); // signal semua worker berhenti

    thread::sleep(std::time::Duration::from_millis(100));

    // ── Fan-in: banyak producer, satu consumer ────────────────
    let (tx2, rx2) = mpsc::channel::<(usize, String)>();

    for i in 0..3 {
        let tx = tx2.clone();
        thread::spawn(move || {
            let hasil = format!("Hasil dari thread {}", i);
            tx.send((i, hasil)).unwrap();
        });
    }
    drop(tx2);

    let mut semua: Vec<(usize, String)> = rx2.iter().collect();
    semua.sort_by_key(|(i, _)| *i);
    for (i, h) in semua { println!("{}: {}", i, h); }
}
```

---

# BAB 7: ThreadSanitizer (TSan) 🔧

## Detect Data Race pada Runtime

```bash
# TSan adalah alat yang SANGAT berguna untuk detect data race
# yang compiler tidak nampak (logical races, unsafe code, dll)

# Aktifkan TSan semasa build
RUSTFLAGS="-Z sanitizer=thread" cargo +nightly test
RUSTFLAGS="-Z sanitizer=thread" cargo +nightly run

# Atau dalam Cargo.toml untuk nightly:
# [profile.dev]
# debug = true
```

```rust
// Contoh: TSan akan detect ini walaupun compile OK!

use std::sync::Arc;
use std::thread;

fn main() {
    // Kod ini menggunakan unsafe — compiler tidak tangkap!
    let data = Arc::new(std::cell::UnsafeCell::new(0i32));

    let d1 = Arc::clone(&data);
    let d2 = Arc::clone(&data);

    let t1 = thread::spawn(move || {
        unsafe {
            // Tulis tanpa synchronization!
            *d1.get() = 1;
        }
    });

    let t2 = thread::spawn(move || {
        unsafe {
            // Tulis serentak — DATA RACE!
            *d2.get() = 2;
        }
    });

    t1.join().unwrap();
    t2.join().unwrap();

    // TSan Output:
    // WARNING: ThreadSanitizer: data race (pid=12345)
    //   Write of size 4 at 0x... by thread T2:
    //     #0 main::{{closure}} src/main.rs:18
    //   Previous write of size 4 at 0x... by thread T1:
    //     #0 main::{{closure}} src/main.rs:12
}

// Cara guna TSan dalam test:
// #[test]
// fn test_thread_safety() {
//     // Jalankan dengan: RUSTFLAGS="-Z sanitizer=thread" cargo +nightly test
//     // TSan akan detect race semasa test!
// }
```

## TSan dalam CI/CD

```yaml
# .github/workflows/tsan.yml
name: ThreadSanitizer

on: [push, pull_request]

jobs:
  tsan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: dtolnay/rust-toolchain@nightly
      - run: |
          RUSTFLAGS="-Z sanitizer=thread" \
          TSAN_OPTIONS="halt_on_error=1" \
          cargo +nightly test --target x86_64-unknown-linux-gnu
```

---

# BAB 8: Miri — Undefined Behavior Detector 🧪

## Detect UB Termasuk Data Race dalam unsafe

```bash
# Install Miri
rustup component add miri

# Jalankan
cargo miri test
cargo miri run
```

```rust
// Miri detect berbagai masalah:
// - Data race
// - Use after free
// - Buffer overflow
// - Uninitialized memory access
// - Invalid pointer dereference

// Contoh yang Miri akan tangkap:

fn main() {
    let v = vec![1, 2, 3];
    let ptr = v.as_ptr();

    drop(v); // v di-drop, pointer tidak valid!

    unsafe {
        // USE AFTER FREE — Miri akan detect ini!
        println!("{}", *ptr);
    }

    // Miri Output:
    // error: Undefined Behavior: pointer to alloc... is dangling
    //        (was later deallocated) at alloc...+0x0
}

// Guna dalam test:
// cargo miri test -- test_thread_safety

// Konfigurasi Miri dalam .cargo/config.toml:
// [target.x86_64-unknown-linux-gnu]
// runner = "cargo miri run --"
```

---

# BAB 9: Loom — Model Checking 🔬

## Test Semua Kemungkinan Concurrent Execution

```toml
[dev-dependencies]
loom = "0.7"
```

```rust
// Loom adalah alat yang paling canggih untuk menguji kod concurrent
// Ia explore SEMUA kemungkinan thread interleaving!
// Sesuai untuk test struktur data concurrent dan lock-free algorithms

#[cfg(test)]
mod tests {
    use loom::sync::{Arc, Mutex};
    use loom::thread;

    // Guna loom::sync bukan std::sync dalam test!
    #[test]
    fn test_dengan_loom() {
        // loom::model explore semua kemungkinan execution!
        loom::model(|| {
            let kiraan = Arc::new(Mutex::new(0));

            let k1 = Arc::clone(&kiraan);
            let t1 = thread::spawn(move || {
                *k1.lock().unwrap() += 1;
            });

            let k2 = Arc::clone(&kiraan);
            let t2 = thread::spawn(move || {
                *k2.lock().unwrap() += 1;
            });

            t1.join().unwrap();
            t2.join().unwrap();

            assert_eq!(2, *kiraan.lock().unwrap());
            // Loom akan pastikan ini SELALU 2 dalam SEMUA kemungkinan ordering
        });
    }

    // Test struktur data lock-free
    #[test]
    fn test_atomic_counter_loom() {
        use loom::sync::atomic::{AtomicUsize, Ordering};

        loom::model(|| {
            let kiraan = Arc::new(AtomicUsize::new(0));

            let k1 = Arc::clone(&kiraan);
            let t1 = thread::spawn(move || k1.fetch_add(1, Ordering::SeqCst));

            let k2 = Arc::clone(&kiraan);
            let t2 = thread::spawn(move || k2.fetch_add(1, Ordering::SeqCst));

            t1.join().unwrap();
            t2.join().unwrap();

            assert_eq!(2, kiraan.load(Ordering::SeqCst));
        });
    }
}
```

---

# BAB 10: Mini Project — Audit Data Race 🏗️

```rust
// Panduan komprehensif: Audit kod untuk data race

use std::sync::{Arc, Mutex, RwLock};
use std::sync::atomic::{AtomicU64, AtomicBool, Ordering};
use std::thread;
use std::time::{Duration, Instant};

// ─── SENARAI SEMAK AUDIT ────────────────────────────────────────

// ✅ 1. Semua akses thread menggunakan Arc?
// ✅ 2. Shared mutable data dalam Mutex atau RwLock?
// ✅ 3. Simple counters menggunakan Atomic?
// ✅ 4. Channel untuk hantar ownership bukan share?
// ✅ 5. Lock scope diminimumkan?
// ✅ 6. Lock order konsisten (elak deadlock)?
// ✅ 7. Tiada unsafe yang berbahaya?
// ✅ 8. Jalankan dengan TSan atau Loom?

// ─── Contoh Audit: SEBELUM dan SELEPAS ─────────────────────────

// SEBELUM audit (ada masalah):
mod sebelum {
    use std::sync::{Arc, Mutex};
    use std::collections::HashMap;
    use std::thread;

    struct StatistikBuruk {
        // ❌ MASALAH 1: Data terlalu besar dalam satu Mutex
        data: Mutex<HashMap<String, Vec<u64>>>,
    }

    impl StatistikBuruk {
        fn rekod(&self, kunci: &str, nilai: u64) {
            let mut data = self.data.lock().unwrap();
            data.entry(kunci.into()).or_default().push(nilai);
            // ❌ MASALAH 2: Lock semasa sort yang mahal!
            let entry = data.get_mut(kunci).unwrap();
            entry.sort();
        }

        fn purata(&self, kunci: &str) -> Option<f64> {
            let data = self.data.lock().unwrap();
            let v = data.get(kunci)?;
            let jumlah: u64 = v.iter().sum();
            Some(jumlah as f64 / v.len() as f64)
            // ❌ MASALAH 3: Clone dalam lock tidak perlu
        }
    }
}

// SELEPAS audit (diperbaiki):
mod selepas {
    use std::sync::{Arc, RwLock};
    use std::sync::atomic::{AtomicU64, Ordering};
    use std::collections::HashMap;

    // ✅ Pisahkan concerns dengan struct berasingan
    struct KiraanStatistik {
        kiraan: AtomicU64,  // lock-free untuk counter
        jumlah: AtomicU64,  // lock-free untuk sum
        min:    AtomicU64,  // lock-free untuk min
        maks:   AtomicU64,  // lock-free untuk max
    }

    impl KiraanStatistik {
        fn baru() -> Self {
            KiraanStatistik {
                kiraan: AtomicU64::new(0),
                jumlah: AtomicU64::new(0),
                min:    AtomicU64::new(u64::MAX),
                maks:   AtomicU64::new(0),
            }
        }

        fn rekod(&self, nilai: u64) {
            self.kiraan.fetch_add(1, Ordering::Relaxed);
            self.jumlah.fetch_add(nilai, Ordering::Relaxed);

            // Update min dengan CAS loop
            let mut min = self.min.load(Ordering::Relaxed);
            while nilai < min {
                match self.min.compare_exchange(min, nilai, Ordering::SeqCst, Ordering::Relaxed) {
                    Ok(_)  => break,
                    Err(x) => min = x,
                }
            }

            // Update max dengan CAS loop
            let mut maks = self.maks.load(Ordering::Relaxed);
            while nilai > maks {
                match self.maks.compare_exchange(maks, nilai, Ordering::SeqCst, Ordering::Relaxed) {
                    Ok(_)  => break,
                    Err(x) => maks = x,
                }
            }
        }

        fn purata(&self) -> f64 {
            let k = self.kiraan.load(Ordering::Relaxed);
            let j = self.jumlah.load(Ordering::Relaxed);
            if k == 0 { 0.0 } else { j as f64 / k as f64 }
        }

        fn ringkasan(&self) -> (u64, u64, u64, f64) {
            let k = self.kiraan.load(Ordering::Relaxed);
            let j = self.jumlah.load(Ordering::Relaxed);
            let min = if k > 0 { self.min.load(Ordering::Relaxed) } else { 0 };
            let maks = if k > 0 { self.maks.load(Ordering::Relaxed) } else { 0 };
            (k, min, maks, if k > 0 { j as f64 / k as f64 } else { 0.0 })
        }
    }

    // ✅ RwLock untuk data yang jarang berubah
    struct RegistriStatistik {
        registry: RwLock<HashMap<String, Arc<KiraanStatistik>>>,
    }

    impl RegistriStatistik {
        fn baru() -> Arc<Self> {
            Arc::new(RegistriStatistik { registry: RwLock::new(HashMap::new()) })
        }

        fn rekod(&self, kunci: &str, nilai: u64) {
            // Cuba baca dulu (cekap — boleh ada banyak reader)
            {
                let registry = self.registry.read().unwrap();
                if let Some(stat) = registry.get(kunci) {
                    stat.rekod(nilai);
                    return;
                }
            } // read lock dilepas

            // Perlu tulis (hanya bila kunci baru)
            {
                let mut registry = self.registry.write().unwrap();
                // Double-check selepas dapat write lock
                let stat = registry.entry(kunci.into())
                    .or_insert_with(|| Arc::new(KiraanStatistik::baru()));
                stat.rekod(nilai);
            } // write lock dilepas
        }

        fn ringkasan(&self, kunci: &str) -> Option<(u64, u64, u64, f64)> {
            let registry = self.registry.read().unwrap();
            registry.get(kunci).map(|s| s.ringkasan())
        }
    }

    pub fn demo() {
        let registry = RegistriStatistik::baru();
        let mut handles = Vec::new();

        for i in 0..10 {
            let r = Arc::clone(&registry);
            handles.push(std::thread::spawn(move || {
                for j in 0..1000 {
                    r.rekod("latency_ms", (i * 10 + j % 100) as u64);
                    r.rekod("request_size", ((i + 1) * 100 + j) as u64);
                }
            }));
        }

        for h in handles { h.join().unwrap(); }

        if let Some((k, min, maks, purata)) = registry.ringkasan("latency_ms") {
            println!("Latency:");
            println!("  Kiraan:  {}", k);
            println!("  Min:     {}ms", min);
            println!("  Maks:    {}ms", maks);
            println!("  Purata:  {:.2}ms", purata);
        }
    }
}

// ─── Checklist dan Panduan ─────────────────────────────────────

fn papar_checklist() {
    println!("\n{'═'*60}");
    println!("{:^60}", "CHECKLIST AUDIT DATA RACE");
    println!("{'═'*60}");

    let senarai = [
        ("COMPILER CHECKS", vec![
            ("✔", "Compile tanpa warning"),
            ("✔", "Tiada unsafe block yang tidak diperiksa"),
            ("✔", "Guna Send + Sync dengan betul"),
        ]),
        ("SYNCHRONIZATION", vec![
            ("✔", "Arc untuk shared ownership"),
            ("✔", "Mutex/RwLock untuk shared mutable state"),
            ("✔", "Atomic untuk simple counters/flags"),
            ("✔", "Channel untuk hantar ownership"),
        ]),
        ("LOCK HYGIENE", vec![
            ("✔", "Lock scope diminimumkan"),
            ("✔", "Tiada operasi mahal dalam lock"),
            ("✔", "Lock order konsisten (elak deadlock)"),
            ("✔", "Guna RwLock untuk read-heavy workload"),
        ]),
        ("TOOLS", vec![
            ("✔", "cargo clippy -- -W clippy::all"),
            ("✔", "ThreadSanitizer (TSan) pada nightly"),
            ("✔", "Miri untuk unsafe code"),
            ("✔", "Loom untuk concurrent algorithm"),
        ]),
    ];

    for (kategori, items) in &senarai {
        println!("\n  {}:", kategori);
        for (icon, item) in items {
            println!("    {} {}", icon, item);
        }
    }

    println!("\n{'═'*60}");
}

fn main() {
    selepas::demo();
    papar_checklist();
}
```

---

# 📋 Rujukan Pantas — Data Race Detection

## Lapisan Pertahanan

```
LAPISAN 1: Compiler (compile time)
  ✔ Send + Sync traits — type system tangkap race
  ✔ Borrow checker — tiada dua mutable access serentak
  ✔ Lifetime — tiada dangling reference

LAPISAN 2: Synchronization Primitif (runtime)
  ✔ Mutex<T>     — exclusive access
  ✔ RwLock<T>    — multiple readers, one writer
  ✔ Atomic*      — lock-free untuk primitive types
  ✔ Channel      — hantar ownership, bukan share

LAPISAN 3: Testing Tools
  ✔ TSan          — detect race pada runtime
  ✔ Miri          — detect UB + race dalam unsafe
  ✔ Loom          — model checking, semua permutasi
  ✔ cargo clippy  — static analysis

LAPISAN 4: Code Review
  ✔ Semak setiap unsafe block
  ✔ Semak lock order (deadlock prevention)
  ✔ Semak scope lock (lama lock = contention)
  ✔ Semak check-then-act patterns
```

## Alat & Arahan

```bash
# Compile check
cargo build
cargo clippy -- -W clippy::all

# ThreadSanitizer (perlu nightly)
RUSTFLAGS="-Z sanitizer=thread" \
  cargo +nightly test

# Miri (UB detector)
cargo miri test

# Loom (model checking)
# Tambah dalam test: use loom::sync::*;
# Jalankan: LOOM_MAX_PREEMPTIONS=3 cargo test

# Address Sanitizer
RUSTFLAGS="-Z sanitizer=address" \
  cargo +nightly test
```

## Panduan Pilih Synchronization

```
Data yang dibaca sahaja → tiada sync diperlukan (immutable!)
Counter/flag → AtomicU64 / AtomicBool (paling laju)
Complex state, write-heavy → Mutex<T>
Complex state, read-heavy → RwLock<T>
Hantar data antara thread → mpsc::channel / tokio::mpsc
Share besar immutable data → Arc<T>
Share besar mutable data → Arc<Mutex<T>> atau Arc<RwLock<T>>
```

---

## 🏆 Cabaran Akhir

Cuba implement dan audit salah satu:

1. **Lock-free Queue** — implement ring buffer dengan Atomic, test dengan Loom
2. **Read-Copy-Update (RCU)** — pattern yang baca tanpa lock, tulis dengan clone
3. **Connection Pool** — manage pool of connections dengan Mutex, detect deadlock
4. **Rate Limiter** — token bucket yang thread-safe dengan Atomic
5. **Event System** — subscriber pattern yang thread-safe, test dengan TSan

---

*Rust menjamin tiada data race dalam safe code.*
*Tapi logical race, deadlock, dan unsafe code perlu tools tambahan.*
*Lapisan pertahanan: Compiler → Primitif → Testing → Review.* 🦀
