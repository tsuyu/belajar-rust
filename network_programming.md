# 🌐 Network Programming dalam Rust — Panduan Lengkap

> Dari TCP socket asas hingga HTTP server async yang production-ready.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Kenapa Rust untuk Network Programming?

```
Masalah bahasa lain:
  C/C++:   Cepat tapi buffer overflow, memory leaks, crashes
  Python:  Selamat tapi LAMBAT, GIL hadkan concurrency
  Java:    Selamat tapi berat, high memory usage
  Node.js: Async bagus tapi single-threaded, JS overhead

Rust penyelesaian:
  ✔ Kelajuan setara C (no GC, no runtime)
  ✔ Memory safety (no buffer overflow, no use-after-free)
  ✔ Async/await yang ergonomic (Tokio)
  ✔ Zero-cost abstractions
  ✔ Fearless concurrency (ownership prevent data races)

Contoh guna:
  Discord   → Guna Rust untuk menggantikan Go service
  Cloudflare → HTTP proxy dalam Rust
  AWS       → Firecracker VM (Lambda, Fargate)
  Dropbox   → Storage backend
  Mozilla   → Firefox networking stack
```

---

## Peta Pembelajaran

```
Bahagian 1  → TCP — Asas Networking
Bahagian 2  → UDP — Connectionless
Bahagian 3  → Async Networking dengan Tokio
Bahagian 4  → HTTP Client dengan reqwest
Bahagian 5  → HTTP Server dengan axum
Bahagian 6  → WebSocket
Bahagian 7  → DNS & Nama Host
Bahagian 8  → TLS / HTTPS
Bahagian 9  → gRPC
Bahagian 10 → Mini Project: KADA API Server
```

---

# BAHAGIAN 1: TCP — Asas Networking 🔌

## TCP Asas — Synchronous

```toml
# Cargo.toml — tiada dependency tambahan untuk std networking!
[package]
name = "tcp-asas"
version = "0.1.0"
edition = "2021"
```

```rust
// ── TCP Server ────────────────────────────────────────────────
use std::net::{TcpListener, TcpStream};
use std::io::{Read, Write, BufRead, BufReader};
use std::thread;

fn kendalikan_klien(mut strim: TcpStream) {
    let alamat = strim.peer_addr().unwrap();
    println!("Klien sambung: {}", alamat);

    let mut baca = BufReader::new(strim.try_clone().unwrap());
    let mut baris = String::new();

    loop {
        baris.clear();
        match baca.read_line(&mut baris) {
            Ok(0) => {
                println!("Klien {} putus", alamat);
                break;
            }
            Ok(_) => {
                let mesej = baris.trim();
                println!("Terima dari {}: {}", alamat, mesej);

                if mesej == "keluar" { break; }

                // Echo balik dengan uppercase
                let balas = format!("ECHO: {}\n", mesej.to_uppercase());
                strim.write_all(balas.as_bytes()).unwrap();
            }
            Err(e) => {
                eprintln!("Error baca: {}", e);
                break;
            }
        }
    }
}

fn jalankan_server_tcp() {
    let pendengar = TcpListener::bind("127.0.0.1:7878").unwrap();
    println!("Server TCP dengat pada 127.0.0.1:7878");

    for sambungan in pendengar.incoming() {
        match sambungan {
            Ok(strim) => {
                // Spawn thread untuk setiap klien
                thread::spawn(|| kendalikan_klien(strim));
            }
            Err(e) => eprintln!("Error sambungan: {}", e),
        }
    }
}
```

```rust
// ── TCP Client ────────────────────────────────────────────────
use std::net::TcpStream;
use std::io::{Write, BufRead, BufReader};

fn klien_tcp() {
    let mut strim = TcpStream::connect("127.0.0.1:7878").unwrap();
    println!("Sambung ke server!");

    let mesej_list = ["hello", "world", "rust networking", "keluar"];

    let mut pembaca = BufReader::new(strim.try_clone().unwrap());
    let mut balas = String::new();

    for mesej in &mesej_list {
        // Hantar mesej
        writeln!(strim, "{}", mesej).unwrap();
        println!("Hantar: {}", mesej);

        if *mesej == "keluar" { break; }

        // Terima balas
        balas.clear();
        pembaca.read_line(&mut balas).unwrap();
        println!("Balas: {}", balas.trim());
    }
}

fn main() {
    // Jalankan server dalam thread
    thread::spawn(|| jalankan_server_tcp());
    thread::sleep(std::time::Duration::from_millis(100));

    // Jalankan klien
    klien_tcp();
}
```

---

## TCP dengan Thread Pool

```rust
use std::net::{TcpListener, TcpStream};
use std::io::{Read, Write};
use std::sync::{Arc, Mutex};
use std::collections::VecDeque;
use std::thread;

struct KoleksiThread {
    pekerja: Vec<thread::JoinHandle<()>>,
    pengirim: Arc<Mutex<VecDeque<TcpStream>>>,
}

impl KoleksiThread {
    fn baru(saiz: usize) -> Self {
        let barisan: Arc<Mutex<VecDeque<TcpStream>>> =
            Arc::new(Mutex::new(VecDeque::new()));

        let mut pekerja = Vec::new();

        for id in 0..saiz {
            let barisan_klon = Arc::clone(&barisan);
            pekerja.push(thread::spawn(move || {
                loop {
                    let strim = {
                        let mut barisan = barisan_klon.lock().unwrap();
                        barisan.pop_front()
                    };

                    if let Some(mut s) = strim {
                        let mut buf = [0u8; 1024];
                        if let Ok(n) = s.read(&mut buf) {
                            let mesej = String::from_utf8_lossy(&buf[..n]);
                            println!("Worker {}: {}", id, mesej.trim());
                            s.write_all(b"HTTP/1.1 200 OK\r\n\r\nHello!\r\n").ok();
                        }
                    } else {
                        thread::sleep(std::time::Duration::from_millis(10));
                    }
                }
            }));
        }

        KoleksiThread { pekerja, pengirim: barisan }
    }

    fn tambah_kerja(&self, strim: TcpStream) {
        self.pengirim.lock().unwrap().push_back(strim);
    }
}
```

