# 🔗 References & Lifetimes dalam Rust — Beginner to Advanced

> Panduan lengkap: borrow, &T, &mut T, lifetime annotations, dan elision rules.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Kenapa References & Lifetimes Wujud?

```
Masalah tanpa references:

  fn cetak(s: String) {      // ambil ownership!
      println!("{}", s);
  }

  let nama = String::from("Ali");
  cetak(nama);               // nama DIPINDAH ke fungsi
  println!("{}", nama);      // ← ERROR! nama dah gone

Penyelesaian — BORROW (pinjam) dengan &:

  fn cetak(s: &String) {     // pinjam sahaja!
      println!("{}", s);
  }

  let nama = String::from("Ali");
  cetak(&nama);              // pinjam, tidak pindah
  println!("{}", nama);      // ✔ nama masih ada!
```

---

## Apa Itu Lifetime?

```
Lifetime = "tempoh hidup" sesuatu nilai dalam memori.

  {                           ← scope mula
      let nama = String::from("Ali");   ← nama lahir
      let r = &nama;          ← r pinjam nama
      println!("{}", r);      ← r guna nama
  }                           ← nama mati, r pun mati

Rust perlu pastikan:
  Reference r TIDAK hidup lebih lama dari yang dipinjam (nama)!
  Ini dipanggil "dangling reference" — sangat bahaya dalam C/C++!
```

---

## Peta Pembelajaran

```
Bab 1  → Borrow & References Asas
Bab 2  → Peraturan Borrowing
Bab 3  → Slice References
Bab 4  → Lifetime Annotations Asas
Bab 5  → Lifetime dalam Fungsi
Bab 6  → Lifetime dalam Struct
Bab 7  → Lifetime Elision Rules
Bab 8  → Static Lifetime & 'static bound
Bab 9  → Advanced Lifetime Patterns
Bab 10 → Mini Project: Text Analyser
```

---

# BAB 1: Borrow & References Asas 🔗

## & — Immutable Reference

```rust
fn main() {
    let nombor = 42i32;
    let teks   = String::from("Hello, Rust!");

    // & = buat reference (pinjam, tidak pindah ownership)
    let r1 = &nombor;     // r1 adalah &i32
    let r2 = &teks;       // r2 adalah &String

    println!("nombor = {}", nombor);  // nombor masih boleh guna
    println!("r1     = {}", r1);      // melalui reference

    println!("teks = {}", teks);      // teks masih boleh guna
    println!("r2   = {}", r2);        // melalui reference

    // Guna reference dalam fungsi — TIDAK pindah ownership
    fn panjang(s: &String) -> usize {
        s.len()  // s dipinjam, tidak owned
    }             // s di-drop DI SINI — tapi hanya reference yang drop, bukan data!

    let p = panjang(&teks);
    println!("Panjang: {}", p);
    println!("teks masih ada: {}", teks);  // ✔ teks masih valid!

    // * = dereference (ambil nilai dari reference)
    let x = 5;
    let rx = &x;
    println!("rx = {}", rx);   // Rust auto-deref untuk println
    println!("rx = {}", *rx);  // explicit deref — sama hasilnya
    assert_eq!(x, *rx);        // compare nilai
}
```

---

## &mut — Mutable Reference

```rust
fn main() {
    let mut nilai = 10;

    // &mut = pinjam DENGAN kebenaran untuk ubah
    let r = &mut nilai;
    *r += 5;             // ubah melalui mutable reference
    println!("Selepas +5: {}", r); // 15

    // Penting: selepas r selesai, nilai boleh guna semula
    println!("nilai terus: {}", nilai); // 15

    // Guna dalam fungsi
    fn tambah_satu(n: &mut i32) {
        *n += 1;
    }

    let mut a = 100;
    tambah_satu(&mut a);
    tambah_satu(&mut a);
    tambah_satu(&mut a);
    println!("a = {}", a); // 103

    // Ubah String melalui &mut
    fn tambah_seru(s: &mut String) {
        s.push_str("!!!");
    }

    let mut kata = String::from("Hello");
    tambah_seru(&mut kata);
    println!("{}", kata); // Hello!!!
}
```

---

## Visualisasi Borrow

```
IMMUTABLE BORROW (&T):

  Nilai:   [ "Hello" ]
             ↑     ↑     ↑
            &s1   &s2   &s3    ← boleh banyak &T serentak!

MUTABLE BORROW (&mut T):

  Nilai:   [ "Hello" ]
                ↑
             &mut s           ← hanya SATU &mut T!
                               tiada &T lain boleh wujud serentak

Ini macam:
  &T    = Buku perpustakaan — ramai boleh baca serentak
  &mut T = Buku untuk edit — hanya satu editor pada satu masa
```

---

## 🧠 Brain Teaser #1

Mengapa kod ini tidak compile?

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;       // pinjam immutable
    let r2 = &s;       // pinjam immutable lagi

    s.push_str(", world"); // ubah s

    println!("{} {}", r1, r2); // guna r1 dan r2
}
```

<details>
<summary>👀 Jawapan</summary>

**Error:** Cannot borrow `s` as mutable because it is also borrowed as immutable.

```
  let r1 = &s;           // r1 borrow immutable s
  let r2 = &s;           // r2 borrow immutable s
  s.push_str(...);       // Cuba ubah s — tapi r1 dan r2 masih aktif!
  println!("{}", r1);    // r1 masih guna s di sini
