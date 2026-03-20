# 🧠 Smart Pointers dalam Rust — Beginner to Advanced

> Panduan lengkap Box, Rc, Arc, RefCell, Cell, Weak dan lebih lagi.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu Smart Pointer?

```
Pointer biasa (raw pointer):
  ┌─────┐          ┌──────────┐
  │ ptr │─────────▶│  data    │
  └─────┘          └──────────┘
  Tunjuk ke alamat. Tiada info lain. Boleh dangling!

Smart Pointer:
  ┌──────────────────────────┐    ┌──────────┐
  │  ptr  │  metadata        │───▶│  data    │
  │       │  (len, cap, dll) │    └──────────┘
  └──────────────────────────┘
  Ada data tambahan. Ada ownership. Auto-cleanup!
```

Smart pointer adalah struct yang **berkelakuan seperti pointer** tapi ada keupayaan tambahan — seperti mengira rujukan, membolehkan mutability dalaman, atau memastikan thread-safety.

---

## Senarai Smart Pointers Rust

```
┌──────────────────┬────────────────────────────────────────────┐
│  Smart Pointer   │  Kegunaan Utama                            │
├──────────────────┼────────────────────────────────────────────┤
│  Box<T>          │  Heap allocation, recursive types          │
│  Rc<T>           │  Multiple owners, single-thread            │
│  Arc<T>          │  Multiple owners, multi-thread             │
│  RefCell<T>      │  Interior mutability, single-thread        │
│  Cell<T>         │  Interior mutability, Copy types           │
│  Weak<T>         │  Non-owning reference (elak cycle)         │
│  Mutex<T>        │  Thread-safe mutable access                │
│  RwLock<T>       │  Many readers OR one writer                │
│  Cow<T>          │  Clone-on-write (lazy clone)               │
└──────────────────┴────────────────────────────────────────────┘
```

---

## Peta Pembelajaran

```
Bab 1  → Box<T> — Heap Allocation
Bab 2  → Rc<T> — Reference Counting
Bab 3  → Arc<T> — Atomic Reference Counting
Bab 4  → RefCell<T> — Interior Mutability
Bab 5  → Cell<T> — Simple Interior Mutability
Bab 6  → Weak<T> — Elak Reference Cycle
Bab 7  → Gabungan — Rc<RefCell<T>>, Arc<Mutex<T>>
Bab 8  → Cow<T> — Clone on Write
Bab 9  → Custom Smart Pointer
Bab 10 → Mini Project: Graf Direkted
```

---

# BAB 1: Box<T> — Heap Allocation 📦

## Apa Itu Box?

```
Tanpa Box (stack):              Dengan Box (heap):

  Stack                           Stack        Heap
  ┌─────────────┐                 ┌───────┐    ┌─────────────┐
  │  data: 42   │                 │  ptr  │───▶│  data: 42   │
  └─────────────┘                 └───────┘    └─────────────┘
  Data terus dalam stack          Box dalam stack, data dalam heap
  Saiz mesti tahu masa compile    Saiz Box tetap = 8 bytes (pointer)
```

```rust
fn main() {
    // Cara paling mudah — bungkus nilai dalam heap
    let b = Box::new(5);
    println!("b = {}", b);       // 5 — dereference automatik!
    println!("b = {}", *b);      // 5 — explicit dereference

    // Box<T> implement Deref — boleh guna macam T biasa
    let s = Box::new(String::from("hello"));
    println!("len = {}", s.len()); // guna method String terus!

    // Box di-drop bila keluar scope — data dalam heap dibersihkan
    {
        let _x = Box::new(vec![1, 2, 3]);
    } // _x di-drop di sini, heap memory dibebaskan

    // Memindah ownership keluar dari Box
    let boxed = Box::new(String::from("Rust"));
    let s: String = *boxed; // dereference dan move keluar
    println!("{}", s);      // "Rust"
}
```

---

## Bila Perlu Box?

### 1. Recursive Types (Paling Penting!)

```rust
// ❌ TIDAK COMPILE — saiz tak terhingga!
// enum List {
//     Cons(i32, List),   // List mengandungi List!
//     Nil,
// }

// ✔ OK — Box ada saiz tetap (pointer = 8 bytes)
#[derive(Debug)]
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use List::{Cons, Nil};

fn main() {
    // Bina senarai: 1 → 2 → 3 → Nil
    let senarai = Cons(1,
        Box::new(Cons(2,
            Box::new(Cons(3,
                Box::new(Nil))))));

    println!("{:?}", senarai);
}
```

```
Memory layout:
  Stack          Heap
  ┌────────┐    ┌─────────────────┐
  │ Cons   │───▶│ 1 │ ptr ────────┼──▶ ┌─────────────────┐
  └────────┘    └─────────────────┘    │ 2 │ ptr ─────────┼──▶ ┌──────────┐
                                       └─────────────────┘    │ 3 │ Nil  │
                                                               └──────────┘
```

### 2. Trait Objects (Dynamic Dispatch)

```rust
trait Bunyi {
    fn bunyi(&self) -> &str;
}

struct Kucing;
struct Anjing;
struct Ikan;

impl Bunyi for Kucing { fn bunyi(&self) -> &str { "Meow" } }
impl Bunyi for Anjing { fn bunyi(&self) -> &str { "Woof" } }
impl Bunyi for Ikan   { fn bunyi(&self) -> &str { "..." } }

fn main() {
    // Box<dyn Trait> — simpan pelbagai type dalam satu Vec!
    let haiwan: Vec<Box<dyn Bunyi>> = vec![
        Box::new(Kucing),
        Box::new(Anjing),
        Box::new(Ikan),
        Box::new(Kucing),
    ];

    for h in &haiwan {
        println!("{}", h.bunyi()); // dynamic dispatch
    }

    // Tanpa Box, tidak boleh kerana saiz berbeza:
    // let v: Vec<dyn Bunyi> = vec![...]; // ← ERROR! saiz tidak tetap
}
```