---

## 🧠 Brain Teaser #1

Apakah perbezaan utama antara TCP dan UDP?

<details>
<summary>👀 Jawapan</summary>

```
TCP (Transmission Control Protocol):
  ✔ Connection-oriented — handshake sebelum hantar data
  ✔ Reliable — data dijamin sampai dan dalam order
  ✔ Flow control — prevent sender overwhelm receiver
  ✔ Error checking — checksums + retransmission
  ✗ Lebih lambat (overhead handshake + acknowledgment)
  ✗ Lebih banyak header overhead
  Guna untuk: HTTP, HTTPS, email, SSH, database

UDP (User Datagram Protocol):
  ✔ Connectionless — hantar terus tanpa handshake
  ✔ Sangat LAJU — tiada overhead
  ✔ Packet kecil — minimal header
  ✗ Tidak reliable — packet boleh hilang atau out-of-order
  ✗ Tiada flow control
  Guna untuk: video streaming, game online, DNS, VoIP

Analogi:
  TCP = Pos berdaftar dengan resit (dijamin sampai)
  UDP = Surat biasa (mungkin hilang, tapi murah dan laju)
```
</details>

---

# BAHAGIAN 2: UDP — Connectionless 📡

## UDP Server dan Client

```rust
use std::net::UdpSocket;
use std::thread;

fn server_udp() {
    let soket = UdpSocket::bind("127.0.0.1:9999").unwrap();
    println!("UDP Server dengar pada :9999");

    let mut buf = [0u8; 65535];

    loop {
        match soket.recv_from(&mut buf) {
            Ok((saiz, dari)) => {
                let mesej = String::from_utf8_lossy(&buf[..saiz]);
                println!("UDP dari {}: {}", dari, mesej.trim());

                // Balas kepada pengirim
                let balas = format!("ACK: {}", mesej.trim());
                soket.send_to(balas.as_bytes(), dari).unwrap();
            }
            Err(e) => eprintln!("Error UDP: {}", e),
        }
    }
}

fn klien_udp() {
    let soket = UdpSocket::bind("127.0.0.1:0").unwrap(); // port rawak
    soket.connect("127.0.0.1:9999").unwrap();

    let mesej_list = ["hello", "udp", "no connection needed"];

    let mut buf = [0u8; 65535];

    for mesej in &mesej_list {
        soket.send(mesej.as_bytes()).unwrap();
        println!("Hantar UDP: {}", mesej);

        // Terima balas (dengan timeout!)
        soket.set_read_timeout(Some(
            std::time::Duration::from_secs(2)
        )).unwrap();

        match soket.recv(&mut buf) {
            Ok(n) => println!("Balas: {}", String::from_utf8_lossy(&buf[..n])),
            Err(_) => println!("Timeout — tiada balas"),
        }
    }
}

fn main() {
    thread::spawn(|| server_udp());
    thread::sleep(std::time::Duration::from_millis(100));
    klien_udp();
}
```

---

## UDP Broadcast & Multicast

```rust
use std::net::UdpSocket;

fn demo_broadcast() {
    // Broadcast — hantar ke semua dalam network
    let soket = UdpSocket::bind("0.0.0.0:0").unwrap();
    soket.set_broadcast(true).unwrap();

    let mesej = b"Siapa ada di sini?";
    soket.send_to(mesej, "255.255.255.255:9999").unwrap();
    println!("Broadcast dihantar!");
}

fn demo_terima_broadcast() {
    let soket = UdpSocket::bind("0.0.0.0:9999").unwrap();
    soket.set_broadcast(true).unwrap();

    let mut buf = [0u8; 1024];
    println!("Dengar broadcast...");

    let (n, dari) = soket.recv_from(&mut buf).unwrap();
    println!("Broadcast dari {}: {}",
        dari,
        String::from_utf8_lossy(&buf[..n])
    );
}
```

---

# BAHAGIAN 3: Async Networking dengan Tokio ⚡

## Setup Tokio

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

## Async TCP Server

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{AsyncReadExt, AsyncWriteExt, AsyncBufReadExt, BufReader};

#[tokio::main]
async fn main() {
    let pendengar = TcpListener::bind("127.0.0.1:8080").await.unwrap();
    println!("Async TCP Server pada :8080");

    loop {
        let (strim, alamat) = pendengar.accept().await.unwrap();
        println!("Sambungan baru: {}", alamat);

        // Spawn task async — tidak block!
        tokio::spawn(async move {
            kendalikan_async(strim).await;
        });
    }
}

async fn kendalikan_async(strim: TcpStream) {
    let (pembaca, mut penulis) = strim.into_split();
    let mut buf_pembaca = BufReader::new(pembaca);
    let mut baris = String::new();

    loop {
        baris.clear();
        match buf_pembaca.read_line(&mut baris).await {
            Ok(0) => break, // sambungan ditutup
            Ok(_) => {
                let mesej = baris.trim().to_string();
                println!("Terima: {}", mesej);
                let balas = format!("ECHO: {}\n", mesej);
                penulis.write_all(balas.as_bytes()).await.unwrap();
            }
            Err(e) => { eprintln!("Error: {}", e); break; }
        }
    }
}
```

---

## Async UDP dengan Tokio

```rust
use tokio::net::UdpSocket;

