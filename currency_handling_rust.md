# 💰 Pengendalian Wang & Kadar Tukaran dalam Rust

> Simpan wang sebagai integer (unit terkecil), bukan float.
> Float = ralat pembulatan. Integer = tepat 100%.

---

## Kenapa Integer, Bukan Float?

```rust
// ❌ MASALAH dengan float
let harga = 0.1_f64 + 0.2_f64;
println!("{}", harga); // 0.30000000000000004 — SALAH!

// Wang: $1.50 + $1.70 = $3.20
let a = 1.50_f64;
let b = 1.70_f64;
println!("{:.10}", a + b); // 3.2000000000000002 — TIDAK TEPAT!

// ✔ PENYELESAIAN: simpan dalam sen (integer)
let a: i64 = 150; // $1.50
let b: i64 = 170; // $1.70
println!("{}", a + b); // 320 → $3.20 — TEPAT!
```

---

## Konsep Utama

```
MATA WANG       UNIT TERKECIL    DECIMAL    CONTOH
────────────────────────────────────────────────────
USD (Dolar)     Sen (cent)       2          $1.50  → 150
MYR (Ringgit)   Sen              2          RM4.50 → 450
EUR (Euro)      Cent             2          €2.99  → 299
GBP (Pound)     Penny            2          £1.00  → 100
JPY (Yen)       Yen (tiada sen)  0          ¥500   → 500
KWD (Dinar)     Fils             3          1.500  → 1500
BHD (Dinar)     Fils             3          0.500  → 500
OMR (Rial)      Baisa            3          1.250  → 1250
IQD (Dinar)     Fils             3          1.000  → 1000

KADAR TUKARAN: simpan × 10000 (4 decimal places)
  1 USD = 4.2356 MYR  →  simpan: 42356
  1 USD = 0.0063 JPY  →  simpan: 63  (dalam 10000 unit)
  Bahagi 10000 untuk dapat kadar sebenar
```

---

## Peta Kandungan

```
Bahagian 1  → Struct Mata Wang (Currency)
Bahagian 2  → Struct Wang (Money)
Bahagian 3  → Operasi Aritmetik
Bahagian 4  → Kadar Tukaran (ExchangeRate)
Bahagian 5  → Penukaran Mata Wang
Bahagian 6  → Pembulatan & Rounding
Bahagian 7  → Format Paparan
Bahagian 8  → Serde (JSON/API)
Bahagian 9  → Kes-kes Tepi (Edge Cases)
Bahagian 10 → Contoh Lengkap: Sistem Invois
```

---

# Bahagian 1: Struct Mata Wang

```rust
/// Definisi mata wang dengan decimal places
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct Currency {
    /// Kod ISO 4217 (contoh: "USD", "MYR", "JPY")
    pub code: &'static str,
    /// Simbol paparan (contoh: "$", "RM", "¥")
    pub symbol: &'static str,
    /// Bilangan decimal (minor unit exponent)
    /// USD = 2 (sen), JPY = 0, KWD = 3 (fils)
    pub decimals: u8,
}

impl Currency {
    /// Faktor pembahagi untuk paparan
    /// USD → 100, JPY → 1, KWD → 1000
    pub const fn factor(&self) -> i64 {
        match self.decimals {
            0 => 1,
            1 => 10,
            2 => 100,
            3 => 1_000,
            4 => 10_000,
            _ => 100, // default
        }
    }
}

// ── Senarai mata wang biasa ───────────────────────────────────

pub const USD: Currency = Currency { code: "USD", symbol: "$",    decimals: 2 };
pub const MYR: Currency = Currency { code: "MYR", symbol: "RM",   decimals: 2 };
pub const EUR: Currency = Currency { code: "EUR", symbol: "€",    decimals: 2 };
pub const GBP: Currency = Currency { code: "GBP", symbol: "£",    decimals: 2 };
pub const SGD: Currency = Currency { code: "SGD", symbol: "S$",   decimals: 2 };
pub const AUD: Currency = Currency { code: "AUD", symbol: "A$",   decimals: 2 };
pub const JPY: Currency = Currency { code: "JPY", symbol: "¥",    decimals: 0 };
pub const KRW: Currency = Currency { code: "KRW", symbol: "₩",    decimals: 0 };
pub const KWD: Currency = Currency { code: "KWD", symbol: "KD",   decimals: 3 };
pub const BHD: Currency = Currency { code: "BHD", symbol: "BD",   decimals: 3 };
pub const OMR: Currency = Currency { code: "OMR", symbol: "OMR",  decimals: 3 };

#[cfg(test)]
mod currency_tests {
    use super::*;

    #[test]
    fn test_factors() {
        assert_eq!(USD.factor(), 100);
        assert_eq!(JPY.factor(), 1);
        assert_eq!(KWD.factor(), 1000);
        assert_eq!(MYR.factor(), 100);
    }
}
```

---

# Bahagian 2: Struct Wang (Money)

