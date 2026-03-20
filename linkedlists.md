# 🔗 Linked Lists dalam Rust — Beginner to Advanced

> Panduan lengkap Linked Lists: dari konsep asas hingga implementasi sendiri.
> Gaya Head First — visual, hands-on, ada Brain Teaser!

---

## Apa Itu Linked List?

```
Array / Vec (contiguous memory):
┌────┬────┬────┬────┬────┐
│ 10 │ 20 │ 30 │ 40 │ 50 │
└────┴────┴────┴────┴────┘
  0    1    2    3    4     ← index, semua bersebelahan

Linked List (nodes berselerak dalam memory):
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ data: 10 │───▶│ data: 20 │───▶│ data: 30 │───▶│ data: 40 │───▶ None
│ next     │    │ next     │    │ next     │    │ next     │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
    head
```

Linked list adalah koleksi **node** di mana setiap node menyimpan data dan **pointer** ke node seterusnya. Berbeza dengan Vec yang menyimpan data bersebelahan dalam memory.

---

## Linked List vs Vec — Bila Guna Yang Mana?

```
┌────────────────────┬─────────────────────────────────────┐
│   Operasi          │   Vec      │   Linked List           │
├────────────────────┼────────────┼─────────────────────────┤
│ Akses by index     │ O(1) ✔✔   │ O(n) ✘                  │
│ Tambah di hujung   │ O(1) ✔    │ O(1) dengan tail ptr ✔  │
│ Tambah di depan    │ O(n) ✘    │ O(1) ✔✔                 │
│ Sisip di tengah    │ O(n) ✘    │ O(1) kalau ada ptr ✔    │
│ Cari elemen        │ O(n)      │ O(n)                    │
│ Cache friendly     │ ✔✔ sangat │ ✘ pointer melompat      │
│ Guna memory        │ Compact   │ Lebih (pointer overhead)│
└────────────────────┴────────────┴─────────────────────────┘
```

> 💡 **Realiti Rust:** Dalam kebanyakan kes, `Vec` atau `VecDeque` lebih laju dari Linked List
> walaupun secara teori. Ini kerana CPU cache sangat suka data bersebelahan.
> Linked List berguna untuk **pembelajaran** dan kes spesifik tertentu.

---

## Peta Pembelajaran

```
Bab 1  → std::collections::LinkedList (built-in)
Bab 2  → Buat Singly Linked List Sendiri
Bab 3  → Operations: push, pop, peek
Bab 4  → Iterate & Display
Bab 5  → Doubly Linked List
Bab 6  → VecDeque — Alternatif Praktikal
Bab 7  → Stack & Queue dengan Linked List
Bab 8  → Pattern Lanjutan
Bab 9  → Masalah Klasik Linked List
Bab 10 → Mini Project: Browser History
```

---

# BAB 1: std::collections::LinkedList (Built-in) 📦

## Import & Guna

```rust
use std::collections::LinkedList;

fn main() {
    let mut senarai: LinkedList<i32> = LinkedList::new();

    // Tambah di belakang
    senarai.push_back(10);
    senarai.push_back(20);
    senarai.push_back(30);

    // Tambah di depan
    senarai.push_front(5);
    senarai.push_front(1);

    println!("{:?}", senarai); // [1, 5, 10, 20, 30]

    // Buang dari depan
    println!("Pop front: {:?}", senarai.pop_front()); // Some(1)

    // Buang dari belakang
    println!("Pop back:  {:?}", senarai.pop_back());  // Some(30)

    println!("{:?}", senarai); // [5, 10, 20]

    // Tengok tanpa buang
    println!("Front: {:?}", senarai.front()); // Some(5)
    println!("Back:  {:?}", senarai.back());  // Some(20)

    // Saiz
    println!("Len: {}", senarai.len());       // 3
    println!("Kosong? {}", senarai.is_empty()); // false
}
```

---

## Lebih Banyak Operations

```rust
use std::collections::LinkedList;

fn main() {
    let mut a: LinkedList<i32> = [1, 2, 3].into_iter().collect();
    let mut b: LinkedList<i32> = [4, 5, 6].into_iter().collect();

    // append — alih semua dari b ke hujung a (b jadi kosong)
    a.append(&mut b);
    println!("a selepas append: {:?}", a); // [1, 2, 3, 4, 5, 6]
    println!("b selepas append: {:?}", b); // []

    // contains
    println!("Ada 3? {}", a.contains(&3)); // true
    println!("Ada 9? {}", a.contains(&9)); // false

    // iterate
    for x in &a {
        print!("{} ", x);
    }
    println!();

    // collect dari iterator
    let v: Vec<i32> = a.iter().copied().collect();
    println!("Vec: {:?}", v);

    // clear
    a.clear();
    println!("Selepas clear: {:?}", a); // []
}
```

---

## split_off — Pecah LinkedList

