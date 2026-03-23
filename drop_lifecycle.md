# 💧 Drop & Resource Lifecycle dalam Rust — Panduan Lengkap

> "Bila sebenarnya resource dilepaskan?"
> Faham RAII, drop order, scope, dan cara elak deadlock
> yang berlaku bukan kerana kod salah — tapi scope salah.

---

## Masalah Yang Wujud

```
Bahasa lain:
  Python:  Garbage collector → resource dilepas "bila-bila masa"
  Java:    Finalizer tidak dijamin dipanggil
  C:       Kena free() manual → mudah lupa, double-free
  PHP:     Connection pool, file handle → leaked dalam long scripts

Rust dengan RAII:
  Resource dilepas TEPAT bila variable keluar dari scope.
  Deterministik. Tidak ada "bila-bila masa".
  Tiada GC. Tiada finalizer yang tidak dijamin.

TAPI ramai tidak sedar:
  "Variable keluar scope" tidak semestinya bila anda fikir!

  // Bug halus ini berlaku lebih kerap dari yang disangka:
  let _guard = mutex.lock().unwrap();    // ← _ prefix TIDAK drop segera!
  let _guard = mutex.lock().unwrap();    // ← ini akan DEADLOCK!
  //     ^^ underscore prefix = drop pada AKHIR scope, bukan serta-merta!
```

---

## Peta Pembelajaran

```
Bab 1  → RAII — Resource Acquisition Is Initialization
Bab 2  → Drop Trait — Tulis Custom Destructor
Bab 3  → Drop Order — Siapa Di-drop Dulu?
Bab 4  → Scope & Drop Manual dengan drop()
Bab 5  → File Handle — Bila Ia Ditutup?
Bab 6  → Mutex Lock — Deadlock dari Scope Salah
Bab 7  → Database Connection — Lifecycle Pool
Bab 8  → Guard Pattern — RAII untuk Custom Resources
Bab 9  → Drop dalam Async — Perangkap Tersembunyi
Bab 10 → Mini Project: Resource Manager
```

---

# BAB 1: RAII — The Core Pattern 🏛️

## RAII: Resource Acquisition Is Initialization

```rust
// RAII = Resource diambil semasa PEMBINAAN (construct)
//        Resource dilepas semasa PEMUSNAHAN (destruct/drop)
//
// Ini bermaksud: resource lifecycle = variable lifetime
// Tiada boleh "lupa" lepaskan resource!

fn demo_raii_asas() {
    // Bila File::create() dipanggil:
    //   → OS membuka fail handle (resource diambil)
    let mut fail = std::fs::File::create("data.txt").unwrap();

    std::io::Write::write_all(&mut fail, b"Hello!").unwrap();

    println!("Fail masih terbuka...");

    // Bila `fail` keluar dari scope DI SINI:
    //   → Drop dipanggil automatik
    //   → fail.flush() dijalankan
    //   → OS menutup fail handle (resource dilepas)
}  // ← `fail` di-drop di sini

// Berbanding cara tidak selamat (C):
// FILE* f = fopen("data.txt", "w");
// fprintf(f, "Hello!");
// // Kalau lupa fclose(f) → FILE DESCRIPTOR LEAK!

fn main() {
    demo_raii_asas();
    // Fail PASTI ditutup sebelum baris ini!
    println!("Fail sudah ditutup");

    // Cuba buka semula — berjaya kerana sudah ditutup
    let kandungan = std::fs::read_to_string("data.txt").unwrap();
    println!("{}", kandungan);

    std::fs::remove_file("data.txt").unwrap();
}
```

---

## RAII dalam Kehidupan Sebenar

```rust
use std::sync::Mutex;

struct Database {
    sambungan: String,
    terbuka:   bool,
}

impl Database {
    fn sambung(url: &str) -> Self {
        println!("[DB] Sambungan dibuka ke: {}", url);
        Database {
            sambungan: url.into(),
            terbuka:   true,
        }
    }

    fn query(&self, sql: &str) -> String {
        format!("[DB] Hasil query: {}", sql)
    }
}

impl Drop for Database {
    fn drop(&mut self) {
        if self.terbuka {
            println!("[DB] Sambungan ditutup: {}", self.sambungan);
            self.terbuka = false;
        }
    }
}

fn ambil_data() -> String {
    let db = Database::sambung("mysql://localhost/kada_db");
    //       ↑ sambungan dibuka di sini

    let hasil = db.query("SELECT * FROM pekerja");

    println!("Data diambil");

    hasil
    // ← `db` di-drop di sini
    // ← sambungan DITUTUP sebelum `hasil` dikembalikan
}  // bukan di sini!

fn main() {
    let data = ambil_data();
    println!("Selepas fungsi: {}", data);
    // Sambungan sudah ditutup bila sampai baris ini!
}
```

---

## 🧠 Brain Teaser #1

Apakah output kod ini, dan BILA setiap resource dilepas?

```rust
struct Sumber(String);

impl Drop for Sumber {
    fn drop(&mut self) { println!("DROP: {}", self.0); }
}

fn buat() -> Sumber {
    let s1 = Sumber("dalam fungsi".into());
    println!("Buat s1");
    s1  // ← move keluar dari fungsi
}   // ← adakah s1 di-drop di sini?

fn main() {
    println!("Mula");
    let s2 = buat();
    println!("Selepas buat()");
    let s3 = Sumber("s3".into());
    println!("Buat s3");
}   // ← drop di sini?
```

<details>
<summary>👀 Jawapan</summary>

