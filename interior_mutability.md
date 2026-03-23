# 🔄 Interior Mutability dalam Rust — Panduan Lengkap

> "Kadang-kala kita perlu ubah data di dalam sesuatu
>  yang secara luarannya adalah immutable."
> Faham Cell, RefCell, Mutex, RwLock, dan bila guna apa.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Masalah Yang Diselesaikan

```
Rust rules:
  1. Boleh ada BANYAK &T (immutable references) ATAU
  2. SATU &mut T (mutable reference) sahaja

Masalah:
  Kadang kita ada &T (immutable) tapi nak ubah data di dalam.

Contoh:
  struct Graf { nod: Vec<Nod>, cache: HashMap<u32, u32> }
  //                           ^^^^^ cache perlu diubah semasa traverse!

  fn traverse(&self) {          // ← &self = immutable!
      // Tapi nak update self.cache???
  }

Penyelesaian = INTERIOR MUTABILITY:
  "Ubah data DI DALAM walaupun ia kelihatan immutable dari luar"
  Periksa peraturan borrowing pada RUNTIME, bukan compile time.
```

---

## Keluarga Interior Mutability

```
                    SINGLE THREAD          MULTI THREAD
                 ┌─────────────────┐    ┌─────────────────┐
  Copy types:    │  Cell<T>        │    │  Atomic*         │
                 │  (no overhead)  │    │  (lock-free!)    │
                 ├─────────────────┤    ├─────────────────┤
  Any type:      │  RefCell<T>     │    │  Mutex<T>        │
  (runtime check)│  (panic if err) │    │  (block if lock) │
                 │                 │    │  RwLock<T>       │
                 │                 │    │  (multiple reads)│
                 ├─────────────────┤    ├─────────────────┤
  One-time init: │  OnceCell<T>    │    │  OnceLock<T>     │
                 │  Lazy<T>        │    │  LazyLock<T>     │
                 └─────────────────┘    └─────────────────┘
```

---

## Peta Pembelajaran

```
Bab 1  → Cell<T> — Copy types, zero overhead
Bab 2  → RefCell<T> — Runtime borrow checking
Bab 3  → RefCell dalam Struct
Bab 4  → Rc<RefCell<T>> — Shared Mutable Data
Bab 5  → Mutex<T> — Thread-safe Mutability
Bab 6  → RwLock<T> — Multiple Readers
Bab 7  → OnceCell & OnceLock
Bab 8  → Lazy & LazyLock
Bab 9  → Pilih Yang Betul
Bab 10 → Mini Project: Cache System
```

---

# BAB 1: Cell\<T\> — Paling Ringan ⚡

## Cell untuk Copy Types

```rust
use std::cell::Cell;

fn main() {
    // Cell<T> — hanya untuk T: Copy
    // Tiada runtime overhead, tiada check
    // TIDAK boleh ambil reference ke dalam — hanya get/set!

    let nilai = Cell::new(42i32);

    // Ubah walaupun tidak mut!
    let r = &nilai;         // &Cell<i32> — immutable reference
    r.set(100);             // TAPI boleh ubah!
    println!("{}", r.get()); // 100

    // Kenapa selamat?
    // Cell hanya benarkan GET (salin keluar) atau SET (salin masuk)
    // Tidak pernah ada dua references ke dalam pada masa yang sama!

    // ── Guna kes biasa ───────────────────────────────────────
    struct Perangkap {
        nama:         String,
        kiraan_akses: Cell<u32>,    // kira berapa kali diakses
    }

    impl Perangkap {
        fn nama(&self) -> &str {
            self.kiraan_akses.set(self.kiraan_akses.get() + 1);
            &self.nama
        }

        fn kiraan(&self) -> u32 {
            self.kiraan_akses.get()
        }
    }

    let p = Perangkap {
        nama:         "Ali".into(),
        kiraan_akses: Cell::new(0),
    };

    println!("{}", p.nama()); // Ali
    println!("{}", p.nama()); // Ali
    println!("Diakses: {} kali", p.kiraan()); // 2
    // p tidak perlu `mut`!

    // ── Cell methods ─────────────────────────────────────────
    let c = Cell::new(10i32);
    c.set(20);                   // set nilai baru
    let v = c.get();             // ambil salinan nilai
    let lama = c.replace(30);    // set dan return nilai lama
    let dalam = c.take();        // ambil nilai, ganti dengan Default
    println!("{} {} {} {}", v, lama, dalam, c.get());
    // 20 20 30 0
}
```

---

## Cell vs Direct Mutation

