# Part 009: Traits และ Generics 🧩

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Traits (คล้าย interfaces)
- เขียน Generic functions และ structs
- Trait bounds
- Default implementations
- Trait objects (dynamic dispatch)

---

## 1. Traits พื้นฐาน

### 1.1 สร้างและ Implement Trait

```rust
// Trait = ชุดของ methods ที่ type ต้อง implement
trait Animal {
    // Required methods
    fn name(&self) -> &str;
    fn sound(&self) -> &str;

    // Default implementation
    fn describe(&self) -> String {
        format!("{} makes the sound '{}'", self.name(), self.sound())
    }

    fn is_loud(&self) -> bool {
        self.sound().len() > 3
    }
}

struct Dog {
    name: String,
}

struct Cat {
    name: String,
}

struct Lion {
    name: String,
}

impl Animal for Dog {
    fn name(&self) -> &str { &self.name }
    fn sound(&self) -> &str { "Woof" }
}

impl Animal for Cat {
    fn name(&self) -> &str { &self.name }
    fn sound(&self) -> &str { "Meow" }

    // Override default implementation
    fn describe(&self) -> String {
        format!("{} purrs... then ignores you", self.name())
    }
}

impl Animal for Lion {
    fn name(&self) -> &str { &self.name }
    fn sound(&self) -> &str { "ROOOAAARRR" }
}

fn print_animal(animal: &impl Animal) {
    println!("{}", animal.describe());
    println!("  Is loud: {}", animal.is_loud());
}

fn main() {
    let dog = Dog { name: "Buddy".to_string() };
    let cat = Cat { name: "Whiskers".to_string() };
    let lion = Lion { name: "Simba".to_string() };

    print_animal(&dog);
    print_animal(&cat);
    print_animal(&lion);

    // Trait objects (dynamic dispatch)
    let animals: Vec<Box<dyn Animal>> = vec![
        Box::new(Dog { name: "Rex".to_string() }),
        Box::new(Cat { name: "Luna".to_string() }),
        Box::new(Lion { name: "Aslan".to_string() }),
    ];

    println!("\nAll animals:");
    for animal in &animals {
        println!("  {}", animal.describe());
    }
}
```

### 1.2 Trait Bounds

```rust
use std::fmt;

// Trait bound: T ต้อง implement Display และ PartialOrd
fn print_largest<T: fmt::Display + PartialOrd>(list: &[T]) {
    if list.is_empty() {
        println!("Empty list");
        return;
    }
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    println!("Largest: {}", largest);
}

// where clause (อ่านง่ายกว่าเมื่อหลาย bounds)
fn compare_and_display<T, U>(t: &T, u: &U)
where
    T: fmt::Display + PartialOrd,
    U: fmt::Display + Clone,
{
    println!("T: {}, U: {}", t, u);
}

// Returning impl Trait
fn make_adder(x: i32) -> impl Fn(i32) -> i32 {
    move |y| x + y
}

fn main() {
    let numbers = vec![34, 50, 25, 100, 65];
    print_largest(&numbers);

    let strings = vec!["hello", "world", "rust"];
    print_largest(&strings);

    let add5 = make_adder(5);
    println!("5 + 10 = {}", add5(10));
    println!("5 + 20 = {}", add5(20));
}
```

### 1.3 Important Standard Traits

