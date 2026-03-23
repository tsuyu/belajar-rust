# 🧠 Thinking in Rust sebagai PHP Developer

> "Masalah bukan belajar Rust. Masalah adalah UNLEARING PHP."
> Panduan untuk tukar mental model dari PHP ke Rust —
> bukan terjemahan, tapi cara fikir yang berbeza.

---

## Realiti yang Perlu Diterima Dulu

```
PHP Developer membawa MENTAL MODEL ini ke Rust:
  ✓ "Variable adalah bekas — letak nilai, pakai bila perlu"
  ✓ "Function terima apa sahaja, return apa sahaja"
  ✓ "Error? Try-catch. Atau senyap kalau tidak penting"
  ✓ "null bermaksud 'tiada nilai' — check kalau penting"
  ✓ "Array boleh simpan apa sahaja"
  ✓ "Type? Berguna untuk IDE, bukan wajib"

Model ini TIDAK BERFUNGSI dalam Rust.
Bukan kerana Rust betul dan PHP salah —
kerana TUJUAN kedua-duanya berbeza.

PHP:  Cepat buat, cepat hantar, server restart bila crash
Rust: Tidak crash, tidak leak, tidak race — dari asas
```

---

## Peta Perjalanan

```
Fasa 1 — Terjemahan Terus (Biasanya Salah)
Fasa 2 — Faham KENAPA Rust buat begini
Fasa 3 — Fikir dalam Rust (bukan terjemah dari PHP)

Kandungan:
  Bab 1  → Variable dan Assignment
  Bab 2  → Types — Dari Longgar ke Ketat
  Bab 3  → Array PHP → Rust
  Bab 4  → null → Option<T>
  Bab 5  → Exception → Result<T, E>
  Bab 6  → Class → Struct + Impl + Trait
  Bab 7  → Reference PHP → Borrow Rust
  Bab 8  → Fungsi PHP → Fungsi Rust
  Bab 9  → Closures dan Callbacks
  Bab 10 → Mental Model Akhir
```

---

# BAB 1: Variable dan Assignment 📦

## PHP — "Bekas yang Boleh Tukar Isi"

```php
// PHP: variable = label untuk nilai
$x = 5;
$x = "hello";     // OK! Boleh tukar type
$x = [1, 2, 3];   // OK! Boleh tukar lagi
$x = null;         // OK! Kosongkan

// PHP: assign = COPY nilai (untuk primitif)
$a = 5;
$b = $a;   // b adalah salinan
$b = 10;
echo $a;   // 5 — a tidak terjejas

// PHP: assign object = SHARE reference
$obj1 = new Pekerja("Ali");
$obj2 = $obj1;     // SAMA object!
$obj2->nama = "Siti";
echo $obj1->nama;  // "Siti" — kedua-dua terjejas!
```

## Rust — "Nama untuk Nilai Dengan Peraturan"

```rust
// Rust: variable = nama untuk nilai, dengan OWNERSHIP
let x = 5;
// let x = "hello"; // ERROR! x sudah i32
// x = 10;          // ERROR! x tidak mutable

// Untuk ubah:
let mut y = 5;
y = 10;  // OK
// y = "hello"; // ERROR! Tidak boleh tukar type

// Shadowing (bukan assignment):
let x = 5;
let x = "hello";  // OK! Ini variable BARU bernama x
let x = x.len();  // OK! x BARU lagi
println!("{}", x); // 5 (panjang "hello")

// ── Mental model shift ────────────────────────────────────────
// PHP: variable adalah label, nilai boleh berubah bebas
// Rust: variable adalah NAMA untuk satu nilai,
//       nilai ikut peraturan ketat

// PHP boleh:  $a = 5; $a = "hello"; $a = null;
// Rust tidak: let a = 5; a = "hello"; a = None;

// Rust boleh: let a = 5; let a = "hello"; let a = 0; (shadowing)
//             tapi ini VARIABLE BARU setiap kali!
```

---

## PHP Reference vs Rust Borrow

```php
// PHP reference: &$var = alias kepada nilai yang sama
function tambah(&$nilai) {
    $nilai += 10;  // ubah nilai asal!
}

$x = 5;
tambah($x);
echo $x; // 15
```

```rust
// Rust borrow: &mut val = pinjaman yang dikawal
fn tambah(nilai: &mut i32) {
    *nilai += 10;  // dereference untuk ubah
}

let mut x = 5;
tambah(&mut x);
println!("{}", x); // 15

// Perbezaan KRITIKAL:
// PHP: boleh ada banyak reference mutable serentak
// Rust: HANYA SATU mutable reference pada satu masa
//       ATAU banyak immutable reference
//       TIDAK KEDUA-DUANYA serentak

// PHP:
// $a = &$x;
// $b = &$x;  // OK — dua alias mutable serentak
// $a = 5; $b = 10; // konflik senyap!

// Rust:
// let a = &mut x;
// let b = &mut x;  // ERROR! cannot borrow mutably twice
```

