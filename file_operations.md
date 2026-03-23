# 📁 File Operations dalam Rust — Panduan Lengkap

> Semua tentang kerja dengan fail dan direktori dalam Rust:
> baca, tulis, salin, pindah, watch, dan async I/O.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Gambaran Besar

```
std::fs   → Operasi fail dan direktori standard
std::path → Path manipulation (PathBuf, Path)
std::io   → I/O traits (Read, Write, BufRead, BufWrite)
tokio::fs → Async file operations

Hierarki I/O Rust:
  File  →  implements Read + Write
  BufReader(File)  → implements BufRead (baris per baris)
  BufWriter(File)  → implements Write (dengan buffer)

Perbezaan Rust vs bahasa lain:
  PHP:   $f = fopen("fail.txt", "r"); — global error handling
  Rust:  File::open("fail.txt")?;     — explicit error handling
         Tiada fail terbuka bocor!    — Drop auto-tutup fail
```

---

## Peta Pembelajaran

```
Bahagian 1  → Path & PathBuf
Bahagian 2  → Baca Fail
Bahagian 3  → Tulis Fail
Bahagian 4  → Operasi Fail (salin, pindah, padam)
Bahagian 5  → Direktori
Bahagian 6  → Metadata & Permissions
Bahagian 7  → Fail Sementara
Bahagian 8  → Fail CSV, JSON, TOML
Bahagian 9  → Async File I/O
Bahagian 10 → Mini Project: Sistem Pengurusan Fail
```

---

# BAHAGIAN 1: Path & PathBuf 🗂️

## Path vs PathBuf

```rust
use std::path::{Path, PathBuf};

fn main() {
    // Path     → &str (borrowed, tidak boleh ubah)
    // PathBuf  → String (owned, boleh ubah)

    // ── Buat PathBuf ──────────────────────────────────────────
    let mut laluan = PathBuf::new();
    laluan.push("/home");
    laluan.push("ali");
    laluan.push("dokumen");
    laluan.push("laporan.txt");
    println!("{}", laluan.display()); // /home/ali/dokumen/laporan.txt

    // Cara lebih mudah
    let laluan2 = PathBuf::from("/home/ali/dokumen/laporan.txt");
    let laluan3: PathBuf = ["/home", "ali", "dokumen", "laporan.txt"]
        .iter()
        .collect();

    // ── Path operations ───────────────────────────────────────
    let p = Path::new("/home/ali/dokumen/laporan.txt");

    println!("Fail wujud:  {}", p.exists());
    println!("Adalah fail: {}", p.is_file());
    println!("Adalah dir:  {}", p.is_dir());

    // Komponen path
    println!("Nama fail:  {:?}", p.file_name());    // Some("laporan.txt")
    println!("Stem:       {:?}", p.file_stem());    // Some("laporan")
    println!("Extension:  {:?}", p.extension());    // Some("txt")
    println!("Parent:     {:?}", p.parent());       // Some("/home/ali/dokumen")

    // Bina path relatif dari komponen
    let direktori = Path::new("/data/kada");
    let fail_penuh = direktori.join("laporan").join("2024.txt");
    println!("{}", fail_penuh.display()); // /data/kada/laporan/2024.txt

    // Tukar extension
    let mut ubah = PathBuf::from("laporan.txt");
    ubah.set_extension("pdf");
    println!("{}", ubah.display()); // laporan.pdf

    // Tukar nama fail sahaja
    let mut ubah2 = PathBuf::from("/home/ali/fail.txt");
    ubah2.set_file_name("fail_baru.txt");
    println!("{}", ubah2.display()); // /home/ali/fail_baru.txt

    // ── Convert antara Path & String ──────────────────────────
    let laluan_str = "/home/ali/fail.txt";

    let p = Path::new(laluan_str);       // &str → &Path
    let pb = PathBuf::from(laluan_str);  // &str → PathBuf

    let s: &str = p.to_str().unwrap();   // &Path → &str (mungkin gagal kalau bukan UTF-8)
    let owned: String = p.to_string_lossy().into_owned(); // &Path → String (lossless)

    // ── Cross-platform path ────────────────────────────────────
    use std::path::MAIN_SEPARATOR;
    println!("Path separator: {}", MAIN_SEPARATOR); // / pada Linux/Mac, \ pada Windows

    // Guna join() untuk cross-platform!
    let laluan_selamat = Path::new("data").join("fail.txt");
    println!("{}", laluan_selamat.display()); // data/fail.txt (atau data\fail.txt pada Windows)
}
```

---

## Path Manipulation Lanjutan

```rust
use std::path::{Path, PathBuf, Component};

fn main() {
    // ── Iterate komponen path ─────────────────────────────────
    let p = Path::new("/home/ali/dokumen/laporan.txt");

    for komponen in p.components() {
        match komponen {
            Component::RootDir    => println!("Root: /"),
            Component::Normal(n)  => println!("Dir/Fail: {:?}", n),
            Component::CurDir     => println!("CurDir: ."),
            Component::ParentDir  => println!("Parent: .."),
            Component::Prefix(x)  => println!("Prefix: {:?}", x), // Windows sahaja
        }
    }

    // ── Normalize path (resolve ./ dan ../) ───────────────────
    let kotor = Path::new("/home/ali/../ali/./dokumen");
    // canonicalize() perlu path wujud!
    // Untuk logik sahaja, guna:
    let bersih: PathBuf = kotor.components().collect();
    println!("{}", bersih.display()); // /home/ali/dokumen (on Linux)

    // ── Semak prefix ──────────────────────────────────────────
    let asas = Path::new("/home/ali");
    let anak = Path::new("/home/ali/dokumen/fail.txt");

    if anak.starts_with(asas) {
        // Dapatkan path relatif
        let relatif = anak.strip_prefix(asas).unwrap();
        println!("Relatif: {}", relatif.display()); // dokumen/fail.txt
    }

    // ── Direktori rumah ───────────────────────────────────────
    if let Some(rumah) = dirs::home_dir() { // perlu crate 'dirs'
        println!("Home: {}", rumah.display());
    }

    // Guna env::var sebagai alternatif
    if let Ok(rumah) = std::env::var("HOME") {
        println!("HOME: {}", rumah);
    }

    // ── Direktori semasa ──────────────────────────────────────
    let semasa = std::env::current_dir().unwrap();
    println!("CWD: {}", semasa.display());

    let exe = std::env::current_exe().unwrap();
    println!("Executable: {}", exe.display());
}
```

---

## 🧠 Brain Teaser #1

