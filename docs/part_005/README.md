# Part 005: Structs และ Enums 🏗️

## 🎯 เป้าหมายของ Part นี้

- สร้างและใช้ Structs ได้
- เข้าใจ Methods และ Associated Functions
- สร้างและใช้ Enums ได้
- ใช้ Option<T> และ Result<T, E>
- ใช้ impl blocks

---

## 1. Structs

### 1.1 การสร้าง Struct พื้นฐาน

```rust
// Named struct
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}

// Tuple struct
struct Color(u8, u8, u8);
struct Point(f64, f64);

// Unit-like struct
struct AlwaysEqual;

fn main() {
    // สร้าง instance
    let user1 = User {
        email: String::from("user@example.com"),
        username: String::from("someuser"),
        active: true,
        sign_in_count: 1,
    };

    // Access fields
    println!("Username: {}", user1.username);
    println!("Email: {}", user1.email);

    // Mutable struct (ทั้ง struct ต้องเป็น mut)
    let mut user2 = User {
        email: String::from("another@example.com"),
        username: String::from("another"),
        active: true,
        sign_in_count: 0,
    };
    user2.email = String::from("newemail@example.com");
    user2.sign_in_count += 1;

    // Struct update syntax
    let user3 = User {
        email: String::from("user3@example.com"),
        ..user1  // copy remaining fields from user1
        // ระวัง: user1 อาจถูก move ถ้า fields เป็น non-Copy type
    };
    println!("user3 username: {}", user3.username);  // copied from user1

    // Tuple structs
    let black = Color(0, 0, 0);
    let white = Color(255, 255, 255);
    let red = Color(255, 0, 0);

    println!("black: ({}, {}, {})", black.0, black.1, black.2);

    let origin = Point(0.0, 0.0);
    let p = Point(3.0, 4.0);

    // Unit struct
    let _subject = AlwaysEqual;
}

fn build_user(email: String, username: String) -> User {
    // Field init shorthand (ชื่อ field เหมือนกับ variable)
    User {
        email,      // เหมือน email: email
        username,   // เหมือน username: username
        active: true,
        sign_in_count: 1,
    }
}
```

### 1.2 Methods กับ impl

```rust
#[derive(Debug)]
struct Rectangle {
    width: f64,
    height: f64,
}

impl Rectangle {
    // Associated function (ไม่มี self) - เหมือน static method
    fn new(width: f64, height: f64) -> Self {
        Self { width, height }
    }

    fn square(size: f64) -> Self {
        Self { width: size, height: size }
    }

    // Method (มี &self) - immutable
    fn area(&self) -> f64 {
        self.width * self.height
    }

    fn perimeter(&self) -> f64 {
        2.0 * (self.width + self.height)
    }

    fn is_square(&self) -> bool {
        (self.width - self.height).abs() < f64::EPSILON
    }

    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }

    // Method ที่ modify (มี &mut self)
    fn scale(&mut self, factor: f64) {
        self.width *= factor;
        self.height *= factor;
    }

    // Method ที่ consume (มี self)
    fn into_square(self) -> Rectangle {
        let size = self.width.min(self.height);
        Rectangle::square(size)
    }
}

// Display trait implementation
use std::fmt;
impl fmt::Display for Rectangle {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Rectangle({}×{})", self.width, self.height)
    }
}

fn main() {
    // Associated functions
    let rect1 = Rectangle::new(10.0, 5.0);
    let square = Rectangle::square(4.0);

    println!("{}", rect1);
    println!("{}", square);

    // Methods
    println!("Area: {}", rect1.area());
    println!("Perimeter: {}", rect1.perimeter());
    println!("Is square: {}", rect1.is_square());
    println!("Square is square: {}", square.is_square());

    let rect2 = Rectangle::new(8.0, 3.0);
    println!("rect1 can hold rect2: {}", rect1.can_hold(&rect2));
    println!("rect2 can hold rect1: {}", rect2.can_hold(&rect1));

    // Mutable method
    let mut rect3 = Rectangle::new(5.0, 3.0);
    rect3.scale(2.0);
    println!("After scale(2): {}", rect3);

    // Method chaining (ถ้า return self)
    // consume method
    let rect4 = Rectangle::new(10.0, 6.0);
    let sq = rect4.into_square();  // rect4 ถูก consume
    println!("Converted to square: {}", sq);

    // Debug format
    println!("{:?}", rect1);
    println!("{:#?}", rect1);
}
```

