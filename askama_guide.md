# 🎨 Askama — Template Engine untuk Rust

> Panduan lengkap Askama: syntax Jinja2-like, template inheritance,
> filters, integrasi Axum, dan cara bina web app dengan HTML rendering.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu Askama?

```
Askama = Template engine untuk Rust yang:
  ✔ Compile-time checked  — ralat template = compile error, bukan runtime!
  ✔ Jinja2-like syntax    — familiar kalau dari Python/PHP
  ✔ Zero-cost abstractions — templat dikompil terus ke Rust code
  ✔ Type-safe             — variable dalam template mesti betul type
  ✔ Fast                  — tiada parsing semasa runtime

Berbanding engine lain:
  Tera      → runtime parsing, lebih flexible tapi lambat
  Handlebars → runtime, popular dalam Node.js world
  Askama    → compile-time, selamat, cepat — cara Rust!

Cara kerja:
  Template .html → Proc Macro → Rust code → Binary
  (compile time)                (runtime: hanya string formatting)
```

---

## Peta Pembelajaran

```
Bahagian 1  → Setup & Template Asas
Bahagian 2  → Variables & Expressions
Bahagian 3  → Control Flow (if, for, while)
Bahagian 4  → Template Inheritance
Bahagian 5  → Macros & Includes
Bahagian 6  → Filters
Bahagian 7  → Custom Filters
Bahagian 8  → Integrasi Axum
Bahagian 9  → Integrasi Actix-web
Bahagian 10 → Mini Project: KADA Web Portal
```

---

# BAHAGIAN 1: Setup & Template Asas 🚀

## Cargo.toml

```toml
[dependencies]
askama          = "0.12"
askama_axum     = "0.4"   # untuk integrasi Axum
axum            = "0.7"   # kalau guna Axum
tokio           = { version = "1", features = ["full"] }
serde           = { version = "1", features = ["derive"] }
```

## Struktur Projek

```
projek/
├── Cargo.toml
├── src/
│   ├── main.rs
│   └── handlers.rs
└── templates/            ← semua template di sini!
    ├── base.html         ← template induk
    ├── index.html        ← halaman utama
    ├── pekerja/
    │   ├── senarai.html
    │   └── profil.html
    └── _macros.html      ← reusable macros (prefix _ untuk convention)
```

---

## Template Pertama

```rust
// src/main.rs
use askama::Template;

// ── Template struct ────────────────────────────────────────────
// Derive Template + tentukan fail template
#[derive(Template)]
#[template(path = "index.html")]  // relatif kepada folder templates/
struct TemplatIndexPage<'a> {
    tajuk:    &'a str,
    mesej:    String,
    versi:    u32,
}

fn main() {
    let halaman = TemplatIndexPage {
        tajuk:  "KADA Mobile Portal",
        mesej:  "Selamat datang!".to_string(),
        versi:  2,
    };

    // Render template → String
    let html = halaman.render().unwrap();
    println!("{}", html);
}
```

```html
<!-- templates/index.html -->
<!DOCTYPE html>
<html lang="ms">
<head>
    <meta charset="UTF-8">
    <title>{{ tajuk }}</title>
</head>
<body>
    <h1>{{ tajuk }}</h1>
    <p>{{ mesej }}</p>
    <footer>Versi {{ versi }}</footer>
</body>
</html>
```

**Output:**
```html
<!DOCTYPE html>
<html lang="ms">
<head>
    <meta charset="UTF-8">
    <title>KADA Mobile Portal</title>
</head>
<body>
    <h1>KADA Mobile Portal</h1>
    <p>Selamat datang!</p>
    <footer>Versi 2</footer>
</body>
</html>
```

---

## Konfigurasi Template

```rust
// Konfigurasi tambahan dalam #[template(...)]

// Path sahaja
#[derive(Template)]
#[template(path = "halaman.html")]
struct Halaman { ... }

// Source inline (tanpa fail)
#[derive(Template)]
#[template(source = "<p>{{ nama }}</p>", ext = "html")]
struct PotonganMudah<'a> { nama: &'a str }

// Dengan escape mode
#[derive(Template)]
#[template(path = "halaman.html", escape = "html")]  // default
struct HalamanSelamat { ... }

#[derive(Template)]
#[template(path = "data.txt", escape = "none")]       // teks biasa
struct DataTeks { ... }

// print = "ast" untuk debug (print AST)
#[derive(Template)]
#[template(path = "debug.html", print = "ast")]
struct DebugTemplate { ... }
```

---

## 🧠 Brain Teaser #1

Mengapa Askama lebih selamat dari engine template runtime seperti Tera?

```rust
// Askama
#[derive(Template)]
#[template(path = "profil.html")]
struct ProfilTemplate {
    nama: String,  // ada field 'nama'
}

// templates/profil.html menggunakan {{ nama_penuh }}
// (typo! sepatutnya 'nama' bukan 'nama_penuh')
```

<details>
<summary>👀 Jawapan</summary>

**Dengan Askama:**
```
error: unknown variable `nama_penuh`
  --> templates/profil.html:5:5
   |
 5 |     {{ nama_penuh }}
   |        ^^^^^^^^^^^ this variable is not defined in `ProfilTemplate`
```

**Ralat berlaku semasa COMPILE TIME** — sebelum program dijalankan!

Dengan engine runtime (Tera, Handlebars):
- Program compile OK
- Semasa runtime, bila template dirender...
- `{{ nama_penuh }}` = string kosong (atau panic) — BUG tersembunyi!