```rust
use std::collections::LinkedList;

fn main() {
    let mut senarai: LinkedList<i32> = (1..=6).collect();
    println!("Asal: {:?}", senarai); // [1, 2, 3, 4, 5, 6]

    // split_off(3) — pecah mulai index 3
    let ekor = senarai.split_off(3);
    println!("Kepala: {:?}", senarai); // [1, 2, 3]
    println!("Ekor:   {:?}", ekor);    // [4, 5, 6]
}
```

---

## 🧠 Brain Teaser #1

Apa output kod ini?

```rust
use std::collections::LinkedList;

fn main() {
    let mut ll: LinkedList<i32> = LinkedList::new();
    ll.push_back(3);
    ll.push_front(2);
    ll.push_front(1);
    ll.push_back(4);
    ll.pop_front();
    ll.push_front(0);
    println!("{:?}", ll);
}
```

<details>
<summary>👀 Jawapan</summary>

Output: `[0, 2, 3, 4]`

Langkah demi langkah:
1. `push_back(3)` → `[3]`
2. `push_front(2)` → `[2, 3]`
3. `push_front(1)` → `[1, 2, 3]`
4. `push_back(4)` → `[1, 2, 3, 4]`
5. `pop_front()` → buang `1` → `[2, 3, 4]`
6. `push_front(0)` → `[0, 2, 3, 4]`
</details>

---

# BAB 2: Buat Singly Linked List Sendiri 🔨

> Ini bahagian paling menarik dan paling **Rust** — bina struktur data dari scratch!
> Kita akan jumpa `Box<T>`, `Option<T>`, dan ownership pattern yang unik.

## Kenapa Susah Dalam Rust?

```
Dalam bahasa lain (C, Java):
  node.next = &nextNode;  // pointer mudah

Dalam Rust:
  - Siapa punya node? (ownership)
  - Boleh ada dua pointer ke node yang sama? (borrow rules)
  - Recursive struct perlu saiz tetap semasa compile (Box!)
```

---

## Definisi Node & List

```rust
// Tanpa Box — ini TIDAK COMPILE!
// enum List {
//     Cons(i32, List),  // ERROR: recursive type has infinite size
//     Nil,
// }

// Dengan Box — OK! Box<T> ada saiz tetap (pointer = 8 bytes)
#[derive(Debug)]
enum List {
    Cons(i32, Box<List>),  // node: data + pointer ke node seterusnya
    Nil,                   // penanda hujung senarai
}

use List::{Cons, Nil};

fn main() {
    // Bina senarai: 1 → 2 → 3 → Nil
    let senarai = Cons(1,
                    Box::new(Cons(2,
                        Box::new(Cons(3,
                            Box::new(Nil))))));

    println!("{:?}", senarai);
}
```

```
Visualisasi Box dalam memory:

Stack:          Heap:
┌────────┐     ┌───────────────┐
│ Cons   │────▶│ 1 │ ptr ──────┼──▶ ┌───────────────┐
│        │     └───────────────┘     │ 2 │ ptr ───────┼──▶ ┌───────────────┐
└────────┘                           └───────────────┘     │ 3 │ ptr ───────┼──▶ Nil
                                                           └───────────────┘
```

---

## Singly Linked List Yang Proper

```rust
#[derive(Debug)]
pub struct LinkedList<T> {
    head: Option<Box<Node<T>>>,
    len:  usize,
}

#[derive(Debug)]
struct Node<T> {
    data: T,
    next: Option<Box<Node<T>>>,
}

impl<T> Node<T> {
    fn baru(data: T) -> Box<Self> {
        Box::new(Node { data, next: None })
    }
}

impl<T> LinkedList<T> {
    // Buat senarai kosong
    pub fn baru() -> Self {
        LinkedList { head: None, len: 0 }
    }

    // Tambah di DEPAN — O(1)
    pub fn push_front(&mut self, data: T) {
        let mut nod_baru = Node::baru(data);
        // Ambil head lama, jadikan next untuk node baru
        nod_baru.next = self.head.take();
        self.head = Some(nod_baru);
        self.len += 1;
    }

    // Buang dari DEPAN — O(1)
    pub fn pop_front(&mut self) -> Option<T> {
        self.head.take().map(|nod| {
            self.head = nod.next;
            self.len -= 1;
            nod.data
        })
    }

    // Tengok depan tanpa buang — O(1)
    pub fn peek_front(&self) -> Option<&T> {
        self.head.as_ref().map(|nod| &nod.data)
    }

    // Saiz
    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }
}

fn main() {
    let mut ll: LinkedList<i32> = LinkedList::baru();

    ll.push_front(30);
    ll.push_front(20);
    ll.push_front(10);

    println!("Depan: {:?}", ll.peek_front()); // Some(10)
    println!("Len:   {}", ll.len());           // 3

    while let Some(val) = ll.pop_front() {
        println!("Pop: {}", val); // 10, 20, 30
    }

    println!("Kosong? {}", ll.is_empty()); // true
}
```

