# ⚠️ Unsafe Rust — Panduan Lengkap

> "Dengan kuasa besar datang tanggungjawab besar."
> Faham bila, kenapa, dan bagaimana guna unsafe dengan betul.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu Unsafe Rust?

```
Safe Rust:
  Compiler jaga semua memory safety.
  Borrow checker, ownership, lifetimes.
  Mustahil untuk ada dangling pointer, data race, dll.
  → Anda hampir tidak boleh buat kesilapan memori.

Unsafe Rust:
  Anda beritahu compiler "Saya yang jamin keselamatan ini."
  Compiler masih check semua safe rules DI LUAR blok unsafe.
  Hanya dalam blok unsafe, beberapa peraturan dilonggarkan.
  → Anda BERTANGGUNGJAWAB untuk keselamatan.

PENTING UNTUK FAHAM:
  unsafe TIDAK mematikan semua safety checks!
  unsafe TIDAK bermaksud "kod berbahaya."
  unsafe bermaksud "keselamatan yang compiler tidak boleh verify
                    secara automatik — programmer yang verify."
```

---

## Lima Perkara Yang Boleh Buat dalam unsafe

```
Di luar blok unsafe — SEMUA ini adalah ERROR:

  1. Dereference raw pointer         (*const T atau *mut T)
  2. Panggil unsafe function/method
  3. Akses/modifikasi static mutable variable
  4. Implement unsafe trait
  5. Akses fields of unions

HANYA dalam blok unsafe boleh buat perkara di atas.
Semua yang lain (borrow checker, type system) MASIH berfungsi!
```

---

## Peta Pembelajaran

```
Bab 1  → Kenapa unsafe Wujud?
Bab 2  → Raw Pointers — *const T dan *mut T
Bab 3  → Unsafe Functions & Methods
Bab 4  → Static Mutable Variables
Bab 5  → Unsafe Traits
Bab 6  → FFI — Foreign Function Interface
Bab 7  → Union Types
Bab 8  → Common Unsafe Patterns
Bab 9  → Safety Abstraction — Wrap unsafe dengan Safe API
Bab 10 → Mini Project: Safe Wrapper untuk C Library
```

---

# BAB 1: Kenapa unsafe Wujud? 🤔

## Ada Perkara Yang Compiler Tidak Boleh Verify

```rust
// Senario: Anda tahu split string adalah valid,
// tapi compiler tidak boleh verify ini semasa compile.

fn split_string_ditengah(s: &str) -> (&str, &str) {
    let mid = s.len() / 2;

    // Kalau mid tidak pada char boundary, ini akan panic!
    // Compiler tidak boleh tahu ini safe semasa compile.
    // Kita yang mesti verify.

    (&s[..mid], &s[mid..])
}

// Contoh lain yang memerlukan unsafe:
//   - Panggil C library (FFI)
//   - Implement low-level data structures (Vec, LinkedList)
//   - Hardware access
//   - Shared memory antara thread (dalam cara tertentu)
//   - SIMD instructions
//   - Interop dengan OS
```

---

## Safe Rust Ada Limitasi Sebenar

```rust
// Ini TIDAK BOLEH dibuat dalam safe Rust:

// 1. Split mut slice menjadi dua bahagian serentak
fn split_mut(slice: &mut [i32]) -> (&mut [i32], &mut [i32]) {
    let mid = slice.len() / 2;
    // safe Rust: tidak boleh ada dua &mut ke data yang sama
    // tapi kita tahu kedua-dua bahagian TIDAK OVERLAP!
    // Compiler tidak boleh verify ini.
    let ptr = slice.as_mut_ptr();
    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), slice.len() - mid),
        )
    }
    // Standard library ada split_at_mut() yang buat ini!
}

// 2. Tukar type tanpa conversion (transmute)
// 3. Buat object dari pointer mentah (FFI)
// 4. Intrinsics dan SIMD
// 5. Atomic operations yang complex
```

---

## unsafe Dalam Standard Library

```rust
// Standard library sendiri PENUH dengan unsafe!
// Tapi ia wrapped dalam safe API yang kita guna setiap hari.

// Vec::push() menggunakan unsafe internally:
// pub fn push(&mut self, value: T) {
//     ...
//     unsafe {
//         let end = self.as_mut_ptr().add(self.len);
//         ptr::write(end, value);
//         ...
//     }
// }

// String::from_utf8_unchecked():
// Caller mesti jamin bytes adalah valid UTF-8!
// Kalau tidak — undefined behavior!

fn contoh_stdlib_unsafe() {
    // Ini safe — Vec manage memory dengan selamat menggunakan unsafe
    let mut v = Vec::new();
    v.push(1);
    v.push(2);

    // Ini unsafe — caller jamin slice adalah valid UTF-8
    let bytes = vec![104, 101, 108, 108, 111]; // "hello"
    let s = unsafe { String::from_utf8_unchecked(bytes) };
    println!("{}", s); // hello
}
```

---

## 🧠 Brain Teaser #1

Apakah yang sebenarnya berlaku apabila kita tulis `unsafe { ... }`?

```rust
unsafe fn berbahaya() -> i32 {
    42
}

fn main() {
    // Kenapa perlu unsafe block di sini?
    let hasil = unsafe { berbahaya() };
    println!("{}", hasil);
}
```

<details>
<summary>👀 Jawapan</summary>

`unsafe { ... }` adalah **kontrak antara anda dan compiler**:

- Compiler berkata: "Saya tidak boleh verify ini selamat secara automatik."
- Anda berkata: "Saya yang verify. Saya jamin ini selamat."
- Compiler berkata: "OK, saya percaya anda."