```rust
use std::fmt;

/// Wang dalam unit terkecil (minor units)
///
/// # Contoh
/// - `Money { amount: 150, currency: USD }` = $1.50
/// - `Money { amount: 500, currency: JPY }` = ¥500
/// - `Money { amount: 1500, currency: KWD }` = 1.500 KWD
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Money {
    /// Amaun dalam unit terkecil (sen, fils, penny, dll)
    /// Boleh negatif (untuk debit/refund)
    pub amount: i64,
    pub currency: Currency,
}

impl Money {
    // ── Pembina (Constructors) ────────────────────────────────

    /// Buat dari unit terkecil (cara paling selamat)
    /// `Money::from_minor(150, USD)` = $1.50
    pub fn from_minor(amount: i64, currency: Currency) -> Self {
        Money { amount, currency }
    }

    /// Buat dari nilai major dengan decimal
    /// `Money::from_major(1, 50, USD)` = $1.50
    /// `Money::from_major(0, 500, JPY)` = RALAT (JPY tiada decimal)
    pub fn from_major(major: i64, minor: u32, currency: Currency) -> Result<Self, MoneyError> {
        // Semak minor tidak melebihi had decimal mata wang
        let max_minor = currency.factor() as u32;
        if minor >= max_minor {
            return Err(MoneyError::InvalidMinorUnit {
                got: minor,
                max: max_minor - 1,
                currency: currency.code,
            });
        }

        let sign = if major < 0 { -1 } else { 1 };
        let amount = major.abs() * currency.factor() + minor as i64;
        Ok(Money { amount: amount * sign, currency })
    }

    /// Buat dari f64 (HATI-HATI: kemungkinan ralat pembulatan!)
    /// Lebih baik guna `from_minor` atau `from_major`
    pub fn from_f64(value: f64, currency: Currency) -> Self {
        let amount = (value * currency.factor() as f64).round() as i64;
        Money { amount, currency }
    }

    /// Wang sifar
    pub fn zero(currency: Currency) -> Self {
        Money { amount: 0, currency }
    }

    // ── Getter ────────────────────────────────────────────────

    /// Bahagian major (ringgit, dolar, dll)
    /// $1.50 → 1
    pub fn major(&self) -> i64 {
        self.amount / self.currency.factor()
    }

    /// Bahagian minor (sen, fils, penny, dll)
    /// $1.50 → 50
    pub fn minor(&self) -> i64 {
        (self.amount % self.currency.factor()).abs()
    }

    /// Nilai sebagai f64 (untuk display atau kiraan kasar sahaja)
    pub fn as_f64(&self) -> f64 {
        self.amount as f64 / self.currency.factor() as f64
    }

    /// Semak sama ada sifar
    pub fn is_zero(&self) -> bool { self.amount == 0 }

    /// Semak sama ada positif
    pub fn is_positive(&self) -> bool { self.amount > 0 }

    /// Semak sama ada negatif
    pub fn is_negative(&self) -> bool { self.amount < 0 }

    /// Nilai mutlak
    pub fn abs(&self) -> Self {
        Money { amount: self.amount.abs(), currency: self.currency }
    }

    /// Negasi
    pub fn negate(&self) -> Self {
        Money { amount: -self.amount, currency: self.currency }
    }
}

// ── Error Types ────────────────────────────────────────────────

#[derive(Debug, thiserror::Error)]
pub enum MoneyError {
    #[error("Mata wang berbeza: {left} vs {right}")]
    CurrencyMismatch { left: &'static str, right: &'static str },

    #[error("Unit minor {got} melebihi had untuk {currency} (max: {max})")]
    InvalidMinorUnit { got: u32, max: u32, currency: &'static str },

    #[error("Pembahagian dengan sifar")]
    DivisionByZero,

    #[error("Limpahan integer (overflow)")]
    Overflow,
}

#[cfg(test)]
mod money_tests {
    use super::*;

    #[test]
    fn test_from_minor() {
        let wang = Money::from_minor(150, USD);
        assert_eq!(wang.major(), 1);
        assert_eq!(wang.minor(), 50);
        assert_eq!(wang.amount, 150);
    }

    #[test]
    fn test_from_major() {
        let wang = Money::from_major(4, 50, MYR).unwrap();
        assert_eq!(wang.amount, 450);

        // JPY tiada decimal — minor mesti 0
        let yen = Money::from_major(500, 0, JPY).unwrap();
        assert_eq!(yen.amount, 500);

        // KWD 3 decimal
        let kwd = Money::from_major(1, 500, KWD).unwrap();
        assert_eq!(kwd.amount, 1500);
    }

    #[test]
    fn test_from_major_invalid() {
        // Minor 100 tidak sah untuk USD (max 99)
        assert!(Money::from_major(1, 100, USD).is_err());
        // Minor 1 tidak sah untuk JPY (tiada decimal)
        assert!(Money::from_major(500, 1, JPY).is_err());
    }

    #[test]
    fn test_negative() {
        let wang = Money::from_minor(-150, USD);
        assert_eq!(wang.major(), -1);
        assert_eq!(wang.minor(), 50); // minor sentiasa positif
        assert!(wang.is_negative());
    }
}
```

---

# Bahagian 3: Operasi Aritmetik

