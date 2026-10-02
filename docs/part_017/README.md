# Part 017: Smart Pointers

## บทนำ

Smart Pointers ใน Rust คือ struct ที่ทำหน้าที่เหมือน pointer แต่มีความสามารถเพิ่มเติม เช่น การจัดการ memory อัตโนมัติ, การ track จำนวน references หรือการอนุญาตให้แก้ไข data ผ่าน reference ที่ immutable ในบทนี้เราจะเรียนรู้ smart pointers หลักทั้งหมดใน Rust

---

## 1. Box\<T\> - Heap Allocation

`Box<T>` เป็น smart pointer ที่เก็บค่าบน heap แทนที่จะเก็บบน stack

### การใช้งานพื้นฐาน

```rust
fn main() {
    // เก็บค่าบน heap
    let b = Box::new(5);
    println!("b = {}", b);
    
    // dereference
    let x = *b + 1;
    println!("x = {}", x);
    
    // Box กับ large data
    let large_data = Box::new([0u8; 1000]);
    println!("Large data on heap, length: {}", large_data.len());
    
    // Box จะถูก drop โดยอัตโนมัติเมื่อออกจาก scope
}
```

### Recursive Types ด้วย Box

```rust
// ปัญหา: recursive type มีขนาดไม่แน่นอนตอน compile time
// enum List { Cons(i32, List), Nil }  // ERROR!

// แก้ด้วย Box
#[derive(Debug)]
enum List {
    Cons(i32, Box<List>),
    Nil,
}

impl List {
    fn new() -> Self {
        List::Nil
    }
    
    fn push(self, value: i32) -> Self {
        List::Cons(value, Box::new(self))
    }
    
    fn len(&self) -> usize {
        match self {
            List::Nil => 0,
            List::Cons(_, tail) => 1 + tail.len(),
        }
    }
    
    fn sum(&self) -> i32 {
        match self {
            List::Nil => 0,
            List::Cons(val, tail) => val + tail.sum(),
        }
    }
}

fn main() {
    let list = List::new()
        .push(1)
        .push(2)
        .push(3);
    
    println!("Length: {}", list.len());
    println!("Sum: {}", list.sum());
}
```

### Box กับ Trait Objects

```rust
trait Animal {
    fn name(&self) -> &str;
    fn sound(&self) -> &str;
    fn describe(&self) -> String {
        format!("{} says {}", self.name(), self.sound())
    }
}

struct Dog;
struct Cat;
struct Bird;

impl Animal for Dog {
    fn name(&self) -> &str { "Dog" }
    fn sound(&self) -> &str { "Woof" }
}

impl Animal for Cat {
    fn name(&self) -> &str { "Cat" }
    fn sound(&self) -> &str { "Meow" }
}

impl Animal for Bird {
    fn name(&self) -> &str { "Bird" }
    fn sound(&self) -> &str { "Tweet" }
}

fn make_animal_sounds(animals: &[Box<dyn Animal>]) {
    for animal in animals {
        println!("{}", animal.describe());
    }
}

fn main() {
    let animals: Vec<Box<dyn Animal>> = vec![
        Box::new(Dog),
        Box::new(Cat),
        Box::new(Bird),
    ];
    
    make_animal_sounds(&animals);
}
```

---

## 2. Rc\<T\> - Reference Counting

`Rc<T>` อนุญาตให้มีเจ้าของหลายคน (multiple ownership) ในสถานการณ์ single-threaded

```rust
use std::rc::Rc;

fn main() {
    // สร้าง Rc
    let a = Rc::new(String::from("shared value"));
    println!("Reference count: {}", Rc::strong_count(&a)); // 1
    
    // clone เพิ่ม reference count (ไม่ copy data)
    let b = Rc::clone(&a);
    println!("Reference count: {}", Rc::strong_count(&a)); // 2
    
    {
        let c = Rc::clone(&a);
        println!("Reference count: {}", Rc::strong_count(&a)); // 3
        println!("c = {}", c);
    } // c ถูก drop ที่นี่
    
    println!("Reference count after c dropped: {}", Rc::strong_count(&a)); // 2
    println!("a = {}", a);
    println!("b = {}", b);
}
```