**Manfaat Askama:**
1. Typo dalam template = compile error (bukan runtime bug)
2. Field dalam struct dan template sentiasa konsisten
3. Refactoring struct → compiler beritahu semua template yang perlu diubah
4. Tiada surprises dalam production!

Ini adalah "correctness by construction" — Rust way!
</details>

---

# BAHAGIAN 2: Variables & Expressions 🔢

## Semua Cara Guna Variables

```html
<!-- templates/contoh.html -->

<!-- ── Basic variable ─────────────────────────────────────── -->
{{ nama }}
{{ umur }}

<!-- ── Rust expressions ───────────────────────────────────── -->
{{ 2 + 2 }}
{{ nama.len() }}
{{ nama.to_uppercase() }}
{{ nama.chars().count() }}

<!-- ── Method calls ──────────────────────────────────────── -->
{{ gaji * 1.06 }}
{{ format!("RM{:.2}", gaji) }}

<!-- ── Ternary-like (guna if) ────────────────────────────── -->
{{ if aktif { "Aktif" } else { "Tidak Aktif" } }}

<!-- ── Access struct fields ──────────────────────────────── -->
{{ pekerja.nama }}
{{ pekerja.bahagian.nama }}

<!-- ── Option ────────────────────────────────────────────── -->
{{ emel.as_deref().unwrap_or("(tiada emel)") }}

<!-- ── Escape HTML (auto dalam mode html) ────────────────── -->
{{ nama }}  <!-- <Ali & Siti> → &lt;Ali &amp; Siti&gt; -->

<!-- ── Raw/unescaped (AWAS! XSS!) ───────────────────────── -->
{{ nama|safe }}  <!-- Tidak di-escape -->
```

```rust
use askama::Template;

#[derive(Template)]
#[template(path = "contoh.html")]
struct ContohTemplate {
    nama:   String,
    umur:   u32,
    gaji:   f64,
    aktif:  bool,
    emel:   Option<String>,
    html_kandungan: String,  // kandungan HTML yang trusted
}
```

---

## Auto-Escaping

```html
<!-- Askama AUTO-escape HTML by default dalam mode "html" -->
<!-- Ini melindungi daripada XSS! -->

{% set input_jahat = "<script>alert('XSS')</script>" %}

{{ input_jahat }}
<!-- Output: &lt;script&gt;alert(&#x27;XSS&#x27;)&lt;/script&gt; -->
<!-- Selamat! Script tidak akan laksana -->

{{ input_jahat|safe }}
<!-- Output: <script>alert('XSS')</script> -->
<!-- BERBAHAYA! Jangan guna pada input pengguna! -->
```

```rust
// Untuk render HTML yang trusted (dari CMS, markdown, dll)
#[derive(Template)]
#[template(path = "artikel.html")]
struct TemplatArtikel {
    tajuk:    String,
    kandungan: String,  // kandungan HTML dari markdown parser (trusted)
}
```

```html
<!-- templates/artikel.html -->
<h1>{{ tajuk }}</h1>  <!-- auto-escaped -->
<div class="kandungan">
    {{ kandungan|safe }}  <!-- HTML trusted, tidak di-escape -->
</div>
```

---

# BAHAGIAN 3: Control Flow 🔄

## if, else if, else

```html
<!-- templates/status.html -->

<!-- ── if asas ──────────────────────────────────────────── -->
{% if aktif %}
    <span class="hijau">Aktif</span>
{% else %}
    <span class="merah">Tidak Aktif</span>
{% endif %}

<!-- ── if else if ────────────────────────────────────────── -->
{% if gaji >= 10000 %}
    <span class="emas">Pengurusan Atasan</span>
{% else if gaji >= 5000 %}
    <span class="biru">Pengurusan Pertengahan</span>
{% else if gaji >= 3000 %}
    <span class="hijau">Staf</span>
{% else %}
    <span class="kelabu">Pelatih</span>
{% endif %}

<!-- ── Semak Option ──────────────────────────────────────── -->
{% if let Some(emel) = emel %}
    <a href="mailto:{{ emel }}">{{ emel }}</a>
{% else %}
    <em>Tiada emel</em>
{% endif %}

<!-- ── Ungkapan kompleks ──────────────────────────────────── -->
{% if pekerja.bahagian == "ICT" && pekerja.aktif %}
    <span class="ict-aktif">ICT Aktif</span>
{% endif %}

<!-- ── Guna matches! ─────────────────────────────────────── -->
{% if matches!(status, Status::Aktif | Status::Baru) %}
    <span>Status baik</span>
{% endif %}
```

```rust
use askama::Template;

#[derive(PartialEq)]
enum Status { Aktif, Baru, Ditangguh }

#[derive(Template)]
#[template(path = "status.html")]
struct StatusTemplate {
    aktif:  bool,
    gaji:   f64,
    emel:   Option<String>,
    status: Status,
}
```

---

## for Loop

