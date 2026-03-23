# 🔴 Redis Integration dalam Rust — Panduan Lengkap

> Dari sambungan asas hingga connection pooling, pub/sub,
> caching pattern, dan session management.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Kenapa Redis?

```
Redis = Remote Dictionary Server
      = In-memory data structure store

Guna kes utama:
  ✔ Caching         — simpan hasil query DB yang mahal
  ✔ Session store   — simpan data session pengguna
  ✔ Rate limiting   — hadkan bilangan request per IP
  ✔ Pub/Sub         — sistem mesej realtime
  ✔ Queue           — background job queue
  ✔ Leaderboard     — sorted set untuk ranking
  ✔ Distributed lock — koordinasi antara servers

Redis vs Memcached:
  Redis   → Data structures (string, hash, list, set, sorted set)
            Persistence (AOF/RDB), Pub/Sub, Lua scripting
  Memcached → String sahaja, lebih mudah, lebih ringan
```

---

## Peta Pembelajaran

```
Bahagian 1  → Setup & Sambungan Asas
Bahagian 2  → String Operations
Bahagian 3  → Hash, List, Set, Sorted Set
Bahagian 4  → TTL & Expiry
Bahagian 5  → Connection Pool dengan deadpool-redis
Bahagian 6  → Async dengan tokio
Bahagian 7  → Pub/Sub
Bahagian 8  → Transactions & Pipelines
Bahagian 9  → Pattern — Cache, Session, Rate Limit
Bahagian 10 → Mini Project: KADA Caching Layer
```

---

# BAHAGIAN 1: Setup & Sambungan Asas 🔌

## Pasang Redis

```bash
# Ubuntu/Debian
sudo apt install redis-server
sudo systemctl start redis

# macOS
brew install redis
brew services start redis

# Windows — guna Docker
docker run -d -p 6379:6379 redis:latest

# Verify
redis-cli ping   # → PONG
redis-cli info server
```

## Cargo.toml

```toml
[dependencies]
# Sync + Async Redis client
redis = { version = "0.25", features = [
    "tokio-comp",     # Tokio async support
    "connection-manager", # Auto-reconnect
    "json",           # JSON support
] }

# Connection pooling
deadpool-redis = "0.15"

# Async runtime
tokio = { version = "1", features = ["full"] }

# Serialization
serde       = { version = "1", features = ["derive"] }
serde_json  = "1"

# Error handling
thiserror = "1"
anyhow    = "1"
```

---

## Sambungan Asas

```rust
use redis::{Client, Commands, RedisResult};

fn main() -> RedisResult<()> {
    // ── Sambungan synchronous ─────────────────────────────────

    // Cara 1: Guna URL
    let klien = Client::open("redis://127.0.0.1:6379")?;
    let mut sambungan = klien.get_connection()?;

    println!("Sambungan berjaya!");

    // SET dan GET asas
    sambungan.set("kunci", "nilai")?;
    let nilai: String = sambungan.get("kunci")?;
    println!("Dapat: {}", nilai);

    // ── Berbagai format URL ───────────────────────────────────
    // redis://127.0.0.1:6379          → standard
    // redis://127.0.0.1:6379/1        → database 1 (default 0)
    // redis://:katalaluan@127.0.0.1   → dengan kata laluan
    // redis://pengguna:laluan@host:port/db
    // rediss://127.0.0.1              → Redis dengan TLS

    // ── Dengan kata laluan ────────────────────────────────────
    // let klien = Client::open("redis://:rahsia123@127.0.0.1:6379")?;

    // ── Test sambungan ────────────────────────────────────────
    let pong: String = redis::cmd("PING").query(&mut sambungan)?;
    println!("Ping → {}", pong); // PONG

    Ok(())
}
```

---

## 🧠 Brain Teaser #1

Apakah perbezaan antara `Connection` dan `ConnectionManager` dalam crate redis?

<details>
<summary>👀 Jawapan</summary>

```
Connection:
  - Satu sambungan biasa ke Redis
  - Kalau sambungan putus → GAGAL, perlu sambung semula manual
  - Sesuai untuk aplikasi kecil atau single-threaded
  - let mut conn = client.get_connection()?;

ConnectionManager:
  - Auto-reconnect bila sambungan putus
  - Mengurus retry secara automatik
  - Sesuai untuk production aplikasi
  - let mgr = ConnectionManager::new(client)?;

Dalam kebanyakan kes production:
  - Guna deadpool-redis untuk connection pool (lebih baik)
  - ConnectionManager sesuai untuk async tunggal
  - Connection asas untuk scripting atau ujian

Analogi:
  Connection      = beli tiket bas sekali (kalau bas cancel, stuck!)
  ConnectionManager = langganan bas (auto-arrange pengganti)
  deadpool-redis  = pool teksi (selalu ada kenderaan sedia)
```
</details>

---

# BAHAGIAN 2: String Operations 📝

## Operasi String Asas