Apakah yang berbeza antara `Path::new("data/fail.txt")` dan `PathBuf::from("data/fail.txt")`?

<details>
<summary>👀 Jawapan</summary>

```
Path      → seperti &str (borrow, tiada pemilikan)
            - Ringan, tidak allocate
            - Tidak boleh diubah
            - Hidup selama rujukan asalnya
            - Biasa untuk parameter fungsi

PathBuf   → seperti String (owned, ada pemilikan)
            - Boleh diubah (.push(), .set_extension(), dll)
            - Allocate memori
            - Boleh disimpan dalam struct
            - Boleh dihantar tanpa lifetime issue

fn baca_fail(path: &Path) { ... }  // ← guna &Path untuk parameter
                                    //   boleh terima &PathBuf auto!

struct Config {
    laluan: PathBuf,  // ← simpan sebagai PathBuf dalam struct
}

Analogi:
  Path    = &str  → pinjam, ringan
  PathBuf = String → milik, boleh ubah

Gunakan Path (&Path) dalam function parameters untuk flexibility.
Gunakan PathBuf apabila perlu simpan atau ubah path.
```
</details>

---

# BAHAGIAN 2: Baca Fail 📖

## Baca Semua Sekaligus

```rust
use std::fs;
use std::io;

fn main() -> io::Result<()> {
    // ── Cara paling mudah: baca semua ke String ───────────────
    let kandungan = fs::read_to_string("fail.txt")?;
    println!("Kandungan:\n{}", kandungan);
    println!("Panjang: {} bytes", kandungan.len());

    // ── Baca sebagai bytes ────────────────────────────────────
    let bytes: Vec<u8> = fs::read("gambar.png")?;
    println!("Saiz: {} bytes", bytes.len());

    // ── Baca dengan pengendalian error yang lebih baik ────────
    match fs::read_to_string("konfigurasi.toml") {
        Ok(kandungan) => println!("Config:\n{}", kandungan),
        Err(e) if e.kind() == io::ErrorKind::NotFound =>
            println!("Fail tidak dijumpai, guna nilai lalai"),
        Err(e) =>
            eprintln!("Error: {}", e),
    }

    Ok(())
}
```

---

## Baca Baris per Baris

```rust
use std::fs::File;
use std::io::{self, BufRead, BufReader};

fn main() -> io::Result<()> {
    let fail = File::open("data.txt")?;
    let pembaca = BufReader::new(fail);

    // ── Iterate baris dengan lines() ──────────────────────────
    for (no, baris) in pembaca.lines().enumerate() {
        let baris = baris?; // lines() return Result<String>
        println!("{:4}: {}", no + 1, baris);
    }

    // ── Baca baris dengan filter ──────────────────────────────
    let fail = File::open("data.txt")?;
    let pembaca = BufReader::new(fail);

    let baris_tidak_kosong: Vec<String> = pembaca
        .lines()
        .filter_map(|l| l.ok())
        .filter(|l| !l.trim().is_empty())
        .collect();

    println!("{} baris tidak kosong", baris_tidak_kosong.len());

    // ── Baca baris dengan read_line() (manual) ─────────────────
    let fail = File::open("data.txt")?;
    let mut pembaca = BufReader::new(fail);
    let mut baris = String::new();
    let mut kiraan = 0;

    loop {
        baris.clear();
        let n = pembaca.read_line(&mut baris)?;
        if n == 0 { break; } // EOF
        kiraan += 1;
        print!("{}", baris);
    }
    println!("Total: {} baris", kiraan);

    Ok(())
}
```

---

## Baca Fail Besar Secara Streaming

```rust
use std::fs::File;
use std::io::{self, Read, BufReader};

fn proses_fail_besar(laluan: &str) -> io::Result<u64> {
    let fail = File::open(laluan)?;
    let metadata = fail.metadata()?;
    let saiz_fail = metadata.len();

    println!("Memproses fail {} MB...", saiz_fail / 1024 / 1024);

    let mut pembaca = BufReader::with_capacity(65536, fail); // buffer 64KB
    let mut buf = vec![0u8; 65536];
    let mut jumlah_bait: u64 = 0;
    let mut jumlah_baris: u64 = 0;

    loop {
        let n = pembaca.read(&mut buf)?;
        if n == 0 { break; }

        jumlah_bait += n as u64;
        jumlah_baris += buf[..n].iter().filter(|&&b| b == b'\n').count() as u64;

        // Laporan kemajuan
        let peratus = jumlah_bait * 100 / saiz_fail;
        print!("\r{:3}% ({} MB diproses)", peratus, jumlah_bait / 1024 / 1024);
    }

    println!("\nSelesai! {} baris, {} bytes", jumlah_baris, jumlah_bait);
    Ok(jumlah_baris)
}

// Memory-mapped files untuk fail SANGAT besar
// (perlu crate 'memmap2')
fn baca_mmap(laluan: &str) -> io::Result<()> {
    let fail = File::open(laluan)?;
    // let mmap = unsafe { memmap2::Mmap::map(&fail)? };
    // Akses seperti slice: &mmap[0..100]
    Ok(())
}
```

---

# BAHAGIAN 3: Tulis Fail ✍️

## Tulis Mudah

```rust
use std::fs;
use std::io::{self, Write, BufWriter};
use std::fs::{File, OpenOptions};

fn main() -> io::Result<()> {
    // ── Tulis String ke fail (OVERWRITE) ──────────────────────
    fs::write("output.txt", "Hello, Rust!\n")?;
    fs::write("output.txt", b"Boleh guna bytes juga\n")?;

    // ── Append ke fail yang sedia ada ─────────────────────────
    let mut fail = OpenOptions::new()
        .append(true)
        .create(true)    // buat fail kalau tidak ada
        .open("log.txt")?;

    writeln!(fail, "Log entry: {}", chrono_waktu())?;
    writeln!(fail, "Lagi satu baris")?;

    // ── Tulis dengan buffer (lebih cepat untuk banyak tulis) ──
    let fail = File::create("besar.txt")?;
    let mut penulis = BufWriter::new(fail);

    for i in 0..10_000 {
        writeln!(penulis, "Baris ke-{}: data penting", i)?;
    }

    penulis.flush()?; // pastikan semua dalam buffer ditulis ke disk!
    // BufWriter flush() secara automatik bila di-drop, tapi explicit lebih baik

    // ── OpenOptions — kawalan penuh ───────────────────────────
    let fail = OpenOptions::new()
        .read(true)       // boleh baca
        .write(true)      // boleh tulis
        .create(true)     // buat kalau tidak ada
        .truncate(false)  // JANGAN padam kandungan lama
        .open("data.txt")?;

    Ok(())
}

fn chrono_waktu() -> String {
    // Simplified — dalam realiti guna crate chrono
    format!("{:?}", std::time::SystemTime::now())
}
```