Apabila ada bug dalam unsafe code → **anda yang bertanggungjawab**, bukan compiler.

`unsafe { berbahaya() }` diperlukan kerana:
1. `berbahaya()` ditandakan `unsafe fn`
2. Ini memberitahu programmer yang memanggil: "Fungsi ini ada syarat yang mesti dipenuhi sebelum dipanggil"
3. `unsafe { }` = "Saya telah baca kontrak dan saya jamin syarat dipenuhi"

**Insight penting:** unsafe tidak menjadikan sesuatu berbahaya. Ia adalah **label** yang memberitahu: "Bahagian ini memerlukan analisis manual untuk keselamatan."
</details>

---

# BAB 2: Raw Pointers — *const T dan *mut T 🎯

## Apa Itu Raw Pointer?

```
Safe Rust references:
  &T     → dijamin valid, aligned, tidak null
  &mut T → dijamin valid, aligned, tidak null, exclusive

Raw pointers (unsafe):
  *const T → boleh null, boleh dangling, boleh unaligned
  *mut T   → boleh null, boleh dangling, boleh unaligned, boleh alias

Raw pointer vs Reference:
  Reference: compiler bagi jaminan
  Raw pointer: anda bagi jaminan

Buat raw pointer = SAFE (tiada bahaya)
Dereference raw pointer = UNSAFE (di sinilah bahaya)
```

```rust
fn main() {
    let x = 5;
    let y = 10;

    // Buat raw pointer — ini SAFE!
    let ptr_x: *const i32 = &x as *const i32;
    let ptr_y: *mut i32   = &y as *const i32 as *mut i32;

    // Cara lain buat raw pointer
    let ptr2 = std::ptr::addr_of!(x); // *const i32
    let val: i32 = 42;
    let ptr3 = &val as *const i32;

    // Dereference raw pointer — perlu unsafe!
    unsafe {
        println!("ptr_x → {}", *ptr_x); // 5
        println!("ptr_y → {}", *ptr_y); // 10

        // Raw pointer boleh null (berbeza dari reference!)
        let null_ptr: *const i32 = std::ptr::null();
        println!("null ptr is null: {}", null_ptr.is_null()); // true

        // Jangan dereference null pointer!
        // *null_ptr  ← UNDEFINED BEHAVIOR! Program crash!
    }

    // Boleh buat raw pointer dari arbitrary address
    let addr = 0x1234usize;
    let ptr_arbitrary = addr as *const i32;
    // Dereference ini = undefined behavior (alamat tidak valid)!
    println!("Arbitrary ptr: {:p}", ptr_arbitrary);
}
```

---

## Operasi Raw Pointer

```rust
fn main() {
    let mut data = [10i32, 20, 30, 40, 50];
    let ptr = data.as_mut_ptr(); // *mut i32

    unsafe {
        // Baca nilai
        println!("data[0] = {}", *ptr);          // 10
        println!("data[1] = {}", *ptr.add(1));   // 20 (pointer arithmetic)
        println!("data[2] = {}", *ptr.offset(2)); // 30

        // Tulis nilai
        *ptr = 100;
        *ptr.add(1) = 200;
        ptr.add(2).write(300); // cara lain untuk write

        // Salin data
        let src = [1i32, 2, 3];
        let dst = data.as_mut_ptr().add(2);
        std::ptr::copy_nonoverlapping(src.as_ptr(), dst, 3);
        // data sekarang: [100, 200, 1, 2, 3]
    }

    println!("{:?}", data); // [100, 200, 1, 2, 3]

    // Pointer arithmetic
    let arr = [1.0f64, 2.0, 3.0, 4.0, 5.0];
    let base: *const f64 = arr.as_ptr();

    unsafe {
        for i in 0..arr.len() {
            println!("arr[{}] = {}", i, *base.add(i));
        }
    }
}
```

---

## Bila Raw Pointer Berguna

```rust
use std::mem;

// 1. Interop dengan C code (FFI)
extern "C" {
    fn memcpy(dst: *mut u8, src: *const u8, n: usize) -> *mut u8;
}

// 2. Implement data structures yang complex
// (contoh: doubly-linked list, memerlukan shared mutable refs)

// 3. Avoid overhead dalam hot path
fn jumlah_cepat(data: &[i32]) -> i64 {
    let mut jumlah: i64 = 0;
    let mut ptr = data.as_ptr();
    let end = unsafe { ptr.add(data.len()) };

    // Loop pointer arithmetic — kadang lebih laju dari bounds-checked indexing
    unsafe {
        while ptr < end {
            jumlah += *ptr as i64;
            ptr = ptr.add(1);
        }
    }
    jumlah
}

// 4. Transmute — tukar type representation
fn f32_bits(f: f32) -> u32 {
    unsafe { mem::transmute(f) }
    // Sama seperti: f.to_bits() tapi lebih langsung
}

fn main() {
    let data = vec![1, 2, 3, 4, 5];
    println!("Jumlah: {}", jumlah_cepat(&data)); // 15

    let pi = std::f32::consts::PI;
    println!("PI bits: {:#010x}", f32_bits(pi)); // 0x40490fdb
}
```

---

# BAB 3: Unsafe Functions & Methods 🔧

## Declare dan Panggil Unsafe Function