```rust
use redis::{Commands, RedisResult, Client};
use std::time::Duration;

fn demo_string(conn: &mut redis::Connection) -> RedisResult<()> {
    // ── SET dan GET ───────────────────────────────────────────

    // SET — simpan nilai
    conn.set("nama", "Ali Ahmad")?;
    conn.set("umur", 25i32)?;
    conn.set("gaji", 4500.50f64)?;

    // GET — ambil nilai
    let nama: String = conn.get("nama")?;
    let umur: i32    = conn.get("umur")?;
    let gaji: f64    = conn.get("gaji")?;
    println!("{}, {}, {}", nama, umur, gaji);

    // GET — nilai tidak wujud → None
    let tiada: Option<String> = conn.get("kunci_tiada")?;
    println!("Tiada: {:?}", tiada); // None

    // ── SET dengan Expiry ─────────────────────────────────────

    // EX = dalam saat
    conn.set_ex("token", "abc123", 3600)?;  // tamat dalam 1 jam

    // PX = dalam milisaat
    conn.pset_ex("cache", "data", 30000)?;  // tamat dalam 30 saat

    // ── Atomic Operations ──────────────────────────────────────

    // SETNX — SET only if Not eXists
    let baru: bool = conn.set_nx("sekali", "pertama")?;
    println!("Set baru: {}", baru);  // true (berjaya)

    let lagi: bool = conn.set_nx("sekali", "kedua")?;
    println!("Set lagi: {}", lagi);  // false (sudah ada)

    let nilai: String = conn.get("sekali")?;
    println!("Nilai: {}", nilai);    // "pertama" (tidak berubah!)

    // GETSET — get nilai lama dan set nilai baru
    let lama: Option<String> = conn.getset("nama", "Siti Hawa")?;
    println!("Lama: {:?}", lama);    // Some("Ali Ahmad")

    // GETDEL — get dan padam
    let diambil: Option<String> = conn.getdel("sekali")?;
    println!("Diambil: {:?}", diambil); // Some("pertama")
    let tiada2: Option<String> = conn.get("sekali")?;
    println!("Selepas getdel: {:?}", tiada2); // None

    // ── Numeric Operations ────────────────────────────────────

    conn.set("kiraan", 0i64)?;
    conn.incr("kiraan", 1i64)?;     // +1
    conn.incr("kiraan", 1i64)?;     // +1
    conn.incr("kiraan", 5i64)?;     // +5
    let k: i64 = conn.get("kiraan")?;
    println!("Kiraan: {}", k);       // 7

    conn.decr("kiraan", 3i64)?;     // -3
    let k2: i64 = conn.get("kiraan")?;
    println!("Selepas decr: {}", k2); // 4

    // ── String Operations ─────────────────────────────────────

    conn.set("mesej", "Hello")?;
    conn.append("mesej", ", World!")?;  // tambah di hujung
    let mesej: String = conn.get("mesej")?;
    println!("{}", mesej);               // Hello, World!

    let panjang: i64 = conn.strlen("mesej")?;
    println!("Panjang: {}", panjang);    // 13

    // Bersihkan
    conn.del("nama")?;
    conn.del("token")?;
    conn.del("mesej")?;
    conn.del("kiraan")?;

    Ok(())
}

fn main() -> RedisResult<()> {
    let klien = Client::open("redis://127.0.0.1:6379")?;
    let mut conn = klien.get_connection()?;
    demo_string(&mut conn)?;
    Ok(())
}
```

---

# BAHAGIAN 3: Hash, List, Set, Sorted Set 🗃️

## Hash — Macam Struct dalam Redis

```rust
use redis::Commands;

fn demo_hash(conn: &mut redis::Connection) -> redis::RedisResult<()> {
    // Hash = field → value dalam satu kunci
    // Sesuai untuk store object (pekerja, produk, dll)

    // ── HSET — set satu atau banyak field ─────────────────────
    conn.hset("pekerja:1", "nama",     "Ali Ahmad")?;
    conn.hset("pekerja:1", "bahagian", "ICT")?;
    conn.hset("pekerja:1", "gaji",     "4500.00")?;

    // Set banyak field sekaligus
    conn.hset_multiple("pekerja:2", &[
        ("nama",     "Siti Hawa"),
        ("bahagian", "HR"),
        ("gaji",     "3800.00"),
    ])?;

    // ── HGET — ambil satu field ───────────────────────────────
    let nama: String = conn.hget("pekerja:1", "nama")?;
    println!("Nama: {}", nama); // Ali Ahmad

    // ── HMGET — ambil banyak field ────────────────────────────
    let (n, b, g): (String, String, String) =
        conn.hget("pekerja:1", &["nama", "bahagian", "gaji"])?;
    println!("{} / {} / RM{}", n, b, g);

    // ── HGETALL — ambil semua field ───────────────────────────
    let semua: std::collections::HashMap<String, String> =
        conn.hgetall("pekerja:1")?;
    for (k, v) in &semua {
        println!("  {}: {}", k, v);
    }

    // ── HEXISTS — semak kewujudan field ───────────────────────
    let ada: bool = conn.hexists("pekerja:1", "nama")?;
    println!("Ada 'nama': {}", ada); // true

    // ── HDEL — padam field ────────────────────────────────────
    conn.hdel("pekerja:1", "gaji")?;

    // ── HKEYS, HVALS, HLEN ────────────────────────────────────
    let kunci: Vec<String> = conn.hkeys("pekerja:1")?;
    let nilai: Vec<String> = conn.hvals("pekerja:1")?;
    let bilangan: i64     = conn.hlen("pekerja:1")?;
    println!("Keys: {:?}", kunci);
    println!("Vals: {:?}", nilai);
    println!("Len:  {}", bilangan);

    // ── HINCRBY — tambah nilai numeric field ──────────────────
    conn.hset("kiraan:hari", "login", 0i64)?;
    conn.hincr("kiraan:hari", "login", 1i64)?;
    conn.hincr("kiraan:hari", "login", 1i64)?;
    let login: i64 = conn.hget("kiraan:hari", "login")?;
    println!("Login hari ini: {}", login); // 2

    // Bersihkan
    conn.del("pekerja:1")?;
    conn.del("pekerja:2")?;
    conn.del("kiraan:hari")?;

    Ok(())
}
```

---

## List, Set & Sorted Set