---

## Tambah push_back

```rust
impl<T> LinkedList<T> {
    // Tambah di BELAKANG — O(n) untuk singly linked list
    pub fn push_back(&mut self, data: T) {
        let nod_baru = Node::baru(data);

        match self.head {
            None => {
                // Senarai kosong — nod baru jadi head
                self.head = Some(nod_baru);
            }
            Some(ref mut nod) => {
                // Traverse ke hujung
                let mut semasa = nod;
                while let Some(ref mut seterusnya) = semasa.next {
                    semasa = seterusnya;
                }
                semasa.next = Some(nod_baru);
            }
        }
        self.len += 1;
    }
}
```

> 💡 **Nota:** `push_back` pada singly linked list adalah O(n) kerana perlu traverse ke hujung.
> Untuk O(1) push_back, simpan pointer `tail` juga — lebih kompleks tapi laju.

---

## 🧠 Brain Teaser #2

Kenapa struct rekursif ini tidak boleh compile tanpa `Box`?

```rust
// TIDAK COMPILE
enum Senarai {
    Nod(i32, Senarai),
    Habis,
}
```

<details>
<summary>👀 Jawapan</summary>

Rust perlu tahu **saiz setiap type semasa compile**. `Senarai` mengandungi `Senarai` di dalamnya — ini bermakna saiznya adalah `i32 + Senarai` = `i32 + i32 + Senarai` = ... tak terhingga!

`Box<T>` selesaikan masalah ini kerana ia adalah **pointer** — saiznya tetap (8 bytes pada 64-bit), tidak kira apa yang ada dalam heap.

```
Senarai (rekursif, saiz ∞) ← GAGAL
Box<Senarai> (pointer, saiz = 8 bytes) ← OK ✔
```
</details>

---

# BAB 3: Implement Display & Iterator 🖨️

## Display untuk LinkedList

```rust
use std::fmt;

impl<T: fmt::Display> fmt::Display for LinkedList<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "[")?;
        let mut semasa = &self.head;
        let mut pertama = true;
        while let Some(nod) = semasa {
            if !pertama { write!(f, " → ")?; }
            write!(f, "{}", nod.data)?;
            semasa = &nod.next;
            pertama = false;
        }
        write!(f, "]")
    }
}

fn main() {
    let mut ll: LinkedList<i32> = LinkedList::baru();
    ll.push_front(30);
    ll.push_front(20);
    ll.push_front(10);
    println!("{}", ll); // [10 → 20 → 30]
}
```

---

## Iterator untuk LinkedList

```rust
// Struct untuk menyimpan state iterasi
pub struct Iter<'a, T> {
    semasa: Option<&'a Box<Node<T>>>,
}

impl<T> LinkedList<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { semasa: self.head.as_ref() }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.semasa.map(|nod| {
            self.semasa = nod.next.as_ref();
            &nod.data
        })
    }
}

fn main() {
    let mut ll: LinkedList<i32> = LinkedList::baru();
    ll.push_front(30);
    ll.push_front(20);
    ll.push_front(10);

    // Guna iterator
    for val in ll.iter() {
        print!("{} ", val); // 10 20 30
    }
    println!();

    // Semua iterator methods tersedia!
    let jumlah: i32 = ll.iter().sum();
    println!("Jumlah: {}", jumlah); // 60

    let besar: Vec<&i32> = ll.iter().filter(|&&x| x > 15).collect();
    println!("Lebih dari 15: {:?}", besar); // [20, 30]
}
```

---

## 🧠 Brain Teaser #3

Apakah perbezaan antara `iter()`, `iter_mut()`, dan `into_iter()` dalam konteks Linked List?

<details>
<summary>👀 Jawapan</summary>

| Method | Borrow | Item | Selepas guna |
|--------|--------|------|-------------|
| `iter()` | Shared `&` | `&T` | List masih boleh guna |
| `iter_mut()` | Mutable `&mut` | `&mut T` | List masih ada, boleh ubah data |
| `into_iter()` | Owned (move) | `T` | List **habis**, tidak boleh guna lagi |

```rust
// iter() — baca sahaja
for val in ll.iter() { /* val: &i32 */ }

// iter_mut() — boleh ubah
for val in ll.iter_mut() { *val *= 2; } // gandakan semua

// into_iter() — ambil ownership
for val in ll { /* val: i32, ll sudah gone */ }
```
</details>

---

# BAB 4: Doubly Linked List 🔁

> Doubly linked list setiap node ada pointer ke **depan DAN belakang**.
> Ini membolehkan traverse dua arah dan O(1) insert/delete di mana-mana.