```rust
// Fungsi yang memerlukan caller memenuhi syarat tertentu
unsafe fn tulis_ke_pointer(ptr: *mut i32, nilai: i32) {
    // UNSAFE: ptr mesti:
    //   - Tidak null
    //   - Valid dan aligned untuk i32
    //   - Tidak alias dengan reference aktif lain
    *ptr = nilai;
}

// Fungsi dengan unsafe block di dalam — SAFE dari luar
fn tulis_ke_slice(slice: &mut [i32], index: usize, nilai: i32) -> bool {
    if index >= slice.len() {
        return false;
    }
    // unsafe di dalam — kita verify bounds manually
    unsafe {
        let ptr = slice.as_mut_ptr().add(index);
        *ptr = nilai;
    }
    true
}

fn main() {
    let mut x = 0i32;

    // Perlu unsafe block untuk panggil unsafe fn
    unsafe {
        // Kita jamin: &mut x adalah valid, aligned, tidak null
        tulis_ke_pointer(&mut x as *mut i32, 42);
    }
    println!("x = {}", x); // 42

    // Safe function — tiada unsafe block diperlukan
    let mut v = vec![0, 0, 0, 0, 0];
    tulis_ke_slice(&mut v, 2, 99);
    println!("{:?}", v); // [0, 0, 99, 0, 0]
}
```

---

## Unsafe Method dalam Trait

```rust
// Trait dengan unsafe method
trait BolehTransmute {
    unsafe fn transmute_ke<T>(&self) -> T
    where T: Sized;
}

impl BolehTransmute for u32 {
    unsafe fn transmute_ke<T>(&self) -> T {
        // SANGAT BERBAHAYA jika T bukan f32!
        // Caller mesti jamin sizeof(T) == sizeof(u32)
        std::mem::transmute_copy(self)
    }
}

// Fungsi unsafe dalam std yang selalu digunakan:
fn contoh_std_unsafe() {
    let v = vec![1u8, 2, 3, 4, 5];

    // from_raw_parts — buat slice dari pointer + len
    unsafe {
        let slice: &[u8] = std::slice::from_raw_parts(v.as_ptr(), v.len());
        println!("{:?}", slice);
    }

    // String::from_utf8_unchecked
    let bytes = b"Hello, KADA!".to_vec();
    let s = unsafe { String::from_utf8_unchecked(bytes) };
    println!("{}", s);

    // get_unchecked — akses tanpa bounds check
    let arr = [10, 20, 30, 40, 50];
    unsafe {
        println!("{}", arr.get_unchecked(3)); // 40 — tiada bounds check!
    }
}
```

---

## 🧠 Brain Teaser #2

Apakah yang salah dengan kod ini?

```rust
fn main() {
    let result;
    {
        let x = 42i32;
        let ptr = &x as *const i32;
        result = ptr; // simpan pointer
    } // x di-drop di sini!

    unsafe {
        println!("{}", *result); // A: OK atau B: Undefined Behavior?
    }
}
```

<details>
<summary>👀 Jawapan</summary>

**B: Undefined Behavior!** 💥

`x` di-drop apabila keluar dari inner scope `{ }`. Memorinya dibebaskan. `result` kini adalah **dangling pointer** — ia menunjuk ke memori yang sudah tidak valid.

Apabila kita dereference `*result`, kita membaca memori yang sudah bebas. Ini adalah **use-after-free** — satu daripada bug paling berbahaya dalam C/C++.

**Di sinilah safe Rust membantu:** Versi safe Rust ini tidak akan compile:
```rust
fn main() {
    let result;
    {
        let x = 42i32;
        result = &x; // &i32 — ERROR! x tidak hidup cukup lama
    }
    println!("{}", result); // COMPILER HALANG ini!
}
```

Tapi dengan raw pointer, compiler **tidak memeriksa lifetime**. Itu sebab kita perlu berhati-hati dengan raw pointer — kita yang bertanggungjawab untuk jamin pointer masih valid.

**Pelajaran:** Raw pointer tidak ada lifetime protection. Pastikan data hidup lebih lama dari pointer yang menunjuk kepadanya.
</details>

---

# BAB 4: Static Mutable Variables ⚡

## Global State dengan unsafe

```rust
// Static immutable — SELAMAT, boleh guna di mana-mana
static VERSI: &str = "2.1.0";
static MAX_SAMBUNGAN: u32 = 100;

// Static mutable — BERBAHAYA, data race jika multi-thread!
static mut KIRAAN_GLOBAL: u32 = 0;
static mut LOG_BUFFER: Vec<String> = Vec::new(); // error: cannot have drop type

// Akses static mutable — PERLU unsafe
fn tambah_kiraan() {
    unsafe {
        KIRAAN_GLOBAL += 1;
    }
}

fn baca_kiraan() -> u32 {
    unsafe { KIRAAN_GLOBAL }
}

// MASALAH dengan static mut:
// Dalam multi-threaded program → DATA RACE!
// Dua thread boleh akses serentak → undefined behavior

// PENYELESAIAN LEBIH BAIK: Atomic types atau Mutex
use std::sync::atomic::{AtomicU32, Ordering};
use std::sync::Mutex;
use std::sync::OnceLock; // Rust 1.70+

static KIRAAN_SELAMAT: AtomicU32 = AtomicU32::new(0);

static KONFIGURASI: OnceLock<String> = OnceLock::new();

fn inisialisasi() {
    KONFIGURASI.get_or_init(|| {
        "config=production".to_string()
    });
}

fn main() {
    // Static immutable — no problem
    println!("Versi: {}", VERSI);

    // Static mutable dengan unsafe
    tambah_kiraan();
    tambah_kiraan();
    println!("Kiraan: {}", baca_kiraan()); // 2

    // Atomic — safe tanpa unsafe!
    KIRAAN_SELAMAT.fetch_add(1, Ordering::SeqCst);
    KIRAAN_SELAMAT.fetch_add(1, Ordering::SeqCst);
    println!("Kiraan selamat: {}", KIRAAN_SELAMAT.load(Ordering::SeqCst)); // 2

    // OnceLock — thread-safe initialization
    inisialisasi();
    println!("Config: {}", KONFIGURASI.get().unwrap());
}
```