```rust
use redis::Commands;

fn demo_list_set(conn: &mut redis::Connection) -> redis::RedisResult<()> {
    // ── LIST — ordered, boleh duplikat ────────────────────────
    // Guna: queue, history, log

    // LPUSH/RPUSH — push ke kiri/kanan
    conn.rpush("barisan", "pertama")?;
    conn.rpush("barisan", "kedua")?;
    conn.rpush("barisan", "ketiga")?;
    conn.lpush("barisan", "sebelum_pertama")?;

    // LRANGE — ambil range
    let senarai: Vec<String> = conn.lrange("barisan", 0, -1)?;
    println!("List: {:?}", senarai);
    // ["sebelum_pertama", "pertama", "kedua", "ketiga"]

    // LPOP/RPOP — pop dari kiri/kanan
    let depan: String = conn.lpop("barisan", None)?;
    println!("Pop kiri: {}", depan); // sebelum_pertama

    let belakang: String = conn.rpop("barisan", None)?;
    println!("Pop kanan: {}", belakang); // ketiga

    // LLEN — panjang
    let panjang: i64 = conn.llen("barisan")?;
    println!("Panjang: {}", panjang); // 2

    // ── SET — tiada order, tiada duplikat ─────────────────────
    // Guna: unique values, tag, membership

    conn.sadd("tag:artikel1", "rust")?;
    conn.sadd("tag:artikel1", "programming")?;
    conn.sadd("tag:artikel1", "rust")?; // duplikat — diabaikan!

    let ahli: Vec<String> = conn.smembers("tag:artikel1")?;
    println!("Tags: {:?}", ahli); // ["rust", "programming"] (random order)

    let ada: bool = conn.sismember("tag:artikel1", "rust")?;
    println!("Ada 'rust': {}", ada); // true

    // Set operations
    conn.sadd("tag:artikel2", "rust")?;
    conn.sadd("tag:artikel2", "async")?;

    let intersection: Vec<String> = conn.sinter(&["tag:artikel1", "tag:artikel2"])?;
    let union_set: Vec<String>    = conn.sunion(&["tag:artikel1", "tag:artikel2"])?;
    let difference: Vec<String>   = conn.sdiff(&["tag:artikel1", "tag:artikel2"])?;
    println!("Intersection: {:?}", intersection); // ["rust"]
    println!("Union:        {:?}", union_set);    // ["rust", "programming", "async"]
    println!("Difference:   {:?}", difference);  // ["programming"]

    // ── SORTED SET — dengan skor ──────────────────────────────
    // Guna: leaderboard, ranking, priority queue

    conn.zadd("papan_kedudukan", "Ali",  85.5)?;
    conn.zadd("papan_kedudukan", "Siti", 92.0)?;
    conn.zadd("papan_kedudukan", "Amin", 78.3)?;
    conn.zadd("papan_kedudukan", "Zara", 95.1)?;

    // ZRANGE — ascending (skor rendah dulu)
    let bawah: Vec<String> = conn.zrange("papan_kedudukan", 0, -1)?;
    println!("Bawah ke Atas: {:?}", bawah);

    // ZREVRANGE — descending (skor tinggi dulu)
    let atas: Vec<(String, f64)> = conn.zrevrange_withscores("papan_kedudukan", 0, -1)?;
    println!("\nPapan Kedudukan:");
    for (i, (nama, skor)) in atas.iter().enumerate() {
        println!("  {}. {} — {:.1} mata", i + 1, nama, skor);
    }

    // ZSCORE — dapatkan skor
    let skor: f64 = conn.zscore("papan_kedudukan", "Siti")?;
    println!("Skor Siti: {}", skor); // 92.0

    // ZRANK — dapatkan ranking (0-indexed)
    let ranking: i64 = conn.zrevrank("papan_kedudukan", "Siti")?;
    println!("Ranking Siti (dari atas): {}", ranking + 1); // 2

    // Bersihkan
    conn.del("barisan")?;
    conn.del("tag:artikel1")?;
    conn.del("tag:artikel2")?;
    conn.del("papan_kedudukan")?;

    Ok(())
}
```

---

# BAHAGIAN 4: TTL & Expiry ⏰

## Urus Masa Tamat

```rust
use redis::Commands;
use std::time::Duration;
use std::thread;

fn demo_ttl(conn: &mut redis::Connection) -> redis::RedisResult<()> {
    // ── Set TTL semasa buat ───────────────────────────────────
    conn.set_ex("token_akses", "abc123", 3600)?; // 1 jam
    conn.pset_ex("otp", "123456", 300_000)?;     // 5 minit (ms)

    // ── Semak TTL ─────────────────────────────────────────────
    let ttl: i64 = conn.ttl("token_akses")?;
    println!("TTL token: {} saat", ttl); // ~3600

    let pttl: i64 = conn.pttl("otp")?;
    println!("PTTL OTP: {} ms", pttl);   // ~300000

    // TTL = -1 → tiada expiry (kekal)
    // TTL = -2 → kunci tidak wujud

    // ── Tambah TTL kepada kunci yang sedia ada ────────────────
    conn.set("tanpa_ttl", "nilai")?;         // tiada TTL
    let ttl_before: i64 = conn.ttl("tanpa_ttl")?;
    println!("TTL sebelum: {}", ttl_before); // -1 (tiada expiry)

    conn.expire("tanpa_ttl", 60)?;           // set 60 saat
    let ttl_after: i64 = conn.ttl("tanpa_ttl")?;
    println!("TTL selepas: {}", ttl_after);  // ~60

    // EXPIREAT — tamat pada timestamp tertentu
    let unix_masa = chrono::Utc::now().timestamp() + 3600;
    conn.expireat("tanpa_ttl", unix_masa as u64)?;

    // ── Padam TTL (buat kekal) ────────────────────────────────
    conn.persist("tanpa_ttl")?;
    let ttl_kekal: i64 = conn.ttl("tanpa_ttl")?;
    println!("TTL kekal: {}", ttl_kekal); // -1 (tiada expiry)

    // ── Demo: OTP yang tamat ──────────────────────────────────
    conn.set_ex("otp_demo", "654321", 2)?; // tamat dalam 2 saat
    let otp: String = conn.get("otp_demo")?;
    println!("OTP sekarang: {}", otp);

    thread::sleep(Duration::from_secs(3));

    let otp_tamat: Option<String> = conn.get("otp_demo")?;
    println!("OTP selepas 3 saat: {:?}", otp_tamat); // None (sudah tamat!)

    // Bersihkan
    conn.del("token_akses")?;
    conn.del("tanpa_ttl")?;

    Ok(())
}

fn main() -> redis::RedisResult<()> {
    let klien = redis::Client::open("redis://127.0.0.1:6379")?;
    let mut conn = klien.get_connection()?;
    demo_ttl(&mut conn)?;
    Ok(())
}
```

---

# BAHAGIAN 5: Connection Pool dengan deadpool-redis 🏊

## Setup Pool