```

Peraturan: Tidak boleh ada `&mut` borrow semasa ada `&` borrow yang masih aktif.

**Penyelesaian:** Pastikan immutable borrows selesai sebelum buat mutable borrow:

```rust
let mut s = String::from("hello");

let r1 = &s;
let r2 = &s;
println!("{} {}", r1, r2); // r1, r2 selesai di sini

s.push_str(", world");     // OK — tiada borrow aktif
println!("{}", s);
```

Rust 2021 edition: Non-Lexical Lifetimes (NLL) — borrow berakhir bila **last use**, bukan hujung scope!
</details>

---

# BAB 2: Peraturan Borrowing 📏

## Dua Peraturan Borrow Checker

```
PERATURAN 1 — Pada satu-satu masa:
  ┌─────────────────────────────────────────────────┐
  │  SAMA ADA:                                      │
  │    Banyak & (immutable references)              │
  │  ATAU:                                          │
  │    Tepat SATU &mut (mutable reference)          │
  │  TIDAK BOLEH kedua-dua serentak!                │
  └─────────────────────────────────────────────────┘

PERATURAN 2 — No dangling:
  ┌─────────────────────────────────────────────────┐
  │  Reference mesti sentiasa VALID                 │
  │  (tidak boleh tunjuk ke data yang dah mati)     │
  └─────────────────────────────────────────────────┘
```

```rust
fn main() {
    let mut s = String::from("hello");

    // ✔ Banyak immutable borrow — OK
    let r1 = &s;
    let r2 = &s;
    let r3 = &s;
    println!("{} {} {}", r1, r2, r3); // semuanya selesai

    // ✔ Satu mutable borrow — OK
    let r4 = &mut s;
    r4.push_str(" world");
    println!("{}", r4); // r4 selesai

    // ✔ Selepas r4 selesai, boleh buat borrow baru
    let r5 = &s;
    println!("{}", r5);

    // ❌ Immutable + mutable serentak — ERROR
    // let r6 = &s;
    // let r7 = &mut s;     // ERROR! r6 masih aktif
    // println!("{} {}", r6, r7);
}
```

---

## Dangling Reference — Rust Cegah!

```rust
// ❌ Ini tidak compile — dangling reference!
// fn buat_dangling() -> &String {
//     let s = String::from("hello");
//     &s   // return reference ke s
// }        // s di-drop di sini! reference tunjuk ke TIADA APA!

// ✔ Betul — return owned value
fn buat_string() -> String {
    let s = String::from("hello");
    s  // pindah ownership keluar, tidak drop
}

// ✔ Betul — reference hidup selagi data hidup
fn ambil_ref<'a>(s: &'a str) -> &'a str {
    s  // return reference yang sama — lifetime sama
}

fn main() {
    let s = buat_string();
    println!("{}", s);

    let teks = String::from("world");
    let r = ambil_ref(&teks);
    println!("{}", r); // r valid — teks masih hidup
}
```

---

## Copy Types vs Reference

```rust
fn main() {
    // Copy types (i32, f64, bool, char, dll)
    // — auto-copy, tidak perlu &
    let x = 5;
    let y = x;       // COPY, bukan move
    println!("{} {}", x, y); // kedua-dua masih valid!

    fn kuasa_dua(n: i32) -> i32 { n * n } // ambil copy, bukan borrow
    let hasil = kuasa_dua(x);
    println!("{}² = {}", x, hasil); // x masih valid!

    // Non-Copy types (String, Vec, dll)
    // — perlu & atau clone untuk tidak lose ownership
    let v = vec![1, 2, 3];

    fn jumlah_v(v: &Vec<i32>) -> i32 { v.iter().sum() } // borrow
    let j = jumlah_v(&v);
    println!("Jumlah: {}, v masih ada: {:?}", j, v);

    // Atau guna slice (lebih idiomatik):
    fn jumlah_s(v: &[i32]) -> i32 { v.iter().sum() }
    println!("Jumlah: {}", jumlah_s(&v));
}
```

---

# BAB 3: Slice References 🍕

## String Slice — &str

```
String:           [ H | e | l | l | o |   | W | o | r | l | d ]
                    0   1   2   3   4   5   6   7   8   9  10