### 1.3 Multiple impl Blocks

```rust
struct Circle {
    radius: f64,
}

impl Circle {
    fn new(radius: f64) -> Self {
        Self { radius }
    }

    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
}

// สามารถมีหลาย impl blocks ได้
impl Circle {
    fn circumference(&self) -> f64 {
        2.0 * std::f64::consts::PI * self.radius
    }

    fn diameter(&self) -> f64 {
        2.0 * self.radius
    }
}

impl std::fmt::Display for Circle {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "Circle(r={})", self.radius)
    }
}

fn main() {
    let c = Circle::new(5.0);
    println!("{}", c);
    println!("Area: {:.2}", c.area());
    println!("Circumference: {:.2}", c.circumference());
    println!("Diameter: {:.1}", c.diameter());
}
```

---

## 2. Enums

### 2.1 Basic Enums

```rust
#[derive(Debug, PartialEq)]
enum Direction {
    North,
    South,
    East,
    West,
}

#[derive(Debug)]
enum Season {
    Spring,
    Summer,
    Autumn,
    Winter,
}

impl Season {
    fn description(&self) -> &str {
        match self {
            Season::Spring => "ฤดูใบไม้ผลิ - อากาศอบอุ่น ดอกไม้บาน",
            Season::Summer => "ฤดูร้อน - อากาศร้อน",
            Season::Autumn => "ฤดูใบไม้ร่วง - ใบไม้เปลี่ยนสี",
            Season::Winter => "ฤดูหนาว - อากาศหนาว หิมะตก",
        }
    }

    fn next(&self) -> Season {
        match self {
            Season::Spring => Season::Summer,
            Season::Summer => Season::Autumn,
            Season::Autumn => Season::Winter,
            Season::Winter => Season::Spring,
        }
    }
}

fn main() {
    let dir = Direction::North;
    println!("{:?}", dir);

    // ใช้ match
    let arrow = match dir {
        Direction::North => "↑",
        Direction::South => "↓",
        Direction::East  => "→",
        Direction::West  => "←",
    };
    println!("Arrow: {}", arrow);

    // Enum comparison
    let dir2 = Direction::North;
    if dir == dir2 {
        println!("Same direction");
    }

    // Seasons
    let season = Season::Spring;
    println!("{:?}: {}", season, season.description());
    println!("Next: {:?}", season.next());

    // Iterate-like with array
    let seasons = [Season::Spring, Season::Summer, Season::Autumn, Season::Winter];
    for s in &seasons {
        println!("{:?}: {}", s, s.description());
    }
}
```

### 2.2 Enums with Data