```rust
use deadpool_redis::{Config, Pool, Runtime, Connection};
use redis::cmd;

async fn buat_pool() -> Pool {
    let konfig = Config::from_url("redis://127.0.0.1:6379");
    konfig.create_pool(Some(Runtime::Tokio1)).unwrap()
}

// Atau dengan konfigurasi penuh
async fn buat_pool_penuh() -> Pool {
    let mut konfig = Config::default();
    konfig.url = Some("redis://127.0.0.1:6379".to_string());
    konfig.pool = Some(deadpool_redis::PoolConfig {
        max_size:    20,            // max 20 sambungan
        ..Default::default()
    });
    konfig.create_pool(Some(Runtime::Tokio1)).unwrap()
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let pool = buat_pool().await;

    // Ambil sambungan dari pool
    let mut conn: Connection = pool.get().await?;

    // Guna sambungan seperti biasa
    redis::cmd("SET").arg("hello").arg("world").execute_async(&mut conn).await?;
    let nilai: String = redis::cmd("GET").arg("hello").query_async(&mut conn).await?;
    println!("Nilai: {}", nilai);

    // Sambungan AUTO-KEMBALI ke pool bila di-drop!

    // Concurrent access
    let pool_klon = pool.clone();
    let mut tugasan = Vec::new();

    for i in 0..10 {
        let p = pool_klon.clone();
        tugasan.push(tokio::spawn(async move {
            let mut c = p.get().await.unwrap();
            let kunci = format!("kiraan:{}", i);
            redis::cmd("INCR").arg(&kunci).execute_async(&mut c).await.unwrap();
            let nilai: i64 = redis::cmd("GET").arg(&kunci).query_async(&mut c).await.unwrap();
            println!("Thread {}: {}", i, nilai);
        }));
    }

    for t in tugasan { t.await?; }

    // Bersihkan
    let mut conn = pool.get().await?;
    for i in 0..10 {
        redis::cmd("DEL").arg(format!("kiraan:{}", i)).execute_async(&mut conn).await?;
    }
    redis::cmd("DEL").arg("hello").execute_async(&mut conn).await?;

    Ok(())
}
```

---

## Pool dalam Axum App State

```rust
use axum::{
    Router,
    routing::get,
    extract::State,
    response::Json,
};
use deadpool_redis::{Config, Pool, Runtime};
use redis::AsyncCommands;
use serde_json::Value;

#[derive(Clone)]
struct AppState {
    redis: Pool,
}

async fn kendalikan_cache(
    State(state): State<AppState>,
) -> Json<Value> {
    let mut conn = state.redis.get().await.unwrap();

    // Guna AsyncCommands untuk async Redis
    let kiraan: i64 = conn.incr("request_count", 1).await.unwrap_or(0);

    Json(serde_json::json!({
        "mesej": "OK",
        "kiraan_request": kiraan
    }))
}

#[tokio::main]
async fn main() {
    let redis_pool = Config::from_url("redis://127.0.0.1:6379")
        .create_pool(Some(Runtime::Tokio1))
        .unwrap();

    let state = AppState { redis: redis_pool };

    let app = Router::new()
        .route("/api", get(kendalikan_cache))
        .with_state(state);

    let pendengar = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("Server pada http://localhost:3000");
    axum::serve(pendengar, app).await.unwrap();
}
```

---

# BAHAGIAN 6: Async Redis Operations ⚡

## AsyncCommands trait

```rust
use redis::AsyncCommands;
use deadpool_redis::{Config, Runtime};

#[tokio::main]
async fn main() -> redis::RedisResult<()> {
    let pool = Config::from_url("redis://127.0.0.1:6379")
        .create_pool(Some(Runtime::Tokio1))
        .unwrap();

    let mut conn = pool.get().await.unwrap();

    // AsyncCommands — sama seperti Commands tapi async
    conn.set("kunci", "nilai").await?;
    let nilai: String = conn.get("kunci").await?;
    println!("{}", nilai);

    // SET dengan EX
    conn.set_ex("token", "abc", 3600u64).await?;

    // Hash
    conn.hset("user:1", "nama", "Ali").await?;
    conn.hset("user:1", "umur", 25i32).await?;
    let nama: String = conn.hget("user:1", "nama").await?;
    println!("Nama: {}", nama);

    // Increment
    conn.incr("kiraan", 1i64).await?;

    // Serentak!
    let (r1, r2, r3): (String, i64, String) = tokio::join!(
        async { conn.get("kunci").await.unwrap_or_default() },
        async { conn.incr("kiraan", 1i64).await.unwrap_or(0) },
        async { conn.hget("user:1", "nama").await.unwrap_or_default() },
    );
    println!("Serentak: {}, {}, {}", r1, r2, r3);

    // Bersihkan
    conn.del("kunci").await?;
    conn.del("token").await?;
    conn.del("user:1").await?;
    conn.del("kiraan").await?;

    Ok(())
}
```

---

# BAHAGIAN 7: Pub/Sub 📣

## Publisher dan Subscriber

```rust
use redis::{Client, Commands, PubSubCommands, Msg};
use std::thread;
use std::time::Duration;

fn demo_pubsub() {
    let klien = Client::open("redis://127.0.0.1:6379").unwrap();

    // ── Subscriber dalam thread berasingan ────────────────────
    let klien_sub = klien.clone();
    let subscriber = thread::spawn(move || {
        let mut conn = klien_sub.get_connection().unwrap();
        let mut pubsub = conn.as_pubsub();

        // Subscribe ke channel
        pubsub.subscribe("berita").unwrap();
        pubsub.subscribe("notifikasi").unwrap();
        println!("Subscriber sedia!");

        // Dengar mesej
        loop {
            let msg: Msg = pubsub.get_message().unwrap();
            let channel: String = msg.get_channel_name().to_string();
            let data: String = msg.get_payload().unwrap();

            println!("[{}] {}", channel, data);

            if data == "HENTI" { break; }
        }
    });

    // ── Publisher ──────────────────────────────────────────────
    thread::sleep(Duration::from_millis(500)); // tunggu subscriber sedia

    let mut pub_conn = klien.get_connection().unwrap();

    pub_conn.publish("berita", "Rust 2.0 dilancarkan!").unwrap();
    pub_conn.publish("notifikasi", "5 pekerja baru berdaftar").unwrap();
    pub_conn.publish("berita", "Redis 8.0 kini tersedia").unwrap();

    thread::sleep(Duration::from_millis(100));
    pub_conn.publish("berita", "HENTI").unwrap();

    subscriber.join().unwrap();
}

fn main() {
    demo_pubsub();
}
```

---

## Async Pub/Sub dengan Tokio

```rust
use redis::aio::PubSub;
use redis::AsyncCommands;
use futures::StreamExt;

#[tokio::main]
async fn main() -> redis::RedisResult<()> {
    let klien = redis::Client::open("redis://127.0.0.1:6379")?;

    // Subscriber async
    let subscriber = tokio::spawn(async move {
        let conn = klien.get_async_connection().await.unwrap();
        let mut pubsub: PubSub = conn.into_pubsub();

        pubsub.subscribe("siaran").await.unwrap();
        pubsub.psubscribe("pela:*").await.unwrap(); // pattern subscribe

        let mut strim = pubsub.into_on_message();

        while let Some(mesej) = strim.next().await {
            let channel = mesej.get_channel_name();
            let data: String = mesej.get_payload().unwrap();
            println!("[{}] {}", channel, data);
            if data == "SELESAI" { break; }
        }
    });

    // Publisher async
    tokio::time::sleep(tokio::time::Duration::from_millis(200)).await;

    let klien2 = redis::Client::open("redis://127.0.0.1:6379")?;
    let mut pub_conn = klien2.get_async_connection().await?;

    pub_conn.publish("siaran", "Hello semua!").await?;
    pub_conn.publish("pela:ICT", "Mesej untuk ICT").await?;
    pub_conn.publish("pela:HR", "Mesej untuk HR").await?;
    pub_conn.publish("siaran", "SELESAI").await?;

    subscriber.await.unwrap();

    Ok(())
}
```