```html
<!-- templates/senarai.html -->

<!-- ── for asas ──────────────────────────────────────────── -->
<ul>
{% for pekerja in pekerja_list %}
    <li>{{ pekerja.nama }} — {{ pekerja.bahagian }}</li>
{% endfor %}
</ul>

<!-- ── for dengan loop object ───────────────────────────── -->
<table>
{% for p in pekerja_list %}
    <tr class="{% if loop.index0 % 2 == 0 %}genap{% else %}ganjil{% endif %}">
        <td>{{ loop.index }}</td>     <!-- 1, 2, 3, ... -->
        <td>{{ loop.index0 }}</td>    <!-- 0, 1, 2, ... -->
        <td>{{ loop.first }}</td>     <!-- true/false -->
        <td>{{ loop.last }}</td>      <!-- true/false -->
        <td>{{ p.nama }}</td>
    </tr>
{% endfor %}
</table>

<!-- ── for dengan else (kalau kosong) ───────────────────── -->
{% for p in pekerja_list %}
    <div>{{ p.nama }}</div>
{% else %}
    <p>Tiada pekerja dijumpai.</p>
{% endfor %}

<!-- ── Iterate dengan index menggunakan enumerate ───────── -->
{% for (i, p) in pekerja_list.iter().enumerate() %}
    <p>{{ i + 1 }}. {{ p.nama }}</p>
{% endfor %}

<!-- ── Nested loops ──────────────────────────────────────── -->
{% for jabatan in jabatan_list %}
    <h3>{{ jabatan.nama }}</h3>
    <ul>
    {% for p in jabatan.pekerja %}
        <li>{{ p.nama }}</li>
    {% endfor %}
    </ul>
{% endfor %}
```

```rust
use askama::Template;

#[derive(Debug)]
struct Pekerja { nama: String, bahagian: String }

#[derive(Debug)]
struct Jabatan { nama: String, pekerja: Vec<Pekerja> }

#[derive(Template)]
#[template(path = "senarai.html")]
struct SenaraiTemplate<'a> {
    pekerja_list: &'a [Pekerja],
    jabatan_list: &'a [Jabatan],
}
```

---

## while, set, let

```html
<!-- templates/lanjutan.html -->

<!-- ── let/set — assign variable dalam template ────────── -->
{% let jumlah = pekerja_list.len() %}
<p>Jumlah pekerja: {{ jumlah }}</p>

{% let gaji_total = pekerja_list.iter().map(|p| p.gaji).sum::<f64>() %}
<p>Jumlah gaji: RM{{ gaji_total }}</p>

<!-- ── set (alias untuk let dalam Askama) ───────────────── -->
{% set tajuk_besar = tajuk.to_uppercase() %}
<h1>{{ tajuk_besar }}</h1>

<!-- ── while ────────────────────────────────────────────── -->
{% let mut kiraan = 0_usize %}
{% while kiraan < 5 %}
    <p>Baris {{ kiraan }}</p>
    {% let kiraan = kiraan + 1 %}
{% endwhile %}
```

---

# BAHAGIAN 4: Template Inheritance 🏛️

## Base Template

```html
<!-- templates/asas.html -->
<!DOCTYPE html>
<html lang="ms">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{% block tajuk %}KADA Portal{% endblock %}</title>

    <!-- CSS dikongsi -->
    <link rel="stylesheet" href="/static/css/tailwind.css">
    <link rel="stylesheet" href="/static/css/app.css">

    <!-- CSS tambahan untuk halaman tertentu -->
    {% block css %}{% endblock %}
</head>
<body class="bg-gray-100">

    <!-- Navigation bar -->
    <nav class="bg-navy-900 text-white p-4">
        <div class="container mx-auto flex justify-between">
            <a href="/" class="text-xl font-bold">🌾 KADA Portal</a>
            <div class="flex gap-4">
                <a href="/pekerja">Pekerja</a>
                <a href="/laporan">Laporan</a>
                {% block nav_tambahan %}{% endblock %}
            </div>
        </div>
    </nav>

    <!-- Header halaman -->
    {% block header %}
    <div class="bg-navy-800 text-white py-8">
        <div class="container mx-auto">
            <h1 class="text-2xl font-bold">{% block tajuk_halaman %}{% endblock %}</h1>
        </div>
    </div>
    {% endblock %}

    <!-- Kandungan utama -->
    <main class="container mx-auto py-8 px-4">
        {% block kandungan %}{% endblock %}
    </main>

    <!-- Footer -->
    <footer class="bg-navy-900 text-white py-4 mt-8">
        <div class="container mx-auto text-center">
            <p>© 2024 KADA — Lembaga Kemajuan Pertanian Kemubu</p>
        </div>
    </footer>

    <!-- JavaScript dikongsi -->
    <script src="/static/js/app.js"></script>
    {% block skrip %}{% endblock %}
</body>
</html>
```

---

## Child Templates

```html
<!-- templates/halaman_utama.html -->
{% extends "asas.html" %}

{% block tajuk %}Halaman Utama — KADA Portal{% endblock %}

{% block tajuk_halaman %}Selamat Datang{% endblock %}

{% block kandungan %}
<div class="grid grid-cols-3 gap-6">
    <div class="bg-white rounded shadow p-6">
        <h3 class="text-lg font-bold">Jumlah Pekerja</h3>
        <p class="text-3xl text-navy-800">{{ jumlah_pekerja }}</p>
    </div>
    <div class="bg-white rounded shadow p-6">
        <h3 class="text-lg font-bold">Kehadiran Hari Ini</h3>
        <p class="text-3xl text-green-600">{{ kehadiran_hari_ini }}</p>
    </div>
    <div class="bg-white rounded shadow p-6">
        <h3 class="text-lg font-bold">Tidak Hadir</h3>
        <p class="text-3xl text-red-600">{{ tidak_hadir }}</p>
    </div>
</div>

<div class="mt-8 bg-white rounded shadow p-6">
    <h2 class="text-xl font-bold mb-4">Aktiviti Terkini</h2>
    {% for aktiviti in aktiviti_terkini %}
    <div class="border-b py-3 flex justify-between">
        <span>{{ aktiviti.keterangan }}</span>
        <span class="text-gray-500">{{ aktiviti.masa }}</span>
    </div>
    {% else %}
    <p class="text-gray-500">Tiada aktiviti hari ini.</p>
    {% endfor %}
</div>
{% endblock %}

{% block skrip %}
<script>
    // Refresh setiap 5 minit
    setTimeout(() => location.reload(), 300000);
</script>
{% endblock %}
```