#[tokio::main]
async fn main() {
    // Server
    let server = tokio::spawn(async {
        let soket = UdpSocket::bind("127.0.0.1:9000").await.unwrap();
        println!("Async UDP Server pada :9000");

        let mut buf = [0u8; 1024];
        loop {
            let (n, dari) = soket.recv_from(&mut buf).await.unwrap();
            let mesej = String::from_utf8_lossy(&buf[..n]);
            println!("UDP dari {}: {}", dari, mesej);
            soket.send_to(b"OK", dari).await.unwrap();
        }
    });

    // Client
    let klien = tokio::spawn(async {
        tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;

        let soket = UdpSocket::bind("0.0.0.0:0").await.unwrap();
        soket.send_to(b"Hello UDP Async!", "127.0.0.1:9000").await.unwrap();

        let mut buf = [0u8; 1024];
        let (n, _) = soket.recv_from(&mut buf).await.unwrap();
        println!("Balas: {}", String::from_utf8_lossy(&buf[..n]));
    });

    let (_, _) = tokio::join!(server, klien);
}
```

---

## Timeout dan Retry

```rust
use tokio::net::TcpStream;
use tokio::time::{timeout, Duration};
use std::io;

async fn sambung_dengan_timeout(
    alamat: &str,
    masa_tunggu: Duration,
) -> io::Result<TcpStream> {
    timeout(masa_tunggu, TcpStream::connect(alamat))
        .await
        .map_err(|_| io::Error::new(io::ErrorKind::TimedOut, "Sambungan timeout"))?
}

async fn sambung_dengan_retry(
    alamat: &str,
    max_cuba: u32,
) -> io::Result<TcpStream> {
    let mut cuba = 0;
    loop {
        match sambung_dengan_timeout(alamat, Duration::from_secs(5)).await {
            Ok(strim) => {
                println!("Berjaya sambung selepas {} percubaan!", cuba + 1);
                return Ok(strim);
            }
            Err(e) if cuba < max_cuba => {
                cuba += 1;
                eprintln!("Percubaan {} gagal: {}. Cuba lagi...", cuba, e);
                tokio::time::sleep(Duration::from_millis(1000 * cuba as u64)).await;
            }
            Err(e) => return Err(e),
        }
    }
}

#[tokio::main]
async fn main() {
    match sambung_dengan_retry("127.0.0.1:8080", 3).await {
        Ok(_strim) => println!("Sambungan berjaya!"),
        Err(e)    => eprintln!("Gagal sambung: {}", e),
    }
}
```

---

# BAHAGIAN 4: HTTP Client dengan reqwest 📥

```toml
[dependencies]
tokio   = { version = "1", features = ["full"] }
reqwest = { version = "0.11", features = ["json"] }
serde   = { version = "1", features = ["derive"] }
serde_json = "1"
```

## GET Request

```rust
use reqwest;
use serde::{Serialize, Deserialize};

#[derive(Debug, Deserialize)]
struct PostAPI {
    id:     u32,
    title:  String,
    body:   String,
    userId: u32,
}

#[tokio::main]
async fn main() -> Result<(), reqwest::Error> {
    let klien = reqwest::Client::new();

    // ── GET Request Mudah ─────────────────────────────────────
    let balas = klien
        .get("https://jsonplaceholder.typicode.com/posts/1")
        .send()
        .await?;

    println!("Status: {}", balas.status());
    println!("Headers: {:?}", balas.headers());

    // Parse sebagai JSON
    let post: PostAPI = balas.json().await?;
    println!("{:#?}", post);

    // ── GET dengan query parameters ───────────────────────────
    let posts: Vec<PostAPI> = klien
        .get("https://jsonplaceholder.typicode.com/posts")
        .query(&[("userId", "1"), ("_limit", "3")])
        .send()
        .await?
        .json()
        .await?;

    println!("Dapat {} posts", posts.len());

    // ── GET text ──────────────────────────────────────────────
    let html = reqwest::get("https://httpbin.org/html")
        .await?
        .text()
        .await?;

    println!("HTML panjang: {} bytes", html.len());

    // ── GET bytes ─────────────────────────────────────────────
    let gambar = reqwest::get("https://httpbin.org/image/png")
        .await?
        .bytes()
        .await?;

    println!("Gambar saiz: {} bytes", gambar.len());

    Ok(())
}
```

---

## POST, Headers, Auth

```rust
use reqwest::{Client, header};
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize)]
struct BuatPost {
    title:  String,
    body:   String,
    userId: u32,
}

#[derive(Debug, Deserialize)]
struct BalasPost {
    id:     u32,
    title:  String,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // ── Bina klien dengan header default ─────────────────────
    let mut headers_lalai = header::HeaderMap::new();
    headers_lalai.insert(
        header::USER_AGENT,
        header::HeaderValue::from_static("KADA-App/1.0"),
    );

    let klien = Client::builder()
        .default_headers(headers_lalai)
        .timeout(std::time::Duration::from_secs(30))
        .build()?;

    // ── POST JSON ─────────────────────────────────────────────
    let data_baru = BuatPost {
        title:  "Hello dari Rust".into(),
        body:   "Rust networking adalah hebat!".into(),
        userId: 1,
    };

    let balas: BalasPost = klien
        .post("https://jsonplaceholder.typicode.com/posts")
        .json(&data_baru)
        .send()
        .await?
        .json()
        .await?;

    println!("Post dicipta dengan ID: {}", balas.id);

    // ── POST dengan form data ─────────────────────────────────
    let _balas = klien
        .post("https://httpbin.org/post")
        .form(&[("nama", "Ali"), ("bahagian", "ICT")])
        .send()
        .await?;

    // ── Request dengan Authorization ──────────────────────────
    let _balas = klien
        .get("https://api.example.com/protected")
        .bearer_auth("token_jwt_anda_di_sini")
        .send()
        .await?;