```
Output:
  Mula
  Buat s1
  Selepas buat()
  Buat s3
  DROP: s3          ← drop dalam susunan TERBALIK (LIFO)
  DROP: dalam fungsi ← s2 drop selepas s3

Penjelasan:
  1. s1 dibuat dalam buat()
  2. s1 di-MOVE keluar dari fungsi sebagai return value
     → TIDAK di-drop dalam fungsi! Ownership berpindah ke s2
  3. s3 dibuat
  4. Bila main() berakhir, drop dalam order TERBALIK:
     → s3 di-drop DULU (dibuat kemudian, drop dahulu — LIFO)
     → s2 di-drop (sebenarnya s1 yang pindah ke s2)

PENTING: Return value TIDAK di-drop dalam fungsi asal!
Ownership BERPINDAH kepada caller.
```
</details>

---

# BAB 2: Drop Trait — Custom Destructor 🔨

## Implement Drop Sendiri

```rust
use std::fs;

struct TempFail {
    laluan: String,
}

impl TempFail {
    fn baru(laluan: &str) -> Self {
        fs::write(laluan, "").unwrap(); // buat fail
        println!("[TempFail] Fail dibuat: {}", laluan);
        TempFail { laluan: laluan.into() }
    }

    fn tulis(&self, data: &str) {
        fs::write(&self.laluan, data).unwrap();
    }
}

impl Drop for TempFail {
    fn drop(&mut self) {
        // Auto-padam fail bila variable di-drop!
        if let Err(e) = fs::remove_file(&self.laluan) {
            eprintln!("[TempFail] Gagal padam: {}", e);
        } else {
            println!("[TempFail] Fail dipadam: {}", self.laluan);
        }
    }
}

// ── Custom destructor untuk connection ────────────────────────

struct KoleksiThread {
    pekerja: Vec<std::thread::JoinHandle<()>>,
    nama:    String,
}

impl KoleksiThread {
    fn baru(nama: &str) -> Self {
        println!("[Pool] Koleksi '{}' dibuat", nama);
        KoleksiThread { pekerja: Vec::new(), nama: nama.into() }
    }

    fn tambah<F>(&mut self, f: F)
    where F: FnOnce() + Send + 'static
    {
        self.pekerja.push(std::thread::spawn(f));
    }
}

impl Drop for KoleksiThread {
    fn drop(&mut self) {
        println!("[Pool] Tunggu semua pekerja selesai...");
        // Flush/join semua threads sebelum drop
        while let Some(h) = self.pekerja.pop() {
            h.join().unwrap();
        }
        println!("[Pool] Koleksi '{}' ditutup", self.nama);
    }
}

fn main() {
    {
        let fail = TempFail::baru("/tmp/test_raii.txt");
        fail.tulis("Data penting");
        println!("Fail wujud: {}", fs::metadata(&fail.laluan).is_ok());
    } // ← TempFail::drop() dipanggil di sini!

    println!("Fail wujud lagi? {}",
        fs::metadata("/tmp/test_raii.txt").is_ok());
    // false — sudah dipadam!
}
```

---

## Peraturan Drop

```rust
// Apa yang TIDAK boleh buat dalam Drop:

use std::sync::Arc;

struct ContohDrop {
    data: Arc<i32>,
}

impl Drop for ContohDrop {
    fn drop(&mut self) {
        // ✔ BOLEH: access self.* (borrow)
        println!("Dropping ContohDrop dengan data: {}", self.data);

        // ✔ BOLEH: move field keluar dengan replace
        // let _ambil = std::mem::replace(&mut self.data, Arc::new(0));

        // ❌ TIDAK BOLEH: explicit drop(self) dalam Drop
        // drop(self); // compile error — infinite recursion!

        // ❌ TIDAK BOLEH: panggil self.drop() secara manual
        // self.drop(); // compile error!

        // ❌ TIDAK BOLEH: return/continue/break dari drop
        // (walaupun boleh panic — tidak disyorkan!)

        // ✔ BOLEH: panic dalam drop (berlaku unwind)
        // tapi ini akan abort kalau double-panic!
    }
}

// ── Mencegah double-drop dengan Option ────────────────────────
struct ResourceSelamat {
    data: Option<Vec<u8>>,
}

impl ResourceSelamat {
    fn baru() -> Self { ResourceSelamat { data: Some(vec![1, 2, 3]) } }

    fn ambil(&mut self) -> Option<Vec<u8>> {
        self.data.take() // take() → returns dan set ke None
    }
}

impl Drop for ResourceSelamat {
    fn drop(&mut self) {
        if let Some(data) = self.data.take() {
            // Hanya buat cleanup kalau data masih ada
            println!("Membersihkan {} bytes", data.len());
        }
        // Kalau data sudah diambil (take), tidak ada yang perlu dibersih
    }
}
```

---

# BAB 3: Drop Order — Siapa Di-drop Dulu? 📋

## LIFO: Last In, First Out