```rust
use std::cell::Cell;

// Bila guna Cell vs variable biasa:

// Variable biasa — perlu &mut self
struct CounterBiasa { nilai: u32 }
impl CounterBiasa {
    fn tambah(&mut self) { self.nilai += 1; }  // perlu mut!
    fn baca(&self) -> u32 { self.nilai }
}

// Dengan Cell — boleh ubah dalam &self
struct CounterCell { nilai: Cell<u32> }
impl CounterCell {
    fn tambah(&self) { self.nilai.set(self.nilai.get() + 1); }  // tiada mut!
    fn baca(&self) -> u32 { self.nilai.get() }
}

fn main() {
    // Kelebihan Cell: boleh ada &self dan ubah dalam masa yang sama
    let c = CounterCell { nilai: Cell::new(0) };
    let r1 = &c;
    let r2 = &c;
    r1.tambah();   // OK — kedua-dua &c dan ubah
    r2.tambah();   // OK
    println!("{}", c.baca()); // 2

    // Dengan CounterBiasa, tidak boleh ada r1 dan r2 dan mut serentak!
}
```

---

## 🧠 Brain Teaser #1

Kenapa `Cell<T>` hanya benarkan `T: Copy`? Apa yang berlaku kalau tidak?

<details>
<summary>👀 Jawapan</summary>

```rust
// Bayangkan Cell<String> (kalau dibenarkan)

struct HipotetisCellString(String);

impl HipotetisCellString {
    fn get(&self) -> String {  // Return String...
        self.0.clone()         // Kena clone! Tidak boleh move keluar
    }                          // dari belakang reference.
}

// Sebenarnya Rust ada UnsafeCell<T> untuk ini — tapi ia unsafe!
// Cell<T> selamat KERANA:
//   T: Copy → get() boleh salin nilai tanpa memindahkan ownership
//   T: Copy → tiada destructor → tiada masalah bila nilai "diganti"
//
// Kalau T tidak Copy:
//   get() perlu move nilai keluar — tapi nilai MASIH dalam Cell!
//   Dua "owner" untuk data yang sama = UB!
//
// Penyelesaian untuk non-Copy types:
//   RefCell<T>  → runtime borrow check, return Ref<T> guard
//   Mutex<T>    → untuk multi-thread
//
// Analogi Cell:
//   Seperti loker yang boleh masuk/keluar SALINAN barang
//   (bukan pindah barang sebenar)
//   Selamat kerana kita tidak boleh pegang dan simpan barang serentak
```
</details>

---

# BAB 2: RefCell\<T\> — Runtime Borrow Check 🔍

## RefCell — Pindahkan Check ke Runtime

```rust
use std::cell::RefCell;

fn main() {
    // RefCell<T> — untuk T yang tidak Copy
    // Periksa borrow rules pada RUNTIME (bukan compile time)
    // PANIC kalau rules dilanggar

    let data = RefCell::new(vec![1, 2, 3]);

    // ── borrow() — immutable reference ───────────────────────
    {
        let r1 = data.borrow();      // Ref<Vec<i32>>
        let r2 = data.borrow();      // OK! Boleh ada banyak &
        println!("{:?} {:?}", *r1, *r2);
    } // r1 dan r2 di-drop di sini

    // ── borrow_mut() — mutable reference ─────────────────────
    {
        let mut rm = data.borrow_mut(); // RefMut<Vec<i32>>
        rm.push(4);
        println!("{:?}", *rm); // [1, 2, 3, 4]
    } // rm di-drop di sini

    // ── Ini akan PANIC! ───────────────────────────────────────
    // let _r1 = data.borrow();
    // let _rm = data.borrow_mut(); // PANIC: already borrowed!

    // ── try_borrow() — selamat tanpa panic ───────────────────
    match data.try_borrow() {
        Ok(r)  => println!("Dapat borrow: {:?}", *r),
        Err(e) => println!("Gagal borrow: {}", e),
    }

    match data.try_borrow_mut() {
        Ok(mut m) => { m.push(5); println!("Dapat borrow_mut"); }
        Err(e)    => println!("Gagal borrow_mut: {}", e),
    }

    // ── Semak state semasa ────────────────────────────────────
    // (tidak ada API rasmi, tapi boleh try_borrow untuk cek)
}
```

---

## Bila RefCell PANIC

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(42i32);

    // ❌ PANIC — borrow_mut semasa ada borrow aktif
    let _r = data.borrow();        // borrow aktif
    // let _rm = data.borrow_mut(); // PANIC! "already borrowed"

    // ❌ PANIC — dua borrow_mut serentak
    let _rm1 = data.borrow_mut();
    // let _rm2 = data.borrow_mut(); // PANIC! "already mutably borrowed"

    // ✔ Selamat — pastikan borrow tidak overlap
    {
        let r = data.borrow();
        println!("{}", *r);
    } // r di-drop di sini
    {
        let mut rm = data.borrow_mut();
        *rm = 100;
    } // rm di-drop di sini

    // ✔ Lebih selamat — guna try_borrow
    if let Ok(r) = data.try_borrow() {
        println!("Nilai: {}", *r);
    }
}
```

---

# BAB 3: RefCell dalam Struct 🏗️

## Pattern yang Berguna

```rust
use std::cell::RefCell;
use std::collections::HashMap;

// ── Pattern 1: Lazy cache dalam struct immutable ──────────────
struct PekerjaService {
    db:    Database,                           // "real" data
    cache: RefCell<HashMap<u32, Pekerja>>,    // mutable cache
}