---

## 🧠 Brain Teaser #1

PHP developer menulis ini dalam Rust. Apa masalahnya?

```rust
fn main() {
    let senarai = vec![1, 2, 3, 4, 5];
    let jumlah = kira_jumlah(senarai);
    println!("Senarai: {:?}", senarai); // cuba print selepas pass ke fungsi
    println!("Jumlah: {}", jumlah);
}

fn kira_jumlah(v: Vec<i32>) -> i32 {
    v.iter().sum()
}
```

<details>
<summary>👀 Jawapan</summary>

```
ERROR: borrow of moved value: `senarai`

Masalah: kira_jumlah(senarai) — senarai DIPINDAH ke fungsi!
         Selepas itu, senarai tidak wujud lagi dalam main().

PHP Mental Model (SALAH):
  "Saya pass $senarai ke fungsi, tapi $senarai masih ada"
  → Dalam PHP, array di-COPY apabila dipass ke fungsi (kecuali &)
  → Jadi PHP developer fikir senarai masih ada

Rust Reality:
  Vec<i32> adalah MOVE type — ownership berpindah ke kira_jumlah
  Selepas return, senarai tidak wujud

Penyelesaian 1: Guna borrow (&)
fn kira_jumlah(v: &[i32]) -> i32 {  // pinjam, bukan ambil
    v.iter().sum()
}
// Panggil: kira_jumlah(&senarai)
// senarai masih valid selepas itu!

Penyelesaian 2: Clone (kalau betul-betul perlu ownership)
let jumlah = kira_jumlah(senarai.clone());
// Mahal tapi senarai masih ada

Penyelesaian 3: Return balik (kalau fungsi perlu ownership)
fn kira_jumlah(v: Vec<i32>) -> (Vec<i32>, i32) {
    let jumlah = v.iter().sum();
    (v, jumlah)  // kembalikan ownership
}
let (senarai, jumlah) = kira_jumlah(senarai);
```
</details>

---

# BAB 2: Types — Dari Longgar ke Ketat 🔒

## PHP — "Type adalah Cadangan"

```php
// PHP: type adalah loosely enforced
function bahagi($a, $b) {
    return $a / $b;
}

echo bahagi(10, 2);     // 5
echo bahagi(10, 0);     // Warning + INF — tidak crash!
echo bahagi("10", "2"); // OK! PHP convert string ke number
echo bahagi([1,2], 2);  // Warning, return null

// PHP type declaration (boleh):
function bahagi_ketat(int $a, int $b): float {
    return $a / $b;
}
// Tapi masih: bahagi_ketat(10, 0) → Warning, tidak crash
```

## Rust — "Type adalah Kontrak"

```rust
// Rust: type adalah kontrak yang WAJIB dipatuhi
fn bahagi(a: f64, b: f64) -> f64 {
    a / b
    // Ini tidak crash untuk 0.0 — return f64::INFINITY
}

// Tapi untuk integer:
fn bahagi_int(a: i32, b: i32) -> i32 {
    a / b  // PANIC jika b == 0! (integer division by zero)
}

// Cara Rust yang betul:
fn bahagi_selamat(a: i32, b: i32) -> Option<i32> {
    if b == 0 { None } else { Some(a / b) }
}

fn main() {
    // bahagi_selamat("10", "2"); // COMPILE ERROR — type salah!

    match bahagi_selamat(10, 0) {
        Some(n) => println!("{}", n),
        None    => println!("Tidak boleh bahagi dengan sifar"),
    }
}

// ── Type conversion dalam PHP vs Rust ────────────────────────

// PHP: auto-convert (banyak magic)
// "5" + 3 = 8
// "5abc" + 3 = 8 (ambil prefix nombor)
// [] + [] = []
// [] + {} = ["0" => null]  ← ini benar!

// Rust: TIADA auto-convert, semua explicit
let n: i32 = 5;
let f: f64 = 3.14;
// let r = n + f; // ERROR! i32 + f64 tidak boleh

let r = n as f64 + f; // explicit cast
let r = f64::from(n) + f; // atau guna From trait

// String ke nombor:
let s = "42";
let n: i32 = s.parse().unwrap(); // explicit parse
// let n: i32 = s; // ERROR!
```

---

# BAB 3: Array PHP → Rust 🗃️

## PHP Array — Swiss Army Knife