---

## Lazy Static — Cara Praktik

```toml
# Cargo.toml
[dependencies]
once_cell = "1"  # atau guna std::sync::OnceLock (Rust 1.70+)
```

```rust
use std::sync::OnceLock;
use std::collections::HashMap;

// Global yang diinisialisasi sekali, boleh guna di mana-mana
static LOOKUP_TABLE: OnceLock<HashMap<&str, i32>> = OnceLock::new();

fn dapatkan_lookup() -> &'static HashMap<&'static str, i32> {
    LOOKUP_TABLE.get_or_init(|| {
        let mut m = HashMap::new();
        m.insert("satu", 1);
        m.insert("dua", 2);
        m.insert("tiga", 3);
        m
    })
}

// Atau dengan once_cell (lebih ergonomic)
use once_cell::sync::Lazy;

static REGEX_EMEL: Lazy<regex::Regex> = Lazy::new(|| {
    regex::Regex::new(r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$")
        .expect("Regex tidak valid")
});

fn main() {
    let table = dapatkan_lookup();
    println!("satu = {:?}", table.get("satu")); // Some(1)
    println!("empat = {:?}", table.get("empat")); // None
}
```

---

# BAB 5: Unsafe Traits 🏷️

## Declare dan Implement Unsafe Trait

```rust
// Unsafe trait = implementor mesti jamin invariants tertentu
// yang compiler tidak boleh check

// Contoh dari std: Send dan Sync
// unsafe trait Send { }   // Type selamat untuk hantar ke thread lain
// unsafe trait Sync { }   // Type selamat untuk share antara threads

// Buat unsafe trait sendiri
unsafe trait ContohUnsafeTrait {
    // Setiap implementor MESTI jamin:
    // - pointer() tidak pernah return null
    // - pointer() sentiasa valid selagi objek hidup
    fn pointer(&self) -> *const u8;
}

struct DataSelamat {
    data: Vec<u8>,
}

// UNSAFE impl kerana kita berjanji memenuhi kontrak trait
unsafe impl ContohUnsafeTrait for DataSelamat {
    fn pointer(&self) -> *const u8 {
        self.data.as_ptr() // Vec tidak pernah return null pointer
    }
}

// Send dan Sync — paling common unsafe trait
use std::cell::Cell;

// Cell<T> tidak implement Sync secara default
// Tapi kalau kita TAHU thread-safe dalam context tertentu:
struct MyWrapper {
    data: Cell<i32>,
}

// UNSAFE! Kita berjanji tidak ada data race
// (contoh: hanya satu thread akses pada satu masa)
unsafe impl Sync for MyWrapper {}

// Negative impl — explicitly NOT Send/Sync
// (ini selamat, tidak perlu unsafe)
struct NotSendable(*mut i32); // raw pointer → tidak Send secara default

fn main() {
    let d = DataSelamat { data: vec![1, 2, 3] };
    let ptr = d.pointer();
    unsafe {
        println!("Byte pertama: {}", *ptr);
    }
}
```

---

# BAB 6: FFI — Foreign Function Interface 🌐

## Panggil Fungsi C dari Rust

```rust
// Link dengan C library
// Cara 1: extern "C" block
extern "C" {
    fn strlen(s: *const std::os::raw::c_char) -> usize;
    fn abs(x: std::os::raw::c_int) -> std::os::raw::c_int;
    fn malloc(size: usize) -> *mut std::os::raw::c_void;
    fn free(ptr: *mut std::os::raw::c_void);
    fn printf(format: *const std::os::raw::c_char, ...) -> std::os::raw::c_int;
}

// Guna libc crate untuk types C yang portable
// [dependencies]
// libc = "0.2"

use std::ffi::{CStr, CString};

fn main() {
    // Panggil fungsi C
    let nilai = unsafe { abs(-42) };
    println!("abs(-42) = {}", nilai); // 42

    // Buat C string dari Rust string
    let rust_str = "Hello, C!";
    let c_str = CString::new(rust_str).expect("CString gagal");

    // Dapatkan panjang menggunakan C strlen
    let panjang = unsafe { strlen(c_str.as_ptr()) };
    println!("Panjang: {}", panjang); // 9

    // Baca C string ke Rust string
    let ptr: *const i8 = c_str.as_ptr();
    let balik = unsafe {
        CStr::from_ptr(ptr)
            .to_str()
            .expect("Bukan UTF-8 valid")
    };
    println!("Balik: {}", balik); // Hello, C!

    // Manual memory management
    unsafe {
        let ptr = malloc(std::mem::size_of::<i32>());
        if !ptr.is_null() {
            let int_ptr = ptr as *mut i32;
            *int_ptr = 42;
            println!("Malloc value: {}", *int_ptr);
            free(ptr); // MESTI free!
        }
    }
}
```

---

## Expose Rust Functions ke C

