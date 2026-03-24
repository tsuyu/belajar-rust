# Prepared Statements: MySQLi (PHP) → SQLx (Rust)

## Persediaan

### PHP
```php
$mysqli = new mysqli('hostname', 'username', 'password', 'database');
```

### Rust (sqlx)
```rust
// Cargo.toml
// sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "mysql", "macros"] }
// tokio = { version = "1", features = ["full"] }

use sqlx::MySqlPool;

let pool = MySqlPool::connect("mysql://username:password@hostname/database").await?;
```

---

## Schema Rujukan

```sql
CREATE TABLE users (
    id    INT AUTO_INCREMENT PRIMARY KEY,
    name  VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL
);
```

---

## SELECT — Satu Baris

### PHP
```php
$stmt = $mysqli->prepare('SELECT name, email FROM users WHERE id = ?');
$userId = 1;
$stmt->bind_param('i', $userId);
$stmt->execute();
$stmt->store_result();
$stmt->bind_result($name, $email);
$stmt->fetch();

echo $name;
echo $email;
```

### Rust
```rust
#[derive(sqlx::FromRow, Debug)]
struct User {
    name:  String,
    email: String,
}

let user_id: i32 = 1;

let user = sqlx::query_as::<_, User>(
    "SELECT name, email FROM users WHERE id = ?"
)
.bind(user_id)
.fetch_one(&pool)       // error jika tiada row
.await?;

println!("{}", user.name);
println!("{}", user.email);
```

> `fetch_one` → panic/error jika tiada hasil. Guna `fetch_optional` jika row mungkin tiada.

---

## SELECT — Pelbagai Baris

### PHP
```php
$stmt = $mysqli->prepare('SELECT name, email FROM users');
$stmt->execute();
$stmt->store_result();
$stmt->bind_result($name, $email);

while ($stmt->fetch()) {
    echo $name;
    echo $email;
}
```

### Rust
```rust
let users = sqlx::query_as::<_, User>(
    "SELECT name, email FROM users"
)
.fetch_all(&pool)
.await?;

for user in &users {
    println!("{}", user.name);
    println!("{}", user.email);
}
```

> Streaming rows (jimat memori untuk dataset besar):
```rust
use futures::TryStreamExt;

let mut rows = sqlx::query_as::<_, User>("SELECT name, email FROM users")
    .fetch(&pool);

while let Some(user) = rows.try_next().await? {
    println!("{}", user.name);
}
```

---

## SELECT — Kira Bilangan Baris

### PHP
```php
$stmt = $mysqli->prepare('SELECT name, email FROM users');
$stmt->execute();
$stmt->store_result();
echo $stmt->num_rows;
```

### Rust
```rust
let count: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM users")
    .fetch_one(&pool)
    .await?;

println!("{}", count);
```

---

## SELECT — Gaya get_result() (fetch_assoc)

### PHP
```php
$stmt = $mysqli->prepare('SELECT name, email FROM users WHERE id > ?');
$greaterThan = 1;
$stmt->bind_param('i', $greaterThan);
$stmt->execute();
$result = $stmt->get_result();

while ($row = $result->fetch_assoc()) {
    echo $row['name'];
    echo $row['email'];
}
```

### Rust
```rust
let greater_than: i32 = 1;

let users = sqlx::query_as::<_, User>(
    "SELECT name, email FROM users WHERE id > ?"
)
.bind(greater_than)
.fetch_all(&pool)
.await?;

for user in &users {
    println!("{} {}", user.name, user.email);
}
```

---

## SELECT — Wildcard LIKE

### PHP
```php
$stmt = $mysqli->prepare('SELECT name, email FROM users WHERE name LIKE ?');
$like = 'a%';
$stmt->bind_param('s', $like);
$stmt->execute();
// ...
```

### Rust
```rust
let pattern = "a%".to_string();

let users = sqlx::query_as::<_, User>(
    "SELECT name, email FROM users WHERE name LIKE ?"
)
.bind(&pattern)
.fetch_all(&pool)
.await?;
```

---

## SELECT — IN (Senarai ID)

### PHP
```php
$userIdArray = [1, 2, 3, 4];
$questionMarks = implode(',', array_fill(0, count($userIdArray), '?'));
$dataTypes = str_repeat('i', count($userIdArray));

$stmt = $mysqli->prepare("SELECT name, email FROM users WHERE id IN ($questionMarks)");
$stmt->bind_param($dataTypes, ...$userIdArray);
$stmt->execute();
// ...
```

### Rust
> sqlx tidak support `IN (?)` dengan slice secara langsung. Guna query builder atau format manual.

```rust
let ids: Vec<i32> = vec![1, 2, 3, 4];

// Bina placeholder: ?, ?, ?, ?
let placeholders = ids
    .iter()
    .enumerate()
    .map(|(i, _)| format!("?"))
    .collect::<Vec<_>>()
    .join(", ");

let sql = format!("SELECT name, email FROM users WHERE id IN ({})", placeholders);

let mut query = sqlx::query_as::<_, User>(&sql);
for id in &ids {
    query = query.bind(id);
}

let users = query.fetch_all(&pool).await?;
```