```php
// PHP array boleh jadi SEMUA:
$list    = [1, 2, 3];                // list biasa
$hashmap = ["nama" => "Ali", "umur" => 25]; // associative
$mixed   = [1, "hello", true, null, [1,2]]; // campur semua!
$sparse  = [0 => "pertama", 5 => "keenam"]; // indeks bukan berturut

// PHP array sangat powerful tapi tiada type safety!
$a = ["nama" => "Ali"];
$a["umur"] = 25;
$a["umur"] = "dua puluh lima"; // OK! Tukar type dalam array
$a[999] = "random key"; // OK!

// Akses:
echo $a["nama"];    // "Ali"
echo $a["tiada"];   // Warning: Undefined index — kemudian NULL
echo $a["tiada"]["lagi"]; // PHP: NULL — tidak crash!
                          // Rust: compile error!
```

## Rust — Setiap Keperluan Ada Jenis Sendiri

```rust
use std::collections::HashMap;

// PHP $list = [1,2,3]  →  Rust Vec<T>
let list: Vec<i32> = vec![1, 2, 3];

// PHP $hash = ["a" => 1]  →  Rust HashMap<K, V>
let mut map: HashMap<&str, i32> = HashMap::new();
map.insert("nama_len", 3);
map.insert("umur", 25);

// Tidak boleh campur type dalam Vec!
// let campur = vec![1, "hello", true]; // COMPILE ERROR!

// Kalau nak campur → guna enum
#[derive(Debug)]
enum Nilai {
    Teks(String),
    Nombor(i64),
    Logik(bool),
    Kosong,
}

let campur: Vec<Nilai> = vec![
    Nilai::Nombor(1),
    Nilai::Teks("hello".into()),
    Nilai::Logik(true),
    Nilai::Kosong,
];

// ── Akses Array yang Selamat ──────────────────────────────────

// PHP: $a["tiada"] → NULL (warning, tidak crash)
// Rust: map["tiada"] → PANIC!

// Cara selamat dalam Rust:
let nama = map.get("nama");    // Option<&i32>
match nama {
    Some(n) => println!("Ada: {}", n),
    None    => println!("Tidak ada"),
}

// Atau dengan unwrap_or:
let n = map.get("tiada").unwrap_or(&0); // default 0 kalau tiada

// ── Perbandingan Operasi Array ────────────────────────────────

// PHP array_push → Vec::push
let mut v = vec![1, 2, 3];
v.push(4);

// PHP array_pop → Vec::pop (return Option!)
let terakhir = v.pop(); // Option<i32>

// PHP count → Vec::len
let bil = v.len(); // usize

// PHP array_map → Iterator::map
let doubled: Vec<i32> = v.iter().map(|&n| n * 2).collect();

// PHP array_filter → Iterator::filter
let positif: Vec<&i32> = v.iter().filter(|&&n| n > 0).collect();

// PHP array_reduce → Iterator::fold
let jumlah: i32 = v.iter().sum(); // atau .fold(0, |acc, &n| acc + n)

// PHP in_array → Vec::contains
let ada = v.contains(&2); // bool

// PHP array_keys → HashMap::keys
let kunci: Vec<&&str> = map.keys().collect();

// PHP array_merge → tidak langsung, guna extend atau chain
let mut a = vec![1, 2];
let b = vec![3, 4];
a.extend(b); // gabung
```

---

# BAB 4: null → Option\<T\> 🚫

## PHP null — Senyap dan Berbahaya

```php
// PHP null = ada di mana-mana
function cariPekerja(int $id): ?Pekerja {
    if ($id === 1) return new Pekerja("Ali");
    return null; // boleh!
}

$p = cariPekerja(999);
echo $p->getNama(); // FATAL ERROR: Call to member on null
// → Server crash, atau white screen
// → User nampak error di production!

// Defensive (tapi mudah lupa):
$p = cariPekerja(999);
if ($p !== null) {
    echo $p->getNama(); // selamat
}

// PHP 8 nullsafe operator:
echo $p?->getNama(); // OK, return null kalau $p null
// Tapi "null" dalam context lain masih bahaya!
```

## Rust Option — Compiler Paksa Handle

```rust
struct Pekerja { nama: String }

fn cari_pekerja(id: u32) -> Option<Pekerja> {
    if id == 1 {
        Some(Pekerja { nama: "Ali".into() })
    } else {
        None
    }
}

fn main() {
    let p = cari_pekerja(999);

    // println!("{}", p.nama); // COMPILE ERROR! p adalah Option, bukan Pekerja

    // MESTI handle:
    match p {
        Some(pekerja) => println!("{}", pekerja.nama),
        None          => println!("Tidak dijumpai"),
    }

    // Atau shorthand:
    if let Some(pekerja) = cari_pekerja(1) {
        println!("{}", pekerja.nama);
    }

    // Atau dengan default:
    let nama = cari_pekerja(1)
        .map(|p| p.nama)
        .unwrap_or_else(|| "Pekerja tidak diketahui".into());

    // PHP nullsafe ?-> dalam Rust = and_then
    // PHP: $p?->getBahagian()?->getNama()
    // Rust:
    let nama_bahagian = cari_pekerja(1)
        .and_then(|p| p.bahagian())   // Option<Bahagian>
        .map(|b| b.nama().to_string()); // Option<String>
}
```