    // ── PUT Request ───────────────────────────────────────────
    let _kemaskini = klien
        .put("https://jsonplaceholder.typicode.com/posts/1")
        .json(&data_baru)
        .send()
        .await?;

    // ── DELETE Request ────────────────────────────────────────
    let status = klien
        .delete("https://jsonplaceholder.typicode.com/posts/1")
        .send()
        .await?
        .status();

    println!("Delete status: {}", status);

    Ok(())
}
```

---

## Muat Turun Fail

```rust
use reqwest::Client;
use tokio::fs::File;
use tokio::io::AsyncWriteExt;
use futures_util::StreamExt;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let klien = Client::new();
    let url = "https://httpbin.org/bytes/1048576"; // 1MB

    let mut balas = klien.get(url).send().await?;

    let jumlah = balas.content_length().unwrap_or(0);
    println!("Muat turun {} bytes...", jumlah);

    let mut fail = File::create("muat_turun.bin").await?;
    let mut muat_turun: u64 = 0;

    // Streaming download — tidak load semua ke memory!
    while let Some(chunk) = balas.chunk().await? {
        fail.write_all(&chunk).await?;
        muat_turun += chunk.len() as u64;

        if jumlah > 0 {
            print!("\rProgress: {:.1}%", muat_turun as f64 / jumlah as f64 * 100.0);
        }
    }

    println!("\nMuat turun selesai: {} bytes", muat_turun);
    tokio::fs::remove_file("muat_turun.bin").await?;
    Ok(())
}
```

---

# BAHAGIAN 5: HTTP Server dengan axum 🖥️

```toml
[dependencies]
tokio  = { version = "1", features = ["full"] }
axum   = "0.7"
serde  = { version = "1", features = ["derive"] }
serde_json = "1"
tower  = "0.4"
tower-http = { version = "0.5", features = ["cors", "trace"] }
```

## Server Asas

```rust
use axum::{
    Router,
    routing::{get, post, put, delete},
    extract::{Path, State, Json, Query},
    http::StatusCode,
    response::IntoResponse,
};
use serde::{Serialize, Deserialize};
use std::sync::{Arc, Mutex};
use std::collections::HashMap;

// ── Data types ────────────────────────────────────────────────

#[derive(Debug, Serialize, Deserialize, Clone)]
struct Pekerja {
    id:       u32,
    nama:     String,
    bahagian: String,
    gaji:     f64,
}

#[derive(Debug, Deserialize)]
struct BuatPekerja {
    nama:     String,
    bahagian: String,
    gaji:     f64,
}

#[derive(Debug, Deserialize)]
struct QuerySaring {
    bahagian: Option<String>,
    had:      Option<usize>,
}

type Db = Arc<Mutex<HashMap<u32, Pekerja>>>;

// ── Handlers ──────────────────────────────────────────────────

async fn halaman_utama() -> impl IntoResponse {
    (StatusCode::OK, "KADA Mobile API v2.0")
}

async fn senarai_pekerja(
    State(db): State<Db>,
    Query(saring): Query<QuerySaring>,
) -> impl IntoResponse {
    let db = db.lock().unwrap();
    let mut senarai: Vec<&Pekerja> = db.values().collect();

    if let Some(bah) = &saring.bahagian {
        senarai.retain(|p| p.bahagian.to_lowercase() == bah.to_lowercase());
    }

    if let Some(had) = saring.had {
        senarai.truncate(had);
    }

    let hasil: Vec<Pekerja> = senarai.into_iter().cloned().collect();
    (StatusCode::OK, Json(hasil))
}

async fn dapatkan_pekerja(
    State(db): State<Db>,
    Path(id): Path<u32>,
) -> impl IntoResponse {
    let db = db.lock().unwrap();
    match db.get(&id) {
        Some(p) => (StatusCode::OK, Json(Some(p.clone()))).into_response(),
        None    => StatusCode::NOT_FOUND.into_response(),
    }
}

async fn buat_pekerja(
    State(db): State<Db>,
    Json(baru): Json<BuatPekerja>,
) -> impl IntoResponse {
    let mut db = db.lock().unwrap();
    let id = db.keys().max().copied().unwrap_or(0) + 1;

    let pekerja = Pekerja {
        id,
        nama:     baru.nama,
        bahagian: baru.bahagian,
        gaji:     baru.gaji,
    };

    db.insert(id, pekerja.clone());
    (StatusCode::CREATED, Json(pekerja))
}

async fn padam_pekerja(
    State(db): State<Db>,
    Path(id): Path<u32>,
) -> impl IntoResponse {
    let mut db = db.lock().unwrap();
    match db.remove(&id) {
        Some(_) => StatusCode::NO_CONTENT,
        None    => StatusCode::NOT_FOUND,
    }
}

// ── Main ──────────────────────────────────────────────────────

#[tokio::main]
async fn main() {
    let db: Db = Arc::new(Mutex::new(HashMap::new()));

    // Isi data awal
    {
        let mut d = db.lock().unwrap();
        d.insert(1, Pekerja { id: 1, nama: "Ali Ahmad".into(), bahagian: "ICT".into(), gaji: 4500.0 });
        d.insert(2, Pekerja { id: 2, nama: "Siti Hawa".into(), bahagian: "HR".into(), gaji: 3800.0 });
        d.insert(3, Pekerja { id: 3, nama: "Amin Razak".into(), bahagian: "ICT".into(), gaji: 5000.0 });
    }

    let app = Router::new()
        .route("/",              get(halaman_utama))
        .route("/pekerja",       get(senarai_pekerja).post(buat_pekerja))
        .route("/pekerja/:id",   get(dapatkan_pekerja).delete(padam_pekerja))
        .with_state(db);

    let pendengar = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("Server berjalan pada http://localhost:3000");
    axum::serve(pendengar, app).await.unwrap();
}
```

---

## Middleware dan Error Handling

```rust
use axum::{
    middleware::{self, Next},
    extract::Request,
    response::Response,
    http::HeaderMap,
};
use tower_http::cors::CorsLayer;
use tower_http::trace::TraceLayer;

