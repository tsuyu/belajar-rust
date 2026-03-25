# 🗺️ Rust Ownership Decision Tree

> Panduan pilih jenis yang betul
> berdasarkan bagaimana data anda digunakan.

---

## Carta Asal (ASCII)

```
Ownership
├── Unique ──────────────── Borrowing
│                               ├── Static ──────────────── T
│                               ├── Local dynamic
│                               │       ├── Cell<T>         (1)
│                               │       └── RefCell<T>      (1)
│                               └── Threaded dynamic
│                                       ├── AtomicT         (2)
│                                       ├── Mutex<T>        (1)
│                                       └── RwLock<T>       (1)
│
├── Locally shared ──────── Mutable?
│                               ├── No  ─────────────────── Rc<T>
│                               └── Yes
│                                       ├── Rc<Cell<T>>
│                                       └── Rc<RefCell<T>>
│
└── Thread-shared ───────── Mutable?
                                ├── No  ─────────────────── Arc<T>
                                └── Yes
                                        ├── Arc<AtomicT>    (2)
                                        ├── Arc<Mutex<T>>   (1)
                                        └── Arc<RwLock<T>>  (1)
```

**Nota:**
- `(1)` — Runtime borrow checking (boleh panic! / deadlock!)
- `(2)` — Lock-free atomic operations

---

## Penjelasan Carta

Carta ini menjawab satu soalan: **"Bagaimana saya nak uruskan data ini?"**

Tiga cabang utama dari `Ownership`:

```
Ownership
  │
  ├── Unique (satu owner)    → Borrowing
  │
  ├── Locally Shared         → Rc<T>
  │   (same thread)
  │
  └── Shared Between Threads → Arc<T>
```

---

## Cabang 1: Unique Ownership → Borrowing

```
Unique Ownership
       │
       └── Borrowing
               │
               ├── Static ──────────────────────────────→ T
               │   (saiz tetap, compile-time known)
               │
               ├── Dynamic (Local, single-thread)
               │       │
               │       ├── Val (value access) ──────────→ Cell<T>
               │       └── Ref (reference access) ──────→ RefCell<T>
               │
               └── Dynamic (Threaded, multi-thread)
                       │
                       ├── AtomicT    (lock-free)
                       ├── Mutex<T>   (exclusive lock)
                       └── RwLock<T>  (read-write lock)
```

---

### T — Static / Stack

**Bila guna:** Data mudah, saiz diketahui semasa compile, tidak perlu share, tidak perlu interior mutability.

**Ciri-ciri:**
- Sepenuhnya di stack — zero heap overhead
- Paling laju, tiada indirection
- Termasuk: `i32`, `f64`, `bool`, `char`, `[T; N]`, `(A, B)`, `&str`, struct semua field Copy

```rust
let x: i32 = 42;
let aktif: bool = true;
let titik: (f64, f64) = (1.0, 2.0);
let nama: &str = "Ali";            // reference ke data static
let arr: [i32; 5] = [1, 2, 3, 4, 5];

// Implement Copy supaya struct berkelakuan seperti T
#[derive(Copy, Clone, Debug)]
struct GPS { lat: f64, lon: f64 }

let lokasi = GPS { lat: 6.05, lon: 102.25 };
let salin = lokasi;       // COPY — lokasi masih valid!
println!("{:?}", lokasi); // OK
```

**Gotcha:**
```rust
// String BUKAN T — ada heap allocation
let s = String::from("hello");
let s2 = s;          // MOVE, bukan copy!
// println!("{}", s); // ERROR: s sudah moved
```

---

### Cell\<T\> — Local Dynamic Val

**Bila guna:** Perlu ubah nilai dalam `&self` (immutable reference), untuk **Copy types** sahaja, single-thread.

**Ciri-ciri:**
- Zero runtime overhead — hanya wrap nilai
- Tiada borrow tracking — tulis guna `set()`, baca guna `get()`
- TIDAK boleh `borrow()` — hanya `get()`/`set()`/`replace()`/`take()`
- Tidak `Sync` — tidak boleh share antara thread

