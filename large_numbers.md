# 🔢 Large Numbers dalam Rust — Panduan Lengkap

> Bagaimana Rust handle nombor besar: dari integer overflow
> hingga arbitrary precision arithmetic.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Masalah Nombor Besar

```
Integer biasa ada HAD!

  u8:    0 hingga 255
  u16:   0 hingga 65,535
  u32:   0 hingga 4,294,967,295 (~4 billion)
  u64:   0 hingga 18,446,744,073,709,551,615 (~18 quintillion)
  u128:  0 hingga 340,282,366,920,938,463,463,374,607,431,768,211,455
  i128:  ±170,141,183,460,469,231,731,687,303,715,884,105,727

  Lebih dari itu? → OVERFLOW!

Contoh masalah:
  1000! (faktorial 1000) → beratus digit
  Kriptografi (RSA) → nombor 2048 bit
  Scientific computing → 10^1000
  Kewangan → 1.234567890123456789 (presisi desimal penuh)
```

---

## Peta Pembelajaran

```
Bahagian 1  → Jenis Integer Rust
Bahagian 2  → Integer Overflow — Apa Berlaku?
Bahagian 3  → Checked Arithmetic
Bahagian 4  → Saturating Arithmetic
Bahagian 5  → Wrapping Arithmetic
Bahagian 6  → Overflowing Arithmetic
Bahagian 7  → u128 & i128 — Built-in Besar
Bahagian 8  → Float Precision Issues
Bahagian 9  → Crate untuk Nombor Besar (BigInt, Decimal)
Bahagian 10 → Mini Project: Kalkulator Presisi Tinggi
```

---

# BAHAGIAN 1: Jenis Integer Rust 🔢

## Semua Integer Types

```rust
fn main() {
    // ── Unsigned integers (tiada negatif) ─────────────────────
    let _a: u8   = 255;                     // 0 .. 2^8-1
    let _b: u16  = 65_535;                  // 0 .. 2^16-1
    let _c: u32  = 4_294_967_295;           // 0 .. 2^32-1
    let _d: u64  = 18_446_744_073_709_551_615; // 0 .. 2^64-1
    let _e: u128 = 340_282_366_920_938_463_463_374_607_431_768_211_455; // 0 .. 2^128-1
    let _f: usize = usize::MAX;             // platform-dependent (32 atau 64 bit)

    // ── Signed integers (ada negatif) ─────────────────────────
    let _g: i8   = 127;                     // -128 .. 127
    let _h: i16  = 32_767;                  // -32768 .. 32767
    let _i: i32  = 2_147_483_647;           // -2^31 .. 2^31-1
    let _j: i64  = 9_223_372_036_854_775_807; // -2^63 .. 2^63-1
    let _k: i128 = 170_141_183_460_469_231_731_687_303_715_884_105_727;
    let _l: isize = isize::MAX;

    // ── Float ─────────────────────────────────────────────────
    let _m: f32 = 3.402_823_5e38_f32;      // ±3.4×10^38 (7 digit precision)
    let _n: f64 = 1.797_693_134_862_315_7e308_f64; // ±1.8×10^308 (15-17 digit)

    // ── Limits ────────────────────────────────────────────────
    println!("u8 MAX:   {}", u8::MAX);    // 255
    println!("u32 MAX:  {}", u32::MAX);   // 4294967295
    println!("u64 MAX:  {}", u64::MAX);   // 18446744073709551615
    println!("u128 MAX: {}", u128::MAX);  // 340282366920938463463374607431768211455
    println!("i32 MIN:  {}", i32::MIN);   // -2147483648
    println!("i32 MAX:  {}", i32::MAX);   // 2147483647
    println!("f64 MAX:  {:.2e}", f64::MAX); // 1.80e308
}
```

---

## Pilih Type yang Sesuai

```
SOAL: Type mana yang sesuai?

Gaji pekerja (RM):
  → f64 (ada desimal, tapi ada precision issue!)
  → Atau i64 dalam sen (lebih tepat!)

ID pekerja (1 hingga 10,000):
  → u32 (lebih dari cukup, 4 billion)

Umur (0 hingga 150):
  → u8 (0-255, lebih dari cukup)

Timestamp Unix (saat dari 1970):
  → i64 (boleh negatif untuk tarikh sebelum 1970)

Saiz fail:
  → u64 (sehingga 18 exabytes)

Counter requests (boleh jutaan):
  → u64 (atau u32 kalau < 4 billion)

Kryptography:
  → u128 atau BigInt crate

Saintifik / Kewangan tinggi:
  → BigDecimal / Decimal crate
```

---

# BAHAGIAN 2: Integer Overflow — Apa Berlaku? 💥