```rust
use std::ops::{Add, Sub, Mul, Neg};

impl Money {
    /// Tambah dua wang
    pub fn add(&self, other: &Money) -> Result<Money, MoneyError> {
        if self.currency != other.currency {
            return Err(MoneyError::CurrencyMismatch {
                left: self.currency.code,
                right: other.currency.code,
            });
        }
        let amount = self.amount.checked_add(other.amount)
            .ok_or(MoneyError::Overflow)?;
        Ok(Money { amount, currency: self.currency })
    }

    /// Tolak dua wang
    pub fn subtract(&self, other: &Money) -> Result<Money, MoneyError> {
        if self.currency != other.currency {
            return Err(MoneyError::CurrencyMismatch {
                left: self.currency.code,
                right: other.currency.code,
            });
        }
        let amount = self.amount.checked_sub(other.amount)
            .ok_or(MoneyError::Overflow)?;
        Ok(Money { amount, currency: self.currency })
    }

    /// Darab dengan integer
    pub fn multiply_int(&self, factor: i64) -> Result<Money, MoneyError> {
        let amount = self.amount.checked_mul(factor)
            .ok_or(MoneyError::Overflow)?;
        Ok(Money { amount, currency: self.currency })
    }

    /// Darab dengan pecahan (contoh: diskaun 0.15)
    /// Guna kadar dalam integer untuk ketepatan
    /// `multiply_ratio(15, 100)` = darab 15/100 = 15%
    pub fn multiply_ratio(&self, numerator: i64, denominator: i64) -> Result<Money, MoneyError> {
        if denominator == 0 {
            return Err(MoneyError::DivisionByZero);
        }
        // Guna i128 untuk elak overflow semasa kiraan
        let amount = (self.amount as i128 * numerator as i128 / denominator as i128) as i64;
        Ok(Money { amount, currency: self.currency })
    }

    /// Bahagi dengan integer
    pub fn divide_int(&self, divisor: i64) -> Result<Money, MoneyError> {
        if divisor == 0 {
            return Err(MoneyError::DivisionByZero);
        }
        Ok(Money { amount: self.amount / divisor, currency: self.currency })
    }

    /// Bahagi wang kepada beberapa bahagian (untuk split bil)
    /// Balansi akan dimasukkan ke bahagian pertama
    pub fn split(&self, parts: usize) -> Result<Vec<Money>, MoneyError> {
        if parts == 0 {
            return Err(MoneyError::DivisionByZero);
        }
        let base = self.amount / parts as i64;
        let remainder = self.amount % parts as i64;

        let mut result = vec![Money { amount: base, currency: self.currency }; parts];
        // Lebihan ditambah ke bahagian pertama
        result[0].amount += remainder;
        Ok(result)
    }

    /// Kira peratusan (contoh: GST 8%)
    /// `percent(8)` = 8% daripada wang ini
    pub fn percent(&self, pct: i64) -> Result<Money, MoneyError> {
        self.multiply_ratio(pct, 100)
    }
}

// ── Operator Overloading (untuk kemudahan) ─────────────────────
// Nota: operator ini panic kalau mata wang berbeza
// Untuk production, lebih baik guna method di atas yang return Result

impl Add for Money {
    type Output = Money;
    fn add(self, other: Money) -> Money {
        self.add(&other).expect("Mata wang mesti sama")
    }
}

impl Sub for Money {
    type Output = Money;
    fn sub(self, other: Money) -> Money {
        self.subtract(&other).expect("Mata wang mesti sama")
    }
}

impl Neg for Money {
    type Output = Money;
    fn neg(self) -> Money { self.negate() }
}

impl PartialOrd for Money {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        if self.currency != other.currency { return None; }
        Some(self.amount.cmp(&other.amount))
    }
}

#[cfg(test)]
mod arithmetic_tests {
    use super::*;

    #[test]
    fn test_tambah() {
        let a = Money::from_minor(150, USD); // $1.50
        let b = Money::from_minor(250, USD); // $2.50
        let c = a.add(&b).unwrap();
        assert_eq!(c.amount, 400); // $4.00
    }

    #[test]
    fn test_split_bil() {
        let bil = Money::from_minor(1000, MYR); // RM10.00
        let bahagian = bil.split(3).unwrap();
        assert_eq!(bahagian[0].amount, 334); // RM3.34 (ada lebihan 1 sen)
        assert_eq!(bahagian[1].amount, 333); // RM3.33
        assert_eq!(bahagian[2].amount, 333); // RM3.33
        // Jumlah: 334 + 333 + 333 = 1000 ✓
        let jumlah: i64 = bahagian.iter().map(|m| m.amount).sum();
        assert_eq!(jumlah, 1000);
    }

    #[test]
    fn test_gst() {
        let harga = Money::from_minor(10000, MYR); // RM100.00
        let gst = harga.percent(8).unwrap();       // 8%
        assert_eq!(gst.amount, 800);               // RM8.00
    }

    #[test]
    fn test_mata_wang_berbeza() {
        let usd = Money::from_minor(100, USD);
        let myr = Money::from_minor(100, MYR);
        assert!(usd.add(&myr).is_err());
    }
}
```

---

# Bahagian 4: Kadar Tukaran (ExchangeRate)

