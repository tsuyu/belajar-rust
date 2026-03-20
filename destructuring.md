# 🔓 Destructuring dalam Rust — Beginner to Advanced

> Panduan lengkap destructuring: dari asas hingga nested, pinned, dan pattern lanjutan.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu Destructuring?

```
Destructuring = "buka bungkusan" dan ambil bahagian yang kita nak.

Tanpa destructuring (cara lama):
  let titik = Titik { x: 3.0, y: 7.0 };
  let x = titik.x;   // satu-satu
  let y = titik.y;   // satu-satu

Dengan destructuring (cara Rust):
  let Titik { x, y } = titik; // sekali gus!

Analoginya:
  Macam buka kotak hadiah dan ambil terus apa yang ada dalam:
  ┌─────────────────────┐      x = 3.0
  │  Titik { x, y }     │  →   y = 7.0
  └─────────────────────┘
```

---

## Kenapa Destructuring Penting?

```
✔ Kurangkan kod berulang
✔ Buat niat kod lebih jelas
✔ Bekerja cantik dengan pattern matching
✔ Elak akses field berulang kali
✔ Sesuai untuk function parameters
✔ Asas kepada semua pattern matching dalam Rust
```

---

## Peta Pembelajaran

```
Bab 1  → Destructuring Variables Asas
Bab 2  → Destructuring Structs
Bab 3  → Destructuring Enums
Bab 4  → Destructuring Tuples
Bab 5  → Destructuring Arrays & Slices
Bab 6  → Nested Destructuring
Bab 7  → Destructuring dalam Function Parameters
Bab 8  → Destructuring dengan References
Bab 9  → Pattern Lanjutan & Edge Cases
Bab 10 → Mini Project: Config Parser
```

---

# BAB 1: Destructuring Variables Asas 🌱

## Let Binding Asas

```rust
fn main() {
    // Tuple destructuring — paling mudah
    let (a, b, c) = (1, 2, 3);
    println!("a={}, b={}, c={}", a, b, c);

    // Swap dua nilai! — tiada temp variable
    let mut x = 5;
    let mut y = 10;
    (x, y) = (y, x);
    println!("x={}, y={}", x, y); // x=10, y=5

    // Ignore nilai dengan _
    let (pertama, _, ketiga) = ("Ali", "IGNORED", "Siti");
    println!("{} dan {}", pertama, ketiga);

    // Ignore baki dengan ..
    let (kepala, ..) = (1, 2, 3, 4, 5);
    println!("Kepala: {}", kepala); // 1

    // Destructure dari function return
    fn min_max(v: &[i32]) -> (i32, i32) {
        let mut min = v[0];
        let mut max = v[0];
        for &x in v.iter().skip(1) {
            if x < min { min = x; }
            if x > max { max = x; }
        }
        (min, max)
    }

    let data = vec![3, 1, 4, 1, 5, 9, 2, 6];
    let (min, max) = min_max(&data);
    println!("Min={}, Max={}", min, max); // Min=1, Max=9
}
```

---

## Destructuring dalam Loop

```rust
fn main() {
    // Destructure dalam for loop
    let pasangan = vec![(1, "satu"), (2, "dua"), (3, "tiga")];
    for (nombor, perkataan) in &pasangan {
        println!("{} = {}", nombor, perkataan);
    }

    // enumerate dengan destructure
    let buah = ["epal", "mangga", "pisang"];
    for (i, nama) in buah.iter().enumerate() {
        println!("[{}] {}", i, nama);
    }

    // Destructure tuple dalam iterator chain
    let hasil: Vec<i32> = pasangan.iter()
        .map(|(n, _)| n * n)          // ambil nombor, ignore perkataan
        .collect();
    println!("{:?}", hasil); // [1, 4, 9]

    // zip + destructure
    let a = vec![1, 2, 3];
    let b = vec!["x", "y", "z"];
    for (num, huruf) in a.iter().zip(b.iter()) {
        println!("{}: {}", num, huruf);
    }
}
```

---

## 🧠 Brain Teaser #1

Apa output kod ini?

```rust
fn main() {
    let ((a, b), (c, d)) = ((1, 2), (3, 4));
    let (x, y, ..) = (10, 20, 30, 40, 50);
    let (.., z) = (100, 200, 300);

    println!("{} {} {} {} {} {} {}", a, b, c, d, x, y, z);
}
```

<details>
<summary>👀 Jawapan</summary>

Output: `1 2 3 4 10 20 300`

- `((a,b),(c,d)) = ((1,2),(3,4))` → a=1, b=2, c=3, d=4
- `(x, y, ..)` → x=10, y=20, baki diabaikan
- `(.., z)` → z=300 (elemen terakhir)
</details>

---

# BAB 2: Destructuring Structs 🏗️

## Struct Asas

```rust
#[derive(Debug)]
struct Titik { x: f64, y: f64 }

#[derive(Debug)]
struct Warna { r: u8, g: u8, b: u8 }

#[derive(Debug)]
struct Pekerja {
    nama:     String,
    gaji:     f64,
    bahagian: String,
    aktif:    bool,
}

fn main() {
    // Destructure — nama binding sama dengan nama field
    let t = Titik { x: 3.0, y: 7.0 };
    let Titik { x, y } = t;
    println!("x={}, y={}", x, y);

    // Beri nama berbeza dengan ':'
    let w = Warna { r: 255, g: 128, b: 0 };
    let Warna { r: merah, g: hijau, b: biru } = w;
    println!("#{:02X}{:02X}{:02X}", merah, hijau, biru); // #FF8000

    // Ignore field dengan ..
    let p = Pekerja {
        nama:     "Ali Ahmad".into(),
        gaji:     5000.0,
        bahagian: "ICT".into(),
        aktif:    true,
    };

    let Pekerja { nama, bahagian, .. } = p; // ignore gaji dan aktif
    println!("{} bekerja di {}", nama, bahagian);

    // Destructure beberapa field, baki ignore
    let p2 = Pekerja {
        nama: "Siti".into(),
        gaji: 4500.0,
        bahagian: "HR".into(),
        aktif: false,
    };
    let Pekerja { nama: n2, aktif, .. } = p2;
    println!("{} aktif? {}", n2, aktif);
}
```