// ── Auth Middleware ───────────────────────────────────────────
async fn semak_auth(
    headers: HeaderMap,
    req: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let token = headers
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.strip_prefix("Bearer "));

    match token {
        Some("token_sah") => Ok(next.run(req).await),
        _ => Err(StatusCode::UNAUTHORIZED),
    }
}

// ── Custom Error Response ─────────────────────────────────────
#[derive(Serialize)]
struct MesejRalat {
    ralat: String,
    kod:   u16,
}

async fn pengendalian_404() -> impl IntoResponse {
    let ralat = MesejRalat {
        ralat: "Halaman tidak dijumpai".into(),
        kod:   404,
    };
    (StatusCode::NOT_FOUND, Json(ralat))
}

// ── App dengan middleware ─────────────────────────────────────
fn bina_app(db: Db) -> Router {
    let laluan_awam = Router::new()
        .route("/", get(halaman_utama))
        .route("/sihat", get(|| async { "OK" }));

    let laluan_lindung = Router::new()
        .route("/pekerja", get(senarai_pekerja).post(buat_pekerja))
        .route("/pekerja/:id", get(dapatkan_pekerja).delete(padam_pekerja))
        .layer(middleware::from_fn(semak_auth));

    Router::new()
        .merge(laluan_awam)
        .merge(laluan_lindung)
        .fallback(pengendalian_404)
        .layer(CorsLayer::permissive())
        .layer(TraceLayer::new_for_http())
        .with_state(db)
}
```

---

# BAHAGIAN 6: WebSocket 🔄

```toml
[dependencies]
tokio      = { version = "1", features = ["full"] }
axum       = { version = "0.7", features = ["ws"] }
futures    = "0.3"
```

```rust
use axum::{
    Router,
    routing::get,
    extract::ws::{WebSocket, WebSocketUpgrade, Message},
    response::IntoResponse,
};
use futures::{StreamExt, SinkExt};

async fn ws_pengendalian(ws: WebSocketUpgrade) -> impl IntoResponse {
    ws.on_upgrade(|soket| kendalikan_ws(soket))
}

async fn kendalikan_ws(mut soket: WebSocket) {
    println!("WebSocket klien sambung!");

    // Hantar mesej selamat datang
    soket.send(Message::Text("Selamat datang ke KADA WebSocket!".into()))
        .await
        .ok();

    // Loop terima dan balas mesej
    while let Some(mesej) = soket.recv().await {
        match mesej {
            Ok(Message::Text(teks)) => {
                println!("WS terima: {}", teks);
                let balas = format!("Echo: {}", teks);
                if soket.send(Message::Text(balas)).await.is_err() {
                    break; // klien putus
                }
            }
            Ok(Message::Binary(data)) => {
                println!("Binary: {} bytes", data.len());
                soket.send(Message::Binary(data)).await.ok();
            }
            Ok(Message::Close(_)) | Err(_) => {
                println!("WS klien putus");
                break;
            }
            _ => {}
        }
    }
}

// ── WebSocket dengan broadcast ────────────────────────────────
use std::sync::Arc;
use tokio::sync::broadcast;

struct AppState {
    tx: broadcast::Sender<String>,
}

async fn ws_siaran(
    ws: WebSocketUpgrade,
    axum::extract::State(state): axum::extract::State<Arc<AppState>>,
) -> impl IntoResponse {
    ws.on_upgrade(|soket| kendalikan_siaran(soket, state))
}

async fn kendalikan_siaran(soket: WebSocket, state: Arc<AppState>) {
    let (mut pengirim, mut penerima) = soket.split();
    let mut rx = state.tx.subscribe();

    // Task: terima broadcast dan hantar ke klien
    let mut hantar_task = tokio::spawn(async move {
        while let Ok(mesej) = rx.recv().await {
            if pengirim.send(Message::Text(mesej)).await.is_err() {
                break;
            }
        }
    });

    // Task: terima dari klien dan broadcast
    let tx = state.tx.clone();
    let mut terima_task = tokio::spawn(async move {
        while let Some(Ok(Message::Text(teks))) = penerima.next().await {
            tx.send(teks).ok();
        }
    });

    // Tunggu mana-mana task selesai
    tokio::select! {
        _ = &mut hantar_task  => terima_task.abort(),
        _ = &mut terima_task => hantar_task.abort(),
    }
}

#[tokio::main]
async fn main() {
    let (tx, _rx) = broadcast::channel(100);
    let state = Arc::new(AppState { tx });

    let app = Router::new()
        .route("/ws", get(ws_pengendalian))
        .route("/siaran", get(ws_siaran))
        .with_state(state);

    let pendengar = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("WebSocket server pada ws://localhost:3000/ws");
    axum::serve(pendengar, app).await.unwrap();
}
```

---

# BAHAGIAN 7: DNS & Nama Host 🔍

```rust
use std::net::{ToSocketAddrs, SocketAddr};
use tokio::net::lookup_host;

fn main() {
    // ── Resolve hostname (sync) ───────────────────────────────

    // ToSocketAddrs — resolve nama host ke IP
    let alamat: Vec<SocketAddr> = "google.com:80"
        .to_socket_addrs()
        .unwrap()
        .collect();

    for a in &alamat {
        println!("google.com → {}", a);
    }

    // Guna IP terus
    let terus: Vec<SocketAddr> = "93.184.216.34:80"
        .to_socket_addrs()
        .unwrap()
        .collect();

    println!("IP terus: {:?}", terus);
}

