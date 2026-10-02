# Part 012: Modules, Crates, และ Packages 📦

## 🎯 เป้าหมายของ Part นี้

- เข้าใจระบบ module ใน Rust
- visibility modifiers (pub, pub(crate), pub(super))
- use statements และ path
- File-based modules
- Library vs Binary crates
- Cargo workspace
- Popular crates
- สร้าง library crate ด้วย modules

---

## 1. Module พื้นฐาน (mod keyword)

```rust
// src/main.rs

// Module ถูกนิยาม inline ด้วย mod keyword
mod greetings {
    // ค่า default คือ private
    fn private_helper() -> &'static str {
        "I'm private"
    }

    // pub ทำให้ accessible จาก outside
    pub fn hello() -> String {
        format!("Hello from greetings! ({})", private_helper())
    }

    pub fn goodbye() -> String {
        String::from("Goodbye!")
    }
}

mod math {
    pub fn add(a: i32, b: i32) -> i32 { a + b }
    pub fn subtract(a: i32, b: i32) -> i32 { a - b }
    
    fn internal_multiply(a: i32, b: i32) -> i32 { a * b }
    
    pub fn square(n: i32) -> i32 {
        internal_multiply(n, n)  // สามารถเรียก private function ภายใน module เดียวกัน
    }
}

fn main() {
    // เรียกใช้ด้วย path แบบ full
    println!("{}", greetings::hello());
    println!("{}", greetings::goodbye());
    // greetings::private_helper();  // ERROR: private!

    println!("{}", math::add(10, 5));      // 15
    println!("{}", math::subtract(10, 5)); // 5
    println!("{}", math::square(7));       // 49
}
```

---

## 2. Nested Modules

```rust
mod company {
    pub mod engineering {
        pub mod frontend {
            pub fn build_ui() -> &'static str { "Building UI..." }
            
            pub mod components {
                pub fn render_button() -> &'static str { "Rendering Button" }
            }
        }
        
        pub mod backend {
            pub fn process_request() -> &'static str { "Processing request..." }
            
            // ใช้ super:: เพื่อ refer to parent module
            pub fn full_pipeline() -> String {
                let ui = super::frontend::build_ui();
                let api = process_request();
                format!("{} + {}", ui, api)
            }
        }
        
        // ใช้ crate:: เพื่อ refer to crate root
        pub fn team_name() -> &'static str {
            "Engineering Team"
        }
    }
    
    pub mod hr {
        pub fn hire(name: &str) -> String {
            format!("Welcome, {}!", name)
        }
        
        // pub(super) ให้เข้าถึงได้จาก parent module เท่านั้น
        pub(super) fn secret_salary_formula() -> f64 {
            50000.0 * 1.5
        }
    }
    
    // Company module สามารถเข้าถึง hr::secret_salary_formula() ได้
    pub fn calculate_budget() -> f64 {
        hr::secret_salary_formula() * 10.0
    }
}

fn main() {
    // Full paths
    println!("{}", company::engineering::frontend::build_ui());
    println!("{}", company::engineering::backend::process_request());
    println!("{}", company::engineering::backend::full_pipeline());
    
    // Nested path
    println!("{}", company::engineering::frontend::components::render_button());
    
    println!("{}", company::hr::hire("Alice"));
    // company::hr::secret_salary_formula();  // ERROR: pub(super) - ไม่ให้จาก main
    
    println!("Budget: {}", company::calculate_budget());
}
```

---

## 3. Visibility Modifiers