```rust
// Clone, Copy
#[derive(Clone, Copy, Debug)]
struct Point {
    x: f64,
    y: f64,
}

// Display, Debug
use std::fmt;

struct Matrix {
    data: [[f64; 2]; 2],
}

impl fmt::Display for Matrix {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "| {:.2} {:.2} |\n| {:.2} {:.2} |",
            self.data[0][0], self.data[0][1],
            self.data[1][0], self.data[1][1])
    }
}

impl fmt::Debug for Matrix {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Matrix({:?})", self.data)
    }
}

// PartialEq, Eq
#[derive(Debug, PartialEq)]
struct Color {
    r: u8, g: u8, b: u8,
}

// PartialOrd, Ord
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord)]
struct Version {
    major: u32,
    minor: u32,
    patch: u32,
}

// Iterator trait
struct Counter {
    count: u32,
    max: u32,
}

impl Counter {
    fn new(max: u32) -> Self {
        Self { count: 0, max }
    }
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.count < self.max {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}

fn main() {
    // Clone, Copy
    let p1 = Point { x: 3.0, y: 4.0 };
    let p2 = p1;  // Copy
    let p3 = p1.clone();  // Clone
    println!("p1={:?}, p2={:?}, p3={:?}", p1, p2, p3);

    // Display
    let m = Matrix { data: [[1.0, 2.0], [3.0, 4.0]] };
    println!("{}", m);
    println!("{:?}", m);

    // PartialEq
    let red = Color { r: 255, g: 0, b: 0 };
    let also_red = Color { r: 255, g: 0, b: 0 };
    let blue = Color { r: 0, g: 0, b: 255 };
    println!("red == also_red: {}", red == also_red);
    println!("red == blue: {}", red == blue);

    // Ord
    let v1 = Version { major: 1, minor: 2, patch: 3 };
    let v2 = Version { major: 1, minor: 3, patch: 0 };
    println!("v1 < v2: {}", v1 < v2);
    let mut versions = vec![
        Version { major: 2, minor: 0, patch: 0 },
        Version { major: 1, minor: 9, patch: 0 },
        Version { major: 1, minor: 0, patch: 5 },
    ];
    versions.sort();
    println!("Sorted: {:?}", versions);

    // Custom Iterator
    let counter = Counter::new(5);
    let sum: u32 = counter.sum();
    println!("Sum 1..5 = {}", sum);

    // Iterator เช่นเดียวกับ built-in
    let result: Vec<u32> = Counter::new(5)
        .filter(|x| x % 2 == 0)
        .map(|x| x * x)
        .collect();
    println!("Even squares: {:?}", result);

    // zip two counters
    let pairs: Vec<(u32, u32)> = Counter::new(5)
        .zip(Counter::new(5).skip(1))
        .collect();
    println!("Pairs: {:?}", pairs);
}
```

---

## 2. Generics

### 2.1 Generic Functions

```rust
// Generic function
fn first<T>(list: &[T]) -> Option<&T> {
    list.first()
}

fn last<T>(list: &[T]) -> Option<&T> {
    list.last()
}

fn contains<T: PartialEq>(list: &[T], target: &T) -> bool {
    list.iter().any(|x| x == target)
}

fn max_item<T: PartialOrd + Copy>(list: &[T]) -> Option<T> {
    list.iter().copied().reduce(|a, b| if a > b { a } else { b })
}

fn main() {
    let numbers = vec![1, 5, 3, 7, 2];
    let strings = vec!["hello", "world", "rust"];

    println!("first: {:?}", first(&numbers));
    println!("last: {:?}", last(&strings));
    println!("contains 7: {}", contains(&numbers, &7));
    println!("contains 'rust': {}", contains(&strings, &"rust"));
    println!("max number: {:?}", max_item(&numbers));
}
```

### 2.2 Generic Structs

