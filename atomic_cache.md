# ⚛️ Atomic & Cache dalam Rust — Panduan Lengkap

> Dari counter thread-safe hingga cache-friendly data structures.
> Faham Atomic operations, Memory Ordering, dan bagaimana CPU cache
> mempengaruhi performance Rust code.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Kenapa Atomic & Cache Penting?

```
ATOMIC:
  Masalah: Mutex terlalu berat untuk operasi mudah (counter, flag)
  Penyelesaian: Atomic operations — thread-safe TANPA lock

  Mutex:   lock → baca/tulis → unlock  (overhead besar!)
  Atomic:  satu CPU instruction → thread-safe!

CACHE:
  Masalah: RAM lambat (100ns), L1 cache laju (1ns)
  Penyelesaian: Reka struktur data yang "cache-friendly"

  Cache miss:  CPU tunggu data dari RAM  → 100× lebih lambat
  Cache hit:   CPU ambil dari L1 cache   → cepat!
```

---

## Peta Pembelajaran

```
Bahagian A — Atomic Types
  A1 → Kenapa Atomic?
  A2 → AtomicBool, AtomicUsize, AtomicI32...
  A3 → Memory Ordering — Orderings Explained
  A4 → AtomicPtr — Atomic Pointer
  A5 → Lock-free Patterns

Bahagian B — Cache Concepts
  B1 → CPU Cache Asas
  B2 → Cache Line & False Sharing
  B3 → Cache-Friendly Data Structures
  B4 → Padding & Alignment
  B5 → Mini Project: High-Performance Counter
```

---

# BAHAGIAN A: Atomic Types ⚛️

## A1: Kenapa Atomic?

```rust
use std::sync::{Arc, Mutex};
use std::thread;

// ─── Masalah tanpa synchronization ───────────────────────────

static mut KIRAAN: u64 = 0; // TIDAK selamat untuk multi-thread!

fn demo_masalah() {
    // Data race — undefined behavior!
    let handles: Vec<_> = (0..10)
        .map(|_| thread::spawn(|| {
            for _ in 0..1000 {
                unsafe { KIRAAN += 1; } // DATA RACE!
            }
        }))
        .collect();

    for h in handles { h.join().unwrap(); }
    unsafe { println!("Tidak selamat: {}", KIRAAN); }
    // Mungkin kurang dari 10000! (lost updates)
}

// ─── Penyelesaian 1: Mutex ────────────────────────────────────
fn demo_mutex() {
    let kiraan = Arc::new(Mutex::new(0u64));
    let mut handles = vec![];

    for _ in 0..10 {
        let k = Arc::clone(&kiraan);
        handles.push(thread::spawn(move || {
            for _ in 0..1000 {
                *k.lock().unwrap() += 1; // lock → ubah → unlock
            }
        }));
    }

    for h in handles { h.join().unwrap(); }
    println!("Mutex: {}", *kiraan.lock().unwrap()); // 10000 ✔
}

// ─── Penyelesaian 2: Atomic — lebih LAJU! ────────────────────
use std::sync::atomic::{AtomicU64, Ordering};

fn demo_atomic() {
    let kiraan = Arc::new(AtomicU64::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let k = Arc::clone(&kiraan);
        handles.push(thread::spawn(move || {
            for _ in 0..1000 {
                k.fetch_add(1, Ordering::Relaxed); // atomic! tiada lock
            }
        }));
    }

    for h in handles { h.join().unwrap(); }
    println!("Atomic: {}", kiraan.load(Ordering::Relaxed)); // 10000 ✔
}
```

---

## A2: Semua Atomic Types

```rust
use std::sync::atomic::{
    AtomicBool,   // bool
    AtomicI8, AtomicI16, AtomicI32, AtomicI64, AtomicIsize,
    AtomicU8, AtomicU16, AtomicU32, AtomicU64, AtomicUsize,
    AtomicPtr,    // *mut T
    Ordering,
};

fn main() {
    // ─── AtomicBool ───────────────────────────────────────────
    let aktif = AtomicBool::new(false);

    // load — baca nilai
    println!("Aktif: {}", aktif.load(Ordering::Acquire));

    // store — tulis nilai
    aktif.store(true, Ordering::Release);

    // swap — tukar dan return nilai lama
    let lama = aktif.swap(false, Ordering::AcqRel);
    println!("Nilai lama: {}", lama); // true

    // compare_exchange — CAS (Compare And Swap)
    let result = aktif.compare_exchange(
        false,            // jangkaan nilai semasa
        true,             // nilai baru
        Ordering::AcqRel, // success ordering
        Ordering::Acquire // failure ordering
    );
    println!("{:?}", result); // Ok(false) — berjaya, nilai lama false

    // ─── AtomicUsize / AtomicU64 ──────────────────────────────
    let kiraan = AtomicUsize::new(0);

    // fetch_add — tambah dan return nilai LAMA
    let sebelum = kiraan.fetch_add(10, Ordering::SeqCst);
    println!("Sebelum: {}", sebelum); // 0
    println!("Sekarang: {}", kiraan.load(Ordering::SeqCst)); // 10

    // fetch_sub — tolak dan return nilai LAMA
    let sebelum2 = kiraan.fetch_sub(3, Ordering::SeqCst);
    println!("Sebelum: {}", sebelum2); // 10
    println!("Sekarang: {}", kiraan.load(Ordering::SeqCst)); // 7

    // fetch_max / fetch_min
    kiraan.fetch_max(100, Ordering::Relaxed);
    println!("Max: {}", kiraan.load(Ordering::Relaxed)); // 100

    kiraan.fetch_min(50, Ordering::Relaxed);
    println!("Min: {}", kiraan.load(Ordering::Relaxed)); // 50

    // Bitwise operations
    let flags = AtomicU32::new(0b0000_1111);
    flags.fetch_or(0b1111_0000, Ordering::Relaxed);  // OR
    println!("OR: {:b}", flags.load(Ordering::Relaxed)); // 11111111

    flags.fetch_and(0b0101_0101, Ordering::Relaxed); // AND
    println!("AND: {:b}", flags.load(Ordering::Relaxed)); // 01010101

    flags.fetch_xor(0b1111_1111, Ordering::Relaxed); // XOR
    println!("XOR: {:b}", flags.load(Ordering::Relaxed)); // 10101010
}
```