&s[0..5] = "Hello"    ← slice dari index 0 hingga 4
&s[6..11] = "World"   ← slice dari index 6 hingga 10
&s[..]    = "Hello World" ← seluruh string
```

```rust
fn main() {
    let s = String::from("Hello World");

    // Buat string slice
    let hello = &s[0..5];   // "Hello"
    let world = &s[6..11];  // "World"
    let semua = &s[..];     // "Hello World"
    let akhir = &s[6..];    // "World"
    let awal  = &s[..5];    // "Hello"

    println!("{} {} {}", hello, world, semua);

    // &str — string literal ADALAH slice!
    let literal: &str = "Hello, Rust!"; // simpan dalam binary, always valid

    // Fungsi yang terima &str boleh terima &String (auto-coerce)
    fn pertama_perkataan(s: &str) -> &str {
        let bytes = s.as_bytes();
        for (i, &b) in bytes.iter().enumerate() {
            if b == b' ' { return &s[..i]; }
        }
        &s[..]
    }

    let s1 = String::from("hello world");
    let s2 = "goodbye moon";

    println!("{}", pertama_perkataan(&s1)); // "hello" — &String → &str
    println!("{}", pertama_perkataan(s2));  // "goodbye" — &str terus
    println!("{}", pertama_perkataan("hi there")); // literal
}
```

---

## Array & Vec Slices

```rust
fn main() {
    let arr = [1, 2, 3, 4, 5];
    let v   = vec![10, 20, 30, 40, 50];

    // Array slice
    let bahagian: &[i32] = &arr[1..4]; // [2, 3, 4]
    println!("{:?}", bahagian);

    // Vec slice
    let tiga_pertama: &[i32] = &v[..3]; // [10, 20, 30]
    println!("{:?}", tiga_pertama);

    // Fungsi yang terima slice — berfungsi untuk array DAN vec!
    fn jumlah(data: &[i32]) -> i32 {
        data.iter().sum()
    }

    println!("Array sum: {}", jumlah(&arr));          // 15
    println!("Vec sum:   {}", jumlah(&v));             // 150
    println!("Slice sum: {}", jumlah(&arr[1..4]));     // 9
    println!("Slice sum: {}", jumlah(&v[2..]));        // 90

    // Mutable slice
    let mut nums = vec![3, 1, 4, 1, 5];
    let bahagian_mut: &mut [i32] = &mut nums[1..4];
    bahagian_mut.sort();
    println!("{:?}", nums); // [3, 1, 1, 4, 5]
}
```

---

## 🧠 Brain Teaser #2

Kenapa fungsi ini tidak compile, dan bagaimana betulkan?

```rust
fn pertama_perkataan(s: &String) -> &str {
    let perkataan = s.split_whitespace().next().unwrap_or("");
    perkataan
}

fn main() {
    let hasil;
    {
        let teks = String::from("hello world");
        hasil = pertama_perkataan(&teks);
    } // teks di-drop di sini!
    println!("{}", hasil); // dangling!
}
```

<details>
<summary>👀 Jawapan</summary>

**Error:** `teks` tidak hidup cukup lama — `hasil` digunakan selepas `teks` di-drop.

Walaupun fungsi `pertama_perkataan` adalah OK, masalahnya adalah dalam `main`: kita cuba guna `hasil` (yang tunjuk ke dalam `teks`) selepas `teks` keluar scope.

**Penyelesaian 1:** Extend lifetime `teks` ke scope yang sama dengan `hasil`:
```rust
fn main() {
    let teks = String::from("hello world");
    let hasil = pertama_perkataan(&teks);
    println!("{}", hasil); // ✔ teks masih hidup!
}
```

**Penyelesaian 2:** Return owned `String` bukan reference:
```rust
fn pertama_perkataan(s: &str) -> String {
    s.split_whitespace().next().unwrap_or("").to_string()
}

fn main() {
    let hasil;
    {
        let teks = String::from("hello world");
        hasil = pertama_perkataan(&teks); // owned String, tidak bergantung pada teks
    }
    println!("{}", hasil); // ✔ hasil adalah String sendiri
}
```
</details>

---

# BAB 4: Lifetime Annotations Asas ⏳

## Bila Perlu Lifetime Annotation?

```
Bila compiler tidak boleh tahu lifetime sendiri:

  fn terpanjang(s1: &str, s2: &str) -> &str {
      // Compiler tanya: return &str ini tunjuk ke s1 atau s2?
      // Lifetime return bergantung pada RUNTIME — compiler tidak tahu!
      if s1.len() > s2.len() { s1 } else { s2 }
  }

  ERROR: missing lifetime specifier

Penyelesaian — beritahu compiler hubungan lifetime:

  fn terpanjang<'a>(s1: &'a str, s2: &'a str) -> &'a str {
      if s1.len() > s2.len() { s1 } else { s2 }
  }

  'a = "output hidup sekurang-kurangnya selama input yang paling pendek"
```

---

## Lifetime Annotations dalam Fungsi

```rust
// 'a = nama lifetime (konvensyen: huruf kecil, biasanya 'a, 'b, 'c)
fn terpanjang<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() > s2.len() {
        s1
    } else {
        s2
    }
}

fn main() {
    let s1 = String::from("panjang sekali");
    let hasil;

    {
        let s2 = String::from("xyz");
        hasil = terpanjang(s1.as_str(), s2.as_str());
        println!("Terpanjang: {}", hasil);
        // ✔ OK — s1 dan s2 kedua-duanya hidup dalam scope ini
    }

    // Ini TIDAK BOLEH:
    // let hasil;
    // {
    //     let s2 = String::from("xyz");
    //     hasil = terpanjang(s1.as_str(), s2.as_str());
    // } // s2 mati!
    // println!("{}", hasil); // ERROR: s2 tidak hidup cukup lama

    // Contoh mudah — return reference ke satu sahaja
    fn pertama<'a>(s1: &'a str, _s2: &str) -> &'a str {
        s1 // sentiasa return s1, lifetime hanya perlu match s1
    }

    let a = String::from("hello");
    let b = String::from("world");
    let r = pertama(&a, &b);
    println!("{}", r);
}
```

---

## Apa 'a Sebenarnya Bermaksud?

```
fn terpanjang<'a>(s1: &'a str, s2: &'a str) -> &'a str

Maksudnya:
  "Ada suatu lifetime 'a.
   s1 akan hidup sekurang-kurangnya selama 'a.
   s2 akan hidup sekurang-kurangnya selama 'a.
   Return value akan hidup selama 'a.

   'a = lifetime yang LEBIH PENDEK antara s1 dan s2."