## Debug Mode vs Release Mode

```rust
fn main() {
    // ── DEBUG MODE (cargo build / cargo run) ──────────────────
    // Integer overflow → PANIC!
    // Ini membantu detect bug semasa development

    let x: u8 = 255;
    // let y = x + 1; // ← PANIC: attempt to add with overflow!
    // thread 'main' panicked at 'attempt to add with overflow'

    // ── RELEASE MODE (cargo build --release) ─────────────────
    // Integer overflow → WRAP AROUND (tiada panic!)
    // 255u8 + 1 = 0 (wraps around)
    // Ini boleh jadi SILENT BUG yang susah detect!

    // Demonstrasi wrapping (explicitly):
    let wrapped = 255u8.wrapping_add(1);
    println!("{}", wrapped); // 0

    // ── KENAPA BERBEZA? ────────────────────────────────────────
    // Debug mode: safety first — panic bila ada overflow
    // Release mode: performance first — wrapping (macam C)
    // Anda mesti EXPLICITLY pilih cara yang anda nak!
}
```

---

## 🧠 Brain Teaser #1

Apakah output kod ini dalam DEBUG mode dan RELEASE mode?

```rust
fn main() {
    let a: i32 = i32::MAX; // 2,147,483,647
    let b: i32 = a + 1;
    println!("{}", b);
}
```

<details>
<summary>👀 Jawapan</summary>

**DEBUG mode:** PANIC!
```
thread 'main' panicked at 'attempt to add with overflow', src/main.rs:3:20
```

**RELEASE mode:** `-2147483648` (i32::MIN)

Kenapa -2147483648?
Dalam binary, `i32::MAX` adalah `0111_1111_1111_1111_1111_1111_1111_1111`.
Tambah 1 → `1000_0000_0000_0000_0000_0000_0000_0000` = `i32::MIN` dalam two's complement!

**Kesimpulan:**
- Debug: safe, detect bug, tapi panic
- Release: laju, tapi SILENT BUG yang boleh menyebabkan kelemahan keselamatan!

**Cara betul:** Guna checked/saturating/wrapping arithmetic secara explicit.
</details>

---

# BAHAGIAN 3: Checked Arithmetic ✅

## checked_* — Return Option

```rust
fn main() {
    // checked_* return Option<T>
    // Some(hasil) → berjaya tanpa overflow
    // None → overflow berlaku!

    let a: u8 = 200;
    let b: u8 = 100;

    // checked_add
    println!("{:?}", a.checked_add(b));   // None (200+100=300 > 255)
    println!("{:?}", 100u8.checked_add(50)); // Some(150)

    // checked_sub
    println!("{:?}", 50u8.checked_sub(100)); // None (50-100 < 0)
    println!("{:?}", 100u8.checked_sub(50)); // Some(50)

    // checked_mul
    println!("{:?}", 200u8.checked_mul(2));  // None (200×2=400 > 255)
    println!("{:?}", 10u8.checked_mul(20));  // Some(200)

    // checked_div
    println!("{:?}", 100u8.checked_div(0));  // None (bahagi dengan 0!)
    println!("{:?}", 100u8.checked_div(4));  // Some(25)

    // checked_pow
    println!("{:?}", 10u32.checked_pow(9)); // None (10^9 = 1B > u32::MAX? no, OK)
    println!("{:?}", 10u32.checked_pow(10)); // None (10^10 = 10B > 4B)

    // checked_neg (untuk signed)
    println!("{:?}", i32::MIN.checked_neg()); // None (negasi MIN overflow!)
    println!("{:?}", (-5i32).checked_neg());  // Some(5)

    // ── Guna dalam kod sebenar ────────────────────────────────
    fn tambah_selamat(a: u32, b: u32) -> u32 {
        a.checked_add(b)
            .expect("Overflow dalam tambah_selamat!")
    }

    // Atau dengan Result:
    fn tambah_result(a: u32, b: u32) -> Result<u32, &'static str> {
        a.checked_add(b).ok_or("Integer overflow!")
    }

    println!("{:?}", tambah_result(100, 200));       // Ok(300)
    println!("{:?}", tambah_result(u32::MAX, 1));    // Err("Integer overflow!")
}
```

---

## Checked dalam Real Code