impl PekerjaService {
    fn dapatkan(&self, id: u32) -> Option<Pekerja> {
        // Semak cache dulu
        if let Some(p) = self.cache.borrow().get(&id) {
            return Some(p.clone()); // cache hit!
        }

        // Cache miss — query DB
        let pekerja = self.db.query(id)?;

        // Simpan dalam cache
        self.cache.borrow_mut().insert(id, pekerja.clone());

        Some(pekerja)
    }
}

// ── Pattern 2: Memoization ────────────────────────────────────
struct Fibonacci {
    memo: RefCell<HashMap<u64, u64>>,
}

impl Fibonacci {
    fn baru() -> Self {
        Fibonacci { memo: RefCell::new(HashMap::new()) }
    }

    fn kira(&self, n: u64) -> u64 {
        if n <= 1 { return n; }

        // Semak memo
        if let Some(&hasil) = self.memo.borrow().get(&n) {
            return hasil;
        }

        // Kira
        let hasil = self.kira(n - 1) + self.kira(n - 2);

        // Simpan dalam memo
        self.memo.borrow_mut().insert(n, hasil);

        hasil
    }
}

fn main() {
    let fib = Fibonacci::baru();
    for i in 0..10 {
        print!("{} ", fib.kira(i));
    }
    println!();
    // 0 1 1 2 3 5 8 13 21 34
}
```

---

## Struct dengan Multiple RefCell

```rust
use std::cell::RefCell;

struct Dokumen {
    tajuk:      String,             // tidak perlu ubah
    kandungan:  RefCell<String>,    // boleh edit
    metadata:   RefCell<Metadata>, // boleh kemaskini
    cache_hash: RefCell<Option<String>>,
}

#[derive(Debug, Clone)]
struct Metadata {
    penulis:   String,
    kali_edit: u32,
    terakhir:  String,
}

impl Dokumen {
    fn tambah_teks(&self, teks: &str) {
        let mut k = self.kandungan.borrow_mut();
        k.push_str(teks);

        let mut m = self.metadata.borrow_mut();
        m.kali_edit += 1;
        m.terakhir = "sekarang".into();

        // Invalidate cache
        *self.cache_hash.borrow_mut() = None;
    }

    fn hash(&self) -> String {
        // Semak cache dulu
        if let Some(ref h) = *self.cache_hash.borrow() {
            return h.clone();
        }

        // Kira hash baru
        let kandungan = self.kandungan.borrow();
        let hash = format!("hash:{}", kandungan.len());

        // Simpan dalam cache
        *self.cache_hash.borrow_mut() = Some(hash.clone());
        hash
    }
}
```

---

# BAB 4: Rc\<RefCell\<T\>\> — Shared Mutable Data 🔀

## Kombinasi Berkuasa

```rust
use std::rc::Rc;
use std::cell::RefCell;

// Rc<T>       → shared ownership (boleh ada banyak pemilik)
// RefCell<T>  → interior mutability
// Rc<RefCell<T>> → shared MUTABLE ownership (dalam single thread!)

#[derive(Debug)]
struct Nod {
    nilai:  i32,
    kiri:   Option<Rc<RefCell<Nod>>>,
    kanan:  Option<Rc<RefCell<Nod>>>,
    ibu:    Option<Rc<RefCell<Nod>>>,  // parent pointer!
}

type NodRef = Rc<RefCell<Nod>>;

fn buat_nod(nilai: i32) -> NodRef {
    Rc::new(RefCell::new(Nod {
        nilai,
        kiri:  None,
        kanan: None,
        ibu:   None,
    }))
}

fn sambung(ibu: &NodRef, anak: &NodRef) {
    // Ubah anak.ibu → ibu
    anak.borrow_mut().ibu = Some(Rc::clone(ibu));

    // Ubah ibu.kiri → anak
    ibu.borrow_mut().kiri = Some(Rc::clone(anak));
}

