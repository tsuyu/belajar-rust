# 📅 Date, Time & Timezone dalam Rust — Panduan Lengkap

> Semua tentang masa: parse, format, timezone,
> arithmetic, duration, dan real-world patterns.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Setup

```toml
[dependencies]
chrono      = { version = "0.4", features = ["serde"] }
chrono-tz   = "0.9"   # timezone database lengkap
serde       = { version = "1", features = ["derive"] }
```

---

## Peta Pembelajaran

```
Bahagian 1  → Konsep Asas: Naive vs Aware
Bahagian 2  → DateTime Sekarang
Bahagian 3  → Parse dari String
Bahagian 4  → Format ke String
Bahagian 5  → Timezone & Conversion
Bahagian 6  → Date/Time Arithmetic
Bahagian 7  → Duration & Comparison
Bahagian 8  → Serde Integration
Bahagian 9  → Real-world Patterns
Bahagian 10 → Mini Project: Sistem Jadual KADA
```

---

# BAHAGIAN 1: Konsep Asas — Naive vs Aware 🗺️

## Jenis DateTime dalam Chrono

```rust
use chrono::{
    NaiveDate,        // Tarikh TANPA timezone
    NaiveTime,        // Masa TANPA timezone
    NaiveDateTime,    // Tarikh + masa TANPA timezone
    DateTime,         // Tarikh + masa DENGAN timezone
    Utc,              // UTC timezone
    Local,            // Timezone sistem tempatan
    FixedOffset,      // Timezone offset tetap (+08:00)
};

fn main() {
    // ── Naive — tiada timezone context ───────────────────────
    // Guna bila: simpan dalam DB, logic yang tidak peduli TZ
    let tarikh = chrono::NaiveDate::from_ymd_opt(2024, 1, 15).unwrap();
    let masa   = chrono::NaiveTime::from_hms_opt(8, 30, 0).unwrap();
    let dt     = chrono::NaiveDateTime::new(tarikh, masa);
    println!("{}", dt); // 2024-01-15 08:30:00

    // ── DateTime<Utc> — UTC dengan timezone ──────────────────
    // Guna bila: simpan dalam sistem, hantar melalui API, bandingkan
    let utc_now = Utc::now();
    println!("{}", utc_now); // 2024-01-15T00:30:00.123456Z

    // ── DateTime<Local> — timezone sistem ────────────────────
    // Guna bila: papar kepada pengguna tempatan
    let local_now = Local::now();
    println!("{}", local_now); // 2024-01-15T08:30:00+08:00

    // ── DateTime<FixedOffset> — offset tetap ─────────────────
    // Guna bila: simpan/hantar dengan timezone yang spesifik
    let myt = chrono::FixedOffset::east_opt(8 * 3600).unwrap(); // +08:00
    let myt_now = Utc::now().with_timezone(&myt);
    println!("{}", myt_now); // 2024-01-15T08:30:00+08:00
}
```

---

## 🧠 Brain Teaser #1

Apakah perbezaan antara `NaiveDateTime` dan `DateTime<Utc>`?

```rust
use chrono::{NaiveDateTime, DateTime, Utc};

let naive = NaiveDateTime::parse_from_str("2024-01-15 08:30:00", "%Y-%m-%d %H:%M:%S").unwrap();
let aware: DateTime<Utc> = Utc::now();

// Mana satu lebih "tepat"?
// Bila guna mana satu?
```

<details>
<summary>👀 Jawapan</summary>

```
NaiveDateTime:
  - Hanya menyimpan: tahun, bulan, hari, jam, minit, saat
  - TIDAK tahu timezone
  - "2024-01-15 08:30:00" — 8:30 di mana? Tidak tahu!
  - Selamat untuk: tarikh lahir, tarikh cuti, jadual tetap
  - Guna dalam DB sebagai tarikh "logik" (bukan masa tepat)
  - Perbandingan antara NaiveDateTime berbeza TZ = SALAH!

DateTime<Utc>:
  - Menyimpan: masa TEPAT sejak epoch
  - TAHU ia adalah UTC
  - "2024-01-15T00:30:00Z" — ini adalah satu titik masa yang tepat
  - Selamat untuk: audit log, timestamp API, bandingkan dua masa
  - Boleh convert ke mana-mana TZ dengan tepat

Analogi:
  NaiveDateTime = "Mesyuarat jam 3 petang" (petang mana?)
  DateTime<Utc> = "Mesyuarat pada Unix timestamp 1705294200" (tepat!)

Panduan:
  ✔ DB column untuk "tarikh berkaitan" (cuti, lahir) → NaiveDate/NaiveDateTime
  ✔ DB column untuk "bila berlaku" (log, transaksi) → DateTime<Utc> atau i64 (timestamp)
  ✔ Display kepada pengguna → tukar ke Local atau TZ mereka
  ✗ JANGAN simpan DateTime<Local> dalam DB — timezone sistem boleh berubah!
```
</details>

---

# BAHAGIAN 2: DateTime Sekarang ⏰

## Cara Dapatkan Masa Semasa