---

## Destructure dalam match

```rust
#[derive(Debug)]
struct Segiempat {
    atas_kiri:  (f64, f64),
    bawah_kanan: (f64, f64),
}

impl Segiempat {
    fn jenis(&self) -> &str {
        let Segiempat {
            atas_kiri: (x1, y1),
            bawah_kanan: (x2, y2),
        } = self;

        let lebar  = (x2 - x1).abs();
        let tinggi = (y2 - y1).abs();

        match (lebar, tinggi) {
            (l, t) if l == t => "Segiempat sama",
            _                => "Segiempat tepat",
        }
    }

    fn luas(&self) -> f64 {
        let Segiempat {
            atas_kiri: (x1, y1),
            bawah_kanan: (x2, y2),
        } = self;
        (x2 - x1).abs() * (y2 - y1).abs()
    }
}

fn main() {
    let s1 = Segiempat {
        atas_kiri:   (0.0, 0.0),
        bawah_kanan: (5.0, 5.0),
    };
    let s2 = Segiempat {
        atas_kiri:   (1.0, 1.0),
        bawah_kanan: (4.0, 3.0),
    };

    println!("{} — luas={}", s1.jenis(), s1.luas()); // Segiempat sama — 25
    println!("{} — luas={}", s2.jenis(), s2.luas()); // Segiempat tepat — 6
}
```

---

## Destructure Struct dengan Medan Bersarang

```rust
#[derive(Debug)]
struct Alamat {
    jalan:  String,
    bandar: String,
    negeri: String,
}

#[derive(Debug)]
struct Pengguna {
    id:     u32,
    nama:   String,
    emel:   String,
    alamat: Alamat,
}

fn main() {
    let pengguna = Pengguna {
        id:   1,
        nama: "Ali Ahmad".into(),
        emel: "ali@email.com".into(),
        alamat: Alamat {
            jalan:  "Jalan Maju 1".into(),
            bandar: "Kuala Lumpur".into(),
            negeri: "Selangor".into(),
        },
    };

    // Destructure nested — satu langkah!
    let Pengguna {
        nama,
        emel,
        alamat: Alamat { bandar, negeri, .. }, // destructure nested
        ..
    } = &pengguna;

    println!("Nama:   {}", nama);
    println!("Emel:   {}", emel);
    println!("Lokasi: {}, {}", bandar, negeri);

    // Dalam match
    match &pengguna {
        Pengguna { id, alamat: Alamat { negeri, .. }, .. }
            if negeri == "Selangor" =>
        {
            println!("Pengguna {} dari Selangor", id);
        }
        _ => println!("Bukan dari Selangor"),
    }
}
```

---

## 🧠 Brain Teaser #2

Padankan kod dengan output yang betul:

```rust
struct A { x: i32, y: i32, z: i32 }

fn main() {
    let a = A { x: 1, y: 2, z: 3 };

    // Kod 1:
    let A { x, .. } = a;

    // Kod 2:
    let A { x: p, y: q, .. } = A { x: 10, y: 20, z: 30 };

    // Kod 3:
    let A { z, x, .. } = A { x: 5, y: 6, z: 7 };

    println!("{} {} {} {}", x, p, q, z);
}
```

<details>
<summary>👀 Jawapan</summary>

Output: `1 10 20 7`

- Kod 1: `x = a.x = 1`
- Kod 2: `p = 10, q = 20` (nama field berbeza dari nama binding)
- Kod 3: `z = 7, x = 5` — urutan dalam destructure TIDAK penting, bergantung pada nama field!

**Penting:** Destructure struct mengikut **nama field**, bukan kedudukan!
</details>

---

# BAB 3: Destructuring Enums 🎭

## Enum Asas

```rust
#[derive(Debug)]
enum Bentuk {
    Bulatan(f64),                           // radius
    Segiempat { lebar: f64, tinggi: f64 },  // struct-like
    Segitiga(f64, f64, f64),                // tiga sisi
    Titik,                                  // unit
}

impl Bentuk {
    fn luas(&self) -> f64 {
        match self {
            // Tuple variant — destructure dengan posisi
            Bentuk::Bulatan(r)               => std::f64::consts::PI * r * r,

            // Struct variant — destructure dengan nama field
            Bentuk::Segiempat { lebar, tinggi } => lebar * tinggi,

            // Multi-tuple — nama pilihan sendiri
            Bentuk::Segitiga(a, b, c)        => {
                let s = (a + b + c) / 2.0;
                (s * (s-a) * (s-b) * (s-c)).sqrt()
            }

            // Unit variant — tiada data untuk destructure
            Bentuk::Titik => 0.0,
        }
    }

    fn nama(&self) -> &str {
        // Guna .. untuk ignore data bila tidak perlu
        match self {
            Bentuk::Bulatan(..)     => "Bulatan",
            Bentuk::Segiempat {..}  => "Segiempat",
            Bentuk::Segitiga(..)    => "Segitiga",
            Bentuk::Titik           => "Titik",
        }
    }
}

fn main() {
    let bentuk = vec![
        Bentuk::Bulatan(5.0),
        Bentuk::Segiempat { lebar: 4.0, tinggi: 6.0 },
        Bentuk::Segitiga(3.0, 4.0, 5.0),
        Bentuk::Titik,
    ];

    for b in &bentuk {
        println!("{}: luas = {:.2}", b.nama(), b.luas());
    }
}
```

---

## Option & Result — Destructure Paling Kerap