fn main() {
    let akar = buat_nod(1);
    let kiri = buat_nod(2);
    let kanan = buat_nod(3);

    sambung(&akar, &kiri);

    // Akar punya rujukan ke kiri, kiri punya rujukan ke akar
    // Ini REFERENCE CYCLE → memory leak!
    // Gunakan Weak<T> untuk mengelak!

    println!("Akar: {}", akar.borrow().nilai);
    println!("Kiri: {}", kiri.borrow().nilai);

    // Cara baca tanpa konflik
    {
        let borrow_akar = akar.borrow();
        if let Some(ref k) = borrow_akar.kiri {
            println!("Kiri nilai: {}", k.borrow().nilai);
        }
    }
}
```

---

## Elak Reference Cycle dengan Weak

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

#[derive(Debug)]
struct NodSelamat {
    nilai: i32,
    anak:  Vec<Rc<RefCell<NodSelamat>>>,
    ibu:   Option<Weak<RefCell<NodSelamat>>>,  // Weak! tidak ada cycle
}

impl NodSelamat {
    fn baru(nilai: i32) -> Rc<RefCell<Self>> {
        Rc::new(RefCell::new(NodSelamat {
            nilai,
            anak: Vec::new(),
            ibu:  None,
        }))
    }

    fn tambah_anak(
        ibu:  &Rc<RefCell<Self>>,
        anak: Rc<RefCell<Self>>,
    ) {
        // Set parent pada anak sebagai Weak reference
        anak.borrow_mut().ibu = Some(Rc::downgrade(ibu));

        // Tambah anak ke senarai anak ibu
        ibu.borrow_mut().anak.push(anak);
    }
}

fn main() {
    let akar = NodSelamat::baru(1);
    let anak1 = NodSelamat::baru(2);
    let anak2 = NodSelamat::baru(3);

    NodSelamat::tambah_anak(&akar, Rc::clone(&anak1));
    NodSelamat::tambah_anak(&akar, Rc::clone(&anak2));

    // Upgrade Weak → Rc untuk akses
    if let Some(ibu_ref) = anak1.borrow().ibu.as_ref() {
        if let Some(ibu) = ibu_ref.upgrade() {
            println!("Ibu: {}", ibu.borrow().nilai); // 1
        }
    }

    // Tiada reference cycle — boleh di-drop dengan betul!
}
```

---

## 🧠 Brain Teaser #2

Apakah output dan apakah masalahnya?

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(vec![1, 2, 3]);

    // Scenario A
    let r = data.borrow();
    let panjang = r.len();
    drop(r);  // ← drop manual
    data.borrow_mut().push(panjang as i32);
    println!("{:?}", data.borrow());

    // Scenario B
    let r2 = data.borrow();
    // data.borrow_mut().push(99); // ← apa jadi?
    println!("Panjang: {}", r2.len());
}
```

<details>
<summary>👀 Jawapan</summary>

```
Scenario A output: [1, 2, 3, 3]
  Berjaya! Kerana:
  1. borrow() → r aktif
  2. r.len() → 3
  3. drop(r) → r di-release SEBELUM borrow_mut
  4. borrow_mut() berjaya → push(3)
  5. Tiada overlap!

Scenario B:
  Jika baris data.borrow_mut().push(99) tidak di-comment:
  PANIC pada runtime!
  "already borrowed: BorrowMutError"
  Kerana r2 masih aktif semasa cuba borrow_mut()

Pelajaran penting:
  RefCell CHECK pada RUNTIME.
  Compiler tidak akan tangkap ini — ia akan PANIC semasa dijalankan!
  
  Ini berbeza dari compile-time checks:
  let v = vec![1, 2, 3];
  let r = &v;
  v.push(4); // ← COMPILE ERROR — compiler tangkap ini!

  Dengan RefCell, kita "swap" keselamatan compile-time
  untuk fleksibiliti runtime.
  GUNA DENGAN BERHATI-HATI!
```
</details>

---

# BAB 5: Mutex\<T\> — Thread-safe Mutability 🔒

## Mutex untuk Multi-Threading

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    // Mutex<T> — satu thread pada satu masa
    // Arc<Mutex<T>> — shared ownership + thread-safe mutability
    let kiraan = Arc::new(Mutex::new(0i32));

    let mut handles = Vec::new();

    for _ in 0..10 {
        let k = Arc::clone(&kiraan);
        handles.push(thread::spawn(move || {
            for _ in 0..100 {
                let mut guard = k.lock().unwrap(); // acquire lock
                *guard += 1;
            } // lock dilepaskan di sini (guard di-drop)
        }));
    }

    for h in handles { h.join().unwrap(); }

    println!("Kiraan: {}", *kiraan.lock().unwrap()); // 1000

    // ── Mutex methods ─────────────────────────────────────────
    let m = Mutex::new(vec![1, 2, 3]);

    // lock() — block sampai dapat lock, panic kalau poisoned
    {
        let mut guard = m.lock().unwrap();
        guard.push(4);
    } // lock dilepas

    // try_lock() — jangan block, return Err kalau tidak dapat
    match m.try_lock() {
        Ok(guard)  => println!("{:?}", *guard),
        Err(e)     => println!("Tidak dapat lock: {}", e),
    }
}
```

---

## Deadlock — Perangkap Mutex