### Rc กับ Shared Graph

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    value: i32,
    children: Vec<Rc<Node>>,
}

impl Node {
    fn new(value: i32) -> Rc<Self> {
        Rc::new(Node {
            value,
            children: vec![],
        })
    }
}

fn main() {
    // Shared node (leaf)
    let leaf = Rc::new(Node {
        value: 10,
        children: vec![],
    });
    
    println!("Leaf count: {}", Rc::strong_count(&leaf)); // 1
    
    let branch1 = Rc::new(Node {
        value: 5,
        children: vec![Rc::clone(&leaf)],
    });
    
    let branch2 = Rc::new(Node {
        value: 7,
        children: vec![Rc::clone(&leaf)],
    });
    
    println!("Leaf count after sharing: {}", Rc::strong_count(&leaf)); // 3
    
    println!("Branch1 value: {}", branch1.value);
    println!("Branch2 value: {}", branch2.value);
    println!("Shared leaf: {}", leaf.value);
}
```

---

## 3. Arc\<T\> - Atomic Reference Counting

`Arc<T>` เหมือน `Rc<T>` แต่ thread-safe ใช้ atomic operations ใน multi-threaded context

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    // Arc สำหรับ shared data ระหว่าง threads
    let shared_data = Arc::new(vec![1, 2, 3, 4, 5]);
    let mut handles = vec![];
    
    for i in 0..3 {
        let data = Arc::clone(&shared_data);
        let handle = thread::spawn(move || {
            println!("Thread {}: sum = {}", i, data.iter().sum::<i32>());
        });
        handles.push(handle);
    }
    
    for handle in handles {
        handle.join().unwrap();
    }
    
    println!("Main: {:?}", shared_data);
    println!("Reference count: {}", Arc::strong_count(&shared_data));
}
```

### Arc กับ Mutex สำหรับ Mutable Data

```rust
use std::sync::{Arc, Mutex};
use std::thread;

struct SharedCounter {
    count: Mutex<u32>,
    name: String,
}

impl SharedCounter {
    fn new(name: &str) -> Arc<Self> {
        Arc::new(SharedCounter {
            count: Mutex::new(0),
            name: name.to_string(),
        })
    }
    
    fn increment(&self) {
        let mut count = self.count.lock().unwrap();
        *count += 1;
    }
    
    fn get(&self) -> u32 {
        *self.count.lock().unwrap()
    }
}

fn main() {
    let counter = SharedCounter::new("my_counter");
    let mut handles = vec![];
    
    for _ in 0..10 {
        let c = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            for _ in 0..100 {
                c.increment();
            }
        });
        handles.push(handle);
    }
    
    for handle in handles {
        handle.join().unwrap();
    }
    
    println!("Counter {}: {}", counter.name, counter.get()); // 1000
}
```

---

## 4. RefCell\<T\> - Interior Mutability

`RefCell<T>` อนุญาตให้แก้ไขค่าผ่าน immutable reference (borrow checking ตอน runtime แทน compile time)

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(vec![1, 2, 3]);
    
    // immutable borrow
    {
        let borrowed = data.borrow();
        println!("Data: {:?}", *borrowed);
        // ไม่สามารถ borrow_mut ในขณะที่มี borrow อยู่
    }
    
    // mutable borrow
    {
        let mut borrowed_mut = data.borrow_mut();
        borrowed_mut.push(4);
        println!("After push: {:?}", *borrowed_mut);
    }
    
    println!("Final: {:?}", data.borrow());
}
```

### RefCell: Mock Object Pattern

```rust
use std::cell::RefCell;