```rust
fn main() {
    // Option<T>
    let nilai: Option<i32> = Some(42);

    // if let destructure
    if let Some(n) = nilai {
        println!("Ada nilai: {}", n);
    }

    // match destructure
    let dua_kali = match nilai {
        Some(n) => Some(n * 2),
        None    => None,
    };
    println!("{:?}", dua_kali); // Some(84)

    // Destructure dalam let
    if let Some(n) = Some(100) {
        println!("n = {}", n);
    }

    // Result<T, E>
    fn buat_kerja(berjaya: bool) -> Result<String, String> {
        if berjaya {
            Ok("Data berjaya diproses".into())
        } else {
            Err("Terdapat ralat!".into())
        }
    }

    match buat_kerja(true) {
        Ok(mesej)  => println!("✔ {}", mesej),
        Err(ralat) => println!("✘ {}", ralat),
    }

    // Destructure Ok dengan pelbagai jenis
    let r: Result<(i32, &str), &str> = Ok((42, "berjaya"));
    if let Ok((nombor, mesej)) = r {
        println!("Nombor: {}, Mesej: {}", nombor, mesej);
    }
}
```

---

## Enum dengan Variant Kompleks

```rust
#[derive(Debug, Clone)]
enum Token {
    Nombor(f64),
    Operator(char),
    Pemboleh(String),
    Fungsi { nama: String, argumen: Vec<f64> },
    Anotasi { kunci: String, nilai: String },
}

fn huraikan(token: &Token) -> String {
    match token {
        Token::Nombor(n) if n.fract() == 0.0 => {
            format!("Integer: {}", *n as i64)
        }
        Token::Nombor(n) => format!("Float: {:.4}", n),

        Token::Operator('+') => "Tambah".to_string(),
        Token::Operator('-') => "Tolak".to_string(),
        Token::Operator('*') => "Kali".to_string(),
        Token::Operator('/') => "Bahagi".to_string(),
        Token::Operator(op)  => format!("Op lain: {}", op),

        Token::Pemboleh(nama) if nama.starts_with('_') => {
            format!("Pemboleh private: {}", nama)
        }
        Token::Pemboleh(nama) => format!("Pemboleh: {}", nama),

        Token::Fungsi { nama, argumen } if argumen.is_empty() => {
            format!("Fungsi {}() — tiada argumen", nama)
        }
        Token::Fungsi { nama, argumen } => {
            format!("Fungsi {}({} argumen)", nama, argumen.len())
        }

        Token::Anotasi { kunci, nilai } => {
            format!("@{}={}", kunci, nilai)
        }
    }
}

fn main() {
    let tokens = vec![
        Token::Nombor(3.0),
        Token::Nombor(3.14),
        Token::Operator('+'),
        Token::Operator('%'),
        Token::Pemboleh("_private".into()),
        Token::Pemboleh("x".into()),
        Token::Fungsi { nama: "sin".into(), argumen: vec![1.57] },
        Token::Fungsi { nama: "print".into(), argumen: vec![] },
        Token::Anotasi { kunci: "doc".into(), nilai: "fungsi utama".into() },
    ];

    for t in &tokens {
        println!("{:?} → {}", t, huraikan(t));
    }
}
```

---

# BAB 4: Destructuring Tuples 📦

## Tuple Pelbagai Saiz

```rust
fn main() {
    // Tuple 2 elemen
    let (a, b) = (10, 20);
    println!("{} {}", a, b);

    // Tuple 3 elemen
    let (x, y, z) = (1.0f64, 2.0f64, 3.0f64);
    println!("Jarak dari asal: {:.2}", (x*x + y*y + z*z).sqrt());

    // Tuple mixed types
    let (nama, umur, aktif): (&str, u32, bool) = ("Ali", 25, true);
    println!("{} ({}) aktif={}", nama, umur, aktif);

    // Nested tuple
    let ((x1, y1), (x2, y2)) = ((0.0f64, 0.0f64), (3.0f64, 4.0f64));
    let jarak = ((x2-x1).powi(2) + (y2-y1).powi(2)).sqrt();
    println!("Jarak: {}", jarak); // 5.0

    // Destructure sebahagian
    let (kepala, ..) = (1, 2, 3, 4, 5);         // kepala = 1
    let (.., ekor)   = (1, 2, 3, 4, 5);         // ekor = 5
    let (a, _, c, ..) = (10, 20, 30, 40, 50);   // a=10, c=30
    println!("{} {} {} {}", kepala, ekor, a, c);

    // Tuple struct (newtype pattern)
    struct Meter(f64);
    struct Kg(f64);

    let Meter(jarak_m) = Meter(100.0);
    let Kg(berat_kg)   = Kg(75.0);
    println!("{}m, {}kg", jarak_m, berat_kg);
}
```

---

## Tuple Struct — Newtype Pattern

```rust
// Newtype pattern — wrap type lain untuk type safety
struct Email(String);
struct NoTelefon(String);
struct Ringgit(f64);

impl Email {
    fn baru(s: &str) -> Result<Self, &'static str> {
        if s.contains('@') {
            Ok(Email(s.to_string()))
        } else {
            Err("Format emel tidak sah")
        }
    }
}

impl Ringgit {
    fn tambah(self, lain: Ringgit) -> Ringgit {
        let Ringgit(a) = self;
        let Ringgit(b) = lain;
        Ringgit(a + b)
    }
}

impl std::fmt::Display for Ringgit {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        let Ringgit(jumlah) = self;
        write!(f, "RM{:.2}", jumlah)
    }
}

fn hantar_emel(Email(alamat): &Email, mesej: &str) {
    println!("Hantar '{}' ke {}", mesej, alamat);
}

fn main() {
    match Email::baru("ali@contoh.com") {
        Ok(emel) => hantar_emel(&emel, "Hello!"),
        Err(e)   => println!("Ralat: {}", e),
    }

    let harga1 = Ringgit(15.50);
    let harga2 = Ringgit(8.00);
    let jumlah = harga1.tambah(harga2);
    println!("Jumlah: {}", jumlah); // RM23.50

    // Destructure dalam let
    let Ringgit(nilai) = Ringgit(42.50);
    println!("Nilai mentah: {}", nilai); // 42.5
}
```