### 3. Pindah Data Besar ke Heap

```rust
fn main() {
    // Data besar — elak copy di stack
    let data_besar = Box::new([0u8; 1_000_000]); // 1MB dalam heap
    // Passing ke fungsi — hanya pointer (8 bytes) yang dipindah!
    proses(data_besar);
}

fn proses(_data: Box<[u8; 1_000_000]>) {
    println!("Proses data...");
}
```

---

## 🧠 Brain Teaser #1

Kenapa kod ini perlu `Box`?

```rust
enum Pokok {
    Daun(i32),
    Nod(Box<Pokok>, i32, Box<Pokok>), // kiri, nilai, kanan
}
```

<details>
<summary>👀 Jawapan</summary>

Tanpa `Box`, `Pokok` akan mengandungi `Pokok` secara langsung dalam dirinya — ini bermakna saiznya tak terhingga (`sizeof(Pokok) = sizeof(Pokok) + ...`).

`Box<Pokok>` mempunyai saiz tetap (8 bytes — saiz pointer), tidak kira seberapa besar pokok yang ada dalam heap.

```
Dengan Box:
  sizeof(Pokok::Nod) = sizeof(Box) + sizeof(i32) + sizeof(Box)
                     = 8 + 4 + 8 = 20 bytes  ← tetap!

Tanpa Box:
  sizeof(Pokok::Nod) = sizeof(Pokok) + 4 + sizeof(Pokok)
                     = ∞  ← GAGAL compile!
```
</details>

---

# BAB 2: Rc<T> — Reference Counting 🔢

## Apa Itu Rc?

```
Ownership biasa — SATU owner:
  let a = String::from("hello");
  let b = a;    // a tidak valid lagi!

Rc<T> — BANYAK owner, reference count:
  let a = Rc::new(String::from("hello"));
  let b = Rc::clone(&a);  // tambah count → 2
  let c = Rc::clone(&a);  // tambah count → 3

  ┌───┐    ┌─────────────────────────────┐
  │ a │───▶│  strong_count: 3            │
  └───┘    │  data: "hello"              │
  ┌───┐    │                             │
  │ b │───▶│                             │
  └───┘    └─────────────────────────────┘
  ┌───┐
  │ c │───▶ (sama allocation)
  └───┘

  drop(b) → count: 2
  drop(c) → count: 1
  drop(a) → count: 0 → data dibersihkan!
```

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(String::from("hello"));
    println!("Count selepas buat a: {}", Rc::strong_count(&a)); // 1

    let b = Rc::clone(&a); // BUKAN deep clone! Hanya tambah count
    println!("Count selepas clone b: {}", Rc::strong_count(&a)); // 2

    {
        let c = Rc::clone(&a);
        println!("Count dalam scope: {}", Rc::strong_count(&a)); // 3
        // c di-drop di sini
    }

    println!("Count selepas c drop: {}", Rc::strong_count(&a)); // 2

    // a dan b kongsi DATA YANG SAMA — bukan copy!
    println!("a: {}", a); // "hello"
    println!("b: {}", b); // "hello"
    println!("Sama? {}", Rc::ptr_eq(&a, &b)); // true — sama pointer!

    // ⚠ Rc hanya IMMUTABLE! Tiada &mut melalui Rc
    // a.push_str(" world"); // ← ERROR!
}
```

---

## Rc dalam Struktur Data

```rust
use std::rc::Rc;

#[derive(Debug)]
struct Nod {
    nilai: i32,
    kiri:  Option<Rc<Nod>>,
    kanan: Option<Rc<Nod>>,
}

fn main() {
    // Buat nod yang dikongsi oleh beberapa pokok
    let dikongsi = Rc::new(Nod { nilai: 5, kiri: None, kanan: None });

    let pokok1 = Rc::new(Nod {
        nilai: 10,
        kiri:  Some(Rc::clone(&dikongsi)),
        kanan: None,
    });

    let pokok2 = Rc::new(Nod {
        nilai: 20,
        kiri:  Some(Rc::clone(&dikongsi)),
        kanan: None,
    });

    // `dikongsi` (nod nilai=5) ada dalam DUA pokok serentak!
    println!("Count dikongsi: {}", Rc::strong_count(&dikongsi)); // 3
    println!("Pokok1 kiri: {:?}", pokok1.kiri.as_ref().map(|n| n.nilai)); // Some(5)
    println!("Pokok2 kiri: {:?}", pokok2.kiri.as_ref().map(|n| n.nilai)); // Some(5)
}
```

---

## ⚠ Rc Tidak Thread-Safe!

```rust
use std::rc::Rc;

// ❌ TIDAK COMPILE — Rc tidak boleh hantar merentasi thread
// fn main() {
//     let a = Rc::new(5);
//     std::thread::spawn(move || {
//         println!("{}", a); // ERROR: Rc cannot be sent between threads
//     });
// }

// ✔ Guna Arc untuk multi-thread (lihat Bab 3)
```

---

# BAB 3: Arc<T> — Atomic Reference Counting ⚛️

## Arc = Rc + Thread Safe

```
Rc<T>  → Reference count pakai operasi biasa → cepat, single-thread sahaja
Arc<T> → Reference count pakai atomic ops    → sedikit lambat, multi-thread OK