```rust
fn kira_jumlah_gaji(gaji: &[u64]) -> Option<u64> {
    // Kalau jumlah overflow u64, return None
    gaji.iter().try_fold(0u64, |acc, &g| acc.checked_add(g))
}

fn kira_faktorial_checked(n: u64) -> Option<u64> {
    (1..=n).try_fold(1u64, |acc, i| acc.checked_mul(i))
}

fn main() {
    let gaji = vec![4500u64, 5000, 3800, 6000, 4200];
    match kira_jumlah_gaji(&gaji) {
        Some(jumlah) => println!("Jumlah: RM{}", jumlah),
        None         => println!("Overflow! Jumlah terlalu besar"),
    }

    // Faktorial
    for n in [5, 10, 20, 21] {
        match kira_faktorial_checked(n) {
            Some(hasil) => println!("{}! = {}", n, hasil),
            None        => println!("{}! = OVERFLOW (> u64::MAX)", n),
        }
    }
}
// Output:
// Jumlah: RM23500
// 5! = 120
// 10! = 3628800
// 20! = 2432902008176640000
// 21! = OVERFLOW (> u64::MAX)
```

---

# BAHAGIAN 4: Saturating Arithmetic 🧱

## saturating_* — Clamp ke MIN/MAX

```rust
fn main() {
    // saturating_* → kalau overflow, hasilnya adalah MIN atau MAX
    // Tidak panic, tidak wrap — "tepu" di sempadan

    let a: u8 = 200;
    let b: u8 = 100;

    // saturating_add
    println!("{}", a.saturating_add(b));    // 255 (bukan 44 atau panic!)
    println!("{}", 100u8.saturating_add(50)); // 150

    // saturating_sub
    println!("{}", 50u8.saturating_sub(100)); // 0 (bukan wrapping atau panic!)
    println!("{}", 100u8.saturating_sub(50)); // 50

    // saturating_mul
    println!("{}", 200u8.saturating_mul(2)); // 255 (max)
    println!("{}", 10u8.saturating_mul(20)); // 200

    // signed — saturate ke i32::MIN atau i32::MAX
    println!("{}", i32::MAX.saturating_add(1));  // 2147483647 (MAX, tidak overflow!)
    println!("{}", i32::MIN.saturating_sub(1));  // -2147483648 (MIN, tidak overflow!)
    println!("{}", i32::MIN.saturating_neg());   // 2147483647 (bukan overflow!)

    // ── Guna yang berguna ─────────────────────────────────────
    // Slider / progress bar — nilai tidak boleh lebih dari max
    struct ProgressBar { nilai: u32, maks: u32 }
    impl ProgressBar {
        fn tambah(&mut self, n: u32) {
            // Nilai tidak akan melebihi maks!
            self.nilai = self.nilai.saturating_add(n).min(self.maks);
        }
    }

    // Health points dalam game
    struct Watak { hp: i32, hp_max: i32 }
    impl Watak {
        fn terima_kerosakan(&mut self, dmg: i32) {
            self.hp = self.hp.saturating_sub(dmg).max(0);
        }
        fn sembuh(&mut self, heal: i32) {
            self.hp = self.hp.saturating_add(heal).min(self.hp_max);
        }
    }
}
```

---

## Bila Guna Saturating?

```
Saturating sesuai untuk:
  ✔ Health points, stamina (tidak boleh < 0 atau > max)
  ✔ Progress bar (0% hingga 100%)
  ✔ Volume audio (0 hingga 100)
  ✔ Counter yang ada batas atas
  ✔ Signal processing (clip audio amplitude)
  ✔ Mana-mana nilai yang ada julat logik
```

---

# BAHAGIAN 5: Wrapping Arithmetic 🔄

## wrapping_* — Intentional Wrap

```rust
fn main() {
    // wrapping_* → explicitly wrap around (macam release mode)
    // Guna bila wrapping adalah INTENDED behavior!

    let x: u8 = 255;
    println!("{}", x.wrapping_add(1));  // 0   (255 + 1 → wrap)
    println!("{}", x.wrapping_add(10)); // 9   (255 + 10 → wrap)

    let y: u8 = 0;
    println!("{}", y.wrapping_sub(1));  // 255 (0 - 1 → wrap)

    println!("{}", i32::MAX.wrapping_add(1)); // i32::MIN
    println!("{}", i32::MIN.wrapping_sub(1)); // i32::MAX

    // wrapping_mul
    println!("{}", 200u8.wrapping_mul(2)); // 144 (400 mod 256)

    // wrapping_pow
    println!("{}", 2u32.wrapping_pow(32)); // 0 (2^32 mod 2^32)

    // wrapping_neg
    println!("{}", 5i32.wrapping_neg());   // -5
    println!("{}", i32::MIN.wrapping_neg()); // i32::MIN (overflow wraps!)

    // ── Guna yang berguna ─────────────────────────────────────

    // Hash functions — sengaja guna wrapping!
    fn simple_hash(s: &str) -> u32 {
        s.bytes().fold(0u32, |acc, b| {
            acc.wrapping_mul(31).wrapping_add(b as u32)
        })
    }
    println!("Hash 'hello': {}", simple_hash("hello"));

    // Cyclic counter — 0,1,2,...,255,0,1,2,...
    let mut counter: u8 = 0;
    for _ in 0..260 {
        print!("{} ", counter);
        counter = counter.wrapping_add(1);
    }
    println!();
}
```