```rust
struct Jejak(String);

impl Drop for Jejak {
    fn drop(&mut self) { println!("DROP: {}", self.0); }
}

fn demo_drop_order() {
    println!("--- Variables dalam scope ---");
    let a = Jejak("A".into());  // dibuat pertama
    let b = Jejak("B".into());  // dibuat kedua
    let c = Jejak("C".into());  // dibuat ketiga

    println!("Keluar dari scope...");
    // Drop order: C, B, A (TERBALIK dari creation order!)
    //
    // Output:
    // Keluar dari scope...
    // DROP: C
    // DROP: B
    // DROP: A
}

fn demo_drop_struct() {
    println!("\n--- Fields dalam struct ---");

    struct Bingkai {
        x: Jejak,    // field pertama
        y: Jejak,    // field kedua
        z: Jejak,    // field ketiga
    }

    let _b = Bingkai {
        x: Jejak("X".into()),
        y: Jejak("Y".into()),
        z: Jejak("Z".into()),
    };

    println!("Drop struct...");
    // Drop order: X, Y, Z (SAMA seperti field order, bukan terbalik!)
    // Struct fields di-drop dalam DECLARATION ORDER
    // (bukan LIFO seperti variables!)
    //
    // Output:
    // Drop struct...
    // DROP: X
    // DROP: Y
    // DROP: Z
}

fn demo_drop_tuple() {
    println!("\n--- Tuple ───");
    let _t = (Jejak("0".into()), Jejak("1".into()), Jejak("2".into()));
    println!("Drop tuple...");
    // Tuple: 0, 1, 2 (sama seperti struct — declaration order)
}

fn main() {
    demo_drop_order();
    demo_drop_struct();
    demo_drop_tuple();
}
```

---

## Drop Order dalam Nested Scope

```rust
struct R(u32);

impl Drop for R {
    fn drop(&mut self) { print!("[{}] ", self.0); }
}

fn main() {
    let a = R(1);  // dibuat pertama

    {
        let b = R(2);
        let c = R(3);
        println!("Dalam scope dalam:");
        // b dan c akan di-drop bila scope ini berakhir
    }
    // Output: [3] [2]   ← c dulu, kemudian b

    println!("\nSelepas scope dalam");
    // a masih hidup!

    let d = R(4);
    println!("Dibuat d");
    // Bila main berakhir:
    // d di-drop dulu, kemudian a
    // Output: [4] [1]

    // Jumlah output: [3] [2]  [4] [1]
}
```

---

# BAB 4: Scope & drop() Manual 🎯

## drop() Fungsi — Drop Awal

```rust
use std::sync::Mutex;

fn main() {
    let mutex = Mutex::new(42i32);

    // ── drop() biasa — drop bila keluar scope ─────────────────
    {
        let guard = mutex.lock().unwrap();
        println!("Nilai: {}", *guard);
    } // guard di-drop di sini → lock dilepas

    // ── drop() manual — drop sebelum scope berakhir ───────────
    let guard = mutex.lock().unwrap();
    println!("Nilai: {}", *guard);

    drop(guard);  // ← drop SEGERA! Lock dilepas di sini
    println!("Lock sudah dilepas");

    // Boleh lock semula sekarang!
    let guard2 = mutex.lock().unwrap();
    println!("Nilai semula: {}", *guard2);

    // ── BAHAYA: _ prefix tidak sama dengan drop() segera! ────
    let mutex2 = Mutex::new(0i32);

    let _guard = mutex2.lock().unwrap();  // _ prefix = masih hidup sampai scope!
    println!("_ guard masih hidup!");
    // mutex2.lock()  ← INI AKAN DEADLOCK!
    // kerana _guard masih memegang lock!

    // BETUL:
    drop(_guard);   // drop dulu
    let _g2 = mutex2.lock().unwrap();  // baru boleh lock semula
}
```

---

## _ vs \_nama — Perbezaan Kritikal

```rust
struct Guard(String);

impl Drop for Guard {
    fn drop(&mut self) { println!("DROP Guard: {}", self.0); }
}

fn main() {
    println!("=== _ (underscore) ===");
    {
        let _ = Guard("A".into());     // DROP SEGERA! Tiada binding!
        println!("Selepas let _ = ...");
    }
    // Output:
    // DROP Guard: A        ← drop SEGERA (sebelum println!)
    // Selepas let _ = ...

    println!("\n=== _nama (underscore prefix) ===");
    {
        let _b = Guard("B".into());    // Binding! Drop pada AKHIR scope
        println!("Selepas let _b = ...");
    }
    // Output:
    // Selepas let _b = ...
    // DROP Guard: B        ← drop di akhir scope

    println!("\n=== Implikasi pada Mutex ===");

    use std::sync::Mutex;
    let m = Mutex::new(0i32);

    // ❌ DEADLOCK! _ drop segera, tapi guard baru cuba lock sebelum...
    // Sebenarnya _ lock selesai SEBELUM baris seterusnya!
    let _ = m.lock().unwrap();   // lock dan TERUS DROP
    // Guard sudah dilepas, lock bebas
    let _ = m.lock().unwrap();   // ini TIDAK deadlock!

    // ❌ DEADLOCK dengan _nama:
    // let _g1 = m.lock().unwrap();  // lock, masih hidup
    // let _g2 = m.lock().unwrap();  // DEADLOCK! _g1 masih pegang lock!
}
```

---

# BAB 5: File Handle — Bila Ia Ditutup? 📁

## File Lifecycle