Bukan bermaksud s1 dan s2 mesti ada lifetime yang SAMA.
Bermaksud return value tidak boleh hidup LEBIH LAMA dari mana-mana input.
```

```rust
fn main() {
    let s1 = String::from("lama");  // lifetime: 'seluruh main'

    let hasil = {
        let s2 = String::from("cd"); // lifetime: 'dalam scope ini sahaja'
        terpanjang(s1.as_str(), s2.as_str())
        // 'a = lifetime s2 (yang lebih pendek)
        // hasil mesti tidak hidup lebih dari s2
    };
    // 'a berakhir di sini (s2 drop), jadi hasil pun tidak valid

    // println!("{}", hasil); // ERROR! hasil tidak hidup cukup lama
}

fn terpanjang<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() > s2.len() { s1 } else { s2 }
}
```

---

# BAB 5: Lifetime dalam Fungsi 🔧

## Pelbagai Lifetime Parameters

```rust
// Dua lifetime berbeza — s1 dan s2 boleh ada lifetime berbeza
// Return bergantung pada s1 sahaja
fn pilih_pertama<'a, 'b>(s1: &'a str, _s2: &'b str) -> &'a str {
    s1  // return s1, so 'b tidak perlu match return type
}

// Tiga parameter, return dari dua kemungkinan
fn pilih<'a, 'b: 'a>(s1: &'a str, s2: &'b str, guna_pertama: bool) -> &'a str {
    // 'b: 'a bermaksud 'b hidup sekurang-kurangnya selama 'a
    if guna_pertama { s1 } else { s2 }
}

fn main() {
    let lama  = String::from("ini lebih panjang");
    let hasil;

    {
        let pendek = String::from("ab");
        hasil = pilih_pertama(lama.as_str(), pendek.as_str());
        // 'a = lifetime lama, 'b = lifetime pendek
        // hasil lifetime = 'a (lifetime lama)
    } // pendek drop, tapi hasil masih valid kerana bergantung pada lama

    println!("{}", hasil); // ✔ OK!
}
```

---

## Lifetime dengan Generic Types

```rust
use std::fmt::Display;

// Gabungan lifetime + generic + trait bound
fn cetak_jika_panjang<'a, T>(
    x:        &'a str,
    y:        &'a str,
    tambahan: T,
) -> &'a str
where
    T: Display,
{
    println!("Tambahan: {}", tambahan);
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let s1 = String::from("hello world");
    let s2 = String::from("hi");

    let hasil = cetak_jika_panjang(
        s1.as_str(),
        s2.as_str(),
        42,  // T = i32
    );
    println!("Terpanjang: {}", hasil);
}
```

---

## 🧠 Brain Teaser #3

Ada berapa banyak lifetime annotation yang diperlukan untuk fungsi ini? Kenapa?

```rust
fn tiga_fungsi_berbeza() {
    // Fungsi 1: ambil satu &str, return &str
    fn satu(s: &str) -> &str {
        s
    }

    // Fungsi 2: ambil dua &str, return &str (bergantung pada kedua-dua)
    fn dua(s1: &str, s2: &str) -> &str {
        if s1.len() > s2.len() { s1 } else { s2 }
    }

    // Fungsi 3: ambil dua &str, return void (tidak return reference)
    fn tiga(s1: &str, s2: &str) {
        println!("{} {}", s1, s2);
    }
}
```

<details>
<summary>👀 Jawapan</summary>

- **Fungsi 1:** `0` lifetime annotation diperlukan!
  → Lifetime Elision Rule: satu input reference = auto-apply lifetime ke output.
  Compiler faham: output bergantung pada input.

- **Fungsi 2:** `1` lifetime annotation diperlukan (`<'a>`)!
  → Compiler tidak boleh tahu output bergantung pada s1 atau s2.
  Kena explicit: `fn dua<'a>(s1: &'a str, s2: &'a str) -> &'a str`

- **Fungsi 3:** `0` lifetime annotation diperlukan!
  → Tiada reference dalam output, tiada ambiguiti.
  Compiler tidak perlu tahu hubungan lifetime.

**Peraturan mudah:** Lifetime annotation HANYA perlu bila ada reference dalam output DAN ada lebih dari satu reference dalam input.
</details>

---

# BAB 6: Lifetime dalam Struct 🏗️

## Struct yang Menyimpan References

```rust
// Struct yang simpan &str — perlu lifetime!
#[derive(Debug)]
struct Petikan<'a> {
    kata:   &'a str,  // pinjam string dari luar
    penulis: &'a str,
}

impl<'a> Petikan<'a> {
    fn baru(kata: &'a str, penulis: &'a str) -> Self {
        Petikan { kata, penulis }
    }

    fn papar(&self) {
        println!("\"{}\" — {}", self.kata, self.penulis);
    }

    // Return reference dari struct — lifetime method
    fn kata_pertama(&self) -> &str {
        self.kata.split_whitespace().next().unwrap_or("")
        // Lifetime elision: input &self, output auto-tied ke &self lifetime
    }
}

fn main() {
    let novel = String::from(
        "Buku ni sangat best. Saya suka membaca."
    );
    let penulis = String::from("Ali Ahmad");

    // Petikan simpan &str yang menunjuk ke novel dan penulis
    let petikan = Petikan::baru(
        novel.split('.').next().expect("Perlu ada titik"),
        &penulis,
    );

    petikan.papar();
    println!("Kata pertama: {}", petikan.kata_pertama());

    // Petikan bergantung pada novel — tidak boleh outlive novel!
    drop(novel); // 'novel' drop dulu, jadi 'petikan' pun tak boleh guna lagi

    // println!("{:?}", petikan); // ERROR! novel dah drop
}
```