---

## A3: Memory Ordering — Yang Paling Mengelirukan 🧠

```
Memory Ordering mengawal BILA perubahan memori KELIHATAN kepada thread lain.

Tanpa ordering: CPU dan compiler BOLEH reorder instructions!
  (untuk performance — instruction reordering adalah normal)

Dengan ordering: kita tetapkan constraints pada reordering.

LIMA ORDERING (dari paling lemah ke paling kuat):

  Relaxed     → Hanya atomicity dijamin, tiada ordering.
                Paling laju, untuk counter yang tidak depend on other ops.

  Acquire     → Baca. Semua reads/writes SELEPAS ini tidak boleh
                naik SEBELUM operation ini.

  Release     → Tulis. Semua reads/writes SEBELUM ini tidak boleh
                turun SELEPAS operation ini.

  AcqRel      → Acquire + Release serentak. Untuk read-modify-write.

  SeqCst      → Sequential Consistency. Paling kuat. Semua thread
                nampak perubahan dalam ORDER yang sama.
                Paling selamat tapi paling lambat.
```

```rust
use std::sync::atomic::{AtomicBool, AtomicI32, Ordering};
use std::sync::Arc;
use std::thread;

// ─── Contoh Acquire/Release ───────────────────────────────────
// Pattern: Producer/Consumer dengan flag

fn demo_acquire_release() {
    let data = Arc::new(AtomicI32::new(0));
    let sedia = Arc::new(AtomicBool::new(false));

    let data2 = Arc::clone(&data);
    let sedia2 = Arc::clone(&sedia);

    // Producer thread
    let producer = thread::spawn(move || {
        data2.store(42, Ordering::Relaxed);      // tulis data
        sedia2.store(true, Ordering::Release);   // RELEASE: pastikan data
                                                  // nampak sebelum flag
    });

    // Consumer thread
    let consumer = thread::spawn(move || {
        // Spin sampai sedia
        while !sedia.load(Ordering::Acquire) {} // ACQUIRE: pastikan data
                                                 // nampak selepas flag
        let nilai = data.load(Ordering::Relaxed);
        println!("Data: {}", nilai); // Dijamin 42!
    });

    producer.join().unwrap();
    consumer.join().unwrap();
}

// ─── Kenapa Relaxed untuk counter? ───────────────────────────
fn demo_relaxed_counter() {
    let total = Arc::new(AtomicU64::new(0));

    // Counter tidak depend on ordering dengan data lain
    // Hanya perlu atomicity (tiada lost updates)
    // Relaxed adalah cukup dan paling laju
    let handles: Vec<_> = (0..4)
        .map(|_| {
            let t = Arc::clone(&total);
            thread::spawn(move || {
                for _ in 0..100_000 {
                    t.fetch_add(1, Ordering::Relaxed); // cukup!
                }
            })
        })
        .collect();

    for h in handles { h.join().unwrap(); }
    println!("Total: {}", total.load(Ordering::Relaxed)); // 400000
}

// Guna AtomicU64 dari std::sync::atomic
use std::sync::atomic::AtomicU64;

fn demo_relaxed_counter() {} // defined above
```

---

## Panduan Ringkas Memory Ordering

```
SOAL: "Apa ordering yang aku perlukan?"

  Baca nilai untuk cek status (bendera, flag)?
    → Acquire

  Tulis nilai untuk isyarat thread lain?
    → Release

  Read-modify-write (fetch_add, swap, CAS)?
    → AcqRel

  Counter yang tidak related dengan data lain?
    → Relaxed

  Nak jamin semua thread nampak order yang sama?
    → SeqCst

  Tidak pasti?
    → SeqCst (selamat, tapi lambat)

Hierarchy:
  Relaxed < Acquire/Release < AcqRel < SeqCst
  Lebih kuat = lebih selamat = lebih lambat
```