```rust
// Buat fungsi Rust yang boleh dipanggil dari C

// no_mangle = jangan ubah nama fungsi (untuk C linking)
// extern "C" = guna C calling convention
#[no_mangle]
pub extern "C" fn tambah_dari_c(a: i32, b: i32) -> i32 {
    a + b
}

#[no_mangle]
pub extern "C" fn salam_dari_rust(nama: *const std::os::raw::c_char) {
    if nama.is_null() {
        return;
    }
    unsafe {
        let c_str = std::ffi::CStr::from_ptr(nama);
        if let Ok(s) = c_str.to_str() {
            println!("Hello from Rust, {}!", s);
        }
    }
}

// Struct yang boleh pass ke C
#[repr(C)]  // Pastikan memory layout sama dengan C
struct KoordinatC {
    lat: f64,
    lon: f64,
}

#[no_mangle]
pub extern "C" fn kira_jarak(a: KoordinatC, b: KoordinatC) -> f64 {
    let dlat = (b.lat - a.lat).to_radians();
    let dlon = (b.lon - a.lon).to_radians();
    let h = (dlat/2.0).sin().powi(2)
          + a.lat.to_radians().cos()
          * b.lat.to_radians().cos()
          * (dlon/2.0).sin().powi(2);
    6371.0 * 2.0 * h.sqrt().atan2((1.0 - h).sqrt())
}
```

---

## Cargo.toml untuk FFI

```toml
# Untuk panggil C library dari Rust
[dependencies]
libc = "0.2"          # C types yang portable

# Untuk expose Rust sebagai C library
[lib]
crate-type = ["cdylib"]   # Dynamic library (.so / .dll)
# atau
crate-type = ["staticlib"] # Static library (.a)

# Build script untuk link C library
[build-dependencies]
cc = "1"
```

```rust
// build.rs — compile dan link C code
fn main() {
    cc::Build::new()
        .file("src/helper.c")
        .compile("helper");

    println!("cargo:rustc-link-lib=ssl");   // link dengan libssl
    println!("cargo:rustc-link-lib=crypto");
}
```

---

# BAB 7: Union Types 🔀

## Union — Like C union

```rust
// Union: semua fields kongsi memori yang sama
// Berguna untuk interop C atau type punning

#[repr(C)]
union IpAddr {
    v4: [u8; 4],    // 4 bytes
    v6: [u8; 16],   // 16 bytes
    raw: u128,       // 16 bytes — sama memori dengan v6!
}

// Semua fields dalam union ada offset 0
// sizeof(IpAddr) = sizeof(largest field) = 16 bytes

fn main() {
    let ip = IpAddr {
        v4: [192, 168, 1, 1],
    };

    // Akses union field — PERLU unsafe!
    unsafe {
        println!("IPv4: {:?}", ip.v4); // [192, 168, 1, 1]
        // Mengakses v6 atau raw untuk data yang ditulis sebagai v4
        // adalah undefined behavior dalam kebanyakan kes!
    }

    // Union untuk type punning (float bits)
    union FloatInt {
        f: f32,
        i: u32,
    }

    let fi = FloatInt { f: 3.14f32 };
    unsafe {
        println!("3.14 bits: {:#010x}", fi.i); // 0x4048f5c3
    }
}
```

---

# BAB 8: Common Unsafe Patterns 📐

## Pattern 1: Split Slice Menjadi Dua &mut

```rust
// Safe Rust tidak benarkan dua &mut ke slice yang sama
// walaupun bahagiannya tidak overlap!

fn split_at_mut_manual<T>(slice: &mut [T], mid: usize) -> (&mut [T], &mut [T]) {
    assert!(mid <= slice.len());

    let len = slice.len();
    let ptr = slice.as_mut_ptr();

    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(
                ptr.add(mid),
                len - mid
            ),
        )
    }
    // SELAMAT kerana:
    // 1. mid <= len (kita assert)
    // 2. Dua slices tidak overlap (mula dari ptr dan ptr+mid)
    // 3. Kita tidak buat dua pointer ke slice yang sama
}

fn main() {
    let mut v = vec![1, 2, 3, 4, 5, 6];
    let (kiri, kanan) = split_at_mut_manual(&mut v, 3);

    kiri.sort_unstable_by(|a, b| b.cmp(a)); // sort descending
    kanan.sort_unstable();                    // sort ascending

    println!("Kiri:  {:?}", kiri);  // [3, 2, 1]
    println!("Kanan: {:?}", kanan); // [4, 5, 6]
    println!("Semua: {:?}", v);     // [3, 2, 1, 4, 5, 6]

    // Standard library ada split_at_mut() yang buat ini!
    let (a, b) = v.split_at_mut(3);
    println!("{:?} {:?}", a, b);
}
```

---

## Pattern 2: Transmute dengan Berhati-hati

```rust
use std::mem;

// transmute — tukar bytes dari satu type ke type lain
// SANGAT BERBAHAYA — pastikan layout memory sama!

fn main() {
    // ✔ Safe transmute: f32 ↔ u32 (saiz sama, ini standard)
    let f: f32 = 1.0;
    let bits: u32 = unsafe { mem::transmute(f) };
    println!("1.0f32 = {:#010x}", bits); // 0x3f800000

    // Ini sebenarnya lebih baik dengan safe alternatives:
    let bits2 = f.to_bits();
    println!("same: {}", bits == bits2); // true

    // ✔ Transmute slice types (saiz sama)
    let bytes: [u8; 4] = [0x00, 0x00, 0x80, 0x3f]; // little-endian 1.0f32
    let nilai: f32 = unsafe { mem::transmute(bytes) };
    println!("bytes → f32: {}", nilai); // 1.0

    // ❌ BERBAHAYA: Transmute ke type yang berbeza saiz
    // let big: u64 = unsafe { mem::transmute(1u32) }; // PANIC atau UB!

    // ❌ BERBAHAYA: Transmute pointer ke reference
    // let r: &i32 = unsafe { mem::transmute(ptr) };
    // Ini boleh buat dangling reference!

    // Cara lebih selamat: transmute → bytemuck crate
    // bytemuck::cast::<u32, f32>(bits) ← safe, checked cast
}
```