---

# BAHAGIAN 6: Overflowing Arithmetic ℹ️

## overflowing_* — Return Value dan Flag

```rust
fn main() {
    // overflowing_* → return (hasil, did_overflow: bool)
    // Berguna bila nak tahu sama ada overflow berlaku atau tidak
    // tapi masih nak nilai wrapped

    let (hasil, overflow) = 200u8.overflowing_add(100);
    println!("hasil: {}, overflow: {}", hasil, overflow);
    // hasil: 44, overflow: true

    let (hasil2, overflow2) = 100u8.overflowing_add(50);
    println!("hasil: {}, overflow: {}", hasil2, overflow2);
    // hasil: 150, overflow: false

    let (hasil3, overflow3) = i32::MAX.overflowing_add(1);
    println!("hasil: {}, overflow: {}", hasil3, overflow3);
    // hasil: -2147483648, overflow: true

    // overflowing_mul
    let (h, o) = 200u8.overflowing_mul(2);
    println!("200 * 2 = {} (overflow: {})", h, o); // 144, true

    // Guna dalam multi-precision arithmetic (advanced)
    fn tambah_u128_extended(a: u128, b: u128) -> (u128, bool) {
        a.overflowing_add(b)
    }
}
```

---

## Ringkasan: Pilih Kaedah Yang Sesuai

```
Situasi                           Guna
──────────────────────────────────────────────────────────
"Overflow = bug, nak panic"        Debug mode (default)
"Overflow = bug, nak handle"       checked_* → Option/Result
"Overflow = clamp ke min/max"      saturating_*
"Overflow = wrap (intended)"       wrapping_*
"Nak tahu sama ada overflow?"      overflowing_*
"Nilai lebih besar dari u128"      BigInt crate
"Presisi desimal penuh"            Decimal/BigDecimal crate

Panduan mudah:
  Wang / kewangan        → Decimal atau i64 dalam sen
  Counter/ID             → u64 atau u32
  Age/size/count kecil   → u8, u16, u32
  Scientific             → f64 atau BigDecimal
  Kriptografi            → BigInt / num-bigint
  Hash functions         → wrapping_*
  Health/progress        → saturating_*
  Anything critical      → checked_*
```

---

# BAHAGIAN 7: u128 & i128 — Besar Built-in 🦣

## Guna u128 untuk Nombor Besar

```rust
fn main() {
    // u128 — 128 bit unsigned
    // Maksimum: 340,282,366,920,938,463,463,374,607,431,768,211,455
    //           (~3.4 × 10^38)

    let besar: u128 = 340_282_366_920_938_463_463_374_607_431_768_211_455;
    println!("{}", besar);
    println!("{}", u128::MAX);

    // Faktorial dengan u128
    fn faktorial_u128(n: u64) -> Option<u128> {
        (1..=n as u128).try_fold(1u128, |acc, i| acc.checked_mul(i))
    }

    for n in [10u64, 20, 30, 34, 35] {
        match faktorial_u128(n) {
            Some(f) => println!("{}! = {}", n, f),
            None    => println!("{}! = OVERFLOW (> u128::MAX)", n),
        }
    }
    // 10! = 3628800
    // 20! = 2432902008176640000
    // 30! = 265252859812191058636308480000000
    // 34! = 295232799039604140847618609643520000000
    // 35! = OVERFLOW

    // UUID — biasa disimpan sebagai u128!
    let uuid: u128 = 0x550e8400_e29b_41d4_a716_446655440000;
    println!("UUID bytes: {:#034x}", uuid);

    // Timestamp nanosecond
    let ns: u128 = std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .unwrap()
        .as_nanos();
    println!("Timestamp ns: {}", ns);
}
```

---

# BAHAGIAN 8: Float Precision Issues ⚠️

## Masalah Presisi Float