```html
<!-- templates/pekerja/senarai.html -->
{% extends "asas.html" %}

{% block tajuk %}Senarai Pekerja — KADA{% endblock %}
{% block tajuk_halaman %}Senarai Pekerja{% endblock %}

{% block nav_tambahan %}
<a href="/pekerja/baru" class="bg-gold-500 px-3 py-1 rounded">+ Tambah</a>
{% endblock %}

{% block kandungan %}
<!-- Filter -->
<div class="bg-white rounded shadow p-4 mb-6">
    <form method="get" class="flex gap-4">
        <input type="text" name="cari" value="{{ filter_cari }}"
               placeholder="Cari nama..." class="border rounded p-2 flex-1">
        <select name="bahagian" class="border rounded p-2">
            <option value="">Semua Bahagian</option>
            {% for b in senarai_bahagian %}
            <option value="{{ b }}" {% if filter_bahagian == b %}selected{% endif %}>
                {{ b }}
            </option>
            {% endfor %}
        </select>
        <button type="submit" class="bg-navy-800 text-white px-4 py-2 rounded">
            Cari
        </button>
    </form>
</div>

<!-- Jadual -->
<div class="bg-white rounded shadow overflow-hidden">
    <table class="w-full">
        <thead class="bg-navy-800 text-white">
            <tr>
                <th class="p-3 text-left">No</th>
                <th class="p-3 text-left">No. Pekerja</th>
                <th class="p-3 text-left">Nama</th>
                <th class="p-3 text-left">Bahagian</th>
                <th class="p-3 text-right">Gaji</th>
                <th class="p-3 text-center">Status</th>
                <th class="p-3 text-center">Tindakan</th>
            </tr>
        </thead>
        <tbody>
        {% for (i, p) in pekerja.iter().enumerate() %}
            <tr class="border-b {% if loop.index0 % 2 == 0 %}bg-gray-50{% endif %}
                       hover:bg-blue-50 transition">
                <td class="p-3">{{ i + 1 }}</td>
                <td class="p-3 font-mono">{{ p.no_pekerja }}</td>
                <td class="p-3 font-medium">{{ p.nama }}</td>
                <td class="p-3">{{ p.bahagian }}</td>
                <td class="p-3 text-right">RM{{ p.gaji|format_ringgit }}</td>
                <td class="p-3 text-center">
                    {% if p.aktif %}
                    <span class="bg-green-100 text-green-800 px-2 py-1 rounded-full text-sm">
                        Aktif
                    </span>
                    {% else %}
                    <span class="bg-red-100 text-red-800 px-2 py-1 rounded-full text-sm">
                        Tidak Aktif
                    </span>
                    {% endif %}
                </td>
                <td class="p-3 text-center">
                    <a href="/pekerja/{{ p.id }}" class="text-blue-600 hover:underline mr-2">Lihat</a>
                    <a href="/pekerja/{{ p.id }}/edit" class="text-green-600 hover:underline">Edit</a>
                </td>
            </tr>
        {% else %}
            <tr>
                <td colspan="7" class="p-6 text-center text-gray-500">
                    Tiada pekerja dijumpai.
                </td>
            </tr>
        {% endfor %}
        </tbody>
    </table>
</div>

<!-- Pagination -->
{% if jumlah_halaman > 1 %}
<div class="mt-4 flex justify-center gap-2">
    {% for h in 1..=jumlah_halaman %}
    <a href="?halaman={{ h }}"
       class="px-3 py-1 rounded {% if h == halaman_semasa %}bg-navy-800 text-white{% else %}bg-white border{% endif %}">
        {{ h }}
    </a>
    {% endfor %}
</div>
{% endif %}
{% endblock %}
```

---

# BAHAGIAN 5: Macros & Includes 🔧

## Macros dalam Askama

```html
<!-- templates/_macros.html -->

<!-- ── Macro: Kad Statistik ─────────────────────────────── -->
{% macro kad_stat(tajuk, nilai, warna="blue") %}
<div class="bg-white rounded shadow p-6 border-l-4 border-{{ warna }}-500">
    <p class="text-gray-600 text-sm">{{ tajuk }}</p>
    <p class="text-3xl font-bold text-{{ warna }}-700 mt-1">{{ nilai }}</p>
</div>
{% endmacro %}

<!-- ── Macro: Badge Status ───────────────────────────────── -->
{% macro badge(teks, warna) %}
<span class="inline-block px-2 py-1 rounded-full text-xs font-medium
             bg-{{ warna }}-100 text-{{ warna }}-800">
    {{ teks }}
</span>
{% endmacro %}

<!-- ── Macro: Butang Tindakan ────────────────────────────── -->
{% macro butang(teks, href, jenis="primary") %}
{% if jenis == "primary" %}
<a href="{{ href }}" class="bg-navy-800 text-white px-4 py-2 rounded hover:bg-navy-700">
    {{ teks }}
</a>
{% else if jenis == "danger" %}
<a href="{{ href }}" class="bg-red-600 text-white px-4 py-2 rounded hover:bg-red-700">
    {{ teks }}
</a>
{% else %}
<a href="{{ href }}" class="border border-gray-300 px-4 py-2 rounded hover:bg-gray-50">
    {{ teks }}
</a>
{% endif %}
{% endmacro %}

<!-- ── Macro: Input Form ─────────────────────────────────── -->
{% macro input_form(label, nama, nilai="", jenis="text", diperlukan=false) %}
<div class="mb-4">
    <label class="block text-sm font-medium text-gray-700 mb-1">
        {{ label }}{% if diperlukan %} <span class="text-red-500">*</span>{% endif %}
    </label>
    <input
        type="{{ jenis }}"
        name="{{ nama }}"
        value="{{ nilai }}"
        {% if diperlukan %}required{% endif %}
        class="w-full border rounded p-2 focus:outline-none focus:ring-2 focus:ring-navy-500"
    >
</div>
{% endmacro %}
```

