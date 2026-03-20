# PHP Arrays: Advanced Techniques → Rust

> Rujukan: https://reintech.io/blog/php-arrays-advanced-techniques-multidimensional-arrays
> Setiap pattern PHP dari artikel asal ditukar kepada Rust idiomatic.

---

## Perbezaan Asas: PHP Array vs Rust

| PHP Array | Rust Equivalent |
|---|---|
| `[]` (mixed keys, any value) | `Vec<T>` (indexed) atau `HashMap<K, V>` (keyed) |
| Associative array `['key' => val]` | `HashMap<String, V>` atau `struct` |
| Array of arrays | `Vec<Vec<T>>` atau `Vec<Struct>` |
| Dynamic typing dalam array | Rust: semua elemen mesti sama jenis — guna `enum` untuk mixed |
| `null` value | `Option<T>` |
| Pass-by-value (copy semantics) | Rust: move semantics — clone secara explicit jika perlu |

---

## 1. Multidimensional Arrays

### Basic Structure & Access

#### PHP
```php
$employees = [
    ['firstName' => 'John', 'lastName' => 'Doe',     'age' => 30, 'department' => 'Engineering'],
    ['firstName' => 'Jane', 'lastName' => 'Smith',   'age' => 28, 'department' => 'Marketing'],
    ['firstName' => 'Bob',  'lastName' => 'Johnson', 'age' => 40, 'department' => 'Engineering'],
];

echo $employees[1]['firstName']; // Jane
$email = $employees[0]['email'] ?? 'not provided';
```

#### Rust — guna struct (lebih selamat & idiomatic)
```rust
#[derive(Debug, Clone)]
struct Employee {
    first_name: String,
    last_name:  String,
    age:        u32,
    department: String,
    email:      Option<String>,  // PHP ?? 'not provided' → Option<String>
}

let employees = vec![
    Employee { first_name: "John".into(), last_name: "Doe".into(),     age: 30, department: "Engineering".into(), email: None },
    Employee { first_name: "Jane".into(), last_name: "Smith".into(),   age: 28, department: "Marketing".into(),   email: None },
    Employee { first_name: "Bob".into(),  last_name: "Johnson".into(), age: 40, department: "Engineering".into(), email: None },
];

// Akses index
println!("{}", employees[1].first_name); // Jane

// Safe access — setara ?? 'not provided'
let email = employees[0].email.as_deref().unwrap_or("not provided");
println!("{}", email);
```

---

### Three-Dimensional Arrays

#### PHP
```php
$salesData = [
    'North' => [
        'Q1' => ['Electronics' => 150000, 'Clothing' => 80000, 'Food' => 45000],
        'Q2' => ['Electronics' => 175000, 'Clothing' => 92000, 'Food' => 48000],
    ],
    'South' => [
        'Q1' => ['Electronics' => 120000, 'Clothing' => 95000, 'Food' => 52000],
        'Q2' => ['Electronics' => 135000, 'Clothing' => 98000, 'Food' => 55000],
    ],
];

$northElectronics = array_sum(array_column($salesData['North'], 'Electronics'));
```

#### Rust
```rust
use std::collections::HashMap;

// HashMap<Region, HashMap<Quarter, HashMap<Category, i64>>>
let mut sales_data: HashMap<&str, HashMap<&str, HashMap<&str, i64>>> = HashMap::new();

sales_data.insert("North", HashMap::from([
    ("Q1", HashMap::from([("Electronics", 150_000), ("Clothing", 80_000), ("Food", 45_000)])),
    ("Q2", HashMap::from([("Electronics", 175_000), ("Clothing", 92_000), ("Food", 48_000)])),
]));
sales_data.insert("South", HashMap::from([
    ("Q1", HashMap::from([("Electronics", 120_000), ("Clothing", 95_000), ("Food", 52_000)])),
    ("Q2", HashMap::from([("Electronics", 135_000), ("Clothing", 98_000), ("Food", 55_000)])),
]));

// Jumlah Electronics untuk North — setara array_sum(array_column(...))
let north_electronics: i64 = sales_data["North"]
    .values()
    .filter_map(|cats| cats.get("Electronics"))
    .sum();

println!("North Electronics Total: ${}", north_electronics);
```