```rust
/// Kadar tukaran disimpan sebagai integer
/// Kadar × 10^precision → integer
///
/// # Contoh
/// 1 USD = 4.2356 MYR
/// Simpan: 42356 (precision = 4)
/// Guna: 42356 / 10000 = 4.2356
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct ExchangeRate {
    /// Mata wang asas (base) — 1 unit mata wang ini
    pub from: Currency,
    /// Mata wang sasaran (quote)
    pub to: Currency,
    /// Kadar × 10^precision (simpan sebagai integer)
    /// Contoh: 4.2356 → 42356 (precision=4)
    pub rate_int: i64,
    /// Bilangan decimal dalam kadar (biasanya 4 atau 6)
    pub precision: u8,
}

impl ExchangeRate {
    /// Buat kadar dari integer
    /// `ExchangeRate::new(USD, MYR, 42356, 4)` = 1 USD = 4.2356 MYR
    pub fn new(from: Currency, to: Currency, rate_int: i64, precision: u8) -> Self {
        ExchangeRate { from, to, rate_int, precision }
    }

    /// Buat kadar dari f64
    /// `ExchangeRate::from_f64(USD, MYR, 4.2356)` — precision auto 4
    pub fn from_f64(from: Currency, to: Currency, rate: f64, precision: u8) -> Self {
        let factor = 10_i64.pow(precision as u32);
        let rate_int = (rate * factor as f64).round() as i64;
        ExchangeRate { from, to, rate_int, precision }
    }

    /// Faktor pembahagi untuk kadar
    /// precision=4 → 10000
    pub fn rate_factor(&self) -> i64 {
        10_i64.pow(self.precision as u32)
    }

    /// Kadar sebagai f64 (untuk display sahaja)
    pub fn as_f64(&self) -> f64 {
        self.rate_int as f64 / self.rate_factor() as f64
    }

    /// Balikkan kadar (from→to menjadi to→from)
    /// 1 USD = 4.2356 MYR → 1 MYR = 0.2361 USD
    pub fn inverse(&self) -> Self {
        let factor = self.rate_factor();
        // rate_int_baru = (factor^2) / rate_int_lama
        // Guna i128 untuk elak overflow
        let new_rate = (factor as i128 * factor as i128 / self.rate_int as i128) as i64;
        ExchangeRate {
            from:      self.to,
            to:        self.from,
            rate_int:  new_rate,
            precision: self.precision,
        }
    }
}

impl fmt::Display for ExchangeRate {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "1 {} = {:.prec$} {}",
            self.from.code,
            self.as_f64(),
            self.to.code,
            prec = self.precision as usize
        )
    }
}

#[cfg(test)]
mod rate_tests {
    use super::*;

    #[test]
    fn test_buat_kadar() {
        let kadar = ExchangeRate::new(USD, MYR, 42356, 4);
        assert_eq!(kadar.as_f64(), 4.2356);
        assert_eq!(kadar.rate_factor(), 10000);
    }

    #[test]
    fn test_dari_f64() {
        let kadar = ExchangeRate::from_f64(USD, MYR, 4.2356, 4);
        assert_eq!(kadar.rate_int, 42356);
    }

    #[test]
    fn test_inverse() {
        let kadar = ExchangeRate::new(USD, MYR, 42356, 4);
        let balik = kadar.inverse();
        assert_eq!(balik.from, MYR);
        assert_eq!(balik.to, USD);
        // 1/4.2356 ≈ 0.2361
        let dijangka = (10000.0_f64 * 10000.0 / 42356.0).round() as i64;
        assert_eq!(balik.rate_int, dijangka);
    }
}
```

---

# Bahagian 5: Penukaran Mata Wang

```rust
impl Money {
    /// Tukar wang dari satu mata wang ke mata wang lain
    ///
    /// # Cara Kiraan
    /// 1. amount_from (dalam minor units)
    /// 2. × rate_int
    /// 3. × factor_to (decimal target)
    /// 4. ÷ rate_factor (10^precision)
    /// 5. ÷ factor_from (decimal source)
    ///
    /// # Contoh
    /// $1.50 (USD) → MYR pada kadar 4.2356
    /// = 150 sen × 42356 × 100 ÷ 10000 ÷ 100
    /// = 150 × 42356 ÷ 10000
    /// = 6353400 ÷ 10000
    /// = 635 sen MYR
    /// = RM6.35
    pub fn convert(&self, rate: &ExchangeRate) -> Result<Money, MoneyError> {
        if self.currency != rate.from {
            return Err(MoneyError::CurrencyMismatch {
                left: self.currency.code,
                right: rate.from.code,
            });
        }

        // Guna i128 untuk elak overflow semasa kiraan
        // Formula: amount × rate_int × factor_to ÷ (rate_factor × factor_from)
        let numerator = self.amount as i128
            * rate.rate_int as i128
            * rate.to.factor() as i128;

        let denominator = rate.rate_factor() as i128
            * self.currency.factor() as i128;

        let converted = (numerator / denominator) as i64;

        Ok(Money { amount: converted, currency: rate.to })
    }
}

#[cfg(test)]
mod convert_tests {
    use super::*;

    #[test]
    fn test_usd_ke_myr() {
        // $1.50 pada kadar 4.2356
        let wang = Money::from_minor(150, USD);
        let kadar = ExchangeRate::new(USD, MYR, 42356, 4);
        let hasil = wang.convert(&kadar).unwrap();

        // 150 × 42356 × 100 ÷ (10000 × 100) = 635.34 → 635 sen = RM6.35
        assert_eq!(hasil.currency, MYR);
        // Hasil antara 633-637 (bergantung pembulatan)
        assert!(hasil.amount >= 630 && hasil.amount <= 640);
    }

    #[test]
    fn test_usd_ke_jpy() {
        // $10.00 pada kadar 149.85 (USD/JPY)
        let wang = Money::from_minor(1000, USD); // $10.00
        let kadar = ExchangeRate::new(USD, JPY, 1498500, 4); // 149.8500

        // JPY tiada decimal (factor=1), USD decimal=2 (factor=100)
        // 1000 × 1498500 × 1 ÷ (10000 × 100) = 1498.5 → 1498 yen
        let hasil = wang.convert(&kadar).unwrap();
        assert_eq!(hasil.currency, JPY);
        assert_eq!(hasil.amount, 1498); // ¥1498
    }

    #[test]
    fn test_kwd_ke_usd() {
        // 1.000 KWD pada kadar 3.2650 (KWD/USD)
        let wang = Money::from_minor(1000, KWD); // 1.000 KWD
        let kadar = ExchangeRate::new(KWD, USD, 32650, 4); // 3.2650

        // KWD factor=1000, USD factor=100
        // 1000 × 32650 × 100 ÷ (10000 × 1000) = 326.5 → 326 sen = $3.26
        let hasil = wang.convert(&kadar).unwrap();
        assert_eq!(hasil.currency, USD);
        assert_eq!(hasil.amount, 326); // $3.26
    }

    #[test]
    fn test_kadar_salah_mata_wang() {
        let wang = Money::from_minor(100, USD);
        let kadar = ExchangeRate::new(EUR, MYR, 42000, 4); // EUR→MYR, bukan USD→MYR
        assert!(wang.convert(&kadar).is_err());
    }
}
```