---

## 🧠 Brain Teaser A1

Kenapa kod ini MUNGKIN print "Data: 0" walaupun producer sudah simpan 42?

```rust
use std::sync::atomic::{AtomicBool, AtomicI32, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    let data  = Arc::new(AtomicI32::new(0));
    let sedia = Arc::new(AtomicBool::new(false));

    let (d2, s2) = (Arc::clone(&data), Arc::clone(&sedia));
    thread::spawn(move || {
        d2.store(42, Ordering::Relaxed);
        s2.store(true, Ordering::Relaxed); // ← masalah!
    });

    while !sedia.load(Ordering::Relaxed) {}
    println!("Data: {}", data.load(Ordering::Relaxed));
    // Mungkin print 0!
}
```

<details>
<summary>👀 Jawapan</summary>

**Masalah:** Kedua-dua `store` menggunakan `Relaxed` ordering.

`Relaxed` hanya menjamin **atomicity** (operasi tidak boleh terbahagi), tapi **tidak menjamin ordering** antara operasi yang berbeza.

CPU/compiler bebas untuk reorder:
```
# Boleh jadi producer buat dalam order ini:
s2.store(true)  // ← disimpan DULU!
d2.store(42)    // ← baru simpan data

# Dan consumer nampak:
sedia = true    ← nampak
data  = 0       ← belum nampak! (masih 0 dalam cache lain)
```

**Penyelesaian:**
```rust
// Producer:
d2.store(42, Ordering::Relaxed);       // data boleh Relaxed
s2.store(true, Ordering::Release);     // flag MESTI Release

// Consumer:
while !sedia.load(Ordering::Acquire) {} // flag MESTI Acquire
println!("{}", data.load(Ordering::Relaxed));
```

`Release` + `Acquire` membentuk **synchronization pair** yang menjamin semua writes sebelum Release kelihatan selepas Acquire yang berjaya.
</details>

---

## A4: Compare-And-Swap (CAS) — Asas Lock-free

```rust
use std::sync::atomic::{AtomicU64, Ordering};

fn main() {
    let nilai = AtomicU64::new(10);

    // compare_exchange(current, new, success_ord, failure_ord)
    // → Ok(old) jika current == actual value (berjaya ditukar)
    // → Err(actual) jika current != actual value (gagal)

    // Cuba tukar dari 10 ke 20
    match nilai.compare_exchange(10, 20, Ordering::AcqRel, Ordering::Acquire) {
        Ok(v)  => println!("CAS berjaya! Nilai lama: {}", v), // v = 10
        Err(v) => println!("CAS gagal! Nilai sebenar: {}", v),
    }
    println!("Nilai sekarang: {}", nilai.load(Ordering::Relaxed)); // 20

    // Cuba lagi (akan gagal kerana nilai dah 20, bukan 10)
    match nilai.compare_exchange(10, 30, Ordering::AcqRel, Ordering::Acquire) {
        Ok(v)  => println!("CAS berjaya: {}", v),
        Err(v) => println!("CAS gagal! Nilai sebenar: {}", v), // v = 20
    }

    // compare_exchange_weak — boleh gagal spuriously (false negative)
    // Lebih laju pada beberapa platform (ARM)
    // Guna dalam loop:
    let mut semasa = nilai.load(Ordering::Relaxed);
    loop {
        let baru = semasa * 2;
        match nilai.compare_exchange_weak(
            semasa, baru,
            Ordering::AcqRel,
            Ordering::Relaxed
        ) {
            Ok(_)  => { println!("Kali dua: {}", baru); break; }
            Err(v) => { semasa = v; } // nilai berubah, cuba lagi
        }
    }
}
```

---

## A5: Lock-free Patterns