---

# BAB 5: Destructuring Arrays & Slices 🔢

## Fixed Array Destructuring

```rust
fn main() {
    // Array 3 elemen
    let [a, b, c] = [10, 20, 30];
    println!("{} {} {}", a, b, c);

    // Dengan ignore
    let [pertama, _, ketiga] = [1, 2, 3];
    println!("Pertama={}, Ketiga={}", pertama, ketiga);

    // Array 2D
    let [[a, b], [c, d]] = [[1, 2], [3, 4]];
    println!("{} {} {} {}", a, b, c, d);

    // Kombinasi jenis
    let [x, y]: [f64; 2] = [1.0, 2.0];
    println!("({}, {})", x, y);

    // Range pattern dalam array
    let arr = [5, 3, 8, 1, 9];
    if let [first, .., last] = arr {
        println!("Mula={}, Akhir={}", first, last); // 5, 9
    }
}
```

---

## Slice Patterns (Lebih Fleksibel)

```rust
fn proses(data: &[i32]) {
    match data {
        // Kosong
        [] => println!("Kosong"),

        // Tepat satu elemen
        [x] => println!("Satu: {}", x),

        // Tepat dua elemen
        [a, b] => println!("Dua: {}, {}", a, b),

        // Tiga atau lebih — ambil kepala dan ekor
        [kepala, badan @ .., ekor] => {
            println!("Kepala={}, Badan={:?}, Ekor={}", kepala, badan, ekor);
        }
    }
}

fn statistik(data: &[f64]) -> Option<(f64, f64, f64)> {
    match data {
        [] => None,
        [satu] => Some((*satu, *satu, *satu)),
        [a, b] => {
            let min = a.min(b);
            let max = a.max(b);
            let purata = (a + b) / 2.0;
            Some((*min, *max, purata))
        }
        [pertama, baki @ ..] => {
            let mut min = *pertama;
            let mut max = *pertama;
            let mut jumlah = *pertama;
            for &v in baki {
                if v < min { min = v; }
                if v > max { max = v; }
                jumlah += v;
            }
            Some((min, max, jumlah / data.len() as f64))
        }
    }
}

fn main() {
    proses(&[]);
    proses(&[42]);
    proses(&[1, 2]);
    proses(&[10, 20, 30, 40, 50]);

    let data = vec![3.0, 1.0, 4.0, 1.0, 5.0, 9.0, 2.0, 6.0];
    if let Some((min, max, purata)) = statistik(&data) {
        println!("Min={}, Max={}, Purata={:.2}", min, max, purata);
    }
}
```

---

## 🧠 Brain Teaser #3

Apa output kod ini?

```rust
fn main() {
    let v = vec![10, 20, 30, 40, 50];

    if let [a, b, baki @ ..] = v.as_slice() {
        println!("a={}, b={}", a, b);
        println!("baki={:?}", baki);
        println!("baki len={}", baki.len());
    }

    if let [.., kedua_akhir, _] = v.as_slice() {
        println!("Kedua dari akhir: {}", kedua_akhir);
    }
}
```

<details>
<summary>👀 Jawapan</summary>

```
a=10, b=20
baki=[30, 40, 50]
baki len=3
Kedua dari akhir: 40
```

- `[a, b, baki @ ..]` — a=10, b=20, baki = slice [30,40,50]
- `[.., kedua_akhir, _]` — `_` match elemen terakhir (50), `kedua_akhir` match sebelumnya (40)
</details>

---

# BAB 6: Nested Destructuring 🪆

## Struct dalam Enum dalam Struct

```rust
#[derive(Debug, Clone)]
struct Titik { x: f64, y: f64 }

#[derive(Debug, Clone)]
enum Bentuk {
    Bulatan { pusat: Titik, radius: f64 },
    Garisan { mula: Titik, tamat: Titik },
    Segiempat { sudut: Titik, lebar: f64, tinggi: f64 },
}

#[derive(Debug)]
struct Lukisan {
    tajuk:  String,
    bentuk: Vec<Bentuk>,
}

fn huraikan_bentuk(b: &Bentuk) {
    match b {
        // Nested destructure: struct variant + nested struct
        Bentuk::Bulatan {
            pusat: Titik { x, y },
            radius,
        } => {
            println!("Bulatan di ({:.1},{:.1}) r={:.1}", x, y, radius);
        }

        Bentuk::Garisan {
            mula:  Titik { x: x1, y: y1 },
            tamat: Titik { x: x2, y: y2 },
        } => {
            let panjang = ((x2-x1).powi(2) + (y2-y1).powi(2)).sqrt();
            println!("Garisan ({:.1},{:.1})→({:.1},{:.1}) len={:.2}",
                x1, y1, x2, y2, panjang);
        }

        Bentuk::Segiempat {
            sudut: Titik { x, y },
            lebar,
            tinggi,
        } => {
            println!("Segiempat @ ({:.1},{:.1}) {}×{}",
                x, y, lebar, tinggi);
        }
    }
}

fn main() {
    let lukisan = Lukisan {
        tajuk: "Geometri".into(),
        bentuk: vec![
            Bentuk::Bulatan {
                pusat: Titik { x: 5.0, y: 5.0 },
                radius: 3.0,
            },
            Bentuk::Garisan {
                mula:  Titik { x: 0.0, y: 0.0 },
                tamat: Titik { x: 3.0, y: 4.0 },
            },
            Bentuk::Segiempat {
                sudut:  Titik { x: 1.0, y: 1.0 },
                lebar:  8.0,
                tinggi: 4.0,
            },
        ],
    };

    println!("Lukisan: {}", lukisan.tajuk);
    for b in &lukisan.bentuk {
        huraikan_bentuk(b);
    }
}
```