---

## Pattern 3: Buat Type dari Raw Parts

```rust
fn main() {
    // Buat Vec dari raw parts
    let mut v: Vec<i32> = Vec::with_capacity(5);

    unsafe {
        // Tulis terus ke buffer
        let ptr = v.as_mut_ptr();
        for i in 0..5 {
            ptr.add(i).write(i as i32 * 10);
        }
        // Set len secara manual (BERBAHAYA jika tidak betul!)
        v.set_len(5);
    }

    println!("{:?}", v); // [0, 10, 20, 30, 40]

    // Pecah Vec kepada parts (leak memory kalau tidak reconstruct!)
    let (ptr, len, cap) = v.into_raw_parts();

    // Reconstruct Vec dari parts
    let v2: Vec<i32> = unsafe {
        Vec::from_raw_parts(ptr, len, cap)
    };

    println!("{:?}", v2); // [0, 10, 20, 30, 40]
    // v2 akan drop dan free memory dengan betul

    // Buat slice dari raw parts
    let data = [1i32, 2, 3, 4, 5];
    let slice: &[i32] = unsafe {
        std::slice::from_raw_parts(data.as_ptr(), 3) // ambil 3 elemen sahaja
    };
    println!("{:?}", slice); // [1, 2, 3]
}
```

---

# BAB 9: Safety Abstraction — Wrap unsafe dengan Safe API 🛡️

## Prinsip Utama: Encapsulate unsafe

```
PRINSIP TERPENTING:

  Buat unsafe API SEKECIL MUNGKIN.
  Wrap unsafe dalam module/struct dengan safe public API.
  Invariants dicheck di sempadan safe/unsafe.

  "Make the unsafe interface as small as possible,
   and make the safe interface as large as possible."

  Kod safe yang guna abstraksi anda tidak perlu tahu
  tentang unsafe di bawah — ia transparent!
```

```rust
// Contoh: Buat safe wrapper untuk raw memory buffer

pub struct SafeBuffer {
    ptr: *mut u8,
    len: usize,
    cap: usize,
}

impl SafeBuffer {
    pub fn new(kapasiti: usize) -> Self {
        let layout = std::alloc::Layout::array::<u8>(kapasiti).unwrap();
        let ptr = unsafe { std::alloc::alloc(layout) };
        if ptr.is_null() {
            std::alloc::handle_alloc_error(layout);
        }

        SafeBuffer { ptr, len: 0, cap: kapasiti }
    }

    pub fn tulis(&mut self, data: &[u8]) -> bool {
        if self.len + data.len() > self.cap {
            return false;
        }

        unsafe {
            std::ptr::copy_nonoverlapping(
                data.as_ptr(),
                self.ptr.add(self.len),
                data.len(),
            );
        }
        self.len += data.len();
        true
    }

    pub fn baca(&self) -> &[u8] {
        unsafe {
            std::slice::from_raw_parts(self.ptr, self.len)
        }
    }

    pub fn len(&self) -> usize { self.len }
    pub fn kapasiti(&self) -> usize { self.cap }
    pub fn is_empty(&self) -> bool { self.len == 0 }
}

impl Drop for SafeBuffer {
    fn drop(&mut self) {
        if self.cap > 0 {
            let layout = std::alloc::Layout::array::<u8>(self.cap).unwrap();
            unsafe { std::alloc::dealloc(self.ptr, layout); }
        }
    }
}

// PENTING: Raw pointer tidak impl Send/Sync secara default
// Kita perlu explicitly impl jika selamat
unsafe impl Send for SafeBuffer {}
unsafe impl Sync for SafeBuffer {}

fn main() {
    let mut buf = SafeBuffer::new(100);

    // Pengguna API tidak perlu tahu tentang unsafe di dalam!
    buf.tulis(b"Hello, ");
    buf.tulis(b"KADA!");

    println!("{:?}", std::str::from_utf8(buf.baca()).unwrap());
    // "Hello, KADA!"

    println!("Len: {}", buf.len()); // 12
} // Drop dipanggil — memory dibebaskan dengan betul!
```

---

## 🧠 Brain Teaser #3

Adakah abstraksi ini selamat? Kenapa ya atau tidak?

```rust
pub struct UnsafeStack<T> {
    data: Vec<T>,
}

impl<T> UnsafeStack<T> {
    pub fn new() -> Self {
        UnsafeStack { data: Vec::new() }
    }

    pub fn push(&mut self, item: T) {
        self.data.push(item);
    }

    pub fn pop_unsafe(&mut self) -> T {
        // Anggap stack tidak pernah kosong
        unsafe {
            let len = self.data.len();
            self.data.set_len(len - 1);
            std::ptr::read(self.data.as_ptr().add(len - 1))
        }
    }
}

fn main() {
    let mut stack: UnsafeStack<String> = UnsafeStack::new();
    stack.push("hello".to_string());
    let s = stack.pop_unsafe(); // Adakah ini selamat?
    println!("{}", s);

    // Bagaimana pula dengan ini?
    let s2 = stack.pop_unsafe(); // Stack kosong!
}
```

<details>
<summary>👀 Jawapan</summary>

**Bahagian pertama (stack ada item) — mungkin selamat, tapi ada masalah halus.**

**Bahagian kedua (stack kosong) — UNDEFINED BEHAVIOR!** 💥