```
     ┌──────────────────────────────────────────┐
     │                                          │
     ▼                                          │
┌──────────┐ ◀──▶ ┌──────────┐ ◀──▶ ┌──────────┐
│ prev:None│      │ prev ────┼──    │ prev ────┼──
│ data: 10 │      │ data: 20 │  │   │ data: 30 │
│ next ────┼──▶   │ next ────┼──▶   │ next:None│
└──────────┘      └──────────┘      └──────────┘
   head                                 tail
```

## Doubly Linked List Menggunakan Rc<RefCell<>>

```rust
use std::cell::RefCell;
use std::rc::Rc;

type Link<T> = Option<Rc<RefCell<DNode<T>>>>;

#[derive(Debug)]
struct DNode<T> {
    data: T,
    prev: Link<T>,
    next: Link<T>,
}

impl<T> DNode<T> {
    fn baru(data: T) -> Rc<RefCell<Self>> {
        Rc::new(RefCell::new(DNode {
            data,
            prev: None,
            next: None,
        }))
    }
}

pub struct DoublyLinkedList<T> {
    head: Link<T>,
    tail: Link<T>,
    len:  usize,
}

impl<T: std::fmt::Debug> DoublyLinkedList<T> {
    pub fn baru() -> Self {
        DoublyLinkedList { head: None, tail: None, len: 0 }
    }

    // Tambah di BELAKANG — O(1) kerana ada tail pointer!
    pub fn push_back(&mut self, data: T) {
        let nod = DNode::baru(data);
        match self.tail.take() {
            None => {
                // Senarai kosong
                self.head = Some(Rc::clone(&nod));
                self.tail = Some(nod);
            }
            Some(ekor_lama) => {
                // Sambung nod baru ke ekor lama
                nod.borrow_mut().prev = Some(Rc::clone(&ekor_lama));
                ekor_lama.borrow_mut().next = Some(Rc::clone(&nod));
                self.tail = Some(nod);
            }
        }
        self.len += 1;
    }

    // Tambah di DEPAN — O(1)
    pub fn push_front(&mut self, data: T) {
        let nod = DNode::baru(data);
        match self.head.take() {
            None => {
                self.head = Some(Rc::clone(&nod));
                self.tail = Some(nod);
            }
            Some(kepala_lama) => {
                nod.borrow_mut().next = Some(Rc::clone(&kepala_lama));
                kepala_lama.borrow_mut().prev = Some(Rc::clone(&nod));
                self.head = Some(nod);
            }
        }
        self.len += 1;
    }

    // Buang dari BELAKANG — O(1)
    pub fn pop_back(&mut self) -> Option<T> {
        self.tail.take().map(|ekor| {
            self.len -= 1;
            match ekor.borrow_mut().prev.take() {
                None => { self.head = None; }
                Some(nod_baru_ekor) => {
                    nod_baru_ekor.borrow_mut().next = None;
                    self.tail = Some(nod_baru_ekor);
                }
            }
            Rc::try_unwrap(ekor).ok().unwrap().into_inner().data
        })
    }

    // Buang dari DEPAN — O(1)
    pub fn pop_front(&mut self) -> Option<T> {
        self.head.take().map(|kepala| {
            self.len -= 1;
            match kepala.borrow_mut().next.take() {
                None => { self.tail = None; }
                Some(nod_baru_kepala) => {
                    nod_baru_kepala.borrow_mut().prev = None;
                    self.head = Some(nod_baru_kepala);
                }
            }
            Rc::try_unwrap(kepala).ok().unwrap().into_inner().data
        })
    }

    pub fn len(&self) -> usize { self.len }
    pub fn is_empty(&self) -> bool { self.len == 0 }

    pub fn peek_front(&self) -> Option<T> where T: Clone {
        self.head.as_ref().map(|n| n.borrow().data.clone())
    }

    pub fn peek_back(&self) -> Option<T> where T: Clone {
        self.tail.as_ref().map(|n| n.borrow().data.clone())
    }
}

fn main() {
    let mut dll: DoublyLinkedList<i32> = DoublyLinkedList::baru();

    dll.push_back(10);
    dll.push_back(20);
    dll.push_back(30);
    dll.push_front(5);

    println!("Depan: {:?}", dll.peek_front()); // Some(5)
    println!("Belakang: {:?}", dll.peek_back()); // Some(30)
    println!("Len: {}", dll.len()); // 4

    println!("Pop depan: {:?}", dll.pop_front()); // Some(5)
    println!("Pop belakang: {:?}", dll.pop_back()); // Some(30)
    println!("Len: {}", dll.len()); // 2
}
```

### Kenapa Rc<RefCell<>>?

```
Box<T>       — satu owner, immutable borrow
Rc<T>        — banyak owner, immutable borrow (reference counting)
RefCell<T>   — satu owner, mutable borrow semasa runtime
Rc<RefCell<T>> — banyak owner, mutable borrow → PERFECT untuk doubly linked list!

Node depan perlu hold Rc ke node belakang.
Node belakang perlu hold Rc ke node depan.
Guna RefCell sebab perlu ubah prev/next selepas buat node.
```