---

## 🧠 Brain Teaser #2

Apakah masalah dengan menggunakan sambungan Redis yang sama untuk Pub/Sub dan operasi biasa?

<details>
<summary>👀 Jawapan</summary>

```
MASALAH: Sambungan yang dalam mod Pub/Sub TIDAK BOLEH
         digunakan untuk operasi Redis biasa!

Bila anda subscribe kepada channel:
  conn.subscribe("channel")?;
  // Kini conn dalam "pub/sub mode"!

Dalam mod pub/sub, sambungan HANYA boleh:
  ✔ SUBSCRIBE ke channel lain
  ✔ PSUBSCRIBE (pattern subscribe)
  ✔ UNSUBSCRIBE
  ✔ PING

TIDAK BOLEH:
  ✗ SET, GET, HSET, dll (operasi biasa)
  ✗ Sebarang command Redis lain

PENYELESAIAN:
  Guna DUA sambungan berasingan:
  1. Sambungan untuk pub/sub (subscribe dan terima mesej)
  2. Sambungan lain untuk operasi biasa (SET, GET, dll)

let mut sub_conn = klien.get_connection()?; // untuk subscribe
let mut op_conn  = klien.get_connection()?; // untuk operasi biasa

// Dalam production:
// Subscriber → sambungan tersendiri dalam dedicated thread/task
// Publisher/Operations → connection pool (deadpool-redis)
```
</details>

---

# BAHAGIAN 8: Transactions & Pipelines 🔄

## Pipeline — Hantar Banyak Command Sekaligus

```rust
use redis::{Client, Commands, Pipeline};

fn demo_pipeline() -> redis::RedisResult<()> {
    let klien = Client::open("redis://127.0.0.1:6379")?;
    let mut conn = klien.get_connection()?;

    // Tanpa pipeline: setiap command = satu round-trip ke Redis
    // Dengan pipeline: semua command dalam SATU round-trip!

    // ── Pipeline asas ─────────────────────────────────────────
    let hasil: (bool, bool, String, String) = redis::pipe()
        .set("a", "1")
        .set("b", "2")
        .get("a")
        .get("b")
        .query(&mut conn)?;

    println!("Pipeline result: {:?}", hasil);

    // ── Pipeline dengan atomic: false (boleh ada command lain celah) ──
    let mut pipe = redis::pipe();
    pipe.set("x", 10)
        .incr("x", 5)
        .get("x");

    let hasil: Vec<redis::Value> = pipe.query(&mut conn)?;
    println!("Pipe atomic=false: {:?}", hasil);

    // ── Pipeline dengan ignore ─────────────────────────────────
    // .ignore() — jangan ambil hasil command ini
    let (_, nilai): ((), String) = redis::pipe()
        .set("temp", "sementara").ignore()
        .get("temp")
        .query(&mut conn)?;
    println!("Nilai: {}", nilai);

    // Bersihkan
    conn.del(&["a", "b", "x", "temp"])?;

    Ok(())
}

// ── Transaction dengan MULTI/EXEC ─────────────────────────────
fn demo_transaction() -> redis::RedisResult<()> {
    let klien = Client::open("redis://127.0.0.1:6379")?;
    let mut conn = klien.get_connection()?;

    // MULTI/EXEC — semua atau tiada (atomic)
    let hasil: (bool, i64, String) = redis::transaction(
        &mut conn,
        &["akaun:bakti"],  // kunci yang di-WATCH
        |conn, pipe| {
            // Baca nilai semasa (di luar transaction)
            let baki: i64 = conn.get("akaun:bakti").unwrap_or(0);

            if baki < 100 {
                // Tak cukup wang — return None untuk abort
                return Ok(None);
            }

            // Dalam transaction
            pipe.decrby("akaun:bakti", 100)
                .incrby("akaun:penerima", 100)
                .get("akaun:bakti")
                .query(conn)
        }
    )?;

    println!("Transaction result: {:?}", hasil);

    // Bersihkan
    conn.del(&["akaun:bakti", "akaun:penerima"])?;

    Ok(())
}

// ── Transfer wang — guna WATCH untuk optimistic locking ───────
fn transfer_wang(
    conn: &mut redis::Connection,
    dari: &str,
    ke: &str,
    jumlah: i64,
) -> redis::RedisResult<bool> {
    loop {
        // WATCH — pantau kunci ini
        redis::cmd("WATCH").arg(dari).execute(conn);

        let baki: i64 = conn.get(dari).unwrap_or(0);

        if baki < jumlah {
            redis::cmd("UNWATCH").execute(conn);
            return Ok(false); // baki tidak cukup
        }

        let hasil: Result<Vec<i64>, _> = redis::transaction(conn, &[dari], |conn, pipe| {
            pipe.decrby(dari, jumlah)
                .incrby(ke, jumlah)
                .query(conn)
        });

        match hasil {
            Ok(_)  => return Ok(true),
            Err(_) => continue, // WATCH triggered — cuba lagi
        }
    }
}

fn main() -> redis::RedisResult<()> {
    demo_pipeline()?;
    demo_transaction()?;

    let klien = Client::open("redis://127.0.0.1:6379")?;
    let mut conn = klien.get_connection()?;

    conn.set("akaun:ali", 500i64)?;
    conn.set("akaun:siti", 100i64)?;

    let berjaya = transfer_wang(&mut conn, "akaun:ali", "akaun:siti", 200)?;
    println!("Transfer: {}", if berjaya { "berjaya" } else { "gagal" });

    let bakti_ali: i64 = conn.get("akaun:ali")?;
    let bakti_siti: i64 = conn.get("akaun:siti")?;
    println!("Ali: RM{}, Siti: RM{}", bakti_ali, bakti_siti);

    conn.del(&["akaun:ali", "akaun:siti"])?;

    Ok(())
}
```

