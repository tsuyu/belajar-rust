# PHP → Rust Cheat Sheet

> Berdasarkan topik dari: https://websitesetup.org/php-cheat-sheet/
> Setiap construct PHP ditukar kepada Rust idiomatic.

---

## 1. Basic Syntax

### PHP
```php
<?php
echo "Hello, World!";
print "Hello again";

// Single-line comment
# Also single-line
/* Multi-line
   comment */
```

### Rust
```rust
fn main() {
    println!("Hello, World!");  // echo / print
    print!("No newline");       // tanpa newline

    // Single-line comment
    /* Multi-line
       comment */
}
```

| PHP | Rust |
|---|---|
| `echo "..."` | `println!("...")` |
| `print "..."` | `print!("...")` |
| `<?php ... ?>` | `fn main() { ... }` |
| `;` wajib | `;` wajib (kecuali last expression) |

---

## 2. Variables & Data Types

### PHP
```php
$name    = "Alice";       // string
$age     = 25;            // int
$price   = 9.99;          // float
$active  = true;          // bool
$nothing = null;          // null

// Weak typing — boleh tukar jenis
$x = 5;
$x = "now a string";
```

### Rust
```rust
let name: &str  = "Alice";   // string slice (static)
let name = String::from("Alice"); // owned String
let age: i32    = 25;
let price: f64  = 9.99;
let active: bool = true;
let nothing: Option<i32> = None;  // PHP null → Option<T>

// Rust: strongly typed, immutable by default
let mut x: i32 = 5;
// x = "string";  ← TIDAK BOLEH — compile error
```

### Type Casting

#### PHP
```php
$x = (int)"42";
$y = (float)"3.14";
$z = (string)100;
$b = (bool)1;
```

#### Rust
```rust
let x: i32 = "42".parse().unwrap();        // string → int
let y: f64 = "3.14".parse().unwrap();      // string → float
let z: String = 100.to_string();           // int → String
let b: bool = 1 != 0;                      // int → bool
let n: i64 = 3.14f64 as i64;              // float → int (truncate)
```

### Data Types Comparison

| PHP | Rust | Nota |
|---|---|---|
| `int` | `i8/i16/i32/i64/u8/u16/u32/u64` | pilih saiz explicit |
| `float` | `f32/f64` | default `f64` |
| `string` | `&str` / `String` | `&str` = borrow, `String` = owned |
| `bool` | `bool` | sama |
| `null` | `Option<T>` | `None` = null, `Some(v)` = ada nilai |
| `array` | `Vec<T>` / `HashMap<K,V>` | typed |
| `object` | `struct` / `enum` | |
| `mixed` | `enum` dengan variants | |

---

## 3. Constants

### PHP
```php
define('MAX_SIZE', 100);
const VERSION = "1.0";

echo MAX_SIZE;
echo VERSION;
```

### Rust
```rust
const MAX_SIZE: u32 = 100;
const VERSION: &str = "1.0";

// Static (global mutable — avoid bila boleh)
static GREETING: &str = "Hello";

println!("{}", MAX_SIZE);
println!("{}", VERSION);
```

---

## 4. Strings

### PHP
```php
$name = "World";
echo "Hello $name";           // interpolation
echo 'Hello ' . $name;       // concatenation
echo strlen($name);           // length
echo strtoupper($name);       // uppercase
echo strtolower($name);       // lowercase
echo str_replace("o","0",$name); // replace
echo substr($name, 1, 3);    // substring
echo strpos($name, "or");    // find position
echo trim("  hello  ");      // trim
echo str_repeat("ab", 3);    // "ababab"
echo str_word_count($name);  // word count
echo strrev($name);          // reverse
echo sprintf("%05d", 42);    // format
```

### Rust
```rust
let name = "World";
let name_s = String::from("World");

println!("Hello {}", name);               // interpolation
let combined = "Hello ".to_string() + name; // concatenation
println!("{}", name.len());               // strlen
println!("{}", name.to_uppercase());      // strtoupper
println!("{}", name.to_lowercase());      // strtolower
println!("{}", name.replace("o", "0"));  // str_replace
println!("{}", &name[1..4]);             // substr(1,3)
println!("{:?}", name.find("or"));       // strpos → Option<usize>
println!("{}", "  hello  ".trim());      // trim
println!("{}", "ab".repeat(3));          // str_repeat → "ababab"
println!("{}", name.split_whitespace().count()); // str_word_count
let rev: String = name.chars().rev().collect(); // strrev
println!("{:05}", 42);                   // sprintf "%05d"
```

### Escape Characters

| PHP | Rust |
|---|---|
| `\n` | `\n` |
| `\t` | `\t` |
| `\r` | `\r` |
| `\\` | `\\` |
| `\"` | `\"` |
| `\$` (dalam double-quote) | tiada — `{}` untuk interpolation |

### Heredoc / Multiline

#### PHP
```php
$text = <<<EOT
Line one
Line two
EOT;
```

#### Rust
```rust
let text = "Line one
Line two";

// Atau dengan raw string (tiada escape processing)
let raw = r#"Line with "quotes" and \n literal backslash"#;
```

---

## 5. Numbers

### PHP
```php
echo abs(-5);          // 5
echo ceil(4.3);        // 5
echo floor(4.7);       // 4
echo round(4.5);       // 5
echo sqrt(16);         // 4
echo pow(2, 8);        // 256
echo max(1,2,3);       // 3
echo min(1,2,3);       // 1
echo rand(1, 100);     // random
echo intdiv(7, 2);     // 3
echo fmod(7.5, 2.0);   // 1.5
echo pi();             // 3.14159...
echo number_format(1234567.891, 2); // "1,234,567.89"
```