---

## Tulis Fail Selamat — Atomic Write

```rust
use std::fs;
use std::io::{self, Write};
use std::path::Path;

// Tulis ke fail sementara dulu, kemudian rename
// Ini memastikan fail tidak corrupt kalau program crash semasa menulis!
fn tulis_atom(laluan: &Path, kandungan: &[u8]) -> io::Result<()> {
    let laluan_sementara = laluan.with_extension("tmp");

    // Tulis ke fail sementara
    fs::write(&laluan_sementara, kandungan)?;

    // Atomic rename — ini adalah operasi atom pada kebanyakan OS!
    fs::rename(&laluan_sementara, laluan)?;

    println!("Tulis atom berjaya ke {}", laluan.display());
    Ok(())
}

// ── Backup sebelum overwrite ──────────────────────────────────
fn tulis_dengan_backup(laluan: &Path, kandungan: &str) -> io::Result<()> {
    // Buat backup kalau fail sudah ada
    if laluan.exists() {
        let backup = laluan.with_extension("bak");
        fs::copy(laluan, &backup)?;
        println!("Backup dibuat: {}", backup.display());
    }

    // Tulis kandungan baru
    fs::write(laluan, kandungan)?;
    println!("Fail dikemaskini: {}", laluan.display());
    Ok(())
}

fn main() -> io::Result<()> {
    let laluan = Path::new("penting.txt");

    // Tulis atom
    tulis_atom(laluan, b"Data penting yang tidak boleh corrupt!")?;

    // Dengan backup
    tulis_dengan_backup(laluan, "Kandungan baru yang selamat")?;

    // Bersihkan
    fs::remove_file(laluan)?;
    if Path::new("penting.bak").exists() {
        fs::remove_file("penting.bak")?;
    }

    Ok(())
}
```

---

## Seek dalam Fail

```rust
use std::fs::{File, OpenOptions};
use std::io::{self, Read, Write, Seek, SeekFrom};

fn main() -> io::Result<()> {
    // Buat fail untuk demo
    fs::write("seek_demo.txt", "ABCDEFGHIJKLMNOPQRSTUVWXYZ")?;

    let mut fail = OpenOptions::new()
        .read(true)
        .write(true)
        .open("seek_demo.txt")?;

    // ── Seek ke kedudukan tertentu ────────────────────────────
    fail.seek(SeekFrom::Start(5))?;  // ke byte ke-5 dari mula
    let mut buf = [0u8; 5];
    fail.read_exact(&mut buf)?;
    println!("Baca dari byte 5: {}", String::from_utf8_lossy(&buf)); // FGHIJ

    // Seek dari akhir
    fail.seek(SeekFrom::End(-5))?;   // 5 byte sebelum akhir
    fail.read_exact(&mut buf)?;
    println!("5 byte terakhir: {}", String::from_utf8_lossy(&buf)); // VWXYZ

    // Seek relatif
    fail.seek(SeekFrom::Start(0))?;  // kembali ke mula
    fail.seek(SeekFrom::Current(10))?; // maju 10 byte
    fail.read_exact(&mut buf)?;
    println!("Dari byte 10: {}", String::from_utf8_lossy(&buf)); // KLMNO

    // Tulis di kedudukan tertentu
    fail.seek(SeekFrom::Start(0))?;
    fail.write_all(b"abcde")?;       // tukar 5 char pertama ke lowercase
    fail.seek(SeekFrom::Start(0))?;
    let mut semua = String::new();
    fail.read_to_string(&mut semua)?;
    println!("Selepas tulis: {}", semua); // abcdeFGHIJKLMNOPQRSTUVWXYZ

    // Dapatkan kedudukan semasa
    let pos = fail.seek(SeekFrom::Current(0))?; // atau fail.stream_position()?
    println!("Kedudukan semasa: {}", pos);

    // Bersihkan
    fs::remove_file("seek_demo.txt")?;
    Ok(())
}
```

---

# BAHAGIAN 4: Operasi Fail 🔧

## Salin, Pindah, Padam

```rust
use std::fs;
use std::path::Path;
use std::io;

fn main() -> io::Result<()> {
    // Buat fail untuk demo
    fs::write("asal.txt", "Ini kandungan asal")?;

    // ── Salin fail ────────────────────────────────────────────
    let saiz = fs::copy("asal.txt", "salinan.txt")?;
    println!("Disalin: {} bytes", saiz);

    // ── Pindah / Rename fail ──────────────────────────────────
    fs::rename("salinan.txt", "dipindah.txt")?;
    println!("Fail dipindah!");
    // AWAS: fs::rename() pada Windows mungkin gagal kalau destinasi wujud!

    // Cara lebih selamat untuk rename/pindah
    fn pindah_selamat(dari: &Path, ke: &Path) -> io::Result<()> {
        if ke.exists() {
            fs::remove_file(ke)?; // padam dulu
        }
        fs::rename(dari, ke)?;
        Ok(())
    }

    // ── Padam fail ────────────────────────────────────────────
    fs::remove_file("dipindah.txt")?;
    println!("Fail dipadam");

    // Padam dengan semak kewujudan dulu
    let fail_padam = Path::new("mungkin_tiada.txt");
    if fail_padam.exists() {
        fs::remove_file(fail_padam)?;
    }
    // Atau:
    match fs::remove_file(fail_padam) {
        Ok(())                                                  => println!("Dipadam"),
        Err(e) if e.kind() == io::ErrorKind::NotFound => println!("Memang tiada"),
        Err(e)                                                  => return Err(e),
    }

    // ── Hardlink ──────────────────────────────────────────────
    fs::write("asal.txt", "Kandungan")?;
    fs::hard_link("asal.txt", "hardlink.txt")?;
    // Kedua-dua fail kongsi data yang sama!

    // ── Symlink ───────────────────────────────────────────────
    #[cfg(unix)]
    std::os::unix::fs::symlink("asal.txt", "symlink.txt")?;
    #[cfg(windows)]
    std::os::windows::fs::symlink_file("asal.txt", "symlink.txt")?;

    // Bersihkan
    fs::remove_file("asal.txt")?;
    fs::remove_file("hardlink.txt")?;

    Ok(())
}
```

---

## Salin Direktori Secara Rekursif