---

## Terjemahan Null Pattern

```
PHP                              Rust
─────────────────────────────────────────────────────────────
$x = null                        let x: Option<T> = None
$x = new Pekerja()               let x: Option<Pekerja> = Some(Pekerja::baru())
if ($x !== null)                 if let Some(p) = x
$x?->method()                   x.as_ref().and_then(|x| x.method())
$x ?? "default"                  x.unwrap_or("default")
isset($x)                        x.is_some()
is_null($x)                      x.is_none()
```

---

# BAB 5: Exception → Result\<T, E\> 🚨

## PHP Exception — "Lontarkan Masalah ke Atas"

```php
// PHP: exception boleh berlaku di mana-mana, tiada dalam signature
function bacaFail(string $laluan): string {
    $kandungan = file_get_contents($laluan);
    if ($kandungan === false) {
        throw new RuntimeException("Gagal baca fail: $laluan");
    }
    return $kandungan;
}

// Caller tidak TAHU bahawa bacaFail boleh throw!
// Kena baca docs, atau tahu dari pengalaman
try {
    $data = bacaFail("data.txt");
    $nombor = (int) $data;
    if ($nombor < 0) throw new InvalidArgumentException("Mesti positif");
    echo $nombor;
} catch (RuntimeException $e) {
    echo "IO Error: " . $e->getMessage();
} catch (InvalidArgumentException $e) {
    echo "Nilai salah: " . $e->getMessage();
} catch (Exception $e) {
    echo "Ralat: " . $e->getMessage();
}
// Masalah: bagaimana tahu exception apa yang boleh berlaku?
// Jawapan dalam PHP: baca docs, atau tahu dari pengalaman pahit
```

## Rust Result — "Ralat Adalah Nilai Pertama Kelas"

```rust
use std::io;
use std::num::ParseIntError;
use thiserror::Error;

// Error adalah sebahagian daripada type signature!
#[derive(Debug, Error)]
enum AppError {
    #[error("Gagal baca fail '{0}': {1}")]
    BacaFail(String, #[source] io::Error),
    #[error("Nilai mesti positif, dapat: {0}")]
    NilaiNegatif(i32),
    #[error("Format nombor tidak sah: {0}")]
    ParseError(#[from] ParseIntError),
}

fn baca_fail(laluan: &str) -> Result<String, AppError> {
    std::fs::read_to_string(laluan)
        .map_err(|e| AppError::BacaFail(laluan.into(), e))
}

fn proses(laluan: &str) -> Result<i32, AppError> {
    let data = baca_fail(laluan)?;  // ? = "kalau Err, return Err"
    let n: i32 = data.trim().parse()?; // ParseIntError → AppError::ParseError
    if n < 0 { return Err(AppError::NilaiNegatif(n)); }
    Ok(n)
}

fn main() {
    // Caller TAHU dari type: proses() boleh return AppError!
    match proses("data.txt") {
        Ok(n)                         => println!("Nombor: {}", n),
        Err(AppError::BacaFail(p, _)) => println!("Fail '{}' tidak ada", p),
        Err(AppError::NilaiNegatif(n)) => println!("Nilai {} negatif", n),
        Err(AppError::ParseError(e))   => println!("Format salah: {}", e),
    }
}

// ── Terjemahan Try/Catch ──────────────────────────────────────
// PHP try { } catch { }   →  match result { Ok → Err }
// PHP throw Exception     →  return Err(...)
// PHP Exception::getMessage() → e.to_string() atau format!("{}", e)
// PHP finally { }         →  Drop trait (auto cleanup)
```

---

# BAB 6: Class → Struct + Impl + Trait 🏗️

## PHP Class — Satu Konsep untuk Semua

```php
// PHP class = data + method + inheritance + interface
class Pekerja implements Boleh Cetak {
    private string $nama;
    protected float $gaji;
    public static int $kiraan = 0;

    public function __construct(string $nama, float $gaji) {
        $this->nama = $nama;
        $this->gaji = $gaji;
        self::$kiraan++;
    }

    public function getNama(): string { return $this->nama; }
    public function setGaji(float $gaji): void { $this->gaji = $gaji; }

    public static function getKiraan(): int { return self::$kiraan; }

    public function cetak(): void { echo "{$this->nama}: RM{$this->gaji}"; }
}

class PekerjaKanan extends Pekerja {
    private int $tahun_khidmat;

    public function __construct(string $nama, float $gaji, int $tahun) {
        parent::__construct($nama, $gaji);
        $this->tahun_khidmat = $tahun;
    }
}
```

## Rust — Pisahkan Konsep

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

// ── Struct: data sahaja ───────────────────────────────────────
#[derive(Debug, Clone)]
struct Pekerja {
    nama: String,   // "private" secara default dalam modul lain
    gaji: f64,
}