---

# Bahagian 6: Pembulatan (Rounding)

```rust
/// Strategi pembulatan
#[derive(Debug, Clone, Copy, PartialEq)]
pub enum RoundingMode {
    /// Bulatkan ke atas kalau tepat setengah (standard)
    HalfUp,
    /// Bulatkan ke bawah kalau tepat setengah
    HalfDown,
    /// Bulatkan ke nombor genap kalau tepat setengah (banker's rounding)
    HalfEven,
    /// Sentiasa bulatkan ke atas
    Ceiling,
    /// Sentiasa bulatkan ke bawah
    Floor,
    /// Buang decimal (truncate)
    Truncate,
}

impl Money {
    /// Bulatkan wang ke bilangan decimal tertentu
    pub fn round_to(&self, decimal_places: u8, mode: RoundingMode) -> Money {
        let target_factor = 10_i64.pow(decimal_places as u32);
        let current_factor = self.currency.factor();

        if target_factor >= current_factor {
            // Tiada perlu bulatkan — sudah lebih kasar
            return *self;
        }

        let ratio = current_factor / target_factor;
        let remainder = self.amount % ratio;
        let base = self.amount - remainder;
        let half = ratio / 2;

        let rounded = match mode {
            RoundingMode::Truncate => base,
            RoundingMode::Floor    => if self.amount < 0 && remainder != 0 { base - ratio } else { base },
            RoundingMode::Ceiling  => if self.amount > 0 && remainder != 0 { base + ratio } else { base },
            RoundingMode::HalfUp   => {
                if remainder.abs() >= half { base + if self.amount >= 0 { ratio } else { -ratio } }
                else { base }
            }
            RoundingMode::HalfDown => {
                if remainder.abs() > half { base + if self.amount >= 0 { ratio } else { -ratio } }
                else { base }
            }
            RoundingMode::HalfEven => {
                let quotient = base / ratio;
                if remainder.abs() == half {
                    if quotient % 2 == 0 { base } else { base + if self.amount >= 0 { ratio } else { -ratio } }
                } else if remainder.abs() > half {
                    base + if self.amount >= 0 { ratio } else { -ratio }
                } else {
                    base
                }
            }
        };

        Money { amount: rounded, currency: self.currency }
    }
}

#[cfg(test)]
mod rounding_tests {
    use super::*;

    #[test]
    fn test_half_up() {
        // RM1.235 → RM1.24 (bulatkan ke atas)
        // amount = 1235 (3 decimal, guna KWD sebagai analog)
        let wang = Money::from_minor(1235, KWD); // 1.235 KWD
        let bulatkan = wang.round_to(2, RoundingMode::HalfUp);
        assert_eq!(bulatkan.amount, 1240); // 1.240
    }

    #[test]
    fn test_half_even_banker() {
        // 1.225 → 1.220 (bulatkan ke genap)
        let wang1 = Money::from_minor(1225, KWD);
        assert_eq!(wang1.round_to(2, RoundingMode::HalfEven).amount, 1220);

        // 1.235 → 1.240 (bulatkan ke genap)
        let wang2 = Money::from_minor(1235, KWD);
        assert_eq!(wang2.round_to(2, RoundingMode::HalfEven).amount, 1240);
    }
}
```

---

# Bahagian 7: Format Paparan

```rust
impl fmt::Display for Money {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        let factor = self.currency.factor();
        let major = self.amount / factor;
        let minor = (self.amount % factor).abs();

        if self.currency.decimals == 0 {
            write!(f, "{}{}", self.currency.symbol, major)
        } else {
            write!(f, "{}{}.{:0>width$}",
                self.currency.symbol,
                major,
                minor,
                width = self.currency.decimals as usize
            )
        }
    }
}

impl Money {
    /// Format dengan pemisah ribuan
    pub fn format_with_thousands(&self) -> String {
        let factor = self.currency.factor();
        let major = (self.amount / factor).abs();
        let minor = (self.amount % factor).abs();
        let negatif = if self.amount < 0 { "-" } else { "" };

        // Tambah pemisah koma setiap 3 digit
        let major_str = major.to_string();
        let dengan_koma: String = major_str.chars().rev()
            .enumerate()
            .flat_map(|(i, c)| {
                if i > 0 && i % 3 == 0 { vec![',', c] } else { vec![c] }
            })
            .collect::<String>()
            .chars()
            .rev()
            .collect();

        if self.currency.decimals == 0 {
            format!("{}{}{}", negatif, self.currency.symbol, dengan_koma)
        } else {
            format!("{}{}{}.{:0>width$}",
                negatif,
                self.currency.symbol,
                dengan_koma,
                minor,
                width = self.currency.decimals as usize
            )
        }
    }

    /// Format untuk JSON/API (hanya angka, tanpa simbol)
    pub fn to_decimal_string(&self) -> String {
        let factor = self.currency.factor();
        let major = self.amount / factor;
        let minor = (self.amount % factor).abs();

        if self.currency.decimals == 0 {
            major.to_string()
        } else {
            format!("{}.{:0>width$}", major, minor, width = self.currency.decimals as usize)
        }
    }
}

#[cfg(test)]
mod format_tests {
    use super::*;

    #[test]
    fn test_display() {
        assert_eq!(Money::from_minor(150, USD).to_string(),   "$1.50");
        assert_eq!(Money::from_minor(450, MYR).to_string(),   "RM4.50");
        assert_eq!(Money::from_minor(500, JPY).to_string(),   "¥500");
        assert_eq!(Money::from_minor(1500, KWD).to_string(), "KD1.500");
        assert_eq!(Money::from_minor(-299, EUR).to_string(), "€-2.99");
    }

    #[test]
    fn test_format_thousands() {
        let wang = Money::from_minor(1234567, MYR); // RM12,345.67
        assert_eq!(wang.format_with_thousands(), "RM12,345.67");

        let besar = Money::from_minor(100000000, JPY); // ¥1,000,000
        assert_eq!(besar.format_with_thousands(), "¥1,000,000");
    }

    #[test]
    fn test_decimal_string() {
        assert_eq!(Money::from_minor(150, USD).to_decimal_string(),  "1.50");
        assert_eq!(Money::from_minor(500, JPY).to_decimal_string(),  "500");
        assert_eq!(Money::from_minor(1500, KWD).to_decimal_string(), "1.500");
    }
}
```