---

# BAHAGIAN 9: Patterns — Cache, Session, Rate Limit 🎯

## Pattern 1: Cache Aside

```rust
use deadpool_redis::{Config, Pool, Runtime};
use redis::AsyncCommands;
use serde::{Serialize, Deserialize};
use serde_json;

#[derive(Debug, Serialize, Deserialize, Clone)]
struct DataPekerja {
    id:       u32,
    nama:     String,
    bahagian: String,
    gaji:     f64,
}

async fn dapatkan_pekerja(
    pool: &Pool,
    id: u32,
) -> Result<DataPekerja, Box<dyn std::error::Error>> {
    let mut conn = pool.get().await?;
    let kunci_cache = format!("cache:pekerja:{}", id);

    // 1. Semak cache dulu
    let cache_hit: Option<String> = conn.get(&kunci_cache).await?;

    if let Some(data_json) = cache_hit {
        println!("🎯 Cache HIT untuk pekerja {}", id);
        let pekerja: DataPekerja = serde_json::from_str(&data_json)?;
        return Ok(pekerja);
    }

    // 2. Cache MISS — ambil dari "database"
    println!("💾 Cache MISS — ambil dari DB untuk pekerja {}", id);

    // Simulate DB query
    tokio::time::sleep(tokio::time::Duration::from_millis(50)).await;
    let pekerja = DataPekerja {
        id,
        nama:     format!("Pekerja {}", id),
        bahagian: "ICT".into(),
        gaji:     4500.0,
    };

    // 3. Simpan dalam cache (TTL 5 minit = 300 saat)
    let data_json = serde_json::to_string(&pekerja)?;
    conn.set_ex(&kunci_cache, &data_json, 300u64).await?;
    println!("✅ Cache disimpan untuk {} saat", 300);

    Ok(pekerja)
}

// Cache invalidation
async fn kemaskini_pekerja(
    pool: &Pool,
    id: u32,
    data_baru: DataPekerja,
) -> Result<(), Box<dyn std::error::Error>> {
    let mut conn = pool.get().await?;
    let kunci_cache = format!("cache:pekerja:{}", id);

    // Kemaskini DB (simulasi)
    println!("💾 Kemaskini DB...");

    // Padam cache (invalidate)
    conn.del(&kunci_cache).await?;
    println!("🗑️ Cache dipadamkan untuk pekerja {}", id);

    Ok(())
}
```

---

## Pattern 2: Session Management

```rust
use redis::AsyncCommands;
use serde::{Serialize, Deserialize};
use uuid::Uuid;

#[derive(Debug, Serialize, Deserialize)]
struct DataSesi {
    id_pengguna: u32,
    nama:        String,
    peranan:     Vec<String>,
    masa_log:    i64,
}

async fn buat_sesi(
    conn: &mut impl AsyncCommands,
    data: DataSesi,
    ttl_saat: u64,
) -> redis::RedisResult<String> {
    let id_sesi = Uuid::new_v4().to_string();
    let kunci = format!("sesi:{}", id_sesi);

    let json = serde_json::to_string(&data).unwrap();
    conn.set_ex(&kunci, json, ttl_saat).await?;

    println!("Sesi dibuat: {}", id_sesi);
    Ok(id_sesi)
}

async fn dapatkan_sesi(
    conn: &mut impl AsyncCommands,
    id_sesi: &str,
) -> redis::RedisResult<Option<DataSesi>> {
    let kunci = format!("sesi:{}", id_sesi);
    let json: Option<String> = conn.get(&kunci).await?;

    match json {
        Some(j) => {
            // Refresh TTL setiap kali diakses (sliding window)
            conn.expire(&kunci, 1800i64).await?; // refresh 30 min
            Ok(serde_json::from_str(&j).ok())
        }
        None => Ok(None),
    }
}

async fn hapus_sesi(
    conn: &mut impl AsyncCommands,
    id_sesi: &str,
) -> redis::RedisResult<()> {
    let kunci = format!("sesi:{}", id_sesi);
    conn.del(&kunci).await?;
    println!("Sesi dipadamkan: {}", id_sesi);
    Ok(())
}
```

---

## Pattern 3: Rate Limiting

```rust
use redis::AsyncCommands;

// Rate limiting menggunakan sliding window
async fn semak_had_kadar(
    conn: &mut impl AsyncCommands,
    pengecam: &str,   // IP atau ID pengguna
    had:      u64,    // max request
    tetingkap: u64,   // dalam saat
) -> redis::RedisResult<(bool, u64, u64)> {
    let kunci = format!("ratelimit:{}:{}", pengecam, tetingkap);
    let sekarang = std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .unwrap()
        .as_secs();

    // Tambah timestamp semasa
    let kiraan: u64 = conn.incr(&kunci, 1).await?;

    if kiraan == 1 {
        // Baru pertama kali dalam tetingkap ini
        conn.expire(&kunci, tetingkap as i64).await?;
    }

    let ttl: i64 = conn.ttl(&kunci).await?;
    let dibenarkan = kiraan <= had;

    println!("{}: {} / {} request (reset dalam {}s)",
        pengecam, kiraan, had, ttl.max(0));

    Ok((dibenarkan, kiraan, ttl.max(0) as u64))
}

// Token bucket rate limiter
async fn token_bucket(
    conn: &mut impl AsyncCommands,
    pengecam: &str,
    kapasiti: i64,
    kadar_isi: f64,   // token per saat
) -> redis::RedisResult<bool> {
    let kunci = format!("bucket:{}", pengecam);
    let sekarang = std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .unwrap()
        .as_secs_f64();

    // Script Lua untuk atomicity
    let skrip = r#"
        local kunci = KEYS[1]
        local kapasiti = tonumber(ARGV[1])
        local kadar = tonumber(ARGV[2])
        local sekarang = tonumber(ARGV[3])

        local data = redis.call('HMGET', kunci, 'token', 'masa')
        local token = tonumber(data[1]) or kapasiti
        local masa_lalu = tonumber(data[2]) or sekarang

        -- Isi token berdasarkan masa berlalu
        local berlalu = sekarang - masa_lalu
        token = math.min(kapasiti, token + berlalu * kadar)

        if token >= 1 then
            token = token - 1
            redis.call('HMSET', kunci, 'token', token, 'masa', sekarang)
            redis.call('EXPIRE', kunci, math.ceil(kapasiti / kadar) + 1)
            return 1  -- dibenarkan
        else
            return 0  -- ditolak
        end
    "#;

    let dibenarkan: i64 = redis::Script::new(skrip)
        .key(format!("bucket:{}", pengecam))
        .arg(kapasiti)
        .arg(kadar_isi)
        .arg(sekarang)
        .invoke_async(conn)
        .await?;

    Ok(dibenarkan == 1)
}
```