// ── Async DNS lookup ──────────────────────────────────────────
#[tokio::main]
async fn dns_async() {
    let alamat: Vec<SocketAddr> = lookup_host("rust-lang.org:443")
        .await
        .unwrap()
        .collect();

    println!("rust-lang.org:");
    for a in &alamat {
        println!("  {}", a);
    }
}

// ── Dapatkan hostname tempatan ────────────────────────────────
fn info_host() {
    use std::net::IpAddr;

    // Dapatkan semua interface IP
    // (perlu crate 'local-ip-address' atau 'if-addrs')

    // Cara mudah dengan socket trick
    let soket = std::net::UdpSocket::bind("0.0.0.0:0").unwrap();
    soket.connect("8.8.8.8:80").unwrap();
    let ip_tempatan = soket.local_addr().unwrap().ip();
    println!("IP tempatan: {}", ip_tempatan);
}
```

---

# BAHAGIAN 8: TLS / HTTPS 🔐

```toml
[dependencies]
tokio       = { version = "1", features = ["full"] }
tokio-rustls = "0.24"
rustls      = "0.21"
reqwest     = { version = "0.11", features = ["json", "rustls-tls"] }
axum        = "0.7"
axum-server = { version = "0.5", features = ["tls-rustls"] }
```

## HTTPS Client

```rust
use reqwest::Client;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // reqwest guna HTTPS secara automatik!
    // (pastikan URL bermula dengan https://)

    let klien = Client::builder()
        .use_rustls_tls() // guna rustls untuk TLS
        .build()?;

    let balas = klien
        .get("https://httpbin.org/json")
        .send()
        .await?;

    println!("Status: {}", balas.status());

    let data: serde_json::Value = balas.json().await?;
    println!("{:#?}", data);

    // Untuk self-signed cert (development sahaja!)
    let klien_dev = Client::builder()
        .danger_accept_invalid_certs(true) // JANGAN di production!
        .build()?;

    Ok(())
}
```

## HTTPS Server dengan Self-Signed Cert

```bash
# Jana self-signed certificate untuk development
openssl req -x509 -newkey rsa:4096 \
    -keyout kunci.pem -out cert.pem \
    -days 365 -nodes \
    -subj "/CN=localhost"
```

```rust
use axum_server::tls_rustls::RustlsConfig;
use axum::{Router, routing::get};