// ── Impl: method ─────────────────────────────────────────────
static KIRAAN: AtomicUsize = AtomicUsize::new(0);

impl Pekerja {
    // Associated function (macam static method PHP)
    fn baru(nama: &str, gaji: f64) -> Self {
        KIRAAN.fetch_add(1, Ordering::Relaxed);
        Pekerja { nama: nama.into(), gaji }
    }

    fn kiraan() -> usize {
        KIRAAN.load(Ordering::Relaxed)
    }

    // Instance method
    fn nama(&self) -> &str { &self.nama }

    // Mutable method (macam setter)
    fn tetapkan_gaji(&mut self, gaji: f64) { self.gaji = gaji; }
}

// ── Trait: interface + default behavior ──────────────────────
trait BolehCetak {
    fn cetak(&self);

    fn cetak_dengan_label(&self, label: &str) {
        print!("[{}] ", label);
        self.cetak();
        println!();
    }
}

impl BolehCetak for Pekerja {
    fn cetak(&self) {
        print!("{}: RM{:.2}", self.nama, self.gaji);
    }
}

// ── "Inheritance" dalam Rust: Composition ────────────────────
// TIDAK ADA inheritance! Guna composition:

struct PekerjaKanan {
    asas:          Pekerja,  // CONTAIN, bukan EXTEND
    tahun_khidmat: u32,
}

impl PekerjaKanan {
    fn baru(nama: &str, gaji: f64, tahun: u32) -> Self {
        PekerjaKanan {
            asas:          Pekerja::baru(nama, gaji),
            tahun_khidmat: tahun,
        }
    }

    fn nama(&self) -> &str { self.asas.nama() }
}

// BolehCetak untuk PekerjaKanan — implement sendiri
impl BolehCetak for PekerjaKanan {
    fn cetak(&self) {
        print!("{} ({}thn): RM{:.2}", self.asas.nama, self.tahun_khidmat, self.asas.gaji);
    }
}

// ── Terjemahan PHP → Rust ────────────────────────────────────
// PHP __construct         → fn baru() / fn new() associated function
// PHP $this->field        → self.field
// PHP self::$static       → static variable (AtomicXxx / LazyLock)
// PHP public/private      → pub modifier (tiada protected)
// PHP extends             → Composition (contain struct)
// PHP implements          → impl Trait for Struct
// PHP interface           → trait
// PHP abstract class      → trait dengan default methods
// PHP __toString          → impl fmt::Display
// PHP __clone             → impl Clone + #[derive(Clone)]
// PHP __destruct          → impl Drop
```

---

# BAB 7: Reference PHP → Borrow Rust 🔗

## Perbandingan Langsung

```php
// PHP: & pass by reference
function gandakan(&$nilai) {
    $nilai *= 2;
}

$n = 5;
gandakan($n);
echo $n; // 10

// PHP: Object sentiasa pass by reference (secara default)
function resetNama($pekerja) {
    $pekerja->nama = "Anonymous"; // ubah object asal!
}

// PHP: clone untuk buat salinan
function resetNamaKlon($pekerja) {
    $klon = clone $pekerja;
    $klon->nama = "Anonymous"; // hanya ubah klon
}
```

```rust
// Rust: explicit borrow dan mutable borrow

fn gandakan(nilai: &mut i32) {
    *nilai *= 2; // dereference dengan *
}

fn main() {
    let mut n = 5;
    gandakan(&mut n); // explicit &mut
    println!("{}", n); // 10
}

// Struct: borrow vs owned
struct Pekerja { nama: String }

fn reset_nama(p: &mut Pekerja) { // mutable borrow
    p.nama = "Anonymous".into();
}

fn reset_nama_klon(p: &Pekerja) -> Pekerja { // immutable borrow + return baru
    Pekerja { nama: "Anonymous".into() }
    // tidak ubah p! return Pekerja baru
}

// ── Peraturan Borrow yang PHP Developer Selalu Lupa ──────────

fn main() {
    let mut p = Pekerja { nama: "Ali".into() };

    let nama_ref = &p.nama;           // immutable borrow
    // reset_nama(&mut p);            // ERROR! ada immutable borrow aktif
    println!("{}", nama_ref);         // guna nama_ref
    // Sekarang nama_ref "habis" scope

    reset_nama(&mut p);               // OK sekarang!
    println!("{}", p.nama);
}
```

---

# BAB 8: Fungsi PHP → Fungsi Rust ⚙️

## Perbezaan Cara Berfikir

```php
// PHP: fungsi sangat longgar
function proses($data, $pilihan = null, ...$tambahan) {
    // $data boleh array, string, object, atau null
    // Tiada guarantee!
    if (is_array($data)) { /* ... */ }
    elseif (is_string($data)) { /* ... */ }
    // dan seterusnya...
}