```rust
#[derive(Debug)]
struct Stack<T> {
    elements: Vec<T>,
}

impl<T> Stack<T> {
    fn new() -> Self {
        Self { elements: Vec::new() }
    }

    fn push(&mut self, item: T) {
        self.elements.push(item);
    }

    fn pop(&mut self) -> Option<T> {
        self.elements.pop()
    }

    fn peek(&self) -> Option<&T> {
        self.elements.last()
    }

    fn is_empty(&self) -> bool {
        self.elements.is_empty()
    }

    fn size(&self) -> usize {
        self.elements.len()
    }
}

// Generic with multiple type params
#[derive(Debug)]
struct Pair<T, U> {
    first: T,
    second: U,
}

impl<T: std::fmt::Display, U: std::fmt::Display> Pair<T, U> {
    fn new(first: T, second: U) -> Self {
        Self { first, second }
    }

    fn show(&self) {
        println!("({}, {})", self.first, self.second);
    }
}

impl<T: PartialOrd + std::fmt::Display, U: PartialOrd + std::fmt::Display> Pair<T, U> {
    fn cmp_display(&self) {
        if self.first >= self.second {
            // can't compare T and U directly - they're different types
        }
    }
}

fn main() {
    // Stack<i32>
    let mut int_stack: Stack<i32> = Stack::new();
    int_stack.push(1);
    int_stack.push(2);
    int_stack.push(3);
    println!("Stack: {:?}", int_stack);
    println!("Peek: {:?}", int_stack.peek());
    println!("Pop: {:?}", int_stack.pop());
    println!("Size: {}", int_stack.size());

    // Stack<String>
    let mut str_stack: Stack<String> = Stack::new();
    str_stack.push("hello".to_string());
    str_stack.push("world".to_string());
    println!("String stack: {:?}", str_stack);

    // Pair
    let p = Pair::new(42, "hello");
    p.show();

    let p2 = Pair::new(3.14, true);
    p2.show();
}
```

### 2.3 Trait Objects vs Generics

```rust
use std::fmt;

trait Shape: fmt::Display {
    fn area(&self) -> f64;
    fn perimeter(&self) -> f64;
}

struct Circle { radius: f64 }
struct Rect { width: f64, height: f64 }

impl Shape for Circle {
    fn area(&self) -> f64 { std::f64::consts::PI * self.radius * self.radius }
    fn perimeter(&self) -> f64 { 2.0 * std::f64::consts::PI * self.radius }
}

impl fmt::Display for Circle {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Circle(r={})", self.radius)
    }
}

impl Shape for Rect {
    fn area(&self) -> f64 { self.width * self.height }
    fn perimeter(&self) -> f64 { 2.0 * (self.width + self.height) }
}

impl fmt::Display for Rect {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Rect({}×{})", self.width, self.height)
    }
}

// Static dispatch (faster, monomorphized)
fn print_shape_info<T: Shape>(shape: &T) {
    println!("{}: area={:.2}, perimeter={:.2}", shape, shape.area(), shape.perimeter());
}

// Dynamic dispatch (flexible, works with heterogeneous collections)
fn total_area(shapes: &[Box<dyn Shape>]) -> f64 {
    shapes.iter().map(|s| s.area()).sum()
}

fn main() {
    // Static dispatch - each call is monomorphized
    print_shape_info(&Circle { radius: 5.0 });
    print_shape_info(&Rect { width: 4.0, height: 6.0 });

    // Dynamic dispatch - heterogeneous collection
    let shapes: Vec<Box<dyn Shape>> = vec![
        Box::new(Circle { radius: 3.0 }),
        Box::new(Rect { width: 4.0, height: 5.0 }),
        Box::new(Circle { radius: 1.5 }),
    ];

    for shape in &shapes {
        print_shape_info(shape.as_ref());
    }
    println!("Total area: {:.2}", total_area(&shapes));
}
```

---

## 3. สรุปและ Exercises

### 3.1 สิ่งที่เรียนรู้

✅ Traits: required + default methods  
✅ Trait bounds (simple, where clause)  
✅ Standard traits (Clone, Display, Iterator, etc.)  
✅ Generic functions และ structs  
✅ Static vs dynamic dispatch  
✅ Trait objects (Box<dyn Trait>)  

### 3.2 Exercise

**Exercise: Generic Cache**
```rust
struct Cache<K, V> { ... }
// Implement: get, set, remove, clear
// Bounded by K: Hash + Eq, V: Clone
// เพิ่ม TTL (time-to-live) support
```

---

*[← Part 008: Error Handling](../part_008/README.md) | [Part 010: Lifetimes →](../part_010/README.md)*