```rust
use std::cell::Cell;

struct Kiraan {
    nilai: Cell<u32>,
}

impl Kiraan {
    fn tambah(&self) {           // &self, bukan &mut self!
        self.nilai.set(self.nilai.get() + 1);
    }
    fn baca(&self) -> u32 {
        self.nilai.get()
    }
}

let k = Kiraan { nilai: Cell::new(0) };
k.tambah();
k.tambah();
println!("{}", k.baca()); // 2 — ubah tanpa mut!
```

**Methods berguna:**
```rust
let c = Cell::new(42i32);
c.set(100);                // tulis nilai baru
let v = c.get();           // baca nilai (Copy)
let lama = c.replace(99);  // tukar, return nilai lama
let ambil = c.take();      // ambil nilai, ganti dengan Default (0)
```

**Gotcha:** `Cell<T>` tidak boleh return reference ke dalaman — hanya Copy keluar. Untuk jenis lebih besar atau perlu reference, guna `RefCell<T>`.

**Use case sebenar:** Counter akses dalam struct, cached flag, lazy compute result kecil.

---

### RefCell\<T\> — Local Dynamic Ref

**Bila guna:** Perlu ubah data dalam `&self`, untuk **mana-mana type** (bukan hanya Copy), single-thread.

**Ciri-ciri:**
- Runtime borrow tracking — simpan `BorrowFlag` dalam struct
- `borrow()` → `Ref<T>` guard (immutable)
- `borrow_mut()` → `RefMut<T>` guard (mutable)
- **PANIC** kalau rules dilanggar semasa runtime
- Tidak `Sync` — tidak boleh share antara thread

```rust
use std::cell::RefCell;

let rf = RefCell::new(vec![1, 2, 3]);

// Borrow immutable
let baca = rf.borrow();
println!("{:?}", *baca);
drop(baca);                  // ← lepaskan SEBELUM borrow_mut!

// Borrow mutable
let mut tulis = rf.borrow_mut();
tulis.push(4);
drop(tulis);

// try_borrow() — selamat tanpa panic
match rf.try_borrow_mut() {
    Ok(mut v) => v.push(5),
    Err(_)    => println!("sedang dipinjam"),
}

println!("{:?}", rf.borrow()); // [1, 2, 3, 4, 5]
```

**Gotcha — PANIC yang biasa:**
```rust
let rf = RefCell::new(0);

// ❌ Ada immutable borrow aktif
let _r1 = rf.borrow();
let _r2 = rf.borrow_mut(); // PANIC!

// ❌ Dua mutable borrow serentak
let _w1 = rf.borrow_mut();
let _w2 = rf.borrow_mut(); // PANIC!
```

**Gotcha — `_` vs nama variable:**
```rust
let rf = RefCell::new(0);

// ❌ SILAP — `_` drop SEGERA, bukan bila scope tamat!
let _ = rf.borrow_mut(); // borrow dilepas terus lepas baris ini

// ✔ BETUL — _guard aktif sehingga scope tamat
let _guard = rf.borrow_mut();
```

**Use case sebenar:** Graph dengan parent pointer, memoization cache, event listener list, lazy initialization.

---

### AtomicT — Threaded Dynamic, Lock-free ⚡

**Bila guna:** Counter, flag, atau nilai primitif dikongsi **antara thread**, perlu performa tinggi tanpa lock.

**Jenis Atomic tersedia:**
```
AtomicBool        → boolean flag
AtomicI8/U8       → 8-bit integer
AtomicI16/U16     → 16-bit integer
AtomicI32/U32     → 32-bit integer
AtomicI64/U64     → 64-bit integer
AtomicIsize/Usize → pointer-sized integer
AtomicPtr<T>      → raw pointer (guna dengan unsafe)
```