// PHP: named arguments (PHP 8+)
proses(data: $myData, pilihan: ["key" => "val"]);

// PHP: type hints (PHP 7+, tapi boleh bypass dengan null)
function kira(int $a, int $b): int {
    return $a + $b;
}
kira(null, 5); // PHP: TypeError (strict mode) atau 0 (non-strict)
```

```rust
// Rust: fungsi sangat ketat tapi expressif

// Setiap parameter MESTI ada type
fn kira(a: i32, b: i32) -> i32 {
    a + b
}

// Tiada default parameter!
// Penyelesaian 1: Optional parameter dengan Option
fn kira_dengan_option(a: i32, b: Option<i32>) -> i32 {
    a + b.unwrap_or(0)
}

// Penyelesaian 2: Builder pattern
struct KiraBuilder { a: i32, b: i32, kadar: f64 }
impl KiraBuilder {
    fn baru(a: i32) -> Self { KiraBuilder { a, b: 0, kadar: 1.0 } }
    fn dengan_b(mut self, b: i32) -> Self { self.b = b; self }
    fn dengan_kadar(mut self, k: f64) -> Self { self.kadar = k; self }
    fn hitung(&self) -> f64 { (self.a + self.b) as f64 * self.kadar }
}

// Penggunaan:
let hasil = KiraBuilder::baru(10)
    .dengan_b(5)
    .dengan_kadar(1.5)
    .hitung();

// Penyelesaian 3: Struct sebagai parameter
struct KiraParams { a: i32, b: i32, kadar: f64 }
impl Default for KiraParams {
    fn default() -> Self { KiraParams { a: 0, b: 0, kadar: 1.0 } }
}
fn kira_params(p: KiraParams) -> f64 {
    (p.a + p.b) as f64 * p.kadar
}
// Guna: kira_params(KiraParams { a: 10, ..Default::default() })

// ── Tiada overloading dalam Rust ─────────────────────────────
// PHP:
// function kira(int $a) { ... }
// function kira(int $a, int $b) { ... } // Error! Sama nama

// Rust: tiada overloading, guna trait atau nama berbeza
trait Kira {
    fn kira(&self) -> i32;
}
struct Satu(i32);
struct Dua(i32, i32);

impl Kira for Satu { fn kira(&self) -> i32 { self.0 } }
impl Kira for Dua  { fn kira(&self) -> i32 { self.0 + self.1 } }
```

---

# BAB 9: Closures dan Callbacks 🎯

## PHP Closure vs Rust Closure

```php
// PHP closure
$darab = function($n) use ($faktor) {
    return $n * $faktor;
};

// PHP: $use bagi akses kepada variable luar
$faktor = 3;
$gandakan = function($n) use ($faktor) { return $n * $faktor; };
echo $gandakan(5); // 15

// PHP: &$faktor untuk ubah variable luar
$kiraan = 0;
$tambah = function() use (&$kiraan) { $kiraan++; };
$tambah(); $tambah();
echo $kiraan; // 2

// PHP: callable — function, closure, [object, method], atau string nama function
function guna_callback(array $data, callable $fn): array {
    return array_map($fn, $data);
}
```

```rust
// Rust closure — tiga jenis
let faktor = 3;

// Fn — borrow immutable (boleh panggil banyak kali)
let gandakan = |n: i32| n * faktor;
println!("{}", gandakan(5)); // 15
println!("{}", gandakan(7)); // 21 — faktor masih accessible

// FnMut — borrow mutable (boleh panggil banyak kali, boleh ubah capture)
let mut kiraan = 0;
let mut tambah = || { kiraan += 1; kiraan };
println!("{}", tambah()); // 1
println!("{}", tambah()); // 2
drop(tambah); // lepaskan borrow
println!("{}", kiraan); // 2

// FnOnce — consume capture (boleh panggil SEKALI sahaja)
let data = vec![1, 2, 3];
let ambil = move || data; // consume data
let diambil = ambil();    // OK
// ambil();               // ERROR! sudah consumed

// ── Callback dalam fungsi ─────────────────────────────────────

// Guna Fn untuk callback yang boleh dipanggil berkali-kali
fn guna_callback<F: Fn(i32) -> i32>(data: &[i32], f: F) -> Vec<i32> {
    data.iter().map(|&n| f(n)).collect()
}

let faktor = 2;
let hasil = guna_callback(&[1, 2, 3], |n| n * faktor);
println!("{:?}", hasil); // [2, 4, 6]

// Callback yang disimpan dalam struct → Box<dyn Fn>
struct PemprosesData {
    transformer: Box<dyn Fn(i32) -> i32>,
}

impl PemprosesData {
    fn baru(f: impl Fn(i32) -> i32 + 'static) -> Self {
        PemprosesData { transformer: Box::new(f) }
    }