```rust
#[derive(Debug)]
enum Shape {
    Circle(f64),
    Rectangle(f64, f64),
    Triangle(f64, f64, f64),
    Point,
}

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle(r) => std::f64::consts::PI * r * r,
            Shape::Rectangle(w, h) => w * h,
            Shape::Triangle(a, b, c) => {
                // Heron's formula
                let s = (a + b + c) / 2.0;
                (s * (s - a) * (s - b) * (s - c)).sqrt()
            },
            Shape::Point => 0.0,
        }
    }

    fn perimeter(&self) -> f64 {
        match self {
            Shape::Circle(r) => 2.0 * std::f64::consts::PI * r,
            Shape::Rectangle(w, h) => 2.0 * (w + h),
            Shape::Triangle(a, b, c) => a + b + c,
            Shape::Point => 0.0,
        }
    }

    fn name(&self) -> &str {
        match self {
            Shape::Circle(_) => "วงกลม",
            Shape::Rectangle(_, _) => "สี่เหลี่ยม",
            Shape::Triangle(_, _, _) => "สามเหลี่ยม",
            Shape::Point => "จุด",
        }
    }
}

// Enum กับ struct data
#[derive(Debug)]
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(u8, u8, u8),
}

impl Message {
    fn process(&self) {
        match self {
            Message::Quit => println!("Quit message received"),
            Message::Move { x, y } => println!("Move to ({}, {})", x, y),
            Message::Write(text) => println!("Write: {}", text),
            Message::ChangeColor(r, g, b) => println!("Change color to ({}, {}, {})", r, g, b),
        }
    }
}

fn main() {
    let shapes = vec![
        Shape::Circle(5.0),
        Shape::Rectangle(4.0, 6.0),
        Shape::Triangle(3.0, 4.0, 5.0),
        Shape::Point,
    ];

    println!("{:<15} {:>10} {:>12}", "รูปทรง", "พื้นที่", "เส้นรอบรูป");
    println!("{}", "─".repeat(40));
    for shape in &shapes {
        println!("{:<15} {:>10.2} {:>12.2}",
            shape.name(),
            shape.area(),
            shape.perimeter()
        );
    }

    println!();
    let messages = vec![
        Message::Quit,
        Message::Move { x: 10, y: 20 },
        Message::Write(String::from("Hello, Rust!")),
        Message::ChangeColor(255, 128, 0),
    ];

    for msg in &messages {
        msg.process();
    }
}
```

---

## 3. Option<T>

### 3.1 Option พื้นฐาน

```rust
fn divide(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 {
        None
    } else {
        Some(a / b)
    }
}

fn find_first_even(numbers: &[i32]) -> Option<i32> {
    numbers.iter().find(|&&x| x % 2 == 0).copied()
}

fn main() {
    // Basic Option usage
    let some_value: Option<i32> = Some(42);
    let no_value: Option<i32> = None;

    println!("{:?}", some_value);  // Some(42)
    println!("{:?}", no_value);    // None

    // Pattern matching
    match some_value {
        Some(v) => println!("Got: {}", v),
        None => println!("Nothing"),
    }

    // if let
    if let Some(v) = some_value {
        println!("Value: {}", v);
    }

    // unwrap (panic if None) - ใช้เฉพาะเมื่อแน่ใจ 100%
    let x = some_value.unwrap();
    println!("Unwrapped: {}", x);

    // unwrap_or - default value
    let y = no_value.unwrap_or(0);
    println!("Or default: {}", y);

    // unwrap_or_else - default via closure
    let z = no_value.unwrap_or_else(|| 42);
    println!("Or computed: {}", z);

    // expect - panic with message
    // let bad = no_value.expect("Should have value");

    // map - transform Some value
    let doubled = some_value.map(|v| v * 2);
    println!("Doubled: {:?}", doubled);  // Some(84)

    let doubled_none = no_value.map(|v| v * 2);
    println!("Doubled None: {:?}", doubled_none);  // None

    // and_then (flatMap) - chain operations
    let result = some_value
        .and_then(|v| if v > 0 { Some(v) } else { None })
        .and_then(|v| Some(v.to_string()));
    println!("Chained: {:?}", result);

    // filter
    let filtered = some_value.filter(|&v| v > 100);
    println!("Filtered (>100): {:?}", filtered);  // None

    // is_some, is_none
    println!("is_some: {}", some_value.is_some());
    println!("is_none: {}", no_value.is_none());

    // Function examples
    println!("\n10 / 3 = {:?}", divide(10.0, 3.0));
    println!("10 / 0 = {:?}", divide(10.0, 0.0));

    let numbers = [1, 3, 5, 4, 7];
    println!("First even: {:?}", find_first_even(&numbers));

    let odds = [1, 3, 5, 7];
    println!("First even in odds: {:?}", find_first_even(&odds));
}
```

### 3.2 Option Chaining