Guna Arc bila:
  ✔ Data dikongsi antara threads
  ✔ Guna dengan tokio::spawn, std::thread::spawn
  ✔ Async programming

Guna Rc bila:
  ✔ Single-thread (GUI, game loop, parser)
  ✔ Nak lebih laju (tiada atomic overhead)
```

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3, 4, 5]);
    let mut handles = vec![];

    for i in 0..3 {
        let data_clone = Arc::clone(&data); // tambah count, hantar ke thread
        let handle = thread::spawn(move || {
            println!("Thread {}: {:?}", i, data_clone);
            println!("Thread {} count: {}", i, Arc::strong_count(&data_clone));
        });
        handles.push(handle);
    }

    for h in handles { h.join().unwrap(); }

    println!("Main count: {}", Arc::strong_count(&data)); // 1 (semua thread habis)
    println!("Data: {:?}", data);
}
```

---

## Arc dengan Mutex untuk Mutable Shared State

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
            let mut guard = k.lock().unwrap(); // ambil lock
            *guard += 1;
        })); // guard di-drop → lock dilepas
    }

    for h in handles { h.join().unwrap(); }

    println!("Kaunter: {}", *kaunter.lock().unwrap()); // 10
}
```

---

## 🧠 Brain Teaser #2

Bila nak guna `Rc<T>` dan bila `Arc<T>`?

```rust
// Senario A: Parser HTML dalam satu thread
// Senario B: Download manager dengan thread pool
// Senario C: Game loop dengan shared game state, single-thread
// Senario D: Web server yang handle banyak request serentak
```

<details>
<summary>👀 Jawapan</summary>

```
A. Parser HTML → Rc<T>
   Satu thread, tiada perlu atomic ops

B. Download manager → Arc<T>
   Thread pool = banyak thread kongsi data

C. Game loop single-thread → Rc<T>
   Single-thread, lebih laju, tiada overhead atomic

D. Web server → Arc<T>
   Banyak request = banyak thread/task serentak
```

**Peraturan mudah:**
- `std::thread::spawn` atau `tokio::spawn` terlibat? → `Arc<T>`
- Semua dalam satu thread? → `Rc<T>` (lebih laju)

Compiler akan beritahu kalau salah! Rc tidak implement `Send` + `Sync`.
</details>

---

# BAB 4: RefCell<T> — Interior Mutability 🔄

## Masalah: Nak Ubah Tapi Tiada &mut

```
Peraturan borrow Rust (compile-time):
  ✔ Banyak &T (immutable borrow) serentak
  ✔ ATAU satu &mut T (mutable borrow)
  ✘ Tidak boleh kedua-dua serentak

RefCell<T> — borrow checking pada RUNTIME:
  .borrow()     → dapat Ref<T>    (macam &T)
  .borrow_mut() → dapat RefMut<T> (macam &mut T)

  Kalau langgar peraturan → PANIC pada runtime (bukan compile error)
```

```rust
use std::cell::RefCell;

fn main() {
    // RefCell membenarkan mutability walaupun binding immutable
    let data = RefCell::new(vec![1, 2, 3]);

    // Baca (borrow)
    {
        let baca = data.borrow(); // Ref<Vec<i32>>
        println!("Data: {:?}", *baca);
    } // Ref di-drop di sini

    // Ubah (borrow_mut)
    {
        let mut tulis = data.borrow_mut(); // RefMut<Vec<i32>>
        tulis.push(4);
        tulis.push(5);
    } // RefMut di-drop di sini

    println!("Selepas ubah: {:?}", data.borrow()); // [1,2,3,4,5]

    // try_borrow / try_borrow_mut — return Result, tidak panic
    match data.try_borrow() {
        Ok(v)  => println!("Berjaya borrow: {:?}", *v),
        Err(e) => println!("Gagal borrow: {}", e),
    }
}
```

---

## RefCell Panic pada Runtime

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(5);

    let _borrow1 = data.borrow();
    let _borrow2 = data.borrow();       // OK — dua immutable OK
    // let _tulis = data.borrow_mut();  // ← PANIC! sudah ada borrow aktif

    println!("Dua borrow OK: {}", *_borrow1);
}
```

```
RefCell runtime borrow rules:
  ┌──────────────────────────────────────────────────┐
  │  State Semasa   │  borrow()  │  borrow_mut()     │
  ├──────────────────────────────────────────────────┤
  │  Tiada borrow   │    OK ✔    │    OK ✔           │
  │  Ada borrow     │    OK ✔    │    PANIC ✘        │
  │  Ada borrow_mut │    PANIC ✘ │    PANIC ✘        │
  └──────────────────────────────────────────────────┘
```

---

## RefCell dalam Struct — Pattern Biasa

```rust
use std::cell::RefCell;

#[derive(Debug)]
struct Cache {
    data:        RefCell<Option<String>>, // boleh ubah walaupun &self
    hits:        RefCell<u32>,
}

impl Cache {
    fn baru() -> Self {
        Cache {
            data: RefCell::new(None),
            hits: RefCell::new(0),
        }
    }

    // &self — immutable receiver, tapi boleh ubah dalaman!
    fn ambil(&self) -> String {
        // Periksa cache
        if let Some(ref cached) = *self.data.borrow() {
            *self.hits.borrow_mut() += 1;
            return cached.clone();
        }

        // Cache miss — kira nilai
        let nilai = "data mahal yang dikira...".to_string();
        *self.data.borrow_mut() = Some(nilai.clone());
        nilai
    }

    fn hits(&self) -> u32 {
        *self.hits.borrow()
    }
}

fn main() {
    let cache = Cache::baru(); // tiada mut!

    println!("{}", cache.ambil()); // kira
    println!("{}", cache.ambil()); // dari cache
    println!("{}", cache.ambil()); // dari cache
    println!("Hits: {}", cache.hits()); // 2
}
```