```rust
use std::fs;
use std::io::{self, Write, BufWriter};

fn contoh_fail_terbuka() -> io::Result<()> {
    println!("=== File Handle Lifecycle ===\n");

    // ── Scenario 1: Fail tutup bila scope tamat ───────────────
    {
        let mut f = fs::File::create("/tmp/test1.txt")?;
        writeln!(f, "Baris 1")?;
        writeln!(f, "Baris 2")?;
        println!("Fail dibuka, sedang ditulis...");
    }   // ← File::drop() dipanggil → flush + close

    println!("Fail PASTI sudah ditutup di sini");
    let k = fs::read_to_string("/tmp/test1.txt")?;
    println!("Kandungan: {}", k.trim());

    // ── Scenario 2: BufWriter — flush PENTING! ────────────────
    {
        let fail = fs::File::create("/tmp/test2.txt")?;
        let mut buf = BufWriter::new(fail);

        writeln!(buf, "Baris dalam buffer")?;
        // Data MUNGKIN masih dalam buffer, belum di-flush ke disk!

        // EKSPLISIT flush
        buf.flush()?;  // ← pastikan data sampai ke disk
        println!("Buffer di-flush");

    }   // ← BufWriter::drop() dipanggil → auto flush + close

    // ── Scenario 3: Fail dibaca oleh dua pihak (error!) ───────
    let fail_baca = fs::File::open("/tmp/test1.txt")?;
    // fail_baca masih terbuka...

    // Pada Windows, ini MUNGKIN gagal kerana fail masih terbuka!
    // Pada Linux, ini OK (boleh open berkali-kali)
    let _ = fs::read_to_string("/tmp/test1.txt");

    drop(fail_baca);  // tutup dulu sebelum operasi lain

    // ── Scenario 4: Fail dalam Error path ─────────────────────
    fn proses_fail(laluan: &str) -> io::Result<String> {
        let fail = fs::File::open(laluan)?;  // buka fail
        let mut pembaca = io::BufReader::new(fail);
        let mut kandungan = String::new();

        io::BufRead::read_line(&mut pembaca, &mut kandungan)?;

        // Kalau berlaku error di sini, fail MASIH di-drop dengan betul!
        // RAII menjamin resource cleanup walaupun ada error
        Ok(kandungan)
    }

    // Bersihkan
    let _ = fs::remove_file("/tmp/test1.txt");
    let _ = fs::remove_file("/tmp/test2.txt");

    Ok(())
}

// ── Masalah: Fail banyak terbuka ──────────────────────────────
fn proses_banyak_fail() -> io::Result<()> {
    let mut terbuka: Vec<fs::File> = Vec::new();

    // ❌ BURUK — semua fail terbuka serentak!
    for i in 0..100 {
        let fail = fs::File::open(format!("/tmp/fail_{}.txt", i))?;
        terbuka.push(fail);
    }
    // Semua 100 fail terbuka! Boleh kehabisan file descriptor!

    // ✔ BAIK — tutup setiap fail sebelum buka yang seterusnya
    for i in 0..100 {
        let fail = fs::File::open(format!("/tmp/fail_{}.txt", i))?;
        // proses fail...
        drop(fail); // ← tutup SEGERA selepas guna
    }

    Ok(())
}

fn main() -> io::Result<()> {
    contoh_fail_terbuka()
}
```

---

# BAB 6: Mutex Lock — Deadlock dari Scope Salah ⚠️

## Kes-kes Deadlock Biasa

```rust
use std::sync::{Arc, Mutex};
use std::thread;

// ── Deadlock Jenis 1: _nama prefix ─────────────────────────
fn deadlock_nama_prefix() {
    let m = Mutex::new(0i32);

    let _guard1 = m.lock().unwrap();  // lock dipegang oleh _guard1
    // _guard1 masih hidup! Scope belum tamat!

    // ❌ DEADLOCK DI SINI!
    // let _guard2 = m.lock().unwrap(); // akan block selama-lamanya!
}

// ── Deadlock Jenis 2: Terlepas pandang drop order ─────────
fn deadlock_drop_order() {
    let m1 = Mutex::new("M1");
    let m2 = Mutex::new("M2");

    // Thread 1: lock M1, kemudian M2
    let (rm1, rm2) = (Arc::new(m1), Arc::new(m2));
    let (c1, c2) = (Arc::clone(&rm1), Arc::clone(&rm2));

    // ❌ Boleh deadlock kalau dua thread lock dalam order berbeza!
    let _t1 = thread::spawn(move || {
        let _g1 = c1.lock().unwrap();
        thread::sleep(std::time::Duration::from_millis(10));
        let _g2 = c2.lock().unwrap();  // tunggu M2
    });

    // Thread 2: lock M2, kemudian M1 → DEADLOCK POTENTIAL!
    // let _t2 = thread::spawn(move || { ... lock M2 dulu, M1 kemudian });

    // _t1.join().unwrap(); // akan hang selama-lamanya!
}

// ── Penyelesaian: Scope eksplisit ─────────────────────────
fn elak_deadlock_scope() {
    let m = Mutex::new(0i32);

    // ✔ CARA 1: Scope eksplisit
    {
        let mut guard = m.lock().unwrap();
        *guard += 1;
        println!("Dalam scope: {}", *guard);
    }  // ← guard di-drop di sini, lock dilepas

    // ✔ CARA 2: drop() manual
    let mut guard = m.lock().unwrap();
    *guard += 1;
    let nilai = *guard;
    drop(guard);  // ← drop segera selepas ambil nilai

    println!("Nilai: {}", nilai);

    // ✔ CARA 3: Guna block expression
    let hasil = {
        let guard = m.lock().unwrap();
        let v = *guard;
        v  // ← guard di-drop bila block tamat, SEBELUM return
    };
    println!("Hasil: {}", hasil);

    // ✔ CARA 4: Immediate dereference
    let nilai = *m.lock().unwrap();  // ← guard drop selepas dereference
    println!("Terus: {}", nilai);
}

// ── Masalah dengan if let / while let ─────────────────────
fn lock_dalam_if_let() {
    let m = Mutex::new(Some(42i32));

    // ❌ Guard hidup sepanjang if let block!
    if let Some(v) = *m.lock().unwrap() {
        println!("Nilai: {}", v);
        // Lock MASIH dipegang di sini!
        // Kalau buat sesuatu yang perlu lock semula → DEADLOCK
    }  // ← guard baru di-drop di sini (bila if block tamat)

    // ✔ Ambil nilai dulu, kemudian proses
    let nilai_opt = *m.lock().unwrap();  // ← guard drop lepas ini
    if let Some(v) = nilai_opt {
        println!("Nilai: {}", v);
        // Lock sudah dilepas, boleh lock semula kalau perlu
    }
}

fn main() {
    elak_deadlock_scope();
    lock_dalam_if_let();
}
```