```rust
use std::sync::{Arc, Mutex};
use std::thread;

// ❌ DEADLOCK — Thread 1 pegang A, tunggu B
//               Thread 2 pegang B, tunggu A
fn deadlock_contoh() {
    let mutex_a = Arc::new(Mutex::new(()));
    let mutex_b = Arc::new(Mutex::new(()));

    let (a1, b1) = (Arc::clone(&mutex_a), Arc::clone(&mutex_b));
    let t1 = thread::spawn(move || {
        let _ga = a1.lock().unwrap(); // pegang A
        thread::sleep(std::time::Duration::from_millis(10));
        let _gb = b1.lock().unwrap(); // tunggu B ← DEADLOCK!
    });

    let (a2, b2) = (Arc::clone(&mutex_a), Arc::clone(&mutex_b));
    let t2 = thread::spawn(move || {
        let _gb = b2.lock().unwrap(); // pegang B
        thread::sleep(std::time::Duration::from_millis(10));
        let _ga = a2.lock().unwrap(); // tunggu A ← DEADLOCK!
    });

    t1.join().unwrap(); // Ini tidak akan selesai!
    t2.join().unwrap();
}

// ✔ Elak deadlock — sentiasa lock dalam ORDER yang sama
fn selamat_dari_deadlock() {
    let mutex_a = Arc::new(Mutex::new(()));
    let mutex_b = Arc::new(Mutex::new(()));

    // KEDUA-DUA thread lock A dulu, kemudian B
    // Tiada deadlock!
}

// ✔ Elak deadlock — minimumkan scope lock
fn kurangkan_scope() {
    let m = Mutex::new(vec![1, 2, 3]);

    // Buruk — pegang lock terlalu lama
    let _guard = m.lock().unwrap();
    // ... banyak kerja ...
    // operasi IO yang lambat semasa lock dipegang!

    // Baik — ambil data, lepaskan lock, proses
    let data = m.lock().unwrap().clone(); // clone dan terus lepas
    // ... proses data tanpa lock ...
}
```

---

## Poison Mutex

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let m = Arc::new(Mutex::new(42i32));
    let m2 = Arc::clone(&m);

    // Thread yang panic semasa pegang lock → mutex POISONED
    let _ = thread::spawn(move || {
        let _guard = m2.lock().unwrap();
        panic!("Thread panic!"); // mutex jadi poisoned!
    }).join();

    // lock() return Err pada poisoned mutex
    match m.lock() {
        Ok(v)  => println!("OK: {}", *v),
        Err(e) => {
            println!("Mutex poisoned! Tapi data masih boleh diakses.");
            let v = e.into_inner(); // ambil data dari poisoned mutex
            println!("Data: {}", *v);
        }
    }

    // Atau guna into_inner() secara langsung
    // Untuk recover dari poisoning:
    let m3 = Arc::new(Mutex::new(0i32));
    // ... selepas panic ...
    let _ = m3.lock().unwrap_or_else(|e| e.into_inner());
}
```

---

# BAB 6: RwLock\<T\> — Readers/Writer Lock 📖

## Multiple Readers, One Writer

```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    // RwLock = boleh ada BANYAK pembaca ATAU SATU penulis
    // Lebih laju dari Mutex kalau read-heavy workload

    let data = Arc::new(RwLock::new(vec![1, 2, 3, 4, 5]));

    // ── Multiple readers serentak ─────────────────────────────
    let mut handles = Vec::new();
    for i in 0..5 {
        let d = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            let r = d.read().unwrap(); // shared read lock
            println!("Thread {} baca: {:?}", i, *r);
            // Semua thread baca serentak!
        }));
    }
    for h in handles { h.join().unwrap(); }

    // ── Single writer ─────────────────────────────────────────
    {
        let mut w = data.write().unwrap(); // exclusive write lock
        w.push(6);
        println!("Selepas tulis: {:?}", *w);
    }

    // ── try_read / try_write ──────────────────────────────────
    match data.try_read() {
        Ok(r)  => println!("Dapat baca: {:?}", *r),
        Err(_) => println!("Tidak dapat baca sekarang"),
    }

    match data.try_write() {
        Ok(mut w) => { w.push(7); println!("Dapat tulis!"); }
        Err(_)    => println!("Tidak dapat tulis sekarang"),
    }
}
```

---

## RwLock vs Mutex — Bila Guna Mana?

```
Mutex:
  ✔ Operasi ringkas (increment counter)
  ✔ Write-heavy (banyak write, sikit read)
  ✔ Operasi pendek (kurang contention)
  ✔ Simple dan mudah reasoning

RwLock:
  ✔ Read-heavy workload (banyak baca, sikit tulis)
  ✔ Konfigurasi / cache yang jarang berubah
  ✔ Baca lebih lama (complex calculations)
  ✗ Boleh ada "writer starvation" kalau readers terlalu banyak

MUTEX IS ALMOST ALWAYS BETTER unless:
  - Reads are >>10x more frequent than writes, AND
  - Read operations take meaningful time
```

---

# BAB 7: OnceCell & OnceLock ☝️

## Inisialisasi Sekali Sahaja

```rust
use std::cell::OnceCell;
use std::sync::OnceLock;

// ── OnceCell — single thread ──────────────────────────────────
fn demo_once_cell() {
    let cell: OnceCell<String> = OnceCell::new();

    // Set sekali sahaja
    cell.set("Hello".to_string()).unwrap();

    // Percubaan set kedua → Err
    assert!(cell.set("Dunia".to_string()).is_err());

    // Get
    println!("{}", cell.get().unwrap()); // Hello

    // get_or_init — set kalau belum ada
    let cell2: OnceCell<i32> = OnceCell::new();
    let nilai = cell2.get_or_init(|| {
        println!("(menginisialisasi...)");
        42
    });
    println!("{}", nilai); // (menginisialisasi...) → 42

    let nilai2 = cell2.get_or_init(|| {
        println!("(ini tidak akan print)");
        99
    });
    println!("{}", nilai2); // 42 (masih yang lama)
}