```rust
use std::fs;
use std::path::Path;
use std::io;

fn salin_direktori(dari: &Path, ke: &Path) -> io::Result<u64> {
    fs::create_dir_all(ke)?;
    let mut jumlah_bait = 0u64;

    for entri in fs::read_dir(dari)? {
        let entri = entri?;
        let jenis = entri.file_type()?;
        let dari_laluan = entri.path();
        let ke_laluan = ke.join(entri.file_name());

        if jenis.is_dir() {
            jumlah_bait += salin_direktori(&dari_laluan, &ke_laluan)?;
        } else {
            jumlah_bait += fs::copy(&dari_laluan, &ke_laluan)?;
            println!("  Salin: {}", dari_laluan.display());
        }
    }

    Ok(jumlah_bait)
}

fn padam_direktori_rekursif(laluan: &Path) -> io::Result<()> {
    if laluan.is_dir() {
        fs::remove_dir_all(laluan)?; // padam direktori dan semua kandungannya
        println!("Direktori dipadam: {}", laluan.display());
    }
    Ok(())
}

fn main() -> io::Result<()> {
    // Buat struktur untuk demo
    fs::create_dir_all("ujian/sub")?;
    fs::write("ujian/a.txt", "fail A")?;
    fs::write("ujian/b.txt", "fail B")?;
    fs::write("ujian/sub/c.txt", "fail C dalam sub")?;

    // Salin direktori
    let bait = salin_direktori(Path::new("ujian"), Path::new("ujian_salinan"))?;
    println!("Disalin: {} bytes", bait);

    // Bersihkan
    fs::remove_dir_all("ujian")?;
    fs::remove_dir_all("ujian_salinan")?;
    Ok(())
}
```

---

# BAHAGIAN 5: Direktori 📂

## Buat, Baca, Urus Direktori

```rust
use std::fs;
use std::path::Path;
use std::io;

fn main() -> io::Result<()> {
    // ── Buat direktori ────────────────────────────────────────
    fs::create_dir("direktori_baru")?;         // GAGAL jika ibu bapa tidak ada
    fs::create_dir_all("a/b/c/d")?;            // buat semua peringkat sekaligus

    // ── Baca kandungan direktori ──────────────────────────────
    for entri in fs::read_dir(".")? {
        let entri = entri?;
        let laluan = entri.path();
        let metadata = entri.metadata()?;

        let jenis = if metadata.is_dir() { "DIR " }
                    else if metadata.is_file() { "FAIL" }
                    else { "LAIN" };

        println!("[{}] {:30} {:>10} bytes",
            jenis,
            laluan.display(),
            metadata.len()
        );
    }

    // ── Cari fail dengan pattern ──────────────────────────────
    fn cari_fail_rekursif(dir: &Path, sambungan: &str) -> io::Result<Vec<std::path::PathBuf>> {
        let mut dijumpai = Vec::new();

        for entri in fs::read_dir(dir)? {
            let entri = entri?;
            let laluan = entri.path();

            if laluan.is_dir() {
                dijumpai.extend(cari_fail_rekursif(&laluan, sambungan)?);
            } else if laluan.extension().map_or(false, |e| e == sambungan) {
                dijumpai.push(laluan);
            }
        }

        Ok(dijumpai)
    }

    let fail_rust = cari_fail_rekursif(Path::new("src"), "rs")?;
    println!("Fail .rs dijumpai:");
    for f in &fail_rust {
        println!("  {}", f.display());
    }

    // ── Padam direktori ───────────────────────────────────────
    fs::remove_dir("direktori_baru")?;        // GAGAL jika tidak kosong
    fs::remove_dir_all("a")?;                 // padam walaupun tidak kosong

    Ok(())
}
```

---

## Glob Pattern dengan walkdir

```toml
[dependencies]
walkdir = "2"
glob    = "0.3"
```

```rust
use walkdir::{WalkDir, DirEntry};
use std::path::Path;

fn main() {
    // ── WalkDir — rekursif dengan lebih kawalan ────────────────
    for entri in WalkDir::new(".")
        .max_depth(3)            // maksimum 3 peringkat dalam
        .follow_links(false)     // jangan ikut symlinks
        .sort_by_file_name()     // sort mengikut nama
    {
        match entri {
            Ok(e) => {
                let inden = "  ".repeat(e.depth());
                let nama = e.file_name().to_string_lossy();
                println!("{}{}", inden, nama);
            }
            Err(e) => eprintln!("Error: {}", e),
        }
    }

    // ── Filter dalam WalkDir ──────────────────────────────────
    fn adalah_tersembunyi(e: &DirEntry) -> bool {
        e.file_name()
            .to_str()
            .map(|s| s.starts_with('.'))
            .unwrap_or(false)
    }

    let fail_tanpa_tersembunyi = WalkDir::new(".")
        .into_iter()
        .filter_entry(|e| !adalah_tersembunyi(e)) // skip folder .git, .vscode dll
        .filter_map(|e| e.ok())
        .filter(|e| e.file_type().is_file())
        .filter(|e| e.path().extension().map_or(false, |ext| ext == "rs"));

    println!("\nFail .rs (tanpa tersembunyi):");
    for entri in fail_tanpa_tersembunyi {
        println!("  {}", entri.path().display());
    }

    // ── Glob pattern ──────────────────────────────────────────
    for laluan in glob::glob("src/**/*.rs").expect("Pattern tidak sah") {
        match laluan {
            Ok(p)  => println!("{}", p.display()),
            Err(e) => eprintln!("{}", e),
        }
    }

    // Pattern lebih kompleks
    for laluan in glob::glob("**/*.{txt,md,toml}").unwrap().flatten() {
        println!("Dokumen: {}", laluan.display());
    }
}
```

---

# BAHAGIAN 6: Metadata & Permissions 📊

## Baca Metadata Fail