### Rust
```rust
println!("{}", i32::abs(-5));           // abs
println!("{}", f64::ceil(4.3));         // ceil
println!("{}", f64::floor(4.7));        // floor
println!("{}", f64::round(4.5));        // round
println!("{}", f64::sqrt(16.0));        // sqrt
println!("{}", i32::pow(2, 8));         // pow
println!("{}", [1,2,3].iter().max());   // max
println!("{}", [1,2,3].iter().min());   // min

use rand::Rng;
let n = rand::thread_rng().gen_range(1..=100); // rand (crate: rand)

println!("{}", 7 / 2);                 // intdiv = 3 (integer division)
println!("{}", 7.5f64 % 2.0);         // fmod
println!("{}", std::f64::consts::PI);  // pi()

// number_format — guna format dengan separator manual atau numfmt crate
```

---

## 6. Operators

### PHP
```php
// Arithmetic
$a + $b; $a - $b; $a * $b; $a / $b; $a % $b; $a ** $b;

// Assignment
$a = 1; $a += 1; $a -= 1; $a *= 2; $a /= 2; $a %= 3;

// Comparison
$a == $b;   // equal (loose)
$a === $b;  // identical (strict type)
$a != $b;   $a !== $b;
$a > $b;    $a < $b;    $a >= $b;   $a <= $b;
$a <=> $b;  // spaceship

// Logical
$a && $b;   $a || $b;   !$a;
$a and $b;  $a or $b;   $a xor $b;

// Null coalescing
$val = $x ?? "default";
$x ??= "default";  // assign if null
```

### Rust
```rust
// Arithmetic
a + b; a - b; a * b; a / b; a % b; i32::pow(a, b);

// Assignment
let mut a = 1; a += 1; a -= 1; a *= 2; a /= 2; a %= 3;

// Comparison
a == b;   // equal (strict — Rust tiada loose ==)
a != b;
a > b;    a < b;    a >= b;   a <= b;
a.cmp(&b); // → Ordering::{Less, Equal, Greater} (setara <=>)

// Logical
a && b;   a || b;   !a;

// Null coalescing (Option)
let val = x.unwrap_or("default");          // ?? "default"
let val = x.unwrap_or_else(|| compute());  // ?? lazy
x.get_or_insert("default");               // ??=
```

> Rust tiada `==` loose — semua comparison adalah strict (setara PHP `===`).

---

## 7. Conditionals

### if / else if / else

#### PHP
```php
if ($age >= 18) {
    echo "Adult";
} elseif ($age >= 13) {
    echo "Teen";
} else {
    echo "Child";
}
```

#### Rust
```rust
if age >= 18 {
    println!("Adult");
} else if age >= 13 {
    println!("Teen");
} else {
    println!("Child");
}

// if sebagai expression
let label = if age >= 18 { "Adult" } else { "Minor" };
```

### switch / match

#### PHP
```php
switch ($day) {
    case "Mon": echo "Monday";    break;
    case "Tue": echo "Tuesday";   break;
    default:    echo "Other";
}
```

#### Rust
```rust
match day {
    "Mon" => println!("Monday"),
    "Tue" => println!("Tuesday"),
    _     => println!("Other"),   // default
}

// match sebagai expression
let name = match day {
    "Mon" => "Monday",
    "Tue" => "Tuesday",
    _     => "Other",
};

// match dengan multiple patterns
match day {
    "Sat" | "Sun" => println!("Weekend"),
    _             => println!("Weekday"),
}
```

### Ternary

#### PHP
```php
$result = ($score >= 50) ? "Pass" : "Fail";
```

#### Rust
```rust
let result = if score >= 50 { "Pass" } else { "Fail" };
```

---

## 8. Loops

### while

#### PHP
```php
$i = 0;
while ($i < 5) {
    echo $i;
    $i++;
}
```

#### Rust
```rust
let mut i = 0;
while i < 5 {
    println!("{}", i);
    i += 1;
}
```

### do...while

#### PHP
```php
$i = 0;
do {
    echo $i;
    $i++;
} while ($i < 5);
```

#### Rust
```rust
let mut i = 0;
loop {
    println!("{}", i);
    i += 1;
    if i >= 5 { break; }
}
```

### for

#### PHP
```php
for ($i = 0; $i < 5; $i++) {
    echo $i;
}
```

#### Rust
```rust
for i in 0..5 {        // 0,1,2,3,4
    println!("{}", i);
}

for i in 0..=5 {       // 0,1,2,3,4,5 (inclusive)
    println!("{}", i);
}
```

### foreach

#### PHP
```php
$fruits = ["apple", "banana", "cherry"];
foreach ($fruits as $fruit) {
    echo $fruit;
}

foreach ($fruits as $index => $fruit) {
    echo "$index: $fruit";
}

// Associative
foreach ($map as $key => $value) {
    echo "$key = $value";
}
```

#### Rust
```rust
let fruits = vec!["apple", "banana", "cherry"];

for fruit in &fruits {
    println!("{}", fruit);
}

for (index, fruit) in fruits.iter().enumerate() {
    println!("{}: {}", index, fruit);
}

// HashMap
use std::collections::HashMap;
let map = HashMap::from([("a", 1), ("b", 2)]);
for (key, value) in &map {
    println!("{} = {}", key, value);
}
```

### break / continue

#### PHP
```php
for ($i = 0; $i < 10; $i++) {
    if ($i == 3) continue;
    if ($i == 7) break;
    echo $i;
}
```