```rust
pub mod visibility_demo {
    // pub - ทุกคนเข้าถึงได้
    pub struct PublicStruct {
        pub public_field: i32,
        private_field: i32,  // private แม้ struct จะเป็น public
    }
    
    impl PublicStruct {
        pub fn new(public: i32, private: i32) -> Self {
            PublicStruct {
                public_field: public,
                private_field: private,
            }
        }
        
        pub fn get_private(&self) -> i32 {
            self.private_field
        }
    }
    
    // pub(crate) - ทั้ง crate เข้าถึงได้ แต่ไม่ให้ external crate
    pub(crate) fn crate_only_function() -> &'static str {
        "Only accessible within this crate"
    }
    
    pub mod inner {
        // pub(super) - parent module เข้าถึงได้
        pub(super) fn super_accessible() -> &'static str {
            "Only parent can call me"
        }
        
        // pub(in crate::visibility_demo) - กำหนด path ที่เข้าถึงได้
        // pub(in crate::visibility_demo) fn path_restricted() -> &'static str { ... }
        
        pub fn public_inner() -> &'static str {
            "I'm public from inner"
        }
    }
    
    pub fn demonstrate() {
        // สามารถเรียก super_accessible ได้เพราะเป็น parent
        println!("{}", inner::super_accessible());
        println!("{}", inner::public_inner());
    }
}

fn main() {
    // pub struct fields
    let s = visibility_demo::PublicStruct::new(42, 100);
    println!("Public field: {}", s.public_field);
    println!("Private via method: {}", s.get_private());
    // s.private_field;  // ERROR: private
    
    // pub(crate) - accessible within crate
    println!("{}", visibility_demo::crate_only_function());
    
    // pub(super) - NOT accessible from here
    // visibility_demo::inner::super_accessible();  // ERROR
    println!("{}", visibility_demo::inner::public_inner());
    
    visibility_demo::demonstrate();
}
```

---

## 4. use Statements

```rust
use std::collections::HashMap;
use std::collections::HashSet;
// หรือใช้ nested paths
use std::collections::{BTreeMap, BTreeSet, VecDeque};

// use ใน module scope
mod geometry {
    use std::f64::consts::PI;
    
    pub struct Circle {
        pub radius: f64,
    }
    
    impl Circle {
        pub fn area(&self) -> f64 {
            PI * self.radius * self.radius
        }
        
        pub fn circumference(&self) -> f64 {
            2.0 * PI * self.radius
        }
    }
    
    pub struct Rectangle {
        pub width: f64,
        pub height: f64,
    }
    
    impl Rectangle {
        pub fn area(&self) -> f64 {
            self.width * self.height
        }
        
        pub fn perimeter(&self) -> f64 {
            2.0 * (self.width + self.height)
        }
    }
}

// Renaming with as
use std::io::Result as IoResult;
use geometry::Circle as Circ;
use geometry::Rectangle as Rect;

// Re-exporting with pub use
mod shapes {
    pub use crate::geometry::Circle;
    pub use crate::geometry::Rectangle;
    
    pub fn describe_circle(c: &Circle) -> String {
        format!("Circle: r={}, area={:.2}", c.radius, c.area())
    }
}

fn main() {
    // ใช้ HashMap ที่ import แล้ว
    let mut map: HashMap<String, i32> = HashMap::new();
    map.insert("a".to_string(), 1);
    
    let mut set: HashSet<i32> = HashSet::new();
    set.insert(42);
    
    let mut deque: VecDeque<i32> = VecDeque::new();
    deque.push_back(1);
    
    // ใช้ alias
    let c = Circ { radius: 5.0 };
    println!("Circle area: {:.2}", c.area());
    
    let r = Rect { width: 4.0, height: 6.0 };
    println!("Rectangle area: {:.2}", r.area());
    
    // ใช้ re-exported types
    let c2 = shapes::Circle { radius: 3.0 };
    println!("{}", shapes::describe_circle(&c2));
    
    // Glob import (ใช้ระวัง - อาจ shadow names)
    use std::collections::*;  // import ทั้งหมด
    let _linked: LinkedList<i32> = LinkedList::new();

    // use ใน function scope
    fn process() -> IoResult<()> {
        use std::io::{self, Write};
        io::stdout().flush()
    }
    process().unwrap();
}
```

---

## 5. File-based Modules

สำหรับ project ขนาดใหญ่ เราแยก modules ออกเป็นไฟล์

```
src/
├── main.rs          (หรือ lib.rs)
├── config.rs        (mod config;)
├── models/
│   ├── mod.rs       (mod models;)
│   ├── user.rs      (module user)
│   └── product.rs   (module product)
├── services/
│   ├── mod.rs
│   ├── user_service.rs
│   └── product_service.rs
└── utils/
    ├── mod.rs
    └── validation.rs
```