// ── OnceLock — multi thread ───────────────────────────────────
static KONFIGURASI: OnceLock<String> = OnceLock::new();
static REGEX_EMEL: OnceLock<regex::Regex> = OnceLock::new();

fn dapatkan_konfigurasi() -> &'static str {
    KONFIGURASI.get_or_init(|| {
        // Baca dari env, DB, atau fail — hanya sekali!
        std::env::var("APP_CONFIG").unwrap_or_else(|_| "default".into())
    })
}

fn regex_emel() -> &'static regex::Regex {
    REGEX_EMEL.get_or_init(|| {
        regex::Regex::new(r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$")
            .expect("Regex tidak sah")
    })
}

fn main() {
    demo_once_cell();

    println!("{}", dapatkan_konfigurasi());

    // Regex dikompil sekali, guna banyak kali
    let emel = "ali@email.com";
    println!("Valid: {}", regex_emel().is_match(emel));
}
```

---

# BAB 8: Lazy & LazyLock 🦥

## Inisialisasi Malas

```rust
use std::cell::LazyCell;
use std::sync::LazyLock;

// ── LazyCell — single thread, stable Rust 1.80+ ───────────────
fn demo_lazy_cell() {
    // Diinisialisasi pada akses pertama
    let malas: LazyCell<Vec<i32>> = LazyCell::new(|| {
        println!("(mengira senarai...)");
        (1..=10).collect()
    });

    println!("Belum akses lagi");
    println!("{:?}", *malas); // (mengira senarai...) → [1,2,...,10]
    println!("{:?}", *malas); // [1,2,...,10] (tidak dikira semula)
}

// ── LazyLock — multi thread, static ──────────────────────────
static JADUAL_LOOKUP: LazyLock<std::collections::HashMap<&str, u32>> =
    LazyLock::new(|| {
        let mut m = std::collections::HashMap::new();
        m.insert("ICT",      1);
        m.insert("HR",       2);
        m.insert("Kewangan", 3);
        m.insert("Pertanian", 4);
        println!("(jadual lookup dibina)");
        m
    });

static NOMBOR_FIBONACCI: LazyLock<Vec<u64>> = LazyLock::new(|| {
    let mut fib = vec![0u64, 1];
    for i in 2..=100 {
        let n = fib[i-1].saturating_add(fib[i-2]);
        fib.push(n);
    }
    fib
});

fn main() {
    demo_lazy_cell();

    // Lookup jadual — dibina sekali, guna banyak kali
    println!("{:?}", JADUAL_LOOKUP.get("ICT"));       // Some(1)
    println!("{:?}", JADUAL_LOOKUP.get("HR"));        // Some(2)

    // Fibonacci — dikira sekali
    println!("Fib(10) = {}", NOMBOR_FIBONACCI[10]);  // 55
    println!("Fib(20) = {}", NOMBOR_FIBONACCI[20]);  // 6765
}
```

---

# BAB 9: Pilih Yang Betul 🎯

## Carta Keputusan

```
Soal: Apa yang anda nak ubah?

┌─ Boleh T: Copy? ─────────────────────────────────────────┐
│  Ya → Cell<T>  (paling ringan, tiada overhead)           │
│  Tidak → teruskan...                                      │
└──────────────────────────────────────────────────────────┘

┌─ Single thread atau Multi thread? ───────────────────────┐
│                                                           │
│  SINGLE THREAD:                                           │
│   Inisialisasi sekali → OnceCell<T>                       │
│   Malas, sekali      → LazyCell<T>                        │
│   Ubah semasa pinjam → RefCell<T>                         │
│   Shared + mutable   → Rc<RefCell<T>>                     │
│                                                           │
│  MULTI THREAD:                                            │
│   Inisialisasi sekali → OnceLock<T>                       │
│   Malas global, sekali → LazyLock<T>                      │
│   Ubah dari thread   → Arc<Mutex<T>>                      │
│   Read-heavy         → Arc<RwLock<T>>                     │
│   Nombor sahaja      → Atomic*(Relaxed/SeqCst)            │
└──────────────────────────────────────────────────────────┘

Soal tambahan:
  "Adakah data berkongsi antara banyak pemilik?"
   Ya → tambah Rc<...> atau Arc<...>
   Tidak → RefCell<T> atau Mutex<T> sahaja
```

---

## Perbandingan Penuh

```
Type              | Thread | Overhead | Panic? | Use Case
──────────────────────────────────────────────────────────────────
Cell<T>           | Single | Zero     | No     | Copy types, counters
RefCell<T>        | Single | Small    | Yes*   | Structs, trees, cache
OnceCell<T>       | Single | Tiny     | No     | Lazy init, config
LazyCell<T>       | Single | Tiny     | No     | Lazy computed values
Mutex<T>          | Multi  | Medium   | No**   | Write-heavy data
RwLock<T>         | Multi  | Medium+  | No**   | Read-heavy data
OnceLock<T>       | Multi  | Tiny     | No     | Static config, regex
LazyLock<T>       | Multi  | Tiny     | No     | Static lazy values
AtomicU32, dll    | Multi  | Zero     | No     | Numbers only, fast
Rc<RefCell<T>>    | Single | Small    | Yes*   | Shared mutable (graph)
Arc<Mutex<T>>     | Multi  | Medium   | No**   | Shared mutable threads