```rust
use std::sync::atomic::{AtomicU64, AtomicBool, Ordering};
use std::sync::Arc;

let kiraan = Arc::new(AtomicU64::new(0));

kiraan.fetch_add(1, Ordering::Relaxed);       // increment
kiraan.fetch_sub(1, Ordering::Relaxed);       // decrement
kiraan.fetch_or(0b1010, Ordering::Relaxed);   // bitwise OR
kiraan.load(Ordering::SeqCst);                // baca
kiraan.store(42, Ordering::Release);          // tulis

// Compare-and-swap — set baru HANYA kalau nilai semasa = dijangka
match kiraan.compare_exchange(0, 1, Ordering::SeqCst, Ordering::Relaxed) {
    Ok(lama)    => println!("Berjaya: {} → 1", lama),
    Err(semasa) => println!("Gagal, nilai semasa: {}", semasa),
}

// Flag untuk hentikan thread
let berjalan = Arc::new(AtomicBool::new(true));
let b = Arc::clone(&berjalan);
std::thread::spawn(move || {
    while b.load(Ordering::Acquire) { /* buat kerja */ }
    println!("Thread berhenti!");
});
berjalan.store(false, Ordering::Release);
```

**Memory Ordering — pilih yang betul:**
```
Relaxed  → Hanya atomicity, tiada ordering guarantee.
           Guna untuk: counter bebas, statistik.

Acquire  → Operasi SELEPAS tidak boleh naik sebelum load ini.
           Guna untuk: "baca flag → baca data dilindungi flag".

Release  → Operasi SEBELUM tidak boleh turun selepas store ini.
           Guna untuk: "tulis data → set flag".

AcqRel   → Acquire + Release dalam satu operasi (fetch_add, CAS).

SeqCst   → Paling ketat, semua thread nampak turutan sama.
           Guna bila tidak pasti. Paling lambat (memory fence).
```

**Contoh Acquire/Release yang betul:**
```rust
use std::sync::atomic::{AtomicBool, AtomicI32, Ordering};
use std::sync::Arc;

let data_siap = Arc::new(AtomicBool::new(false));
let data      = Arc::new(AtomicI32::new(0));

// Thread penulis
let (ds, d) = (Arc::clone(&data_siap), Arc::clone(&data));
std::thread::spawn(move || {
    d.store(42, Ordering::Relaxed);    // tulis data
    ds.store(true, Ordering::Release); // isyarat: data siap
});

// Thread pembaca
let (ds, d) = (Arc::clone(&data_siap), Arc::clone(&data));
std::thread::spawn(move || {
    while !ds.load(Ordering::Acquire) {}
    println!("{}", d.load(Ordering::Relaxed)); // pasti 42
});
```

**Gotcha:** Jangan guna `Relaxed` untuk flag yang mengawal akses data lain — guna `Acquire`/`Release`.

**Use case sebenar:** Request counter, active connection count, global ID generator, shutdown flag.

---

### Mutex\<T\> — Threaded Dynamic, Exclusive Lock

**Bila guna:** Data kompleks (bukan primitif) yang perlu **ditulis** dari berbilang thread.

**Ciri-ciri:**
- Hanya SATU thread boleh pegang lock pada satu masa
- Thread lain block sehingga lock dilepas
- Lock dilepas automatik bila `MutexGuard` di-drop (RAII)
- Boleh **poisoned** kalau thread panic semasa pegang lock

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let data = Arc::new(Mutex::new(vec![1, 2, 3]));

let d = Arc::clone(&data);
thread::spawn(move || {
    let mut guard = d.lock().unwrap();
    guard.push(4);
    // lock dilepas bila guard di-drop
});
```

**Elak deadlock — scope lock seminimum mungkin:**
```rust
let m = Arc::new(Mutex::new(0i32));

// ❌ BURUK — lock dipegang semasa kerja lambat
let guard = m.lock().unwrap();
buat_kerja_lambat();  // semua thread lain tunggu!
println!("{}", *guard);

// ✔ BAIK — lock sesingkat mungkin
{
    let mut guard = m.lock().unwrap();
    *guard += 1;
}  // lock dilepas di sini
buat_kerja_lambat();  // kerja lambat di luar lock
```

**Elak deadlock — turutan lock mesti konsisten:**
```rust
// ❌ Thread 1: A→B, Thread 2: B→A → DEADLOCK!