```rust
use chrono::{Utc, Local, FixedOffset, Datelike, Timelike};

fn main() {
    // ── UTC sekarang ──────────────────────────────────────────
    let sekarang_utc = Utc::now();
    println!("UTC:   {}", sekarang_utc);

    // ── Local (sistem) sekarang ───────────────────────────────
    let sekarang_local = Local::now();
    println!("Local: {}", sekarang_local);

    // ── Malaysia Time (+08:00) ────────────────────────────────
    let myt = FixedOffset::east_opt(8 * 3600).unwrap();
    let sekarang_myt = Utc::now().with_timezone(&myt);
    println!("MYT:   {}", sekarang_myt);

    // ── Dapatkan komponen ─────────────────────────────────────
    let dt = Utc::now();

    println!("Tahun:  {}", dt.year());
    println!("Bulan:  {}", dt.month());    // 1-12
    println!("Hari:   {}", dt.day());      // 1-31
    println!("Jam:    {}", dt.hour());     // 0-23
    println!("Minit:  {}", dt.minute());   // 0-59
    println!("Saat:   {}", dt.second());   // 0-59
    println!("Nano:   {}", dt.nanosecond());
    println!("Weekday: {:?}", dt.weekday()); // Mon, Tue, dll
    println!("Ordinal: {}", dt.ordinal());   // hari ke-N dalam tahun

    // ── Unix Timestamp ────────────────────────────────────────
    let ts = dt.timestamp();                // saat sejak epoch
    let ts_ms = dt.timestamp_millis();     // milisaat
    let ts_us = dt.timestamp_micros();     // mikrosaat
    let ts_ns = dt.timestamp_nanos_opt();  // nanosaat (Option!)

    println!("Unix:  {}", ts);
    println!("Milli: {}", ts_ms);

    // ── Dari Unix Timestamp ───────────────────────────────────
    use chrono::TimeZone;
    let dari_ts = Utc.timestamp_opt(ts, 0).unwrap();
    println!("Dari ts: {}", dari_ts);

    // ── Hari dalam minggu ─────────────────────────────────────
    use chrono::Weekday;
    let hari = dt.weekday();
    let adalah_hujung_minggu = matches!(hari, Weekday::Sat | Weekday::Sun);
    println!("Hujung minggu: {}", adalah_hujung_minggu);
    println!("Nombor hari (0=Mon): {}", hari.num_days_from_monday());
}
```

---

# BAHAGIAN 3: Parse dari String 🔍

## Format Parse Biasa

```rust
use chrono::{DateTime, NaiveDate, NaiveDateTime, NaiveTime, Utc, TimeZone};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // ── Parse NaiveDate ───────────────────────────────────────
    let tarikh = NaiveDate::parse_from_str("2024-01-15", "%Y-%m-%d")?;
    println!("{}", tarikh); // 2024-01-15

    let tarikh2 = NaiveDate::parse_from_str("15/01/2024", "%d/%m/%Y")?;
    println!("{}", tarikh2); // 2024-01-15

    let tarikh3 = NaiveDate::parse_from_str("15 Januari 2024", "%d %B %Y")?;
    // %B = nama bulan penuh (English)

    // ── Parse NaiveDateTime ───────────────────────────────────
    let dt = NaiveDateTime::parse_from_str(
        "2024-01-15 08:30:00",
        "%Y-%m-%d %H:%M:%S"
    )?;
    println!("{}", dt);

    let dt2 = NaiveDateTime::parse_from_str(
        "15-Jan-2024 08:30:00",
        "%d-%b-%Y %H:%M:%S"
    )?;

    // ── Parse DateTime dengan timezone (RFC3339/ISO 8601) ─────
    let rfc = DateTime::parse_from_rfc3339("2024-01-15T08:30:00+08:00")?;
    println!("{}", rfc);

    let rfc_utc = DateTime::parse_from_rfc3339("2024-01-15T00:30:00Z")?;
    println!("{}", rfc_utc);

    // ── Parse RFC2822 (email format) ──────────────────────────
    let email_dt = DateTime::parse_from_rfc2822("Mon, 15 Jan 2024 08:30:00 +0800")?;
    println!("{}", email_dt);

    // ── Parse dengan multiple format (cuba satu persatu) ──────
    fn parse_tarikh_flexible(s: &str) -> Option<NaiveDate> {
        let format_list = [
            "%Y-%m-%d",          // 2024-01-15
            "%d/%m/%Y",          // 15/01/2024
            "%d-%m-%Y",          // 15-01-2024
            "%d %B %Y",          // 15 January 2024
            "%d %b %Y",          // 15 Jan 2024
            "%Y%m%d",            // 20240115
        ];

        for fmt in &format_list {
            if let Ok(d) = NaiveDate::parse_from_str(s, fmt) {
                return Some(d);
            }
        }
        None
    }

    println!("{:?}", parse_tarikh_flexible("2024-01-15"));    // Some
    println!("{:?}", parse_tarikh_flexible("15/01/2024"));    // Some
    println!("{:?}", parse_tarikh_flexible("15 January 2024")); // Some
    println!("{:?}", parse_tarikh_flexible("bukan tarikh"));  // None

    Ok(())
}
```

---

## Format Codes Rujukan

```
Tahun:
  %Y = tahun 4 digit (2024)
  %y = tahun 2 digit (24)

Bulan:
  %m = bulan 2 digit (01-12)
  %B = nama penuh (January)
  %b = nama pendek (Jan)

Hari:
  %d = hari 2 digit (01-31)
  %e = hari (1-31, tiada leading zero)
  %j = ordinal day (001-366)

Hari Minggu:
  %A = nama penuh (Monday)
  %a = nama pendek (Mon)
  %u = 1=Monday, 7=Sunday
  %w = 0=Sunday, 6=Saturday

Masa:
  %H = jam 24h (00-23)
  %I = jam 12h (01-12)
  %M = minit (00-59)
  %S = saat (00-59)
  %f = nanosaat
  %p = AM/PM
  %P = am/pm

Timezone:
  %z = +0800
  %:z = +08:00
  %Z = nama TZ (MYT)

Kombinasi:
  %T = %H:%M:%S (08:30:00)
  %D = %m/%d/%y (01/15/24)
  %F = %Y-%m-%d (2024-01-15)
  %R = %H:%M (08:30)
  %c = full (Mon Jan 15 08:30:00 2024)
```

---

# BAHAGIAN 4: Format ke String 📝

## Output yang Berbeza