```rust
use std::fs;
use std::time::SystemTime;
use std::path::Path;

fn main() -> std::io::Result<()> {
    let laluan = Path::new("Cargo.toml");

    if !laluan.exists() {
        fs::write(laluan, "[package]\nname = \"demo\"")?;
    }

    let meta = fs::metadata(laluan)?;

    println!("Jenis:         {}", if meta.is_file() { "Fail" } else { "Direktori" });
    println!("Saiz:          {} bytes", meta.len());
    println!("Baca sahaja:   {}", meta.permissions().readonly());

    // Tarikh/masa
    let ubah = meta.modified()?;
    let cipta = meta.created();       // mungkin tidak disokong semua platform
    let akses = meta.accessed();      // mungkin tidak disokong semua platform

    fn format_masa(masa: SystemTime) -> String {
        let saat = masa.duration_since(SystemTime::UNIX_EPOCH)
            .unwrap_or_default()
            .as_secs();
        // Simplified — guna chrono untuk format betul
        format!("Unix: {}", saat)
    }

    println!("Ubah:          {}", format_masa(ubah));

    // ── Permission (Unix) ─────────────────────────────────────
    #[cfg(unix)]
    {
        use std::os::unix::fs::PermissionsExt;
        let mod_bit = meta.permissions().mode();
        println!("Mode:          {:o}", mod_bit & 0o777);
        // 644 = rw-r--r-- (owner baca+tulis, group+lain baca sahaja)
        // 755 = rwxr-xr-x (owner semua, group+lain baca+laksana)
        // 600 = rw------- (owner sahaja)

        // Semak permission
        println!("Owner readable:   {}", mod_bit & 0o400 != 0);
        println!("Owner writable:   {}", mod_bit & 0o200 != 0);
        println!("Owner executable: {}", mod_bit & 0o100 != 0);

        // Set permission
        let mut kebenaran = meta.permissions();
        kebenaran.set_mode(0o755);
        // fs::set_permissions(laluan, kebenaran)?;
    }

    // ── Set baca sahaja ───────────────────────────────────────
    let mut kebenaran = meta.permissions();
    kebenaran.set_readonly(true);
    // fs::set_permissions(laluan, kebenaran)?;

    Ok(())
}
```

---

## Symlink dan Hardlink Info

```rust
use std::fs;
use std::path::Path;

#[cfg(unix)]
fn main() -> std::io::Result<()> {
    use std::os::unix::fs::MetadataExt;
    use std::os::unix::fs::FileTypeExt;

    let laluan = Path::new("Cargo.toml");
    let meta = fs::metadata(laluan)?;
    let meta_symlink = fs::symlink_metadata(laluan)?; // tidak follow symlink!

    // Maklumat Unix-specific
    println!("inode:    {}", meta.ino());
    println!("nlinks:   {}", meta.nlink()); // bilangan hardlinks
    println!("uid:      {}", meta.uid());   // user ID
    println!("gid:      {}", meta.gid());   // group ID
    println!("blocks:   {}", meta.blocks()); // bilangan 512-byte blocks

    // Semak jenis fail
    let jenis = meta_symlink.file_type();
    println!("Adalah symlink: {}", jenis.is_symlink());
    println!("Adalah socket:  {}", jenis.is_socket());
    println!("Adalah FIFO:    {}", jenis.is_fifo());

    // Baca destinasi symlink
    if jenis.is_symlink() {
        let dest = fs::read_link(laluan)?;
        println!("Symlink → {}", dest.display());
    }

    Ok(())
}

#[cfg(not(unix))]
fn main() { println!("Unix sahaja untuk demo ini"); }
```

---

# BAHAGIAN 7: Fail Sementara 🗒️

## tempfile crate

```toml
[dependencies]
tempfile = "3"
```

```rust
use tempfile::{tempfile, NamedTempFile, TempDir};
use std::io::{Write, Read, Seek, SeekFrom};

fn main() -> std::io::Result<()> {
    // ── Fail sementara tanpa nama ─────────────────────────────
    // Auto-padam bila File di-drop!
    let mut fail_temp = tempfile()?;
    writeln!(fail_temp, "Data sementara")?;
    fail_temp.seek(SeekFrom::Start(0))?;

    let mut kandungan = String::new();
    fail_temp.read_to_string(&mut kandungan)?;
    println!("Sementara: {}", kandungan);
    // fail_temp di-drop di sini → auto-padam!

    // ── Named temporary file ──────────────────────────────────
    let mut fail_bernama = NamedTempFile::new()?;
    println!("Laluan: {}", fail_bernama.path().display());
    // Biasanya dalam /tmp/ (Linux) atau %TEMP% (Windows)

    writeln!(fail_bernama, "Data dengan nama")?;

    // Boleh "persist" (kekal) fail ini
    let laluan_kekal = std::path::PathBuf::from("data_kekal.txt");
    fail_bernama.persist(&laluan_kekal)?;
    println!("Dipersist ke: {}", laluan_kekal.display());
    std::fs::remove_file(&laluan_kekal)?;

    // ── Direktori sementara ───────────────────────────────────
    let dir_temp = TempDir::new()?;
    println!("Dir temp: {}", dir_temp.path().display());

    // Buat fail dalam direktori temp
    let fail_dalam = dir_temp.path().join("data.txt");
    std::fs::write(&fail_dalam, "Data dalam temp dir")?;

    // dir_temp di-drop → direktori dan semua kandungannya auto-padam!

    Ok(())
}
```

---

## 🧠 Brain Teaser #2

Mengapa menggunakan `BufWriter` adalah lebih baik dari menulis terus ke `File` untuk banyak operasi tulis kecil?

<details>
<summary>👀 Jawapan</summary>

```
Tanpa BufWriter (tulis terus ke File):
  Setiap writeln!() → system call ke OS!
  1000 writeln!() = 1000 system calls = LAMBAT!

  write() → syscall → kernel → disk
  write() → syscall → kernel → disk
  write() → syscall → kernel → disk
  ... × 1000

Dengan BufWriter:
  Tulis ke BUFFER dalam memory dulu.
  Bila buffer penuh (8KB default) → SATU syscall ke OS.
  1000 writeln!() ≈ 4-5 syscalls sahaja = LAJU!

  write() → buffer
  write() → buffer
  write() → buffer (penuh!)
             → syscall → kernel → disk  (sekali sahaja!)

Benchmark biasa:
  Tanpa buffer: ~100ms untuk 10,000 baris
  Dengan BufWriter: ~5ms untuk 10,000 baris (20× lebih laju!)

AWAS: Ingat flush() bila selesai!
  - BufWriter.flush() → paksa tulis semua buffer ke disk
  - BufWriter auto-flush bila di-drop, tapi mungkin lose data
    kalau program crash sebelum drop
  - Explicit flush() adalah lebih selamat untuk data kritikal
```
</details>

---

# BAHAGIAN 8: Fail CSV, JSON, TOML 📋

## CSV

```toml
[dependencies]
csv   = "1"
serde = { version = "1", features = ["derive"] }
```