```rust
struct Address {
    street: Option<String>,
    city: String,
    country: String,
}

struct Person {
    name: String,
    age: u32,
    address: Option<Address>,
}

impl Person {
    fn city(&self) -> Option<&str> {
        self.address.as_ref().map(|addr| addr.city.as_str())
    }

    fn street(&self) -> Option<&str> {
        self.address.as_ref()
            .and_then(|addr| addr.street.as_deref())
    }
}

fn main() {
    let person_with_address = Person {
        name: String::from("สมชาย"),
        age: 30,
        address: Some(Address {
            street: Some(String::from("123 ถ.สุขุมวิท")),
            city: String::from("กรุงเทพฯ"),
            country: String::from("ไทย"),
        }),
    };

    let person_no_address = Person {
        name: String::from("สมหญิง"),
        age: 25,
        address: None,
    };

    println!("{}'s city: {:?}", person_with_address.name, person_with_address.city());
    println!("{}'s street: {:?}", person_with_address.name, person_with_address.street());
    println!("{}'s city: {:?}", person_no_address.name, person_no_address.city());

    // ? operator กับ Option
    fn get_street_length(person: &Person) -> Option<usize> {
        Some(person.address.as_ref()?.street.as_ref()?.len())
    }

    println!("Street length: {:?}", get_street_length(&person_with_address));
    println!("Street length (no addr): {:?}", get_street_length(&person_no_address));
}
```

---

## 4. Result<T, E>

### 4.1 Result พื้นฐาน

```rust
use std::num::ParseIntError;

fn parse_positive(s: &str) -> Result<u32, String> {
    match s.trim().parse::<i32>() {
        Err(e) => Err(format!("Parse error: {}", e)),
        Ok(n) if n < 0 => Err(format!("Expected positive, got {}", n)),
        Ok(n) => Ok(n as u32),
    }
}

fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("Division by zero"))
    } else {
        Ok(a / b)
    }
}

fn main() {
    // Basic Result
    let ok: Result<i32, String> = Ok(42);
    let err: Result<i32, String> = Err(String::from("something went wrong"));

    // Pattern matching
    match ok {
        Ok(v) => println!("Success: {}", v),
        Err(e) => println!("Error: {}", e),
    }

    // unwrap (panic if Err)
    let value = ok.unwrap();
    println!("Value: {}", value);

    // unwrap_or
    let value = err.unwrap_or(0);
    println!("Or default: {}", value);

    // map
    let doubled = ok.map(|v| v * 2);
    println!("Doubled: {:?}", doubled);

    // map_err
    let mapped = err.map_err(|e| format!("Wrapped: {}", e));
    println!("Mapped error: {:?}", mapped);

    // and_then
    let result = Ok::<i32, String>(10)
        .and_then(|v| if v > 0 { Ok(v * 2) } else { Err("Negative".to_string()) });
    println!("and_then: {:?}", result);

    // is_ok, is_err
    println!("ok.is_ok(): {}", ok.is_ok());
    println!("err.is_err(): {}", err.is_err());

    // Function usage
    let tests = ["42", "-5", "hello", "100"];
    for test in &tests {
        match parse_positive(test) {
            Ok(n) => println!("'{}' → {}", test, n),
            Err(e) => println!("'{}' → Error: {}", test, e),
        }
    }

    // Collect Results
    let inputs = vec!["1", "2", "3", "4", "5"];
    let results: Result<Vec<u32>, _> = inputs.iter()
        .map(|s| parse_positive(s))
        .collect();
    println!("All valid: {:?}", results);

    let inputs_with_error = vec!["1", "bad", "3"];
    let results: Result<Vec<u32>, _> = inputs_with_error.iter()
        .map(|s| parse_positive(s))
        .collect();
    println!("With error: {:?}", results);
}
```

### 4.2 ? Operator