---

## Pattern Lock yang Betul

```rust
use std::sync::{Arc, Mutex, RwLock};
use std::collections::HashMap;

struct KonfigApp {
    data: RwLock<HashMap<String, String>>,
}

impl KonfigApp {
    fn baru() -> Self {
        KonfigApp { data: RwLock::new(HashMap::new()) }
    }

    // ✔ BETUL: ambil nilai, lepas lock, return nilai
    fn dapatkan(&self, kunci: &str) -> Option<String> {
        self.data.read().unwrap()
            .get(kunci)
            .cloned()  // ← clone String sebelum guard drop
        // guard drop di sini (akhir expression chain)
    }

    // ✔ BETUL: lock, ubah, drop
    fn tetapkan(&self, kunci: &str, nilai: &str) {
        self.data.write().unwrap()
            .insert(kunci.into(), nilai.into());
        // write guard drop di sini
    }

    // ❌ BAHAYA: return reference ke dalam locked data
    // fn dapatkan_ref(&self, kunci: &str) -> Option<&String> {
    //     self.data.read().unwrap().get(kunci)
    //     // ↑ Guard di-drop tapi &String masih digunakan!
    //     // Compiler akan tangkap ini — lifetime error
    // }

    // ✔ BETUL: kalau perlu buat banyak operasi atom
    fn kemas_banyak(&self, data: Vec<(&str, &str)>) {
        let mut guard = self.data.write().unwrap();
        for (k, v) in data {
            guard.insert(k.into(), v.into());
        }
        // guard drop di sini — satu lock untuk semua operasi!
    }
}
```

---

# BAB 7: Database Connection — Lifecycle Pool 🗄️

## Connection Pool RAII

```rust
use std::sync::{Arc, Mutex};
use std::collections::VecDeque;

// Simulasi database connection
struct Sambungan {
    id:     u32,
    url:    String,
    aktif:  bool,
}

impl Sambungan {
    fn baru(id: u32, url: &str) -> Self {
        println!("[Sambungan #{}] Dibuka ke: {}", id, url);
        Sambungan { id, url: url.into(), aktif: true }
    }

    fn query(&self, sql: &str) -> String {
        format!("[Sambungan #{}] {}", self.id, sql)
    }
}

impl Drop for Sambungan {
    fn drop(&mut self) {
        if self.aktif {
            println!("[Sambungan #{}] Ditutup (tidak dikembalikan ke pool!)", self.id);
        }
    }
}

// Pool yang urus lifecycle sambungan
struct Pool {
    url:      String,
    tersedia: Arc<Mutex<VecDeque<Sambungan>>>,
    id_seter: Arc<Mutex<u32>>,
}

// Guard yang auto-kembalikan sambungan ke pool
struct SambunganPool<'a> {
    sambungan: Option<Sambungan>,
    pool:      &'a Pool,
}

impl<'a> std::ops::Deref for SambunganPool<'a> {
    type Target = Sambungan;
    fn deref(&self) -> &Sambungan {
        self.sambungan.as_ref().unwrap()
    }
}

impl<'a> Drop for SambunganPool<'a> {
    fn drop(&mut self) {
        if let Some(mut s) = self.sambungan.take() {
            println!("[Pool] Sambungan #{} dikembalikan ke pool", s.id);
            s.aktif = false; // tandakan sudah dalam pool (elak drop message)
            self.pool.tersedia.lock().unwrap().push_back(s);
        }
    }
}

impl Pool {
    fn baru(url: &str, saiz: u32) -> Self {
        let pool = Pool {
            url:      url.into(),
            tersedia: Arc::new(Mutex::new(VecDeque::new())),
            id_seter: Arc::new(Mutex::new(0)),
        };

        let mut q = pool.tersedia.lock().unwrap();
        let mut id = pool.id_seter.lock().unwrap();
        for _ in 0..saiz {
            *id += 1;
            q.push_back(Sambungan::baru(*id, url));
        }
        drop(q);
        drop(id);

        pool
    }

    fn dapatkan(&self) -> SambunganPool<'_> {
        let mut q = self.tersedia.lock().unwrap();
        let sambungan = if let Some(s) = q.pop_front() {
            println!("[Pool] Sambungan #{} diambil dari pool", s.id);
            s
        } else {
            let mut id = self.id_seter.lock().unwrap();
            *id += 1;
            let baru_id = *id;
            drop(id);
            drop(q);
            println!("[Pool] Buat sambungan baru");
            Sambungan::baru(baru_id, &self.url)
        };

        SambunganPool { sambungan: Some(sambungan), pool: self }
    }

    fn saiz_tersedia(&self) -> usize {
        self.tersedia.lock().unwrap().len()
    }
}

fn main() {
    let pool = Pool::baru("mysql://localhost/kada_db", 3);
    println!("Pool tersedia: {}\n", pool.saiz_tersedia());

    {
        let conn = pool.dapatkan();
        let hasil = conn.query("SELECT * FROM pekerja");
        println!("{}", hasil);
        println!("Pool tersedia semasa guna: {}", pool.saiz_tersedia());
    }  // ← SambunganPool::drop() → sambungan dikembalikan ke pool!

    println!("\nPool tersedia selepas: {}", pool.saiz_tersedia());
}
```