Masalah:
1. `set_len(len - 1)` pada stack kosong: `len = 0`, `0 - 1` akan **panic** kerana usize tidak boleh jadi negatif, atau jadi `usize::MAX` (overflow!) bergantung pada debug/release mode.
2. `ptr::read()` pada indeks yang tidak valid = membaca memori di luar Vec = UB.

**Abstraksi yang betul:**
```rust
pub fn pop(&mut self) -> Option<T> {
    if self.data.is_empty() {
        return None; // Handle kes kosong dengan selamat!
    }

    unsafe {
        let len = self.data.len();
        self.data.set_len(len - 1);
        Some(std::ptr::read(self.data.as_ptr().add(len - 1)))
    }
}
```

**Pelajaran:** Safe abstraction mesti handle SEMUA kes edge. Jika fungsi `pub` boleh menyebabkan UB dari luar, ia bukan safe abstraction — ia adalah unsafe function yang tersorok!

Fungsi dengan `unsafe` dalam implementasi BOLEH jadi `pub` (tanpa `unsafe` dalam signature) **hanya jika** pengguna tidak boleh menyebabkan UB tidak kira apa input yang diberikan.
</details>

---

## Cara Tulis Unsafe yang Baik

```rust
// ─── Template untuk safe wrapper ──────────────────────────────

/// Dokumen dengan jelas KENAPA ini selamat (Safety comment)
///
/// # Safety
///
/// Caller mesti jamin:
/// - Syarat 1: ...
/// - Syarat 2: ...
unsafe fn fungsi_unsafe_dengan_kontrak(ptr: *const i32) -> i32 {
    // Implementasi
    *ptr
}

// Untuk unsafe block dalam fungsi safe:
fn fungsi_safe(slice: &[i32]) -> i32 {
    // SAFETY: slice.len() >= 1 dijamin kerana kita check di atas
    assert!(!slice.is_empty(), "slice tidak boleh kosong");

    unsafe {
        // SAFETY: index 0 valid kerana slice tidak kosong (check di atas)
        *slice.get_unchecked(0)
    }
}

// Guna clippy lint untuk enforce safety docs:
// #![deny(clippy::undocumented_unsafe_blocks)]
```

---

# BAB 10: Mini Project — Safe Wrapper untuk C Library 🏗️

```rust
// Simulate wrapper untuk C library
// (dalam real-world, gunakan tools seperti bindgen)

// Simulasi C API (dalam real project, ini dari header file)
mod c_api {
    use std::os::raw::{c_char, c_int, c_void};

    // Simulate C struct
    #[repr(C)]
    pub struct CDatabase {
        _private: [u8; 0], // opaque pointer pattern
    }

    // Simulate C functions
    pub unsafe fn db_buka(laluan: *const c_char) -> *mut CDatabase {
        // Dalam real code, ini call C function
        // Kita simulasi dengan Box
        println!("[C] Buka database: {:?}",
            std::ffi::CStr::from_ptr(laluan));
        Box::into_raw(Box::new(CDatabase { _private: [] }))
    }

    pub unsafe fn db_tutup(db: *mut CDatabase) {
        if !db.is_null() {
            println!("[C] Tutup database");
            drop(Box::from_raw(db));
        }
    }

    pub unsafe fn db_simpan(
        db: *mut CDatabase,
        kunci: *const c_char,
        nilai: *const c_char,
    ) -> c_int {
        println!("[C] Simpan: {:?} = {:?}",
            std::ffi::CStr::from_ptr(kunci),
            std::ffi::CStr::from_ptr(nilai));
        0 // 0 = success
    }

    pub unsafe fn db_baca(
        db: *mut CDatabase,
        kunci: *const c_char,
        buf: *mut c_char,
        buf_len: usize,
    ) -> c_int {
        println!("[C] Baca: {:?}", std::ffi::CStr::from_ptr(kunci));
        // Simulate copy "nilai_test" ke buffer
        let hasil = b"nilai_test\0";
        let salin = hasil.len().min(buf_len);
        std::ptr::copy_nonoverlapping(hasil.as_ptr() as *const c_char, buf, salin);
        salin as c_int
    }
}

// ─── Safe Rust Wrapper ────────────────────────────────────────

use std::ffi::{CString, CStr};

/// Error type untuk operasi database
#[derive(Debug, thiserror::Error)]
pub enum DbError {
    #[error("Gagal buka database: {0}")]
    BukaGagal(String),
    #[error("Operasi gagal dengan kod: {0}")]
    OperasiGagal(i32),
    #[error("String tidak sah: {0}")]
    StringTidakSah(#[from] std::ffi::NulError),
    #[error("UTF-8 tidak sah")]
    Utf8Gagal(#[from] std::str::Utf8Error),
}

pub type Result<T> = std::result::Result<T, DbError>;

/// Safe wrapper untuk C database library
pub struct Database {
    ptr: *mut c_api::CDatabase,
}

impl Database {
    /// Buka sambungan database
    pub fn buka(laluan: &str) -> Result<Self> {
        let c_laluan = CString::new(laluan)?; // Handle NulError

        let ptr = unsafe { c_api::db_buka(c_laluan.as_ptr()) };

        if ptr.is_null() {
            return Err(DbError::BukaGagal(laluan.to_string()));
        }

        Ok(Database { ptr })
    }

    /// Simpan pasangan kunci-nilai
    pub fn simpan(&mut self, kunci: &str, nilai: &str) -> Result<()> {
        let c_kunci = CString::new(kunci)?;
        let c_nilai = CString::new(nilai)?;

        let kod = unsafe {
            c_api::db_simpan(self.ptr, c_kunci.as_ptr(), c_nilai.as_ptr())
        };

        if kod != 0 {
            Err(DbError::OperasiGagal(kod))
        } else {
            Ok(())
        }
    }

    /// Baca nilai berdasarkan kunci
    pub fn baca(&self, kunci: &str) -> Result<String> {
        let c_kunci = CString::new(kunci)?;
        let mut buf = vec![0i8; 256]; // Buffer untuk C string

        let saiz = unsafe {
            c_api::db_baca(
                self.ptr,
                c_kunci.as_ptr(),
                buf.as_mut_ptr(),
                buf.len(),
            )
        };

        if saiz < 0 {
            return Err(DbError::OperasiGagal(saiz));
        }

        // Convert C string ke Rust String
        let c_str = unsafe { CStr::from_ptr(buf.as_ptr()) };
        Ok(c_str.to_str()?.to_string())
    }
}

// Drop = tutup sambungan bila Database di-drop
impl Drop for Database {
    fn drop(&mut self) {
        if !self.ptr.is_null() {
            unsafe { c_api::db_tutup(self.ptr); }
            self.ptr = std::ptr::null_mut();
        }
    }
}

// PENTING: Jangan implement Send/Sync kecuali C library thread-safe
// unsafe impl Send for Database {}

// ─── Main ──────────────────────────────────────────────────────

fn main() -> Result<()> {
    // Guna wrapper sepenuhnya SELAMAT — tiada unsafe di sini!
    let mut db = Database::buka("/tmp/test.db")?;

    db.simpan("nama", "Ali Ahmad")?;
    db.simpan("bahagian", "ICT")?;

    let nama = db.baca("nama")?;
    println!("Nama: {}", nama);

    let bahagian = db.baca("bahagian")?;
    println!("Bahagian: {}", bahagian);

    // Database::drop() dipanggil di sini — tutup sambungan!
    Ok(())
}
```