---

# BAHAGIAN 10: Mini Project — KADA Caching Layer 🏗️

```toml
[dependencies]
tokio         = { version = "1", features = ["full"] }
redis         = { version = "0.25", features = ["tokio-comp"] }
deadpool-redis = "0.15"
axum          = "0.7"
serde         = { version = "1", features = ["derive"] }
serde_json    = "1"
uuid          = { version = "1", features = ["v4"] }
thiserror     = "1"
```

```rust
use deadpool_redis::{Config, Pool, Runtime};
use redis::AsyncCommands;
use axum::{
    Router,
    routing::{get, post, delete},
    extract::{State, Path, Json},
    http::StatusCode,
    response::IntoResponse,
    middleware::{self, Next},
    extract::Request,
};
use serde::{Serialize, Deserialize};
use serde_json::Value;
use std::sync::Arc;
use uuid::Uuid;

// ─── Ralat ────────────────────────────────────────────────────

#[derive(Debug, thiserror::Error)]
enum AppError {
    #[error("Redis error: {0}")]
    Redis(#[from] redis::RedisError),
    #[error("Pool error: {0}")]
    Pool(#[from] deadpool_redis::PoolError),
    #[error("JSON error: {0}")]
    Json(#[from] serde_json::Error),
    #[error("Tidak dijumpai")]
    TidakJumpai,
    #[error("Had kadar dilampaui")]
    HadKadar,
}

impl IntoResponse for AppError {
    fn into_response(self) -> axum::response::Response {
        let (status, mesej) = match self {
            AppError::TidakJumpai => (StatusCode::NOT_FOUND, self.to_string()),
            AppError::HadKadar   => (StatusCode::TOO_MANY_REQUESTS, self.to_string()),
            _                    => (StatusCode::INTERNAL_SERVER_ERROR, self.to_string()),
        };
        (status, Json(serde_json::json!({"ralat": mesej}))).into_response()
    }
}

type BalasApp<T> = Result<(StatusCode, Json<T>), AppError>;

// ─── Types ────────────────────────────────────────────────────

#[derive(Debug, Serialize, Deserialize, Clone)]
struct Pekerja {
    id:       String,
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

#[derive(Debug, Serialize, Deserialize)]
struct DataSesi {
    id_pengguna: String,
    nama:        String,
    peranan:     Vec<String>,
}

// ─── App State ────────────────────────────────────────────────

#[derive(Clone)]
struct AppState {
    redis: Pool,
}

impl AppState {
    fn baru(pool: Pool) -> Self {
        AppState { redis: pool }
    }

    async fn conn(&self) -> Result<deadpool_redis::Connection, AppError> {
        Ok(self.redis.get().await?)
    }
}

// ─── Cache Helpers ────────────────────────────────────────────

async fn cache_set<T: Serialize>(
    conn: &mut impl AsyncCommands,
    kunci: &str,
    data: &T,
    ttl: u64,
) -> Result<(), AppError> {
    let json = serde_json::to_string(data)?;
    conn.set_ex(kunci, json, ttl).await?;
    Ok(())
}

async fn cache_get<T: for<'de> Deserialize<'de>>(
    conn: &mut impl AsyncCommands,
    kunci: &str,
) -> Result<Option<T>, AppError> {
    let json: Option<String> = conn.get(kunci).await?;
    match json {
        Some(j) => Ok(serde_json::from_str(&j)?),
        None    => Ok(None),
    }
}

// ─── Rate Limiter Middleware ───────────────────────────────────

async fn had_kadar_middleware(
    State(state): State<AppState>,
    req: Request,
    next: Next,
) -> Result<axum::response::Response, AppError> {
    let ip = req.headers()
        .get("x-forwarded-for")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("127.0.0.1");

    let mut conn = state.conn().await?;
    let kunci = format!("ratelimit:{}", ip);
    let kiraan: i64 = conn.incr(&kunci, 1).await?;

    if kiraan == 1 {
        conn.expire(&kunci, 60i64).await?; // reset setiap minit
    }

    if kiraan > 100 { // max 100 request per minit
        return Err(AppError::HadKadar);
    }

    Ok(next.run(req).await)
}

// ─── Handlers ─────────────────────────────────────────────────

async fn sihat(State(state): State<AppState>) -> BalasApp<Value> {
    let mut conn = state.conn().await?;
    let pong: String = redis::cmd("PING").query_async(&mut conn).await?;

    Ok((StatusCode::OK, Json(serde_json::json!({
        "status": "OK",
        "redis": pong,
        "servis": "KADA Cache API"
    }))))
}

async fn dapatkan_pekerja(
    State(state): State<AppState>,
    Path(id): Path<String>,
) -> BalasApp<Pekerja> {
    let mut conn = state.conn().await?;
    let kunci = format!("pekerja:{}", id);

    // Cuba cache dulu
    if let Some(pekerja) = cache_get::<Pekerja>(&mut conn, &kunci).await? {
        println!("🎯 Cache HIT: {}", id);
        return Ok((StatusCode::OK, Json(pekerja)));
    }

    // Cache MISS — simulate DB
    println!("💾 Cache MISS: {} — query DB", id);
    tokio::time::sleep(tokio::time::Duration::from_millis(30)).await;

    let pekerja = Pekerja {
        id:       id.clone(),
        nama:     format!("Pekerja {}", id),
        bahagian: "ICT".into(),
        gaji:     4500.0,
    };

    cache_set(&mut conn, &kunci, &pekerja, 300).await?;
    conn.sadd("senarai:pekerja", &id).await?;

    Ok((StatusCode::OK, Json(pekerja)))
}

async fn buat_pekerja(
    State(state): State<AppState>,
    Json(data): Json<BuatPekerja>,
) -> BalasApp<Pekerja> {
    let mut conn = state.conn().await?;

    let pekerja = Pekerja {
        id:       Uuid::new_v4().to_string(),
        nama:     data.nama,
        bahagian: data.bahagian,
        gaji:     data.gaji,
    };

    let kunci = format!("pekerja:{}", pekerja.id);
    cache_set(&mut conn, &kunci, &pekerja, 300).await?;
    conn.sadd("senarai:pekerja", &pekerja.id).await?;

    println!("✅ Pekerja baru: {}", pekerja.id);
    Ok((StatusCode::CREATED, Json(pekerja)))
}

async fn padam_pekerja(
    State(state): State<AppState>,
    Path(id): Path<String>,
) -> BalasApp<Value> {
    let mut conn = state.conn().await?;
    let kunci = format!("pekerja:{}", id);

    let dipadam: i64 = conn.del(&kunci).await?;
    if dipadam == 0 {
        return Err(AppError::TidakJumpai);
    }

    conn.srem("senarai:pekerja", &id).await?;
    println!("🗑️ Pekerja dipadamkan: {}", id);

    Ok((StatusCode::OK, Json(serde_json::json!({"mesej": "Berjaya dipadam"}))))
}

async fn stat_cache(State(state): State<AppState>) -> BalasApp<Value> {
    let mut conn = state.conn().await?;

    let jumlah_pekerja: i64 = conn.scard("senarai:pekerja").await?;
    let info: String = redis::cmd("INFO").arg("memory").query_async(&mut conn).await?;

    let memori = info.lines()
        .find(|l| l.starts_with("used_memory_human:"))
        .and_then(|l| l.split(':').nth(1))
        .unwrap_or("?")
        .trim()
        .to_string();

    Ok((StatusCode::OK, Json(serde_json::json!({
        "jumlah_pekerja_cached": jumlah_pekerja,
        "memori_redis": memori,
    }))))
}

// ─── Main ──────────────────────────────────────────────────────

#[tokio::main]
async fn main() {
    let pool = Config::from_url("redis://127.0.0.1:6379")
        .create_pool(Some(Runtime::Tokio1))
        .expect("Gagal buat Redis pool");

    // Test sambungan
    {
        let mut conn = pool.get().await.expect("Gagal sambung Redis");
        let pong: String = redis::cmd("PING").query_async(&mut conn).await.unwrap();
        println!("Redis: {}", pong);
    }

    let state = AppState::baru(pool);

    let app = Router::new()
        .route("/sihat",          get(sihat))
        .route("/stat",           get(stat_cache))
        .route("/pekerja",        post(buat_pekerja))
        .route("/pekerja/:id",    get(dapatkan_pekerja).delete(padam_pekerja))
        .layer(middleware::from_fn_with_state(state.clone(), had_kadar_middleware))
        .with_state(state);

    let pendengar = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();

    println!("{'═'*50}");
    println!("  KADA Caching Layer");
    println!("  http://localhost:3000");
    println!("{'═'*50}");
    println!("  Endpoints:");
    println!("  GET  /sihat");
    println!("  GET  /stat");
    println!("  POST /pekerja");
    println!("  GET  /pekerja/:id");
    println!("  DEL  /pekerja/:id");
    println!("{'═'*50}");

    axum::serve(pendengar, app).await.unwrap();
}
```