```rust
use std::sync::atomic::{AtomicUsize, AtomicBool, Ordering};
use std::sync::Arc;
use std::thread;

// ─── Pattern 1: Lock-free Flag (Once) ────────────────────────

struct SekaliSahaja {
    dilakukan: AtomicBool,
}

impl SekaliSahaja {
    const fn baru() -> Self {
        SekaliSahaja { dilakukan: AtomicBool::new(false) }
    }

    fn laksana<F: FnOnce()>(&self, f: F) {
        // compare_exchange — hanya satu thread akan berjaya
        if self.dilakukan
            .compare_exchange(false, true, Ordering::AcqRel, Ordering::Acquire)
            .is_ok()
        {
            f();
        }
    }
}

static INISIALISASI: SekaliSahaja = SekaliSahaja::baru();

// ─── Pattern 2: Lock-free Counter dengan Reset ────────────────

struct KiraSemula {
    nilai:   AtomicUsize,
    had:     usize,
}

impl KiraSemula {
    fn baru(had: usize) -> Self {
        KiraSemula { nilai: AtomicUsize::new(0), had }
    }

    /// Return true jika telah sampai had dan reset
    fn tambah(&self) -> bool {
        let mut semasa = self.nilai.load(Ordering::Relaxed);
        loop {
            let baru = semasa + 1;
            if baru >= self.had {
                match self.nilai.compare_exchange_weak(
                    semasa, 0,
                    Ordering::AcqRel,
                    Ordering::Relaxed
                ) {
                    Ok(_)  => return true,
                    Err(v) => { semasa = v; continue; }
                }
            } else {
                match self.nilai.compare_exchange_weak(
                    semasa, baru,
                    Ordering::Relaxed,
                    Ordering::Relaxed
                ) {
                    Ok(_)  => return false,
                    Err(v) => { semasa = v; continue; }
                }
            }
        }
    }
}

// ─── Pattern 3: Spin Lock ─────────────────────────────────────

struct SpinLock {
    dikunci: AtomicBool,
}

impl SpinLock {
    const fn baru() -> Self {
        SpinLock { dikunci: AtomicBool::new(false) }
    }

    fn kunci(&self) {
        while self.dikunci
            .compare_exchange_weak(
                false, true,
                Ordering::Acquire,
                Ordering::Relaxed
            )
            .is_err()
        {
            // Spin sambil tunggu — hint kepada CPU untuk kurangkan power
            std::hint::spin_loop();
        }
    }

    fn buka(&self) {
        self.dikunci.store(false, Ordering::Release);
    }
}

fn main() {
    // Demo SekaliSahaja
    let handles: Vec<_> = (0..5)
        .map(|i| thread::spawn(move || {
            INISIALISASI.laksana(|| {
                println!("Thread {} inisialisasi! (hanya sekali)", i);
            });
        }))
        .collect();
    for h in handles { h.join().unwrap(); }

    // Demo SpinLock
    let lock  = Arc::new(SpinLock::baru());
    let kiraan = Arc::new(AtomicUsize::new(0));

    let handles: Vec<_> = (0..4)
        .map(|_| {
            let (l, k) = (Arc::clone(&lock), Arc::clone(&kiraan));
            thread::spawn(move || {
                for _ in 0..1000 {
                    l.kunci();
                    let v = k.load(Ordering::Relaxed);
                    k.store(v + 1, Ordering::Relaxed);
                    l.buka();
                }
            })
        })
        .collect();

    for h in handles { h.join().unwrap(); }
    println!("Kiraan: {}", kiraan.load(Ordering::Relaxed)); // 4000
}
```

---

# BAHAGIAN B: Cache Concepts 🏎️

## B1: CPU Cache Asas

```
Hierarki memori (dari paling laju ke paling lambat):

  Register   │ < 1ns    │ beberapa dozen bytes
  L1 Cache   │ ~1-4ns   │ 32-64 KB per core
  L2 Cache   │ ~10ns    │ 256KB-1MB per core
  L3 Cache   │ ~30-50ns │ 4-64 MB shared
  RAM        │ ~100ns   │ GBs
  SSD        │ ~100µs   │ TBs
  HDD        │ ~10ms    │ TBs

Cache Line = unit terkecil yang dipindah antara cache & RAM
           = biasanya 64 BYTES

Bila CPU akses memori:
  1. Check L1 → ada? (CACHE HIT) → laju!
  2. Check L2 → ada? → agak laju
  3. Check L3 → ada? → lebih lambat
  4. RAM → tiada dalam cache (CACHE MISS) → lambat!

Consecutive access = cache-friendly (data dalam cache line yang sama)
Random access      = cache-unfriendly (banyak cache misses)
```

---

## B2: Cache Line & False Sharing 😱

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use std::thread;
use std::time::Instant;

// ─── FALSE SHARING — masalah tersembunyi ──────────────────────

// Dua counter ini dalam memori bersebelahan (false sharing!)
struct KiraFalseShar {
    a: AtomicU64, // bytes 0-7
    b: AtomicU64, // bytes 8-15
    // Kedua-dua dalam CACHE LINE YANG SAMA (64 bytes)!
}

// Bila thread 1 tulis 'a' dan thread 2 tulis 'b':
// Walaupun ia variable BERBEZA, cache line yang SAMA!
// CPU perlu "negotiate" cache line ownership → LAMBAT!

// ─── Penyelesaian: Padding ────────────────────────────────────

// Cara 1: Manual padding
#[repr(C)]
struct KiraSelamat {
    a:      AtomicU64,
    _pad_a: [u8; 56], // pad hingga 64 bytes (cache line size)
    b:      AtomicU64,
    _pad_b: [u8; 56],
}

// Cara 2: Guna #[repr(align)] 
#[repr(align(64))] // align ke cache line boundary
struct CacheLine<T> {
    nilai: T,
}

struct KiraAligned {
    a: CacheLine<AtomicU64>,
    b: CacheLine<AtomicU64>,
}

// ─── Benchmark: False Sharing vs Padded ──────────────────────