---

## Struct dengan Multiple Lifetimes

```rust
// Dua bahagian dengan lifetime berbeza
#[derive(Debug)]
struct Teks<'a, 'b> {
    kepala: &'a str,  // bahagian pertama
    ekor:   &'b str,  // bahagian kedua — boleh dari sumber berbeza!
}

impl<'a, 'b> Teks<'a, 'b> {
    fn gabung(&self) -> String {
        format!("{} {}", self.kepala, self.ekor)
    }
}

fn main() {
    let s1 = String::from("Hello");

    let teks;
    {
        let s2 = String::from("World");
        teks = Teks {
            kepala: &s1,   // lifetime 'a = lifetime s1
            ekor:   &s2,   // lifetime 'b = lifetime s2
        };
        println!("{}", teks.gabung()); // ✔ kedua-dua s1, s2 masih hidup
    } // s2 drop — teks tidak boleh guna 'ekor' lagi

    // Hanya boleh guna kepala (s1 masih hidup)
    // println!("{}", teks.ekor); // ERROR!
    println!("{}", teks.kepala);  // ✔ s1 masih hidup
}
```

---

## Lifetime dalam impl Block

```rust
#[derive(Debug)]
struct Parser<'a> {
    input: &'a str,
    posisi: usize,
}

impl<'a> Parser<'a> {
    fn baru(input: &'a str) -> Self {
        Parser { input, posisi: 0 }
    }

    // Return &str yang bergantung pada 'input' — tied ke 'a lifetime struct
    fn seterusnya_perkataan(&mut self) -> Option<&'a str> {
        let mula = self.posisi;
        while self.posisi < self.input.len() {
            if self.input.as_bytes()[self.posisi] == b' ' {
                let perkataan = &self.input[mula..self.posisi];
                self.posisi += 1;
                return Some(perkataan);
            }
            self.posisi += 1;
        }
        if mula < self.input.len() {
            Some(&self.input[mula..])
        } else {
            None
        }
    }
}

fn main() {
    let teks = String::from("hello world rust programming");
    let mut parser = Parser::baru(&teks);

    while let Some(perkataan) = parser.seterusnya_perkataan() {
        println!("Perkataan: '{}'", perkataan);
    }
}
```

---

# BAB 7: Lifetime Elision Rules 🧹

## Tiga Peraturan Elision

```
Lifetime Elision = compiler auto-tambah lifetime annotation
                   mengikut peraturan, tidak perlu tulis manual

Peraturan 1 — SETIAP input reference dapat lifetime berbeza:
  fn f(s: &str, t: &str)
  →  fn f<'a, 'b>(s: &'a str, t: &'b str)

Peraturan 2 — Kalau ada SATU input lifetime, output dapat lifetime yang sama:
  fn f(s: &str) -> &str
  →  fn f<'a>(s: &'a str) -> &'a str

Peraturan 3 — Kalau ada &self atau &mut self, output dapat lifetime self:
  fn kaedah(&self, s: &str) -> &str
  →  fn kaedah<'a, 'b>(&'a self, s: &'b str) -> &'a str
```

```rust
// Semua contoh di bawah TIDAK perlu annotation manual:

// Peraturan 2: satu input → output tied ke input
fn pertama_bait(s: &str) -> &str {
    s.split_whitespace().next().unwrap_or("")
}

// Peraturan 3: method dengan &self → output tied ke self
struct Teks { isi: String }
impl Teks {
    fn perwakilan(&self, tambah: &str) -> &str {
        // Output &str bergantung pada &self, bukan 'tambah'
        // Compiler tahu ini kerana ada &self — Peraturan 3
        self.isi.as_str()
    }
}

// Perlu annotation — dua input &str, output &str (ambiguiti)
fn gabung_pendek<'a>(s1: &'a str, s2: &str) -> &'a str {
    // Explicitly: return bergantung pada s1 sahaja
    &s1[..s1.len().min(3)]
}

fn main() {
    let t = Teks { isi: String::from("hello") };
    let tambahan = String::from("world");
    let hasil = t.perwakilan(&tambahan);
    println!("{}", hasil);
}
```

---

# BAB 8: Static Lifetime & 'static Bound ♾️

## 'static — Hidup Selama Program

```rust
fn main() {
    // String literals = 'static — disimpan dalam binary
    let s: &'static str = "Hello, world!";

    // 'static lifetime bermaksud data hidup selama program jalan
    static PESAN: &str = "Selamat datang ke KADA!";
    println!("{}", PESAN);

    // Fungsi yang return 'static
    fn nama_versi() -> &'static str {
        "v2.1.0"  // literal — 'static
    }
    println!("Versi: {}", nama_versi());

    // 'static dalam Box
    let s: Box<dyn std::fmt::Display + 'static> = Box::new(42);
    println!("{}", s);
}
```

---

## 'static Bound dalam Generics