---

# BAB 5: VecDeque — Alternatif Praktikal 🚀

> Dalam kebanyakan kes sebenar, **`VecDeque`** adalah pilihan terbaik
> berbanding membina Linked List sendiri.

```rust
use std::collections::VecDeque;

fn main() {
    let mut dq: VecDeque<i32> = VecDeque::new();

    // push_back / push_front — O(1) amortized untuk kedua-dua!
    dq.push_back(20);
    dq.push_back(30);
    dq.push_front(10);
    dq.push_front(5);

    println!("{:?}", dq); // [5, 10, 20, 30]

    // pop_back / pop_front — O(1) untuk kedua-dua!
    println!("Pop front: {:?}", dq.pop_front()); // Some(5)
    println!("Pop back:  {:?}", dq.pop_back());  // Some(30)

    // front / back — tengok tanpa buang
    println!("Front: {:?}", dq.front()); // Some(10)
    println!("Back:  {:?}", dq.back());  // Some(20)

    // Masih boleh akses by index seperti Vec!
    println!("Index 0: {:?}", dq.get(0)); // Some(10)

    // Rotate
    let mut dq2: VecDeque<i32> = (1..=5).collect();
    dq2.rotate_left(2);
    println!("Rotate left 2: {:?}", dq2); // [3, 4, 5, 1, 2]

    // Convert dari/ke Vec
    let v: Vec<i32> = dq.into_iter().collect();
    println!("Jadi Vec: {:?}", v);

    let dq3: VecDeque<i32> = vec![1, 2, 3].into();
    println!("Dari Vec: {:?}", dq3);
}
```

## Bila Guna Apa

```
VecDeque   → Queue, Deque, push/pop depan & belakang, perlu akses index
LinkedList → Sangat banyak insert/delete di tengah dengan iterator
Vec        → Array biasa, push/pop belakang, akses by index kerap
```

---

# BAB 6: Stack & Queue 📚

## Stack (LIFO) dengan LinkedList

```rust
// Stack — Last In, First Out
// Guna push_front dan pop_front sahaja
struct Stack<T> {
    data: std::collections::LinkedList<T>,
}

impl<T: std::fmt::Debug> Stack<T> {
    fn baru() -> Self {
        Stack { data: std::collections::LinkedList::new() }
    }

    fn push(&mut self, val: T) {
        self.data.push_front(val);
    }

    fn pop(&mut self) -> Option<T> {
        self.data.pop_front()
    }

    fn peek(&self) -> Option<&T> {
        self.data.front()
    }

    fn is_empty(&self) -> bool {
        self.data.is_empty()
    }

    fn len(&self) -> usize {
        self.data.len()
    }
}

fn main() {
    let mut stack: Stack<&str> = Stack::baru();

    // Undo history — macam Ctrl+Z
    stack.push("taip 'H'");
    stack.push("taip 'e'");
    stack.push("taip 'l'");
    stack.push("taip 'l'");
    stack.push("taip 'o'");

    println!("Atas stack: {:?}", stack.peek()); // "taip 'o'"

    // Undo tiga kali
    for _ in 0..3 {
        println!("Undo: {:?}", stack.pop());
    }
    // "taip 'o'", "taip 'l'", "taip 'l'"

    println!("Baki: {}", stack.len()); // 2
}
```

---

## Queue (FIFO) dengan LinkedList

```rust
// Queue — First In, First Out
// push_back untuk enqueue, pop_front untuk dequeue
struct Queue<T> {
    data: std::collections::LinkedList<T>,
}

impl<T: std::fmt::Debug + Clone> Queue<T> {
    fn baru() -> Self {
        Queue { data: std::collections::LinkedList::new() }
    }

    fn enqueue(&mut self, val: T) {
        self.data.push_back(val);
        println!("  Enqueue: {:?}", val);
    }

    fn dequeue(&mut self) -> Option<T> {
        self.data.pop_front()
    }

    fn tengok_depan(&self) -> Option<&T> {
        self.data.front()
    }

    fn is_empty(&self) -> bool {
        self.data.is_empty()
    }

    fn len(&self) -> usize {
        self.data.len()
    }
}

fn main() {
    let mut giliran: Queue<String> = Queue::baru();

    println!("=== Giliran Servis ===");
    giliran.enqueue("Pelanggan A".into());
    giliran.enqueue("Pelanggan B".into());
    giliran.enqueue("Pelanggan C".into());

    println!("\nServis bermula...");
    while !giliran.is_empty() {
        if let Some(pelanggan) = giliran.dequeue() {
            println!("  Sedang layan: {}", pelanggan);
        }
    }
    println!("Semua dilayan!");
}
```

---

## 🧠 Brain Teaser #4