fn benchmark_false_sharing() {
    let iterasi = 10_000_000u64;

    // Versi dengan false sharing
    let kira = Arc::new(KiraFalseShar {
        a: AtomicU64::new(0),
        b: AtomicU64::new(0),
    });

    let kira2 = Arc::clone(&kira);
    let mula = Instant::now();

    let h1 = thread::spawn(move || {
        for _ in 0..iterasi { kira.a.fetch_add(1, Ordering::Relaxed); }
    });
    let h2 = thread::spawn(move || {
        for _ in 0..iterasi { kira2.b.fetch_add(1, Ordering::Relaxed); }
    });

    h1.join().unwrap();
    h2.join().unwrap();
    println!("False sharing: {}ms", mula.elapsed().as_millis());
}

fn demo_cache_line_size() {
    println!("Cache line size (estimated): 64 bytes");
    println!("AtomicU64 size: {} bytes", std::mem::size_of::<AtomicU64>());
    println!("CacheLine<AtomicU64> align: {} bytes", 
             std::mem::align_of::<CacheLine<AtomicU64>>());
}
```

---

## B3: Cache-Friendly Data Structures 📐

```rust
// ─── Array of Structs (AoS) vs Struct of Arrays (SoA) ────────

// AoS — biasa, tapi cache-unfriendly untuk sebahagian access
#[derive(Debug)]
struct PekerjaAoS {
    id:    u32,
    gaji:  f64,
    aktif: bool,
}

// SoA — cache-friendly untuk access sebahagian data
struct PekerjaSoA {
    id:    Vec<u32>,
    gaji:  Vec<f64>,
    aktif: Vec<bool>,
    // Bilangan pekerja mesti sama!
}

impl PekerjaSoA {
    fn baru() -> Self {
        PekerjaSoA { id: vec![], gaji: vec![], aktif: vec![] }
    }

    fn tambah(&mut self, id: u32, gaji: f64, aktif: bool) {
        self.id.push(id);
        self.gaji.push(gaji);
        self.aktif.push(aktif);
    }

    // Cache-friendly: hanya load 'gaji' array ke cache
    fn jumlah_gaji_aktif(&self) -> f64 {
        self.gaji.iter()
            .zip(self.aktif.iter())
            .filter(|(_, &aktif)| aktif)
            .map(|(&gaji, _)| gaji)
            .sum()
    }
}

fn demo_aos_vs_soa() {
    // AoS — untuk cari satu pekerja, load semua fields
    let pekerja_aos = vec![
        PekerjaAoS { id: 1, gaji: 5000.0, aktif: true },
        PekerjaAoS { id: 2, gaji: 4500.0, aktif: false },
        PekerjaAoS { id: 3, gaji: 6000.0, aktif: true },
    ];

    // Kira jumlah gaji aktif (AoS — setiap struct load semua fields)
    let jumlah_aos: f64 = pekerja_aos.iter()
        .filter(|p| p.aktif)
        .map(|p| p.gaji)
        .sum();
    // Problem: Kita cuma nak 'aktif' dan 'gaji', tapi load 'id' juga!

    // SoA — lebih cache-friendly untuk bulk operations
    let mut pekerja_soa = PekerjaSoA::baru();
    pekerja_soa.tambah(1, 5000.0, true);
    pekerja_soa.tambah(2, 4500.0, false);
    pekerja_soa.tambah(3, 6000.0, true);

    // Cache-friendly: hanya load 'gaji' dan 'aktif' arrays
    let jumlah_soa = pekerja_soa.jumlah_gaji_aktif();

    println!("AoS jumlah: {}", jumlah_aos); // 11000
    println!("SoA jumlah: {}", jumlah_soa); // 11000
}
```

---

## B4: Alignment & Padding

```rust
use std::mem;

// Alignment — CPU suka data yang aligned pada boundary yang sesuai
// u32 mahu pada 4-byte boundary
// u64 mahu pada 8-byte boundary
// f64 mahu pada 8-byte boundary

#[derive(Debug)]
struct Celupar {
    a: u8,    // 1 byte
    // compiler insert 3 bytes padding di sini!
    b: u32,   // 4 bytes — mahu align 4
    c: u8,    // 1 byte
    // compiler insert 7 bytes padding di sini!
    d: u64,   // 8 bytes — mahu align 8
}
// Total: 1 + 3 + 4 + 1 + 7 + 8 = 24 bytes!

#[derive(Debug)]
struct Teratur {
    d: u64,   // 8 bytes
    b: u32,   // 4 bytes
    a: u8,    // 1 byte
    c: u8,    // 1 byte
    // 2 bytes padding di akhir untuk align struct
}
// Total: 8 + 4 + 1 + 1 + 2 = 16 bytes! (lebih kecil!)

fn demo_alignment() {
    println!("Celupar:  {} bytes", mem::size_of::<Celupar>());  // 24
    println!("Teratur:  {} bytes", mem::size_of::<Teratur>());  // 16

    println!("align u8:  {}", mem::align_of::<u8>());  // 1
    println!("align u32: {}", mem::align_of::<u32>()); // 4
    println!("align u64: {}", mem::align_of::<u64>()); // 8
    println!("align f64: {}", mem::align_of::<f64>()); // 8
}