---

## Nested Option & Result

```rust
fn main() {
    // Nested Option
    let data: Option<Option<i32>> = Some(Some(42));
    match data {
        Some(Some(n)) => println!("Dua lapis Some: {}", n),
        Some(None)    => println!("Lapis luar ada, dalam None"),
        None          => println!("Kosong terus"),
    }

    // Option dalam Vec dalam Result
    let hasil: Result<Vec<Option<i32>>, &str> =
        Ok(vec![Some(1), None, Some(3), None, Some(5)]);

    match hasil {
        Err(e) => println!("Error: {}", e),
        Ok(senarai) => {
            for (i, item) in senarai.iter().enumerate() {
                match item {
                    Some(n) => println!("  [{}] = {}", i, n),
                    None    => println!("  [{}] = (tiada)", i),
                }
            }
        }
    }

    // Flatten nested dengan and_then / flatten
    let nested: Option<Option<i32>> = Some(Some(99));
    let flat = nested.flatten(); // Some(99)
    println!("Flatten: {:?}", flat);

    // Transpose Result<Option<T>> ↔ Option<Result<T>>
    let r: Result<Option<i32>, &str> = Ok(Some(5));
    let o: Option<Result<i32, &str>> = r.transpose();
    println!("Transpose: {:?}", o); // Some(Ok(5))
}
```

---

# BAB 7: Destructuring dalam Function Parameters 🔧

## Direct Parameter Destructuring

```rust
// Tuple parameter
fn tambah_titik((x1, y1): (f64, f64), (x2, y2): (f64, f64)) -> (f64, f64) {
    (x1 + x2, y1 + y2)
}

// Struct parameter
#[derive(Debug)]
struct Pesakit { nama: String, umur: u8, berat_kg: f64 }

fn bmi(Pesakit { berat_kg, .. }: &Pesakit, tinggi_m: f64) -> f64 {
    berat_kg / (tinggi_m * tinggi_m)
}

// Enum parameter — perlu match
#[derive(Debug)]
enum Arahan {
    Maju { langkah: u32 },
    Belok(i32),  // darjah
    Berhenti,
}

fn laksana(arahan: &Arahan) -> String {
    match arahan {
        Arahan::Maju { langkah }  => format!("Maju {} langkah", langkah),
        Arahan::Belok(darjah)     => format!("Belok {}°", darjah),
        Arahan::Berhenti          => "Berhenti".to_string(),
    }
}

fn main() {
    let hasil = tambah_titik((1.0, 2.0), (3.0, 4.0));
    println!("Titik tambah: {:?}", hasil); // (4.0, 6.0)

    let p = Pesakit { nama: "Ali".into(), umur: 30, berat_kg: 70.0 };
    println!("BMI: {:.1}", bmi(&p, 1.70)); // ~24.2

    let arahan_list = vec![
        Arahan::Maju { langkah: 5 },
        Arahan::Belok(90),
        Arahan::Berhenti,
    ];
    for a in &arahan_list {
        println!("{}", laksana(a));
    }
}
```

---

## Destructure dalam Closures

```rust
fn main() {
    let pasangan = vec![(1, "satu"), (2, "dua"), (3, "tiga")];

    // Destructure dalam closure parameter
    let nombor: Vec<i32> = pasangan.iter()
        .map(|(n, _)| *n)       // ambil nombor sahaja
        .collect();
    println!("{:?}", nombor);   // [1, 2, 3]

    let perkataan: Vec<&&str> = pasangan.iter()
        .map(|(_, s)| s)        // ambil string sahaja
        .collect();
    println!("{:?}", perkataan);

    // Filter dengan destructure
    let besar: Vec<(i32, &&str)> = pasangan.iter()
        .map(|(n, s)| (*n, s))
        .filter(|(n, _)| *n > 1)
        .collect();
    println!("{:?}", besar);    // [(2, "dua"), (3, "tiga")]

    // Struct dalam closure
    #[derive(Debug)]
    struct Item { id: u32, harga: f64 }

    let barang = vec![
        Item { id: 1, harga: 10.50 },
        Item { id: 2, harga: 5.00 },
        Item { id: 3, harga: 20.00 },
    ];

    let jumlah: f64 = barang.iter()
        .map(|Item { harga, .. }| harga)  // destructure struct dalam closure
        .sum();
    println!("Jumlah: RM{:.2}", jumlah); // RM35.50

    let mahal: Vec<u32> = barang.iter()
        .filter(|Item { harga, .. }| *harga > 8.0)
        .map(|Item { id, .. }| *id)
        .collect();
    println!("Item mahal (ID): {:?}", mahal); // [1, 3]
}
```

---

## 🧠 Brain Teaser #4

Tulis fungsi `stat` yang terima `&[(String, Vec<f64>)]` dan return `Vec<(String, f64, f64)>` berisi (nama, min, max). Gunakan destructuring sepenuhnya.

<details>
<summary>👀 Jawapan</summary>

```rust
fn stat(data: &[(String, Vec<f64>)]) -> Vec<(String, f64, f64)> {
    data.iter()
        .filter_map(|(nama, nilai)| {  // destructure tuple param
            // slice pattern untuk min/max
            match nilai.as_slice() {
                [] => None,
                [satu] => Some((nama.clone(), *satu, *satu)),
                [pertama, baki @ ..] => {
                    let (min, max) = baki.iter().fold(
                        (*pertama, *pertama),
                        |(min, max), &v| (min.min(v), max.max(v))
                    );
                    Some((nama.clone(), min, max))
                }
            }
        })
        .collect()
}

fn main() {
    let data = vec![
        ("Matematik".to_string(), vec![85.0, 90.0, 78.0]),
        ("Sains".to_string(),     vec![70.0, 95.0, 88.0]),
        ("Kosong".to_string(),    vec![]),
    ];

    for (nama, min, max) in stat(&data) {
        println!("{}: min={}, max={}", nama, min, max);
    }
}
```
</details>