```html
<!-- templates/dashboard.html -->
{% extends "asas.html" %}

<!-- Import macros dari fail lain -->
{% import "_macros.html" as macros %}

{% block kandungan %}
<div class="grid grid-cols-4 gap-4 mb-8">
    <!-- Guna macro yang diimport -->
    {{ macros::kad_stat(tajuk="Jumlah Pekerja", nilai=jumlah_pekerja, warna="blue") }}
    {{ macros::kad_stat(tajuk="Kehadiran Hari Ini", nilai=kehadiran, warna="green") }}
    {{ macros::kad_stat(tajuk="Cuti Hari Ini", nilai=cuti, warna="yellow") }}
    {{ macros::kad_stat(tajuk="Tidak Hadir", nilai=tidak_hadir, warna="red") }}
</div>

<div class="flex gap-2">
    {{ macros::butang(teks="Tambah Pekerja", href="/pekerja/baru") }}
    {{ macros::butang(teks="Eksport", href="/eksport", jenis="secondary") }}
    {{ macros::butang(teks="Padam Semua", href="/padam", jenis="danger") }}
</div>
{% endblock %}
```

---

## include — Masukkan Template Lain

```html
<!-- templates/header.html -->
<div class="bg-yellow-50 border-l-4 border-yellow-400 p-4 mb-6">
    <p class="font-bold text-yellow-800">⚠️ Amaran Sistem</p>
    <p class="text-yellow-700">Penyelenggaraan dijadualkan pada 23:00 - 01:00</p>
</div>
```

```html
<!-- templates/halaman.html -->
{% extends "asas.html" %}

{% block kandungan %}
<!-- Include template lain -->
{% include "header.html" %}

<!-- Include dengan condition -->
{% if tunjuk_amaran %}
{% include "amaran.html" %}
{% endif %}

<div class="kandungan-utama">
    ...
</div>
{% endblock %}
```

---

# BAHAGIAN 6: Filters 🔍

## Built-in Filters

```html
<!-- templates/filters.html -->

<!-- ── String filters ─────────────────────────────────────── -->
{{ nama | upper }}          <!-- ALI AHMAD -->
{{ nama | lower }}          <!-- ali ahmad -->
{{ nama | capitalize }}     <!-- Ali ahmad (hanya huruf pertama) -->
{{ nama | title }}          <!-- Ali Ahmad (setiap perkataan) -->
{{ nama | trim }}           <!-- buang whitespace depan belakang -->
{{ nama | truncate(20) }}   <!-- potong kepada 20 aksara -->
{{ nama | replace("Ali", "Ahmad") }} <!-- ganti substring -->

<!-- ── Number filters ─────────────────────────────────────── -->
{{ gaji | round }}          <!-- 4500 -->
{{ gaji | round(2) }}       <!-- 4500.00 -->
{{ nisbah | round(4) }}     <!-- 0.1234 -->
{{ nombor | abs }}          <!-- nilai mutlak -->

<!-- ── Collection filters ────────────────────────────────── -->
{{ senarai | length }}      <!-- jumlah elemen -->
{{ senarai | first }}       <!-- elemen pertama -->
{{ senarai | last }}        <!-- elemen terakhir -->
{{ senarai | join(", ") }}  <!-- ["a","b","c"] → "a, b, c" -->
{{ senarai | sort }}        <!-- sort -->
{{ senarai | reverse }}     <!-- terbalik -->

<!-- ── Escaping ───────────────────────────────────────────── -->
{{ html_kandungan | safe }}  <!-- tidak di-escape (trusted!) -->
{{ input_pengguna | e }}     <!-- escape HTML (sama dengan default) -->
{{ url | urlencode }}        <!-- encode untuk URL -->

<!-- ── Conditional ───────────────────────────────────────── -->
{{ nilai | default("N/A") }}               <!-- kalau falsy -->
{{ option_nilai | default("kosong") }}     <!-- kalau None -->

<!-- ── Debug ─────────────────────────────────────────────── -->
{{ nilai | debug }}          <!-- format dengan {:?} -->
{{ nilai | json }}           <!-- serialize ke JSON -->

<!-- ── Chaining filters ───────────────────────────────────── -->
{{ nama | trim | upper | truncate(15) }}
```

---

# BAHAGIAN 7: Custom Filters 🛠️

## Tulis Filter Sendiri