```rust
fn main() {
    // MASALAH: Float tidak boleh represent semua nombor desimal tepat!

    let a = 0.1f64;
    let b = 0.2f64;
    let c = a + b;

    println!("{}", c);           // 0.30000000000000004 (bukan 0.3!)
    println!("{}", c == 0.3f64); // false! (bukan anda silap)

    // Kenapa?
    // 0.1 dalam binary ≈ 0.0001100110011001100110011... (tidak habis!)
    // Macam 1/3 = 0.333... dalam desimal

    // ── CARA COMPARE FLOAT ────────────────────────────────────

    // ❌ JANGAN compare float dengan ==
    // if 0.1 + 0.2 == 0.3 { ... }  // SALAH!

    // ✔ Guna epsilon comparison
    const EPSILON: f64 = 1e-10;
    fn hampir_sama(a: f64, b: f64) -> bool {
        (a - b).abs() < EPSILON
    }

    println!("{}", hampir_sama(0.1 + 0.2, 0.3)); // true

    // ✔ Guna relative epsilon untuk nombor besar
    fn hampir_sama_relatif(a: f64, b: f64, epsilon: f64) -> bool {
        let diff = (a - b).abs();
        let max = a.abs().max(b.abs());
        if max == 0.0 {
            diff < epsilon
        } else {
            diff / max < epsilon
        }
    }

    // ── WANG: JANGAN GUNA FLOAT! ──────────────────────────────
    // ❌ SANGAT BAHAYA untuk kewangan
    let harga: f64 = 19.99;
    let cukai: f64 = harga * 0.06; // 6% GST
    println!("Cukai: {:.10}", cukai); // 1.1993999999999998 !

    // ✔ Guna integer dalam unit terkecil (sen)
    let harga_sen: i64 = 1999; // RM19.99 dalam sen
    let cukai_sen: i64 = harga_sen * 6 / 100; // 119 sen = RM1.19
    println!("Cukai: RM{:.2}", cukai_sen as f64 / 100.0); // RM1.19
}
```

---

# BAHAGIAN 9: Crate untuk Nombor Besar 📦

## num-bigint — Arbitrary Precision Integer

```toml
[dependencies]
num-bigint = "0.4"
num-traits = "0.2"
```

```rust
use num_bigint::{BigInt, BigUint};
use num_traits::{Zero, One, ToPrimitive};
use std::str::FromStr;

fn main() {
    // ── BigUint — arbitrary precision unsigned ─────────────────

    // Buat BigUint
    let a = BigUint::from(1_000_000_000u64);
    let b = BigUint::from(1_000_000_000u64);

    let hasil = &a * &b;
    println!("{}", hasil); // 1000000000000000000

    // Nombor yang lebih besar dari u128!
    let giga = BigUint::from(10u32).pow(30);
    println!("10^30 = {}", giga);

    // Parse dari string
    let besar = BigUint::from_str("123456789012345678901234567890").unwrap();
    println!("{}", besar);

    // ── Faktorial dengan BigUint ───────────────────────────────
    fn faktorial_besar(n: u64) -> BigUint {
        (1..=n).fold(BigUint::one(), |acc, i| acc * i)
    }

    println!("100! = {}", faktorial_besar(100));
    // Ratusan digit!

    println!("Panjang 1000!: {} digit", faktorial_besar(1000).to_string().len());
    // 1000! ada 2568 digit!

    // ── BigInt — dengan negatif ────────────────────────────────
    let x = BigInt::from(-1_000_000i64);
    let y = BigInt::from(3_000_000i64);
    println!("{}", &x + &y);  // 2000000
    println!("{}", &x * &y);  // -3000000000000

    // Convert balik ke primitive (kalau muat!)
    let kecil = BigUint::from(42u32);
    println!("{:?}", kecil.to_u64()); // Some(42)

    let tak_muat = faktorial_besar(25);
    println!("{:?}", tak_muat.to_u64()); // None (terlalu besar untuk u64)
}
```

---

## rust_decimal — Presisi Desimal untuk Wang

```toml
[dependencies]
rust_decimal          = "1"
rust_decimal_macros   = "1"
```