---

# Bahagian 8: Serde (JSON/API)

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
use serde::{Serialize, Deserialize, Serializer, Deserializer};
use serde::de::{self, Visitor};

/// Representasi JSON untuk Money
/// Simpan sebagai: { "amount": 150, "currency": "USD" }
/// ATAU: { "amount": "1.50", "currency": "USD" } (string decimal)
#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct MoneyJson {
    /// Amaun dalam minor units (integer)
    pub amount: i64,
    /// Kod mata wang ISO 4217
    pub currency: String,
}

impl From<Money> for MoneyJson {
    fn from(m: Money) -> Self {
        MoneyJson {
            amount:   m.amount,
            currency: m.currency.code.to_string(),
        }
    }
}

/// Untuk API yang hantar decimal string (lebih selamat dari float)
#[derive(Debug, Serialize, Deserialize)]
pub struct MoneyDecimalJson {
    /// Nilai dalam decimal string: "1.50", "500", "1.500"
    pub amount: String,
    /// Kod mata wang
    pub currency: String,
}

impl From<Money> for MoneyDecimalJson {
    fn from(m: Money) -> Self {
        MoneyDecimalJson {
            amount:   m.to_decimal_string(),
            currency: m.currency.code.to_string(),
        }
    }
}

/// Contoh struct untuk respons API harga
#[derive(Debug, Serialize, Deserialize)]
pub struct HargaApi {
    pub produk_id:  u32,
    pub nama:       String,
    #[serde(with = "money_serde")]
    pub harga:      Money,
    #[serde(with = "money_serde")]
    pub diskaun:    Money,
}

/// Module serde untuk Money ↔ { amount, currency }
pub mod money_serde {
    use super::*;

    pub fn serialize<S: Serializer>(money: &Money, s: S) -> Result<S::Ok, S::Error> {
        MoneyJson::from(*money).serialize(s)
    }

    pub fn deserialize<'de, D: Deserializer<'de>>(d: D) -> Result<Money, D::Error> {
        let json = MoneyJson::deserialize(d)?;
        let currency = match json.currency.as_str() {
            "USD" => USD, "MYR" => MYR, "EUR" => EUR, "GBP" => GBP,
            "JPY" => JPY, "KWD" => KWD, "SGD" => SGD, "AUD" => AUD,
            other => return Err(de::Error::custom(
                format!("Mata wang tidak dikenali: {}", other)
            )),
        };
        Ok(Money::from_minor(json.amount, currency))
    }
}

#[cfg(test)]
mod serde_tests {
    use super::*;