* Panic bila borrow rules dilanggar (runtime check)
** Panic kalau mutex poisoned atau deadlock
```

---

# BAB 10: Mini Project — Cache System 🏗️

```rust
use std::cell::RefCell;
use std::collections::HashMap;
use std::sync::{Arc, RwLock};
use std::time::{Duration, Instant};
use serde::{Serialize, Deserialize};

// ─── Single-thread Cache (dengan RefCell) ────────────────────

struct CacheSingleThread<K, V> {
    stor:    RefCell<HashMap<K, (V, Instant)>>,
    ttl:     Duration,
    maks:    usize,
}

impl<K: std::hash::Hash + Eq + Clone, V: Clone> CacheSingleThread<K, V> {
    fn baru(ttl_saat: u64, maks: usize) -> Self {
        CacheSingleThread {
            stor: RefCell::new(HashMap::new()),
            ttl:  Duration::from_secs(ttl_saat),
            maks,
        }
    }

    fn dapatkan(&self, kunci: &K) -> Option<V> {
        let stor = self.stor.borrow();
        stor.get(kunci).and_then(|(nilai, masa)| {
            if masa.elapsed() < self.ttl {
                Some(nilai.clone())
            } else {
                None // tamat tempoh
            }
        })
    }

    fn simpan(&self, kunci: K, nilai: V) {
        let mut stor = self.stor.borrow_mut();

        // Elak melampaui had
        if stor.len() >= self.maks {
            // Buang entry yang paling lama
            if let Some(kunci_lama) = stor
                .iter()
                .min_by_key(|(_, (_, masa))| *masa)
                .map(|(k, _)| k.clone())
            {
                stor.remove(&kunci_lama);
            }
        }

        stor.insert(kunci, (nilai, Instant::now()));
    }

    fn padam(&self, kunci: &K) {
        self.stor.borrow_mut().remove(kunci);
    }

    fn bersih_tamat(&self) -> usize {
        let mut stor = self.stor.borrow_mut();
        let ttl = self.ttl;
        let sebelum = stor.len();
        stor.retain(|_, (_, masa)| masa.elapsed() < ttl);
        sebelum - stor.len()
    }
}

// ─── Multi-thread Cache (dengan RwLock) ──────────────────────

#[derive(Clone)]
struct CacheMultiThread<K, V> {
    stor: Arc<RwLock<HashMap<K, (V, Instant)>>>,
    ttl:  Duration,
    maks: usize,
}

impl<K, V> CacheMultiThread<K, V>
where
    K: std::hash::Hash + Eq + Clone + Send + Sync + 'static,
    V: Clone + Send + Sync + 'static,
{
    fn baru(ttl_saat: u64, maks: usize) -> Self {
        CacheMultiThread {
            stor: Arc::new(RwLock::new(HashMap::new())),
            ttl:  Duration::from_secs(ttl_saat),
            maks,
        }
    }

    fn dapatkan(&self, kunci: &K) -> Option<V> {
        let stor = self.stor.read().unwrap();
        stor.get(kunci).and_then(|(nilai, masa)| {
            if masa.elapsed() < self.ttl {
                Some(nilai.clone())
            } else {
                None
            }
        })
    }

    fn dapatkan_atau_muat<F>(&self, kunci: K, muat: F) -> V
    where
        F: FnOnce() -> V,
    {
        // Cuba baca dulu (tanpa write lock)
        if let Some(v) = self.dapatkan(&kunci) {
            return v;
        }

        // Perlu muat — ambil write lock
        let nilai = muat();
        self.simpan(kunci, nilai.clone());
        nilai
    }

    fn simpan(&self, kunci: K, nilai: V) {
        let mut stor = self.stor.write().unwrap();

        if stor.len() >= self.maks && !stor.contains_key(&kunci) {
            if let Some(kunci_lama) = stor
                .iter()
                .min_by_key(|(_, (_, masa))| *masa)
                .map(|(k, _)| k.clone())
            {
                stor.remove(&kunci_lama);
            }
        }

        stor.insert(kunci, (nilai, Instant::now()));
    }

    fn saiz(&self) -> usize {
        self.stor.read().unwrap().len()
    }

    fn bersih_tamat(&self) -> usize {
        let mut stor = self.stor.write().unwrap();
        let ttl = self.ttl;
        let sebelum = stor.len();
        stor.retain(|_, (_, masa)| masa.elapsed() < ttl);
        sebelum - stor.len()
    }
}

// ─── Demo ─────────────────────────────────────────────────────