trait Messenger {
    fn send(&self, msg: &str);
}

struct LimitTracker<'a, T: Messenger> {
    messenger: &'a T,
    value: usize,
    max: usize,
}

impl<'a, T: Messenger> LimitTracker<'a, T> {
    fn new(messenger: &'a T, max: usize) -> LimitTracker<'a, T> {
        LimitTracker {
            messenger,
            value: 0,
            max,
        }
    }
    
    fn set_value(&mut self, value: usize) {
        self.value = value;
        let percentage = self.value as f64 / self.max as f64;
        
        if percentage >= 1.0 {
            self.messenger.send("Error: over quota!");
        } else if percentage >= 0.9 {
            self.messenger.send("Warning: 90% of quota used");
        } else if percentage >= 0.75 {
            self.messenger.send("Notice: 75% of quota used");
        }
    }
}

// Mock ที่ต้องเก็บ sent_messages แม้ self เป็น &self
struct MockMessenger {
    sent_messages: RefCell<Vec<String>>,
}

impl MockMessenger {
    fn new() -> MockMessenger {
        MockMessenger {
            sent_messages: RefCell::new(vec![]),
        }
    }
}

impl Messenger for MockMessenger {
    fn send(&self, msg: &str) {
        // ใช้ borrow_mut() แม้ self เป็น &self
        self.sent_messages.borrow_mut().push(msg.to_string());
    }
}

fn main() {
    let mock = MockMessenger::new();
    let mut tracker = LimitTracker::new(&mock, 100);
    
    tracker.set_value(80);
    tracker.set_value(95);
    tracker.set_value(100);
    
    let messages = mock.sent_messages.borrow();
    println!("Sent {} messages:", messages.len());
    for msg in messages.iter() {
        println!("  - {}", msg);
    }
}
```

---

## 5. Rc\<RefCell\<T\>\> Pattern

เป็น pattern ที่ใช้บ่อยสำหรับ shared mutable data ใน single-threaded context

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    value: i32,
    next: Option<Rc<RefCell<Node>>>,
}

impl Node {
    fn new(value: i32) -> Rc<RefCell<Self>> {
        Rc::new(RefCell::new(Node {
            value,
            next: None,
        }))
    }
}

fn main() {
    let node1 = Node::new(1);
    let node2 = Node::new(2);
    let node3 = Node::new(3);
    
    // เชื่อม nodes
    node1.borrow_mut().next = Some(Rc::clone(&node2));
    node2.borrow_mut().next = Some(Rc::clone(&node3));
    
    // แก้ไข node2 ผ่านทั้ง node1 และ direct reference
    {
        let mut n2 = node2.borrow_mut();
        n2.value = 20;
    }
    
    // traverse list
    let mut current = Some(Rc::clone(&node1));
    while let Some(node) = current {
        let n = node.borrow();
        println!("Value: {}", n.value);
        current = n.next.as_ref().map(|next| Rc::clone(next));
    }
}
```

### Spreadsheet-like data

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug, Clone)]
enum CellValue {
    Number(f64),
    Text(String),
    Formula(String), // simplified
}

struct Cell {
    value: CellValue,
    dependents: Vec<Rc<RefCell<Cell>>>,
}

impl Cell {
    fn new(value: CellValue) -> Rc<RefCell<Self>> {
        Rc::new(RefCell::new(Cell {
            value,
            dependents: vec![],
        }))
    }
    
    fn get_value(&self) -> &CellValue {
        &self.value
    }
    
    fn set_value(&mut self, value: CellValue) {
        self.value = value;
        // Notify dependents (simplified)
        println!("Cell updated, notifying {} dependents", self.dependents.len());
    }
    
    fn add_dependent(&mut self, cell: Rc<RefCell<Cell>>) {
        self.dependents.push(cell);
    }
}