---

# BAB 5: Cell<T> — Simple Interior Mutability 🔋

## Cell vs RefCell

```
Cell<T>:
  ✔ Lebih cepat (tiada borrow tracking)
  ✔ Selamat — tidak ada borrow, hanya get/set
  ✘ Hanya untuk T yang implement Copy
  ✘ Tidak boleh dapat reference ke data dalam Cell

RefCell<T>:
  ✔ Untuk semua T (termasuk non-Copy)
  ✔ Boleh dapat reference ke data dalam
  ✘ Ada borrow tracking overhead
  ✘ Boleh panic jika salah guna
```

```rust
use std::cell::Cell;

fn main() {
    let nilai = Cell::new(42);

    // get — dapat salinan (Copy sahaja)
    println!("Nilai: {}", nilai.get()); // 42

    // set — tukar nilai
    nilai.set(100);
    println!("Nilai baru: {}", nilai.get()); // 100

    // update — ubah berdasarkan nilai lama
    nilai.update(|x| x + 1);
    println!("Selepas +1: {}", nilai.get()); // 101

    // Dalam struct — field boleh "mutable" tanpa mut binding
    struct Kiraan {
        nilai: Cell<u32>,
        nama:  String,
    }

    let k = Kiraan { nilai: Cell::new(0), nama: "test".into() };
    k.nilai.set(k.nilai.get() + 1); // update tanpa `mut k`!
    k.nilai.set(k.nilai.get() + 1);
    println!("Kiraan '{}': {}", k.nama, k.nilai.get()); // 2
}
```

---

## Perbandingan Cell, RefCell, Mutex

```
┌────────────────┬──────────────┬────────────────┬─────────────────┐
│                │  Cell<T>     │  RefCell<T>    │  Mutex<T>       │
├────────────────┼──────────────┼────────────────┼─────────────────┤
│  Thread safe   │  ✘ No        │  ✘ No          │  ✔ Yes          │
│  Type req.     │  Copy        │  Any           │  Send           │
│  Borrow check  │  None        │  Runtime       │  Lock (runtime) │
│  Overhead      │  Sangat kecil│  Kecil         │  Sederhana      │
│  Panic?        │  Tidak       │  Boleh         │  Boleh (poison) │
│  Get reference │  ✘ Tidak     │  ✔ Ya          │  ✔ Ya (guard)   │
└────────────────┴──────────────┴────────────────┴─────────────────┘
```

---

# BAB 6: Weak<T> — Elak Reference Cycle 🔗

## Masalah: Reference Cycle dengan Rc

```
Reference Cycle — memory leak!

  ┌──────────────────┐     ┌──────────────────┐
  │  Rc A            │────▶│  Rc B            │
  │  count = 1       │     │  count = 1       │
  │  data: ...       │     │  data: ...       │
  │  next → B ───────┼────▶│  next → A ───────┼──┐
  └──────────────────┘     └──────────────────┘  │
         ▲                                        │
         └────────────────────────────────────────┘

  Drop A: count A: 1→0? Tidak! B masih pegang Rc ke A
  Drop B: count B: 1→0? Tidak! A masih pegang Rc ke B
  MEMORY LEAK! Kedua-dua tidak pernah di-drop.
```

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

#[derive(Debug)]
struct Nod {
    nilai:  i32,
    parent: RefCell<Weak<Nod>>,  // Weak — tidak tambah count!
    anak:   RefCell<Vec<Rc<Nod>>>,
}

impl Nod {
    fn baru(nilai: i32) -> Rc<Self> {
        Rc::new(Nod {
            nilai,
            parent: RefCell::new(Weak::new()),
            anak:   RefCell::new(vec![]),
        })
    }
}

fn main() {
    let ibu = Nod::baru(10);
    let anak = Nod::baru(20);

    // Sambung anak ke ibu
    anak.parent.borrow_mut().replace(Rc::downgrade(&ibu)); // Weak!
    ibu.anak.borrow_mut().push(Rc::clone(&anak));

    println!("Ibu strong count: {}", Rc::strong_count(&ibu)); // 1
    println!("Anak strong count: {}", Rc::strong_count(&anak)); // 2

    // Weak — perlu upgrade untuk guna
    if let Some(p) = anak.parent.borrow().upgrade() {
        println!("Parent nilai: {}", p.nilai); // 10
    }

    drop(ibu);
    // Selepas drop ibu, Weak ke ibu jadi None
    println!("Parent masih ada? {}",
        anak.parent.borrow().upgrade().is_some()); // false!
}
```

---

## Strong vs Weak Count

```rust
use std::rc::{Rc, Weak};

fn main() {
    let strong = Rc::new("data penting");

    let weak1: Weak<&str> = Rc::downgrade(&strong); // tidak tambah strong count
    let weak2: Weak<&str> = Rc::downgrade(&strong);

    println!("Strong: {}", Rc::strong_count(&strong)); // 1
    println!("Weak:   {}", Rc::weak_count(&strong));   // 2

    // Upgrade weak → Option<Rc<T>>
    println!("weak1 valid: {}", weak1.upgrade().is_some()); // true

    drop(strong); // strong count → 0, data dibersihkan!

    // Selepas strong drop, weak tidak valid lagi
    println!("weak1 valid: {}", weak1.upgrade().is_some()); // false!
    println!("weak2 valid: {}", weak2.upgrade().is_some()); // false!
}
```

---

## 🧠 Brain Teaser #3

Apakah masalah kod ini dan bagaimana perbaiki?

```rust
use std::rc::Rc;
use std::cell::RefCell;