---

# BAB 8: Destructuring dengan References 🔗

## ref Keyword & & dalam Pattern

```rust
fn main() {
    let s = String::from("hello");

    // & dalam pattern — match reference, dapat nilai
    match &s {
        nilai => println!("Nilai: {}", nilai), // nilai: &String
    }

    // ref — buat reference ke dalam binding
    let list = vec![1, 2, 3];
    match list {
        ref v => println!("Len: {}", v.len()), // v: &Vec<i32>
        // list masih boleh guna selepas ni
    }
    println!("List: {:?}", list); // OK!

    // Destructure reference ke struct
    #[derive(Debug)]
    struct Titik { x: f64, y: f64 }

    let t = Titik { x: 3.0, y: 4.0 };
    let &Titik { x, y } = &t; // x, y adalah f64 (Copy)
    // Atau:
    let Titik { x: rx, y: ry } = &t; // rx, ry adalah &f64

    println!("Copy: ({}, {})", x, y);
    println!("Ref:  ({}, {})", rx, ry);
}
```

---

## Destructure &mut Reference

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5];

    // Iterate mut dengan destructure
    for x in &mut v {
        *x *= 2;
    }
    println!("{:?}", v); // [2, 4, 6, 8, 10]

    // Destructure &mut tuple
    let mut pasangan = (10, 20);
    let (ref mut a, ref mut b) = pasangan;
    *a += 1;
    *b += 1;
    println!("{:?}", pasangan); // (11, 21)

    // Closure dengan mutable destructure
    let mut data: Vec<(String, i32)> = vec![
        ("Ali".into(), 80),
        ("Siti".into(), 75),
    ];

    for (_, markah) in &mut data {
        *markah += 10; // bonus 10 markah
    }

    for (nama, markah) in &data {
        println!("{}: {}", nama, markah); // Ali: 90, Siti: 85
    }
}
```

---

## Deref Coercions dalam Destructure

```rust
fn main() {
    // String → &str melalui deref
    let s = String::from("hello world");

    // Guna &str methods melalui deref
    if let [kepala, ..] = s.split_whitespace().collect::<Vec<_>>().as_slice() {
        println!("Perkataan pertama: {}", kepala);
    }

    // Box<T> — deref dalam destructure
    let boxed = Box::new((1i32, 2i32));
    let (a, b) = *boxed; // dereference Box
    println!("{} {}", a, b);

    // Deref dengan &
    let r: &(i32, i32) = &(10, 20);
    let &(x, y) = r;  // pattern & pada reference
    println!("{} {}", x, y);

    // Atau guna .0 / .1 biasa
    let (p, q) = *r; // dereference dulu
    println!("{} {}", p, q);
}
```

---

# BAB 9: Pattern Lanjutan & Edge Cases 🎯

## Destructure dengan Shadowing

```rust
fn main() {
    let x = 5;

    // Shadow x dengan nilai baru dalam destructure
    let x = match x {
        n @ 1..=5 => n * 2,    // shadow x dengan n*2
        n         => n,
    };
    println!("x = {}", x); // 10

    // Perhatian! Pattern binding boleh shadow variable luar
    let y = Some(10);
    if let Some(y) = y {       // y dalam if let shadows y luar
        println!("Dalam: y = {}", y); // 10
    }
    println!("Luar:  y = {:?}", y);   // Some(10) — luar tidak berubah!

    // Destructure dengan mut
    let (mut a, b) = (1, 2);
    a += 10;
    println!("{} {}", a, b); // 11 2
}
```

---

## Irrefutable vs Refutable Patterns

```rust
fn main() {
    // IRREFUTABLE — sentiasa match, boleh guna let biasa
    let (a, b) = (1, 2);               // ✔ tuple
    let [x, y, z] = [10, 20, 30];      // ✔ array dengan saiz tepat
    let Titik { x: px, y: py } = Titik { x: 1.0, y: 2.0 }; // ✔ struct

    struct Titik { x: f64, y: f64 }

    // REFUTABLE — mungkin tidak match
    // let Some(n) = Some(5);           // ⚠ WARNING/ERROR dalam Rust 2024

    // Guna if let atau let else untuk refutable
    let Some(n) = Some(5) else { return; };
    println!("n = {}", n);

    // let else untuk validation
    fn parse_port(s: &str) -> Option<u16> {
        let Ok(port) = s.parse::<u16>() else { return None; };
        let (1..=65535) = port else { return None; };
        Some(port)
    }

    println!("{:?}", parse_port("8080")); // Some(8080)
    println!("{:?}", parse_port("0"));    // None
    println!("{:?}", parse_port("abc"));  // None
}
```

---

## Destructure String & &str

```rust
fn main() {
    // String tidak boleh destructure seperti array terus
    // Tapi boleh guna methods + slice
    let s = "hello world";

    // Split + destructure sebagai slice
    let bahagian: Vec<&str> = s.split_whitespace().collect();
    if let [a, b] = bahagian.as_slice() {
        println!("Dua perkataan: '{}' '{}'", a, b);
    }

    // Chars + slice
    let huruf: Vec<char> = s.chars().collect();
    if let [pertama, .., terakhir] = huruf.as_slice() {
        println!("Pertama='{}', Terakhir='{}'", pertama, terakhir);
    }

    // Match pada string slices
    let arahan = "tambah 10";
    let bahagian2: Vec<&str> = arahan.splitn(2, ' ').collect();

    match bahagian2.as_slice() {
        ["tambah", val] => println!("Tambah: {}", val),
        ["tolak",  val] => println!("Tolak: {}", val),
        ["senarai"]     => println!("Papar senarai"),
        [perintah, ..] => println!("Perintah tidak dikenali: {}", perintah),
        []              => println!("Input kosong"),
    }
}
```

---

## Destructure Iterator Results

```rust
fn main() {
    // partition + destructure
    let nombor = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    let (genap, ganjil): (Vec<i32>, Vec<i32>) =
        nombor.iter().partition(|&&x| x % 2 == 0);
    println!("Genap:  {:?}", genap);  // [2,4,6,8,10]
    println!("Ganjil: {:?}", ganjil); // [1,3,5,7,9]

    // unzip — Vec<(A,B)> → (Vec<A>, Vec<B>)
    let pasangan = vec![(1, 'a'), (2, 'b'), (3, 'c')];
    let (nombor2, huruf): (Vec<i32>, Vec<char>) = pasangan.into_iter().unzip();
    println!("Nombor: {:?}", nombor2); // [1,2,3]
    println!("Huruf:  {:?}", huruf);   // ['a','b','c']

    // min_by_key + destructure
    #[derive(Debug)]
    struct Pelajar { nama: &'static str, markah: u32 }
    let kelas = vec![
        Pelajar { nama: "Ali",  markah: 85 },
        Pelajar { nama: "Siti", markah: 92 },
        Pelajar { nama: "Amin", markah: 78 },
    ];

    if let Some(Pelajar { nama, markah }) =
        kelas.iter().min_by_key(|p| p.markah)
    {
        println!("Markah paling rendah: {} ({})", nama, markah);
    }
}
```

---

# BAB 10: Mini Project — Config Parser 🔧

```rust
use std::collections::HashMap;

// ─── Types ────────────────────────────────────────────────────

#[derive(Debug, Clone, PartialEq)]
enum NilaiConfig {
    Teks(String),
    Nombor(f64),
    Bool(bool),
    Senarai(Vec<NilaiConfig>),
    Jadual(HashMap<String, NilaiConfig>),
    Null,
}

impl NilaiConfig {
    fn sebagai_teks(&self) -> Option<&str> {
        match self {
            NilaiConfig::Teks(s) => Some(s),
            _                   => None,
        }
    }

    fn sebagai_nombor(&self) -> Option<f64> {
        match self {
            NilaiConfig::Nombor(n) => Some(*n),
            _                     => None,
        }
    }

    fn sebagai_bool(&self) -> Option<bool> {
        match self {
            NilaiConfig::Bool(b) => Some(*b),
            _                   => None,
        }
    }
}

impl std::fmt::Display for NilaiConfig {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            NilaiConfig::Teks(s)    => write!(f, "\"{}\"", s),
            NilaiConfig::Nombor(n)  => write!(f, "{}", n),
            NilaiConfig::Bool(b)    => write!(f, "{}", b),
            NilaiConfig::Null       => write!(f, "null"),
            NilaiConfig::Senarai(v) => {
                write!(f, "[")?;
                for (i, item) in v.iter().enumerate() {
                    if i > 0 { write!(f, ", ")?; }
                    write!(f, "{}", item)?;
                }
                write!(f, "]")
            }
            NilaiConfig::Jadual(m) => {
                write!(f, "{{")?;
                for (i, (k, v)) in m.iter().enumerate() {
                    if i > 0 { write!(f, ", ")?; }
                    write!(f, "{}: {}", k, v)?;
                }
                write!(f, "}}")
            }
        }
    }
}