fn main() {
    let a1 = Cell::new(CellValue::Number(10.0));
    let b1 = Cell::new(CellValue::Number(20.0));
    
    // b1 depends on a1
    a1.borrow_mut().add_dependent(Rc::clone(&b1));
    
    println!("a1: {:?}", a1.borrow().get_value());
    println!("b1: {:?}", b1.borrow().get_value());
    
    // Update a1 (will notify b1)
    a1.borrow_mut().set_value(CellValue::Number(15.0));
    println!("a1 updated: {:?}", a1.borrow().get_value());
}
```

---

## 6. Cow\<'a, T\> - Clone on Write

`Cow` (Clone on Write) ช่วยหลีกเลี่ยงการ clone ที่ไม่จำเป็น

```rust
use std::borrow::Cow;

// function ที่อาจต้อง modify string หรือไม่ก็ได้
fn ensure_uppercase<'a>(s: &'a str) -> Cow<'a, str> {
    if s.chars().all(|c| c.is_uppercase() || !c.is_alphabetic()) {
        // ไม่ต้อง modify, return borrowed
        Cow::Borrowed(s)
    } else {
        // ต้อง modify, return owned
        Cow::Owned(s.to_uppercase())
    }
}

fn process_string(input: &str) -> Cow<str> {
    if input.contains(' ') {
        // จำเป็นต้อง allocate new string
        Cow::Owned(input.replace(' ', "_"))
    } else {
        // ไม่ต้อง allocate
        Cow::Borrowed(input)
    }
}

fn main() {
    let already_upper = "HELLO";
    let mixed = "Hello World";
    
    let r1 = ensure_uppercase(already_upper);
    let r2 = ensure_uppercase(mixed);
    
    // r1 เป็น Borrowed (ไม่มี allocation)
    // r2 เป็น Owned (มี allocation)
    println!("r1: {} (owned: {})", r1, matches!(r1, Cow::Owned(_)));
    println!("r2: {} (owned: {})", r2, matches!(r2, Cow::Owned(_)));
    
    // Cow กับ Vec
    fn ensure_sorted(v: &[i32]) -> Cow<[i32]> {
        if v.windows(2).all(|w| w[0] <= w[1]) {
            Cow::Borrowed(v)
        } else {
            let mut sorted = v.to_vec();
            sorted.sort();
            Cow::Owned(sorted)
        }
    }
    
    let sorted = vec![1, 2, 3, 4, 5];
    let unsorted = vec![5, 3, 1, 4, 2];
    
    let r3 = ensure_sorted(&sorted);
    let r4 = ensure_sorted(&unsorted);
    
    println!("Sorted (borrowed: {}): {:?}", matches!(r3, Cow::Borrowed(_)), r3);
    println!("Unsorted result (owned: {}): {:?}", matches!(r4, Cow::Owned(_)), r4);
}
```

---

## 7. Pin\<T\> สำหรับ Self-Referential Types

`Pin<T>` ป้องกันการย้าย (move) value ในหน่วยความจำ ใช้สำหรับ self-referential structs

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

// Self-referential struct
struct SelfRef {
    data: String,
    self_ptr: *const String, // pointer ไปยัง data ของตัวเอง
    _pin: PhantomPinned,     // บอก compiler ว่า struct นี้ไม่ควรถูก move
}

impl SelfRef {
    fn new(data: String) -> Pin<Box<Self>> {
        let mut boxed = Box::pin(SelfRef {
            data,
            self_ptr: std::ptr::null(),
            _pin: PhantomPinned,
        });
        
        // ตั้งค่า self pointer หลัง pin
        let self_ptr = &boxed.data as *const String;
        unsafe {
            let mut_ref = Pin::as_mut(&mut boxed);
            Pin::get_unchecked_mut(mut_ref).self_ptr = self_ptr;
        }
        
        boxed
    }
    
    fn get_data(&self) -> &str {
        &self.data
    }
    
    fn get_via_ptr(&self) -> &str {
        unsafe { &*self.self_ptr }
    }
}

fn main() {
    let pinned = SelfRef::new("Hello, Pin!".to_string());
    
    println!("Data: {}", pinned.get_data());
    println!("Via ptr: {}", pinned.get_via_ptr());
    
    // ทั้งสองควรเป็น value เดียวกัน
    assert_eq!(pinned.get_data(), pinned.get_via_ptr());
}
```