---

# BAB 8: Guard Pattern 🛡️

## RAII Guard untuk Custom Resources

```rust
use std::sync::atomic::{AtomicBool, Ordering};

// ── Fail Lock Guard ────────────────────────────────────────
struct KunciFailGuard {
    laluan_kunci: String,
}

impl KunciFailGuard {
    fn kunci(laluan_fail: &str) -> Option<Self> {
        let laluan_kunci = format!("{}.lock", laluan_fail);

        // Cuba buat fail kunci
        if std::path::Path::new(&laluan_kunci).exists() {
            println!("Fail sudah dikunci oleh proses lain!");
            return None;
        }

        std::fs::write(&laluan_kunci, std::process::id().to_string()).ok()?;
        println!("Fail dikunci: {}", laluan_kunci);
        Some(KunciFailGuard { laluan_kunci })
    }
}

impl Drop for KunciFailGuard {
    fn drop(&mut self) {
        let _ = std::fs::remove_file(&self.laluan_kunci);
        println!("Kunci dilepas: {}", self.laluan_kunci);
    }
}

// ── Progress Guard ─────────────────────────────────────────
struct ProgressGuard {
    nama:    String,
    bermula: std::time::Instant,
}

impl ProgressGuard {
    fn mula(nama: &str) -> Self {
        println!("[MULA] {}", nama);
        ProgressGuard { nama: nama.into(), bermula: std::time::Instant::now() }
    }
}

impl Drop for ProgressGuard {
    fn drop(&mut self) {
        println!("[SIAP] {} ({}ms)",
            self.nama,
            self.bermula.elapsed().as_millis()
        );
    }
}

// ── Guna Guards ────────────────────────────────────────────
fn proses_dengan_guard() {
    let _progres = ProgressGuard::mula("Proses data");

    // Walaupun ada return awal atau panic, progres akan report!

    std::thread::sleep(std::time::Duration::from_millis(50));

    println!("Sedang proses...");

    // _progres di-drop di sini → "[SIAP] Proses data (50ms)"
}

fn main() {
    proses_dengan_guard();

    // File lock
    if let Some(_kunci) = KunciFailGuard::kunci("/tmp/operasi_kritikal") {
        println!("Operasi kritikal sedang berjalan...");
        std::thread::sleep(std::time::Duration::from_millis(100));
        // _kunci di-drop → kunci dilepas automatik
    } else {
        println!("Tidak dapat kunci!");
    }
}
```

---

## 🧠 Brain Teaser #2

Apakah bug dalam kod ini, dan bagaimana betulkan?

```rust
use std::sync::Mutex;

struct Cache {
    data:  Mutex<std::collections::HashMap<String, String>>,
}

impl Cache {
    fn dapatkan_atau_kira(&self, kunci: &str) -> String {
        // Cuba ambil dari cache
        if let Some(v) = self.data.lock().unwrap().get(kunci) {
            return v.clone();
        }

        // Tidak dalam cache — kira
        let nilai = format!("dikira:{}", kunci.len());

        // Simpan dalam cache
        self.data.lock().unwrap().insert(kunci.into(), nilai.clone());
        //         ^^^^ Adakah ini masalah?

        nilai
    }
}
```

<details>
<summary>👀 Jawapan</summary>

```rust
// Kod asal sebenarnya BETUL tapi untuk alasan yang tidak obvious!
//
// Baris 1: self.data.lock().unwrap().get(kunci)
//   → Temporary MutexGuard dibuat, .get() dipanggil, guard DROP
//     pada AKHIR statement (selepas semicolon)
//   → Apabila kita return v.clone(), guard sudah drop sebelum return
//
// Baris 2: self.data.lock().unwrap().insert(...)
//   → Lock baru dipegang, insert, guard DROP
//
// Ini TIDAK deadlock kerana guard pertama sudah drop
// sebelum guard kedua dibuat.

// TAPI ada race condition dalam multi-thread!
// Antara "tidak dalam cache" check dan "simpan dalam cache",
// thread lain boleh masuk dan kira juga!
// (check-then-act race condition)

// Penyelesaian BETUL:
fn dapatkan_atau_kira_selamat(&self, kunci: &str) -> String {
    // Semak dulu tanpa lock panjang
    if let Some(v) = self.data.lock().unwrap().get(kunci).cloned() {
        return v;
    }

    // Kira di luar lock (operasi mahal tidak block thread lain)
    let nilai = format!("dikira:{}", kunci.len());

    // Lock semula untuk simpan
    let mut guard = self.data.lock().unwrap();

    // Semak semula (mungkin thread lain sudah simpan)
    guard.entry(kunci.into()).or_insert_with(|| nilai.clone());

    guard.get(kunci).unwrap().clone()
}

// atau lebih elegan:
fn dapatkan_atau_kira_v2(&self, kunci: &str) -> String {
    let kira = || format!("dikira:{}", kunci.len());

    self.data.lock().unwrap()
        .entry(kunci.into())
        .or_insert_with(kira)
        .clone()
    // lock di-drop selepas .clone()
}
```
</details>

---

# BAB 9: Drop dalam Async — Perangkap Tersembunyi ⚡

## Async Drop dan Masalahnya