```rust
use csv::{Reader, Writer};
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct Pekerja {
    no_pekerja: String,
    nama:       String,
    bahagian:   String,
    gaji:       f64,
    aktif:      bool,
}

fn tulis_csv(fail: &str, pekerja: &[Pekerja]) -> csv::Result<()> {
    let mut penulis = Writer::from_path(fail)?;

    for p in pekerja {
        penulis.serialize(p)?;
    }

    penulis.flush()?;
    println!("CSV ditulis: {}", fail);
    Ok(())
}

fn baca_csv(fail: &str) -> csv::Result<Vec<Pekerja>> {
    let mut pembaca = Reader::from_path(fail)?;
    let mut rekod = Vec::new();

    for hasil in pembaca.deserialize() {
        let pekerja: Pekerja = hasil?;
        rekod.push(pekerja);
    }

    Ok(rekod)
}

fn main() -> csv::Result<()> {
    let pekerja = vec![
        Pekerja { no_pekerja: "KADA001".into(), nama: "Ali Ahmad".into(),
                  bahagian: "ICT".into(), gaji: 4500.0, aktif: true },
        Pekerja { no_pekerja: "KADA002".into(), nama: "Siti Hawa".into(),
                  bahagian: "HR".into(), gaji: 3800.0, aktif: true },
    ];

    tulis_csv("pekerja.csv", &pekerja)?;

    let dibaca = baca_csv("pekerja.csv")?;
    for p in &dibaca {
        println!("{}: {} (RM{:.2})", p.no_pekerja, p.nama, p.gaji);
    }

    std::fs::remove_file("pekerja.csv")?;
    Ok(())
}
```

---

## JSON

```toml
[dependencies]
serde      = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
use serde::{Serialize, Deserialize};
use serde_json;
use std::fs;
use std::path::Path;

#[derive(Debug, Serialize, Deserialize)]
struct Konfigurasi {
    versi:   String,
    pekerja: Vec<PekerjaMudah>,
    tetapan: Tetapan,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
struct PekerjaMudah {
    nama:     String,
    bahagian: String,
}

#[derive(Debug, Serialize, Deserialize)]
struct Tetapan {
    debug: bool,
    port:  u16,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let konfig = Konfigurasi {
        versi: "2.0".into(),
        pekerja: vec![
            PekerjaMudah { nama: "Ali".into(), bahagian: "ICT".into() },
        ],
        tetapan: Tetapan { debug: false, port: 8080 },
    };

    // Tulis JSON ke fail
    let json = serde_json::to_string_pretty(&konfig)?;
    fs::write("konfig.json", &json)?;
    println!("JSON:\n{}", json);

    // Baca dari fail
    let kandungan = fs::read_to_string("konfig.json")?;
    let dibaca: Konfigurasi = serde_json::from_str(&kandungan)?;
    println!("Port: {}", dibaca.tetapan.port);

    // Baca terus dari fail (lebih efisien)
    let fail = fs::File::open("konfig.json")?;
    let dibaca2: Konfigurasi = serde_json::from_reader(fail)?;
    println!("Versi: {}", dibaca2.versi);

    // Tulis terus ke fail
    let fail = fs::File::create("konfig2.json")?;
    serde_json::to_writer_pretty(fail, &konfig)?;

    // Bersihkan
    fs::remove_file("konfig.json")?;
    fs::remove_file("konfig2.json")?;
    Ok(())
}
```

---

## TOML

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
toml  = "0.8"
```

```rust
use serde::{Serialize, Deserialize};
use std::fs;

#[derive(Debug, Serialize, Deserialize)]
struct AppConfig {
    aplikasi: AppSection,
    pangkalan: DbSection,
}

#[derive(Debug, Serialize, Deserialize)]
struct AppSection {
    nama:  String,
    port:  u16,
    debug: bool,
}

#[derive(Debug, Serialize, Deserialize)]
struct DbSection {
    url:     String,
    pool:    u32,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let toml_str = r#"
        [aplikasi]
        nama  = "KADA API"
        port  = 8080
        debug = false

        [pangkalan]
        url  = "mysql://localhost/kada_db"
        pool = 10
    "#;

    // Parse TOML
    let konfig: AppConfig = toml::from_str(toml_str)?;
    println!("{} pada :{}", konfig.aplikasi.nama, konfig.aplikasi.port);

    // Tulis TOML
    fs::write("konfig.toml", toml_str)?;
    let kandungan = fs::read_to_string("konfig.toml")?;
    let dibaca: AppConfig = toml::from_str(&kandungan)?;
    println!("DB: {}", dibaca.pangkalan.url);

    // Serialize ke TOML
    let toml_output = toml::to_string_pretty(&konfig)?;
    println!("TOML:\n{}", toml_output);

    fs::remove_file("konfig.toml")?;
    Ok(())
}
```

---

# BAHAGIAN 9: Async File I/O ⚡

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
use tokio::fs;
use tokio::io::{AsyncReadExt, AsyncWriteExt, AsyncBufReadExt, BufReader};

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // ── Baca fail async ───────────────────────────────────────
    let kandungan = fs::read_to_string("Cargo.toml").await?;
    println!("Panjang: {}", kandungan.len());

    let bytes = fs::read("Cargo.toml").await?;
    println!("Bytes: {}", bytes.len());

    // ── Tulis fail async ──────────────────────────────────────
    fs::write("output.txt", "Hello async!\n").await?;

    // ── Buka fail dan baca baris per baris async ──────────────
    let fail = fs::File::open("Cargo.toml").await?;
    let mut pembaca = BufReader::new(fail);
    let mut baris = String::new();

    loop {
        baris.clear();
        let n = pembaca.read_line(&mut baris).await?;
        if n == 0 { break; }
        print!("{}", baris);
    }

    // ── Async operasi fail lain ───────────────────────────────
    fs::copy("Cargo.toml", "Cargo.toml.bak").await?;
    fs::rename("Cargo.toml.bak", "bak.toml").await?;
    fs::remove_file("bak.toml").await?;
    fs::create_dir_all("temp/sub").await?;
    fs::remove_dir_all("temp").await?;

    // ── Serentak — baca banyak fail sekaligus! ────────────────
    let (r1, r2) = tokio::join!(
        fs::read_to_string("Cargo.toml"),
        fs::read_to_string("Cargo.lock"),
    );

    println!("Cargo.toml: {} bytes", r1.unwrap_or_default().len());
    println!("Cargo.lock: {} bytes", r2.unwrap_or_default().len());

    // ── Spawn concurrent tasks ────────────────────────────────
    let mut tugasan = Vec::new();
    for i in 0..5 {
        tugasan.push(tokio::spawn(async move {
            let nama = format!("fail_{}.txt", i);
            fs::write(&nama, format!("Kandungan fail {}", i)).await?;
            let kandungan = fs::read_to_string(&nama).await?;
            fs::remove_file(&nama).await?;
            Ok::<_, std::io::Error>(kandungan.len())
        }));
    }

    for (i, t) in tugasan.into_iter().enumerate() {
        println!("Fail {}: {} bytes", i, t.await.unwrap().unwrap());
    }

    fs::remove_file("output.txt").await?;
    Ok(())
}
```