    fn proses(&self, data: &[i32]) -> Vec<i32> {
        data.iter().map(|&n| (self.transformer)(n)).collect()
    }
}
```

---

# BAB 10: Mental Model Akhir 🧠

## Cara Berfikir PHP vs Cara Berfikir Rust

```
PHP MENTAL MODEL:
  "Saya ada data, saya perlu buat sesuatu dengannya"
  → Pass ke mana sahaja
  → Akan ada bila perlu
  → Error? catch kemudian atau ignore

RUST MENTAL MODEL:
  "Data ini milik siapa? Berapa lama ia perlu ada?
   Boleh ada dua bahagian kod akses serentak?"
  → Fikir OWNERSHIP
  → Fikir LIFETIME
  → Fikir THREAD SAFETY

Perubahan cara fikir yang diperlukan:
  ┌─────────────────────────────────────────────────────┐
  │ PHP: "Buat dahulu, selesaikan masalah kemudian"     │
  │ Rust: "Fikir dahulu, tidak ada masalah kemudian"    │
  └─────────────────────────────────────────────────────┘
```

---

## Terjemahan Konsep PHP → Rust

```rust
// Panduan mini: "Saya nak buat X dalam PHP. Macam mana dalam Rust?"

// ── Data Structures ──────────────────────────────────────────
// PHP: $arr = []           → Vec::new() atau vec![]
// PHP: $arr = ["a" => 1]  → HashMap::new() + insert
// PHP: new stdClass()     → struct atau HashMap<String, serde_json::Value>
// PHP: json_decode($json) → serde_json::from_str(&json)
// PHP: json_encode($data) → serde_json::to_string(&data)

// ── String Operations ─────────────────────────────────────────
// PHP: strlen($s)              → s.len()
// PHP: strtolower($s)          → s.to_lowercase()
// PHP: strtoupper($s)          → s.to_uppercase()
// PHP: trim($s)                → s.trim()
// PHP: str_contains($s, "x")   → s.contains("x")
// PHP: str_starts_with($s, "x")→ s.starts_with("x")
// PHP: substr($s, 0, 5)        → &s[0..5]
// PHP: str_replace("a", "b",$s)→ s.replace("a", "b")
// PHP: explode(",", $s)        → s.split(",").collect::<Vec<_>>()
// PHP: implode(",", $arr)      → arr.join(",")
// PHP: sprintf("%d", $n)       → format!("{}", n)
// PHP: "Hello $name"           → format!("Hello {}", name)

// ── Array/Vec Operations ─────────────────────────────────────
// PHP: array_push($arr, $val)  → arr.push(val)
// PHP: array_pop($arr)         → arr.pop() → Option
// PHP: array_shift($arr)       → arr.remove(0)
// PHP: array_unshift($arr,$v)  → arr.insert(0, v)
// PHP: count($arr)             → arr.len()
// PHP: in_array($val, $arr)    → arr.contains(&val)
// PHP: array_map(fn, $arr)     → arr.iter().map(fn).collect()
// PHP: array_filter($arr, fn)  → arr.iter().filter(fn).collect()
// PHP: array_reduce($arr,fn,$i)→ arr.iter().fold(i, fn)
// PHP: array_slice($arr,0,3)   → arr[0..3].to_vec()
// PHP: sort($arr)              → arr.sort()
// PHP: usort($arr, fn)         → arr.sort_by(fn)
// PHP: array_unique($arr)      → menggunakan HashSet

// ── Control Flow ─────────────────────────────────────────────
// PHP: if/else/elseif          → if/else/else if
// PHP: switch/case             → match (exhaustive!)
// PHP: for ($i=0; $i<n; $i++) → for i in 0..n
// PHP: foreach ($arr as $v)   → for v in &arr
// PHP: foreach ($arr as $k=>$v)→ for (k, v) in &map
// PHP: while ($cond)           → while cond
// PHP: break/continue          → break/continue

// ── Error Handling ────────────────────────────────────────────
// PHP: try { } catch { }       → match result { Ok => Err }
// PHP: throw new Exception()   → return Err(...)
// PHP: $e->getMessage()        → e.to_string()
// PHP: @silence_error($fn())   → result.ok() atau result.unwrap_or_default()

// ── OOP ──────────────────────────────────────────────────────
// PHP: class Foo { }           → struct Foo { } + impl Foo { }
// PHP: interface Bar { }       → trait Bar { }
// PHP: implements Bar          → impl Bar for Foo
// PHP: extends Parent          → (tiada! guna composition)
// PHP: $this->                 → self.
// PHP: self::                  → Self:: atau fungsi modul
// PHP: new Foo()               → Foo::new() atau Foo { field: val }
// PHP: $obj->method()          → obj.method()
// PHP: Foo::staticMethod()     → Foo::fungsi_bebas()
// PHP: __construct             → fn new() atau fn buat()
// PHP: __destruct              → impl Drop
// PHP: __toString              → impl fmt::Display
// PHP: clone $obj              → obj.clone() (perlu #[derive(Clone)])
// PHP: public/private          → pub atau tiada pub (private dalam modul)
// PHP: protected               → (tiada! guna pub(crate) atau redesign)
```

---

## Perangkap Paling Biasa PHP Developer dalam Rust

```rust
// ── PERANGKAP 1: Lupa & untuk borrow ────────────────────────
let v = vec![1, 2, 3];
// for n in v { } // MOVE! v tidak valid selepas
for n in &v { }   // BORROW — v masih valid
for n in v.iter() { } // sama seperti &v