#### Rust
```rust
for i in 0..10 {
    if i == 3 { continue; }
    if i == 7 { break; }
    println!("{}", i);
}

// break dengan nilai (dari loop)
let result = loop {
    if condition { break 42; }
};
```

---

## 9. Arrays

### Indexed Array

#### PHP
```php
$fruits = ["apple", "banana", "cherry"];
echo $fruits[0];           // apple
$fruits[] = "date";        // append
echo count($fruits);       // 4
array_push($fruits, "elderberry");
array_pop($fruits);
array_shift($fruits);      // remove first
array_unshift($fruits, "avocado"); // prepend
echo in_array("banana", $fruits);
sort($fruits);
rsort($fruits);
$sliced = array_slice($fruits, 1, 2);
$merged = array_merge($a, $b);
$unique = array_unique($fruits);
$reversed = array_reverse($fruits);
echo implode(", ", $fruits);
$arr = explode(",", "a,b,c");
```

#### Rust
```rust
let mut fruits = vec!["apple", "banana", "cherry"];
println!("{}", fruits[0]);          // apple
fruits.push("date");                // append
println!("{}", fruits.len());       // count
fruits.push("elderberry");          // array_push
fruits.pop();                       // array_pop
fruits.remove(0);                   // array_shift
fruits.insert(0, "avocado");       // array_unshift
println!("{}", fruits.contains(&"banana")); // in_array
fruits.sort();                      // sort
fruits.sort_by(|a,b| b.cmp(a));    // rsort
let sliced = &fruits[1..3];        // array_slice
let merged: Vec<_> = a.iter().chain(b.iter()).collect(); // array_merge
fruits.dedup();                     // array_unique (selepas sort)
fruits.reverse();                   // array_reverse
println!("{}", fruits.join(", ")); // implode
let arr: Vec<&str> = "a,b,c".split(',').collect(); // explode
```

### Associative Array (HashMap)

#### PHP
```php
$person = [
    "name"  => "Alice",
    "age"   => 30,
    "email" => "alice@example.com",
];

echo $person["name"];
$person["phone"] = "012-3456789";
unset($person["email"]);
echo array_key_exists("age", $person);
echo isset($person["name"]);
$keys   = array_keys($person);
$values = array_values($person);
ksort($person);   // sort by key
asort($person);   // sort by value, preserve keys
```

#### Rust
```rust
use std::collections::HashMap;

let mut person: HashMap<&str, &str> = HashMap::from([
    ("name",  "Alice"),
    ("email", "alice@example.com"),
]);

println!("{}", person["name"]);
person.insert("phone", "012-3456789");  // tambah / update
person.remove("email");                 // unset
println!("{}", person.contains_key("age")); // array_key_exists
println!("{}", person.get("name").is_some()); // isset

let keys:   Vec<&&str> = person.keys().collect();
let values: Vec<&&str> = person.values().collect();

// sort by key
let mut sorted: Vec<_> = person.iter().collect();
sorted.sort_by_key(|(k, _)| *k);          // ksort
sorted.sort_by_key(|(_, v)| *v);          // asort
```

### Array Functions

| PHP | Rust |
|---|---|
| `count($arr)` | `arr.len()` |
| `array_push($arr, $v)` | `arr.push(v)` |
| `array_pop($arr)` | `arr.pop()` |
| `array_shift($arr)` | `arr.remove(0)` |
| `array_unshift($arr, $v)` | `arr.insert(0, v)` |
| `in_array($v, $arr)` | `arr.contains(&v)` |
| `array_search($v, $arr)` | `arr.iter().position(\|x\| x == &v)` |
| `array_reverse($arr)` | `arr.iter().rev().collect()` |
| `sort($arr)` | `arr.sort()` |
| `rsort($arr)` | `arr.sort_by(\|a,b\| b.cmp(a))` |
| `array_unique($arr)` | `arr.sort(); arr.dedup()` |
| `array_slice($arr, $off, $n)` | `&arr[off..off+n]` |
| `array_merge($a, $b)` | `a.extend(b)` |
| `implode($sep, $arr)` | `arr.join(sep)` |
| `explode($sep, $str)` | `str.split(sep).collect()` |
| `array_map($fn, $arr)` | `arr.iter().map(fn).collect()` |
| `array_filter($arr, $fn)` | `arr.iter().filter(fn).collect()` |
| `array_reduce($arr, $fn, $init)` | `arr.iter().fold(init, fn)` |
| `array_keys($arr)` | `map.keys().collect()` |
| `array_values($arr)` | `map.values().collect()` |
| `array_key_exists($k, $arr)` | `map.contains_key(&k)` |
| `range(1, 10)` | `(1..=10).collect::<Vec<_>>()` |

---

## 10. Functions

### PHP
```php
function greet(string $name, string $greeting = "Hello"): string {
    return "$greeting, $name!";
}

echo greet("Alice");
echo greet("Bob", "Hi");

// Variadic
function sum(int ...$nums): int {
    return array_sum($nums);
}
echo sum(1, 2, 3, 4); // 10

// Anonymous / closure
$double = function($x) { return $x * 2; };
$triple = fn($x) => $x * 3; // arrow function

// Passing function
function apply(callable $fn, int $x): int {
    return $fn($x);
}
echo apply($double, 5); // 10
```

### Rust
```rust
fn greet(name: &str, greeting: &str) -> String {
    format!("{}, {}!", greeting, name)
}

println!("{}", greet("Alice", "Hello"));
println!("{}", greet("Bob", "Hi"));

// Default parameter → overloading atau Option
fn greet_default(name: &str, greeting: Option<&str>) -> String {
    let g = greeting.unwrap_or("Hello");
    format!("{}, {}!", g, name)
}

// Variadic → slice
fn sum(nums: &[i32]) -> i32 {
    nums.iter().sum()
}
println!("{}", sum(&[1, 2, 3, 4])); // 10

// Closure
let double = |x: i32| x * 2;
let triple = |x: i32| x * 3;

// Higher-order function
fn apply<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 {
    f(x)
}
println!("{}", apply(double, 5)); // 10
```