```rust
use chrono::{Utc, Local, DateTime, FixedOffset, NaiveDate};

fn main() {
    let dt = Utc::now();

    // ── Format asas ───────────────────────────────────────────
    println!("{}", dt.format("%Y-%m-%d"));           // 2024-01-15
    println!("{}", dt.format("%d/%m/%Y"));           // 15/01/2024
    println!("{}", dt.format("%H:%M:%S"));           // 08:30:00
    println!("{}", dt.format("%d %B %Y, %I:%M %p")); // 15 January 2024, 08:30 AM
    println!("{}", dt.format("%Y-%m-%dT%H:%M:%S%.3fZ")); // ISO 8601

    // ── Format standard ───────────────────────────────────────
    println!("{}", dt.to_rfc3339());         // 2024-01-15T00:30:00.123456Z
    println!("{}", dt.to_rfc2822());         // Mon, 15 Jan 2024 00:30:00 +0000

    // ── Format Bahasa Melayu ──────────────────────────────────
    fn format_tarikh_my(dt: &chrono::DateTime<Utc>) -> String {
        let hari = match dt.weekday() {
            chrono::Weekday::Mon => "Isnin",
            chrono::Weekday::Tue => "Selasa",
            chrono::Weekday::Wed => "Rabu",
            chrono::Weekday::Thu => "Khamis",
            chrono::Weekday::Fri => "Jumaat",
            chrono::Weekday::Sat => "Sabtu",
            chrono::Weekday::Sun => "Ahad",
        };

        let bulan = match dt.month() {
            1  => "Januari",     2  => "Februari",  3  => "Mac",
            4  => "April",       5  => "Mei",        6  => "Jun",
            7  => "Julai",       8  => "Ogos",       9  => "September",
            10 => "Oktober",     11 => "November",   12 => "Disember",
            _  => "?",
        };

        format!("{}, {} {} {}", hari, dt.day(), bulan, dt.year())
    }

    // Convert ke MYT dulu
    let myt = FixedOffset::east_opt(8 * 3600).unwrap();
    let myt_dt = dt.with_timezone(&myt);
    println!("{}", format_tarikh_my(&dt)); // Isnin, 15 Januari 2024

    // ── Format untuk display vs storage ──────────────────────
    let utc_stored = dt.to_rfc3339();          // simpan dalam DB/API
    let display_myt = myt_dt.format("%d/%m/%Y %H:%M");  // papar kepada pengguna
    println!("Simpan:  {}", utc_stored);
    println!("Papar:   {}", display_myt);

    // ── Masa relatif (sederhana) ──────────────────────────────
    fn masa_lalu(ts: i64) -> String {
        let saat = Utc::now().timestamp() - ts;
        match saat {
            0..=59         => format!("{} saat lepas", saat),
            60..=3599      => format!("{} minit lepas", saat / 60),
            3600..=86399   => format!("{} jam lepas", saat / 3600),
            86400..=604799 => format!("{} hari lepas", saat / 86400),
            _              => format!("{} minggu lepas", saat / 604800),
        }
    }

    let ts_lama = Utc::now().timestamp() - 7200; // 2 jam lepas
    println!("{}", masa_lalu(ts_lama)); // 2 jam lepas
}
```

---

# BAHAGIAN 5: Timezone & Conversion 🌏

## Timezone Lengkap dengan chrono-tz

```toml
[dependencies]
chrono    = "0.4"
chrono-tz = "0.9"
```

```rust
use chrono::{Utc, TimeZone};
use chrono_tz::Asia::Kuala_Lumpur;
use chrono_tz::Tz;

fn main() {
    // ── Guna chrono-tz ────────────────────────────────────────
    let utc_now = Utc::now();

    // Convert ke Malaysia Time
    let myt = utc_now.with_timezone(&Kuala_Lumpur);
    println!("MYT:  {}", myt);  // 2024-01-15 08:30:00 MYT

    // Pelbagai timezone Asia
    use chrono_tz::Asia;
    let tokyo      = utc_now.with_timezone(&Asia::Tokyo);       // +09:00
    let singapore  = utc_now.with_timezone(&Asia::Singapore);   // +08:00
    let jakarta    = utc_now.with_timezone(&Asia::Jakarta);     // +07:00
    let kolkata    = utc_now.with_timezone(&Asia::Kolkata);     // +05:30

    println!("Tokyo:     {}", tokyo.format("%H:%M %Z"));
    println!("Singapore: {}", singapore.format("%H:%M %Z"));
    println!("Jakarta:   {}", jakarta.format("%H:%M %Z"));
    println!("Kolkata:   {}", kolkata.format("%H:%M %Z"));

    // ── Buat DateTime dalam timezone tertentu ─────────────────
    // "Mesyuarat jam 9 pagi MYT pada 2024-01-15"
    let mesyuarat_myt = Kuala_Lumpur
        .with_ymd_and_hms(2024, 1, 15, 9, 0, 0)
        .unwrap();

    // Sama masa dalam UTC
    let mesyuarat_utc = mesyuarat_myt.with_timezone(&Utc);
    println!("Mesyuarat MYT: {}", mesyuarat_myt);  // 09:00 MYT
    println!("Mesyuarat UTC: {}", mesyuarat_utc);  // 01:00 UTC

    // ── Parse timezone dari string ────────────────────────────
    let tz: Tz = "Asia/Kuala_Lumpur".parse().unwrap();
    let dt = tz.with_ymd_and_hms(2024, 1, 15, 8, 30, 0).unwrap();
    println!("{}", dt);

    // ── FixedOffset untuk timezone tetap ─────────────────────
    use chrono::FixedOffset;
    let plus8 = FixedOffset::east_opt(8 * 3600).unwrap();  // +08:00
    let minus5 = FixedOffset::west_opt(5 * 3600).unwrap(); // -05:00

    let dt_plus8 = utc_now.with_timezone(&plus8);
    println!("+08:00: {}", dt_plus8);

    // ── Daylight Saving Time (DST) ────────────────────────────
    // Malaysia TIDAK ada DST — selalu +08:00
    // Tapi Amerika ada!
    use chrono_tz::America::New_York;
    let ny_winter = New_York.with_ymd_and_hms(2024, 1, 15, 12, 0, 0).unwrap();
    let ny_summer = New_York.with_ymd_and_hms(2024, 7, 15, 12, 0, 0).unwrap();
    println!("NY Jan: {} ({})", ny_winter.format("%H:%M"), ny_winter.format("%Z")); // EST
    println!("NY Jul: {} ({})", ny_summer.format("%H:%M"), ny_summer.format("%Z")); // EDT
    // chrono-tz auto-handle DST!
}
```