```rust
use rust_decimal::Decimal;
use rust_decimal_macros::dec;
use rust_decimal::prelude::*;

fn main() {
    // ── Decimal — fixed-point, tiada precision error! ──────────

    // Guna macro dec! untuk literal
    let harga = dec!(19.99);
    let cukai_kadar = dec!(0.06); // 6%

    let cukai = harga * cukai_kadar;
    println!("Harga:  {}", harga);  // 19.99
    println!("Cukai:  {}", cukai);  // 1.1994 (tepat!)
    println!("Jumlah: {}", harga + cukai); // 21.1894

    // Bulatkan ke 2 desimal (untuk wang)
    let cukai_bulatkan = cukai.round_dp(2);
    println!("Cukai dibulatkan: {}", cukai_bulatkan); // 1.20

    // Berbanding f64:
    let f_harga: f64 = 19.99;
    let f_cukai = f_harga * 0.06;
    println!("f64 cukai: {:.10}", f_cukai); // 1.1993999999998 (salah!)

    // ── Operasi asas ──────────────────────────────────────────
    let a = dec!(10.5);
    let b = dec!(3.2);
    println!("{}", a + b);  // 13.7
    println!("{}", a - b);  // 7.3
    println!("{}", a * b);  // 33.60
    println!("{}", a / b);  // 3.28125 (tepat!)

    // ── Parse dari string ─────────────────────────────────────
    let d = Decimal::from_str("123.456789").unwrap();
    println!("{}", d); // 123.456789

    // ── Convert ke/dari f64 ───────────────────────────────────
    let f = 3.14f64;
    let d = Decimal::from_f64(f).unwrap();
    println!("{}", d); // 3.14

    let balik: f64 = d.to_f64().unwrap();
    println!("{}", balik); // 3.14

    // ── Guna untuk kira gaji ──────────────────────────────────
    let gaji_asas = dec!(4500.00);
    let elaun     = dec!(500.00);
    let potongan  = dec!(300.00);
    let cukai_pc  = dec!(0.11);   // 11% tax

    let gaji_kasar = gaji_asas + elaun;
    let cukai_amt  = gaji_kasar * cukai_pc;
    let gaji_bersih = gaji_kasar - cukai_amt - potongan;

    println!("Gaji kasar:  RM{}", gaji_kasar);
    println!("Cukai (11%): RM{}", cukai_amt.round_dp(2));
    println!("Potongan:    RM{}", potongan);
    println!("Gaji bersih: RM{}", gaji_bersih.round_dp(2));
}
```

---

## num-rational — Pecahan Tepat

```toml
[dependencies]
num-rational = "0.4"
num-bigint   = "0.4"
```

```rust
use num_rational::Ratio;
use num_bigint::BigInt;

fn main() {
    // Ratio<i32> — pecahan biasa
    let satu_pertiga = Ratio::new(1i32, 3);
    let satu_perempat = Ratio::new(1i32, 4);

    let hasil = satu_pertiga + satu_perempat;
    println!("{}", hasil);     // 7/12 (tepat!)
    println!("{:.6}", hasil.to_f64().unwrap()); // 0.583333

    // 1/3 + 1/3 + 1/3 = 1 (tepat!)
    let tiga_pertiga = satu_pertiga + satu_pertiga + satu_pertiga;
    println!("{}", tiga_pertiga); // 1/1 = 1 (tepat!)

    // f64 comparison:
    let f_sum = 1.0/3.0 + 1.0/3.0 + 1.0/3.0;
    println!("{:.20}", f_sum); // 1.00000000000000000000? (mungkin tidak!)

    // Ratio dengan BigInt untuk arbitrari precision
    let pi_approx = Ratio::<BigInt>::new(
        BigInt::from(355i32),
        BigInt::from(113i32)
    );
    println!("π ≈ {}", pi_approx); // 355/113
    println!("  = {:.10}", pi_approx.to_f64().unwrap()); // 3.1415929204
}
```

---

# BAHAGIAN 10: Mini Project — Kalkulator Presisi Tinggi 🧮

```toml
[dependencies]
num-bigint    = "0.4"
num-traits    = "0.2"
rust_decimal  = "1"
rust_decimal_macros = "1"
```