> Alternatif bersih: guna crate [`sqlx-in`](https://crates.io/crates/sqlx-in) atau filter dalam Rust selepas `fetch_all`.

---

## SELECT — Had & Ofset

### PHP
```php
$stmt = $mysqli->prepare("SELECT name, email FROM users LIMIT ? OFFSET ?");
$limit = 2;
$offset = 1;
$stmt->bind_param('ii', $limit, $offset);
```

### Rust
```rust
let limit: i64  = 2;
let offset: i64 = 1;

let users = sqlx::query_as::<_, User>(
    "SELECT name, email FROM users LIMIT ? OFFSET ?"
)
.bind(limit)
.bind(offset)
.fetch_all(&pool)
.await?;
```

---

## SELECT — Antara Nilai (BETWEEN)

### PHP
```php
$stmt = $mysqli->prepare("SELECT name, email FROM users WHERE id BETWEEN ? AND ?");
$betweenStart = 2;
$betweenEnd   = 4;
$stmt->bind_param('ii', $betweenStart, $betweenEnd);
```

### Rust
```rust
let between_start: i32 = 2;
let between_end: i32   = 4;

let users = sqlx::query_as::<_, User>(
    "SELECT name, email FROM users WHERE id BETWEEN ? AND ?"
)
.bind(between_start)
.bind(between_end)
.fetch_all(&pool)
.await?;
```

---

## INSERT — Satu Baris

### PHP
```php
$stmt = $mysqli->prepare('INSERT INTO users (name, email) VALUES (?,?)');
$name  = 'Akhil';
$email = 'akhil@example.com';
$stmt->bind_param('ss', $name, $email);
$stmt->execute();
```

### Rust
```rust
let name  = "Akhil";
let email = "akhil@example.com";

sqlx::query("INSERT INTO users (name, email) VALUES (?, ?)")
    .bind(name)
    .bind(email)
    .execute(&pool)
    .await?;
```

---

## INSERT — Dapatkan last_insert_id

### PHP
```php
$stmt->execute();
echo 'Your account id is ' . $stmt->insert_id;
```

### Rust
```rust
let result = sqlx::query("INSERT INTO users (name, email) VALUES (?, ?)")
    .bind("Akhil")
    .bind("akhil@example.com")
    .execute(&pool)
    .await?;

let insert_id = result.last_insert_id();
println!("Your account id is {}", insert_id);
```

---

## INSERT — Pukal / Pelbagai Baris

### PHP
```php
$newUsers = [
    ['sulliops',  'sulliops@example.com'],
    ['infinity',  'infinity@example.com'],
    ['aivarasco', 'aivarasco@example.com'],
];

$stmt = $mysqli->prepare('INSERT INTO users (name, email) VALUES (?,?)');

foreach ($newUsers as $user) {
    $stmt->bind_param('ss', $user[0], $user[1]);
    $stmt->execute();
    echo "{$user[0]}'s account id is {$stmt->insert_id}";
}
```

### Rust
```rust
let new_users = vec![
    ("sulliops",  "sulliops@example.com"),
    ("infinity",  "infinity@example.com"),
    ("aivarasco", "aivarasco@example.com"),
];

for (name, email) in &new_users {
    let result = sqlx::query("INSERT INTO users (name, email) VALUES (?, ?)")
        .bind(name)
        .bind(email)
        .execute(&pool)
        .await?;

    println!("{}'s account id is {}", name, result.last_insert_id());
}
```

> Untuk bulk insert berprestasi tinggi, guna transaction:
```rust
let mut tx = pool.begin().await?;

for (name, email) in &new_users {
    sqlx::query("INSERT INTO users (name, email) VALUES (?, ?)")
        .bind(name)
        .bind(email)
        .execute(&mut *tx)
        .await?;
}

tx.commit().await?;
```

---

## UPDATE

### PHP
```php
$stmt = $mysqli->prepare('UPDATE users SET email = ? WHERE id = ? LIMIT 1');
$email = 'newemail@example.com';
$id    = 2;
$stmt->bind_param('si', $email, $id);
$stmt->execute();
```

### Rust
```rust
let email = "newemail@example.com";
let id: i32 = 2;

sqlx::query("UPDATE users SET email = ? WHERE id = ? LIMIT 1")
    .bind(email)
    .bind(id)
    .execute(&pool)
    .await?;
```

---

## UPDATE — Baris Terjejas

### PHP
```php
$stmt->execute();
echo $stmt->affected_rows; // 1
```

### Rust
```rust
let result = sqlx::query("UPDATE users SET email = ? WHERE name = ? LIMIT 1")
    .bind("newemail@example.com")
    .bind("teodor")
    .execute(&pool)
    .await?;

println!("{}", result.rows_affected()); // 1
```

---

## DELETE

### PHP
```php
$stmt = $mysqli->prepare('DELETE FROM users WHERE id = ?');
$userId = 4;
$stmt->bind_param('i', $userId);
$stmt->execute();
echo $stmt->affected_rows;
```

### Rust
```rust
let user_id: i32 = 4;

let result = sqlx::query("DELETE FROM users WHERE id = ?")
    .bind(user_id)
    .execute(&pool)
    .await?;

println!("{}", result.rows_affected());
```

---

## Pengendalian Ralat

### PHP
```php
// Preparation failure
$stmt = $mysqli->prepare('SELECT * FROM no_table WHERE id = ?');
echo $mysqli->error;

// Execution failure
if (!$stmt->execute()) {
    echo $stmt->error;
}
```

### Rust
```rust
use sqlx::Error as SqlxError;

// sqlx pulangkan Result — guna ? atau match
match sqlx::query("SELECT * FROM no_table WHERE id = ?")
    .bind(1)
    .fetch_one(&pool)
    .await
{
    Ok(row) => { /* proses row */ }
    Err(SqlxError::RowNotFound) => {
        eprintln!("Tiada row dijumpai");
    }
    Err(e) => {
        eprintln!("DB error: {}", e);
    }
}
```

> Pattern `AppError` (ikut skill php-to-rust):
```rust
// src/error.rs
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
    #[error("Not found: {0}")]
    NotFound(String),
}

pub type Result<T> = std::result::Result<T, AppError>;
```

---

## Rujukan Pantas — PHP MySQLi → Rust sqlx

| PHP MySQLi | Rust sqlx |
|---|---|
| `$mysqli->prepare(sql)` | `sqlx::query(sql)` / `query_as::<_, T>(sql)` |
| `bind_param('i', $val)` | `.bind(val)` |
| `$stmt->execute()` | `.execute(&pool).await?` |
| `$stmt->fetch_one()` | `.fetch_one(&pool).await?` |
| `$stmt->store_result()` | tidak perlu — sqlx handle async |
| `$stmt->bind_result(...)` | derive `sqlx::FromRow` pada struct |
| `while ($stmt->fetch())` | `.fetch_all()` + `for` / `.fetch()` stream |
| `$stmt->num_rows` | `fetch_all().len()` atau `COUNT(*)` query |
| `$stmt->insert_id` | `result.last_insert_id()` |
| `$stmt->affected_rows` | `result.rows_affected()` |
| `$mysqli->error` | `Err(e) => eprintln!("{}", e)` |
| `get_result()->fetch_assoc()` | `.fetch_all()` → iterate struct |

---

## Pemetaan Jenis MySQL ke Rust

| Jenis MySQL | Jenis Rust | Nota |
|---|---|---|
| `TINYINT` | `i8` | |
| `TINYINT UNSIGNED` | `u8` | |
| `SMALLINT` | `i16` | |
| `SMALLINT UNSIGNED` | `u16` | |
| `INT` / `MEDIUMINT` | `i32` | |
| `INT UNSIGNED` | `u32` | |
| `BIGINT` | `i64` | |
| `BIGINT UNSIGNED` | `u64` | |
| `TINYINT(1)` / `BOOLEAN` | `bool` | MySQL simpan sebagai 0/1 |
| `FLOAT` | `f32` | |
| `DOUBLE` | `f64` | |
| `DECIMAL` / `NUMERIC` | `rust_decimal::Decimal` | Perlukan crate `rust_decimal` |
| `VARCHAR` / `TEXT` / `CHAR` | `String` | |
| `BLOB` / `BINARY` / `VARBINARY` | `Vec<u8>` | |
| `DATE` | `chrono::NaiveDate` | Perlukan feature `chrono` |
| `TIME` | `chrono::NaiveTime` | Perlukan feature `chrono` |
| `DATETIME` | `chrono::NaiveDateTime` | Perlukan feature `chrono` |
| `TIMESTAMP` | `chrono::DateTime<Utc>` | Perlukan feature `chrono` |
| `JSON` | `serde_json::Value` | Perlukan feature `json` |
| `NULL` / nullable column | `Option<T>` | Contoh: `Option<String>` |

### Contoh Struct dengan Pelbagai Jenis

```rust
use chrono::NaiveDateTime;

#[derive(sqlx::FromRow, Debug)]
struct Product {
    id:         i32,
    name:       String,
    price:      f64,
    stock:      Option<i32>,        // nullable INT
    is_active:  bool,               // TINYINT(1)
    created_at: NaiveDateTime,      // DATETIME
}
```

### Cargo.toml dengan Chrono & JSON

```toml
sqlx = { version = "0.7", features = [
    "runtime-tokio-rustls",
    "mysql",
    "macros",
    "chrono",        # untuk Date/Time
    "json",          # untuk JSON
] }
chrono = { version = "0.4", features = ["serde"] }
```

---

## Cargo.toml Minimum (Asas)

```toml
[dependencies]
sqlx   = { version = "0.7", features = ["runtime-tokio-rustls", "mysql", "macros"] }
tokio  = { version = "1", features = ["full"] }
thiserror = "1"
futures = "0.3"   # untuk TryStreamExt (streaming rows)
```