struct Nod {
    nilai: i32,
    seterusnya: Option<Rc<RefCell<Nod>>>,
}

fn main() {
    let a = Rc::new(RefCell::new(Nod { nilai: 1, seterusnya: None }));
    let b = Rc::new(RefCell::new(Nod { nilai: 2, seterusnya: None }));

    // Buat cycle!
    a.borrow_mut().seterusnya = Some(Rc::clone(&b));
    b.borrow_mut().seterusnya = Some(Rc::clone(&a)); // ← masalah!
}
```

<details>
<summary>👀 Jawapan</summary>

**Masalah:** Reference cycle! `a` → `b` → `a` → ... Bila `a` dan `b` drop, strong count tidak pernah jadi 0 → **memory leak**.

**Penyelesaian:** Guna `Weak<T>` untuk pointer yang "kembali":

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Nod {
    nilai: i32,
    seterusnya: Option<Weak<RefCell<Nod>>>, // Weak, bukan Rc!
}

fn main() {
    let a = Rc::new(RefCell::new(Nod { nilai: 1, seterusnya: None }));
    let b = Rc::new(RefCell::new(Nod { nilai: 2, seterusnya: None }));

    a.borrow_mut().seterusnya = Some(Rc::downgrade(&b));
    b.borrow_mut().seterusnya = Some(Rc::downgrade(&a)); // Weak!

    // Tiada cycle! Drop berfungsi dengan betul.
    println!("a count: {}", Rc::strong_count(&a)); // 1
    println!("b count: {}", Rc::strong_count(&b)); // 1
}
```
</details>

---

# BAB 7: Gabungan — Pattern Berkuasa 💪

## Rc<RefCell<T>> — Share + Mutate (Single Thread)

```
Rc<T>       → share ownership, immutable
RefCell<T>  → interior mutability, runtime borrow check
Rc<RefCell<T>> → share + mutate dalam single thread!
```

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
struct Pelajar {
    nama:   String,
    markah: f64,
}

fn main() {
    // Senarai yang dikongsi dan boleh diubah
    let senarai: Rc<RefCell<Vec<Pelajar>>> = Rc::new(RefCell::new(vec![]));

    let senarai_a = Rc::clone(&senarai); // bahagian A guna
    let senarai_b = Rc::clone(&senarai); // bahagian B guna

    // Bahagian A tambah pelajar
    senarai_a.borrow_mut().push(Pelajar { nama: "Ali".into(),  markah: 85.0 });
    senarai_a.borrow_mut().push(Pelajar { nama: "Siti".into(), markah: 92.0 });

    // Bahagian B baca dan ubah
    {
        let mut guard = senarai_b.borrow_mut();
        for p in guard.iter_mut() {
            p.markah += 5.0; // bonus 5 markah
        }
    }

    // Semua perubahan nampak melalui mana-mana clone
    for p in senarai.borrow().iter() {
        println!("{}: {}", p.nama, p.markah);
    }
    // Ali: 90, Siti: 97
}
```

---

## Arc<Mutex<T>> — Share + Mutate (Multi Thread)

```
Arc<T>     → share ownership merentasi threads
Mutex<T>   → thread-safe mutability
Arc<Mutex<T>> → share + mutate merentasi threads!
```

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::collections::HashMap;

fn main() {
    let direktori: Arc<Mutex<HashMap<String, u32>>> =
        Arc::new(Mutex::new(HashMap::new()));

    let mut handles = vec![];

    // 5 thread, setiap satu tambah beberapa entries
    for i in 0..5 {
        let dir = Arc::clone(&direktori);
        handles.push(thread::spawn(move || {
            let mut guard = dir.lock().unwrap();
            guard.insert(format!("thread_{}_a", i), i * 10);
            guard.insert(format!("thread_{}_b", i), i * 20);
        }));
    }

    for h in handles { h.join().unwrap(); }

    let final_dir = direktori.lock().unwrap();
    println!("Jumlah entries: {}", final_dir.len()); // 10

    let mut keys: Vec<&String> = final_dir.keys().collect();
    keys.sort();
    for k in keys { println!("  {} = {}", k, final_dir[k]); }
}
```

---

## Ringkasan Kombinasi

```
Single-thread, immutable share:   Rc<T>
Single-thread, mutable share:     Rc<RefCell<T>>
Multi-thread, immutable share:    Arc<T>
Multi-thread, mutable share:      Arc<Mutex<T>>
Multi-thread, many read/one write: Arc<RwLock<T>>
Pointer ke parent (tree/graph):   Weak<T> atau Arc::downgrade
```

---

# BAB 8: Cow<T> — Clone on Write 🐄

## Apa Itu Cow?

```
Cow = Clone on Write

  Masalah: Fungsi terima &str, mungkin perlu modify, mungkin tidak.
  Kalau modify → perlu allocate String baru (kos tinggi!)
  Kalau tidak modify → buang-buang clone String

  Cow<str> = lazily clone sahaja bila perlu!

  Cow::Borrowed(&str)  → guna reference, tiada allocation
  Cow::Owned(String)   → guna owned String, ada allocation
```