```rust
use std::thread;

// T: 'static bermaksud T tidak mengandungi reference dengan lifetime pendek
// (boleh selamat digunakan dalam thread yang mungkin hidup lama)
fn jalankan_dalam_thread<T: std::fmt::Debug + Send + 'static>(val: T) {
    thread::spawn(move || {
        println!("Thread: {:?}", val);
    }).join().unwrap();
}

fn main() {
    // i32 adalah 'static (tiada reference)
    jalankan_dalam_thread(42i32);

    // String adalah 'static (owned, tiada borrowed reference)
    jalankan_dalam_thread(String::from("hello"));

    // Vec<i32> adalah 'static
    jalankan_dalam_thread(vec![1, 2, 3]);

    // &str literal adalah 'static
    jalankan_dalam_thread("literal str");

    // &String TIDAK 'static — bergantung pada String luar
    // let s = String::from("borrowed");
    // jalankan_dalam_thread(&s); // ERROR! &s tidak 'static
}
```

---

## 🧠 Brain Teaser #4

Apakah perbezaan antara `&'static str` dan `String`?

```rust
fn main() {
    let a: &'static str = "hello";   // A
    let b: String       = String::from("hello");  // B

    // Bila A lebih sesuai?
    // Bila B lebih sesuai?
}
```

<details>
<summary>👀 Jawapan</summary>

```
&'static str:
  ✔ Data simpan dalam binary program
  ✔ Sangat laju (tiada heap allocation)
  ✔ Boleh share merentasi thread tanpa clone
  ✔ Sesuai untuk: literal, const, static values
  ✘ Tidak boleh diubah
  ✘ Tidak boleh dibina semasa runtime

String:
  ✔ Boleh dibina/diubah semasa runtime
  ✔ Boleh tambah, tukar, padam kandungan
  ✔ Sesuai untuk: user input, generated strings, dynamic content
  ✘ Heap allocation (sedikit lebih lambat)
  ✘ Perlu clone untuk share

Panduan:
  "Nilai akan selalu sama, tahu semasa compile?" → &'static str
  "Nilai bergantung pada runtime atau perlu diubah?" → String
```
</details>

---

# BAB 9: Advanced Lifetime Patterns 🎯

## Variance — Covariant vs Contravariant vs Invariant

```rust
// COVARIANT: &'a T — boleh guna 'longer bagi tempat yang jangka 'shorter
fn demo_covariant() {
    let s: &'static str = "hello";   // 'static (paling panjang)
    let r: &str = s;                  // ✔ OK — 'static coerce ke lifetime pendek
}

// INVARIANT: &'a mut T — MESTI sama persis!
fn demo_invariant() {
    let mut s1 = String::from("hello");
    let mut s2 = String::from("world");

    let r1: &mut String = &mut s1;
    // r1 = &mut s2; // ← Ini OK (reassign reference)

    // Tapi dalam fungsi, &mut T adalah invariant pada T:
    fn ubah(s: &mut &str) {
        *s = "lain";
    }
    // Invariant kerana boleh "menyimpan" reference berbeza lifetime
}
```

---

## Higher-Ranked Trait Bounds (HRTB)

```rust
// for<'a> — "untuk SEMUA lifetime 'a"
// Berguna bila closure perlu berfungsi dengan pelbagai lifetime

fn guna_closure<F>(f: F, s: &str)
where
    F: for<'a> Fn(&'a str) -> &'a str,
    // Bermaksud: F boleh terima &str dengan MANA-MANA lifetime
{
    println!("{}", f(s));
}

fn main() {
    guna_closure(|s| s, "hello");
    guna_closure(|s| &s[..3], "goodbye"); // "goo"

    // Contoh lebih praktikal — sort dengan comparator
    let mut v = vec!["banana", "apple", "cherry"];
    v.sort_by(|a: &&str, b: &&str| a.cmp(b));
    println!("{:?}", v);
}
```

---

## Lifetime Subtyping

```rust
// 'b: 'a — 'b hidup sekurang-kurangnya selama 'a
// Baca: "'b outlives 'a"

fn pertama_atau_kedua<'a, 'b: 'a>(
    s1: &'a str,
    s2: &'b str,
    guna_s1: bool,
) -> &'a str {
    if guna_s1 {
        s1
    } else {
        s2  // ✔ OK: 'b: 'a bermaksud s2 hidup sekurang-kurangnya selama s1
    }
}

fn main() {
    let s1 = String::from("hello");
    let hasil;

    {
        let s2 = String::from("world");
        // s2 hidup lebih pendek dari s1 — 'b: 'a terpenuhi jika s2 >= s1 lifetime
        // Dalam kes ini, kita perlu s2 hidup selama kita guna hasil
        hasil = pertama_atau_kedua(&s1, &s2, false);
        println!("{}", hasil); // ✔ dalam scope s2
    }

    // println!("{}", hasil); // ✘ s2 dah drop
}
```

---

## NLL — Non-Lexical Lifetimes

```rust
fn main() {
    let mut data = vec![1, 2, 3];

    // Tanpa NLL (Rust lama), ini TIDAK compile:
    // Borrow dianggap aktif sehingga HUJUNG scope
    let r = &data[0];          // borrow start
    println!("r = {}", r);     // last use of r
    // Dengan NLL: borrow BERAKHIR di sini (selepas last use)

    data.push(4);              // ✔ OK dengan NLL! r tidak aktif lagi
    println!("{:?}", data);

    // Contoh lebih kompleks
    let mut s = String::from("hello");
    let kata = s.split_whitespace().next().unwrap_or(""); // &str borrow s
    println!("{}", kata);  // last use of kata

    s.push_str(" world"); // ✔ OK selepas last use kata
    println!("{}", s);
}
```