// ─── Parser ───────────────────────────────────────────────────

fn parse_nilai(s: &str) -> NilaiConfig {
    let s = s.trim();
    match s {
        "null" | "NULL" | ""   => NilaiConfig::Null,
        "true" | "yes" | "1"  => NilaiConfig::Bool(true),
        "false" | "no" | "0"  => NilaiConfig::Bool(false),
        _ if s.starts_with('"') && s.ends_with('"') => {
            NilaiConfig::Teks(s[1..s.len()-1].to_string())
        }
        _ if s.starts_with('[') && s.ends_with(']') => {
            let dalam = &s[1..s.len()-1];
            let elems: Vec<NilaiConfig> = dalam
                .split(',')
                .map(|e| parse_nilai(e.trim()))
                .collect();
            NilaiConfig::Senarai(elems)
        }
        _ => match s.parse::<f64>() {
            Ok(n)  => NilaiConfig::Nombor(n),
            Err(_) => NilaiConfig::Teks(s.to_string()),
        }
    }
}

fn parse_config(teks: &str) -> HashMap<String, NilaiConfig> {
    let mut config = HashMap::new();

    for baris in teks.lines() {
        let baris = baris.trim();

        // Skip komen dan baris kosong
        match baris {
            "" => continue,
            b if b.starts_with('#') => continue,
            _ => {}
        }

        // Parse KEY=VALUE
        if let Some((kunci, nilai_str)) = baris.split_once('=') {
            let kunci = kunci.trim().to_string();
            let nilai = parse_nilai(nilai_str.trim());
            config.insert(kunci, nilai);
        }
    }
    config
}

// ─── Config Accessor ──────────────────────────────────────────

struct Config {
    data: HashMap<String, NilaiConfig>,
}

impl Config {
    fn muat(teks: &str) -> Self {
        Config { data: parse_config(teks) }
    }

    fn dapatkan(&self, kunci: &str) -> Option<&NilaiConfig> {
        self.data.get(kunci)
    }

    fn teks(&self, kunci: &str) -> Option<&str> {
        self.data.get(kunci)?.sebagai_teks()
    }

    fn nombor(&self, kunci: &str) -> Option<f64> {
        self.data.get(kunci)?.sebagai_nombor()
    }