### Return Multiple Values

#### PHP
```php
function minMax(array $arr): array {
    return [min($arr), max($arr)];
}
[$min, $max] = minMax([3, 1, 4, 1, 5]);
```

#### Rust
```rust
fn min_max(arr: &[i32]) -> (i32, i32) {
    (*arr.iter().min().unwrap(), *arr.iter().max().unwrap())
}

let (min, max) = min_max(&[3, 1, 4, 1, 5]);
println!("{} {}", min, max);
```

---

## 11. Error Handling

### PHP
```php
// try/catch
try {
    if (!file_exists("file.txt")) {
        throw new Exception("File not found");
    }
} catch (Exception $e) {
    echo "Error: " . $e->getMessage();
} finally {
    echo "Always runs";
}

// Custom exception
class ValidationException extends Exception {}

try {
    throw new ValidationException("Invalid input");
} catch (ValidationException $e) {
    echo $e->getMessage();
}
```

### Rust
```rust
use std::fs;

// Result<T, E> — setara try/catch
match fs::read_to_string("file.txt") {
    Ok(content)  => println!("{}", content),
    Err(e)       => eprintln!("Error: {}", e),
}

// ? operator — propagate error (setara throw)
fn read_file(path: &str) -> Result<String, std::io::Error> {
    let content = fs::read_to_string(path)?;  // throw jika error
    Ok(content)
}

// Custom error type (guna thiserror)
use thiserror::Error;

#[derive(Debug, Error)]
enum AppError {
    #[error("File not found: {0}")]
    NotFound(String),
    #[error("Validation error: {0}")]
    Validation(String),
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
}

fn validate(input: &str) -> Result<(), AppError> {
    if input.is_empty() {
        return Err(AppError::Validation("Input kosong".into()));
    }
    Ok(())
}

// "finally" → defer dengan Drop atau struktur
// Rust tiada finally — guna RAII (destructor dipanggil auto)
```

| PHP | Rust |
|---|---|
| `try { } catch { }` | `match result { Ok() => , Err() => }` |
| `throw new Exception(...)` | `return Err(AppError::...)` |
| `$e->getMessage()` | `e.to_string()` |
| `finally` | RAII — destructor dipanggil auto |
| `Exception` | `std::error::Error` trait |
| Custom exception class | `#[derive(Error)]` enum |

---

## 12. Object-Oriented Programming

### Class & Object

#### PHP
```php
class Animal {
    public string $name;
    protected int $age;
    private string $secret = "hidden";

    public function __construct(string $name, int $age) {
        $this->name = $name;
        $this->age  = $age;
    }

    public function speak(): string {
        return "{$this->name} says hello";
    }

    public static function create(string $name): static {
        return new static($name, 0);
    }
}

$a = new Animal("Cat", 3);
echo $a->speak();
echo Animal::create("Dog")->name;
```

#### Rust
```rust
pub struct Animal {
    pub name: String,
    age:      u32,       // private by default dalam module
}

impl Animal {
    // Constructor (setara __construct)
    pub fn new(name: &str, age: u32) -> Self {
        Animal { name: name.to_string(), age }
    }

    // Static method (setara static::create)
    pub fn create(name: &str) -> Self {
        Animal::new(name, 0)
    }

    // Instance method
    pub fn speak(&self) -> String {
        format!("{} says hello", self.name)
    }

    // Mutable method
    pub fn grow_older(&mut self) {
        self.age += 1;
    }
}

let a = Animal::new("Cat", 3);
println!("{}", a.speak());
let d = Animal::create("Dog");
println!("{}", d.name);
```

### Inheritance

#### PHP
```php
class Dog extends Animal {
    public function speak(): string {
        return "{$this->name} says Woof!";
    }
}
```

#### Rust — Traits (tiada extends, guna trait)
```rust
trait Speak {
    fn speak(&self) -> String;
    fn name(&self) -> &str;
}

struct Dog { name: String }
struct Cat { name: String }

impl Speak for Dog {
    fn speak(&self) -> String { format!("{} says Woof!", self.name) }
    fn name(&self) -> &str    { &self.name }
}

impl Speak for Cat {
    fn speak(&self) -> String { format!("{} says Meow!", self.name) }
    fn name(&self) -> &str    { &self.name }
}

// Polymorphism via trait object
fn make_noise(animal: &dyn Speak) {
    println!("{}", animal.speak());
}

make_noise(&Dog { name: "Rex".into() });
make_noise(&Cat { name: "Whiskers".into() });
```

### Interface

#### PHP
```php
interface Drawable {
    public function draw(): string;
    public function area(): float;
}

class Circle implements Drawable {
    public function __construct(private float $radius) {}
    public function draw(): string  { return "Drawing circle"; }
    public function area(): float   { return M_PI * $this->radius ** 2; }
}
```

#### Rust
```rust
trait Drawable {
    fn draw(&self) -> String;
    fn area(&self) -> f64;
}

struct Circle { radius: f64 }

impl Drawable for Circle {
    fn draw(&self) -> String { "Drawing circle".into() }
    fn area(&self) -> f64   { std::f64::consts::PI * self.radius.powi(2) }
}
```

### Abstract Class