```rust
// src/models/user.rs
#[derive(Debug, Clone)]
pub struct User {
    pub id: u32,
    pub username: String,
    pub email: String,
    pub(crate) password_hash: String,
}

impl User {
    pub fn new(id: u32, username: &str, email: &str, password: &str) -> Self {
        User {
            id,
            username: username.to_string(),
            email: email.to_string(),
            password_hash: format!("hash_{}", password),  // simplified
        }
    }
    
    pub fn display_name(&self) -> &str {
        &self.username
    }
    
    pub(crate) fn verify_password(&self, password: &str) -> bool {
        self.password_hash == format!("hash_{}", password)
    }
}
```

```rust
// src/models/product.rs
#[derive(Debug, Clone)]
pub struct Product {
    pub id: u32,
    pub name: String,
    pub price: f64,
    pub category: String,
}

impl Product {
    pub fn new(id: u32, name: &str, price: f64, category: &str) -> Self {
        Product {
            id,
            name: name.to_string(),
            price,
            category: category.to_string(),
        }
    }
    
    pub fn discounted_price(&self, discount_pct: f64) -> f64 {
        self.price * (1.0 - discount_pct / 100.0)
    }
}
```

```rust
// src/models/mod.rs
pub mod user;
pub mod product;

// Re-export for convenience
pub use user::User;
pub use product::Product;
```

```rust
// src/services/user_service.rs
use crate::models::User;

pub struct UserService {
    users: Vec<User>,
}

impl UserService {
    pub fn new() -> Self {
        UserService { users: Vec::new() }
    }
    
    pub fn add_user(&mut self, user: User) {
        self.users.push(user);
    }
    
    pub fn find_by_id(&self, id: u32) -> Option<&User> {
        self.users.iter().find(|u| u.id == id)
    }
    
    pub fn find_by_email(&self, email: &str) -> Option<&User> {
        self.users.iter().find(|u| u.email == email)
    }
    
    pub fn authenticate(&self, email: &str, password: &str) -> Option<&User> {
        self.find_by_email(email)
            .filter(|u| u.verify_password(password))
    }
    
    pub fn all_users(&self) -> &[User] {
        &self.users
    }
}
```

```rust
// src/main.rs
mod models;
mod services;

use models::{User, Product};
use services::user_service::UserService;

fn main() {
    let mut service = UserService::new();
    
    service.add_user(User::new(1, "alice", "alice@example.com", "secret"));
    service.add_user(User::new(2, "bob", "bob@example.com", "password123"));
    
    if let Some(user) = service.authenticate("alice@example.com", "secret") {
        println!("Authenticated: {}", user.display_name());
    }
    
    if let Some(user) = service.find_by_id(2) {
        println!("Found: {:?}", user);
    }
    
    println!("Total users: {}", service.all_users().len());
}
```

---

## 6. Library Crate vs Binary Crate

```toml
# Cargo.toml สำหรับ library crate
[package]
name = "my_library"
version = "0.1.0"
edition = "2021"

[lib]
name = "my_library"
path = "src/lib.rs"

# สำหรับ binary crate ด้วยกัน
[[bin]]
name = "my_app"
path = "src/main.rs"
```

```rust
// src/lib.rs - Library crate entry point
pub mod math;
pub mod string_utils;
pub mod collections;

// Re-export common items
pub use math::Matrix;
pub use string_utils::StringExt;

/// สรุปฟีเจอร์ทั้งหมดของ library
pub fn version() -> &'static str {
    env!("CARGO_PKG_VERSION")
}
```