```rust
use std::fs;
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Parse(ParseIntError),
    Custom(String),
}

impl From<io::Error> for AppError {
    fn from(e: io::Error) -> Self {
        AppError::Io(e)
    }
}

impl From<ParseIntError> for AppError {
    fn from(e: ParseIntError) -> Self {
        AppError::Parse(e)
    }
}

impl std::fmt::Display for AppError {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        match self {
            AppError::Io(e) => write!(f, "IO error: {}", e),
            AppError::Parse(e) => write!(f, "Parse error: {}", e),
            AppError::Custom(s) => write!(f, "Custom error: {}", s),
        }
    }
}

fn read_username() -> Result<String, AppError> {
    // ? operator: ถ้า Err ให้ return Err ทันที (เหมือน early return)
    let content = fs::read_to_string("username.txt")?;
    let username = content.trim().to_string();

    if username.is_empty() {
        return Err(AppError::Custom("Username cannot be empty".to_string()));
    }

    Ok(username)
}

fn process_age(age_str: &str) -> Result<u32, AppError> {
    let age: u32 = age_str.trim().parse()?;  // ? converts ParseIntError to AppError

    if age > 150 {
        return Err(AppError::Custom(format!("Age {} seems unrealistic", age)));
    }

    Ok(age)
}

fn main() {
    // ? operator แบบง่าย
    fn simple_parse(s: &str) -> Result<i32, std::num::ParseIntError> {
        let trimmed = s.trim();
        let n = trimmed.parse()?;  // ? operator
        Ok(n * 2)
    }

    println!("{:?}", simple_parse("  42  "));   // Ok(84)
    println!("{:?}", simple_parse("not_num"));  // Err(...)

    // Multiple ? in chain
    fn compute(a: &str, b: &str) -> Result<i32, std::num::ParseIntError> {
        let x: i32 = a.parse()?;
        let y: i32 = b.parse()?;
        Ok(x + y)
    }

    println!("{:?}", compute("10", "20"));    // Ok(30)
    println!("{:?}", compute("10", "bad"));   // Err(...)

    // With custom errors
    match process_age("25") {
        Ok(age) => println!("Age: {}", age),
        Err(e) => println!("Error: {}", e),
    }

    match process_age("not_a_number") {
        Ok(age) => println!("Age: {}", age),
        Err(e) => println!("Error: {}", e),
    }

    match process_age("200") {
        Ok(age) => println!("Age: {}", age),
        Err(e) => println!("Error: {}", e),
    }
}
```

---

## 5. ตัวอย่างใหญ่: Product Catalog

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
enum Category {
    Electronics,
    Clothing,
    Food,
    Books,
}

impl std::fmt::Display for Category {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        match self {
            Category::Electronics => write!(f, "Electronics"),
            Category::Clothing => write!(f, "Clothing"),
            Category::Food => write!(f, "Food"),
            Category::Books => write!(f, "Books"),
        }
    }
}

#[derive(Debug, Clone)]
struct Product {
    id: u32,
    name: String,
    price: f64,
    stock: u32,
    category: Category,
}

impl Product {
    fn new(id: u32, name: &str, price: f64, stock: u32, category: Category) -> Self {
        Self {
            id,
            name: name.to_string(),
            price,
            stock,
            category,
        }
    }

    fn is_available(&self) -> bool {
        self.stock > 0
    }

    fn apply_discount(&mut self, percent: f64) {
        self.price *= 1.0 - (percent / 100.0);
    }
}

impl std::fmt::Display for Product {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "[{}] {} - ฿{:.2} ({})",
            self.id, self.name, self.price,
            if self.is_available() {
                format!("มี {} ชิ้น", self.stock)
            } else {
                "หมด".to_string()
            }
        )
    }
}

struct Catalog {
    products: Vec<Product>,
}

impl Catalog {
    fn new() -> Self {
        Self { products: Vec::new() }
    }

    fn add(&mut self, product: Product) {
        self.products.push(product);
    }

    fn find_by_id(&self, id: u32) -> Option<&Product> {
        self.products.iter().find(|p| p.id == id)
    }

    fn search(&self, query: &str) -> Vec<&Product> {
        let q = query.to_lowercase();
        self.products.iter()
            .filter(|p| p.name.to_lowercase().contains(&q))
            .collect()
    }

    fn by_category(&self, category: &str) -> Vec<&Product> {
        self.products.iter()
            .filter(|p| format!("{}", p.category).to_lowercase() == category.to_lowercase())
            .collect()
    }

    fn available(&self) -> Vec<&Product> {
        self.products.iter()
            .filter(|p| p.is_available())
            .collect()
    }