---

# BAB 10: Mini Project — Text Analyser 📊

```rust
use std::collections::HashMap;

// ─── Struct dengan Lifetimes ─────────────────────────────────

#[derive(Debug)]
struct Dokumen<'a> {
    tajuk:     &'a str,
    kandungan: &'a str,
    penulis:   &'a str,
}

impl<'a> Dokumen<'a> {
    fn baru(tajuk: &'a str, kandungan: &'a str, penulis: &'a str) -> Self {
        Dokumen { tajuk, kandungan, penulis }
    }

    fn bilangan_perkataan(&self) -> usize {
        self.kandungan.split_whitespace().count()
    }

    fn bilangan_aksara(&self) -> usize {
        self.kandungan.chars().count()
    }

    fn bilangan_baris(&self) -> usize {
        self.kandungan.lines().count()
    }
}

// ─── Analyser dengan references ──────────────────────────────

struct Analyser<'a> {
    dokumen: Vec<Dokumen<'a>>,
}

impl<'a> Analyser<'a> {
    fn baru() -> Self {
        Analyser { dokumen: Vec::new() }
    }

    fn tambah(&mut self, dok: Dokumen<'a>) {
        self.dokumen.push(dok);
    }

    fn terpanjang(&self) -> Option<&Dokumen<'a>> {
        self.dokumen.iter().max_by_key(|d| d.bilangan_perkataan())
    }

    fn terpendek(&self) -> Option<&Dokumen<'a>> {
        self.dokumen.iter().min_by_key(|d| d.bilangan_perkataan())
    }

    fn cari_oleh_penulis(&self, penulis: &str) -> Vec<&Dokumen<'a>> {
        self.dokumen.iter()
            .filter(|d| d.penulis == penulis)
            .collect()
    }

    fn frekuensi_perkataan(&self) -> HashMap<&str, usize> {
        let mut freq: HashMap<&str, usize> = HashMap::new();
        for dok in &self.dokumen {
            for perkataan in dok.kandungan.split_whitespace() {
                // Trim tanda baca
                let bersih = perkataan.trim_matches(|c: char| !c.is_alphabetic());
                if !bersih.is_empty() {
                    *freq.entry(bersih).or_insert(0) += 1;
                }
            }
        }
        freq
    }

    fn laporan_ringkas(&self) {
        println!("\n{'═'*55}");
        println!("{:^55}", "LAPORAN ANALISIS TEKS");
        println!("{'═'*55}");
        println!("Jumlah dokumen: {}", self.dokumen.len());

        let jumlah_kata: usize = self.dokumen.iter()
            .map(|d| d.bilangan_perkataan())
            .sum();
        println!("Jumlah perkataan: {}", jumlah_kata);

        println!("\n{:-<55}", "");
        println!("{:<5} {:<20} {:<10} {:<10}",
            "Bil", "Tajuk", "Perkataan", "Aksara");
        println!("{:-<55}", "");

        for (i, d) in self.dokumen.iter().enumerate() {
            println!("{:<5} {:<20} {:<10} {:<10}",
                i + 1,
                &d.tajuk[..d.tajuk.len().min(20)],
                d.bilangan_perkataan(),
                d.bilangan_aksara());
        }
        println!("{:-<55}", "");
    }

    fn top_perkataan(&self, n: usize) {
        let mut freq: Vec<(&&str, &usize)> = self.frekuensi_perkataan()
            .iter()
            .collect::<Vec<_>>()
            .into_iter()
            .collect();

        // Sort by frequency descending
        freq.sort_by(|a, b| b.1.cmp(a.1));

        println!("\nTop {} perkataan:", n);
        for (perkataan, kali) in freq.iter().take(n) {
            println!("  {:<20} {}x", perkataan, kali);
        }
    }
}

// ─── Helper function dengan lifetime ─────────────────────────

fn cari_ayat_dengan<'a>(teks: &'a str, cari: &str) -> Vec<&'a str> {
    teks.lines()
        .filter(|baris| baris.contains(cari))
        .collect()
}

fn perkataan_terpanjang_dalam<'a>(teks: &'a str) -> Option<&'a str> {
    teks.split_whitespace()
        .max_by_key(|w| w.len())
}

// ─── Main ─────────────────────────────────────────────────────

fn main() {
    // Data — simpan dalam variables dengan lifetime yang cukup panjang
    let tajuk1  = "Laporan Tahunan KADA 2024";
    let tajuk2  = "Panduan Penanaman Padi";
    let tajuk3  = "Pekeliling ICT Terbaru";

    let kandungan1 = "\
        KADA telah berjaya meningkatkan pengeluaran padi tahun ini. \
        Program bantuan benih dan baja telah membantu petani tempatan. \
        Jumlah keluasan tanaman meningkat kepada dua puluh ribu hektar. \
        Kerjasama dengan petani di Kemubu terus diperkukuhkan. \
        KADA komited untuk membangunkan sektor pertanian padi negara.";

    let kandungan2 = "\
        Penanaman padi memerlukan persiapan tanah yang rapi dan teliti. \
        Petani perlu memastikan sistem pengairan berfungsi dengan baik. \
        Benih padi berkualiti tinggi perlu dipilih untuk hasil maksimum. \
        Pembajaan dilakukan mengikut jadual yang ditetapkan pakar KADA. \
        Musim menuai biasanya berlaku dua kali setahun di kawasan Kemubu.";

    let kandungan3 = "\
        Semua kakitangan KADA perlu menggunakan sistem KADA Mobile App. \
        Sistem GPS digunakan untuk pengesahan lokasi kehadiran harian. \
        Gambar selfie diperlukan untuk pengesahan identiti pekerja ICT. \
        ICT Unit menyediakan sokongan teknikal untuk semua pengguna sistem. \
        Sebarang masalah teknikal perlu dilaporkan melalui sistem tiket.";

    let penulis1 = "Jabatan Perancangan";
    let penulis2 = "Jabatan Pertanian";
    let penulis3 = "ICT Unit";

    // Buat analyser dan tambah dokumen
    let mut analyser = Analyser::baru();

    analyser.tambah(Dokumen::baru(tajuk1, kandungan1, penulis1));
    analyser.tambah(Dokumen::baru(tajuk2, kandungan2, penulis2));
    analyser.tambah(Dokumen::baru(tajuk3, kandungan3, penulis3));

    // Laporan
    analyser.laporan_ringkas();
    analyser.top_perkataan(5);

    // Cari dokumen
    println!("\n=== Terpanjang & Terpendek ===");
    if let Some(d) = analyser.terpanjang() {
        println!("Terpanjang: {} ({} patah kata)",
            d.tajuk, d.bilangan_perkataan());
    }
    if let Some(d) = analyser.terpendek() {
        println!("Terpendek: {} ({} patah kata)",
            d.tajuk, d.bilangan_perkataan());
    }

    // Cari oleh penulis
    println!("\n=== Dokumen oleh ICT Unit ===");
    for d in analyser.cari_oleh_penulis("ICT Unit") {
        println!("  {} — {} baris",
            d.tajuk, d.bilangan_baris());
    }

    // Guna helper functions dengan lifetime
    println!("\n=== Ayat mengandungi 'KADA' ===");
    let ayat_kada = cari_ayat_dengan(kandungan1, "KADA");
    for a in &ayat_kada {
        println!("  → {}", a.trim());
    }

    println!("\n=== Perkataan terpanjang ===");
    for (tajuk, kandungan) in &[
        (tajuk1, kandungan1),
        (tajuk2, kandungan2),
        (tajuk3, kandungan3),
    ] {
        if let Some(panjang) = perkataan_terpanjang_dalam(kandungan) {
            println!("  {}: '{}'", tajuk, panjang);
        }
    }

    println!("\n{'═'*55}");
    println!("Analisis selesai!");
}
```