```rust
// src/math.rs (หรือ src/math/mod.rs)
pub struct Matrix {
    rows: usize,
    cols: usize,
    data: Vec<Vec<f64>>,
}

impl Matrix {
    pub fn new(rows: usize, cols: usize) -> Self {
        Matrix {
            rows,
            cols,
            data: vec![vec![0.0; cols]; rows],
        }
    }
    
    pub fn set(&mut self, row: usize, col: usize, val: f64) {
        self.data[row][col] = val;
    }
    
    pub fn get(&self, row: usize, col: usize) -> f64 {
        self.data[row][col]
    }
    
    pub fn multiply(&self, other: &Matrix) -> Option<Matrix> {
        if self.cols != other.rows {
            return None;
        }
        
        let mut result = Matrix::new(self.rows, other.cols);
        for i in 0..self.rows {
            for j in 0..other.cols {
                let mut sum = 0.0;
                for k in 0..self.cols {
                    sum += self.get(i, k) * other.get(k, j);
                }
                result.set(i, j, sum);
            }
        }
        Some(result)
    }
    
    pub fn transpose(&self) -> Matrix {
        let mut result = Matrix::new(self.cols, self.rows);
        for i in 0..self.rows {
            for j in 0..self.cols {
                result.set(j, i, self.get(i, j));
            }
        }
        result
    }
    
    pub fn display(&self) {
        for row in &self.data {
            let formatted: Vec<String> = row.iter()
                .map(|x| format!("{:6.2}", x))
                .collect();
            println!("[{}]", formatted.join(", "));
        }
    }
}

pub fn gcd(mut a: u64, mut b: u64) -> u64 {
    while b != 0 {
        let temp = b;
        b = a % b;
        a = temp;
    }
    a
}

pub fn lcm(a: u64, b: u64) -> u64 {
    a / gcd(a, b) * b
}

pub fn is_prime(n: u64) -> bool {
    if n < 2 { return false; }
    if n == 2 { return true; }
    if n % 2 == 0 { return false; }
    let sqrt = (n as f64).sqrt() as u64;
    (3..=sqrt).step_by(2).all(|i| n % i != 0)
}

pub fn primes_up_to(limit: u64) -> Vec<u64> {
    (2..=limit).filter(|&n| is_prime(n)).collect()
}
```

---

## 7. Cargo Workspace

Workspace ช่วยจัดการ multiple crates ที่เกี่ยวข้องกัน

```
my_workspace/
├── Cargo.toml          (workspace root)
├── shared/             (shared library)
│   ├── Cargo.toml
│   └── src/lib.rs
├── api_server/         (binary crate)
│   ├── Cargo.toml
│   └── src/main.rs
├── cli_tool/           (binary crate)
│   ├── Cargo.toml
│   └── src/main.rs
└── common_utils/       (utility library)
    ├── Cargo.toml
    └── src/lib.rs
```

```toml
# my_workspace/Cargo.toml
[workspace]
members = [
    "shared",
    "api_server",
    "cli_tool",
    "common_utils",
]

# Shared dependencies (Cargo 1.64+)
[workspace.dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
log = "0.4"
```

```toml
# my_workspace/shared/Cargo.toml
[package]
name = "shared"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { workspace = true }
```

```toml
# my_workspace/api_server/Cargo.toml
[package]
name = "api_server"
version = "0.1.0"
edition = "2021"

[dependencies]
shared = { path = "../shared" }
common_utils = { path = "../common_utils" }
tokio = { workspace = true }
```

```rust
// shared/src/lib.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ApiResponse<T> {
    pub success: bool,
    pub data: Option<T>,
    pub error: Option<String>,
}

impl<T> ApiResponse<T> {
    pub fn ok(data: T) -> Self {
        ApiResponse {
            success: true,
            data: Some(data),
            error: None,
        }
    }
    
    pub fn err(message: &str) -> Self {
        ApiResponse {
            success: false,
            data: None,
            error: Some(message.to_string()),
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct User {
    pub id: u32,
    pub name: String,
    pub email: String,
}
```

```rust
// common_utils/src/lib.rs
use std::time::{SystemTime, UNIX_EPOCH};

pub fn current_timestamp() -> u64 {
    SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_secs()
}

pub fn slugify(s: &str) -> String {
    s.to_lowercase()
        .chars()
        .map(|c| if c.is_alphanumeric() { c } else { '-' })
        .collect::<String>()
        .split('-')
        .filter(|s| !s.is_empty())
        .collect::<Vec<_>>()
        .join("-")
}

pub fn truncate(s: &str, max_len: usize) -> String {
    if s.len() <= max_len {
        s.to_string()
    } else {
        format!("{}...", &s[..max_len - 3])
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_slugify() {
        assert_eq!(slugify("Hello World!"), "hello-world");
        assert_eq!(slugify("Rust Programming"), "rust-programming");
    }
    
    #[test]
    fn test_truncate() {
        assert_eq!(truncate("Hello, World!", 10), "Hello, ...");
        assert_eq!(truncate("Hi", 10), "Hi");
    }
}
```