---

# 📋 Rujukan Pantas — Unsafe Rust Cheat Sheet

## Lima Superpower Unsafe

```rust
unsafe {
    // 1. Dereference raw pointer
    let val = *ptr;

    // 2. Panggil unsafe function
    fungsi_unsafe();
    std::mem::transmute::<f32, u32>(1.0);

    // 3. Akses static mutable
    STATIC_MUT += 1;

    // 4. Implement unsafe trait
    // (dalam impl block, bukan dalam unsafe block)
}

// 4. Implement unsafe trait:
unsafe impl Send for MyType {}

// 5. Akses union field
let val = unsafe { my_union.field };
```

## Raw Pointer Operations

```rust
let ptr: *const T = &val;         // buat dari reference
let ptr: *mut T   = &mut val;
let ptr: *const T = std::ptr::null();     // null pointer
let ptr: *mut T   = std::ptr::null_mut();

// Semua ini perlu unsafe:
*ptr                               // dereference
ptr.add(n)                         // pointer arithmetic
ptr.offset(n)                      // pointer arithmetic (signed)
ptr.read()                         // sama seperti *ptr
ptr.write(val)                     // sama seperti *ptr = val
ptr.copy_to(dst, n)                // copy n items
std::slice::from_raw_parts(ptr, n) // buat slice

// Ini SAFE (tiada unsafe diperlukan):
ptr.is_null()                      // semak null
ptr as usize                       // tukar ke integer
&val as *const T                   // buat raw pointer
```

## Peraturan Keselamatan

```
UNSAFE BOLEH BUAT:
  ✔ Raw pointer dereference
  ✔ Panggil unsafe fn
  ✔ Akses static mut
  ✔ impl unsafe trait
  ✔ Akses union field

UNSAFE TIDAK BOLEH BUAT:
  ✘ Skip borrow checker (masih ada!)
  ✘ Skip type checking
  ✘ Buat lifetime sembarangan
  ✘ Ignore race conditions (masih UB!)

INVARIANTS YANG MESTI DIJAGA:
  ✔ Pointer tidak null sebelum dereference
  ✔ Pointer valid dan aligned
  ✔ Tiada aliasing mutable references
  ✔ Memory tidak freed sebelum akses
  ✔ Data dalam state yang valid
```

## Panduan Guna unsafe

```
1. MINIMUMKAN unsafe code
   → Buat blok unsafe sekecil mungkin
   → Wrap dalam safe abstraction

2. DOKUMENTASIKAN dengan // SAFETY:
   → Terangkan KENAPA ini selamat
   → Senaraikan invariants yang dijamin

3. TEST dengan miri (Miri interpreter)
   cargo install miri
   cargo miri test

4. AUDIT dengan cargo-audit
   Semak ada vulnerability dalam dependencies

5. GUNA ABSTRAKSI SEDIA ADA dulu
   slice.split_at_mut() > unsafe manual split
   atomic > static mut
   OnceLock > lazy_static dengan manual unsafe
```

---

## 🏆 Cabaran Akhir

Cuba implement salah satu:

1. **Safe Ring Buffer** — circular buffer menggunakan raw memory, expose safe push/pop
2. **Arena Allocator** — allocate banyak objects dalam satu block, free semuanya sekaligus
3. **Atomic Stack** — lock-free stack menggunakan `AtomicPtr` dan CAS operations
4. **Memory-Mapped File** — wrapper selamat untuk `mmap` syscall
5. **SIMD String Search** — guna unsafe SIMD intrinsics untuk cari substring dengan laju

---

*unsafe Rust bukan musuh — ia adalah alat berkuasa yang memerlukan pengetahuan dan tanggungjawab.*
*Safe Rust adalah default. unsafe adalah bila kita memerlukan kuasa penuh.* 🦀