---

# 📋 Rujukan Pantas — Redis Cheat Sheet

## Data Structures & Commands

```
STRING:    SET, GET, MSET, MGET, INCR, DECR, APPEND, STRLEN
HASH:      HSET, HGET, HMSET, HMGET, HGETALL, HDEL, HEXISTS, HLEN, HINCR
LIST:      LPUSH, RPUSH, LPOP, RPOP, LRANGE, LLEN, LINDEX
SET:       SADD, SREM, SMEMBERS, SISMEMBER, SCARD, SUNION, SINTER, SDIFF
ZSET:      ZADD, ZRANGE, ZREVRANGE, ZSCORE, ZRANK, ZREVRANK, ZCARD, ZREM
GENERIC:   DEL, EXISTS, TYPE, TTL, PTTL, EXPIRE, PERSIST, KEYS
```

## Rust Types Mapping

```rust
// SET → simpan
conn.set("kunci", "string")?;      // String
conn.set("kunci", 42i64)?;         // Integer
conn.set("kunci", 3.14f64)?;       // Float
conn.set_ex("kunci", "val", 60)?;  // Dengan TTL

// GET → ambil
let s: String       = conn.get("kunci")?;
let n: i64          = conn.get("kunci")?;
let opt: Option<String> = conn.get("kunci")?; // None kalau tidak ada
```

## Connection Pool

```rust
use deadpool_redis::{Config, Pool, Runtime};

let pool = Config::from_url("redis://127.0.0.1:6379")
    .create_pool(Some(Runtime::Tokio1))?;

let mut conn = pool.get().await?; // ambil dari pool
// ... guna conn ...
// conn auto-kembali ke pool bila di-drop
```

## Pattern Penting

```
Cache Aside:    semak cache → miss → query DB → simpan cache
Write Through:  tulis DB + cache serentak
Write Behind:   tulis cache dahulu, DB kemudian (async)
Cache Warm:     preload cache semasa startup
TTL Pattern:    set TTL supaya data tidak lapuk
Invalidation:   DEL cache bila data berubah
```

## Key Naming Convention

```
Gunakan : sebagai separator
  pekerja:1234                  → hash untuk pekerja ID 1234
  sesi:uuid                     → session data
  cache:pekerja:1234            → cache untuk pekerja
  ratelimit:127.0.0.1           → rate limit untuk IP
  lock:operasi_kritikal         → distributed lock
  stat:login:2024-01-15         → statistik
  queue:tugas:ICT               → job queue untuk ICT
```

---

## 🏆 Cabaran Akhir

Cuba implement salah satu:

1. **Leaderboard Sistem** — simpan skor pemain dengan ZADD, tunjuk Top 10 dengan ZREVRANGE
2. **Job Queue** — LPUSH untuk tambah kerja, BRPOP untuk proses kerja (blocking pop)
3. **Distributed Lock** — SETNX untuk lock, expire untuk auto-release kalau process crash
4. **Real-time Chat** — Pub/Sub untuk hantar/terima mesej, LIST untuk store history
5. **Cache Warming** — script yang preload data popular dari DB ke Redis semasa startup

---

*Redis + Rust = Caching yang selamat, laju, dan reliable.*
*Ownership Rust memastikan sambungan Redis tidak bocor,*
*async Tokio membolehkan ribuan request serentak.* 🦀