---

## 2. Advanced Iteration Techniques

### Nested Loop

#### PHP
```php
foreach ($employees as $index => $employee) {
    echo "Employee #" . ($index + 1) . ": ";
    foreach ($employee as $field => $value) {
        echo ucfirst($field) . ": $value | ";
    }
}
```

#### Rust
```rust
for (index, emp) in employees.iter().enumerate() {
    println!("Employee #{}: {} {} | Age: {} | Dept: {}",
        index + 1,
        emp.first_name,
        emp.last_name,
        emp.age,
        emp.department
    );
}
```

---

### Recursive Iteration (Unknown Depth)

#### PHP
```php
function displayNestedArray(array $array, int $depth = 0): void {
    $indent = str_repeat('  ', $depth);
    foreach ($array as $key => $value) {
        if (is_array($value)) {
            echo $indent . $key . ":\n";
            displayNestedArray($value, $depth + 1);
        } else {
            echo $indent . $key . ": " . $value . "\n";
        }
    }
}
```

#### Rust — guna `serde_json::Value` untuk arbitrary nested data
```rust
use serde_json::Value;

fn display_nested(value: &Value, depth: usize) {
    let indent = "  ".repeat(depth);
    match value {
        Value::Object(map) => {
            for (key, val) in map {
                if val.is_object() || val.is_array() {
                    println!("{}{}:", indent, key);
                    display_nested(val, depth + 1);
                } else {
                    println!("{}{}: {}", indent, key, val);
                }
            }
        }
        Value::Array(arr) => {
            for (i, val) in arr.iter().enumerate() {
                println!("{}[{}]:", indent, i);
                display_nested(val, depth + 1);
            }
        }
        _ => println!("{}{}", indent, value),
    }
}

// Guna:
let data: Value = serde_json::json!({
    "North": { "Q1": { "Electronics": 150000 } }
});
display_nested(&data, 0);
```

> Untuk struct yang depth-nya diketahui: guna nested `for` biasa. `serde_json::Value` untuk data yang genuinely dynamic.

---

### Memory-Efficient Streaming (Iterators)

#### PHP
```php
function yieldLargeDataset() {
    $file = fopen('large-dataset.json', 'r');
    while ($line = fgets($file)) {
        yield json_decode($line, true);
    }
    fclose($file);
}

foreach (yieldLargeDataset() as $record) {
    // Process one at a time
}
```

#### Rust — guna `BufReader` line iterator
```rust
use std::fs::File;
use std::io::{BufRead, BufReader};
use serde_json::Value;

fn stream_large_dataset(path: &str) -> impl Iterator<Item = Value> {
    let file = File::open(path).expect("File not found");
    BufReader::new(file)
        .lines()
        .filter_map(|line| line.ok())
        .filter_map(|line| serde_json::from_str(&line).ok())
}

for record in stream_large_dataset("large-dataset.json") {
    // proses satu baris pada satu masa — tiada load keseluruhan fail
    println!("{}", record);
}
```

---

## 3. Advanced Array Functions

### array_column() — Extract Column

#### PHP
```php
$ages = array_column($employees, 'age');
$averageAge = array_sum($ages) / count($ages);

$agesByName = array_column($employees, 'age', 'firstName');
// ['John' => 30, 'Jane' => 28, 'Bob' => 40]
```

#### Rust
```rust
// Extract column → Vec
let ages: Vec<u32> = employees.iter().map(|e| e.age).collect();
let average_age = ages.iter().sum::<u32>() as f64 / ages.len() as f64;
println!("Average age: {:.1}", average_age);

// Extract dengan key (setara array_column dengan index key)
let ages_by_name: HashMap<&str, u32> = employees
    .iter()
    .map(|e| (e.first_name.as_str(), e.age))
    .collect();

println!("{:?}", ages_by_name); // {"John": 30, "Jane": 28, "Bob": 40}
```

---

### array_map() — Transform Data

#### PHP
```php
$employeesWithSalary = array_map(function($employee) {
    $employee['estimatedSalary'] = 50000 + ($employee['age'] * 1000);
    return $employee;
}, $employees);
```