---

# 📋 Rujukan Pantas — References & Lifetimes Cheat Sheet

## References

```rust
&T           // immutable reference — baca sahaja
&mut T       // mutable reference — baca dan tulis

*r           // dereference (ambil nilai)
&val         // buat reference
&mut val     // buat mutable reference

// Peraturan:
// 1. Banyak &T ATAU satu &mut T (tidak boleh campur, tidak serentak)
// 2. Reference mesti valid (tidak dangling)
```

## Lifetime Annotations

```rust
// Fungsi
fn f<'a>(s: &'a str) -> &'a str { s }
fn f<'a>(s1: &'a str, s2: &'a str) -> &'a str { .. }
fn f<'a, 'b: 'a>(s1: &'a str, s2: &'b str) -> &'a str { .. }

// Struct
struct S<'a> { s: &'a str }
impl<'a> S<'a> { fn method(&self) -> &str { .. } }

// Lifetime subtyping: 'b: 'a = 'b outlives 'a

// 'static = hidup selama program
let s: &'static str = "literal";
```

## Elision Rules (Bila Tidak Perlu Annotate)

```
Rule 1: Setiap input reference → lifetime berbeza
Rule 2: Satu input ref → output dapat lifetime yang sama
Rule 3: &self atau &mut self → output dapat lifetime self

Perlu annotate bila:
  - Lebih satu input reference + ada output reference
  - Dan bukan method (&self rule tidak apply)
```

## Slice Types

```rust
&str         // string slice
&[T]         // array/vec slice
&mut [T]     // mutable slice

s.as_str()   // String → &str
v.as_slice() // Vec<T> → &[T]
```

## Cara Fikir Lifetime

```
"Adakah reference ini mungkin outlive nilai yang dipinjam?"
  Ya  → compiler akan ERROR, perlu refactor
  Tidak → OK

"Return reference bergantung pada input mana?"
  Semua input sama lifetime → satu 'a cukup
  Bergantung pada satu input → annotate dengan lifetime input tersebut
  Tidak bergantung pada input → return owned value
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **String Pool** — struct yang cache &str dan return reference ke dalam pool
2. **Chain Iterator** — struct `Chain<'a>` yang iterate dua slice serentak
3. **Memoize** — cache function results menggunakan reference ke input sebagai key
4. **Diff Tool** — bandingkan dua `&str` dan return `Vec<&str>` baris yang berbeza
5. **Mini Regex** — simple pattern matching yang return `Option<&str>` slice yang match

---

*References & Lifetimes in Rust — dari `&T` mudah hingga HRTB dan NLL. Selamat belajar!* 🦀