```rust
// api_server/src/main.rs
use shared::{ApiResponse, User};
use common_utils::{current_timestamp, slugify};

fn get_user(id: u32) -> ApiResponse<User> {
    if id == 1 {
        ApiResponse::ok(User {
            id: 1,
            name: "Alice".to_string(),
            email: "alice@example.com".to_string(),
        })
    } else {
        ApiResponse::err("User not found")
    }
}

fn main() {
    println!("API Server started at: {}", current_timestamp());
    
    let response = get_user(1);
    if response.success {
        if let Some(user) = response.data {
            println!("Found user: {} ({})", user.name, user.email);
            println!("Slug: {}", slugify(&user.name));
        }
    }
    
    let not_found = get_user(999);
    println!("Error: {:?}", not_found.error);
}
```

---

## 8. Popular Crates Overview

```toml
# Cargo.toml - commonly used crates
[dependencies]
# Serialization/Deserialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# Async runtime
tokio = { version = "1", features = ["full"] }

# HTTP client
reqwest = { version = "0.11", features = ["json"] }

# Web framework
actix-web = "4"

# Error handling
anyhow = "1.0"
thiserror = "1.0"

# Logging
log = "0.4"
env_logger = "0.10"
tracing = "0.1"
tracing-subscriber = "0.3"

# CLI argument parsing
clap = { version = "4", features = ["derive"] }

# Regular expressions
regex = "1"

# Date and time
chrono = { version = "0.4", features = ["serde"] }
time = "0.3"

# Random numbers
rand = "0.8"

# UUID
uuid = { version = "1", features = ["v4", "serde"] }

# Database
sqlx = { version = "0.7", features = ["runtime-tokio-native-tls", "postgres"] }

# Configuration
config = "0.13"
dotenv = "0.15"

# Iterators (extra)
itertools = "0.12"

# Parallelism
rayon = "1.8"
```

```rust
// ตัวอย่างการใช้ serde
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Config {
    host: String,
    port: u16,
    database_url: String,
    max_connections: u32,
}

fn main() {
    // Serialize to JSON
    let config = Config {
        host: "localhost".to_string(),
        port: 8080,
        database_url: "postgres://localhost/mydb".to_string(),
        max_connections: 10,
    };
    
    let json = serde_json::to_string_pretty(&config).unwrap();
    println!("Config JSON:\n{}", json);
    
    // Deserialize from JSON
    let json_str = r#"
    {
        "host": "0.0.0.0",
        "port": 3000,
        "database_url": "postgres://prod/db",
        "max_connections": 50
    }
    "#;
    
    let parsed: Config = serde_json::from_str(json_str).unwrap();
    println!("Parsed: {:?}", parsed);
    
    // Serialize Vec
    let items: Vec<i32> = vec![1, 2, 3, 4, 5];
    let items_json = serde_json::to_string(&items).unwrap();
    println!("Items: {}", items_json);
}
```

```rust
// ตัวอย่างการใช้ anyhow สำหรับ error handling
use anyhow::{Context, Result, anyhow};
use std::fs;
use std::path::Path;

fn read_config(path: &Path) -> Result<String> {
    fs::read_to_string(path)
        .with_context(|| format!("Failed to read config from {:?}", path))
}

fn parse_port(s: &str) -> Result<u16> {
    s.parse::<u16>()
        .map_err(|e| anyhow!("Invalid port '{}': {}", s, e))
}

fn setup_server(config_path: &Path) -> Result<()> {
    let content = read_config(config_path)?;
    println!("Config loaded: {} bytes", content.len());
    
    let port = parse_port("8080")?;
    println!("Server will listen on port {}", port);
    
    // Error case
    let bad_port = parse_port("not_a_number");
    match bad_port {
        Ok(p) => println!("Port: {}", p),
        Err(e) => println!("Error: {:#}", e),
    }
    
    Ok(())
}

fn main() {
    match setup_server(Path::new("config.toml")) {
        Ok(_) => println!("Server setup complete"),
        Err(e) => eprintln!("Setup failed: {:#}", e),
    }
}
```