#### Rust
```rust
#[derive(Debug, Clone)]
struct EmployeeWithSalary {
    first_name:       String,
    age:              u32,
    department:       String,
    estimated_salary: u32,
}

let employees_with_salary: Vec<EmployeeWithSalary> = employees
    .iter()
    .map(|e| EmployeeWithSalary {
        first_name:       e.first_name.clone(),
        age:              e.age,
        department:       e.department.clone(),
        estimated_salary: 50_000 + (e.age * 1_000),
    })
    .collect();
```

> **Nota:** Rust tidak modify-in-place dengan `array_map` — output jenis baru atau clone. Ini intentional untuk type safety.

---

### array_walk_recursive() — Modify In Place

#### PHP
```php
array_walk_recursive($salesData, function(&$value, $key) {
    if (is_numeric($value)) {
        $value = (int)($value * 100); // dollars → cents
    }
});
```

#### Rust — guna nested `iter_mut()`
```rust
// Dengan typed structure: modify terus
let mut sales: Vec<Vec<i64>> = vec![
    vec![150_000, 80_000, 45_000],
    vec![175_000, 92_000, 48_000],
];

// Walk recursive → convert dollars to cents
for row in sales.iter_mut() {
    for value in row.iter_mut() {
        *value *= 100;
    }
}
```

Untuk HashMap bersarang:
```rust
let mut data: HashMap<&str, HashMap<&str, i64>> = HashMap::from([
    ("Q1", HashMap::from([("Electronics", 150_000i64), ("Clothing", 80_000)])),
]);

for quarter in data.values_mut() {
    for amount in quarter.values_mut() {
        *amount *= 100;
    }
}
```

---

### array_merge_recursive() — Merge Configs

#### PHP
```php
$finalConfig = array_merge_recursive($defaultConfig, $userConfig);
// True override: [...$default, ...$user]
```

#### Rust — guna `extend()` atau merge manual
```rust
use std::collections::HashMap;

let mut default_config: HashMap<&str, &str> = HashMap::from([
    ("host",    "localhost"),
    ("charset", "utf8mb4"),
]);

let user_config: HashMap<&str, &str> = HashMap::from([
    ("host",     "prod-db.example.com"),
    ("password", "secure123"),
]);

// Extend: user_config override default (setara spread operator PHP)
default_config.extend(user_config);

println!("{:?}", default_config);
// host = prod-db.example.com, charset = utf8mb4, password = secure123
```

---

## 4. Sorting Multidimensional Arrays

### array_multisort() — Sort by Column

#### PHP
```php
// Sort by price DESC, then stock ASC
$prices = array_column($products, 'price');
$stock  = array_column($products, 'stock');
array_multisort($prices, SORT_DESC, $stock, SORT_ASC, $products);
```

#### Rust — guna `sort_by` dengan tuple comparison
```rust
#[derive(Debug, Clone)]
struct Product {
    name:  String,
    price: u32,
    stock: u32,
}

let mut products = vec![
    Product { name: "Laptop".into(),   price: 1200, stock: 5   },
    Product { name: "Mouse".into(),    price: 25,   stock: 150 },
    Product { name: "Keyboard".into(), price: 75,   stock: 45  },
    Product { name: "Monitor".into(),  price: 300,  stock: 12  },
];

// Sort: price DESC, kemudian stock ASC
products.sort_by(|a, b| {
    b.price.cmp(&a.price)             // DESC
        .then(a.stock.cmp(&b.stock))  // ASC jika price sama
});

for p in &products {
    println!("{}: ${} (stock: {})", p.name, p.price, p.stock);
}
```

---

### usort() — Custom Sort

#### PHP
```php
usort($employees, function($a, $b) {
    $deptCompare = strcmp($a['department'], $b['department']);
    if ($deptCompare !== 0) return $deptCompare;
    return $b['age'] <=> $a['age']; // age DESC
});
```

#### Rust
```rust
employees.sort_by(|a, b| {
    a.department.cmp(&b.department)   // dept ASC
        .then(b.age.cmp(&a.age))      // age DESC
});
```

Sort dengan weighted score:
```rust
// 70% stock, 30% inverse price
products.sort_by(|a, b| {
    let score = |p: &Product| (p.stock as f64 * 0.7) + ((1000.0 - p.price as f64) * 0.3);
    score(b).partial_cmp(&score(a)).unwrap() // higher score first
});
```