// ✔ Semua thread lock dalam turutan sama: A dulu, B kemudian
let _a = mutex_a.lock().unwrap();
let _b = mutex_b.lock().unwrap();
```

**Poison mutex:**
```rust
let m = Arc::new(Mutex::new(42i32));
match m.lock() {
    Ok(val)  => println!("OK: {}", *val),
    Err(err) => {
        let val = err.into_inner(); // recover data
        println!("Poisoned tapi recovered: {}", *val);
    }
}
```

**Gotcha — `_guard` vs `_`:**
```rust
let m = Mutex::new(0);
let _ = m.lock().unwrap();      // ❌ lock dilepas SEGERA!
let _guard = m.lock().unwrap(); // ✔ lock aktif sampai akhir scope
```

**Use case sebenar:** App state web server, shared HashMap/cache, connection pool, job queue.

---

### RwLock\<T\> — Threaded Dynamic, Read-Write Lock

**Bila guna:** Data yang **lebih kerap dibaca** berbanding ditulis, dikongsi antara thread.

**Ciri-ciri:**
- Boleh ada **BANYAK** `read()` serentak
- Hanya **SATU** `write()` pada satu masa, tiada read semasa write
- Lebih cekap dari `Mutex` untuk read-heavy workload
- Boleh writer starvation jika read tidak habis-habis

```rust
use std::sync::{Arc, RwLock};

let konfig = Arc::new(RwLock::new(std::collections::HashMap::new()));

// Banyak thread boleh baca serentak
let r = konfig.read().unwrap();
println!("{:?}", r.get("kunci"));
drop(r);

// Hanya satu thread boleh tulis
let mut w = konfig.write().unwrap();
w.insert("kunci".to_string(), "nilai".to_string());
```

**Bila Mutex lebih baik:**
```
Guna Mutex bila:
  - Mostly write operations
  - Operasi pendek
  - Tidak pasti → Mutex adalah default selamat

Guna RwLock bila:
  - Reads jauh >> Writes (10x atau lebih)
  - Read ambil masa lama
  - Contoh: konfigurasi app, lookup table, routing table
```

**Use case sebenar:** App config yang jarang berubah, in-memory lookup table, static data di-refresh kadang-kadang.

---

## Cabang 2: Locally Shared → Rc\<T\>

```
Locally Shared (same thread)
       │
       └── Mutable?
               │
               ├── No  ──────────────────────────→ Rc<T>
               │
               └── Yes
                     │
                     ├── Val (Copy types) ────────→ Rc<Cell<T>>
                     └── Ref (any type) ──────────→ Rc<RefCell<T>>
```

---

### Rc\<T\> — Shared Read-only, Single Thread

**Bila guna:** Data yang perlu dimiliki beberapa bahagian kod dalam **thread yang sama**, tanpa perlu ubah.

**Ciri-ciri:**
- Reference counting biasa (bukan atomic) — lebih laju dari `Arc` dalam single-thread
- **BUKAN `Send`** — compile error kalau cuba hantar ke thread lain
- Clone = murah (hanya increment counter, bukan clone data)
- Data di-drop bila **semua** Rc di-drop (kiraan = 0)

```rust
use std::rc::Rc;

let data = Rc::new(vec![1, 2, 3]);
let klon1 = Rc::clone(&data); // clone pointer, bukan data!
let klon2 = Rc::clone(&data);

println!("Owners: {}", Rc::strong_count(&data)); // 3

drop(klon1);
drop(klon2);
println!("Owners: {}", Rc::strong_count(&data)); // 1
```

**Weak\<T\> — pecahkan reference cycle:**
```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

// ⚠️ Tanpa Weak: A→B, B→A → cycle → memory leak!

struct Nod {
    nilai: i32,
    anak:  Vec<Rc<RefCell<Nod>>>,
    ibu:   Option<Weak<RefCell<Nod>>>, // Weak — tiada ownership
}

let ibu = Rc::new(RefCell::new(Nod { nilai: 1, anak: vec![], ibu: None }));
let anak = Rc::new(RefCell::new(Nod {
    nilai: 2, anak: vec![],
    ibu: Some(Rc::downgrade(&ibu)),  // Rc → Weak
}));
ibu.borrow_mut().anak.push(Rc::clone(&anak));