```rust
use tokio::sync::Mutex as AsyncMutex;

// ── Masalah: Drop tidak async ─────────────────────────────
// Rust tidak ada "async drop" — Drop::drop() adalah synchronous!
// Ini bermaksud: .await dalam Drop adalah TIDAK BOLEH

struct KoneksiAsync {
    pool: String,
}

impl Drop for KoneksiAsync {
    fn drop(&mut self) {
        // ❌ TIDAK BOLEH: .await dalam drop!
        // runtime.block_on(kembalikan_ke_pool(self)) // PANIC!

        // ✔ Boleh: hantar ke channel
        println!("Koneksi dikembalikan ke pool (synchronous)");
    }
}

// ── Masalah: MutexGuard merentasi .await ──────────────────
async fn contoh_bahaya() {
    let mutex = AsyncMutex::new(0i32);

    // ❌ BAHAYA dalam async!
    // let guard = mutex.lock().await;
    // some_async_fn().await;  // ← guard masih ada! Ini MUNGKIN OK
    //                              dengan Tokio tapi tidak baik
    //                              dalam context lain

    // ✔ CARA BETUL: lepas lock sebelum .await
    {
        let mut guard = mutex.lock().await;
        *guard += 1;
    }  // ← guard di-drop di sini

    tokio::time::sleep(std::time::Duration::from_millis(10)).await;

    let guard = mutex.lock().await;
    println!("Nilai: {}", *guard);
}

// ── select! dan drop ──────────────────────────────────────
async fn contoh_select() {
    use tokio::sync::mpsc;

    let (tx, mut rx) = mpsc::channel::<String>(10);

    tokio::select! {
        Some(mesej) = rx.recv() => {
            println!("Dapat: {}", mesej);
            // tx masih hidup di sini — tidak di-drop dalam branch ini!
        }
        _ = tokio::time::sleep(std::time::Duration::from_secs(1)) => {
            println!("Timeout!");
        }
    }
    // tx di-drop SELEPAS select! block tamat
    // (kecuali ia di-move ke dalam branch)
}

// ── Pattern: Cleanup yang perlu async ─────────────────────
struct SumberAsync {
    id: u32,
}

impl SumberAsync {
    async fn bersih(&mut self) {
        tokio::time::sleep(std::time::Duration::from_millis(10)).await;
        println!("Bersih async #{}", self.id);
    }
}

// Drop biasa (synchronous)
impl Drop for SumberAsync {
    fn drop(&mut self) {
        // Tidak boleh .await di sini!
        println!("Drop sync #{} (cleanup akan pending!)", self.id);
    }
}

// ✔ Penyelesaian: Explicit cleanup method
async fn guna_sumber() {
    let mut sumber = SumberAsync { id: 1 };
    // ... guna sumber ...
    sumber.bersih().await;  // ← explicit cleanup SEBELUM drop
    // drop(sumber) dipanggil selepas ini (synchronous sahaja)
}
```

---

# BAB 10: Mini Project — Resource Manager 🏗️

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::time::{Instant, Duration};

// ─── Resource Registry ────────────────────────────────────────

trait Sumber: Send + Sync {
    fn nama(&self) -> &str;
    fn bersih(&mut self);
    fn sihat(&self) -> bool;
}

// ─── File Resource ────────────────────────────────────────────

struct SumberFail {
    nama:  String,
    laluan: String,
    tulis: std::fs::File,
}

impl SumberFail {
    fn buka(laluan: &str) -> std::io::Result<Self> {
        let fail = std::fs::OpenOptions::new()
            .create(true).append(true).open(laluan)?;
        println!("[SumberFail] Dibuka: {}", laluan);
        Ok(SumberFail {
            nama:   format!("Fail:{}", laluan),
            laluan: laluan.into(),
            tulis:  fail,
        })
    }

    fn log(&mut self, mesej: &str) -> std::io::Result<()> {
        use std::io::Write;
        writeln!(self.tulis, "[{}] {}", chrono_mudah(), mesej)
    }
}

impl Sumber for SumberFail {
    fn nama(&self) -> &str { &self.nama }
    fn bersih(&mut self) {
        use std::io::Write;
        let _ = self.tulis.flush();
        println!("[SumberFail] Di-flush: {}", self.laluan);
    }
    fn sihat(&self) -> bool { true }
}

impl Drop for SumberFail {
    fn drop(&mut self) {
        self.bersih();
        println!("[SumberFail] Ditutup: {}", self.laluan);
    }
}

// ─── Timed Resource ───────────────────────────────────────────

struct SumberMasa {
    nama:     String,
    mula:     Instant,
    had_ms:   u64,
}

impl SumberMasa {
    fn baru(nama: &str, had_ms: u64) -> Self {
        println!("[SumberMasa] Pemasa '{}' ({}ms) dimulakan", nama, had_ms);
        SumberMasa { nama: nama.into(), mula: Instant::now(), had_ms }
    }

    fn masa_berlalu_ms(&self) -> u64 {
        self.mula.elapsed().as_millis() as u64
    }
}

impl Sumber for SumberMasa {
    fn nama(&self) -> &str { &self.nama }
    fn bersih(&mut self) {}
    fn sihat(&self) -> bool {
        self.masa_berlalu_ms() <= self.had_ms
    }
}

impl Drop for SumberMasa {
    fn drop(&mut self) {
        let masa = self.masa_berlalu_ms();
        if masa > self.had_ms {
            println!("[SumberMasa] ⚠️  '{}' tamat masa! {}ms > {}ms",
                self.nama, masa, self.had_ms);
        } else {
            println!("[SumberMasa] '{}' selesai dalam {}ms", self.nama, masa);
        }
    }
}

// ─── Resource Manager ─────────────────────────────────────────

struct PengursSumber {
    sumber: HashMap<String, Box<dyn Sumber>>,
}

impl PengursSumber {
    fn baru() -> Self {
        PengursSumber { sumber: HashMap::new() }
    }