// #[repr(C)] — layout seperti C (tiada reordering)
#[repr(C)]
struct LayoutC {
    a: u8,
    b: u32,
    c: u8,
}
// Sama seperti C — tidak dioptimumkan oleh compiler

// #[repr(packed)] — tiada padding (mungkin unaligned!)
#[repr(packed)]
struct Packed {
    a: u8,
    b: u32,
    c: u8,
}
// 6 bytes, tapi akses ke 'b' mungkin lambat atau UB pada beberapa CPU!

// Explicit alignment
#[repr(align(64))] // align ke cache line
struct CacheAligned {
    data: [u8; 64],
}
```

---

## B5: Mini Project — High-Performance Counter 🏎️

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use std::thread;
use std::time::{Duration, Instant};

// ─── Versi 1: Counter mudah (baseline) ───────────────────────

struct CounterMudah {
    nilai: AtomicU64,
}

impl CounterMudah {
    fn baru() -> Self { CounterMudah { nilai: AtomicU64::new(0) } }
    fn tambah(&self) { self.nilai.fetch_add(1, Ordering::Relaxed); }
    fn baca(&self)   -> u64 { self.nilai.load(Ordering::Relaxed) }
}

// ─── Versi 2: Sharded Counter (kurangkan contention) ─────────

// Setiap core/thread ada "shard" sendiri
// Tambah ke shard tempatan → tiada contention!
// Baca → jumlah semua shards

const BILANGAN_SHARD: usize = 8;

#[repr(align(64))] // Elak false sharing!
struct Shard {
    nilai: AtomicU64,
}

struct CounterShard {
    shards: Vec<Shard>,
}

impl CounterShard {
    fn baru() -> Self {
        CounterShard {
            shards: (0..BILANGAN_SHARD)
                .map(|_| Shard { nilai: AtomicU64::new(0) })
                .collect(),
        }
    }

    fn tambah(&self, thread_id: usize) {
        let shard_idx = thread_id % BILANGAN_SHARD;
        self.shards[shard_idx].nilai.fetch_add(1, Ordering::Relaxed);
    }

    fn baca(&self) -> u64 {
        self.shards.iter()
            .map(|s| s.nilai.load(Ordering::Relaxed))
            .sum()
    }
}

// ─── Versi 3: Padded Counter (elak false sharing sepenuhnya) ──

struct CounterPadded {
    shards: Vec<ShardPadded>,
}

#[repr(C)]
struct ShardPadded {
    nilai: AtomicU64,
    _pad:  [u8; 56], // total = 64 bytes (satu cache line)
}

impl ShardPadded {
    fn baru() -> Self {
        ShardPadded { nilai: AtomicU64::new(0), _pad: [0; 56] }
    }
}

impl CounterPadded {
    fn baru(n_shard: usize) -> Self {
        CounterPadded {
            shards: (0..n_shard).map(|_| ShardPadded::baru()).collect()
        }
    }

    fn tambah(&self, thread_id: usize) {
        let idx = thread_id % self.shards.len();
        self.shards[idx].nilai.fetch_add(1, Ordering::Relaxed);
    }

    fn baca(&self) -> u64 {
        self.shards.iter()
            .map(|s| s.nilai.load(Ordering::SeqCst))
            .sum()
    }
}

// ─── Benchmark ────────────────────────────────────────────────

fn benchmark<F: Fn(usize) + Send + Sync + 'static>(
    nama: &str,
    fungsi: Arc<F>,
    n_thread: usize,
    iterasi: u64,
) -> Duration {
    let handles: Vec<_> = (0..n_thread)
        .map(|id| {
            let f = Arc::clone(&fungsi);
            thread::spawn(move || {
                for _ in 0..iterasi {
                    f(id);
                }
            })
        })
        .collect();

    let mula = Instant::now();
    for h in handles { h.join().unwrap(); }
    let masa = mula.elapsed();

    println!("{:<20} {:>8}ms  ({} threads × {} ops)",
        nama, masa.as_millis(), n_thread, iterasi);

    masa
}

fn main() {
    let n_thread = 4;
    let iterasi = 1_000_000u64;

    println!("Benchmark Counter ({} threads, {} iterasi each)\n",
             n_thread, iterasi);
    println!("{:-<60}", "");

    // Versi 1: Counter mudah
    let counter1 = Arc::new(CounterMudah::baru());
    let c1 = Arc::clone(&counter1);
    benchmark("CounterMudah", Arc::new(move |_| c1.tambah()), n_thread, iterasi);
    println!("  Nilai: {}", counter1.baca());

    // Versi 2: Sharded
    let counter2 = Arc::new(CounterShard::baru());
    let c2 = Arc::clone(&counter2);
    benchmark("CounterShard", Arc::new(move |id| c2.tambah(id)), n_thread, iterasi);
    println!("  Nilai: {}", counter2.baca());

    // Versi 3: Padded
    let counter3 = Arc::new(CounterPadded::baru(n_thread));
    let c3 = Arc::clone(&counter3);
    benchmark("CounterPadded", Arc::new(move |id| c3.tambah(id)), n_thread, iterasi);
    println!("  Nilai: {}", counter3.baca());

    println!("{:-<60}", "");
    println!("\nSaiz struct:");
    println!("  AtomicU64:           {} bytes", std::mem::size_of::<AtomicU64>());
    println!("  Shard (no pad):      {} bytes", std::mem::size_of::<Shard>());
    println!("  ShardPadded:         {} bytes", std::mem::size_of::<ShardPadded>());
    println!("  align ShardPadded:   {} bytes", std::mem::align_of::<ShardPadded>());

    // ─── Demo Atomic Operations ───────────────────────────────

    println!("\n=== Atomic Operations Demo ===\n");

    let nilai = AtomicU64::new(100);

    println!("Asal:           {}", nilai.load(Ordering::Relaxed));

    println!("fetch_add(10):  {} → {}",
        nilai.fetch_add(10, Ordering::Relaxed),
        nilai.load(Ordering::Relaxed));

    println!("fetch_sub(5):   {} → {}",
        nilai.fetch_sub(5, Ordering::Relaxed),
        nilai.load(Ordering::Relaxed));

    println!("fetch_max(200): {} → {}",
        nilai.fetch_max(200, Ordering::Relaxed),
        nilai.load(Ordering::Relaxed));

    println!("fetch_min(150): {} → {}",
        nilai.fetch_min(150, Ordering::Relaxed),
        nilai.load(Ordering::Relaxed));

    // CAS pattern
    let semasa = nilai.load(Ordering::Relaxed);
    match nilai.compare_exchange(semasa, semasa * 2, Ordering::AcqRel, Ordering::Acquire) {
        Ok(lama)  => println!("CAS berjaya: {} → {}", lama, nilai.load(Ordering::Relaxed)),
        Err(sebenar) => println!("CAS gagal, nilai: {}", sebenar),
    }

    println!("\n=== Cache-Friendly vs Unfriendly ===\n");

    let n = 1_000_000usize;
    let data: Vec<i64> = (0..n as i64).collect();

    // Sequential access — cache friendly
    let mula = Instant::now();
    let jumlah: i64 = data.iter().sum();
    println!("Sequential sum:  {}ms ({})",
        mula.elapsed().as_micros(), jumlah);

    // Random access — cache unfriendly
    let mut indices: Vec<usize> = (0..n).collect();
    // Fisher-Yates shuffle
    for i in (1..n).rev() {
        let j = i / 2; // simplified, not truly random
        indices.swap(i, j);
    }

    let mula = Instant::now();
    let jumlah2: i64 = indices.iter().map(|&i| data[i]).sum();
    println!("Random access:   {}ms ({})",
        mula.elapsed().as_micros(), jumlah2);
}
```