---

## 9. Practical: Library Crate กับ Modules

```rust
// สร้าง library crate: text_processor
// src/lib.rs

pub mod tokenizer;
pub mod analyzer;
pub mod formatter;

pub use tokenizer::Tokenizer;
pub use analyzer::{Analyzer, TextStats};
pub use formatter::Formatter;

pub fn process_text(text: &str) -> TextStats {
    let tokenizer = Tokenizer::new();
    let tokens = tokenizer.tokenize(text);
    
    let analyzer = Analyzer::new();
    analyzer.analyze(&tokens)
}
```

```rust
// src/tokenizer.rs
pub struct Tokenizer {
    min_word_len: usize,
}

impl Tokenizer {
    pub fn new() -> Self {
        Tokenizer { min_word_len: 1 }
    }
    
    pub fn with_min_length(min_len: usize) -> Self {
        Tokenizer { min_word_len: min_len }
    }
    
    pub fn tokenize<'a>(&self, text: &'a str) -> Vec<&'a str> {
        text.split_whitespace()
            .filter(|word| word.len() >= self.min_word_len)
            .map(|word| {
                // Remove leading/trailing punctuation
                word.trim_matches(|c: char| !c.is_alphanumeric())
            })
            .filter(|word| !word.is_empty())
            .collect()
    }
    
    pub fn sentences(text: &str) -> Vec<&str> {
        text.split(|c| c == '.' || c == '!' || c == '?')
            .map(|s| s.trim())
            .filter(|s| !s.is_empty())
            .collect()
    }
}

impl Default for Tokenizer {
    fn default() -> Self {
        Self::new()
    }
}
```

```rust
// src/analyzer.rs
use std::collections::HashMap;

#[derive(Debug)]
pub struct TextStats {
    pub word_count: usize,
    pub unique_words: usize,
    pub avg_word_length: f64,
    pub most_common: Vec<(String, usize)>,
    pub longest_word: String,
}

pub struct Analyzer {
    top_n: usize,
}

impl Analyzer {
    pub fn new() -> Self {
        Analyzer { top_n: 10 }
    }
    
    pub fn with_top_n(n: usize) -> Self {
        Analyzer { top_n: n }
    }
    
    pub fn analyze(&self, tokens: &[&str]) -> TextStats {
        let word_count = tokens.len();
        
        let mut freq: HashMap<String, usize> = HashMap::new();
        for &word in tokens {
            *freq.entry(word.to_lowercase()).or_insert(0) += 1;
        }
        
        let unique_words = freq.len();
        
        let total_len: usize = tokens.iter().map(|w| w.len()).sum();
        let avg_word_length = if word_count > 0 {
            total_len as f64 / word_count as f64
        } else {
            0.0
        };
        
        let mut freq_vec: Vec<(String, usize)> = freq.into_iter().collect();
        freq_vec.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(&b.0)));
        
        let most_common: Vec<(String, usize)> = freq_vec.into_iter()
            .take(self.top_n)
            .collect();
        
        let longest_word = tokens.iter()
            .max_by_key(|w| w.len())
            .unwrap_or(&"")
            .to_string();
        
        TextStats {
            word_count,
            unique_words,
            avg_word_length,
            most_common,
            longest_word,
        }
    }
    
    pub fn readability_score(stats: &TextStats) -> f64 {
        // Simple Flesch-like score
        let ratio = stats.unique_words as f64 / stats.word_count.max(1) as f64;
        let length_penalty = (stats.avg_word_length - 4.0).max(0.0);
        (ratio * 100.0) - (length_penalty * 5.0)
    }
}

impl Default for Analyzer {
    fn default() -> Self {
        Self::new()
    }
}
```