    fn daftar(&mut self, sumber: Box<dyn Sumber>) {
        let nama = sumber.nama().to_string();
        println!("[Pengurus] Daftar: {}", nama);
        self.sumber.insert(nama, sumber);
    }

    fn buang(&mut self, nama: &str) {
        if let Some(mut s) = self.sumber.remove(nama) {
            s.bersih();  // bersih sebelum drop
            println!("[Pengurus] Buang: {}", nama);
            // s di-drop di sini → Drop trait dipanggil
        }
    }

    fn semak_kesihatan(&self) {
        println!("\n[Pengurus] Laporan Kesihatan:");
        for (nama, sumber) in &self.sumber {
            let status = if sumber.sihat() { "✔ Sihat" } else { "✗ Masalah" };
            println!("  {} — {}", nama, status);
        }
    }

    fn bilangan(&self) -> usize { self.sumber.len() }
}

impl Drop for PengursSumber {
    fn drop(&mut self) {
        println!("\n[Pengurus] Menutup {} sumber...", self.sumber.len());
        // Bersihkan semua sumber
        for (nama, sumber) in &mut self.sumber {
            sumber.bersih();
            println!("[Pengurus] Bersih: {}", nama);
        }
        // sumber di-drop automatik bila HashMap di-drop
        println!("[Pengurus] Semua sumber ditutup.");
    }
}

fn chrono_mudah() -> String {
    format!("{:?}", std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .unwrap()
        .as_secs())
}

// ─── Demo ─────────────────────────────────────────────────────

fn main() {
    println!("{'═'*50}");
    println!("  Resource Manager Demo");
    println!("{'═'*50}\n");

    {
        let mut pengurus = PengursSumber::baru();

        // Daftar pelbagai jenis sumber
        pengurus.daftar(Box::new(
            SumberMasa::baru("Operasi-1", 5000)
        ));
        pengurus.daftar(Box::new(
            SumberMasa::baru("Operasi-2", 1000)
        ));

        println!("\nBilangan sumber: {}", pengurus.bilangan());

        // Simulate some work
        std::thread::sleep(Duration::from_millis(100));

        pengurus.semak_kesihatan();

        // Buang satu sumber secara manual
        pengurus.buang("SumberMasa:Operasi-1");

        println!("\nSelepas buang: {} sumber", pengurus.bilangan());

        // Simulate more work
        std::thread::sleep(Duration::from_millis(200));

        println!("\nKeluar scope pengurus...");

    }   // ← PengursSumber::drop() dipanggil:
        //   → bersih semua sumber
        //   → drop semua sumber (Drop trait)

    println!("\n{'═'*50}");
    println!("  Semua resource PASTI dilepaskan!");
    println!("{'═'*50}");
}
```

---

# 📋 Rujukan Pantas — Drop & Lifecycle Cheat Sheet

## Drop Order

```
Variables dalam scope:   LIFO (last in, first out)
  let a = ...; let b = ...; let c = ...;
  → drop: c, b, a

Struct fields:           Declaration order
  struct S { x: T, y: T, z: T }
  → drop: x, y, z

Tuple/Array elements:    Declaration order
  (A, B, C)  → drop: A, B, C
  [A, B, C]  → drop: A, B, C
```

## Simbol & Semantik

```rust
let x = expr;       // Binding — drop bila scope tamat
let _ = expr;       // NO binding — drop SEGERA (serta-merta!)
let _x = expr;      // Binding — drop bila scope tamat (SAMA seperti let x!)
drop(x);            // Drop manual — drop segera, sebelum scope tamat
```

## Pattern untuk Resource Selamat

```rust
// ✔ File — tutup sebelum baca
{
    let mut f = File::create("x.txt")?;
    write!(f, "...")?;
} // f ditutup di sini

// ✔ Mutex — lepas lock sebelum await
let val = {
    let g = mutex.lock().unwrap();
    *g  // copy value
};  // guard drop di sini

// ✔ DB connection — kembalikan ke pool
let conn = pool.get().await?;
let result = conn.query("...").await?;
drop(conn); // kembalikan sebelum proses result

// ✔ Guard pattern
let _timer = ProgressGuard::mula("Operasi");
// ... _timer report bila drop
```

## Perangkap Biasa

```
❌ _guard = mutex.lock()   → masih hidup sampai scope!
✔  _ = mutex.lock()        → drop segera (berguna untuk OnceLock)
✔  drop(_guard)             → drop manual bila dah tak perlu

❌ Lock merentasi .await   → mungkin deadlock dalam async
✔  Lepas lock dulu, baru .await

❌ Return reference dalam lock → lifetime error
✔  Clone/copy nilai, lepas lock, return nilai

❌ .await dalam Drop       → TIDAK boleh!
✔  Explicit cleanup sebelum drop
```

---

## 🏆 Cabaran Akhir

Cuba implement salah satu:

1. **Scoped Thread Pool** — thread pool yang tunggu semua thread selesai dalam Drop
2. **Transaction Guard** — DB transaction yang commit dalam Drop kalau ok, rollback kalau ada error
3. **Tempfile Manager** — fail temp yang auto-padam dan handle failure dalam Drop
4. **Connection Limiter** — semaphore yang track berapa connection terbuka, auto-release dalam Drop
5. **Timed Lock** — mutex yang auto-release selepas timeout melalui background thread

---

*Drop dalam Rust bukan hanya "pembersihan memori" —*
*ia adalah jaminan bahawa SETIAP resource dilepaskan pada MASA yang TEPAT.*
*Fahami scope, faham bila resource hidup, elak deadlock dan resource leak.* 🦀