Stack digunakan untuk semak sama ada kurungan dalam string adalah seimbang. Tulis fungsi `is_balanced("({[]})")` → `true`, `is_balanced("({[})")` → `false`.

<details>
<summary>👀 Jawapan</summary>

```rust
fn is_balanced(s: &str) -> bool {
    let mut stack: Vec<char> = Vec::new();

    for c in s.chars() {
        match c {
            '(' | '{' | '[' => stack.push(c),
            ')' => if stack.pop() != Some('(') { return false; },
            '}' => if stack.pop() != Some('{') { return false; },
            ']' => if stack.pop() != Some('[') { return false; },
            _   => {}
        }
    }
    stack.is_empty()
}

fn main() {
    println!("{}", is_balanced("({[]})"));  // true
    println!("{}", is_balanced("({[})"))); // false
    println!("{}", is_balanced("(){}[]")); // true
    println!("{}", is_balanced("((())")); // false
}
```
</details>

---

# BAB 7: Pattern Lanjutan 🎯

## Reverse Linked List

```rust
use std::collections::LinkedList;

fn reverse<T: Clone>(ll: &LinkedList<T>) -> LinkedList<T> {
    let mut hasil = LinkedList::new();
    for item in ll {
        hasil.push_front(item.clone());
    }
    hasil
}

fn main() {
    let ll: LinkedList<i32> = (1..=5).collect();
    println!("Asal:    {:?}", ll);           // [1, 2, 3, 4, 5]
    println!("Terbalik:{:?}", reverse(&ll)); // [5, 4, 3, 2, 1]
}
```

---

## Merge Dua Sorted Lists

```rust
use std::collections::LinkedList;

fn merge_sorted(
    mut a: LinkedList<i32>,
    mut b: LinkedList<i32>,
) -> LinkedList<i32> {
    let mut hasil = LinkedList::new();

    loop {
        match (a.front(), b.front()) {
            (Some(&x), Some(&y)) => {
                if x <= y {
                    hasil.push_back(a.pop_front().unwrap());
                } else {
                    hasil.push_back(b.pop_front().unwrap());
                }
            }
            (Some(_), None) => hasil.append(&mut a),
            (None, Some(_)) => hasil.append(&mut b),
            (None, None)    => break,
        }
    }
    hasil
}

fn main() {
    let a: LinkedList<i32> = [1, 3, 5, 7].into_iter().collect();
    let b: LinkedList<i32> = [2, 4, 6, 8].into_iter().collect();

    let gabung = merge_sorted(a, b);
    println!("Merged: {:?}", gabung); // [1, 2, 3, 4, 5, 6, 7, 8]
}
```

---

## Detect Cycle (Floyd's Algorithm)

> Dalam implementasi linked list dengan raw pointer atau Rc, cycle boleh berlaku.
> Floyd's "tortoise and hare" algorithm mengesan cycle dalam O(n) masa, O(1) space.

```rust
// Demonstrasi konsep dengan Vec (simulate linked list via index)
fn ada_cycle(next: &[Option<usize>], start: usize) -> bool {
    let mut perlahan = Some(start); // kura-kura: maju 1 langkah
    let mut laju     = Some(start); // arnab:     maju 2 langkah

    loop {
        // Kura-kura maju 1
        perlahan = perlahan.and_then(|i| next[i]);

        // Arnab maju 2
        laju = laju
            .and_then(|i| next[i])
            .and_then(|i| next[i]);

        match (perlahan, laju) {
            (None, _) | (_, None) => return false, // sampai hujung, tiada cycle
            (Some(p), Some(l)) if p == l => return true, // bertemu! ada cycle
            _ => continue,
        }
    }
}

fn main() {
    // Senarai tanpa cycle: 0→1→2→3→None
    let tiada_cycle = vec![Some(1), Some(2), Some(3), None];
    println!("Tiada cycle: {}", ada_cycle(&tiada_cycle, 0)); // false

    // Senarai dengan cycle: 0→1→2→3→1 (kembali ke index 1)
    let ada_cycle_v = vec![Some(1), Some(2), Some(3), Some(1)];
    println!("Ada cycle:   {}", ada_cycle(&ada_cycle_v, 0)); // true
}
```

---

## LRU Cache Menggunakan LinkedList + HashMap