// Akses ibu dari anak
if let Some(ibu_weak) = anak.borrow().ibu.as_ref() {
    if let Some(ibu_rc) = ibu_weak.upgrade() { // Weak → Option<Rc>
        println!("Ibu: {}", ibu_rc.borrow().nilai); // 1
    }
}
// Bila ibu di-drop, weak.upgrade() return None — selamat!
```

**Gotcha:**
```rust
let data = Rc::new(42);
// thread::spawn(move || println!("{}", data)); // COMPILE ERROR!
// Guna Arc<T> untuk multi-thread
```

---

### Rc\<Cell\<T\>\> — Shared Mutable Copy, Single Thread

**Bila guna:** Beberapa bahagian kod perlu **baca dan tulis** nilai Copy yang sama, dalam satu thread.

```rust
use std::rc::Rc;
use std::cell::Cell;

let kiraan = Rc::new(Cell::new(0u32));
let k1 = Rc::clone(&kiraan);
let k2 = Rc::clone(&kiraan);

k1.set(k1.get() + 1);
k2.set(k2.get() + 10);

println!("{}", kiraan.get()); // 11
```

**Use case:** Counter dikongsi beberapa struct, accumulated value, shared flag.

---

### Rc\<RefCell\<T\>\> — Shared Mutable Any, Single Thread

**Bila guna:** Beberapa bahagian kod perlu **baca dan tulis** data kompleks yang sama, dalam satu thread.

```rust
use std::rc::Rc;
use std::cell::RefCell;

let data = Rc::new(RefCell::new(vec![1, 2, 3]));

let d1 = Rc::clone(&data);
let d2 = Rc::clone(&data);

d1.borrow_mut().push(4);
d2.borrow_mut().push(5);

println!("{:?}", data.borrow()); // [1, 2, 3, 4, 5]

// ⚠️ AWAS: boleh ada reference cycle → memory leak!
// Gunakan Weak<T> untuk memutuskan cycle
```

**Use case sebenar:** DOM-like tree, observer/event pattern, graph dengan edge dua hala.

---

## Cabang 3: Shared Between Threads → Arc\<T\>

```
Shared Between Threads (multi-thread)
       │
       └── Mutable?
               │
               ├── No  ──────────────────────────→ Arc<T>
               │
               └── Yes
                     │
                     ├── Atomic ─────────────────→ Arc<AtomicT>
                     ├── Exclusive lock ──────────→ Arc<Mutex<T>>
                     └── Read-write lock ─────────→ Arc<RwLock<T>>
```

---

### Arc\<T\> — Shared Read-only, Multi-Thread

**Bila guna:** Data yang perlu **dibaca** dari berbilang thread tanpa perlu ubah.

**Ciri-ciri:**
- Atomic Reference Counting — counter guna `fetch_add` atomic
- `Send + Sync` — selamat untuk multi-thread
- Clone = murah (~5ns, atomic increment sahaja)
- Data **tidak disalin** — semua thread kongsi memori sama

```rust
use std::sync::Arc;
use std::thread;

let data = Arc::new(vec![1, 2, 3, 4, 5]);

for i in 0..5 {
    let d = Arc::clone(&data);
    thread::spawn(move || {
        println!("Thread {}: {:?}", i, d);
    });
}
// Data kongsi antara 5 thread — tiada copy!
```

**Arc<Weak<T>> — cycle breaking multi-thread:**
```rust
use std::sync::{Arc, Weak};

let kuat = Arc::new(42);
let lemah: Weak<i32> = Arc::downgrade(&kuat);

match lemah.upgrade() {
    Some(val) => println!("Masih hidup: {}", val),
    None      => println!("Sudah di-drop"),
}

drop(kuat);
assert!(lemah.upgrade().is_none()); // kuat tiada, lemah return None
```

**Use case sebenar:** Konfigurasi app dikongsi semua handler, lookup table statik, shared immutable dataset.

---

### Arc\<AtomicT\> — Shared Mutable Atomic, Multi-Thread ⚡

**Bila guna:** Counter, flag, atau nilai primitif yang perlu **ditulis** dari berbilang thread, performa tinggi.

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicU64, Ordering};
use std::thread;

let kiraan = Arc::new(AtomicU64::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let k = Arc::clone(&kiraan);
    handles.push(thread::spawn(move || {
        for _ in 0..1000 {
            k.fetch_add(1, Ordering::Relaxed);
        }
    }));
}

for h in handles { h.join().unwrap(); }
println!("{}", kiraan.load(Ordering::Relaxed)); // 10000 — sentiasa tepat!
```