#[tokio::main]
async fn main() {
    let konfig = RustlsConfig::from_pem_file("cert.pem", "kunci.pem")
        .await
        .unwrap();

    let app = Router::new()
        .route("/", get(|| async { "HTTPS Server!" }));

    let alamat = std::net::SocketAddr::from(([127, 0, 0, 1], 443));
    println!("HTTPS server pada https://localhost");

    axum_server::bind_rustls(alamat, konfig)
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

---

# BAHAGIAN 9: gRPC 📞

```toml
[dependencies]
tonic      = "0.10"
prost      = "0.12"
tokio      = { version = "1", features = ["full"] }

[build-dependencies]
tonic-build = "0.10"
```

```protobuf
// proto/pekerja.proto
syntax = "proto3";
package pekerja;

message DapatkanPekerjaPermintaan {
    uint32 id = 1;
}

message PekerjaMaklumat {
    uint32 id     = 1;
    string nama   = 2;
    string jabatan = 3;
    double gaji   = 4;
}

service SyarikatPerkhidmatan {
    rpc DapatkanPekerja (DapatkanPekerjaPermintaan) returns (PekerjaMaklumat);
}
```

```rust
// build.rs
fn main() {
    tonic_build::compile_protos("proto/pekerja.proto").unwrap();
}
```

```rust
// Server gRPC
use tonic::{transport::Server, Request, Response, Status};

pub mod pekerja {
    tonic::include_proto!("pekerja");
}

use pekerja::syarikat_perkhidmatan_server::{SyarikatPerkhidmatan, SyarikatPerkhidmatanServer};
use pekerja::{DapatkanPekerjaPermintaan, PekerjaMaklumat};

#[derive(Default)]
pub struct SyarikatPerkhidmatanImpl;

#[tonic::async_trait]
impl SyarikatPerkhidmatan for SyarikatPerkhidmatanImpl {
    async fn dapatkan_pekerja(
        &self,
        permintaan: Request<DapatkanPekerjaPermintaan>,
    ) -> Result<Response<PekerjaMaklumat>, Status> {
        let id = permintaan.into_inner().id;

        // Simulate DB lookup
        if id == 1 {
            Ok(Response::new(PekerjaMaklumat {
                id,
                nama:    "Ali Ahmad".into(),
                jabatan: "ICT".into(),
                gaji:    4500.0,
            }))
        } else {
            Err(Status::not_found(format!("Pekerja {} tidak dijumpai", id)))
        }
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let alamat = "[::1]:50051".parse()?;
    let perkhidmatan = SyarikatPerkhidmatanImpl::default();

    println!("gRPC Server pada {}", alamat);

    Server::builder()
        .add_service(SyarikatPerkhidmatanServer::new(perkhidmatan))
        .serve(alamat)
        .await?;

    Ok(())
}
```

---

# BAHAGIAN 10: Mini Project — KADA API Server 🏗️

```toml
[dependencies]
tokio       = { version = "1", features = ["full"] }
axum        = "0.7"
serde       = { version = "1", features = ["derive"] }
serde_json  = "1"
tower-http  = { version = "0.5", features = ["cors", "trace"] }
reqwest     = { version = "0.11", features = ["json"] }
uuid        = { version = "1", features = ["v4"] }
chrono      = { version = "0.4", features = ["serde"] }
tokio-rustls = "0.24"
thiserror   = "1"
```

```rust
use axum::{
    Router,
    routing::{get, post},
    extract::{State, Json, Path, Query},
    http::StatusCode,
    response::IntoResponse,
    middleware::{self, Next},
    extract::Request,
};
use serde::{Serialize, Deserialize};
use std::sync::Arc;
use tokio::sync::RwLock;
use std::collections::HashMap;
use chrono::Utc;
use uuid::Uuid;

// ─── Types ────────────────────────────────────────────────────

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Pekerja {
    id:           String,
    no_pekerja:   String,
    nama:         String,
    bahagian:     String,
    aktif:        bool,
    dibuat_pada:  String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
struct RekodKehadiran {
    id:           String,
    id_pekerja:   String,
    masa_masuk:   String,
    masa_keluar:  Option<String>,
    koordinat:    Koordinat,
    kaedah:       String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Koordinat {
    lat: f64,
    lon: f64,
}

#[derive(Debug, Deserialize)]
struct BuatPekerja {
    no_pekerja: String,
    nama:       String,
    bahagian:   String,
}

#[derive(Debug, Deserialize)]
struct DaftarMasuk {
    id_pekerja: String,
    koordinat:  Koordinat,
    kaedah:     String,  // "gps", "selfie", "manual"
}

#[derive(Debug, Serialize)]
struct ApiResponse<T: Serialize> {
    berjaya:    bool,
    data:       Option<T>,
    mesej:      Option<String>,
}

impl<T: Serialize> ApiResponse<T> {
    fn ok(data: T) -> Self {
        ApiResponse { berjaya: true, data: Some(data), mesej: None }
    }
    fn mesej(mesej: &str) -> ApiResponse<()> {
        ApiResponse { berjaya: true, data: None, mesej: Some(mesej.into()) }
    }
}

impl ApiResponse<()> {
    fn ralat(mesej: &str) -> Self {
        ApiResponse { berjaya: false, data: None, mesej: Some(mesej.into()) }
    }
}

// ─── App State ────────────────────────────────────────────────

#[derive(Default, Clone)]
struct AppState {
    pekerja:   Arc<RwLock<HashMap<String, Pekerja>>>,
    kehadiran: Arc<RwLock<Vec<RekodKehadiran>>>,
}

impl AppState {
    fn baru() -> Self {
        let mut state = AppState::default();

        // Data awal
        let pekerja_awal = vec![
            Pekerja {
                id: "p001".into(), no_pekerja: "KADA001".into(),
                nama: "Ali Ahmad".into(), bahagian: "ICT".into(),
                aktif: true, dibuat_pada: Utc::now().to_rfc3339(),
            },
            Pekerja {
                id: "p002".into(), no_pekerja: "KADA002".into(),
                nama: "Siti Hawa".into(), bahagian: "HR".into(),
                aktif: true, dibuat_pada: Utc::now().to_rfc3339(),
            },
        ];

        let mut peta = HashMap::new();
        for p in pekerja_awal {
            peta.insert(p.id.clone(), p);
        }
        *state.pekerja.try_write().unwrap() = peta;
        state
    }
}

// ─── Middleware ────────────────────────────────────────────────

async fn log_permintaan(req: Request, next: Next) -> impl IntoResponse {
    let kaedah = req.method().clone();
    let uri    = req.uri().clone();
    let masa   = Utc::now().format("%H:%M:%S").to_string();

    println!("[{}] {} {}", masa, kaedah, uri);
    next.run(req).await
}

// ─── Handlers ─────────────────────────────────────────────────

async fn sihat() -> impl IntoResponse {
    Json(serde_json::json!({
        "status": "OK",
        "nama":   "KADA Mobile API",
        "versi":  "2.1.0",
        "masa":   Utc::now().to_rfc3339(),
    }))
}

async fn senarai_pekerja(
    State(state): State<AppState>,
) -> impl IntoResponse {
    let pekerja = state.pekerja.read().await;
    let mut senarai: Vec<Pekerja> = pekerja.values().cloned().collect();
    senarai.sort_by(|a, b| a.nama.cmp(&b.nama));
    Json(ApiResponse::ok(senarai))
}

async fn dapatkan_pekerja(
    State(state): State<AppState>,
    Path(id): Path<String>,
) -> impl IntoResponse {
    let pekerja = state.pekerja.read().await;
    match pekerja.get(&id) {
        Some(p) => (StatusCode::OK, Json(ApiResponse::ok(p.clone()))).into_response(),
        None    => (StatusCode::NOT_FOUND,
                    Json(ApiResponse::<()>::ralat("Pekerja tidak dijumpai"))).into_response(),
    }
}

async fn buat_pekerja(
    State(state): State<AppState>,
    Json(baru): Json<BuatPekerja>,
) -> impl IntoResponse {
    if baru.nama.trim().is_empty() {
        return (StatusCode::BAD_REQUEST,
                Json(ApiResponse::<()>::ralat("Nama tidak boleh kosong"))).into_response();
    }

    let id = Uuid::new_v4().to_string();
    let pekerja = Pekerja {
        id:          id.clone(),
        no_pekerja:  baru.no_pekerja,
        nama:        baru.nama,
        bahagian:    baru.bahagian,
        aktif:       true,
        dibuat_pada: Utc::now().to_rfc3339(),
    };

    state.pekerja.write().await.insert(id, pekerja.clone());
    println!("  → Pekerja baru: {}", pekerja.nama);

    (StatusCode::CREATED, Json(ApiResponse::ok(pekerja))).into_response()
}

async fn daftar_masuk(
    State(state): State<AppState>,
    Json(data): Json<DaftarMasuk>,
) -> impl IntoResponse {
    // Semak pekerja wujud
    let pekerja = state.pekerja.read().await;
    if !pekerja.contains_key(&data.id_pekerja) {
        return (StatusCode::NOT_FOUND,
                Json(ApiResponse::<()>::ralat("Pekerja tidak dijumpai"))).into_response();
    }
    drop(pekerja);

    // Semak dalam kawasan (koordinat KADA Kemubu)
    let jarak = kira_jarak(data.koordinat.lat, data.koordinat.lon, 6.0538, 102.2503);
    if jarak > 1.0 {
        return (StatusCode::FORBIDDEN,
                Json(ApiResponse::<()>::ralat(
                    &format!("Lokasi di luar kawasan ({:.2}km dari pejabat)", jarak)
                ))).into_response();
    }

    let rekod = RekodKehadiran {
        id:          Uuid::new_v4().to_string(),
        id_pekerja:  data.id_pekerja.clone(),
        masa_masuk:  Utc::now().to_rfc3339(),
        masa_keluar: None,
        koordinat:   data.koordinat,
        kaedah:      data.kaedah,
    };

    state.kehadiran.write().await.push(rekod.clone());
    println!("  → Kehadiran: pekerja {} ({:.2}km)", data.id_pekerja, jarak);

    (StatusCode::CREATED, Json(ApiResponse::ok(rekod))).into_response()
}

async fn senarai_kehadiran(
    State(state): State<AppState>,
) -> impl IntoResponse {
    let kehadiran = state.kehadiran.read().await;
    Json(ApiResponse::ok(kehadiran.clone()))
}

fn kira_jarak(lat1: f64, lon1: f64, lat2: f64, lon2: f64) -> f64 {
    let dlat = (lat2 - lat1).to_radians();
    let dlon = (lon2 - lon1).to_radians();
    let a = (dlat/2.0).sin().powi(2)
          + lat1.to_radians().cos() * lat2.to_radians().cos()
          * (dlon/2.0).sin().powi(2);
    6371.0 * 2.0 * a.sqrt().atan2((1.0-a).sqrt())
}

// ─── Main ──────────────────────────────────────────────────────

#[tokio::main]
async fn main() {
    use tower_http::cors::CorsLayer;

    let state = AppState::baru();

    let app = Router::new()
        .route("/",             get(sihat))
        .route("/sihat",        get(sihat))
        .route("/pekerja",      get(senarai_pekerja).post(buat_pekerja))
        .route("/pekerja/:id",  get(dapatkan_pekerja))
        .route("/kehadiran",    get(senarai_kehadiran).post(daftar_masuk))
        .layer(middleware::from_fn(log_permintaan))
        .layer(CorsLayer::permissive())
        .with_state(state);

    let pendengar = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();

    println!("{'═'*50}");
    println!("  KADA Mobile API Server");
    println!("  http://localhost:3000");
    println!("{'═'*50}");
    println!("  Endpoint:");
    println!("  GET  /sihat");
    println!("  GET  /pekerja");
    println!("  POST /pekerja");
    println!("  GET  /pekerja/:id");
    println!("  GET  /kehadiran");
    println!("  POST /kehadiran");
    println!("{'═'*50}");

    axum::serve(pendengar, app).await.unwrap();
}
```

---

# 📋 Rujukan Pantas — Network Programming Cheat Sheet

## Crates Utama

```toml
# Async runtime (WAJIB untuk networking)
tokio = { version = "1", features = ["full"] }

# HTTP Client
reqwest = { version = "0.11", features = ["json"] }

# HTTP Server
axum = "0.7"
tower-http = { version = "0.5", features = ["cors", "trace"] }

# WebSocket
axum = { version = "0.7", features = ["ws"] }

# gRPC
tonic = "0.10"
prost = "0.12"

# TLS
rustls = "0.21"
tokio-rustls = "0.24"
```

## TCP Patterns

```rust
// Server
let pendengar = TcpListener::bind("0.0.0.0:port").await?;
let (strim, alamat) = pendengar.accept().await?;
tokio::spawn(async move { kendalikan(strim).await; });

// Client
let strim = TcpStream::connect("host:port").await?;
let (pembaca, penulis) = strim.into_split();

// Read/Write
let mut buf = [0u8; 1024];
let n = strim.read(&mut buf).await?;
strim.write_all(data).await?;
```

## HTTP Client Patterns

```rust
let klien = reqwest::Client::new();

// GET
let balas = klien.get(url).send().await?;
let data: T = balas.json().await?;

// POST JSON
let balas = klien.post(url).json(&data).send().await?;

// Dengan headers
klien.get(url)
    .bearer_auth("token")
    .header("X-Custom", "value")
    .send().await?;
```

## HTTP Server Patterns (axum)

```rust
// Route
Router::new()
    .route("/", get(handler))
    .route("/:id", get(get_one).put(update).delete(remove))

// Handler
async fn handler(
    State(state): State<AppState>,
    Path(id): Path<u32>,
    Query(q): Query<QueryParams>,
    Json(body): Json<RequestBody>,
) -> impl IntoResponse {
    (StatusCode::OK, Json(response))
}

// State (shared app state)
let app = Router::new()
    .route("/", get(handler))
    .with_state(Arc::new(state));
```

## WebSocket Patterns

```rust
async fn ws_handler(ws: WebSocketUpgrade) -> impl IntoResponse {
    ws.on_upgrade(|soket| async {
        let (mut pengirim, mut penerima) = soket.split();
        while let Some(Ok(msg)) = penerima.next().await {
            pengirim.send(msg).await.ok();
        }
    })
}
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **Chat Server** — WebSocket server yang broadcast mesej ke semua klien dengan room support
2. **Load Balancer** — proxy TCP yang agihkan sambungan ke beberapa backend servers
3. **HTTP Proxy** — proxy yang forward HTTP request, log, dan cache response
4. **File Server** — serve static files dengan range request dan streaming
5. **API Gateway** — aggregate beberapa microservice API ke satu endpoint

---

*Network programming dalam Rust — selamat seperti Python, laju seperti C.*
*Dengan Tokio dan axum, bina server yang boleh handle jutaan request.* 🦀