### Pin ใน async context

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

// Manual Future implementation ที่ต้องใช้ Pin
struct CounterFuture {
    count: u32,
    target: u32,
}

impl Future for CounterFuture {
    type Output = u32;
    
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        self.count += 1;
        if self.count >= self.target {
            Poll::Ready(self.count)
        } else {
            cx.waker().wake_by_ref(); // request re-poll
            Poll::Pending
        }
    }
}

// ใช้งาน
#[tokio::main]
async fn main() {
    let future = CounterFuture { count: 0, target: 5 };
    let result = future.await;
    println!("Counter reached: {}", result);
}
```

---

## 8. Deref และ DerefMut Traits

```rust
use std::ops::{Deref, DerefMut};

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T;
    
    fn deref(&self) -> &T {
        &self.0
    }
}

impl<T> DerefMut for MyBox<T> {
    fn deref_mut(&mut self) -> &mut T {
        &mut self.0
    }
}

fn hello(name: &str) {
    println!("Hello, {}!", name);
}

fn main() {
    let x = 5;
    let y = MyBox::new(x);
    
    assert_eq!(5, x);
    assert_eq!(5, *y); // *y calls deref() then dereferences
    
    let s = MyBox::new(String::from("Rust"));
    hello(&s); // deref coercion: MyBox<String> -> String -> str
    
    // DerefMut
    let mut v = MyBox::new(vec![1, 2, 3]);
    v.push(4); // calls deref_mut() to get &mut Vec
    println!("Vec: {:?}", *v);
}
```

### Deref Coercion Chain

```rust
fn print_str(s: &str) {
    println!("{}", s);
}

fn main() {
    // Deref coercion chain:
    // &Box<String> -> &String -> &str
    let s = Box::new(String::from("hello"));
    print_str(&s);  // works! Box<String> -> String -> str
    
    // &String -> &str
    let owned = String::from("world");
    print_str(&owned); // works!
    
    // Multiple levels
    let nested = Box::new(Box::new(String::from("nested")));
    print_str(&nested); // Box<Box<String>> -> Box<String> -> String -> str
}
```

---

## 9. Drop Trait

```rust
struct Resource {
    name: String,
    id: u32,
}

impl Resource {
    fn new(name: &str, id: u32) -> Self {
        println!("Creating resource '{}' ({})", name, id);
        Resource {
            name: name.to_string(),
            id,
        }
    }
}

impl Drop for Resource {
    fn drop(&mut self) {
        println!("Dropping resource '{}' ({})", self.name, self.id);
        // cleanup code here
    }
}

fn main() {
    println!("Start");
    
    let r1 = Resource::new("database", 1);
    
    {
        let r2 = Resource::new("file", 2);
        let r3 = Resource::new("network", 3);
        println!("Inside block");
        // r3 dropped first (LIFO order), then r2
    }
    
    println!("After block");
    
    // Early drop
    let r4 = Resource::new("early_drop", 4);
    drop(r4); // explicit drop
    println!("After explicit drop");
    
    println!("End");
    // r1 dropped here
}
```

---

## 10. Practical: Linked List ด้วย Box

```rust
#[derive(Debug)]
pub struct LinkedList<T> {
    head: Option<Box<Node<T>>>,
    size: usize,
}

#[derive(Debug)]
struct Node<T> {
    value: T,
    next: Option<Box<Node<T>>>,
}

impl<T: std::fmt::Debug + Clone> LinkedList<T> {
    pub fn new() -> Self {
        LinkedList { head: None, size: 0 }
    }
    