---

# BAHAGIAN 10: Mini Project — Sistem Pengurusan Fail 📂

```toml
[dependencies]
walkdir   = "2"
chrono    = "0.4"
serde     = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
use std::fs;
use std::path::{Path, PathBuf};
use std::io;
use serde::{Serialize, Deserialize};
use std::collections::HashMap;
use walkdir::WalkDir;

// ─── Types ────────────────────────────────────────────────────

#[derive(Debug, Serialize, Deserialize, Clone)]
struct InfoFail {
    laluan:      String,
    nama:        String,
    sambungan:   Option<String>,
    saiz:        u64,
    ubah_pada:   u64,  // Unix timestamp
    adalah_dir:  bool,
}

#[derive(Debug, Serialize)]
struct LaporanDirektori {
    direktori:     String,
    jumlah_fail:   usize,
    jumlah_dir:    usize,
    jumlah_saiz:   u64,
    mengikut_ext:  HashMap<String, (usize, u64)>, // ext → (kiraan, saiz)
    fail_terbesar: Vec<InfoFail>,
    fail_terkini:  Vec<InfoFail>,
}

// ─── Fungsi Utama ─────────────────────────────────────────────

fn kumpul_info(direktori: &Path, max_depth: usize) -> io::Result<Vec<InfoFail>> {
    let mut senarai = Vec::new();

    for entri in WalkDir::new(direktori)
        .max_depth(max_depth)
        .into_iter()
        .filter_map(|e| e.ok())
    {
        let meta = entri.metadata()?;
        let laluan = entri.path();

        let ubah_pada = meta.modified()
            .unwrap_or(std::time::SystemTime::UNIX_EPOCH)
            .duration_since(std::time::SystemTime::UNIX_EPOCH)
            .unwrap_or_default()
            .as_secs();

        senarai.push(InfoFail {
            laluan:    laluan.to_string_lossy().to_string(),
            nama:      entri.file_name().to_string_lossy().to_string(),
            sambungan: laluan.extension().map(|e| e.to_string_lossy().to_string()),
            saiz:      if meta.is_file() { meta.len() } else { 0 },
            ubah_pada,
            adalah_dir: meta.is_dir(),
        });
    }

    Ok(senarai)
}

fn jana_laporan(direktori: &Path) -> io::Result<LaporanDirektori> {
    let semua = kumpul_info(direktori, usize::MAX)?;

    let mut mengikut_ext: HashMap<String, (usize, u64)> = HashMap::new();
    let mut fail_sahaja: Vec<&InfoFail> = Vec::new();

    let mut jumlah_fail = 0;
    let mut jumlah_dir = 0;
    let mut jumlah_saiz = 0u64;

    for info in &semua {
        if info.adalah_dir {
            jumlah_dir += 1;
        } else {
            jumlah_fail += 1;
            jumlah_saiz += info.saiz;
            fail_sahaja.push(info);

            let ext = info.sambungan.clone().unwrap_or_else(|| "(tiada)".into());
            let rekod = mengikut_ext.entry(ext).or_insert((0, 0));
            rekod.0 += 1;
            rekod.1 += info.saiz;
        }
    }

    // 5 fail terbesar
    let mut ikut_saiz = fail_sahaja.clone();
    ikut_saiz.sort_by(|a, b| b.saiz.cmp(&a.saiz));
    let fail_terbesar: Vec<InfoFail> = ikut_saiz.iter().take(5).map(|&f| f.clone()).collect();

    // 5 fail terkini
    let mut ikut_masa = fail_sahaja;
    ikut_masa.sort_by(|a, b| b.ubah_pada.cmp(&a.ubah_pada));
    let fail_terkini: Vec<InfoFail> = ikut_masa.iter().take(5).map(|&f| f.clone()).collect();

    Ok(LaporanDirektori {
        direktori:   direktori.to_string_lossy().to_string(),
        jumlah_fail,
        jumlah_dir,
        jumlah_saiz,
        mengikut_ext,
        fail_terbesar,
        fail_terkini,
    })
}

fn format_saiz(bait: u64) -> String {
    const KB: u64 = 1024;
    const MB: u64 = 1024 * KB;
    const GB: u64 = 1024 * MB;

    if bait >= GB      { format!("{:.2} GB", bait as f64 / GB as f64) }
    else if bait >= MB { format!("{:.2} MB", bait as f64 / MB as f64) }
    else if bait >= KB { format!("{:.2} KB", bait as f64 / KB as f64) }
    else               { format!("{} bytes", bait) }
}

fn cari_fail_duplikat(direktori: &Path) -> io::Result<HashMap<u64, Vec<String>>> {
    // Cari fail dengan saiz yang sama (mungkin duplikat)
    let mut ikut_saiz: HashMap<u64, Vec<String>> = HashMap::new();

    for entri in WalkDir::new(direktori)
        .into_iter()
        .filter_map(|e| e.ok())
        .filter(|e| e.file_type().is_file())
    {
        let saiz = entri.metadata()?.len();
        if saiz > 0 { // ignore fail kosong
            ikut_saiz
                .entry(saiz)
                .or_default()
                .push(entri.path().to_string_lossy().to_string());
        }
    }

    // Hanya simpan yang ada lebih dari 1 fail (mungkin duplikat)
    Ok(ikut_saiz.into_iter().filter(|(_, v)| v.len() > 1).collect())
}

fn bersih_fail_lama(direktori: &Path, hari: u64) -> io::Result<(usize, u64)> {
    let had_masa = std::time::SystemTime::now()
        .duration_since(std::time::SystemTime::UNIX_EPOCH)
        .unwrap()
        .as_secs()
        - (hari * 86400);

    let mut dipadam = 0;
    let mut saiz_dikitar = 0u64;

    for entri in WalkDir::new(direktori)
        .into_iter()
        .filter_map(|e| e.ok())
        .filter(|e| e.file_type().is_file())
    {
        let meta = entri.metadata()?;
        let ubah = meta.modified()
            .unwrap_or(std::time::SystemTime::UNIX_EPOCH)
            .duration_since(std::time::SystemTime::UNIX_EPOCH)
            .unwrap_or_default()
            .as_secs();

        if ubah < had_masa {
            println!("  Padam: {}", entri.path().display());
            saiz_dikitar += meta.len();
            // fs::remove_file(entri.path())?; // uncomment untuk padam betul
            dipadam += 1;
        }
    }

    Ok((dipadam, saiz_dikitar))
}

// ─── Main ──────────────────────────────────────────────────────

fn main() -> io::Result<()> {
    let direktori = Path::new(".");

    println!("{'═'*60}");
    println!("{:^60}", "SISTEM PENGURUSAN FAIL KADA");
    println!("{'═'*60}");

    // Jana laporan
    println!("\n📊 Jana laporan untuk: {}\n", direktori.display());

    let laporan = jana_laporan(direktori)?;

    println!("Ringkasan:");
    println!("  Jumlah fail:      {:>10}", laporan.jumlah_fail);
    println!("  Jumlah direktori: {:>10}", laporan.jumlah_dir);
    println!("  Jumlah saiz:      {:>10}", format_saiz(laporan.jumlah_saiz));

    // Mengikut sambungan
    println!("\nSambungan fail:");
    let mut ext_senarai: Vec<(&String, &(usize, u64))> = laporan.mengikut_ext.iter().collect();
    ext_senarai.sort_by(|a, b| b.1.1.cmp(&a.1.1)); // sort by saiz

    for (ext, (kiraan, saiz)) in ext_senarai.iter().take(10) {
        println!("  .{:<12} {:>5} fail  {:>12}",
            ext, kiraan, format_saiz(*saiz));
    }

    // Fail terbesar
    println!("\nFail terbesar:");
    for fail in &laporan.fail_terbesar {
        println!("  {:>12}  {}", format_saiz(fail.saiz), fail.laluan);
    }

    // Fail terkini
    println!("\nFail terkini diubah:");
    for fail in &laporan.fail_terkini {
        println!("  {}", fail.laluan);
    }

    // Simpan laporan
    let laporan_json = serde_json::to_string_pretty(&laporan)?;
    fs::write("laporan_fail.json", &laporan_json)?;
    println!("\n✔ Laporan disimpan: laporan_fail.json");

    // Cari kemungkinan duplikat
    println!("\nMencari kemungkinan duplikat...");
    let duplikat = cari_fail_duplikat(direktori)?;
    if duplikat.is_empty() {
        println!("  Tiada duplikat dijumpai");
    } else {
        for (saiz, fail_list) in duplikat.iter().take(5) {
            println!("  Saiz {} — {} fail mungkin duplikat:",
                format_saiz(*saiz), fail_list.len());
            for f in fail_list {
                println!("    {}", f);
            }
        }
    }

    // Bersihkan fail lama (demo — tidak padam sebenar)
    println!("\nSemak fail lama (>30 hari)...");
    let (kiraan, saiz) = bersih_fail_lama(direktori, 30)?;
    println!("  {} fail ({}) akan dipadam",
        kiraan, format_saiz(saiz));

    // Bersihkan
    fs::remove_file("laporan_fail.json")?;

    println!("\n{'═'*60}");
    println!("Selesai!");

    Ok(())
}
```