    fn bool_val(&self, kunci: &str) -> Option<bool> {
        self.data.get(kunci)?.sebagai_bool()
    }

    fn nombor_atau(&self, kunci: &str, lalai: f64) -> f64 {
        self.nombor(kunci).unwrap_or(lalai)
    }

    fn teks_atau<'a>(&'a self, kunci: &str, lalai: &'a str) -> &'a str {
        self.teks(kunci).unwrap_or(lalai)
    }

    fn papar(&self) {
        let mut kunci: Vec<&String> = self.data.keys().collect();
        kunci.sort();
        println!("\n{'═'*40}");
        println!("{:^40}", "KONFIGURASI");
        println!("{'═'*40}");
        for k in kunci {
            println!("  {:<20} = {}", k, self.data[k]);
        }
        println!("{'═'*40}");
    }
}

// ─── Main ──────────────────────────────────────────────────────

fn main() {
    let fail_config = r#"
# Konfigurasi Aplikasi KADA
# Versi 1.0

app_nama     = "KADA Mobile API"
app_versi    = "2.1.0"
app_debug    = false
app_port     = 8080

# Database
db_host      = "localhost"
db_port      = 5432
db_nama      = "kada_db"
db_ssl       = true
db_pool_min  = 2
db_pool_max  = 20

# Cache
cache_aktif  = true
cache_ttl    = 300
cache_prefix = "kada:"

# Fitur
fitur_gps    = true
fitur_selfie = true
fitur_notif  = false

# Senarai admin
admin_roles  = ["admin", "super_admin", "ICT"]
    "#;

    let config = Config::muat(fail_config);
    config.papar();

    println!("\n=== Akses Config ===");

    // Destructure dari config
    match config.dapatkan("app_port") {
        Some(NilaiConfig::Nombor(port)) => println!("Port: {}", port),
        Some(nilai) => println!("Port dalam format lain: {}", nilai),
        None        => println!("Port tidak ditetapkan"),
    }

    // Guna accessor helpers
    println!("Nama app: {}", config.teks_atau("app_nama", "Unknown"));
    println!("Debug:    {:?}", config.bool_val("app_debug"));
    println!("DB Host:  {}", config.teks_atau("db_host", "localhost"));
    println!("Pool max: {}", config.nombor_atau("db_pool_max", 10.0));

    // Destructure senarai dari config
    match config.dapatkan("admin_roles") {
        Some(NilaiConfig::Senarai(roles)) => {
            println!("\nPeranan admin ({} peranan):", roles.len());
            for role in roles {
                match role {
                    NilaiConfig::Teks(s) => println!("  - {}", s),
                    lain                 => println!("  - {:?}", lain),
                }
            }
        }
        _ => println!("Tiada senarai peranan"),
    }

    // Semak fitur
    println!("\n=== Status Fitur ===");
    for fitur in &["gps", "selfie", "notif", "bluetooth"] {
        let kunci = format!("fitur_{}", fitur);
        match config.bool_val(&kunci) {
            Some(true)  => println!("  ✔ {}", fitur),
            Some(false) => println!("  ✘ {}", fitur),
            None        => println!("  ? {} (tidak ditetapkan)", fitur),
        }
    }
}
```

---

# 📋 Rujukan Pantas — Destructuring Cheat Sheet

## Semua Jenis Destructuring

```rust
// Tuple
let (a, b, c) = (1, 2, 3);
let (x, .., z) = (1, 2, 3, 4, 5); // x=1, z=5
let (a, _, c) = (1, 2, 3);        // ignore tengah

// Struct
let Titik { x, y } = titik;
let Titik { x: a, y: b } = titik;  // rename
let Titik { x, .. } = titik;        // ignore baki

// Enum
match e {
    Some(val)          => ..,        // tuple variant
    Ok(val)            => ..,
    Struct { field }   => ..,        // struct variant
    Unit               => ..,        // unit variant
}

// Array (fixed size)
let [a, b, c] = [1, 2, 3];

// Slice (dynamic)
match slice {
    []            => ..,
    [x]           => ..,
    [a, b]        => ..,
    [h, rest @ ..] => ..,
    [.., tail]    => ..,
}

// Nested
let Outer { inner: Inner { x, y }, .. } = outer;
let (Some(a), Some(b)) = (opt1, opt2);

// Function params
fn f((x, y): (i32, i32)) { .. }
fn g(Struct { field, .. }: &Struct) { .. }

// Closure params
vec.iter().map(|(k, v)| ..)
vec.iter().map(|Struct { f, .. }| ..)

// Reference
let &(a, b) = ref_to_tuple;
let ref x = val;      // x: &Type
```

## Kata Kunci Penting

```
_     → ignore nilai (tidak bind)
..    → ignore berbilang elemen (slice/tuple/struct)
ref   → bind by reference (elak move)
@     → bind DAN test pattern
|     → multiple patterns (OR)
if    → guard — tambah syarat
```

## Panduan Pilih

```
Nak ambil field dari struct?   → let Struct { field, .. } = val
Nak buka Option?               → if let Some(x) = opt
Nak buka Result?               → if let Ok(v) = res
Nak handle banyak kes?         → match val { Pat1 => .., Pat2 => .. }
Nak balik awal jika gagal?     → let Pat = val else { return; }
Nak filter + ambil?            → filter_map(|x| if let Pat = x { .. })
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **CSV Parser** — parse setiap baris CSV guna slice patterns + destructure
2. **Expression Evaluator** — destructure AST nested untuk evaluate
3. **State Machine** — destructure (State, Event) tuple untuk transition
4. **Protocol Parser** — bina parser untuk protocol binary guna slice patterns
5. **JSON Differ** — bandingkan dua `NilaiConfig` dan kenal pasti perbezaan

---

*Destructuring in Rust — dari `let (a, b) = ...` hingga nested patterns kompleks. Selamat belajar!* 🦀