    #[test]
    fn test_serialize_money() {
        let wang = Money::from_minor(450, MYR);
        let json_val = MoneyJson::from(wang);
        let json = serde_json::to_string(&json_val).unwrap();
        assert_eq!(json, r#"{"amount":450,"currency":"MYR"}"#);
    }

    #[test]
    fn test_decimal_json() {
        let wang = Money::from_minor(1500, KWD);
        let json_val = MoneyDecimalJson::from(wang);
        let json = serde_json::to_string(&json_val).unwrap();
        assert_eq!(json, r#"{"amount":"1.500","currency":"KWD"}"#);
    }
}
```

---

# Bahagian 9: Kes-kes Tepi (Edge Cases)

```rust
/// Dokumentasi kes tepi yang perlu diambil kira

// ── 1. Overflow ───────────────────────────────────────────────
// i64 max = 9,223,372,036,854,775,807
// USD sen: boleh simpan sehingga $92,233,720,368,547,758.07
// Untuk GDP negara pun cukup! Biasanya selamat.
// Tapi semasa penukaran mata wang, guna i128:
let a: i128 = 150_i128 * 42356_i128 * 100_i128; // OK dalam i128
let b = (a / (10000 * 100)) as i64;              // convert balik ke i64

// ── 2. Negatif ────────────────────────────────────────────────
// Nilai negatif = debit / refund / hutang
let refund = Money::from_minor(-5000, MYR); // -RM50.00
println!("{}", refund); // RM-50.00
// major() return -50, minor() return 0 (abs)

// ── 3. JPY dalam penukaran ────────────────────────────────────
// JPY factor = 1 (tiada decimal)
// ¥100 → USD pada kadar 0.0067
// Kadar: 0.0067 × 10000 = 67 (rate_int)
let yen = Money::from_minor(100, JPY);
let kadar = ExchangeRate::new(JPY, USD, 67, 4); // 0.0067
// 100 × 67 × 100 ÷ (10000 × 1) = 0.67 sen = $0.00
// Untuk kecilnya ¥1 = $0.0067, 100 yen = $0.67
// Perhatikan: pembulatan ke bawah! Guna RoundingMode yang sesuai.

// ── 4. Kadar bertingkat (triangular arbitrage) ────────────────
// USD → EUR → JPY (bukan terus USD → JPY)
// Setiap penukaran ada ralat pembulatan kecil
// Simpan kadar dengan precision lebih tinggi (6 decimal) untuk reduce error

// ── 5. String input dari pengguna ─────────────────────────────
/// Parse string decimal ke Money
pub fn parse_money(input: &str, currency: Currency) -> Result<Money, String> {
    let bersih = input.trim().replace(',', ""); // buang koma pemisah

    if let Some(titik) = bersih.find('.') {
        let bahagian_major = &bersih[..titik];
        let bahagian_minor = &bersih[titik + 1..];

        // Pastikan panjang decimal betul
        if bahagian_minor.len() > currency.decimals as usize {
            return Err(format!(
                "Terlalu banyak decimal untuk {} (max {})",
                currency.code, currency.decimals
            ));
        }

        let major: i64 = bahagian_major.parse()
            .map_err(|e| format!("Nilai tidak sah: {}", e))?;
        let minor_str = format!("{:0<width$}", bahagian_minor,
                                width = currency.decimals as usize);
        let minor: i64 = minor_str.parse()
            .map_err(|e| format!("Decimal tidak sah: {}", e))?;

        let sign = if major < 0 { -1 } else { 1 };
        let amount = major.abs() * currency.factor() + minor;
        Ok(Money::from_minor(amount * sign, currency))
    } else {
        // Tiada titik decimal
        let major: i64 = bersih.parse()
            .map_err(|e| format!("Nilai tidak sah: {}", e))?;
        Ok(Money::from_minor(major * currency.factor(), currency))
    }
}

#[cfg(test)]
mod edge_case_tests {
    use super::*;

    #[test]
    fn test_parse_money() {
        assert_eq!(parse_money("1.50", USD).unwrap().amount,  150);
        assert_eq!(parse_money("500", JPY).unwrap().amount,   500);
        assert_eq!(parse_money("1.500", KWD).unwrap().amount, 1500);
        assert_eq!(parse_money("1,234.56", USD).unwrap().amount, 123456);
        assert!(parse_money("1.234", USD).is_err()); // USD max 2 decimal
    }

    #[test]
    fn test_sifar() {
        let sifar = Money::zero(USD);
        assert!(sifar.is_zero());
        assert!(!sifar.is_positive());
        assert!(!sifar.is_negative());
    }
}
```

---

# Bahagian 10: Contoh Lengkap — Sistem Invois

```rust
use std::collections::HashMap;

/// Item dalam invois
#[derive(Debug, Clone)]
pub struct ItemInvois {
    pub nama:     String,
    pub kuantiti: u32,
    pub harga:    Money, // harga seunit
}

impl ItemInvois {
    pub fn jumlah(&self) -> Result<Money, MoneyError> {
        self.harga.multiply_int(self.kuantiti as i64)
    }
}

/// Invois lengkap
#[derive(Debug)]
pub struct Invois {
    pub no:        String,
    pub mata_wang: Currency,
    pub item:      Vec<ItemInvois>,
    pub diskaun:   Option<i64>, // peratusan (contoh: 10 = 10%)
    pub cukai:     Option<i64>, // peratusan (contoh: 8 = 8% GST)
}

#[derive(Debug)]
pub struct RingkasanInvois {
    pub subtotal:     Money,
    pub diskaun:      Money,
    pub cukai:        Money,
    pub jumlah_bayar: Money,
}

impl Invois {
    pub fn kira(&self) -> Result<RingkasanInvois, MoneyError> {
        // 1. Kira subtotal
        let mut subtotal = Money::zero(self.mata_wang);
        for item in &self.item {
            let jumlah = item.jumlah()?;
            subtotal = subtotal.add(&jumlah)?;
        }

        // 2. Kira diskaun
        let diskaun = match self.diskaun {
            Some(pct) => subtotal.percent(pct)?,
            None      => Money::zero(self.mata_wang),
        };

        // 3. Harga selepas diskaun
        let selepas_diskaun = subtotal.subtract(&diskaun)?;

        // 4. Kira cukai (GST/SST)
        let cukai = match self.cukai {
            Some(pct) => selepas_diskaun.percent(pct)?,
            None      => Money::zero(self.mata_wang),
        };

        // 5. Jumlah akhir
        let jumlah_bayar = selepas_diskaun.add(&cukai)?;

        Ok(RingkasanInvois { subtotal, diskaun, cukai, jumlah_bayar })
    }

    pub fn cetak(&self) {
        println!("╔══════════════════════════════════════╗");
        println!("║  INVOIS NO: {:<26} ║", self.no);
        println!("╠══════════════════════════════════════╣");
        println!("║  {:<20} {:>6}  {:>8} ║", "Perkara", "Kuantiti", "Jumlah");
        println!("╠══════════════════════════════════════╣");

        for item in &self.item {
            if let Ok(jumlah) = item.jumlah() {
                println!("║  {:<20} {:>6}  {:>8} ║",
                    item.nama, item.kuantiti,
                    jumlah.format_with_thousands());
            }
        }

        if let Ok(ringkasan) = self.kira() {
            println!("╠══════════════════════════════════════╣");
            println!("║  {:<28} {:>8} ║", "Subtotal:",
                ringkasan.subtotal.format_with_thousands());
            if ringkasan.diskaun.amount > 0 {
                println!("║  {:<28} {:>8} ║",
                    format!("Diskaun ({}%):", self.diskaun.unwrap_or(0)),
                    format!("-{}", ringkasan.diskaun.format_with_thousands()));
            }
            if ringkasan.cukai.amount > 0 {
                println!("║  {:<28} {:>8} ║",
                    format!("Cukai ({}%):", self.cukai.unwrap_or(0)),
                    ringkasan.cukai.format_with_thousands());
            }
            println!("╠══════════════════════════════════════╣");
            println!("║  {:<28} {:>8} ║", "JUMLAH BAYAR:",
                ringkasan.jumlah_bayar.format_with_thousands());
            println!("╚══════════════════════════════════════╝");
        }
    }
}

fn main() {
    // ── Demo 1: Invois MYR ────────────────────────────────────
    let invois = Invois {
        no:        "INV-2024-001".into(),
        mata_wang: MYR,
        item:      vec![
            ItemInvois {
                nama:     "Laptop Dell".into(),
                kuantiti: 2,
                harga:    Money::from_minor(499900, MYR), // RM4,999.00
            },
            ItemInvois {
                nama:     "Mouse Logitech".into(),
                kuantiti: 5,
                harga:    Money::from_minor(8900, MYR), // RM89.00
            },
        ],
        diskaun:   Some(10), // 10%
        cukai:     Some(8),  // 8% SST
    };

    invois.cetak();

    // ── Demo 2: Penukaran Mata Wang ───────────────────────────
    println!("\n=== Penukaran Mata Wang ===\n");

    let wang_usd = Money::from_minor(10000, USD); // $100.00
    let kadar_usd_myr = ExchangeRate::from_f64(USD, MYR, 4.2356, 4);
    let kadar_usd_jpy = ExchangeRate::from_f64(USD, JPY, 149.85, 4);
    let kadar_usd_kwd = ExchangeRate::from_f64(USD, KWD, 0.3082, 4);

    println!("Wang asal: {}", wang_usd.format_with_thousands());
    println!("{}", kadar_usd_myr);
    println!("{}", kadar_usd_jpy);
    println!("{}", kadar_usd_kwd);
    println!();

    if let Ok(myr) = wang_usd.convert(&kadar_usd_myr) {
        println!("{} = {}", wang_usd, myr.format_with_thousands());
    }
    if let Ok(jpy) = wang_usd.convert(&kadar_usd_jpy) {
        println!("{} = {}", wang_usd, jpy.format_with_thousands());
    }
    if let Ok(kwd) = wang_usd.convert(&kadar_usd_kwd) {
        println!("{} = {}", wang_usd, kwd.format_with_thousands());
    }

    // ── Demo 3: Split bil ─────────────────────────────────────
    println!("\n=== Split Bil ===\n");

    let bil = Money::from_minor(10000, MYR); // RM100.00
    let bahagian = bil.split(3).unwrap();

    println!("Bil {} dibahagi 3:", bil);
    for (i, b) in bahagian.iter().enumerate() {
        println!("  Orang {}: {}", i + 1, b);
    }
}

// Output dijangka:
// ╔══════════════════════════════════════╗
// ║  INVOIS NO: INV-2024-001              ║
// ╠══════════════════════════════════════╣
// ║  Perkara            Kuantiti   Jumlah ║
// ╠══════════════════════════════════════╣
// ║  Laptop Dell             2  RM9,998.00 ║
// ║  Mouse Logitech          5    RM445.00 ║
// ╠══════════════════════════════════════╣
// ║  Subtotal:                 RM10,443.00 ║
// ║  Diskaun (10%):           -RM1,044.30 ║
// ║  Cukai (8%):                 RM751.89 ║
// ╠══════════════════════════════════════╣
// ║  JUMLAH BAYAR:             RM10,150.59 ║
// ╚══════════════════════════════════════╝
```

---

## Ringkasan Konsep Utama

```
WANG (Money):
  Simpan: amount: i64  (unit terkecil = sen/fils/penny)
  $1.50   → amount = 150,  currency = USD (decimals=2)
  ¥500    → amount = 500,  currency = JPY (decimals=0)
  1.500KD → amount = 1500, currency = KWD (decimals=3)

KADAR TUKARAN (ExchangeRate):
  Simpan: rate_int: i64  (kadar × 10^precision)
  4.2356 MYR/USD → rate_int = 42356, precision = 4
  Guna:   rate_int / 10^precision = kadar sebenar

PENUKARAN:
  amount_result = amount_from × rate_int × factor_to
                  ÷ (rate_factor × factor_from)
  Guna i128 semasa kiraan untuk elak overflow!

PEMBULATAN:
  Guna HalfUp untuk standard
  Guna HalfEven (Banker's) untuk kewangan serius
  Sentiasa bulatkan SELEPAS kiraan, bukan semasa

JANGAN:
  ✗ Simpan wang sebagai f64/f32
  ✗ Bahagi sebelum darab (hilang precision)
  ✗ Campurkan mata wang tanpa semak
  ✗ Lupa handle overflow untuk nilai besar
```

---

*Simpan integer, papar decimal.
Ketepatan kewangan bergantung pada disiplin ini.* 🦀