#### PHP
```php
abstract class Shape {
    abstract public function area(): float;

    public function describe(): string {
        return "Area is " . $this->area();
    }
}
```

#### Rust — trait dengan default method
```rust
trait Shape {
    fn area(&self) -> f64; // abstract — mesti implement

    fn describe(&self) -> String { // default method
        format!("Area is {}", self.area())
    }
}

struct Rectangle { width: f64, height: f64 }

impl Shape for Rectangle {
    fn area(&self) -> f64 { self.width * self.height }
}

let r = Rectangle { width: 4.0, height: 5.0 };
println!("{}", r.describe()); // "Area is 20"
```

### Magic Methods / Traits

| PHP Magic Method | Rust Equivalent |
|---|---|
| `__construct` | `fn new() -> Self` dalam `impl` |
| `__toString` | `impl std::fmt::Display` |
| `__clone` | `#[derive(Clone)]` |
| `__get` / `__set` | tiada — akses field terus atau getter/setter fn |
| `__destruct` | `impl Drop` |
| `__invoke` | `impl Fn` / closure |

```rust
use std::fmt;

struct Point { x: f64, y: f64 }

// __toString
impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

// __destruct
impl Drop for Point {
    fn drop(&mut self) {
        println!("Point dropped");
    }
}

let p = Point { x: 1.0, y: 2.0 };
println!("{}", p);  // "(1, 2)"
```

---

## 13. File Handling

### PHP
```php
// Baca fail
$content = file_get_contents("file.txt");
$lines   = file("file.txt");               // array of lines

// Tulis fail
file_put_contents("file.txt", "Hello");
file_put_contents("file.txt", "More", FILE_APPEND);

// Guna fopen
$fh = fopen("file.txt", "r");
while (!feof($fh)) {
    echo fgets($fh);
}
fclose($fh);

// Check exists
if (file_exists("file.txt")) { ... }
unlink("file.txt");         // delete
rename("old.txt", "new.txt");
copy("src.txt", "dst.txt");
mkdir("newdir");
rmdir("newdir");
```

### Rust
```rust
use std::fs;
use std::io::{self, BufRead, Write};

// Baca fail
let content = fs::read_to_string("file.txt")?;           // file_get_contents
let lines: Vec<String> = content.lines()
    .map(String::from).collect();                         // file()

// Tulis fail
fs::write("file.txt", "Hello")?;                         // file_put_contents
let mut file = fs::OpenOptions::new()
    .append(true).open("file.txt")?;
writeln!(file, "More")?;                                  // FILE_APPEND

// Line-by-line (fgets)
use std::io::BufReader;
use std::fs::File;
let fh = BufReader::new(File::open("file.txt")?);
for line in fh.lines() {
    println!("{}", line?);
}

// Check exists
if std::path::Path::new("file.txt").exists() { }
fs::remove_file("file.txt")?;                            // unlink
fs::rename("old.txt", "new.txt")?;                       // rename
fs::copy("src.txt", "dst.txt")?;                         // copy
fs::create_dir("newdir")?;                               // mkdir
fs::remove_dir("newdir")?;                               // rmdir
```

| PHP | Rust |
|---|---|
| `file_get_contents` | `fs::read_to_string` |
| `file_put_contents` | `fs::write` |
| `fopen / fclose` | `File::open` / auto-close (Drop) |
| `fgets` | `BufReader::lines()` |
| `file_exists` | `Path::new(...).exists()` |
| `unlink` | `fs::remove_file` |
| `rename` | `fs::rename` |
| `copy` | `fs::copy` |
| `mkdir` | `fs::create_dir` |
| `rmdir` | `fs::remove_dir` |

---

## 14. Regular Expressions

### PHP
```php
// Test match
preg_match('/^\d+$/', "123");         // returns 1 or 0

// Capture groups
preg_match('/(\d{4})-(\d{2})-(\d{2})/', "2024-03-20", $m);
echo $m[1]; // 2024

// Find all
preg_match_all('/\d+/', "3 cats and 5 dogs", $matches);
print_r($matches[0]); // [3, 5]

// Replace
echo preg_replace('/\s+/', '-', "hello world"); // "hello-world"

// Split
$parts = preg_split('/[\s,]+/', "one, two three");
```

### Rust — crate `regex`
```rust
use regex::Regex;

// Test match
let re = Regex::new(r"^\d+$").unwrap();
println!("{}", re.is_match("123")); // true

// Capture groups
let re = Regex::new(r"(\d{4})-(\d{2})-(\d{2})").unwrap();
if let Some(caps) = re.captures("2024-03-20") {
    println!("{}", &caps[1]); // 2024
    println!("{}", &caps[2]); // 03
}

// Find all
let re = Regex::new(r"\d+").unwrap();
let nums: Vec<&str> = re.find_iter("3 cats and 5 dogs")
    .map(|m| m.as_str()).collect();
println!("{:?}", nums); // ["3", "5"]

// Replace
let re = Regex::new(r"\s+").unwrap();
println!("{}", re.replace_all("hello world", "-")); // "hello-world"

// Split
let re = Regex::new(r"[\s,]+").unwrap();
let parts: Vec<&str> = re.split("one, two three").collect();
```

```toml
# Cargo.toml
[dependencies]
regex = "1"
```

---

## 15. Date & Time

### PHP
```php
echo date("Y-m-d");              // "2024-03-20"
echo date("H:i:s");              // "14:30:00"
echo date("D, d M Y");           // "Wed, 20 Mar 2024"
echo time();                     // Unix timestamp

$ts = mktime(12, 0, 0, 3, 20, 2024);
echo date("Y-m-d", $ts);

echo date("Y-m-d", strtotime("+7 days"));
echo date_diff(
    date_create("2024-01-01"),
    date_create("2024-12-31")
)->days;
```