```rust
// src/filters.rs

use askama::Result;

// Filter mudah — format ringgit
pub fn format_ringgit(nilai: &f64) -> Result<String> {
    Ok(format!("{:,.2}", nilai))
    // 4500.0 → "4,500.00"
}

// Filter dengan parameter
pub fn potong_teks(teks: &str, panjang: usize, sufiks: &str) -> Result<String> {
    if teks.chars().count() <= panjang {
        Ok(teks.to_string())
    } else {
        let dipotong: String = teks.chars().take(panjang).collect();
        Ok(format!("{}{}", dipotong, sufiks))
    }
}

// Filter untuk format tarikh
pub fn format_tarikh(timestamp: &i64) -> Result<String> {
    use chrono::{DateTime, Utc, TimeZone};
    let dt: DateTime<Utc> = Utc.timestamp_opt(*timestamp, 0).unwrap();
    Ok(dt.format("%d/%m/%Y %H:%M").to_string())
}

// Filter untuk nombor dengan separator
pub fn nombor_format(n: &u64) -> Result<String> {
    let s = n.to_string();
    let mut hasil = String::new();
    for (i, c) in s.chars().rev().enumerate() {
        if i > 0 && i % 3 == 0 {
            hasil.push(',');
        }
        hasil.push(c);
    }
    Ok(hasil.chars().rev().collect())
}

// Filter enum ke teks
pub fn status_teks(aktif: &bool) -> Result<&'static str> {
    Ok(if *aktif { "Aktif" } else { "Tidak Aktif" })
}

// Filter dengan multiple types guna trait
pub fn warna_gred(skor: &f64) -> Result<&'static str> {
    Ok(match *skor as u32 {
        90..=100 => "green",
        75..=89  => "blue",
        60..=74  => "yellow",
        _        => "red",
    })
}
```

```rust
// src/main.rs
use askama::Template;

// Import modul filters
mod filters;

#[derive(Template)]
#[template(path = "laporan.html")]
struct LaporanTemplate {
    gaji:       f64,
    keterangan: String,
    timestamp:  i64,
    kiraan:     u64,
    aktif:      bool,
    skor:       f64,
}
```

```html
<!-- templates/laporan.html -->
<!-- Custom filters digunakan terus! -->
<p>Gaji: RM{{ gaji|format_ringgit }}</p>
<p>Ringkasan: {{ keterangan|potong_teks(50, "...") }}</p>
<p>Masa: {{ timestamp|format_tarikh }}</p>
<p>Kiraan: {{ kiraan|nombor_format }}</p>
<p class="text-{{ skor|warna_gred }}-600">
    Skor: {{ skor|round(1) }}
</p>
<span class="{% if aktif %}bg-green{% else %}bg-red{% endif %}-100">
    {{ aktif|status_teks }}
</span>
```

---

# BAHAGIAN 8: Integrasi Axum 🌐

## Setup Lengkap

```toml
[dependencies]
askama       = "0.12"
askama_axum  = "0.4"
axum         = "0.7"
tokio        = { version = "1", features = ["full"] }
serde        = { version = "1", features = ["derive"] }
tower-http   = { version = "0.5", features = ["fs"] }
```

```rust
// src/main.rs
use axum::{
    Router,
    routing::{get, post},
    extract::{State, Path, Query, Form},
    response::IntoResponse,
};
use askama::Template;
use askama_axum::IntoResponse as AskamaResponse;
use tower_http::services::ServeDir;
use std::sync::Arc;

mod handlers;
mod filters;
mod models;

#[derive(Clone)]
struct AppState {
    // db, cache, dll
}

#[tokio::main]
async fn main() {
    let state = AppState {};

    let app = Router::new()
        .route("/",               get(handlers::halaman_utama))
        .route("/pekerja",        get(handlers::senarai_pekerja))
        .route("/pekerja/baru",   get(handlers::borang_pekerja).post(handlers::simpan_pekerja))
        .route("/pekerja/:id",    get(handlers::profil_pekerja))
        .route("/pekerja/:id/edit", get(handlers::edit_pekerja))
        // Serve static files
        .nest_service("/static", ServeDir::new("static"))
        .with_state(state);

    let pendengar = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("Server: http://localhost:3000");
    axum::serve(pendengar, app).await.unwrap();
}
```

---

## Handlers dengan Template