```rust
// src/formatter.rs
use super::analyzer::TextStats;

pub struct Formatter {
    width: usize,
}

impl Formatter {
    pub fn new() -> Self {
        Formatter { width: 60 }
    }
    
    pub fn with_width(width: usize) -> Self {
        Formatter { width }
    }
    
    pub fn format_stats(&self, stats: &TextStats) -> String {
        let mut output = String::new();
        let separator = "=".repeat(self.width);
        
        output.push_str(&separator);
        output.push('\n');
        output.push_str("TEXT ANALYSIS REPORT\n");
        output.push_str(&separator);
        output.push('\n');
        
        output.push_str(&format!("Total words:      {}\n", stats.word_count));
        output.push_str(&format!("Unique words:     {}\n", stats.unique_words));
        output.push_str(&format!("Avg word length:  {:.2} chars\n", stats.avg_word_length));
        output.push_str(&format!("Longest word:     {}\n", stats.longest_word));
        
        output.push('\n');
        output.push_str("TOP WORDS:\n");
        output.push_str(&"-".repeat(30));
        output.push('\n');
        
        for (i, (word, count)) in stats.most_common.iter().enumerate() {
            let bar_len = (*count as f64 / stats.word_count as f64 * 20.0) as usize;
            let bar = "#".repeat(bar_len);
            output.push_str(&format!(
                "{:2}. {:15} {:4} |{}\n",
                i + 1, word, count, bar
            ));
        }
        
        output.push_str(&separator);
        output
    }
    
    pub fn word_wrap(&self, text: &str) -> String {
        let mut result = String::new();
        let mut line_len = 0;
        
        for word in text.split_whitespace() {
            if line_len + word.len() + 1 > self.width && line_len > 0 {
                result.push('\n');
                line_len = 0;
            } else if line_len > 0 {
                result.push(' ');
                line_len += 1;
            }
            
            result.push_str(word);
            line_len += word.len();
        }
        
        result
    }
}

impl Default for Formatter {
    fn default() -> Self {
        Self::new()
    }
}
```

```rust
// src/main.rs (หรือ examples/demo.rs)
// use text_processor::{process_text, Tokenizer, Analyzer, Formatter};

fn main() {
    let text = "The quick brown fox jumps over the lazy dog. \
                The fox was very quick and the dog was very lazy. \
                Quick brown foxes are known for jumping over lazy dogs. \
                This is a classic pangram used in typography.";
    
    let tokenizer = crate::tokenizer::Tokenizer::with_min_length(3);
    let tokens = tokenizer.tokenize(text);
    
    let analyzer = crate::analyzer::Analyzer::with_top_n(5);
    let stats = analyzer.analyze(&tokens);
    
    let formatter = crate::formatter::Formatter::with_width(50);
    println!("{}", formatter.format_stats(&stats));
    
    let score = crate::analyzer::Analyzer::readability_score(&stats);
    println!("Readability score: {:.1}", score);
    
    println!("\nSentences:");
    for (i, sent) in crate::tokenizer::Tokenizer::sentences(text).iter().enumerate() {
        println!("  {}. {}", i + 1, sent);
    }
    
    println!("\nWrapped text:");
    println!("{}", formatter.word_wrap(text));
}
```

---

## 10. สรุป

| Concept | Description | ตัวอย่าง |
|---------|-------------|---------|
| `mod` | กำหนด module | `mod math { ... }` |
| `pub` | public visibility | `pub fn foo()` |
| `pub(crate)` | visible ใน crate | `pub(crate) fn bar()` |
| `pub(super)` | visible ใน parent | `pub(super) fn baz()` |
| `use` | import path | `use std::collections::HashMap` |
| `use ... as` | rename import | `use std::io::Result as IoResult` |
| `pub use` | re-export | `pub use module::Type` |
| File module | แยกไฟล์ | `mod foo;` → `src/foo.rs` |
| `lib.rs` | library entry point | `pub mod ...` |
| `main.rs` | binary entry point | `fn main()` |
| Workspace | หลาย crates | `[workspace] members = [...]` |

### Best Practices

1. **ใช้ modules** เพื่อจัดระเบียบ code
2. **ปิด private โดย default** - เปิดเฉพาะสิ่งที่จำเป็น
3. **Re-export** items ที่ใช้บ่อยใน `lib.rs` หรือ `mod.rs`
4. **Workspace** สำหรับ large projects ที่มีหลาย crates
5. **semantic naming** - ชื่อ module ควรสะท้อน responsibility

---

*[← Part 011: Closures และ Iterators](../part_011/README.md) | [Part 013: Testing ใน Rust →](../part_013/README.md)*