---

# 📋 Rujukan Pantas — File Operations Cheat Sheet

## Operasi Fail Asas

```rust
// Baca
fs::read_to_string("fail.txt")?      // → String
fs::read("fail.bin")?                 // → Vec<u8>

// Tulis (overwrite)
fs::write("fail.txt", "kandungan")?
fs::write("fail.bin", &bytes)?

// Append
OpenOptions::new().append(true).create(true).open("log.txt")?

// Salin
fs::copy("dari.txt", "ke.txt")?       // → u64 (bytes)

// Pindah/Rename
fs::rename("lama.txt", "baru.txt")?

// Padam
fs::remove_file("fail.txt")?

// Metadata
fs::metadata("fail.txt")?             // → Metadata
```

## Buka Fail dengan Pilihan

```rust
use std::fs::OpenOptions;

OpenOptions::new()
    .read(true)       // boleh baca
    .write(true)      // boleh tulis
    .create(true)     // buat kalau tiada
    .create_new(true) // buat BARU sahaja (gagal kalau ada)
    .append(true)     // tambah di hujung
    .truncate(true)   // kosongkan fail bila buka
    .open("fail.txt")?
```

## Path Operations

```rust
Path::new("dir/fail.txt")
PathBuf::from("dir/fail.txt")
path.join("subfail")           // gabung path
path.exists()                  // semak kewujudan
path.is_file()                 // adakah fail?
path.is_dir()                  // adakah direktori?
path.file_name()               // nama fail → Option<OsStr>
path.file_stem()               // nama tanpa ext → Option<OsStr>
path.extension()               // sambungan → Option<OsStr>
path.parent()                  // direktori ibu → Option<&Path>
path.with_extension("pdf")     // tukar sambungan
path.to_str()                  // → Option<&str>
path.display()                 // untuk println!
```

## Direktori

```rust
fs::create_dir("baru")?
fs::create_dir_all("a/b/c")?   // rekursif
fs::remove_dir("kosong")?
fs::remove_dir_all("penuh")?   // rekursif (padam semua!)
fs::read_dir(".")?             // → iterator DirEntry
```

## BufReader & BufWriter

```rust
// Baca baris per baris
let pembaca = BufReader::new(File::open("fail.txt")?);
for baris in pembaca.lines() {
    let baris = baris?; // String
}

// Tulis dengan buffer (lebih laju!)
let penulis = BufWriter::new(File::create("fail.txt")?);
writeln!(penulis, "Baris {}", n)?;
penulis.flush()?; // penting!
```

## Async (Tokio)

```rust
tokio::fs::read_to_string("fail.txt").await?
tokio::fs::write("fail.txt", data).await?
tokio::fs::copy("dari", "ke").await?
tokio::fs::rename("lama", "baru").await?
tokio::fs::remove_file("fail.txt").await?
tokio::fs::create_dir_all("a/b").await?
```

---

## 🏆 Cabaran Akhir

Cuba implement salah satu:

1. **Log Rotator** — rotate fail log bila saiz melebihi had, simpan N log lama
2. **File Synchronizer** — sync fail antara dua direktori (hanya tulis yang berubah)
3. **Fail Encryptor** — encrypt/decrypt fail dengan XOR atau AES
4. **CSV to JSON Converter** — baca CSV, tukar ke JSON, dengan progress bar
5. **Directory Watcher** — detect perubahan dalam direktori (fail baru, ubah, padam)

---

*File operations dalam Rust — selamat, cekap, dan expressive.*
*Drop menutup fail automatik. BufWriter mempercepatkan I/O.*
*Atomic write melindungi data anda dari corruption.* 🦀