---

## 🧠 Brain Teaser #2

Apakah output? Kenapa berbeza?

```rust
use chrono::{Utc, TimeZone};
use chrono_tz::Asia::Kuala_Lumpur;

fn main() {
    let utc = Utc.with_ymd_and_hms(2024, 1, 15, 0, 0, 0).unwrap();

    let myt = utc.with_timezone(&Kuala_Lumpur);

    println!("UTC: {:?}", utc.date_naive());
    println!("MYT: {:?}", myt.date_naive());
}
```

<details>
<summary>👀 Jawapan</summary>

```
UTC: 2024-01-15
MYT: 2024-01-15

TUNGGU... ini sama? Mari cuba dengan masa berbeza:

let utc_lewat = Utc.with_ymd_and_hms(2024, 1, 15, 17, 30, 0).unwrap();
// 17:30 UTC = 01:30 pada 16 Januari MYT!

let myt_lewat = utc_lewat.with_timezone(&Kuala_Lumpur);
println!("UTC: {:?}", utc_lewat.date_naive()); // 2024-01-15
println!("MYT: {:?}", myt_lewat.date_naive()); // 2024-01-16 (ESOK!)

PELAJARAN PENTING:
  "Tarikh" bergantung pada timezone!
  Satu masa UTC boleh jadi dua tarikh berbeza di timezone berbeza.

  UTC:  2024-01-15 17:30:00 → tarikh "15"
  MYT:  2024-01-16 01:30:00 → tarikh "16" (+8 jam)

Implikasi:
  - "Laporan hari ini" di mana? UTC atau MYT pengguna?
  - "Tarikh expire" jam berapa tepat?
  - SELALU store UTC dalam DB
  - Convert ke TZ pengguna hanya untuk DISPLAY
```
</details>

---

# BAHAGIAN 6: Date/Time Arithmetic ➕

## Tambah, Tolak, Modifikasi

```rust
use chrono::{Utc, Duration, TimeZone, Datelike, Timelike};

fn main() {
    let sekarang = Utc::now();

    // ── Tambah/Tolak Duration ─────────────────────────────────
    let esok        = sekarang + Duration::days(1);
    let semalam     = sekarang - Duration::days(1);
    let 2jam_lepas  = sekarang - Duration::hours(2);
    let 30min_akan  = sekarang + Duration::minutes(30);
    let 1minggu     = sekarang + Duration::weeks(1);

    println!("Esok:    {}", esok.format("%Y-%m-%d"));
    println!("Semalam: {}", semalam.format("%Y-%m-%d"));

    // ── Duration asas ─────────────────────────────────────────
    let d1 = Duration::days(7);
    let d2 = Duration::hours(24);
    let d3 = Duration::minutes(90);
    let d4 = Duration::seconds(3600);

    // Duration boleh negatif
    let negatif = Duration::days(-3);

    // ── Dapatkan komponen Duration ────────────────────────────
    let d = Duration::hours(25);
    println!("Total saat: {}", d.num_seconds());  // 90000
    println!("Total minit: {}", d.num_minutes()); // 1500
    println!("Total jam: {}", d.num_hours());     // 25
    println!("Total hari: {}", d.num_days());      // 1 (25/24 = 1)

    // ── Modifikasi DateTime ───────────────────────────────────
    // Ambil tarikh yang sama, tukar masa
    use chrono::NaiveDate;
    let tarikh = NaiveDate::from_ymd_opt(2024, 1, 15).unwrap();

    // Mula hari (00:00:00)
    let mula_hari = tarikh.and_hms_opt(0, 0, 0).unwrap();

    // Akhir hari (23:59:59)
    let akhir_hari = tarikh.and_hms_opt(23, 59, 59).unwrap();

    // ── Hari pertama / akhir bulan ────────────────────────────
    fn hari_pertama_bulan(dt: &chrono::NaiveDate) -> chrono::NaiveDate {
        chrono::NaiveDate::from_ymd_opt(dt.year(), dt.month(), 1).unwrap()
    }

    fn hari_akhir_bulan(dt: &chrono::NaiveDate) -> chrono::NaiveDate {
        // Hari pertama bulan seterusnya - 1 hari
        let bulan_depan = if dt.month() == 12 {
            chrono::NaiveDate::from_ymd_opt(dt.year() + 1, 1, 1)
        } else {
            chrono::NaiveDate::from_ymd_opt(dt.year(), dt.month() + 1, 1)
        }.unwrap();
        bulan_depan - Duration::days(1)
    }

    let tarikh_ujian = chrono::NaiveDate::from_ymd_opt(2024, 2, 15).unwrap();
    println!("Mula Feb: {}", hari_pertama_bulan(&tarikh_ujian)); // 2024-02-01
    println!("Akhir Feb: {}", hari_akhir_bulan(&tarikh_ujian));  // 2024-02-29 (2024 tahun kabisat)

    // ── Tambah bulan dengan betul ─────────────────────────────
    // chrono ada crate helper untuk ini
    fn tambah_bulan(dt: chrono::NaiveDate, bulan: u32) -> chrono::NaiveDate {
        let tahun_tambah  = (dt.month() - 1 + bulan) / 12;
        let bulan_baru    = (dt.month() - 1 + bulan) % 12 + 1;
        let tahun_baru    = dt.year() + tahun_tambah as i32;

        // Handle kes seperti 31 Jan + 1 bulan = 28/29 Feb
        chrono::NaiveDate::from_ymd_opt(tahun_baru, bulan_baru, dt.day())
            .or_else(|| chrono::NaiveDate::from_ymd_opt(tahun_baru, bulan_baru, 28))
            .unwrap()
    }

    let jan31 = chrono::NaiveDate::from_ymd_opt(2024, 1, 31).unwrap();
    println!("Jan31 + 1 bulan = {}", tambah_bulan(jan31, 1)); // 2024-02-28 (bukan 2024-02-31!)
}
```

---