---

### uasort() — Sort Preserve Keys (HashMap)

#### PHP
```php
$employeesById = [
    'E001' => ['name' => 'John', 'performance' => 92],
    'E002' => ['name' => 'Jane', 'performance' => 88],
    'E003' => ['name' => 'Bob',  'performance' => 95],
];
uasort($employeesById, fn($a, $b) => $b['performance'] <=> $a['performance']);
```

#### Rust — sort HashMap entries
```rust
use std::collections::HashMap;

let employees_by_id: HashMap<&str, (&str, u32)> = HashMap::from([
    ("E001", ("John", 92u32)),
    ("E002", ("Jane", 88u32)),
    ("E003", ("Bob",  95u32)),
]);

// HashMap tiada order — collect ke Vec<> dan sort
let mut sorted: Vec<(&&str, &(&str, u32))> = employees_by_id.iter().collect();
sorted.sort_by(|a, b| b.1.1.cmp(&a.1.1)); // sort by performance DESC

for (id, (name, perf)) in sorted {
    println!("{}: {} - {}%", id, name, perf);
}
```

---

## 5. Practical Patterns

### groupBy() — Group by Criteria

#### PHP
```php
function groupBy(array $array, string $key): array {
    $result = [];
    foreach ($array as $item) {
        $result[$item[$key]][] = $item;
    }
    return $result;
}
$employeesByDept = groupBy($employees, 'department');
```

#### Rust
```rust
use std::collections::HashMap;

fn group_by_dept(employees: &[Employee]) -> HashMap<&str, Vec<&Employee>> {
    let mut result: HashMap<&str, Vec<&Employee>> = HashMap::new();
    for emp in employees {
        result.entry(emp.department.as_str()).or_default().push(emp);
    }
    result
}

let by_dept = group_by_dept(&employees);
for (dept, emps) in &by_dept {
    println!("{}: {} orang", dept, emps.len());
}
```

Group by multiple levels:
```rust
// Group by dept → then by age bracket
fn group_by_dept_and_age<'a>(
    employees: &'a [Employee],
) -> HashMap<&'a str, HashMap<&'static str, Vec<&'a Employee>>> {
    let mut result: HashMap<&str, HashMap<&str, Vec<&Employee>>> = HashMap::new();

    for emp in employees {
        let age_bracket = if emp.age < 30 { "junior" } else { "senior" };
        result
            .entry(emp.department.as_str())
            .or_default()
            .entry(age_bracket)
            .or_default()
            .push(emp);
    }
    result
}
```

---

### flattenArray() — Flatten Nested

#### PHP
```php
function flattenArray(array $array): array {
    $result = [];
    array_walk_recursive($array, function($value) use (&$result) {
        $result[] = $value;
    });
    return $result;
}

function flattenWithKeys(array $array, string $prefix = ''): array {
    $result = [];
    foreach ($array as $key => $value) {
        $newKey = $prefix . $key;
        if (is_array($value)) {
            $result = array_merge($result, flattenWithKeys($value, $newKey . '.'));
        } else {
            $result[$newKey] = $value;
        }
    }
    return $result;
}
// Result: ['North.Q1.Electronics' => 150000, ...]
```

#### Rust — flatten Vec<Vec<T>>
```rust
// Flatten Vec<Vec<T>>
let nested = vec![vec![1, 2, 3], vec![4, 5], vec![6]];
let flat: Vec<i32> = nested.into_iter().flatten().collect();
println!("{:?}", flat); // [1, 2, 3, 4, 5, 6]
```

Flatten dengan dot-key (setara `flattenWithKeys`):
```rust
use std::collections::HashMap;
use serde_json::Value;

fn flatten_with_keys(value: &Value, prefix: &str) -> HashMap<String, String> {
    let mut result = HashMap::new();

    match value {
        Value::Object(map) => {
            for (k, v) in map {
                let new_key = if prefix.is_empty() {
                    k.clone()
                } else {
                    format!("{}.{}", prefix, k)
                };
                result.extend(flatten_with_keys(v, &new_key));
            }
        }
        _ => {
            result.insert(prefix.to_string(), value.to_string());
        }
    }
    result
}

// Output: {"North.Q1.Electronics": "150000", ...}
```