### Rust — crate `chrono`
```rust
use chrono::{Local, Utc, Duration, NaiveDate};

// Current date/time
let now = Local::now();
println!("{}", now.format("%Y-%m-%d"));       // date("Y-m-d")
println!("{}", now.format("%H:%M:%S"));       // date("H:i:s")
println!("{}", now.format("%a, %d %b %Y"));  // date("D, d M Y")
println!("{}", now.timestamp());              // time()

// Specific datetime
let dt = NaiveDate::from_ymd_opt(2024, 3, 20).unwrap()
    .and_hms_opt(12, 0, 0).unwrap();

// Add duration
let later = now + Duration::days(7);         // strtotime("+7 days")
println!("{}", later.format("%Y-%m-%d"));

// Difference
let d1 = NaiveDate::from_ymd_opt(2024, 1,  1).unwrap();
let d2 = NaiveDate::from_ymd_opt(2024, 12, 31).unwrap();
let diff = d2.signed_duration_since(d1);
println!("{} days", diff.num_days());        // date_diff → .days
```

```toml
[dependencies]
chrono = { version = "0.4", features = ["serde"] }
```

---

## 16. Math Functions

| PHP | Rust |
|---|---|
| `abs($x)` | `x.abs()` |
| `ceil($x)` | `x.ceil()` |
| `floor($x)` | `x.floor()` |
| `round($x)` | `x.round()` |
| `sqrt($x)` | `x.sqrt()` |
| `pow($x,$n)` | `x.powi(n)` / `x.powf(n)` |
| `log($x)` | `x.ln()` |
| `log10($x)` | `x.log10()` |
| `exp($x)` | `x.exp()` |
| `max($a,$b)` | `a.max(b)` |
| `min($a,$b)` | `a.min(b)` |
| `pi()` | `std::f64::consts::PI` |
| `M_E` | `std::f64::consts::E` |
| `intdiv($a,$b)` | `a / b` (integer division) |
| `fmod($a,$b)` | `a % b` |
| `rand($min,$max)` | `rng.gen_range(min..=max)` (crate: rand) |
| `mt_rand()` | `rand::random::<u32>()` |
| `number_format($n,$dec)` | `format!("{:.dec$}", n)` |
| `is_nan($x)` | `x.is_nan()` |
| `is_finite($x)` | `x.is_finite()` |
| `is_infinite($x)` | `x.is_infinite()` |

---

## 17. Type Checking

### PHP
```php
is_int($x);     is_float($x);   is_string($x);
is_bool($x);    is_null($x);    is_array($x);
is_object($x);  is_numeric($x);
gettype($x);    // returns "integer", "string", etc.
```

### Rust — compile-time, tiada runtime type check diperlukan
```rust
// Rust: type diketahui pada compile time
// Untuk dynamic/enum variants guna match

enum Value {
    Int(i32),
    Float(f64),
    Str(String),
    Bool(bool),
    Null,
}

fn describe(v: &Value) {
    match v {
        Value::Int(n)   => println!("int: {}", n),
        Value::Float(f) => println!("float: {}", f),
        Value::Str(s)   => println!("string: {}", s),
        Value::Bool(b)  => println!("bool: {}", b),
        Value::Null     => println!("null"),
    }
}

// Untuk numeric string check
let s = "42";
let is_numeric = s.parse::<f64>().is_ok(); // is_numeric()
```

---

## 18. Output & Formatting

### PHP
```php
echo "Hello";
print "Hello";
echo "Value: $var";
printf("%.2f", 3.14159);
$s = sprintf("Name: %s, Age: %d", "Alice", 30);
var_dump($var);        // debug
print_r($array);       // pretty print array
```

### Rust
```rust
println!("Hello");                              // echo
print!("Hello");                                // echo (no newline)
println!("Value: {}", var);                     // string interpolation
println!("{:.2}", 3.14159_f64);                 // printf("%.2f")
let s = format!("Name: {}, Age: {}", "Alice", 30); // sprintf
println!("{:?}", var);                          // var_dump (Debug)
println!("{:#?}", array);                       // print_r (pretty Debug)
eprintln!("Error: {}", msg);                    // stderr output
```

### Format Specifiers

| PHP `printf` | Rust `println!` |
|---|---|
| `%s` | `{}` |
| `%d` | `{}` (integer) |
| `%f` | `{}` |
| `%.2f` | `{:.2}` |
| `%05d` | `{:05}` |
| `%-10s` | `{:<10}` |
| `%10s` | `{:>10}` |
| `%b` | `{:b}` (binary) |
| `%x` | `{:x}` (hex) |
| `%o` | `{:o}` (octal) |
| `%e` | `{:e}` (scientific) |

---

## 19. Scope & Closures

### PHP
```php
$x = 10;

function noAccess() {
    // echo $x; // ERROR — PHP tiada implicit capture
}

function withGlobal() {
    global $x;
    echo $x; // 10
}

// Closure capture
$multiplier = 3;
$fn = function($n) use ($multiplier) {
    return $n * $multiplier;
};

// Capture by reference
$fn2 = function($n) use (&$multiplier) {
    $multiplier += 1;
    return $n * $multiplier;
};
```

### Rust
```rust
let x = 10;

fn no_access() {
    // println!("{}", x); // ERROR — Rust juga tiada implicit capture dalam fn
}

// Closure capture (auto-detect borrow/move)
let multiplier = 3;
let fn1 = |n: i32| n * multiplier;       // capture by borrow

// Capture by move
let fn2 = move |n: i32| n * multiplier;  // clone nilai ke dalam closure

// Mutable capture
let mut count = 0;
let mut increment = || { count += 1; count };
println!("{}", increment()); // 1
println!("{}", increment()); // 2
```