```rust
use std::borrow::Cow;

fn elak_pemindahan_huruf(s: &str) -> Cow<str> {
    if s.chars().any(|c| !c.is_alphanumeric() && c != ' ') {
        // Perlu ubah — kena clone
        let bersih: String = s.chars()
            .filter(|c| c.is_alphanumeric() || *c == ' ')
            .collect();
        Cow::Owned(bersih) // allocation berlaku di sini
    } else {
        // Tiada perlu ubah — guna reference terus!
        Cow::Borrowed(s)   // tiada allocation!
    }
}

fn main() {
    let biasa  = "hello world";
    let kotor  = "hello, world! (rust)";

    let r1 = elak_pemindahan_huruf(biasa);
    let r2 = elak_pemindahan_huruf(kotor);

    println!("{}", r1); // "hello world"  — Borrowed (no alloc)
    println!("{}", r2); // "hello world rust" — Owned (alloc sekali sahaja)

    // Semak jenis
    println!("r1 owned? {}", matches!(r1, Cow::Owned(_)));    // false
    println!("r2 owned? {}", matches!(r2, Cow::Owned(_)));    // true

    // Cow dalam function signature — terima &str atau String
    fn papar(s: Cow<str>) {
        println!("Papar: {}", s);
    }

    papar(Cow::Borrowed("dari &str"));           // tiada clone
    papar(Cow::Owned(String::from("dari String"))); // owned
    papar("literal".into());                     // auto-convert
}
```

---

## Cow dalam Struct

```rust
use std::borrow::Cow;

// Struct yang efisien — data boleh borrowed atau owned
#[derive(Debug)]
struct Konfigurasi<'a> {
    nama:    Cow<'a, str>,
    versi:   Cow<'a, str>,
    port:    u16,
}

impl<'a> Konfigurasi<'a> {
    fn dari_env() -> Self {
        Konfigurasi {
            nama:  Cow::Borrowed("MyApp"),   // string literal — no alloc!
            versi: Cow::Borrowed("1.0.0"),   // string literal — no alloc!
            port:  8080,
        }
    }

    fn dengan_nama_custom(nama: String) -> Self {
        Konfigurasi {
            nama:  Cow::Owned(nama),         // owned String
            versi: Cow::Borrowed("1.0.0"),
            port:  8080,
        }
    }
}

fn main() {
    let cfg1 = Konfigurasi::dari_env();
    let cfg2 = Konfigurasi::dengan_nama_custom("CustomApp".to_string());

    println!("{:?}", cfg1);
    println!("{:?}", cfg2);
}
```

---

## 🧠 Brain Teaser #4

Bila nak guna `Cow<str>` berbanding `String` atau `&str`?

<details>
<summary>👀 Jawapan</summary>

```
&str          → Bila sentiasa borrowed, tiada perlu own
String        → Bila sentiasa owned, atau perlu modify
Cow<str>      → Bila KADANG-KADANG perlu modify, kadang tidak

Contoh penggunaan Cow yang baik:

1. Fungsi sanitize/escape:
   → Input mungkin OK (return &str terus)
   → Input mungkin ada karakter khas (return String baru)

2. Config yang biasanya static tapi kadang dynamic:
   → Default value: Cow::Borrowed("default")
   → Custom value:  Cow::Owned(nilai_dari_env)

3. Serialization/Deserialization:
   → serde guna Cow<str> untuk elak allocation bila tidak perlu

Peraturan mudah:
  "Aku perlu return string, tapi tak tahu sama ada
   perlu allocate atau tidak" → Cow<str>
```
</details>

---

# BAB 9: Custom Smart Pointer 🔨

## Implement Deref

```rust
use std::ops::Deref;

struct KotakKu<T>(T);

impl<T> KotakKu<T> {
    fn baru(x: T) -> Self {
        KotakKu(x)
    }
}

impl<T> Deref for KotakKu<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.0 // return reference ke data dalam
    }
}

fn main() {
    let k = KotakKu::baru(5);

    // Boleh dereference macam pointer biasa
    println!("{}", *k);  // 5

    // Deref coercion — &KotakKu<String> → &String → &str
    let s = KotakKu::baru(String::from("hello"));
    println!("Len: {}", s.len()); // method String boleh guna terus!

    // Fungsi yang terima &str boleh terima &KotakKu<String>
    fn cetak_str(s: &str) { println!("{}", s); }
    cetak_str(&s); // deref coercion berlaku: &KotakKu<String> → &str
}
```

---

## Implement Drop

```rust
struct ResourcePenting {
    nama: String,
}

impl ResourcePenting {
    fn baru(nama: &str) -> Self {
        println!("  [+] Buat resource: {}", nama);
        ResourcePenting { nama: nama.to_string() }
    }
}

impl Drop for ResourcePenting {
    fn drop(&mut self) {
        // Dipanggil auto bila keluar scope
        println!("  [-] Bebaskan resource: {}", self.nama);
        // Tutup fail, lepas lock, cleanup, dll.
    }
}

fn main() {
    println!("Mula");
    let r1 = ResourcePenting::baru("Database Connection");
    {
        let r2 = ResourcePenting::baru("File Handle");
        let r3 = ResourcePenting::baru("Network Socket");
        println!("Dalam scope dalam");
    } // r3 drop dulu (LIFO!), kemudian r2
    println!("Keluar scope dalam");

    // drop(r1) awal — boleh drop manual
    drop(r1);
    println!("Selepas drop manual r1");
    println!("Tamat");
}
```

**Output:**
```
Mula
  [+] Buat resource: Database Connection
  [+] Buat resource: File Handle
  [+] Buat resource: Network Socket
Dalam scope dalam
  [-] Bebaskan resource: Network Socket  ← LIFO
  [-] Bebaskan resource: File Handle
Keluar scope dalam
  [-] Bebaskan resource: Database Connection ← drop manual
Selepas drop manual r1
Tamat
```