    pub fn push_front(&mut self, value: T) {
        let new_node = Box::new(Node {
            value,
            next: self.head.take(),
        });
        self.head = Some(new_node);
        self.size += 1;
    }
    
    pub fn pop_front(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            self.head = node.next;
            self.size -= 1;
            node.value
        })
    }
    
    pub fn peek(&self) -> Option<&T> {
        self.head.as_ref().map(|node| &node.value)
    }
    
    pub fn len(&self) -> usize {
        self.size
    }
    
    pub fn is_empty(&self) -> bool {
        self.size == 0
    }
    
    pub fn iter(&self) -> ListIter<T> {
        ListIter {
            current: self.head.as_deref(),
        }
    }
    
    pub fn contains(&self, value: &T) -> bool
    where
        T: PartialEq,
    {
        self.iter().any(|v| v == value)
    }
    
    pub fn reverse(&mut self) {
        let mut prev = None;
        let mut current = self.head.take();
        
        while let Some(mut node) = current {
            let next = node.next.take();
            node.next = prev;
            prev = Some(node);
            current = next;
        }
        
        self.head = prev;
    }
    
    pub fn to_vec(&self) -> Vec<T> {
        self.iter().cloned().collect()
    }
}

pub struct ListIter<'a, T> {
    current: Option<&'a Node<T>>,
}

impl<'a, T> Iterator for ListIter<'a, T> {
    type Item = &'a T;
    
    fn next(&mut self) -> Option<Self::Item> {
        self.current.map(|node| {
            self.current = node.next.as_deref();
            &node.value
        })
    }
}

impl<T: std::fmt::Debug + Clone> Default for LinkedList<T> {
    fn default() -> Self {
        Self::new()
    }
}

fn main() {
    let mut list: LinkedList<i32> = LinkedList::new();
    
    println!("=== LinkedList Demo ===");
    
    // Push elements
    for i in 1..=5 {
        list.push_front(i);
        println!("Pushed {}, size: {}", i, list.len());
    }
    
    println!("\nList contents: {:?}", list.to_vec());
    println!("Peek: {:?}", list.peek());
    println!("Contains 3: {}", list.contains(&3));
    println!("Contains 10: {}", list.contains(&10));
    
    // Pop elements
    println!("\nPopping elements:");
    while let Some(val) = list.pop_front() {
        println!("  Popped: {}", val);
    }
    
    println!("List empty: {}", list.is_empty());
    
    // Reverse
    let mut list2: LinkedList<&str> = LinkedList::new();
    for word in ["rust", "is", "awesome"] {
        list2.push_front(word);
    }
    
    println!("\nBefore reverse: {:?}", list2.to_vec());
    list2.reverse();
    println!("After reverse: {:?}", list2.to_vec());
}
```

---

## สรุป Smart Pointers

| Smart Pointer | Use Case | Thread-safe | Mutable |
|--------------|----------|-------------|---------|
| `Box<T>` | Heap allocation, recursive types | ✓ (ownership) | ✓ |
| `Rc<T>` | Shared ownership, single-thread | ✗ | ✗ |
| `Arc<T>` | Shared ownership, multi-thread | ✓ | ✗ |
| `RefCell<T>` | Interior mutability | ✗ | ✓ (runtime) |
| `Rc<RefCell<T>>` | Shared mutable, single-thread | ✗ | ✓ |
| `Arc<Mutex<T>>` | Shared mutable, multi-thread | ✓ | ✓ |
| `Cow<T>` | Clone on write optimization | - | - |
| `Pin<T>` | Self-referential types, async | - | - |

---

## Navigation

- [← Part 016: Async/Await และ Tokio](../part_016/README.md)
- [→ Part 018: Advanced Cargo and Tooling](../part_018/README.md)
- [กลับหน้าหลัก](../../README.md)