**Use case sebenar:** Request counter, active connection count, global ID generator, shutdown flag.

---

### Arc\<Mutex\<T\>\> — Shared Mutable Exclusive, Multi-Thread

**Bila guna:** Data kompleks yang perlu **ditulis** dari berbilang thread.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let state = Arc::new(Mutex::new(std::collections::HashMap::new()));

let s1 = Arc::clone(&state);
let t1 = thread::spawn(move || {
    s1.lock().unwrap().insert("a", 1);
});

let s2 = Arc::clone(&state);
let t2 = thread::spawn(move || {
    s2.lock().unwrap().insert("b", 2);
});

t1.join().unwrap();
t2.join().unwrap();

println!("{:?}", state.lock().unwrap()); // {"a": 1, "b": 2}
```

**Use case sebenar:** Shared cache antara thread, connection pool, shared app state web server.

---

### Arc\<RwLock\<T\>\> — Shared Mutable Read-Write, Multi-Thread

**Bila guna:** Data yang **lebih kerap dibaca** berbanding ditulis, dikongsi antara thread.

```rust
use std::sync::{Arc, RwLock};
use std::thread;

let konfig = Arc::new(RwLock::new(String::from("nilai asal")));

// Banyak thread boleh baca serentak
for i in 0..5 {
    let k = Arc::clone(&konfig);
    thread::spawn(move || {
        println!("Thread {} baca: {}", i, k.read().unwrap());
    });
}

// Update (exclusive)
*konfig.write().unwrap() = "nilai baru".into();
```

**Use case sebenar:** App config yang dikemaskini jarang, shared lookup table, API rate limit config.

---

## Ringkasan Carta

| Situasi | Jenis | Thread-safe? | Mutable? | Overhead | Bila Guna |
|---------|-------|:---:|:---:|---------|-----------|
| Stack/static | `T` | ✓ | ✗ | Zero | Data primitif, saiz tetap |
| Interior mut, Copy | `Cell<T>` | ✗ | ✓ | Zero | Counter kecil dalam struct |
| Interior mut, any | `RefCell<T>` | ✗ | ✓ | Runtime check | Graph, tree, cache |
| Shared, single | `Rc<T>` | ✗ | ✗ | Ref count | Shared config single-thread |
| Shared mut Copy, single | `Rc<Cell<T>>` | ✗ | ✓ | Ref count | Shared counter single-thread |
| Shared mut any, single | `Rc<RefCell<T>>` | ✗ | ✓ | Ref + runtime | DOM tree, observer pattern |
| Shared, multi | `Arc<T>` | ✓ | ✗ | Atomic count | Shared immutable data multi-thread |
| Shared counter/flag | `Arc<AtomicT>` | ✓ | ✓ | Lock-free | Hit counter, shutdown flag |
| Shared mut exclusive | `Arc<Mutex<T>>` | ✓ | ✓ | OS lock | Shared HashMap, cache |
| Shared mut read-heavy | `Arc<RwLock<T>>` | ✓ | ✓ | OS lock | App config, lookup table |

---

## Decision Tree (Teks)

```
Saya ada data. Macam mana nak urusnya?
│
├── Hanya SAYA ada (unique ownership)
│     │
│     └── Perlu interior mutability?
│           ├── Tidak → guna T biasa
│           ├── Ya, Copy type, single-thread → Cell<T>
│           ├── Ya, any type, single-thread  → RefCell<T>
│           ├── Ya, primitive, multi-thread  → AtomicT
│           ├── Ya, exclusive, multi-thread  → Mutex<T>
│           └── Ya, read-heavy, multi-thread → RwLock<T>
│
├── KONGSI antara beberapa bahagian (SAME THREAD)
│     │
│     └── Perlu ubah data?
│           ├── Tidak → Rc<T>
│           ├── Ya, Copy type → Rc<Cell<T>>
│           └── Ya, any type  → Rc<RefCell<T>>
│
└── KONGSI antara THREAD BERBEZA
      │
      └── Perlu ubah data?
            ├── Tidak → Arc<T>
            ├── Ya, primitive/bool → Arc<AtomicT>
            ├── Ya, exclusive access → Arc<Mutex<T>>
            └── Ya, read-heavy → Arc<RwLock<T>>