# BAHAGIAN 7: Duration & Comparison ⚖️

## Kira Beza Masa & Bandingkan

```rust
use chrono::{Utc, NaiveDate, NaiveDateTime, Duration, TimeZone};

fn main() {
    // ── Kira perbezaan antara dua DateTime ───────────────────
    let mula = Utc.with_ymd_and_hms(2024, 1, 1, 0, 0, 0).unwrap();
    let akhir = Utc.with_ymd_and_hms(2024, 1, 15, 8, 30, 0).unwrap();

    let beza = akhir - mula;  // Duration
    println!("Beza hari:  {}", beza.num_days());    // 14
    println!("Beza jam:   {}", beza.num_hours());   // 344
    println!("Beza minit: {}", beza.num_minutes()); // 20670

    // ── Kira umur ─────────────────────────────────────────────
    fn kira_umur(tarikh_lahir: NaiveDate) -> u32 {
        let hari_ini = Utc::now().date_naive();
        let mut umur = hari_ini.year() - tarikh_lahir.year();

        // Kalau belum sampai hari jadi tahun ini
        if hari_ini.month() < tarikh_lahir.month()
            || (hari_ini.month() == tarikh_lahir.month()
                && hari_ini.day() < tarikh_lahir.day())
        {
            umur -= 1;
        }

        umur as u32
    }

    let lahir = NaiveDate::from_ymd_opt(1990, 6, 15).unwrap();
    println!("Umur: {} tahun", kira_umur(lahir));

    // ── Bandingkan DateTime ───────────────────────────────────
    let dt1 = Utc.with_ymd_and_hms(2024, 1, 15, 8, 0, 0).unwrap();
    let dt2 = Utc.with_ymd_and_hms(2024, 1, 15, 10, 0, 0).unwrap();

    println!("dt1 < dt2: {}", dt1 < dt2);   // true
    println!("dt1 > dt2: {}", dt1 > dt2);   // false
    println!("dt1 == dt2: {}", dt1 == dt2); // false

    let min_dt = dt1.min(dt2);  // yang lebih awal
    let max_dt = dt1.max(dt2);  // yang lebih lambat

    // ── Semak dalam julat ─────────────────────────────────────
    let sekarang = Utc::now();
    let token_dibuat = sekarang - Duration::minutes(20);
    let token_tamat = token_dibuat + Duration::hours(1);

    let token_sah = sekarang >= token_dibuat && sekarang <= token_tamat;
    println!("Token sah: {}", token_sah);

    // ── Masa bekerja (tanpa hujung minggu) ───────────────────
    fn hari_bekerja(mula: NaiveDate, akhir: NaiveDate) -> i64 {
        use chrono::Weekday;
        let mut hari = 0i64;
        let mut semasa = mula;
        while semasa < akhir {
            if !matches!(semasa.weekday(), Weekday::Sat | Weekday::Sun) {
                hari += 1;
            }
            semasa += Duration::days(1);
        }
        hari
    }

    let mula_kira = NaiveDate::from_ymd_opt(2024, 1, 15).unwrap(); // Isnin
    let akhir_kira = NaiveDate::from_ymd_opt(2024, 1, 22).unwrap(); // Isnin
    println!("Hari bekerja: {}", hari_bekerja(mula_kira, akhir_kira)); // 5
}
```

---

# BAHAGIAN 8: Serde Integration 🔄

## Simpan & Load DateTime

```toml
[dependencies]
chrono = { version = "0.4", features = ["serde"] }
serde  = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
use chrono::{DateTime, Utc, NaiveDate, NaiveDateTime};
use serde::{Serialize, Deserialize};

// ── Struct dengan datetime fields ─────────────────────────────
#[derive(Debug, Serialize, Deserialize, Clone)]
struct RekodKehadiran {
    id:           u32,
    pekerja_id:   u32,

    // DateTime<Utc> — serialize ke ISO 8601 string
    masa_masuk:   DateTime<Utc>,

    // Option<DateTime<Utc>> — null dalam JSON kalau None
    masa_keluar:  Option<DateTime<Utc>>,

    // NaiveDate — serialize ke "YYYY-MM-DD"
    tarikh:       NaiveDate,

    // Custom format dengan serde attributes
    #[serde(with = "chrono::serde::ts_seconds")]
    dibuat_pada:  DateTime<Utc>,  // simpan sebagai unix timestamp integer
}

// ── Custom format untuk serde ─────────────────────────────────
mod format_my {
    use chrono::{DateTime, Utc, NaiveDateTime};
    use serde::{self, Deserialize, Serializer, Deserializer};

    const FORMAT: &str = "%d/%m/%Y %H:%M:%S";

    pub fn serialize<S>(dt: &DateTime<Utc>, s: S) -> Result<S::Ok, S::Error>
    where S: Serializer {
        let formatted = dt.format(FORMAT).to_string();
        s.serialize_str(&formatted)
    }

    pub fn deserialize<'de, D>(d: D) -> Result<DateTime<Utc>, D::Error>
    where D: Deserializer<'de> {
        let s = String::deserialize(d)?;
        let dt = NaiveDateTime::parse_from_str(&s, FORMAT)
            .map_err(serde::de::Error::custom)?;
        Ok(DateTime::from_naive_utc_and_offset(dt, Utc))
    }
}

#[derive(Debug, Serialize, Deserialize)]
struct RekodCustom {
    #[serde(with = "format_my")]
    masa: DateTime<Utc>,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let rekod = RekodKehadiran {
        id:          1,
        pekerja_id:  101,
        masa_masuk:  Utc::now(),
        masa_keluar: None,
        tarikh:      chrono::NaiveDate::from_ymd_opt(2024, 1, 15).unwrap(),
        dibuat_pada: Utc::now(),
    };

    // Serialize ke JSON
    let json = serde_json::to_string_pretty(&rekod)?;
    println!("{}", json);
    // Output:
    // {
    //   "id": 1,
    //   "pekerja_id": 101,
    //   "masa_masuk": "2024-01-15T08:30:00Z",
    //   "masa_keluar": null,
    //   "tarikh": "2024-01-15",
    //   "dibuat_pada": 1705294200
    // }

    // Deserialize dari JSON
    let json_input = r#"{
        "id": 2,
        "pekerja_id": 102,
        "masa_masuk": "2024-01-15T08:45:00Z",
        "masa_keluar": "2024-01-15T17:30:00Z",
        "tarikh": "2024-01-15",
        "dibuat_pada": 1705294200
    }"#;

    let rekod2: RekodKehadiran = serde_json::from_str(json_input)?;
    println!("{:#?}", rekod2);

    Ok(())
}
```