---

## Smart Pointer Ringkas — Scoped Timer

```rust
use std::time::Instant;

struct ScopedTimer {
    label: String,
    mula:  Instant,
}

impl ScopedTimer {
    fn baru(label: &str) -> Self {
        println!("⏱  [{}] mula", label);
        ScopedTimer {
            label: label.to_string(),
            mula:  Instant::now(),
        }
    }
}

impl Drop for ScopedTimer {
    fn drop(&mut self) {
        println!("⏱  [{}] tamat: {}μs",
            self.label,
            self.mula.elapsed().as_micros());
    }
}

fn operasi_mahal() {
    let _timer = ScopedTimer::baru("operasi_mahal");
    // Simulate kerja
    std::thread::sleep(std::time::Duration::from_millis(10));
    // _timer auto-drop bila fungsi tamat → cetak masa
}

fn main() {
    operasi_mahal();
    {
        let _t = ScopedTimer::baru("scope kecil");
        let _x: Vec<i32> = (0..1000).collect();
    }
}
```

---

# BAB 10: Mini Project — Graf Direkted 🕸️

> Projek ini gabungkan `Rc`, `RefCell`, `Weak`, dan custom display.

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;
use std::collections::HashMap;
use std::fmt;

// ─── Node Graf ────────────────────────────────────────────────

#[derive(Debug)]
struct Nod {
    id:      u32,
    label:   String,
    tepi:    RefCell<Vec<Tepi>>,  // boleh tambah tepi tanpa &mut
}

#[derive(Debug, Clone)]
struct Tepi {
    ke:     Weak<Nod>,   // Weak — elak cycle!
    berat:  f64,
}

impl Nod {
    fn baru(id: u32, label: &str) -> Rc<Self> {
        Rc::new(Nod {
            id,
            label:  label.to_string(),
            tepi:   RefCell::new(vec![]),
        })
    }

    fn sambung(&self, ke: &Rc<Nod>, berat: f64) {
        self.tepi.borrow_mut().push(Tepi {
            ke:    Rc::downgrade(ke), // tukar ke Weak
            berat,
        });
    }

    fn jiran(&self) -> Vec<(String, f64)> {
        self.tepi.borrow().iter()
            .filter_map(|t| {
                t.ke.upgrade().map(|nod| (nod.label.clone(), t.berat))
            })
            .collect()
    }
}

// ─── Graf ─────────────────────────────────────────────────────

struct Graf {
    nod: HashMap<u32, Rc<Nod>>,
    id_seterusnya: u32,
}

impl Graf {
    fn baru() -> Self {
        Graf { nod: HashMap::new(), id_seterusnya: 1 }
    }

    fn tambah_nod(&mut self, label: &str) -> u32 {
        let id = self.id_seterusnya;
        self.id_seterusnya += 1;
        self.nod.insert(id, Nod::baru(id, label));
        println!("  + Nod [{}] '{}'", id, label);
        id
    }

    fn sambung(&self, dari_id: u32, ke_id: u32, berat: f64) {
        if let (Some(dari), Some(ke)) = (self.nod.get(&dari_id), self.nod.get(&ke_id)) {
            dari.sambung(ke, berat);
            println!("  → Tepi [{}]──{:.1}──▶[{}]",
                dari.label, berat, ke.label);
        }
    }

    fn cari_nod(&self, id: u32) -> Option<&Rc<Nod>> {
        self.nod.get(&id)
    }

    // BFS — cari laluan terpendek (bilangan tepi)
    fn bfs(&self, mula_id: u32, tamat_id: u32) -> Option<Vec<String>> {
        use std::collections::VecDeque;

        let mut antrian: VecDeque<(u32, Vec<String>)> = VecDeque::new();
        let mut dilawat = std::collections::HashSet::new();

        if let Some(nod) = self.nod.get(&mula_id) {
            antrian.push_back((mula_id, vec![nod.label.clone()]));
            dilawat.insert(mula_id);
        }

        while let Some((semasa_id, laluan)) = antrian.pop_front() {
            if semasa_id == tamat_id {
                return Some(laluan);
            }

            if let Some(nod) = self.nod.get(&semasa_id) {
                for tepi in nod.tepi.borrow().iter() {
                    if let Some(jiran) = tepi.ke.upgrade() {
                        if !dilawat.contains(&jiran.id) {
                            dilawat.insert(jiran.id);
                            let mut laluan_baru = laluan.clone();
                            laluan_baru.push(jiran.label.clone());
                            antrian.push_back((jiran.id, laluan_baru));
                        }
                    }
                }
            }
        }
        None
    }

    fn papar(&self) {
        println!("\n{'─'*40}");
        println!("Graf ({} nod):", self.nod.len());
        let mut ids: Vec<u32> = self.nod.keys().copied().collect();
        ids.sort();
        for id in ids {
            if let Some(nod) = self.nod.get(&id) {
                let jiran = nod.jiran();
                if jiran.is_empty() {
                    println!("  [{}] {} → (tiada tepi keluar)", id, nod.label);
                } else {
                    print!("  [{}] {} → ", id, nod.label);
                    for (label, berat) in &jiran {
                        print!("{}({:.1}) ", label, berat);
                    }
                    println!();
                }
            }
        }
        println!("{'─'*40}");
    }
}

// ─── Main ──────────────────────────────────────────────────────