```rust
use std::collections::{HashMap, LinkedList};

// LRU Cache — Least Recently Used
// Elemen yang paling lama tidak digunakan dibuang bila penuh
struct LruCache {
    kapasiti: usize,
    map:      HashMap<String, i32>,
    urutan:   LinkedList<String>, // depan = paling baru, belakang = paling lama
}

impl LruCache {
    fn baru(kapasiti: usize) -> Self {
        LruCache {
            kapasiti,
            map:    HashMap::new(),
            urutan: LinkedList::new(),
        }
    }

    fn get(&mut self, kunci: &str) -> Option<i32> {
        if self.map.contains_key(kunci) {
            // Alih ke depan (paling baru)
            self.urutan.retain(|k| k != kunci);
            self.urutan.push_front(kunci.to_string());
            self.map.get(kunci).copied()
        } else {
            None
        }
    }

    fn put(&mut self, kunci: &str, nilai: i32) {
        if self.map.contains_key(kunci) {
            // Kemaskini — alih ke depan
            self.urutan.retain(|k| k != kunci);
        } else if self.map.len() >= self.kapasiti {
            // Penuh — buang yang paling lama (belakang)
            if let Some(lama) = self.urutan.pop_back() {
                self.map.remove(&lama);
                println!("  Evict: '{}'", lama);
            }
        }
        self.urutan.push_front(kunci.to_string());
        self.map.insert(kunci.to_string(), nilai);
    }

    fn papar(&self) {
        print!("Cache [depan→belakang]: ");
        for k in &self.urutan {
            print!("{}={} ", k, self.map[k]);
        }
        println!();
    }
}

fn main() {
    let mut cache = LruCache::baru(3);

    cache.put("a", 1); cache.papar();
    cache.put("b", 2); cache.papar();
    cache.put("c", 3); cache.papar();

    println!("get 'a': {:?}", cache.get("a")); // Some(1), 'a' jadi paling baru
    cache.papar();

    cache.put("d", 4); // penuh! 'b' dibuang (paling lama)
    cache.papar();

    println!("get 'b': {:?}", cache.get("b")); // None — dah dibuang
}
```

---

## 🧠 Brain Teaser #5 (Advanced)

Tulis fungsi `tengah_node` yang return nilai elemen di **tengah** linked list dalam **satu lintasan** (satu pass, tanpa kira panjang dulu).

Hint: guna dua pointer — satu laju (maju 2), satu perlahan (maju 1).

<details>
<summary>👀 Jawapan</summary>

```rust
fn tengah_nilai(ll: &std::collections::LinkedList<i32>) -> Option<i32> {
    let v: Vec<i32> = ll.iter().copied().collect();
    v.get(v.len() / 2).copied()
}

// Cara dengan dua pointer (simulate dengan Vec untuk demo):
fn tengah_dua_pointer(data: &[i32]) -> Option<&i32> {
    if data.is_empty() { return None; }
    let mut perlahan = 0;
    let mut laju     = 0;

    while laju + 1 < data.len() {
        laju     += 2.min(data.len() - 1 - laju);
        perlahan += 1;
        if laju >= data.len() - 1 { break; }
    }
    data.get(perlahan)
}

fn main() {
    let data = vec![1, 2, 3, 4, 5];
    println!("Tengah: {:?}", tengah_dua_pointer(&data)); // Some(3)

    let data2 = vec![1, 2, 3, 4, 5, 6];
    println!("Tengah: {:?}", tengah_dua_pointer(&data2)); // Some(4)
}
```

Konsep: arnab maju 2 langkah, kura-kura 1 langkah. Bila arnab sampai hujung, kura-kura ada di tengah!
</details>

---

# BAB 8: Mini Project — Browser History 🌐