---

# BAHAGIAN 9: Real-world Patterns 🌍

## Patterns yang Selalu Digunakan

```rust
use chrono::{Utc, Local, TimeZone, Duration, NaiveDate, Datelike, NaiveDateTime, DateTime};
use chrono_tz::Asia::Kuala_Lumpur;

// ── Pattern 1: Simpan UTC, Papar dalam TZ pengguna ────────────
fn simpan_dan_papar() {
    // Simpan dalam DB — sentiasa UTC
    let masa_disimpan: DateTime<Utc> = Utc::now();

    // Papar kepada pengguna Malaysia
    let masa_myt = masa_disimpan.with_timezone(&Kuala_Lumpur);
    println!("Simpan (UTC): {}", masa_disimpan.to_rfc3339());
    println!("Papar (MYT): {}", masa_myt.format("%d/%m/%Y %I:%M %p"));
}

// ── Pattern 2: Pengesahan tarikh input ───────────────────────
fn sahkan_tarikh(input: &str) -> Result<NaiveDate, String> {
    NaiveDate::parse_from_str(input, "%Y-%m-%d")
        .map_err(|e| format!("Format tarikh tidak sah '{}': {}", input, e))
}

fn sahkan_julat_tarikh(
    mula: &str,
    akhir: &str,
) -> Result<(NaiveDate, NaiveDate), String> {
    let mula_dt = sahkan_tarikh(mula)?;
    let akhir_dt = sahkan_tarikh(akhir)?;

    if mula_dt > akhir_dt {
        return Err(format!("Tarikh mula '{}' mesti sebelum tarikh akhir '{}'", mula, akhir));
    }

    let beza = (akhir_dt - mula_dt).num_days();
    if beza > 365 {
        return Err(format!("Julat tidak boleh melebihi 365 hari ({} hari)", beza));
    }

    Ok((mula_dt, akhir_dt))
}

// ── Pattern 3: Kira gaji / billing ───────────────────────────
fn kira_gaji_prorate(gaji_penuh: f64, tarikh_mula: NaiveDate, bulan: u32, tahun: i32) -> f64 {
    // Hari pertama bulan
    let mula_bulan = NaiveDate::from_ymd_opt(tahun, bulan, 1).unwrap();

    // Hari akhir bulan
    let akhir_bulan = {
        let bulan_depan = if bulan == 12 {
            NaiveDate::from_ymd_opt(tahun + 1, 1, 1)
        } else {
            NaiveDate::from_ymd_opt(tahun, bulan + 1, 1)
        }.unwrap();
        bulan_depan - Duration::days(1)
    };

    let jumlah_hari = (akhir_bulan - mula_bulan).num_days() + 1;
    let hari_bekerja = (akhir_bulan - tarikh_mula.max(mula_bulan)).num_days() + 1;

    (gaji_penuh / jumlah_hari as f64) * hari_bekerja as f64
}

// ── Pattern 4: Token JWT expiry ───────────────────────────────
fn jana_expiry_token(tamat_minit: i64) -> (DateTime<Utc>, i64) {
    let sekarang = Utc::now();
    let tamat = sekarang + Duration::minutes(tamat_minit);
    let ts = tamat.timestamp(); // untuk JWT "exp" claim
    (tamat, ts)
}

fn semak_token_sah(exp_timestamp: i64) -> bool {
    let sekarang = Utc::now().timestamp();
    let lebih_masa = 30i64; // 30 saat lebih masa untuk clock skew
    sekarang <= exp_timestamp + lebih_masa
}

// ── Pattern 5: Log dengan timestamp ──────────────────────────
macro_rules! log_ts {
    ($level:literal, $($arg:tt)*) => {
        println!("[{}] [{}] {}",
            Utc::now().format("%Y-%m-%dT%H:%M:%S%.3fZ"),
            $level,
            format!($($arg)*)
        );
    };
}

fn demo_log() {
    log_ts!("INFO",  "Server dimulakan");
    log_ts!("WARN",  "Cache {} tidak bertindak balas", "Redis");
    log_ts!("ERROR", "Pangkalan data gagal: {}", "connection timeout");
}

// ── Pattern 6: Tarikh bekerja Malaysia ───────────────────────
fn adalah_hari_bekerja(tarikh: NaiveDate) -> bool {
    use chrono::Weekday;
    !matches!(tarikh.weekday(), Weekday::Sat | Weekday::Sun)
    // TODO: tambah cuti umum Malaysia
}

fn hari_bekerja_seterusnya(dari: NaiveDate) -> NaiveDate {
    let mut tarikh = dari + Duration::days(1);
    while !adalah_hari_bekerja(tarikh) {
        tarikh += Duration::days(1);
    }
    tarikh
}

fn main() {
    simpan_dan_papar();

    match sahkan_julat_tarikh("2024-01-01", "2024-06-30") {
        Ok((m, a)) => println!("Julat: {} hingga {}", m, a),
        Err(e)     => println!("Ralat: {}", e),
    }

    let gaji = kira_gaji_prorate(
        4500.0,
        NaiveDate::from_ymd_opt(2024, 1, 10).unwrap(), // mula bekerja Jan 10
        1, 2024 // bulan Januari 2024
    );
    println!("Gaji prorate: RM{:.2}", gaji); // dari Jan 10 → Jan 31

    let (tamat, ts) = jana_expiry_token(60);
    println!("Token tamat: {} (ts: {})", tamat.format("%H:%M:%S"), ts);
    println!("Token sah: {}", semak_token_sah(ts));

    demo_log();

    let jumaat = NaiveDate::from_ymd_opt(2024, 1, 19).unwrap(); // Jumaat
    println!("Hari bekerja seterusnya selepas Jumaat: {}",
        hari_bekerja_seterusnya(jumaat)); // 2024-01-22 (Isnin)
}
```