fn main() {
    let mut g = Graf::baru();

    println!("=== Bina Graf ===");
    let kl  = g.tambah_nod("KL");
    let pg  = g.tambah_nod("Penang");
    let jb  = g.tambah_nod("JB");
    let kch = g.tambah_nod("Kuching");
    let sb  = g.tambah_nod("Sabah");
    let iph = g.tambah_nod("Ipoh");

    println!("\n=== Tambah Tepi (jarak km) ===");
    g.sambung(kl,  pg,  330.0);
    g.sambung(kl,  jb,  330.0);
    g.sambung(kl,  iph, 200.0);
    g.sambung(pg,  kl,  330.0);
    g.sambung(pg,  iph, 160.0);
    g.sambung(iph, kl,  200.0);
    g.sambung(iph, pg,  160.0);
    g.sambung(jb,  kl,  330.0);
    g.sambung(kch, sb,  900.0); // Malaysia Timur — tiada sambungan terus ke Semenanjung

    g.papar();

    println!("\n=== Cari Laluan (BFS) ===");
    for (dari, ke) in [(kl, jb), (pg, jb), (kch, pg), (kl, sb)] {
        let nama_dari = &g.nod[&dari].label;
        let nama_ke   = &g.nod[&ke].label;
        match g.bfs(dari, ke) {
            Some(laluan) => println!("  {} → {}: {:?}", nama_dari, nama_ke, laluan),
            None         => println!("  {} → {}: Tiada laluan!", nama_dari, nama_ke),
        }
    }

    println!("\n=== Semak Weak Reference ===");
    let kl_nod = g.cari_nod(kl).unwrap();
    let weak_kl: Weak<Nod> = Rc::downgrade(kl_nod);
    println!("KL strong count: {}", Rc::strong_count(kl_nod));
    println!("KL weak   count: {}", Rc::weak_count(kl_nod));
    println!("Weak valid: {}", weak_kl.upgrade().is_some()); // true

    // Drop seluruh graf
    drop(g);
    println!("Selepas drop graf:");
    println!("Weak valid: {}", weak_kl.upgrade().is_some()); // false!
}
```

---

# 📋 Rujukan Pantas — Smart Pointers Cheat Sheet

## Pilih Smart Pointer Yang Tepat

```
Soal diri anda:

Nak heap allocation / recursive type?
  └── Box<T>

Nak banyak owner?
  ├── Single-thread? → Rc<T>
  └── Multi-thread?  → Arc<T>

Nak mutate melalui immutable reference?
  ├── Copy type, simple?       → Cell<T>
  ├── Non-copy / perlu &mut?   → RefCell<T>    (single-thread)
  └── Thread-safe mutable?     → Mutex<T>      (multi-thread)

Nak elak reference cycle?
  └── Weak<T>  (atau Arc::downgrade untuk Arc)

Nak clone hanya bila perlu?
  └── Cow<T>

Nak shared + mutable:
  ├── Single-thread: Rc<RefCell<T>>
  └── Multi-thread:  Arc<Mutex<T>>
```

## Ringkasan Big-O & Overhead

```
┌──────────────────┬─────────────────────────────────────────┐
│  Smart Pointer   │  Overhead                               │
├──────────────────┼─────────────────────────────────────────┤
│  Box<T>          │  Satu pointer (8 bytes)                 │
│  Rc<T>           │  Dua counter (strong + weak, 16 bytes)  │
│  Arc<T>          │  Dua atomic counter (16 bytes)          │
│  RefCell<T>      │  Satu borrow counter                    │
│  Cell<T>         │  Hampir tiada (wrapper sahaja)          │
│  Weak<T>         │  Pointer + tiada ownership              │
│  Mutex<T>        │  OS mutex + poison flag                 │
│  Cow<T>          │  Enum overhead (Borrowed/Owned)         │
└──────────────────┴─────────────────────────────────────────┘
```

## Kaedah Penting

```rust
// Box
Box::new(val)              // allocate dalam heap
*boxed                     // dereference / move keluar

// Rc / Arc
Rc::new(val)               // buat smart pointer
Rc::clone(&ptr)            // tambah strong count
Rc::strong_count(&ptr)     // berapa strong reference
Rc::weak_count(&ptr)       // berapa weak reference
Rc::downgrade(&ptr)        // dapat Weak<T>
Rc::ptr_eq(&a, &b)         // adakah tunjuk ke sama?

// Weak
weak.upgrade()             // Option<Rc<T>> — None jika dah drop

// RefCell
refcell.borrow()           // Ref<T> — immutable
refcell.borrow_mut()       // RefMut<T> — mutable
refcell.try_borrow()       // Result<Ref<T>, _>
refcell.try_borrow_mut()   // Result<RefMut<T>, _>

// Cell
cell.get()                 // dapat salinan (Copy sahaja)
cell.set(val)              // tukar nilai
cell.update(|x| x + 1)    // ubah berdasarkan nilai lama

// Cow
Cow::Borrowed(&val)        // dari reference
Cow::Owned(val)            // dari owned value
cow.into_owned()           // dapat owned value (clone jika perlu)
cow.to_mut()               // &mut T (clone jika perlu)
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **Observer Pattern** — `Rc<RefCell<>>` untuk subject dan observers
2. **Thread Pool** — `Arc<Mutex<VecDeque<Job>>>` sebagai work queue
3. **Cache LRU** — `HashMap` + `Weak` references untuk nilai yang boleh expire
4. **DOM Tree** — parent-child dengan `Rc` (parent → child) dan `Weak` (child → parent)
5. **Plugin System** — `Box<dyn Trait>` untuk pelbagai jenis plugin

---

*Smart Pointers in Rust — dari `Box<T>` hingga `Rc<RefCell<Weak<T>>>`. Selamat belajar!* 🦀