```rust
use num_bigint::BigUint;
use num_traits::{Zero, One};
use rust_decimal::Decimal;
use rust_decimal_macros::dec;
use rust_decimal::prelude::*;
use std::str::FromStr;
use std::fmt;

// ─── Kalkulator Matematik Asas ────────────────────────────────

fn faktorial(n: u64) -> BigUint {
    (1..=n).fold(BigUint::one(), |acc, i| acc * i)
}

fn fibonacci(n: u64) -> BigUint {
    if n == 0 { return BigUint::zero(); }
    if n == 1 { return BigUint::one(); }

    let mut a = BigUint::zero();
    let mut b = BigUint::one();
    for _ in 2..=n {
        let c = &a + &b;
        a = b;
        b = c;
    }
    b
}

fn kuasa_besar(asas: u64, eksponen: u64) -> BigUint {
    BigUint::from(asas).pow(eksponen as u32)
}

// ─── Kalkulator Kewangan ──────────────────────────────────────

struct KiraFaedah {
    prinsipal: Decimal,
    kadar:     Decimal, // dalam peratus, contoh: 3.5 untuk 3.5%
    tempoh:    u32,     // dalam tahun
}

impl KiraFaedah {
    fn faedah_mudah(&self) -> Decimal {
        // SI = P × R × T / 100
        self.prinsipal * self.kadar * Decimal::from(self.tempoh)
            / dec!(100)
    }

    fn faedah_kompaun_tahunan(&self) -> Decimal {
        // A = P × (1 + R/100)^T
        let kadar_desimal = self.kadar / dec!(100);
        let faktor = (dec!(1) + kadar_desimal)
            .powd(Decimal::from(self.tempoh));
        self.prinsipal * faktor
    }

    fn faedah_kompaun_bulanan(&self) -> Decimal {
        // A = P × (1 + R/(100×12))^(T×12)
        let kadar_bulanan = self.kadar / (dec!(100) * dec!(12));
        let tempoh_bulan = self.tempoh * 12;
        let faktor = (dec!(1) + kadar_bulanan)
            .powd(Decimal::from(tempoh_bulan));
        self.prinsipal * faktor
    }
}

impl fmt::Display for KiraFaedah {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Prinsipal RM{} @ {}% selama {} tahun",
               self.prinsipal, self.kadar, self.tempoh)
    }
}

// ─── Statistik Besar ──────────────────────────────────────────

fn statistik_desimal(data: &[Decimal]) -> Option<(Decimal, Decimal, Decimal)> {
    if data.is_empty() { return None; }

    let n = Decimal::from(data.len() as u64);
    let min = data.iter().copied().reduce(Decimal::min)?;
    let max = data.iter().copied().reduce(Decimal::max)?;
    let purata = data.iter().copied().sum::<Decimal>() / n;

    Some((min, max, purata))
}

// ─── Demo ─────────────────────────────────────────────────────

fn main() {
    println!("{'═'*60}");
    println!("{:^60}", "KALKULATOR PRESISI TINGGI");
    println!("{'═'*60}");

    // ── Matematik ─────────────────────────────────────────────
    println!("\n📐 MATEMATIK BESAR\n");

    for n in [10u64, 20, 50, 100] {
        let f = faktorial(n);
        let s = f.to_string();
        if s.len() <= 30 {
            println!("{}! = {}", n, s);
        } else {
            println!("{}! = {}...{} ({} digit)",
                n, &s[..10], &s[s.len()-5..], s.len());
        }
    }

    println!("\nFibonacci:");
    for n in [10u64, 50, 100, 200] {
        let fib = fibonacci(n);
        let s = fib.to_string();
        if s.len() <= 30 {
            println!("F({}) = {}", n, s);
        } else {
            println!("F({}) = {}...{} ({} digit)",
                n, &s[..10], &s[s.len()-5..], s.len());
        }
    }

    println!("\nKuasa besar:");
    println!("2^100 = {}", kuasa_besar(2, 100));
    println!("10^30 = {}", kuasa_besar(10, 30));

    // ── Kewangan ─────────────────────────────────────────────
    println!("\n💰 KIRA FAEDAH\n");

    let pelaburan = KiraFaedah {
        prinsipal: dec!(10000.00),
        kadar:     dec!(5.5),
        tempoh:    10,
    };

    println!("{}", pelaburan);
    println!("  Faedah mudah:             RM{:.2}", pelaburan.faedah_mudah());
    println!("  Faedah kompaun (tahunan): RM{:.2}",
        pelaburan.faedah_kompaun_tahunan().round_dp(2));
    println!("  Faedah kompaun (bulanan): RM{:.2}",
        pelaburan.faedah_kompaun_bulanan().round_dp(2));

    // Pinjaman kereta
    let pinjaman = KiraFaedah {
        prinsipal: dec!(80000.00),
        kadar:     dec!(3.5),
        tempoh:    9,
    };
    println!("\n{}", pinjaman);
    let jumlah = pinjaman.faedah_kompaun_bulanan();
    let bayaran = jumlah / Decimal::from(pinjaman.tempoh * 12);
    println!("  Jumlah bayaran: RM{:.2}", jumlah.round_dp(2));
    println!("  Ansuran bulanan: RM{:.2}", bayaran.round_dp(2));

    // ── Perbandingan presisi ──────────────────────────────────
    println!("\n🔬 PERBANDINGAN PRESISI\n");

    // f64 vs Decimal untuk kewangan
    let nilai_f64: f64 = 0.1 + 0.2 + 0.3 + 0.4;
    let nilai_dec = dec!(0.1) + dec!(0.2) + dec!(0.3) + dec!(0.4);

    println!("0.1 + 0.2 + 0.3 + 0.4:");
    println!("  f64:     {:.20}", nilai_f64);
    println!("  Decimal: {}", nilai_dec);
    println!("  Betul?   {}", nilai_dec == dec!(1.0));

    // GST calculation
    let harga_produk = dec!(99.90);
    let gst = harga_produk * dec!(0.06);
    println!("\nHarga: RM{}", harga_produk);
    println!("GST 6%: RM{}", gst.round_dp(2));
    println!("Jumlah: RM{}", (harga_produk + gst).round_dp(2));

    // ── Statistik ─────────────────────────────────────────────
    println!("\n📊 STATISTIK GAJI\n");

    let gaji: Vec<Decimal> = vec![
        dec!(4500.00), dec!(5000.00), dec!(3800.00),
        dec!(6000.00), dec!(4200.00), dec!(5500.00),
    ];

    if let Some((min, max, purata)) = statistik_desimal(&gaji) {
        println!("Gaji minimum: RM{}", min);
        println!("Gaji maksimum: RM{}", max);
        println!("Gaji purata:  RM{:.2}", purata.round_dp(2));
    }

    println!("\n{'═'*60}");
    println!("Selesai!");
}
```