---

# BAHAGIAN 10: Mini Project — Sistem Jadual KADA 🏗️

```rust
use chrono::{
    DateTime, Duration, NaiveDate, NaiveDateTime, NaiveTime,
    Utc, TimeZone, Datelike, Timelike, Weekday,
};
use chrono_tz::Asia::Kuala_Lumpur;
use serde::{Serialize, Deserialize};
use std::collections::HashMap;

// ─── Types ────────────────────────────────────────────────────

#[derive(Debug, Clone, Serialize, Deserialize)]
struct SlotMesyuarat {
    id:          u32,
    tajuk:       String,
    mula:        DateTime<Utc>,    // simpan UTC
    tamat:       DateTime<Utc>,    // simpan UTC
    lokasi:      String,
    peserta:     Vec<String>,
}

impl SlotMesyuarat {
    fn baru(
        tajuk: &str,
        mula_myt: NaiveDateTime,
        tempoh_minit: i64,
        lokasi: &str,
        peserta: Vec<String>,
    ) -> Self {
        // Convert dari MYT ke UTC untuk simpan
        let mula_utc = Kuala_Lumpur
            .from_local_datetime(&mula_myt)
            .unwrap()
            .with_timezone(&Utc);

        let tamat_utc = mula_utc + Duration::minutes(tempoh_minit);

        SlotMesyuarat {
            id:      rand_id(),
            tajuk:   tajuk.into(),
            mula:    mula_utc,
            tamat:   tamat_utc,
            lokasi:  lokasi.into(),
            peserta,
        }
    }

    // Display dalam MYT
    fn papar_masa(&self) -> String {
        let mula_myt = self.mula.with_timezone(&Kuala_Lumpur);
        let tamat_myt = self.tamat.with_timezone(&Kuala_Lumpur);
        format!(
            "{} — {} {}",
            mula_myt.format("%d %b %Y"),
            mula_myt.format("%H:%M"),
            tamat_myt.format("— %H:%M"),
        )
    }

    fn tempoh_minit(&self) -> i64 {
        (self.tamat - self.mula).num_minutes()
    }

    fn bertindih_dengan(&self, lain: &SlotMesyuarat) -> bool {
        self.mula < lain.tamat && self.tamat > lain.mula
    }
}

fn rand_id() -> u32 {
    use std::time::{SystemTime, UNIX_EPOCH};
    (SystemTime::now().duration_since(UNIX_EPOCH).unwrap().subsec_nanos() % 10000) as u32
}

// ─── Sistem Jadual ─────────────────────────────────────────────

struct SistemJadual {
    slot: Vec<SlotMesyuarat>,
    zon_masa: chrono_tz::Tz,
}

impl SistemJadual {
    fn baru() -> Self {
        SistemJadual {
            slot:     Vec::new(),
            zon_masa: Kuala_Lumpur,
        }
    }

    fn tambah_slot(&mut self, slot: SlotMesyuarat) -> Result<(), String> {
        // Semak pertindihan
        let bertindih: Vec<&SlotMesyuarat> = self.slot.iter()
            .filter(|s| s.bertindih_dengan(&slot))
            .collect();

        if !bertindih.is_empty() {
            return Err(format!(
                "Bertindih dengan: {}",
                bertindih.iter().map(|s| s.tajuk.clone()).collect::<Vec<_>>().join(", ")
            ));
        }

        self.slot.push(slot);
        self.slot.sort_by_key(|s| s.mula);
        Ok(())
    }

    fn slot_hari(&self, tarikh: NaiveDate) -> Vec<&SlotMesyuarat> {
        self.slot.iter()
            .filter(|s| {
                let tarikh_myt = s.mula.with_timezone(&self.zon_masa).date_naive();
                tarikh_myt == tarikh
            })
            .collect()
    }

    fn slot_minggu(&self, mula_minggu: NaiveDate) -> Vec<&SlotMesyuarat> {
        let akhir_minggu = mula_minggu + Duration::days(7);
        self.slot.iter()
            .filter(|s| {
                let tarikh = s.mula.with_timezone(&self.zon_masa).date_naive();
                tarikh >= mula_minggu && tarikh < akhir_minggu
            })
            .collect()
    }

    fn cetak_jadual_hari(&self, tarikh: NaiveDate) {
        let nama_hari = match tarikh.weekday() {
            Weekday::Mon => "Isnin",   Weekday::Tue => "Selasa",
            Weekday::Wed => "Rabu",    Weekday::Thu => "Khamis",
            Weekday::Fri => "Jumaat",  Weekday::Sat => "Sabtu",
            Weekday::Sun => "Ahad",
        };

        println!("\n{'─'*50}");
        println!("  {} — {}", nama_hari, tarikh.format("%d %B %Y"));
        println!("{'─'*50}");

        let senarai = self.slot_hari(tarikh);
        if senarai.is_empty() {
            println!("  (Tiada mesyuarat)");
        } else {
            for slot in senarai {
                let mula = slot.mula.with_timezone(&self.zon_masa);
                let tamat = slot.tamat.with_timezone(&self.zon_masa);
                println!(
                    "  {:5} — {:5}  {} ({}min)",
                    mula.format("%H:%M"),
                    tamat.format("%H:%M"),
                    slot.tajuk,
                    slot.tempoh_minit()
                );
                println!("             📍 {}", slot.lokasi);
                println!("             👥 {}", slot.peserta.join(", "));
            }
        }
    }

    fn statistik_minggu(&self, mula_minggu: NaiveDate) -> HashMap<&str, i64> {
        let senarai = self.slot_minggu(mula_minggu);
        let mut stat = HashMap::new();
        stat.insert("jumlah_mesyuarat", senarai.len() as i64);
        stat.insert("jumlah_minit", senarai.iter().map(|s| s.tempoh_minit()).sum());
        stat
    }
}

// ─── Demo ──────────────────────────────────────────────────────

fn main() {
    let mut jadual = SistemJadual::baru();

    // Isnin 15 Jan 2024
    let isnin = NaiveDate::from_ymd_opt(2024, 1, 15).unwrap();

    // Tambah mesyuarat dalam MYT
    let m1 = SlotMesyuarat::baru(
        "Mesyuarat Pengurusan",
        NaiveDateTime::new(isnin, NaiveTime::from_hms_opt(9, 0, 0).unwrap()),
        60, "Bilik Mesyuarat A",
        vec!["Ali".into(), "Siti".into(), "Dr. Hassan".into()],
    );

    let m2 = SlotMesyuarat::baru(
        "Taklimat ICT",
        NaiveDateTime::new(isnin, NaiveTime::from_hms_opt(11, 0, 0).unwrap()),
        90, "Lab Komputer",
        vec!["Ali".into(), "Amin".into()],
    );

    let m3 = SlotMesyuarat::baru(
        "Review Projek KADA Mobile",
        NaiveDateTime::new(isnin, NaiveTime::from_hms_opt(14, 30, 0).unwrap()),
        120, "Bilik Mesyuarat B",
        vec!["Ali".into(), "Siti".into(), "Amin".into(), "Zara".into()],
    );

    // Mesyuarat yang bertindih
    let m_bertindih = SlotMesyuarat::baru(
        "Mesyuarat Bertindih",
        NaiveDateTime::new(isnin, NaiveTime::from_hms_opt(9, 30, 0).unwrap()),
        30, "Bilik Lain", vec![],
    );

    match jadual.tambah_slot(m1) {
        Ok(()) => println!("✔ Mesyuarat 1 ditambah"),
        Err(e) => println!("✗ {}", e),
    }
    match jadual.tambah_slot(m2) {
        Ok(()) => println!("✔ Mesyuarat 2 ditambah"),
        Err(e) => println!("✗ {}", e),
    }
    match jadual.tambah_slot(m3) {
        Ok(()) => println!("✔ Mesyuarat 3 ditambah"),
        Err(e) => println!("✗ {}", e),
    }
    match jadual.tambah_slot(m_bertindih) {
        Ok(()) => println!("✔ Ditambah"),
        Err(e) => println!("✗ Bertindih: {}", e),
    }

    // Cetak jadual
    jadual.cetak_jadual_hari(isnin);

    // Statistik
    let stat = jadual.statistik_minggu(isnin);
    println!("\n📊 Statistik Minggu:");
    println!("   Jumlah mesyuarat: {}", stat["jumlah_mesyuarat"]);
    println!("   Jumlah masa: {} minit ({:.1} jam)",
        stat["jumlah_minit"],
        *stat["jumlah_minit"] as f64 / 60.0);

    // Demo serde
    if let Some(slot) = jadual.slot.first() {
        let json = serde_json::to_string_pretty(slot).unwrap();
        println!("\n📄 JSON (UTC stored):\n{}", json);
    }
}
```