```rust
// src/handlers.rs
use axum::{
    extract::{State, Path, Query, Form},
    response::IntoResponse,
    http::StatusCode,
};
use askama::Template;
use askama_axum::IntoResponse as AskamaIntoResponse;
use serde::Deserialize;

use crate::{AppState, models::*};

// ── Template Structs ───────────────────────────────────────────

#[derive(Template)]
#[template(path = "halaman_utama.html")]
struct TemplateHalamanUtama {
    jumlah_pekerja:  usize,
    kehadiran_hari:  usize,
    tidak_hadir:     usize,
    aktiviti_terkini: Vec<Aktiviti>,
}

#[derive(Deserialize)]
struct QuerySenarai {
    cari:     Option<String>,
    bahagian: Option<String>,
    halaman:  Option<usize>,
}

#[derive(Template)]
#[template(path = "pekerja/senarai.html")]
struct TemplateSenaraiPekerja {
    pekerja:          Vec<Pekerja>,
    senarai_bahagian: Vec<String>,
    filter_cari:      String,
    filter_bahagian:  String,
    halaman_semasa:   usize,
    jumlah_halaman:   usize,
}

#[derive(Template)]
#[template(path = "pekerja/profil.html")]
struct TemplateProfilPekerja<'a> {
    pekerja:     &'a Pekerja,
    kehadiran:   Vec<RekodKehadiran>,
}

#[derive(Template)]
#[template(path = "pekerja/borang.html")]
struct TemplateBorangPekerja {
    pekerja:  Option<Pekerja>,   // None = baru, Some = edit
    bahagian: Vec<String>,
    ralat:    Vec<String>,
}

// ── Handlers ──────────────────────────────────────────────────

pub async fn halaman_utama(
    State(_state): State<AppState>,
) -> impl IntoResponse {
    let template = TemplateHalamanUtama {
        jumlah_pekerja:  150,
        kehadiran_hari:  128,
        tidak_hadir:     22,
        aktiviti_terkini: vec![
            Aktiviti { keterangan: "Ali Ahmad daftar masuk".into(), masa: "08:02".into() },
            Aktiviti { keterangan: "Siti Hawa daftar masuk".into(), masa: "08:15".into() },
        ],
    };

    // askama_axum::IntoResponse auto-handle render + Content-Type: text/html
    AskamaIntoResponse::into_response(template)
}

pub async fn senarai_pekerja(
    State(_state): State<AppState>,
    Query(query): Query<QuerySenarai>,
) -> impl IntoResponse {
    let cari = query.cari.unwrap_or_default();
    let bahagian = query.bahagian.unwrap_or_default();
    let halaman = query.halaman.unwrap_or(1);

    // Simulate data
    let semua_pekerja = vec![
        Pekerja { id: 1, no_pekerja: "KADA001".into(), nama: "Ali Ahmad".into(),
                  bahagian: "ICT".into(), gaji: 4500.0, aktif: true },
        Pekerja { id: 2, no_pekerja: "KADA002".into(), nama: "Siti Hawa".into(),
                  bahagian: "HR".into(), gaji: 3800.0, aktif: true },
    ];

    // Filter
    let pekerja: Vec<Pekerja> = semua_pekerja.into_iter()
        .filter(|p| cari.is_empty() || p.nama.to_lowercase().contains(&cari.to_lowercase()))
        .filter(|p| bahagian.is_empty() || p.bahagian == bahagian)
        .collect();

    let template = TemplateSenaraiPekerja {
        senarai_bahagian: vec!["ICT".into(), "HR".into(), "Kewangan".into()],
        jumlah_halaman:  (pekerja.len() + 9) / 10,
        filter_cari:     cari,
        filter_bahagian: bahagian,
        halaman_semasa:  halaman,
        pekerja,
    };

    AskamaIntoResponse::into_response(template)
}

#[derive(Deserialize)]
pub struct FormPekerja {
    nama:     String,
    bahagian: String,
    gaji:     f64,
}

pub async fn simpan_pekerja(
    State(_state): State<AppState>,
    Form(data): Form<FormPekerja>,
) -> impl IntoResponse {
    // Validate
    let mut ralat = Vec::new();
    if data.nama.trim().is_empty() { ralat.push("Nama tidak boleh kosong".into()); }
    if data.gaji < 0.0 { ralat.push("Gaji tidak boleh negatif".into()); }

    if !ralat.is_empty() {
        let template = TemplateBorangPekerja {
            pekerja:  None,
            bahagian: vec!["ICT".into(), "HR".into()],
            ralat,
        };
        return (StatusCode::UNPROCESSABLE_ENTITY,
                AskamaIntoResponse::into_response(template));
    }

    // Simpan ke DB...
    // Redirect ke halaman pekerja
    (StatusCode::SEE_OTHER, axum::response::Redirect::to("/pekerja")).into_response()
}

pub async fn profil_pekerja(
    State(_state): State<AppState>,
    Path(id): Path<u32>,
) -> impl IntoResponse {
    let pekerja = Pekerja {
        id:         id,
        no_pekerja: "KADA001".into(),
        nama:       "Ali Ahmad".into(),
        bahagian:   "ICT".into(),
        gaji:       4500.0,
        aktif:      true,
    };

    let template = TemplateProfilPekerja {
        pekerja:   &pekerja,
        kehadiran: vec![],
    };

    AskamaIntoResponse::into_response(template)
}

pub async fn borang_pekerja() -> impl IntoResponse {
    AskamaIntoResponse::into_response(TemplateBorangPekerja {
        pekerja:  None,
        bahagian: vec!["ICT".into(), "HR".into(), "Kewangan".into()],
        ralat:    vec![],
    })
}

pub async fn edit_pekerja(Path(id): Path<u32>) -> impl IntoResponse {
    let pekerja = Pekerja {
        id, no_pekerja: "KADA001".into(), nama: "Ali".into(),
        bahagian: "ICT".into(), gaji: 4500.0, aktif: true,
    };
    AskamaIntoResponse::into_response(TemplateBorangPekerja {
        pekerja:  Some(pekerja),
        bahagian: vec!["ICT".into(), "HR".into()],
        ralat:    vec![],
    })
}
```

---

# BAHAGIAN 9: Integrasi Actix-web 🕸️

```toml
[dependencies]
askama         = "0.12"
askama_actix   = "0.14"
actix-web      = "4"
```

```rust
use actix_web::{web, App, HttpServer, HttpResponse, Responder};
use askama::Template;
use askama_actix::TemplateToResponse;

#[derive(Template)]
#[template(path = "index.html")]
struct IndexTemplate {
    tajuk: String,
    nama:  String,
}

async fn halaman_utama() -> impl Responder {
    let template = IndexTemplate {
        tajuk: "KADA Portal".into(),
        nama:  "Ali Ahmad".into(),
    };

    // askama_actix: .to_response()
    template.to_response()
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/", web::get().to(halaman_utama))
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

---

# BAHAGIAN 10: Mini Project — KADA Web Portal 🏗️

```
Struktur Projek:
kada-portal/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── handlers/
│   │   ├── mod.rs
│   │   ├── auth.rs
│   │   ├── pekerja.rs
│   │   └── kehadiran.rs
│   ├── models.rs
│   └── filters.rs
├── templates/
│   ├── asas.html
│   ├── index.html
│   ├── log_masuk.html
│   ├── pekerja/
│   │   ├── senarai.html
│   │   ├── profil.html
│   │   └── borang.html
│   ├── kehadiran/
│   │   └── senarai.html
│   └── _macros.html
└── static/
    ├── css/
    └── js/