---

### Search in Multidimensional Arrays

#### PHP
```php
function findInArray(array $array, string $key, $value): ?array {
    foreach ($array as $item) {
        if (isset($item[$key]) && $item[$key] === $value) {
            return $item;
        }
    }
    return null;
}

$seniorEngineers = array_filter($employees, fn($e) =>
    $e['department'] === 'Engineering' && $e['age'] > 35
);
```

#### Rust
```rust
// find() → Option<&Employee>
let found = employees.iter().find(|e| e.first_name == "Jane");
println!("{:?}", found);

// filter() → Vec<&Employee>
let senior_engineers: Vec<&Employee> = employees
    .iter()
    .filter(|e| e.department == "Engineering" && e.age > 35)
    .collect();

println!("{} senior engineers", senior_engineers.len());
```

---

### buildTree() — Relational / Parent-Child

#### PHP
```php
function buildTree(array $elements, $parentId = null): array {
    $branch = [];
    foreach ($elements as $element) {
        if ($element['parent_id'] === $parentId) {
            $children = buildTree($elements, $element['id']);
            if ($children) {
                $element['children'] = $children;
            }
            $branch[] = $element;
        }
    }
    return $branch;
}
```

#### Rust
```rust
#[derive(Debug, Clone)]
struct Comment {
    id:        u32,
    parent_id: Option<u32>,
    text:      String,
    children:  Vec<Comment>,
}

fn build_tree(flat: &[Comment], parent_id: Option<u32>) -> Vec<Comment> {
    flat.iter()
        .filter(|c| c.parent_id == parent_id)
        .map(|c| {
            let mut node = c.clone();
            node.children = build_tree(flat, Some(c.id));
            node
        })
        .collect()
}

let comments = vec![
    Comment { id: 1, parent_id: None,    text: "First comment".into(),  children: vec![] },
    Comment { id: 2, parent_id: Some(1), text: "Reply to first".into(), children: vec![] },
    Comment { id: 3, parent_id: Some(1), text: "Another reply".into(),  children: vec![] },
    Comment { id: 4, parent_id: None,    text: "Second comment".into(), children: vec![] },
    Comment { id: 5, parent_id: Some(2), text: "Nested reply".into(),   children: vec![] },
];

let tree = build_tree(&comments, None);
println!("{:#?}", tree);
```

---

## 6. Performance Considerations

### Memory Management — Pass by Reference

#### PHP
```php
// Bad: copies entire array
function process($array) { ... }

// Good: pass by reference
function process(array &$array) {
    foreach ($array as &$item) { ... }
    unset($item); // break reference
}
```

#### Rust — ownership/borrow = no copy by default
```rust
// Rust: fn ambil &[Employee] — tiada copy, zero-cost
fn process_employees(employees: &[Employee]) {
    for emp in employees {
        println!("{}", emp.first_name);
    }
}

// Modify in place: guna &mut
fn uppercase_names(employees: &mut Vec<Employee>) {
    for emp in employees.iter_mut() {
        emp.first_name = emp.first_name.to_uppercase();
    }
}
```

> Dalam Rust, parameter `&T` (borrow) tidak copy data — setara PHP pass-by-reference secara default.

---

### Optimizing Array Operations — Chained Iterators

#### PHP
```php
// Slow: multiple iterations
$result = [];
foreach ($employees as $emp) {
    if ($emp['age'] > 30) {
        $result[] = strtoupper($emp['firstName']);
    }
}

// Faster: chained
$result = array_map(
    'strtoupper',
    array_column(
        array_filter($employees, fn($e) => $e['age'] > 30),
        'firstName'
    )
);
```

#### Rust — lazy iterator chain (zero intermediate allocation)
```rust
// Rust iterator chain: lazy — tiada Vec perantara sampai .collect()
let result: Vec<String> = employees
    .iter()
    .filter(|e| e.age > 30)
    .map(|e| e.first_name.to_uppercase())
    .collect();

println!("{:?}", result);
```

> Iterator Rust adalah **lazy** — `filter` + `map` tidak buat Vec perantara. Lebih efisien daripada PHP chained functions.

---