---

# 📋 Rujukan Pantas — DateTime Cheat Sheet

## Buat DateTime

```rust
use chrono::{Utc, Local, TimeZone, NaiveDate, NaiveDateTime};

Utc::now()                                    // UTC sekarang
Local::now()                                  // Local sekarang
Utc.with_ymd_and_hms(2024, 1, 15, 8, 30, 0) // specific UTC

NaiveDate::from_ymd_opt(2024, 1, 15)         // tarikh sahaja
NaiveDateTime::parse_from_str(s, fmt)        // parse dari string
DateTime::parse_from_rfc3339(s)              // parse ISO 8601
Utc.timestamp_opt(ts, 0)                     // dari unix timestamp
```

## Format

```rust
dt.format("%Y-%m-%d")            // "2024-01-15"
dt.format("%d/%m/%Y %H:%M")      // "15/01/2024 08:30"
dt.to_rfc3339()                  // "2024-01-15T08:30:00Z"
dt.timestamp()                   // 1705294200 (unix ts)
dt.timestamp_millis()            // milisaat
```

## Arithmetic

```rust
dt + Duration::days(7)           // tambah 7 hari
dt - Duration::hours(2)          // tolak 2 jam
dt2 - dt1                        // → Duration
beza.num_days()                  // komponen Duration
beza.num_hours()
beza.num_minutes()
```

## Timezone

```rust
Utc::now().with_timezone(&Local)           // UTC → Local
Utc::now().with_timezone(&Kuala_Lumpur)   // UTC → MYT
myt_dt.with_timezone(&Utc)                // MYT → UTC
dt.date_naive()                           // buang timezone, ambil tarikh
```

## Panduan Simpan

```
✔ DB        → DateTime<Utc> atau Unix timestamp (i64)
✔ API JSON  → RFC 3339 string ("2024-01-15T08:30:00Z")
✔ Display   → convert ke TZ pengguna sebelum format
✗ JANGAN   → simpan DateTime<Local> dalam DB
✗ JANGAN   → bandingkan NaiveDateTime dari TZ berbeza
```

---

## 🏆 Cabaran Akhir

Cuba implement salah satu:

1. **Penukar Timezone** — masukkan masa dan dari/ke timezone, output waktu setara
2. **Pengira Hari Cuti** — senarai cuti umum Malaysia, kira hari bekerja antara dua tarikh
3. **Pengesahan Jadual** — detect mesyuarat bertindih, cadangkan slot kosong
4. **Laporan Kehadiran** — kira jam bekerja, OT, dan lewat berdasarkan rekod masuk/keluar
5. **Reminder System** — jana senarai tarikh berulang (mingguan/bulanan) dalam julat tertentu

---

*Masa dalam perisian = perincian yang paling mudah dipandang remeh.*
*Simpan UTC. Papar dalam TZ pengguna. Gunakan chrono dengan bijak.* 🦀