---

# 📋 Rujukan Pantas — Large Numbers Cheat Sheet

## Integer Types & Limits

```
Type     Bit  Min                    Max
─────────────────────────────────────────────────────────
u8        8   0                      255
u16      16   0                      65,535
u32      32   0                      4,294,967,295
u64      64   0                      18,446,744,073,709,551,615
u128    128   0                      3.4 × 10^38 (39 digit)
i8        8   -128                   127
i16      16   -32,768                32,767
i32      32   -2,147,483,648        2,147,483,647
i64      64   -9.2 × 10^18          9.2 × 10^18
i128    128   -1.7 × 10^38          1.7 × 10^38
f32      32   -3.4 × 10^38          3.4 × 10^38 (7 digit precision)
f64      64   -1.8 × 10^308         1.8 × 10^308 (15-17 digit)
```

## Overflow Handling Methods

```rust
// Semak sama ada akan overflow SEBELUM buat operasi
a.checked_add(b)       // → Option<T>: None jika overflow
a.checked_sub(b)       // → Option<T>
a.checked_mul(b)       // → Option<T>
a.checked_div(b)       // → Option<T>: None jika div by 0
a.checked_pow(n)       // → Option<T>

// Tepu di sempadan (clamp ke MIN/MAX)
a.saturating_add(b)    // → T: MAX jika overflow, MIN jika underflow
a.saturating_sub(b)    // → T
a.saturating_mul(b)    // → T

// Wrap around (intentional overflow)
a.wrapping_add(b)      // → T: wrap around
a.wrapping_sub(b)      // → T
a.wrapping_mul(b)      // → T
a.wrapping_pow(n)      // → T

// Dapatkan nilai DAN flag overflow
a.overflowing_add(b)   // → (T, bool): (hasil, did_overflow)
a.overflowing_sub(b)   // → (T, bool)
a.overflowing_mul(b)   // → (T, bool)
```

## Crates untuk Nombor Besar

```toml
[dependencies]
# Arbitrary precision integer
num-bigint    = "0.4"   # BigInt, BigUint
num-traits    = "0.2"   # Zero, One, ToPrimitive

# Decimal presisi (untuk wang)
rust_decimal  = "1"     # Decimal type
rust_decimal_macros = "1" # dec!() macro

# Pecahan tepat
num-rational  = "0.4"   # Ratio<T>

# All-in-one math
num           = "0.4"   # include semua di atas
```

## Panduan Pilih

```
Data                     Type yang sesuai
─────────────────────────────────────────────────────────
Umur, bilangan kecil     u8, u16
ID, port, index          u32
Saiz fail, timestamp     u64
UUID                     u128
Kriptografi, RSA         BigUint (num-bigint)
Faktorial besar          BigUint
Wang / kewangan          Decimal (rust_decimal) / i64 sen
Saintifik normal         f64
Saintifik besar/tepat    BigDecimal
Pecahan tepat            Ratio (num-rational)
```

---

## 🏆 Cabaran Akhir

Cuba implement salah satu:

1. **BigInteger Calculator** — kalkulator yang boleh kira faktorial 1000, fibonacci 500, dan kuasa 2^256
2. **Kalkulator Faedah Kompaun Harian** — kira faedah dengan Decimal, dengan breakdown bulanan
3. **Wang Converter** — convert antara USD, MYR, EUR dengan Decimal (tiada precision loss!)
4. **Statistik Gaji** — min, max, median, purata, sisihan piawai dengan Decimal
5. **Overflow Detector** — fungsi yang terima dua nombor dan operasi, check sama ada overflow akan berlaku

---

*Large numbers dalam Rust — dari u8 yang kecil hingga BigInt yang tidak ada had.*
*Pilih type yang betul, handle overflow dengan bijak.* 🦀