fn main() {
    println!("=== Single Thread Cache (RefCell) ===\n");

    let cache: CacheSingleThread<String, String> =
        CacheSingleThread::baru(5, 3); // TTL 5 saat, maks 3

    // Simpan
    cache.simpan("pekerja:1".into(), "Ali Ahmad".into());
    cache.simpan("pekerja:2".into(), "Siti Hawa".into());

    // Baca
    println!("{:?}", cache.dapatkan(&"pekerja:1".into())); // Some("Ali Ahmad")
    println!("{:?}", cache.dapatkan(&"tiada".into()));     // None

    // Cache dalam struct immutable!
    struct Servis {
        cache: CacheSingleThread<u32, String>,
    }

    impl Servis {
        fn dapatkan_nama(&self, id: u32) -> String {
            // Semak cache (ubah self.cache walaupun &self!)
            if let Some(nama) = self.cache.dapatkan(&id) {
                println!("Cache HIT: {}", id);
                return nama;
            }

            // Simulate DB query
            let nama = format!("Pekerja {}", id);
            println!("Cache MISS: {} → '{}'", id, nama);
            self.cache.simpan(id, nama.clone());
            nama
        }
    }

    let servis = Servis {
        cache: CacheSingleThread::baru(60, 100),
    };

    println!("\n{}", servis.dapatkan_nama(1)); // MISS
    println!("{}", servis.dapatkan_nama(1));   // HIT
    println!("{}", servis.dapatkan_nama(2));   // MISS

    println!("\n=== Multi Thread Cache (RwLock) ===\n");

    let cache_mt: CacheMultiThread<String, i32> =
        CacheMultiThread::baru(10, 100);

    let cache_klon = cache_mt.clone();

    // Spawn threads
    let mut handles = Vec::new();
    for i in 0..5 {
        let c = cache_mt.clone();
        handles.push(std::thread::spawn(move || {
            let kunci = format!("kunci:{}", i % 3); // 0, 1, 2 untuk overlap
            let nilai = c.dapatkan_atau_muat(kunci.clone(), || {
                std::thread::sleep(Duration::from_millis(10));
                i * 10
            });
            println!("Thread {}: {} = {}", i, kunci, nilai);
        }));
    }

    for h in handles { h.join().unwrap(); }
    println!("\nSaiz cache: {}", cache_klon.saiz());
}
```

---

# 📋 Rujukan Pantas — Interior Mutability Cheat Sheet

## Quick Selection Guide

```rust
// Copy types, single thread, zero overhead
let c = Cell::new(42u32);
c.set(100);
let v = c.get();

// Any type, single thread, runtime check
let r = RefCell::new(vec![1,2,3]);
r.borrow().len();          // immutable
r.borrow_mut().push(4);    // mutable

// Shared ownership + mutability, single thread
let shared = Rc::new(RefCell::new(data));
let klon = Rc::clone(&shared);
klon.borrow_mut().ubah();

// Thread-safe, any type, exclusive access
let m = Arc::new(Mutex::new(data));
*m.lock().unwrap() = nilai_baru;

// Thread-safe, read-heavy
let rw = Arc::new(RwLock::new(data));
rw.read().unwrap();        // multiple readers ok
rw.write().unwrap();       // one writer only

// Initialize once, single thread
let o = OnceCell::new();
o.get_or_init(|| compute());

// Initialize once, multi thread (static)
static O: OnceLock<Type> = OnceLock::new();
O.get_or_init(|| compute());

// Lazy, single thread
let l = LazyCell::new(|| compute());

// Lazy, multi thread (static)
static L: LazyLock<Type> = LazyLock::new(|| compute());
```

## Prinsip Penting

```
1. Cell<T>  → T: Copy sahaja, zero overhead
2. RefCell  → PANIC kalau rules dilanggar (runtime!)
3. Mutex    → DEADLOCK kalau tidak berhati-hati
4. RwLock   → Lebih baik dari Mutex HANYA untuk read-heavy
5. Arc/Rc   → Untuk shared ownership
6. OnceCell/Lock → Init sekali, baca selalu
7. Lazy     → Deferred init pada akses pertama
```

---

## 🏆 Cabaran Akhir

Cuba implement salah satu:

1. **Observer Pattern** — guna `Rc<RefCell<Vec<Box<dyn Observer>>>>` untuk event system
2. **Thread-safe Counter** — bandingkan kelajuan `Mutex<u64>` vs `AtomicU64` vs `RwLock<u64>`
3. **Lazy Config** — `LazyLock<Config>` yang baca fail JSON sekali semasa startup
4. **Memo dengan TTL** — `RefCell<HashMap<K, (V, Instant)>>` dengan auto-expire
5. **Double-linked List** — `Rc<RefCell<Nod>>` + `Weak<RefCell<Nod>>` untuk prev pointer

---

*Interior mutability adalah "escape hatch" yang selamat dari Rust borrow checker.*
*Guna hanya bila perlu — compile-time checks selalu lebih baik.*
*Pilih type yang paling ringan yang sesuai dengan keperluan anda.* 🦀