```

---

## Nota Penting

```
(1) Runtime borrow checking:
    Cell, RefCell, Mutex, RwLock
    → Semua ini cek semasa runtime
    → Cell/RefCell boleh PANIC
    → Mutex/RwLock boleh DEADLOCK

(2) Lock-free (atomic):
    AtomicT, Arc<AtomicT>
    → Tiada lock, tiada blocking
    → Guna CPU atomic instructions
    → Hanya untuk primitive types (integer, bool, pointer)
    → Perlu faham Memory Ordering (Relaxed/Acquire/Release/SeqCst)
```

---

## Anti-patterns Biasa

```rust
// ❌ 1. Mutex scope terlalu luas
let guard = m.lock().unwrap();
buat_kerja_lambat();    // semua thread lain tunggu!
// ✔ Lepaskan lock selepas guna, kerja lambat di luar

// ❌ 2. RefCell borrow_mut dua kali tanpa drop
let _b1 = rf.borrow_mut();
let _b2 = rf.borrow_mut(); // PANIC!
// ✔ Pastikan borrow pertama drop sebelum borrow semula

// ❌ 3. Rc dalam multi-thread
let rc = Rc::new(42);
// thread::spawn(move || ...); // COMPILE ERROR
// ✔ Guna Arc<T> untuk multi-thread

// ❌ 4. Deadlock — lock dua mutex secara terbalik
// Thread 1: lock A, tunggu B
// Thread 2: lock B, tunggu A → hang selamanya!
// ✔ Sentiasa lock dalam turutan yang konsisten (A dulu, B kemudian)

// ❌ 5. _ drop guard segera
let _ = m.lock().unwrap();     // lock dilepas TERUS!
// ✔ Guna nama variable untuk kekalkan guard
let _guard = m.lock().unwrap();

// ❌ 6. RwLock untuk write-heavy workload
// Overhead lebih besar dari Mutex tanpa faedah
// ✔ Guna Mutex kalau write kerap atau tidak pasti
```

---

## Panduan Pantas

```
Default pilihan:
  Single-thread, tidak perlu share  →  T (stack variable)
  Single-thread, perlu share        →  Rc<T>
  Multi-thread, tidak perlu share   →  T + move ke thread
  Multi-thread, perlu share         →  Arc<T>

Bila perlu mutability:
  Single + Copy type                →  Cell<T>
  Single + any type                 →  RefCell<T>
  Multi + number/bool               →  Arc<AtomicXxx>
  Multi + complex data              →  Arc<Mutex<T>>
  Multi + read >> write             →  Arc<RwLock<T>>

Jangan guna:
  Rc<T> untuk multi-thread          →  Compile error!
  RefCell<T> untuk multi-thread     →  Compile error!
  Mutex tanpa Arc                   →  Tidak boleh clone
  RwLock bila write-heavy           →  Guna Mutex sahaja
  AtomicT untuk struct/enum         →  Guna Mutex<T>
```

---

## Perbandingan Performance (anggaran relatif)

```
Operasi                         Overhead
──────────────────────────────────────────────────
Stack access (T)                ~0 ns   — paling laju
Cell<T> get/set                 ~0 ns   — sama seperti T
Rc clone/drop                   ~2 ns   — increment i32
Arc clone/drop                  ~5 ns   — atomic increment
RefCell borrow/borrow_mut       ~2 ns   — check flag
AtomicU64 fetch_add Relaxed     ~2 ns   — satu instruksi CPU
AtomicU64 fetch_add SeqCst      ~10 ns  — memory fence
Mutex lock (uncontended)        ~15 ns  — system call ringan
Mutex lock (contended)          ~100 ns+ — OS scheduler terlibat
RwLock read (uncontended)       ~20 ns
RwLock write (uncontended)      ~25 ns
──────────────────────────────────────────────────
Angka anggaran pada x86_64 moden. Benchmark sendiri.
```

---

*Satu gambar yang bernilai seribu baris dokumentasi.* 🦀