---

## 🧠 Brain Teaser B

Mengapa versi B lebih lambat dari versi A walaupun kedua-dua buat operasi yang sama?

```rust
use std::sync::{Arc, Mutex};
use std::thread;

// Versi A
fn versi_a() {
    let counter = Arc::new([
        AtomicU64::new(0), // index 0
        AtomicU64::new(0), // index 1
        AtomicU64::new(0), // index 2
        AtomicU64::new(0), // index 3
    ]);

    let handles: Vec<_> = (0..4).map(|i| {
        let c = Arc::clone(&counter);
        thread::spawn(move || {
            for _ in 0..1_000_000 {
                c[i].fetch_add(1, Ordering::Relaxed); // thread i guna index i
            }
        })
    }).collect();

    for h in handles { h.join().unwrap(); }
}

// Versi B — sama logic, sedikit berbeza
fn versi_b() {
    #[repr(C)]
    struct Counters {
        a: AtomicU64,
        b: AtomicU64,
        c: AtomicU64,
        d: AtomicU64,
    }

    let counter = Arc::new(Counters {
        a: AtomicU64::new(0),
        b: AtomicU64::new(0),
        c: AtomicU64::new(0),
        d: AtomicU64::new(0),
    });

    let (c1, c2, c3, c4) = (
        Arc::clone(&counter), Arc::clone(&counter),
        Arc::clone(&counter), Arc::clone(&counter)
    );

    let handles = vec![
        thread::spawn(move || { for _ in 0..1_000_000 { c1.a.fetch_add(1, Ordering::Relaxed); } }),
        thread::spawn(move || { for _ in 0..1_000_000 { c2.b.fetch_add(1, Ordering::Relaxed); } }),
        thread::spawn(move || { for _ in 0..1_000_000 { c3.c.fetch_add(1, Ordering::Relaxed); } }),
        thread::spawn(move || { for _ in 0..1_000_000 { c4.d.fetch_add(1, Ordering::Relaxed); } }),
    ];

    for h in handles { h.join().unwrap(); }
}
```

<details>
<summary>👀 Jawapan</summary>

**False Sharing!**

Dalam Versi B, `a`, `b`, `c`, `d` adalah dalam `struct Counters` dengan `#[repr(C)]`. Mereka berada dalam memori bersebelahan:

```
Memori: [a: 8 bytes][b: 8 bytes][c: 8 bytes][d: 8 bytes]
         <────────── satu cache line (64 bytes) ──────────>
```

Walaupun setiap thread hanya touch field sendiri, **semua dalam cache line yang sama**! Apabila thread 1 menulis `a`, cache line tersebut "dirty" dalam CPU thread 1. CPU thread 2 perlu "invalidate" cache line dan ambil semula dari shared cache untuk tulis `b`. Ini berulang-ulang → CACHE CONTENTION!

**Versi A** menggunakan array `[AtomicU64; 4]` — sama masalah! Array juga bersebelahan dalam memori.

**Penyelesaian:**
```rust
#[repr(align(64))]  // Setiap counter pada cache line sendiri!
struct PaddedCounter(AtomicU64);

struct Counters {
    a: PaddedCounter,  // cache line 1
    b: PaddedCounter,  // cache line 2
    c: PaddedCounter,  // cache line 3
    d: PaddedCounter,  // cache line 4
}
```

Atau guna padding manual:
```rust
#[repr(C)]
struct Counters {
    a: AtomicU64,
    _pad_a: [u8; 56],  // 8 + 56 = 64 bytes (satu cache line)
    b: AtomicU64,
    _pad_b: [u8; 56],
    // ...
}
```
</details>

---

# 📋 Rujukan Pantas — Atomic & Cache Cheat Sheet

## Atomic Types

```rust
AtomicBool     → bool
AtomicI8/16/32/64/Isize  → signed integers
AtomicU8/16/32/64/Usize  → unsigned integers
AtomicPtr<T>   → *mut T

// Buat
let a = AtomicU64::new(0);
static A: AtomicU64 = AtomicU64::new(0);
```

## Atomic Operations

```rust
a.load(ord)                           // baca nilai
a.store(val, ord)                     // tulis nilai
a.swap(val, ord)                      // tukar, return lama
a.compare_exchange(cur, new, s, f)    // CAS — return Ok(lama)/Err(sebenar)
a.compare_exchange_weak(...)          // CAS weak (boleh fail spuriously, laju)

// Integer operations (return nilai LAMA)
a.fetch_add(n, ord)
a.fetch_sub(n, ord)
a.fetch_max(n, ord)
a.fetch_min(n, ord)
a.fetch_and(n, ord)  // bitwise
a.fetch_or(n, ord)
a.fetch_xor(n, ord)
```

## Memory Ordering

```rust
Ordering::Relaxed  // hanya atomicity, tiada ordering
Ordering::Acquire  // guna untuk load/read (synchronize dengan Release)
Ordering::Release  // guna untuk store/write (synchronize dengan Acquire)
Ordering::AcqRel   // guna untuk read-modify-write (fetch_add, CAS)
Ordering::SeqCst   // paling kuat, semua thread nampak order yang sama

// Panduan cepat:
// load  → Acquire (atau Relaxed kalau independent)
// store → Release (atau Relaxed kalau independent)
// CAS   → AcqRel success, Acquire failure
// counter → Relaxed
// tidak pasti → SeqCst
```

## Cache Concepts

```
Cache line = 64 bytes (biasa)
L1 hit     = ~1-4 ns
L1 miss    = ~100 ns (ke RAM)

FALSE SHARING:
  Berlaku bila dua thread tulis ke field berbeza
  dalam struct yang sama cache line.
  FIX: padding atau #[repr(align(64))]

CACHE-FRIENDLY:
  ✔ Sequential access (arrays, Vec)
  ✔ Small structs
  ✔ SoA untuk bulk operations
  ✘ Linked lists (random memory access)
  ✘ Large structs dengan random field access
```

## Code Patterns

```rust
// Satu kali inisialisasi
static INISIALISASI: std::sync::Once = std::sync::Once::new();
INISIALISASI.call_once(|| { /* setup */ });

// Atau OnceLock (Rust 1.70+)
static DATA: std::sync::OnceLock<String> = std::sync::OnceLock::new();
DATA.get_or_init(|| "setup".to_string());

// Sharded counter (kurangkan contention)
struct ShardedCounter {
    shards: Vec<ShardedPadded>,
}

// Padded struct
#[repr(align(64))]
struct Padded<T>(T);
```

---

## 🏆 Cabaran Akhir

Cuba implement salah satu:

1. **Lock-free Queue** — SPSC (Single Producer Single Consumer) queue menggunakan AtomicUsize untuk head/tail
2. **RCU (Read-Copy-Update)** — optimistic read, atomic swap untuk update
3. **Seqlock** — readers tiada lock, writer increment sequence counter
4. **Cache-Oblivious Matrix Multiply** — susun data untuk minimumkan cache misses
5. **Thread-local Counter** — counter per-thread yang digabung semasa baca (zero contention write!)

---

*Atomic operations dan cache awareness adalah senjata untuk kod concurrent yang benar-benar laju.*
*Faham hardware, tulis kod yang berfungsi bersama CPU, bukan menentangnya.* 🦀