    fn price_range(&self, min: f64, max: f64) -> Vec<&Product> {
        self.products.iter()
            .filter(|p| p.price >= min && p.price <= max)
            .collect()
    }

    fn stats(&self) -> HashMap<String, f64> {
        let mut stats = HashMap::new();
        if self.products.is_empty() {
            return stats;
        }

        let prices: Vec<f64> = self.products.iter().map(|p| p.price).collect();
        stats.insert("min_price".to_string(), prices.iter().cloned().fold(f64::INFINITY, f64::min));
        stats.insert("max_price".to_string(), prices.iter().cloned().fold(f64::NEG_INFINITY, f64::max));
        stats.insert("avg_price".to_string(), prices.iter().sum::<f64>() / prices.len() as f64);
        stats.insert("total_products".to_string(), self.products.len() as f64);
        stats.insert("available_products".to_string(), self.products.iter().filter(|p| p.is_available()).count() as f64);
        stats
    }
}

fn main() {
    let mut catalog = Catalog::new();

    catalog.add(Product::new(1, "Laptop", 35000.0, 10, Category::Electronics));
    catalog.add(Product::new(2, "T-Shirt", 299.0, 50, Category::Clothing));
    catalog.add(Product::new(3, "Rice", 45.0, 100, Category::Food));
    catalog.add(Product::new(4, "Rust Programming Book", 899.0, 5, Category::Books));
    catalog.add(Product::new(5, "Headphones", 2500.0, 0, Category::Electronics));
    catalog.add(Product::new(6, "Python Book", 750.0, 8, Category::Books));
    catalog.add(Product::new(7, "Jeans", 1200.0, 20, Category::Clothing));
    catalog.add(Product::new(8, "Keyboard", 3500.0, 15, Category::Electronics));

    println!("=== Product Catalog ===\n");

    println!("All Products:");
    for p in &catalog.products {
        println!("  {}", p);
    }

    println!("\nBooks:");
    for p in catalog.by_category("books") {
        println!("  {}", p);
    }

    println!("\nAvailable Products:");
    for p in catalog.available() {
        println!("  {}", p);
    }

    println!("\nSearch 'book':");
    for p in catalog.search("book") {
        println!("  {}", p);
    }

    println!("\nPrice Range 500-5000:");
    for p in catalog.price_range(500.0, 5000.0) {
        println!("  {}", p);
    }

    println!("\nFind by ID 3: {:?}", catalog.find_by_id(3).map(|p| &p.name));
    println!("Find by ID 99: {:?}", catalog.find_by_id(99));

    println!("\nStats:");
    let stats = catalog.stats();
    let mut stat_keys: Vec<&String> = stats.keys().collect();
    stat_keys.sort();
    for key in stat_keys {
        println!("  {}: {:.2}", key, stats[key]);
    }
}
```

---

## 6. สรุปและ Exercises

### 6.1 สิ่งที่เรียนรู้

✅ Named structs, Tuple structs, Unit structs  
✅ Methods (&self, &mut self, self)  
✅ Associated functions (::new)  
✅ impl blocks  
✅ Enums แบบต่างๆ (ไม่มีข้อมูล, มีข้อมูล, มี struct)  
✅ Option<T> และ Result<T, E>  
✅ ? operator  
✅ Pattern matching กับ enums  

### 6.2 Exercises

**Exercise 1: Bank Account**
```rust
struct BankAccount { ... }
// Implement: deposit, withdraw, balance, transfer
// ใช้ Result สำหรับ error handling
```

**Exercise 2: Student Grade System**
```rust
struct Student { ... }
enum Grade { A, B, C, D, F }
// Implement: calculate_grade, is_passed, gpa
```

**Exercise 3: Simple Parser**
```rust
enum Token { Number(f64), Plus, Minus, Multiply, Divide }
// Parse "1 + 2 * 3" เป็น Vec<Token>
```

---

*[← Part 004: Ownership และ Borrowing](../part_004/README.md) | [Part 006: Pattern Matching →](../part_006/README.md)*
