# 🦀 CRUD with SQLx + Askama using Rust

> Versi Rust bagi tutorial: [CRUD with MySQLi Prepared Statement using PHP](https://phppot.com/php/crud-with-mysqli-prepared-statement-using-php/)
> Stack: **Axum** (web) + **SQLx** (database) + **Askama** (template engine)

---

## Perbandingan Konsep PHP → Rust

| PHP (MySQLi) | Rust (SQLx + Axum + Askama) |
|---|---|
| `$conn->prepare("INSERT ...")` | `sqlx::query!("INSERT ...")` |
| `$sql->bind_param("sss", ...)` | `.bind(val1).bind(val2).bind(val3)` |
| `$sql->execute()` | `.execute(&pool).await?` |
| `$result->fetch_assoc()` | `sqlx::query_as!(Struct, ...).fetch_all()` |
| `header('location:index.php')` | `Redirect::to("/")` |
| `try/catch` (implisit) | `Result<T, AppError>` + `?` |
| `$_POST['name']` | `Form(payload): Form<CreateEmployee>` |
| `$_GET['id']` | `Path(id): Path<u32>` |
| `echo $row["name"]` dalam `.php` fail | `{{ employee.name }}` dalam `.html` template |
| `<?php foreach($list as $e): ?>` | `{% for e in employees %}` |
| HTML + PHP bercampur dalam satu fail | Template `.html` terpisah, compile-time checked |

---

## Struktur Projek

```
crud-employee/
├── Cargo.toml
├── .env
├── migrations/
│   └── 001_create_employee.sql
├── templates/                  ← BARU: Askama template files
│   ├── base.html               ← layout induk (extends)
│   ├── list.html               ← senarai pekerja
│   ├── create.html             ← borang tambah
│   └── edit.html               ← borang edit
└── src/
    ├── main.rs
    ├── error.rs
    ├── db.rs
    ├── models.rs
    ├── templates.rs            ← BARU: Askama template structs
    └── handlers.rs
```

> 💡 **Askama vs PHP templating:**
> PHP campur HTML + logic dalam fail `.php` yang sama.
> Askama pisahkan — logic dalam `.rs`, markup dalam `.html`.
> Template di-**compile semasa build** — typo dalam template = compile error, bukan runtime error!

---

## Cargo.toml

```toml
[package]
name = "crud-employee"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = { version = "0.7", features = ["form"] }
sqlx = { version = "0.7", features = ["mysql", "runtime-tokio", "macros"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
thiserror = "1"
dotenvy = "0.15"
tracing = "0.1"
tracing-subscriber = "0.3"
askama = { version = "0.12", features = ["with-axum"] }  # ← BARU
askama_axum = "0.4"                                       # ← BARU: IntoResponse untuk Axum
```

---

## .env

```env
DATABASE_URL=mysql://root:password@localhost/crud_db
SERVER_ADDR=0.0.0.0:3000
```

---

## migrations/001_create_employee.sql

```sql
-- PHP asal guna table: tbl_emp_details
-- Rust guna schema yang sama supaya query konsisten

CREATE TABLE IF NOT EXISTS tbl_emp_details (
    id         INT AUTO_INCREMENT PRIMARY KEY,
    department VARCHAR(100) NOT NULL,
    name       VARCHAR(100) NOT NULL,
    email      VARCHAR(150) NOT NULL
);
```

---

## templates/base.html

```html
<!DOCTYPE html>
<html lang="ms">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}Pengurusan Pekerja{% endblock %}</title>
    <style>
        body { font-family: sans-serif; max-width: 800px; margin: 40px auto; padding: 0 20px; }
        table { width: 100%; border-collapse: collapse; }
        th, td { border: 1px solid #ccc; padding: 8px 12px; text-align: left; }
        th { background: #f0f0f0; }
        a { color: #0066cc; }
        .btn { padding: 6px 14px; background: #0066cc; color: white;
               border: none; cursor: pointer; text-decoration: none; }
        .msg-success { color: green; padding: 8px; background: #e6ffe6; margin-bottom: 12px; }
        .msg-error   { color: red;   padding: 8px; background: #ffe6e6; margin-bottom: 12px; }
    </style>
</head>
<body>
    <h1>🦀 Pengurusan Pekerja</h1>
    <hr>
    {% block content %}{% endblock %}
</body>
</html>
```

---

## templates/list.html

```html
{# PHP asal: index.php — papar senarai + link edit/padam #}
{% extends "base.html" %}

{% block title %}Senarai Pekerja{% endblock %}

{% block content %}
<a href="/add" class="btn">+ Tambah Pekerja Baru</a>
<br><br>

{% if employees.is_empty() %}
    <p>Tiada pekerja dalam sistem.</p>
{% else %}
<table>
    <thead>
        <tr>
            <th>Jabatan</th>
            <th>Nama</th>
            <th>Email</th>
            <th colspan="2">Tindakan</th>
        </tr>
    </thead>
    <tbody>
        {# PHP: while($row = $result->fetch_assoc()) #}
        {% for emp in employees %}
        <tr>
            <td>{{ emp.department }}</td>
            <td>{{ emp.name }}</td>
            <td>{{ emp.email }}</td>
            <td><a href="/edit/{{ emp.id }}">✏️ Edit</a></td>
            <td>
                <a href="/delete/{{ emp.id }}"
                   onclick="return confirm('Pasti nak padam {{ emp.name }}?')">
                   🗑️ Padam
                </a>
            </td>
        </tr>
        {% endfor %}
    </tbody>
</table>
{% endif %}
{% endblock %}
```

---

## templates/create.html

```html
{# PHP asal: add.php — borang tambah pekerja baru #}
{% extends "base.html" %}

{% block title %}Tambah Pekerja{% endblock %}

{% block content %}
<a href="/">← Balik Senarai</a>
<h2>Tambah Pekerja Baru</h2>

{# PHP: if(!empty($error_message)) { echo $error_message; } #}
{% if let Some(msg) = error_message %}
    <div class="msg-error">{{ msg }}</div>
{% endif %}

{# PHP: <form name="frmUser" method="post" action=""> #}
<form method="post" action="/add">
    <table>
        <tr>
            <td><label>Jabatan</label></td>
            <td><input type="text" name="department" required></td>
        </tr>
        <tr>
            <td><label>Nama</label></td>
            <td><input type="text" name="name" required></td>
        </tr>
        <tr>
            <td><label>Email</label></td>
            <td><input type="text" name="email" required></td>
        </tr>
        <tr>
            <td colspan="2">
                <button type="submit" class="btn">Simpan</button>
            </td>
        </tr>
    </table>
</form>
{% endblock %}
```

---

## templates/edit.html

```html
{# PHP asal: edit.php — borang kemaskini pekerja #}
{% extends "base.html" %}

{% block title %}Edit Pekerja{% endblock %}

{% block content %}
<a href="/">← Balik Senarai</a>
<h2>Edit Pekerja</h2>

{# PHP: if(!empty($success_message)) { echo $success_message; } #}
{% if let Some(msg) = success_message %}
    <div class="msg-success">{{ msg }}</div>
{% endif %}

{# PHP: value="<?php echo $row['department']?>" #}
<form method="post" action="/edit/{{ employee.id }}">
    <table>
        <tr>
            <td><label>Jabatan</label></td>
            <td><input type="text" name="department" value="{{ employee.department }}"></td>
        </tr>
        <tr>
            <td><label>Nama</label></td>
            <td><input type="text" name="name" value="{{ employee.name }}"></td>
        </tr>
        <tr>
            <td><label>Email</label></td>
            <td><input type="text" name="email" value="{{ employee.email }}"></td>
        </tr>
        <tr>
            <td colspan="2">
                <button type="submit" class="btn">Kemaskini</button>
            </td>
        </tr>
    </table>
</form>
{% endblock %}
```

---

## src/templates.rs

```rust
// Setiap struct = satu template fail
// #[template(path = "...")] → lokasi fail dalam /templates/
// Askama generate render() method semasa compile — bukan runtime

use askama::Template;
use crate::models::Employee;

// PHP: index.php — $employees array dipass ke HTML
#[derive(Template)]
#[template(path = "list.html")]
pub struct ListTemplate {
    pub employees: Vec<Employee>,
}

// PHP: add.php (GET) — borang kosong
#[derive(Template)]
#[template(path = "create.html")]
pub struct CreateTemplate {
    pub error_message: Option<String>,  // PHP: $error_message
}

// PHP: edit.php (GET) — borang dengan nilai sedia ada
#[derive(Template)]
#[template(path = "edit.html")]
pub struct EditTemplate {
    pub employee:        Employee,       // PHP: $row
    pub success_message: Option<String>, // PHP: $success_message
}
```

---

## src/error.rs

```rust
// PHP: tiada explicit error type — Rust wajib ada
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("Employee tidak dijumpai: id={0}")]
    NotFound(u32),

    #[error("Input tidak sah: {0}")]
    Validation(String),

    // BARU: Askama render error
    #[error("Template error: {0}")]
    Template(#[from] askama::Error),
}

// Supaya Axum boleh return AppError sebagai HTTP response
impl axum::response::IntoResponse for AppError {
    fn into_response(self) -> axum::response::Response {
        use axum::http::StatusCode;
        let (status, msg) = match &self {
            AppError::NotFound(_)    => (StatusCode::NOT_FOUND, self.to_string()),
            AppError::Validation(_)  => (StatusCode::BAD_REQUEST, self.to_string()),
            AppError::Template(_)    => (StatusCode::INTERNAL_SERVER_ERROR, self.to_string()),
            AppError::Database(_)    => (StatusCode::INTERNAL_SERVER_ERROR, self.to_string()),
        };
        (status, msg).into_response()
    }
}

pub type Result<T> = std::result::Result<T, AppError>;
```

---

```rust
// PHP: tiada explicit error type — Rust wajib ada
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("Employee tidak dijumpai: id={0}")]
    NotFound(u32),

    #[error("Input tidak sah: {0}")]
    Validation(String),
}

// Supaya Axum boleh return AppError sebagai HTTP response
impl axum::response::IntoResponse for AppError {
    fn into_response(self) -> axum::response::Response {
        use axum::http::StatusCode;
        let (status, msg) = match &self {
            AppError::NotFound(_)    => (StatusCode::NOT_FOUND, self.to_string()),
            AppError::Validation(_)  => (StatusCode::BAD_REQUEST, self.to_string()),
            AppError::Database(_)    => (StatusCode::INTERNAL_SERVER_ERROR, self.to_string()),
        };
        (status, msg).into_response()
    }
}

pub type Result<T> = std::result::Result<T, AppError>;
```

---

## src/models.rs

```rust
use serde::{Deserialize, Serialize};

// PHP: array dari $result->fetch_assoc()
// Rust: strongly-typed struct
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Employee {
    pub id:         i32,
    pub department: String,
    pub name:       String,
    pub email:      String,
}

// PHP: $_POST pada create form
#[derive(Debug, Deserialize)]
pub struct CreateEmployee {
    pub department: String,
    pub name:       String,
    pub email:      String,
}

// PHP: $_POST pada edit form
#[derive(Debug, Deserialize)]
pub struct UpdateEmployee {
    pub department: String,
    pub name:       String,
    pub email:      String,
}
```

---

## src/db.rs

```rust
use sqlx::mysql::MySqlPool;

// PHP: require_once("db.php") + $conn = new mysqli(...)
// Rust: connection pool dikongsi seluruh app
pub async fn connect(database_url: &str) -> Result<MySqlPool, sqlx::Error> {
    MySqlPool::connect(database_url).await
}
```

---

## src/handlers.rs

```rust
use axum::{
    extract::{Form, Path, State},
    response::Redirect,
};
use askama_axum::IntoResponse;
use sqlx::MySqlPool;
use crate::{
    error::{AppError, Result},
    models::*,
    templates::*,  // ListTemplate, CreateTemplate, EditTemplate
};

// ─────────────────────────────────────────────────
// READ — Senarai semua pekerja
// PHP: index.php → SELECT * + foreach loop dalam HTML
// Rust: query DB → isi ListTemplate → Askama render
// ─────────────────────────────────────────────────
pub async fn list_employees(
    State(pool): State<MySqlPool>,
) -> Result<impl IntoResponse> {

    let employees = sqlx::query_as!(
        Employee,
        "SELECT id, department, name, email FROM tbl_emp_details"
    )
    .fetch_all(&pool)
    .await?;

    // PHP: extract($data); include 'index.php';
    // Rust: bina template struct, Askama render jadi HTML
    Ok(ListTemplate { employees })
}

// ─────────────────────────────────────────────────
// CREATE (GET) — Papar borang tambah kosong
// PHP: add.php (tanpa $_POST)
// ─────────────────────────────────────────────────
pub async fn show_create_form() -> impl IntoResponse {
    CreateTemplate { error_message: None }
}

// ─────────────────────────────────────────────────
// CREATE (POST) — Simpan rekod baru
// PHP:
//   $sql->bind_param("sss", $department, $name, $email);
//   if($sql->execute()) { $success_message = "Added Successfully"; }
// ─────────────────────────────────────────────────
pub async fn create_employee(
    State(pool): State<MySqlPool>,
    Form(payload): Form<CreateEmployee>,
) -> Result<impl IntoResponse> {

    if payload.name.trim().is_empty() {
        // PHP: $error_message = "Problem in Adding New Record";
        // Rust: re-render form dengan error message
        return Ok(CreateTemplate {
            error_message: Some("Nama tidak boleh kosong".into()),
        }.into_response());
    }

    sqlx::query!(
        "INSERT INTO tbl_emp_details (department, name, email) VALUES (?, ?, ?)",
        payload.department,
        payload.name,
        payload.email,
    )
    .execute(&pool)
    .await?;

    tracing::info!("Pekerja baru ditambah: {}", payload.name);

    // PHP: header('location:index.php')
    Ok(Redirect::to("/").into_response())
}

// ─────────────────────────────────────────────────
// UPDATE (GET) — Papar borang edit dengan nilai sedia ada
// PHP: edit.php — SELECT WHERE id=? → isi value="" dalam form
// ─────────────────────────────────────────────────
pub async fn show_edit_form(
    State(pool): State<MySqlPool>,
    Path(id): Path<u32>,
) -> Result<impl IntoResponse> {

    let employee = sqlx::query_as!(
        Employee,
        "SELECT id, department, name, email FROM tbl_emp_details WHERE id = ?",
        id
    )
    .fetch_optional(&pool)
    .await?
    .ok_or(AppError::NotFound(id))?;

    // PHP: extract($row); include 'edit.php';
    Ok(EditTemplate {
        employee,
        success_message: None,
    })
}

// ─────────────────────────────────────────────────
// UPDATE (POST) — Simpan perubahan
// PHP:
//   $sql->bind_param("sssi", $department, $name, $email, $_GET["id"]);
//   if($sql->execute()) { $success_message = "Edited Successfully"; }
// ─────────────────────────────────────────────────
pub async fn update_employee(
    State(pool): State<MySqlPool>,
    Path(id): Path<u32>,
    Form(payload): Form<UpdateEmployee>,
) -> Result<impl IntoResponse> {

    let result = sqlx::query!(
        "UPDATE tbl_emp_details SET department = ?, name = ?, email = ? WHERE id = ?",
        payload.department,
        payload.name,
        payload.email,
        id,
    )
    .execute(&pool)
    .await?;

    if result.rows_affected() == 0 {
        return Err(AppError::NotFound(id));
    }

    tracing::info!("Pekerja id={} dikemaskini", id);

    // PHP: $success_message = "Edited Successfully" → re-fetch & render edit form
    // Rust: re-fetch employee → render EditTemplate dengan success message
    let employee = sqlx::query_as!(
        Employee,
        "SELECT id, department, name, email FROM tbl_emp_details WHERE id = ?",
        id
    )
    .fetch_one(&pool)
    .await?;

    Ok(EditTemplate {
        employee,
        success_message: Some("Kemaskini berjaya!".into()),
    }.into_response())
}

// ─────────────────────────────────────────────────
// DELETE — Padam rekod
// PHP: delete.php → DELETE WHERE id=? → header('location:index.php')
// ─────────────────────────────────────────────────
pub async fn delete_employee(
    State(pool): State<MySqlPool>,
    Path(id): Path<u32>,
) -> Result<impl IntoResponse> {

    sqlx::query!(
        "DELETE FROM tbl_emp_details WHERE id = ?",
        id
    )
    .execute(&pool)
    .await?;

    tracing::info!("Pekerja id={} dipadam", id);
    Ok(Redirect::to("/"))
}
```

---

## src/main.rs

```rust
use axum::{routing::{get, post}, Router};
use sqlx::MySqlPool;
use std::env;

mod db;
mod error;
mod handlers;
mod models;
mod templates;  // ← BARU

#[tokio::main]
async fn main() {
    // Setup logging (PHP: tiada equivalent — Rust encourage structured logging)
    tracing_subscriber::fmt::init();

    // Load .env (PHP: nilai DB biasanya hardcode dalam db.php)
    dotenvy::dotenv().ok();

    let database_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL mesti ada dalam .env");

    let server_addr = env::var("SERVER_ADDR")
        .unwrap_or_else(|_| "0.0.0.0:3000".to_string());

    // Connection pool (PHP: buat connection baru tiap request)
    let pool: MySqlPool = db::connect(&database_url)
        .await
        .expect("Gagal connect database");

    tracing::info!("Database connected ✓");

    // Route mapping (PHP: setiap fail .php = satu route)
    //   index.php  → GET  /
    //   add.php    → GET  /add  +  POST /add
    //   edit.php   → GET  /edit/:id  +  POST /edit/:id
    //   delete.php → GET  /delete/:id
    let app = Router::new()
        .route("/",           get(handlers::list_employees))
        .route("/add",        get(handlers::show_create_form))
        .route("/add",        post(handlers::create_employee))
        .route("/edit/:id",   get(handlers::show_edit_form))
        .route("/edit/:id",   post(handlers::update_employee))
        .route("/delete/:id", get(handlers::delete_employee))
        .with_state(pool);

    let listener = tokio::net::TcpListener::bind(&server_addr)
        .await
        .expect("Gagal bind server address");

    tracing::info!("Server berjalan di http://{}", server_addr);

    axum::serve(listener, app)
        .await
        .expect("Server error");
}
```

---

## Cara Jalankan

```bash
# 1. Buat database
mysql -u root -p -e "CREATE DATABASE crud_db;"
mysql -u root -p crud_db < migrations/001_create_employee.sql

# 2. Setup .env
cp .env.example .env
# edit DATABASE_URL

# 3. Run
cargo run

# 4. Buka browser
# http://localhost:3000
```

---

## Perbezaan Utama PHP vs Rust

| Aspek | PHP (MySQLi) | Rust (SQLx + Askama) |
|---|---|---|
| **Connection** | Baru tiap request | Connection pool dikongsi |
| **Prepared stmt** | `bind_param("sss", ...)` | `.bind(v1).bind(v2)` |
| **Fetch row** | `fetch_assoc()` → array | `query_as!(Struct)` → typed struct |
| **Null safety** | `$row["id"]` boleh null tanpa warning | `Option<T>` — wajib handle |
| **Error** | `if($sql->execute())` check manual | `?` propagate secara automatic |
| **Redirect** | `header('location:...')` | `Redirect::to("/")` |
| **Security** | Prepared stmt elak SQL injection | Query macro compile-time check |
| **Concurrency** | Tiap request = 1 thread | Async/await — ribuan request serentak |
| **Templating** | HTML + PHP bercampur dalam `.php` | Template `.html` terpisah dari logic |
| **Template error** | Runtime (blank page / notice) | Compile-time — typo = build gagal |
| **Template syntax** | `<?php echo $name; ?>` | `{{ name }}` |
| **Template loop** | `<?php foreach($list as $e): ?>` | `{% for e in list %}` |
| **Template inherit** | `include 'header.php'` | `{% extends "base.html" %}` |

---

## 🧠 Brain Teaser

Cuba jawab sebelum tengok jawapan:

**Kenapa PHP boleh buat `$row["id"]` terus tapi Rust perlu `.ok_or(AppError::NotFound(id))?`?**

<details>
<summary>👀 Jawapan</summary>

PHP akan return `null` kalau tiada data — dan jika tak check, kod teruskan dengan `null` (boleh sebabkan bug senyap).

Rust `fetch_optional` return `Option<Employee>` — compiler **paksa** kita handle kes `None` sebelum guna data. `.ok_or(...)` tukar `None` → `Err(...)`, dan `?` return error kalau tiada rekod. Lebih selamat dan explicit.
</details>

---

*Rust CRUD — berdasarkan tutorial PHPpot, ditulis semula dengan idiom Rust: SQLx + Axum + Askama + thiserror.*