---

## 20. Enums

### PHP
```php
// PHP 8.1+ Backed Enum
enum Status: string {
    case Active   = 'active';
    case Inactive = 'inactive';
    case Pending  = 'pending';
}

echo Status::Active->value;   // "active"
$s = Status::from('active');  // → Status::Active
```

### Rust — Enum (lebih powerful)
```rust
// Basic enum
#[derive(Debug, PartialEq)]
enum Status {
    Active,
    Inactive,
    Pending,
}

// Enum dengan data (tiada dalam PHP)
#[derive(Debug)]
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    Color(u8, u8, u8),
}

let msg = Message::Move { x: 10, y: 20 };
match msg {
    Message::Quit             => println!("Quit"),
    Message::Move { x, y }   => println!("Move to {},{}", x, y),
    Message::Write(s)         => println!("Write: {}", s),
    Message::Color(r, g, b)   => println!("Color: {},{},{}", r, g, b),
}

// Backed enum (setara PHP 8.1)
#[derive(Debug, PartialEq)]
enum StatusStr {
    Active,
    Inactive,
}

impl StatusStr {
    fn value(&self) -> &str {
        match self {
            StatusStr::Active   => "active",
            StatusStr::Inactive => "inactive",
        }
    }
    fn from_str(s: &str) -> Option<Self> {
        match s {
            "active"   => Some(StatusStr::Active),
            "inactive" => Some(StatusStr::Inactive),
            _          => None,
        }
    }
}
```

---

## 21. Generics (Tiada dalam PHP — Rust only)

```rust
// Generic function
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest { largest = item; }
    }
    largest
}

println!("{}", largest(&[34, 50, 25, 100])); // 100
println!("{}", largest(&["apple", "mango"])); // "mango"

// Generic struct
struct Pair<T> {
    first:  T,
    second: T,
}

impl<T: std::fmt::Display> Pair<T> {
    fn show(&self) {
        println!("{} and {}", self.first, self.second);
    }
}

Pair { first: 5, second: 10 }.show();
Pair { first: "a", second: "b" }.show();
```

---

## 22. Ownership & Borrowing (Rust Unik)

> Tiada equivalent dalam PHP. Ini adalah konsep terpenting dalam Rust.

```rust
// OWNERSHIP — setiap nilai ada satu owner
let s1 = String::from("hello");
let s2 = s1;           // s1 di-MOVE ke s2 — s1 tidak boleh digunakan lagi
// println!("{}", s1); // ERROR: s1 moved

// CLONE — deep copy explicit
let s1 = String::from("hello");
let s2 = s1.clone();   // kedua-dua valid
println!("{} {}", s1, s2);

// BORROWING — pinjam tanpa ambil ownership
fn print_len(s: &String) {
    println!("{}", s.len());
}
let s = String::from("hello");
print_len(&s);         // pinjam dengan &
println!("{}", s);     // s masih valid

// MUTABLE BORROW — satu pada satu masa
fn add_world(s: &mut String) {
    s.push_str(", world");
}
let mut s = String::from("hello");
add_world(&mut s);
println!("{}", s);    // "hello, world"
```

---

## 23. Cargo & Project Structure

### PHP (Composer)
```
composer.json        ← dependencies
vendor/              ← packages
src/                 ← source
index.php            ← entry point
```

### Rust (Cargo)
```
Cargo.toml           ← setara composer.json
Cargo.lock           ← setara composer.lock
src/
  main.rs            ← binary entry point
  lib.rs             ← library entry point
  module/
    mod.rs
target/              ← compiled output (gitignore)
```

```toml
# Cargo.toml
[package]
name    = "my-app"
version = "0.1.0"
edition = "2021"

[dependencies]
serde       = { version = "1", features = ["derive"] }
serde_json  = "1"
tokio       = { version = "1", features = ["full"] }
sqlx        = { version = "0.7", features = ["mysql", "runtime-tokio-rustls"] }
chrono      = "0.4"
regex       = "1"
thiserror   = "1"
rand        = "0.8"
```

```bash
cargo new my-app        # composer create-project
cargo build             # compile
cargo run               # php index.php
cargo test              # phpunit
cargo add serde         # composer require
cargo update            # composer update
cargo doc --open        # phpdoc
```

---

## 24. Turbofish `::<>` (Rust Unik)

> Tiada equivalent dalam PHP. Turbofish adalah cara untuk **nyatakan jenis generik secara explicit** pada panggilan fungsi atau method — bila Rust tidak boleh infer sendiri.

### Kenapa dipanggil "turbofish"?

Sintaksnya `::<>` menyerupai ikan yang melompat — dan komuniti Rust menamaknya *turbofish* secara jenaka.

```
::<T>   ← turbofish
 ^^
 ||__ muncung ikan
 |___ badan + sirip
```

---

### Sintaks Asas

```rust
fungsi::<Jenis>(args)
value.method::<Jenis>()
```

---

### Bila perlu Turbofish?

Turbofish diperlukan bila Rust **tidak dapat infer jenis** daripada konteks. Ini berlaku dalam 3 situasi utama:

#### 1. `parse()` — tukar String kepada jenis lain

Tanpa turbofish, Rust tidak tahu nak parse kepada jenis apa:

```rust
// ❌ ERROR — Rust tidak tahu jenis apa
let n = "42".parse().unwrap();

// ✅ Cara 1: turbofish
let n = "42".parse::<i32>().unwrap();

// ✅ Cara 2: type annotation (sama kuat, berbeza gaya)
let n: i32 = "42".parse().unwrap();
```