## 7. Deep Copy & Nested Value Modification

### Deep Copy

#### PHP
```php
$copy = json_decode(json_encode($original), true); // quick deep copy

function deepCopy(array $array): array {
    $copy = [];
    foreach ($array as $key => $value) {
        $copy[$key] = is_array($value) ? deepCopy($value) : $value;
    }
    return $copy;
}
```

#### Rust
```rust
// Derive Clone → .clone() buat deep copy secara automatic
#[derive(Debug, Clone)]
struct Config {
    host:    String,
    timeout: u32,
}

let original = Config { host: "localhost".into(), timeout: 30 };
let copy = original.clone(); // deep copy — string di-clone juga

// Untuk Vec<Struct>:
let original_employees = employees.clone(); // deep copy semua employees
```

---

### setNestedValue() — Set Deeply Nested Key

#### PHP
```php
function setNestedValue(array &$array, array $keys, $value): void {
    $current = &$array;
    foreach ($keys as $key) {
        $current = &$current[$key];
    }
    $current = $value;
}
setNestedValue($config, ['database', 'connection', 'timeout'], 30);
```

#### Rust — guna nested HashMap dengan entry API
```rust
use std::collections::HashMap;

type NestedMap = HashMap<String, serde_json::Value>;

fn set_nested(map: &mut serde_json::Value, keys: &[&str], value: serde_json::Value) {
    if keys.is_empty() { return; }

    if keys.len() == 1 {
        if let Some(obj) = map.as_object_mut() {
            obj.insert(keys[0].to_string(), value);
        }
        return;
    }

    if let Some(obj) = map.as_object_mut() {
        let next = obj.entry(keys[0]).or_insert_with(|| serde_json::json!({}));
        set_nested(next, &keys[1..], value);
    }
}

let mut config = serde_json::json!({});
set_nested(&mut config, &["database", "connection", "timeout"], serde_json::json!(30));
println!("{}", config["database"]["connection"]["timeout"]); // 30
```

---

## Rujukan Pantas: PHP Array Functions → Rust

| PHP | Rust |
|---|---|
| `array_map($fn, $arr)` | `iter().map(fn).collect()` |
| `array_filter($arr, $fn)` | `iter().filter(fn).collect()` |
| `array_column($arr, 'field')` | `iter().map(\|e\| e.field).collect()` |
| `array_sum($arr)` | `iter().sum()` |
| `count($arr)` | `arr.len()` |
| `in_array($val, $arr)` | `arr.contains(&val)` |
| `array_push($arr, $v)` | `arr.push(v)` |
| `array_pop($arr)` | `arr.pop()` |
| `array_shift($arr)` | `arr.remove(0)` |
| `array_unshift($arr, $v)` | `arr.insert(0, v)` |
| `array_slice($arr, $off, $len)` | `&arr[off..off+len]` atau `.iter().skip().take()` |
| `array_merge($a, $b)` | `a.extend(b)` atau `[a, b].concat()` |
| `array_unique($arr)` | sort + dedup: `arr.sort(); arr.dedup()` |
| `array_reverse($arr)` | `arr.iter().rev().collect()` |
| `array_flip($arr)` | `iter().map(\|(k,v)\| (v,k)).collect()` |
| `usort($arr, $fn)` | `arr.sort_by(\|a,b\| fn(a,b))` |
| `uasort($arr, $fn)` | collect ke Vec<(k,v)>, sort, reconstruct HashMap |
| `array_walk_recursive($arr, $fn)` | nested `iter_mut()` loops |
| `array_merge_recursive($a, $b)` | recursive merge function atau `extend()` |
| `isset($arr[$k])` | `map.contains_key(&k)` |
| `$arr[$k] ?? $default` | `map.get(&k).copied().unwrap_or(default)` |
| PHP generator `yield` | `impl Iterator` atau `BufReader::lines()` |
| `json_encode` deep copy | `#[derive(Clone)]` + `.clone()` |

---

## Cargo.toml (dependencies yang digunakan)

```toml
[dependencies]
serde       = { version = "1", features = ["derive"] }
serde_json  = "1"
```

> `Vec`, `HashMap`, `Iterator` — semua dari standard library, tiada crate tambahan diperlukan.