```rust
use std::collections::LinkedList;

#[derive(Debug, Clone)]
struct HalamanWeb {
    url:   String,
    tajuk: String,
}

impl HalamanWeb {
    fn baru(url: &str, tajuk: &str) -> Self {
        HalamanWeb { url: url.into(), tajuk: tajuk.into() }
    }
}

struct BrowserHistory {
    sejarah:  LinkedList<HalamanWeb>, // semua halaman dilawati
    semasa:   usize,                  // index halaman semasa
    had_sejarah: usize,               // had maksimum sejarah
}

impl BrowserHistory {
    fn baru(had: usize) -> Self {
        BrowserHistory {
            sejarah: LinkedList::new(),
            semasa:  0,
            had_sejarah: had,
        }
    }

    fn lawat(&mut self, url: &str, tajuk: &str) {
        let halaman = HalamanWeb::baru(url, tajuk);

        // Buang sejarah "ke depan" bila pergi ke halaman baru
        // (macam klik pautan baru selepas tekan Back)
        let kekal = self.semasa;
        let mut baharu: LinkedList<HalamanWeb> = LinkedList::new();
        for (i, h) in self.sejarah.iter().enumerate() {
            if i < kekal { baharu.push_back(h.clone()); }
        }
        self.sejarah = baharu;

        // Had sejarah
        if self.sejarah.len() >= self.had_sejarah {
            self.sejarah.pop_front();
        } else {
            self.semasa += 1;
        }

        self.sejarah.push_back(halaman.clone());
        println!("🌐 Melawat: {} ({})", tajuk, url);
    }

    fn balik(&mut self) -> Option<HalamanWeb> {
        if self.semasa > 1 {
            self.semasa -= 1;
            self.halaman_semasa()
        } else {
            println!("⚠ Tiada sejarah untuk balik");
            None
        }
    }

    fn hadapan(&mut self) -> Option<HalamanWeb> {
        if self.semasa < self.sejarah.len() {
            self.semasa += 1;
            self.halaman_semasa()
        } else {
            println!("⚠ Tiada halaman hadapan");
            None
        }
    }

    fn halaman_semasa(&self) -> Option<HalamanWeb> {
        self.sejarah.iter().nth(self.semasa - 1).cloned()
    }

    fn papar_sejarah(&self) {
        println!("\n=== Sejarah Pelayaran ===");
        for (i, h) in self.sejarah.iter().enumerate() {
            let penanda = if i + 1 == self.semasa { "▶" } else { " " };
            println!("  {} [{}] {} — {}", penanda, i + 1, h.tajuk, h.url);
        }
        println!("========================");
    }
}

fn main() {
    let mut browser = BrowserHistory::baru(10);

    browser.lawat("https://google.com",   "Google");
    browser.lawat("https://rust-lang.org", "Rust Lang");
    browser.lawat("https://docs.rs",       "Docs.rs");
    browser.lawat("https://crates.io",     "Crates.io");

    browser.papar_sejarah();

    println!("\n⬅ Tekan Back dua kali:");
    if let Some(h) = browser.balik() {
        println!("  Sekarang di: {}", h.tajuk);
    }
    if let Some(h) = browser.balik() {
        println!("  Sekarang di: {}", h.tajuk);
    }

    browser.papar_sejarah();

    println!("\n➡ Tekan Forward sekali:");
    if let Some(h) = browser.hadapan() {
        println!("  Sekarang di: {}", h.tajuk);
    }

    println!("\n🌐 Lawat halaman baru (memotong forward history):");
    browser.lawat("https://github.com", "GitHub");
    browser.hadapan(); // tiada hadapan lagi

    browser.papar_sejarah();
}
```

---

# 📋 Rujukan Pantas — Linked List Cheat Sheet

## std::collections::LinkedList Methods

| Method | Kegunaan | Big-O |
|--------|----------|-------|
| `push_front(v)` | Tambah di depan | O(1) |
| `push_back(v)` | Tambah di belakang | O(1) |
| `pop_front()` | Buang dari depan | O(1) |
| `pop_back()` | Buang dari belakang | O(1) |
| `front()` | Tengok depan | O(1) |
| `back()` | Tengok belakang | O(1) |
| `len()` | Bilangan elemen | O(1) |
| `is_empty()` | Semak kosong | O(1) |
| `contains(&v)` | Semak kewujudan | O(n) |
| `append(&mut b)` | Gabung dua list | O(1) |
| `split_off(i)` | Pecah di index i | O(n) |
| `clear()` | Kosongkan semua | O(n) |

## Pilih Koleksi Yang Tepat

```
┌────────────────────────────────────────────────────┐
│  Nak push/pop belakang sahaja?     → Vec            │
│  Nak push/pop depan & belakang?    → VecDeque       │
│  Nak akses by index?               → Vec / VecDeque │
│  Banyak insert/delete di tengah?   → LinkedList     │
│  Nak Stack (LIFO)?                 → Vec (push/pop) │
│  Nak Queue (FIFO)?                 → VecDeque       │
│  Perlu urutan & no duplikat?       → BTreeSet       │
└────────────────────────────────────────────────────┘
```

## Type Alias Berguna untuk Linked List Kompleks

```rust
// Singly linked (owned)
type Link<T> = Option<Box<Node<T>>>;

// Doubly linked (shared, mutable)
type DLink<T> = Option<Rc<RefCell<DNode<T>>>>;

// Thread-safe doubly linked
type ALink<T> = Option<Arc<Mutex<DNode<T>>>>;
```

## Box vs Rc vs Arc

```
Box<T>              — satu owner, heap allocate, O(1) deref
Rc<T>               — banyak owner, single-threaded
Arc<T>              — banyak owner, multi-threaded (atomic)
RefCell<T>          — interior mutability, runtime borrow check
Rc<RefCell<T>>      — shared + mutable, single-thread
Arc<Mutex<T>>       — shared + mutable, multi-thread
```

---

## 🏆 Cabaran Akhir

Cuba bina salah satu:

1. **Playlist Muzik** — doubly linked list, skip depan/belakang, shuffle
2. **Undo/Redo System** — dua stack (undo stack + redo stack)
3. **Josephus Problem** — n orang dalam bulatan, setiap k orang keluar
4. **Polynomial Calculator** — linked list untuk wakil polynomial, tambah & darab
5. **Memory Allocator (Mini)** — free list menggunakan linked list

---

*Linked Lists in Rust — dari `push_front()` hingga `Rc<RefCell<>>` doubly linked. Selamat belajar!* 🦀