#### 2. `collect()` — kumpul iterator ke dalam collection

```rust
let words = vec!["hello", "world"];

// ❌ ERROR — collect kepada apa?
let upper = words.iter().map(|w| w.to_uppercase()).collect();

// ✅ Turbofish — nyata jenis collection
let upper = words.iter().map(|w| w.to_uppercase()).collect::<Vec<String>>();

// ✅ Atau type annotation
let upper: Vec<String> = words.iter().map(|w| w.to_uppercase()).collect();
```

#### 3. Fungsi generik — pilih implementasi spesifik

```rust
fn buat_default<T: Default>() -> T {
    T::default()
}

// ❌ ERROR — T apa?
let x = buat_default();

// ✅ Turbofish — specify T
let x = buat_default::<i32>();   // → 0
let y = buat_default::<String>(); // → ""
let z = buat_default::<Vec<u8>>(); // → []
```

---

### Contoh Lanjut

#### `String::from` vs turbofish pada method chaining

```rust
// Turbofish berguna dalam method chain panjang
let result = ["1", "2", "3", "4", "5"]
    .iter()
    .map(|s| s.parse::<i32>().unwrap())
    .filter(|n| n % 2 == 0)
    .collect::<Vec<i32>>();

println!("{:?}", result);  // [2, 4]
```

#### HashMap collect

```rust
use std::collections::HashMap;

let pairs = vec![("a", 1), ("b", 2), ("c", 3)];

// Turbofish untuk HashMap
let map = pairs.into_iter().collect::<HashMap<&str, i32>>();

// Atau underscore — Rust infer dari annotation
let map: HashMap<_, _> = pairs.into_iter().collect();
```

#### Fungsi dalam standard library

```rust
// Vec::with_capacity — pilih jenis elemen
let v = Vec::<i32>::with_capacity(10);

// std::mem::size_of
let size = std::mem::size_of::<u64>();  // → 8 bytes
println!("u64 = {} bytes", size);

// Into/From dengan turbofish
let s = String::from("hello");
let bytes = Vec::<u8>::from(s.clone());
```

---

### Turbofish vs Type Annotation — Bila Guna Mana?

| Situasi | Pilihan | Contoh |
|---|---|---|
| Single assignment, jelas | Type annotation | `let x: i32 = "5".parse().unwrap()` |
| Method chain panjang | Turbofish | `.collect::<Vec<_>>()` |
| Tiada variable (terus pass) | Turbofish | `foo("5".parse::<i32>().unwrap())` |
| Generic function tanpa return assign | Turbofish | `size_of::<f64>()` |

```rust
// Tiada variable — MESTI turbofish
println!("{}", "42".parse::<i32>().unwrap() * 2);

// Ada variable — annotation lebih readable
let n: i32 = "42".parse().unwrap();
println!("{}", n * 2);
```

---

### Wildcards dalam Turbofish

Gunakan `_` bila sebahagian jenis boleh di-infer:

```rust
// Fully explicit
let v = vec![1, 2, 3].into_iter().collect::<Vec<i32>>();

// Partial — Rust infer inner type
let v = vec![1, 2, 3].into_iter().collect::<Vec<_>>();

// HashMap — infer kedua-dua K dan V
let map = pairs.into_iter().collect::<HashMap<_, _>>();
```

---

### Ringkasan

| Perkara | Keterangan |
|---|---|
| Sintaks | `fungsi::<T>()` atau `method::<T>()` |
| Tujuan | Nyatakan jenis generik bila Rust tidak dapat infer |
| Alternatif | Type annotation `let x: T = ...` |
| Bila MESTI turbofish | Tiada variable untuk letak annotation |
| Simbol `_` | Biar Rust infer sebahagian jenis |

---

## Rujukan Pantas: PHP → Rust

| Konsep PHP | Rust |
|---|---|
| `echo` / `print` | `println!` / `print!` |
| `$variable` | `let variable` |
| `null` | `Option<T>` → `None` |
| `array []` | `Vec<T>` / `HashMap<K,V>` |
| `try/catch` | `Result<T,E>` + `match` / `?` |
| `class` | `struct` + `impl` |
| `interface` | `trait` |
| `abstract class` | `trait` dengan default methods |
| `extends` | `impl Trait for Struct` |
| `function` | `fn` |
| `callable` / `closure` | `Fn` / `FnMut` / `FnOnce` closures |
| `foreach` | `for x in iter` |
| `require` / `include` | `mod` + `use` |
| `namespace` | `mod` |
| `composer.json` | `Cargo.toml` |
| `->` (method call) | `.` |
| `::` (static) | `::` |
| `$this` | `self` |
| `self::` | `Self::` |
| `new Foo()` | `Foo::new()` |
| `&$ref` (pass by ref) | `&mut` borrow |
| `var_dump()` | `println!("{:#?}", val)` |
| `isset()` | `.is_some()` |
| `empty()` | `.is_empty()` |
| `count()` | `.len()` |
| `strtotime()` | `chrono::DateTime::parse_from_str` |
| `json_encode()` | `serde_json::to_string()` |
| `json_decode()` | `serde_json::from_str()` |
| `base64_encode()` | `base64::encode()` (crate: base64) |
| `md5()` / `sha1()` | crate: `md5` / `sha1` |
| `sleep($sec)` | `std::thread::sleep(Duration::from_secs(n))` |
| `getenv()` | `std::env::var("KEY")` |
| `phpinfo()` | `rustc --version` / `cargo --version` |