// ── PERANGKAP 2: Index masuk Vec tanpa check ─────────────────
let v = vec![1, 2, 3];
// let n = v[10]; // PANIC! IndexOutOfBounds
let n = v.get(10); // Option<&i32> — selamat!

// ── PERANGKAP 3: Tukar value dalam for loop ──────────────────
let mut v = vec![1, 2, 3];
for n in &mut v { // perlu &mut
    *n *= 2;      // perlu * untuk dereference
}
println!("{:?}", v); // [2, 4, 6]

// ── PERANGKAP 4: String vs &str mengelirukan ─────────────────
// String = owned (macam PHP $str = "hello")
// &str   = borrowed (macam PHP &$str, tapi immutable dan lebih luas guna)

fn terima_str(s: &str) { println!("{}", s); } // lebih flexible

let owned = String::from("hello");
terima_str(&owned);  // String → &str (auto deref)
terima_str("world"); // literal &str juga OK

// ── PERANGKAP 5: Struct tidak ada method tanpa impl ──────────
struct Pekerja { nama: String }
// Pekerja::baru() — ERROR! Belum ada impl

// Mesti tambah:
impl Pekerja {
    fn baru(nama: &str) -> Self {
        Pekerja { nama: nama.into() }
    }
}

// ── PERANGKAP 6: Clone tidak percuma ─────────────────────────
// PHP: clone $obj — programmer tahu itu mahal
// Rust: .clone() — nampak mudah tapi MAHAL untuk Vec besar

// Jangan:
let v: Vec<Vec<String>> = vec![];
let _ = v.clone(); // clone Vec<Vec<String>> = clone semua String!

// Lebih baik: guna reference, atau Arc untuk share
```

---

## Phrasebook: PHP ke Rust

```
Bila rasa nak tulis ini (PHP habit):

PHP: $x = null;
RUST: let x: Option<T> = None;
      atau redesign supaya x tidak perlu null

PHP: throw new Exception($msg);
RUST: return Err(AppError::from($msg));
      atau bail!($msg) dengan anyhow

PHP: catch (Exception $e) { log($e); }
RUST: match result { Ok(v) => v, Err(e) => { log!("{}", e); default } }

PHP: function f($a, $b = null) { }
RUST: fn f(a: T, b: Option<U>) { }
      atau guna builder pattern

PHP: extends ParentClass
RUST: struct Child { parent: ParentData, ... }
      + impl SameTrait for Child

PHP: (int)$val
RUST: val as i32 atau i32::from(val) atau val.try_into()?

PHP: array_map(function($x) { return $x * 2; }, $arr)
RUST: arr.iter().map(|x| x * 2).collect::<Vec<_>>()

PHP: !empty($str)
RUST: !str.is_empty()
      atau str.len() > 0

PHP: isset($arr['key'])
RUST: map.contains_key("key")
      atau map.get("key").is_some()

PHP: $arr[] = $val;
RUST: vec.push(val);

PHP: echo $x;
RUST: println!("{}", x);
      (atau print!("{}", x) tanpa newline)
```

---

## Mindset Shift — Ringkasan

```
DARI                              KE
──────────────────────────────────────────────────────────────
"Ini salah syntax PHP"           "Ini cara Rust yang betul"

"Kenapa kena tulis banyak?"      "Kerana compiler tolong verify"

"Compiler terlalu strict"        "Compiler adalah rakan — bukan musuh"

"Nak tukar type je pun susah"    "Type system melindungi dari bug"

"Boleh guna null je"             "Option<T> lebih selamat"

"Array PHP lebih mudah"          "Vec+HashMap lebih tepat untuk tugasan"

"Kenapa perlu &?"                "Supaya jelas siapa yang memiliki data"

"Exception lebih mudah"          "Result lebih jelas tentang ralat"

"Inheritance lebih natural"      "Composition lebih flexible"

"Nak prototype cepat"            "Guna .unwrap() dulu, betulkan kemudian"
                                 (ini sah! Tandakan TODO)
```

---

*Belajar Rust sebagai PHP developer bukan tentang syntax.*
*Ia tentang cara berfikir yang berbeza tentang data, ownership, dan ralat.*
*Pelabur terbaik: masa untuk faham KENAPA, bukan hanya BAGAIMANA.* 🦀