```

```rust
// src/models.rs
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Pekerja {
    pub id:        u32,
    pub no_pekerja: String,
    pub nama:      String,
    pub bahagian:  String,
    pub gaji:      f64,
    pub aktif:     bool,
}

#[derive(Debug, Clone)]
pub struct RekodKehadiran {
    pub id:           u32,
    pub nama_pekerja: String,
    pub masa_masuk:   String,
    pub masa_keluar:  Option<String>,
    pub status:       String,
}

#[derive(Debug, Clone)]
pub struct Aktiviti {
    pub keterangan: String,
    pub masa:       String,
}
```

```rust
// src/filters.rs
use askama::Result;

pub fn format_ringgit(nilai: &f64) -> Result<String> {
    let ringgit = *nilai as u64;
    let sen = ((*nilai - ringgit as f64) * 100.0).round() as u64;
    Ok(format!("{},{:02}.{:02}", ringgit / 1000, ringgit % 1000, sen))
}

pub fn potong(teks: &str, panjang: usize) -> Result<String> {
    if teks.chars().count() <= panjang {
        return Ok(teks.to_string());
    }
    Ok(format!("{}...", teks.chars().take(panjang).collect::<String>()))
}
```

```html
<!-- templates/log_masuk.html -->
<!DOCTYPE html>
<html lang="ms">
<head>
    <meta charset="UTF-8">
    <title>Log Masuk — KADA Portal</title>
    <link rel="stylesheet" href="/static/css/app.css">
</head>
<body class="min-h-screen bg-navy-900 flex items-center justify-center">
    <div class="bg-white rounded-lg shadow-xl p-8 w-full max-w-md">
        <div class="text-center mb-8">
            <h1 class="text-2xl font-bold text-navy-900">🌾 KADA Portal</h1>
            <p class="text-gray-600 mt-1">Log masuk untuk teruskan</p>
        </div>

        {% if !ralat.is_empty() %}
        <div class="bg-red-50 border border-red-200 rounded p-3 mb-4">
            {% for r in ralat %}
            <p class="text-red-700 text-sm">• {{ r }}</p>
            {% endfor %}
        </div>
        {% endif %}

        <form method="post" action="/log-masuk">
            <div class="mb-4">
                <label class="block text-sm font-medium text-gray-700 mb-1">
                    No. Pekerja
                </label>
                <input type="text" name="no_pekerja" value="{{ no_pekerja }}"
                       class="w-full border rounded p-2 focus:ring-2 focus:ring-navy-500"
                       placeholder="KADA001" autofocus required>
            </div>
            <div class="mb-6">
                <label class="block text-sm font-medium text-gray-700 mb-1">
                    Kata Laluan
                </label>
                <input type="password" name="kata_laluan"
                       class="w-full border rounded p-2 focus:ring-2 focus:ring-navy-500"
                       required>
            </div>
            <button type="submit"
                    class="w-full bg-navy-800 text-white py-2 rounded font-medium
                           hover:bg-navy-700 transition">
                Log Masuk
            </button>
        </form>
    </div>
</body>
</html>
```

---

# 📋 Rujukan Pantas — Askama Cheat Sheet

## Setup

```toml
askama       = "0.12"
askama_axum  = "0.4"  # untuk Axum
```

## Struct Template

```rust
#[derive(Template)]
#[template(path = "halaman.html")]           // fail dalam templates/
#[template(source = "<p>{{ x }}</p>", ext = "html")]  // inline
struct MyTemplate {
    nama: String,
    senarai: Vec<Item>,
    opt: Option<String>,
}

// Render
let html: String = template.render()?;

// Axum
use askama_axum::IntoResponse;
let resp = IntoResponse::into_response(template);
```

## Syntax Rujukan

```
Variables:
  {{ variable }}              output variable (auto-escaped)
  {{ obj.field }}             access struct field
  {{ func() }}                call method
  {{ expr | filter }}         apply filter

Control:
  {% if cond %}...{% endif %}
  {% if let Some(x) = opt %}...{% endif %}
  {% for item in list %}...{% endfor %}
  {% for item in list %}...{% else %}...{% endfor %}
  {% let x = expr %}          assign dalam template
  {% set x = expr %}          same as let

Inheritance:
  {% extends "asas.html" %}   extend template
  {% block nama %}...{% endblock %}  define/override block

Macros:
  {% macro nama(param) %}...{% endmacro %}  define
  {{ self::nama(arg) }}       call dalam fail sama
  {% import "macros.html" as m %}
  {{ m::nama(arg) }}          call dari fail lain

Include:
  {% include "partial.html" %}

Built-in Filters:
  upper, lower, trim, capitalize, title
  truncate(n), replace(from, to)
  round, round(n), abs
  length, first, last, join(sep), sort, reverse
  safe, e, urlencode
  default(val)
  debug, json
```

## Custom Filter

```rust
// src/filters.rs
pub fn nama_filter(input: &Type, param: usize) -> askama::Result<String> {
    Ok(format!("..."))
}

// Dalam template: {{ nilai|nama_filter(5) }}
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu dengan Askama + Axum:

1. **Blog Engine** — senarai artikel, halaman artikel penuh, pagination
2. **Dashboard Admin** — statistik, carta ringkas, senarai pengguna
3. **Form Builder** — borang dengan validasi, paparan ralat, redirect
4. **E-commerce** — katalog produk, cart, checkout dengan template inheritance
5. **Laporan PDF** — guna Askama untuk render HTML, kemudian print to PDF

---

*Askama = Template yang diperiksa semasa compile.*
*Kalau template compile, ia betul. Tiada kejutan semasa runtime.* 🦀
