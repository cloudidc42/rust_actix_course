# Part 020: Procedural Macros and derive

## บทนำ

Macros ใน Rust เป็นเครื่องมือ metaprogramming ที่ทรงพลัง ช่วยให้เราเขียน code ที่ generate code อื่น ลด boilerplate และสร้าง DSL (Domain Specific Language) ในบทนี้เราจะเรียนรู้ทั้ง declarative macros (`macro_rules!`) และ procedural macros

---

## 1. Declarative Macros (macro_rules!)

Declarative macros ใช้ pattern matching เพื่อ transform code

### พื้นฐาน macro_rules!

```rust
// macro_rules! ง่ายๆ
macro_rules! say_hello {
    () => {
        println!("Hello, World!");
    };
    ($name:expr) => {
        println!("Hello, {}!", $name);
    };
}

// macro ที่รับหลาย arguments
macro_rules! print_all {
    ($($arg:expr),*) => {
        $(
            println!("{}", $arg);
        )*
    };
}

// macro ที่สร้าง struct
macro_rules! make_point {
    ($x:expr, $y:expr) => {
        Point { x: $x, y: $y }
    };
}

#[derive(Debug)]
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    say_hello!();
    say_hello!("Rust");
    
    print_all!("one", "two", "three");
    print_all!(1, 2, 3, 4, 5);
    
    let p = make_point!(3.0, 4.0);
    println!("{:?}", p);
}
```

### Macro Designators (Fragment Specifiers)

```rust
macro_rules! demonstrate_types {
    // expr - expression
    (expr $e:expr) => {
        println!("expr: {}", $e);
    };
    // ident - identifier
    (ident $i:ident) => {
        let $i = 42;
        println!("ident: {}", $i);
    };
    // ty - type
    (ty $t:ty) => {
        println!("type size: {}", std::mem::size_of::<$t>());
    };
    // block - block of code
    (block $b:block) => {
        let result = $b;
        println!("block result: {}", result);
    };
    // stmt - statement
    (stmt $s:stmt) => {
        $s;
    };
    // literal
    (lit $l:literal) => {
        println!("literal: {}", $l);
    };
    // pattern
    (pat $p:pat => $e:expr) => {
        // pattern matching example
        let val = 5;
        match val {
            $p => println!("matched: {}", $e),
            _ => println!("no match"),
        }
    };
    // path
    (path $p:path) => {
        println!("path used");
        let _ = $p::default();
    };
    // meta - meta attribute
    (meta $m:meta) => {
        #[$m]
        fn tagged_fn() {}
    };
    // tt - token tree (catch-all)
    (tt $($t:tt)*) => {
        println!("tokens: {}", stringify!($($t)*));
    };
}

fn main() {
    demonstrate_types!(expr 1 + 2 * 3);
    demonstrate_types!(ident my_var);
    demonstrate_types!(ty u64);
    demonstrate_types!(block { 10 + 20 });
    demonstrate_types!(lit 42);
}
```

### Recursive Macros

```rust
// สร้าง HashMap ด้วย macro
macro_rules! map {
    () => {
        ::std::collections::HashMap::new()
    };
    ($($key:expr => $value:expr),+ $(,)?) => {
        {
            let mut m = ::std::collections::HashMap::new();
            $(
                m.insert($key, $value);
            )+
            m
        }
    };
}

// สร้าง Vec ด้วย macro (คล้าย vec!)
macro_rules! make_vec {
    () => { Vec::new() };
    ($elem:expr; $n:expr) => {
        vec![$elem; $n]
    };
    ($($x:expr),+ $(,)?) => {
        {
            let mut v = Vec::new();
            $(v.push($x);)+
            v
        }
    };
}

// Recursive macro สำหรับ nested calls
macro_rules! nested_ops {
    ($val:expr) => { $val };
    ($val:expr, add $x:expr $(, $rest:tt)*) => {
        nested_ops!($val + $x $(, $rest)*)
    };
    ($val:expr, mul $x:expr $(, $rest:tt)*) => {
        nested_ops!($val * $x $(, $rest)*)
    };
}

fn main() {
    let m = map! {
        "one" => 1,
        "two" => 2,
        "three" => 3,
    };
    println!("{:?}", m);
    
    let v = make_vec![1, 2, 3, 4, 5];
    println!("{:?}", v);
    
    let result = nested_ops!(10, add 5, mul 2, add 3);
    println!("Result: {}", result); // (10 + 5) * 2 + 3 = 33
}
```

---

## 2. Common Derive Traits

### Debug

```rust
#[derive(Debug)]
struct Server {
    host: String,
    port: u16,
    max_connections: u32,
}

#[derive(Debug)]
enum Status {
    Running,
    Stopped,
    Error(String),
}

fn main() {
    let server = Server {
        host: "localhost".to_string(),
        port: 8080,
        max_connections: 100,
    };
    
    // {:?} - debug format
    println!("{:?}", server);
    
    // {:#?} - pretty debug format
    println!("{:#?}", server);
    
    let status = Status::Error("connection refused".to_string());
    println!("{:?}", status);
    
    // dbg! macro - prints and returns the value
    let x = dbg!(2 + 3);
    println!("x = {}", x);
}
```

### Clone และ Copy

```rust
#[derive(Debug, Clone)]
struct Config {
    host: String,     // String ไม่ impl Copy ดังนั้น Config ก็ Clone เท่านั้น
    port: u16,
    debug: bool,
}

#[derive(Debug, Clone, Copy)]
struct Point {
    x: f64,   // f64 impl Copy
    y: f64,
}

fn main() {
    let config1 = Config {
        host: "localhost".to_string(),
        port: 8080,
        debug: true,
    };
    
    // Clone: explicit deep copy
    let config2 = config1.clone();
    println!("{:?}", config1); // config1 ยังใช้ได้
    println!("{:?}", config2);
    
    // Copy: implicit bitwise copy
    let p1 = Point { x: 1.0, y: 2.0 };
    let p2 = p1;  // copy, ไม่ใช่ move
    println!("{:?}", p1); // p1 ยังใช้ได้
    println!("{:?}", p2);
}
```

### PartialEq, Eq, PartialOrd, Ord

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
struct Version {
    major: u32,
    minor: u32,
    patch: u32,
}

#[derive(Debug, Clone, PartialEq, PartialOrd)]
struct Temperature(f64);

// Custom PartialOrd สำหรับ Version
impl PartialOrd for Version {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        Some(self.cmp(other))
    }
}

impl Ord for Version {
    fn cmp(&self, other: &Self) -> std::cmp::Ordering {
        self.major.cmp(&other.major)
            .then(self.minor.cmp(&other.minor))
            .then(self.patch.cmp(&other.patch))
    }
}

fn main() {
    let v1 = Version { major: 1, minor: 2, patch: 3 };
    let v2 = Version { major: 1, minor: 2, patch: 4 };
    let v3 = v1.clone();
    
    println!("v1 == v3: {}", v1 == v3);
    println!("v1 != v2: {}", v1 != v2);
    println!("v1 < v2: {}", v1 < v2);
    println!("v2 > v1: {}", v2 > v1);
    
    let mut versions = vec![v2.clone(), v1.clone(), v3.clone()];
    versions.sort();
    println!("Sorted: {:?}", versions);
    
    let t1 = Temperature(100.0);
    let t2 = Temperature(200.0);
    println!("t1 < t2: {}", t1 < t2);
}
```

### Hash

```rust
use std::collections::{HashMap, HashSet};

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct UserId {
    prefix: String,
    number: u64,
}

fn main() {
    let id1 = UserId { prefix: "USR".to_string(), number: 1001 };
    let id2 = UserId { prefix: "USR".to_string(), number: 1002 };
    let id3 = id1.clone();
    
    // ใช้ใน HashMap
    let mut users: HashMap<UserId, String> = HashMap::new();
    users.insert(id1.clone(), "Alice".to_string());
    users.insert(id2.clone(), "Bob".to_string());
    
    println!("{:?}", users.get(&id1));
    println!("{:?}", users.get(&id3)); // id3 == id1
    
    // ใช้ใน HashSet
    let mut id_set: HashSet<UserId> = HashSet::new();
    id_set.insert(id1.clone());
    id_set.insert(id2.clone());
    id_set.insert(id3); // duplicate ของ id1
    
    println!("Set size: {}", id_set.len()); // 2 ไม่ใช่ 3
}
```

### Default

```rust
#[derive(Debug, Default)]
struct AppConfig {
    host: String,        // Default: ""
    port: u16,          // Default: 0
    max_connections: u32, // Default: 0
    debug: bool,         // Default: false
    timeout_ms: Option<u64>, // Default: None
}

// Custom Default implementation
#[derive(Debug)]
struct ServerConfig {
    host: String,
    port: u16,
    workers: usize,
}

impl Default for ServerConfig {
    fn default() -> Self {
        ServerConfig {
            host: "127.0.0.1".to_string(),
            port: 8080,
            workers: num_cpus(), // sensible default
        }
    }
}

fn num_cpus() -> usize {
    4 // simplified
}

fn main() {
    // derive Default
    let config = AppConfig::default();
    println!("{:?}", config);
    
    // ..Default::default() pattern (struct update syntax)
    let custom = AppConfig {
        host: "example.com".to_string(),
        port: 443,
        ..Default::default()
    };
    println!("{:?}", custom);
    
    // Custom Default
    let server = ServerConfig::default();
    println!("{:?}", server);
    
    // Default with Option
    let opt: Option<i32> = Default::default(); // None
    println!("Default Option: {:?}", opt);
    
    let vec: Vec<i32> = Default::default(); // empty vec
    println!("Default Vec: {:?}", vec);
}
```

---

## 3. Serde Derive (Serialize, Deserialize)

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
use serde::{Deserialize, Serialize};
use serde_json;

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: u64,
    name: String,
    email: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    phone: Option<String>,
    #[serde(default)]
    active: bool,
}

#[derive(Debug, Serialize, Deserialize)]
struct ApiResponse<T> {
    success: bool,
    #[serde(skip_serializing_if = "Option::is_none")]
    data: Option<T>,
    #[serde(skip_serializing_if = "Option::is_none")]
    error: Option<String>,
    #[serde(default = "default_version")]
    version: String,
}

fn default_version() -> String {
    "1.0".to_string()
}

// Rename fields
#[derive(Debug, Serialize, Deserialize)]
struct DbRecord {
    #[serde(rename = "user_id")]
    id: u64,
    #[serde(rename = "full_name")]
    name: String,
    #[serde(rename = "created_at")]
    created: String,
}

// Custom serialization
use serde::{Serializer, Deserializer};

#[derive(Debug)]
struct ColorHex(u32);

impl Serialize for ColorHex {
    fn serialize<S: Serializer>(&self, serializer: S) -> Result<S::Ok, S::Error> {
        serializer.serialize_str(&format!("#{:06X}", self.0))
    }
}

impl<'de> Deserialize<'de> for ColorHex {
    fn deserialize<D: Deserializer<'de>>(deserializer: D) -> Result<Self, D::Error> {
        let s = String::deserialize(deserializer)?;
        let hex = s.trim_start_matches('#');
        u32::from_str_radix(hex, 16)
            .map(ColorHex)
            .map_err(serde::de::Error::custom)
    }
}

fn main() {
    // Serialize
    let user = User {
        id: 1,
        name: "Alice".to_string(),
        email: "alice@example.com".to_string(),
        phone: Some("0812345678".to_string()),
        active: true,
    };
    
    let json = serde_json::to_string_pretty(&user).unwrap();
    println!("Serialized:\n{}", json);
    
    // Deserialize
    let json_input = r#"{
        "id": 2,
        "name": "Bob",
        "email": "bob@example.com",
        "active": false
    }"#;
    
    let parsed: User = serde_json::from_str(json_input).unwrap();
    println!("\nDeserialized: {:?}", parsed);
    
    // ApiResponse
    let response = ApiResponse {
        success: true,
        data: Some(user),
        error: None,
        version: "2.0".to_string(),
    };
    println!("\nResponse:\n{}", serde_json::to_string_pretty(&response).unwrap());
    
    // ColorHex
    let color = ColorHex(0xFF5733);
    println!("\nColor: {}", serde_json::to_string(&color).unwrap());
    
    let parsed_color: ColorHex = serde_json::from_str(r#""#FF5733""#).unwrap();
    println!("Parsed color: {}", parsed_color.0);
}
```

---

## 4. thiserror Derive (Error)

```toml
[dependencies]
thiserror = "1"
```

```rust
use thiserror::Error;
use std::num::ParseIntError;
use std::io;

// Error enum ด้วย thiserror
#[derive(Debug, Error)]
enum DatabaseError {
    #[error("Connection failed: {host}:{port}")]
    ConnectionFailed { host: String, port: u16 },
    
    #[error("Query failed: {0}")]
    QueryFailed(String),
    
    #[error("Record not found: id={id}")]
    NotFound { id: u64 },
    
    #[error("IO error")]
    IoError(#[from] io::Error),
    
    #[error("Parse error")]
    ParseError(#[from] ParseIntError),
    
    #[error("Unknown error")]
    Unknown,
}

// Nested errors
#[derive(Debug, Error)]
enum ApiError {
    #[error("Database error: {0}")]
    Database(#[from] DatabaseError),
    
    #[error("Authentication failed")]
    Unauthorized,
    
    #[error("Validation error: {field} - {message}")]
    Validation { field: String, message: String },
    
    #[error("Rate limit exceeded: {limit} requests per {window}s")]
    RateLimit { limit: u32, window: u32 },
}

fn get_user(id: u64) -> Result<String, DatabaseError> {
    if id == 0 {
        return Err(DatabaseError::NotFound { id });
    }
    if id > 1000 {
        return Err(DatabaseError::QueryFailed("ID too large".to_string()));
    }
    Ok(format!("User {}", id))
}

fn api_get_user(id: u64) -> Result<String, ApiError> {
    if id == 999 {
        return Err(ApiError::Unauthorized);
    }
    
    // DatabaseError automatically converts to ApiError via #[from]
    let user = get_user(id)?;
    Ok(user)
}

fn main() {
    // Test various errors
    let test_ids = [1u64, 0, 2000, 999];
    
    for id in &test_ids {
        match api_get_user(*id) {
            Ok(user) => println!("OK: {}", user),
            Err(e) => println!("Error: {}", e),
        }
    }
    
    // Error chain
    let db_err = DatabaseError::ConnectionFailed {
        host: "localhost".to_string(),
        port: 5432,
    };
    
    let api_err: ApiError = db_err.into(); // DatabaseError -> ApiError
    println!("\nConverted error: {}", api_err);
    
    // Validation error
    let validation = ApiError::Validation {
        field: "email".to_string(),
        message: "Invalid format".to_string(),
    };
    println!("Validation: {}", validation);
}
```

---

## 5. Procedural Macros - Custom Derive

Procedural macros ต้องอยู่ใน crate แยกต่างหากที่มี `proc-macro = true`

### Setup

```toml
# my-derive/Cargo.toml
[package]
name = "my-derive"
version = "0.1.0"
edition = "2021"

[lib]
proc-macro = true

[dependencies]
syn = { version = "2", features = ["full"] }
quote = "1"
proc-macro2 = "1"
```

### Custom Derive Macro

```rust
// my-derive/src/lib.rs
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput, Data, Fields};

/// Custom derive สำหรับ Describe trait
#[proc_macro_derive(Describe)]
pub fn derive_describe(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;
    
    // ดึง field names
    let field_names = match &input.data {
        Data::Struct(data) => {
            match &data.fields {
                Fields::Named(fields) => {
                    fields.named.iter()
                        .map(|f| f.ident.as_ref().unwrap().to_string())
                        .collect::<Vec<_>>()
                }
                _ => vec![],
            }
        }
        _ => vec![],
    };
    
    let description = format!(
        "Struct '{}' with fields: {}",
        name,
        field_names.join(", ")
    );
    
    let expanded = quote! {
        impl Describe for #name {
            fn describe(&self) -> String {
                #description.to_string()
            }
            
            fn type_name() -> &'static str {
                stringify!(#name)
            }
        }
    };
    
    TokenStream::from(expanded)
}

/// Custom derive พร้อม attributes
#[proc_macro_derive(Builder, attributes(builder))]
pub fn derive_builder(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;
    let builder_name = syn::Ident::new(&format!("{}Builder", name), name.span());
    
    // Generate builder struct and methods
    // (simplified implementation)
    let expanded = quote! {
        pub struct #builder_name {
            // builder fields would be generated here
        }
        
        impl #name {
            pub fn builder() -> #builder_name {
                #builder_name {}
            }
        }
    };
    
    TokenStream::from(expanded)
}
```

### ใช้ Custom Derive

```rust
// src/main.rs
use my_derive::Describe;

trait Describe {
    fn describe(&self) -> String;
    fn type_name() -> &'static str where Self: Sized;
}

#[derive(Debug, Describe)]
struct User {
    id: u64,
    name: String,
    email: String,
}

#[derive(Debug, Describe)]
struct Product {
    id: u64,
    name: String,
    price: f64,
    stock: u32,
}

fn main() {
    let user = User {
        id: 1,
        name: "Alice".to_string(),
        email: "alice@example.com".to_string(),
    };
    
    println!("{}", user.describe());
    println!("Type: {}", User::type_name());
    
    let product = Product {
        id: 101,
        name: "Rust Book".to_string(),
        price: 29.99,
        stock: 50,
    };
    
    println!("{}", product.describe());
}
```

---

## 6. Attribute Macros

```rust
// my-derive/src/lib.rs (เพิ่ม attribute macro)
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, ItemFn};

/// Attribute macro สำหรับ timing functions
#[proc_macro_attribute]
pub fn timed(_attr: TokenStream, item: TokenStream) -> TokenStream {
    let input = parse_macro_input!(item as ItemFn);
    let name = &input.sig.ident;
    let name_str = name.to_string();
    let block = &input.block;
    let sig = &input.sig;
    let vis = &input.vis;
    
    let expanded = quote! {
        #vis #sig {
            let start = std::time::Instant::now();
            let result = (|| #block)();
            let duration = start.elapsed();
            println!("Function '{}' took {:?}", #name_str, duration);
            result
        }
    };
    
    TokenStream::from(expanded)
}

/// Attribute macro สำหรับ retry logic
#[proc_macro_attribute]
pub fn retry(attr: TokenStream, item: TokenStream) -> TokenStream {
    // Parse max_retries จาก attribute
    let max_retries = if attr.is_empty() {
        3usize
    } else {
        // parse "max_retries = N" syntax
        3
    };
    
    let input = parse_macro_input!(item as ItemFn);
    let sig = &input.sig;
    let block = &input.block;
    let vis = &input.vis;
    
    let expanded = quote! {
        #vis #sig {
            let mut attempts = 0;
            loop {
                let result = (|| #block)();
                match result {
                    Ok(v) => return Ok(v),
                    Err(e) => {
                        attempts += 1;
                        if attempts >= #max_retries {
                            return Err(e);
                        }
                        println!("Retry attempt {}/{}", attempts, #max_retries);
                    }
                }
            }
        }
    };
    
    TokenStream::from(expanded)
}
```

```rust
// src/main.rs - ใช้ attribute macros
use my_derive::{timed, retry};

#[timed]
fn expensive_calculation(n: u64) -> u64 {
    (0..n).sum()
}

#[retry]
fn flaky_operation(attempt: &mut u32) -> Result<String, String> {
    *attempt += 1;
    if *attempt < 3 {
        Err("temporary failure".to_string())
    } else {
        Ok("success!".to_string())
    }
}

fn main() {
    let result = expensive_calculation(1_000_000);
    println!("Result: {}", result);
    
    let mut attempt = 0;
    match flaky_operation(&mut attempt) {
        Ok(s) => println!("Got: {}", s),
        Err(e) => println!("Failed: {}", e),
    }
}
```

---

## 7. Function-like Macros

```rust
// my-derive/src/lib.rs
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, LitStr};

/// Function-like macro สำหรับ SQL queries
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {
    let query = parse_macro_input!(input as LitStr);
    let query_str = query.value();
    
    // Validate query at compile time (simplified)
    if !query_str.trim_start().to_uppercase().starts_with("SELECT") &&
       !query_str.trim_start().to_uppercase().starts_with("INSERT") &&
       !query_str.trim_start().to_uppercase().starts_with("UPDATE") &&
       !query_str.trim_start().to_uppercase().starts_with("DELETE") {
        panic!("Invalid SQL: must start with SELECT, INSERT, UPDATE, or DELETE");
    }
    
    let expanded = quote! {
        {
            let query: &'static str = #query_str;
            // In real usage, would return a PreparedStatement
            query
        }
    };
    
    TokenStream::from(expanded)
}

/// Function-like macro สำหรับ HTML generation (simplified)
#[proc_macro]
pub fn html(input: TokenStream) -> TokenStream {
    // ในการใช้งานจริงจะ parse HTML-like syntax
    let tokens = input.to_string();
    
    let expanded = quote! {
        {
            // Generated HTML string
            String::from(#tokens)
        }
    };
    
    TokenStream::from(expanded)
}
```

---

## 8. Practical: Custom Derive for Builder Pattern

### Builder Macro สมบูรณ์

```rust
// ตัวอย่าง Builder pattern แบบ manual (ใช้อ้างอิง)
#[derive(Debug)]
struct Server {
    host: String,
    port: u16,
    max_connections: u32,
    timeout_seconds: u64,
    tls_enabled: bool,
    auth_required: bool,
}

// Builder struct
struct ServerBuilder {
    host: Option<String>,
    port: Option<u16>,
    max_connections: Option<u32>,
    timeout_seconds: Option<u64>,
    tls_enabled: Option<bool>,
    auth_required: Option<bool>,
}

impl ServerBuilder {
    fn new() -> Self {
        ServerBuilder {
            host: None,
            port: None,
            max_connections: None,
            timeout_seconds: None,
            tls_enabled: None,
            auth_required: None,
        }
    }
    
    fn host(mut self, host: impl Into<String>) -> Self {
        self.host = Some(host.into());
        self
    }
    
    fn port(mut self, port: u16) -> Self {
        self.port = Some(port);
        self
    }
    
    fn max_connections(mut self, max: u32) -> Self {
        self.max_connections = Some(max);
        self
    }
    
    fn timeout_seconds(mut self, timeout: u64) -> Self {
        self.timeout_seconds = Some(timeout);
        self
    }
    
    fn tls_enabled(mut self, enabled: bool) -> Self {
        self.tls_enabled = Some(enabled);
        self
    }
    
    fn auth_required(mut self, required: bool) -> Self {
        self.auth_required = Some(required);
        self
    }
    
    fn build(self) -> Result<Server, String> {
        Ok(Server {
            host: self.host.ok_or("host is required")?,
            port: self.port.ok_or("port is required")?,
            max_connections: self.max_connections.unwrap_or(100),
            timeout_seconds: self.timeout_seconds.unwrap_or(30),
            tls_enabled: self.tls_enabled.unwrap_or(false),
            auth_required: self.auth_required.unwrap_or(true),
        })
    }
}

impl Server {
    fn builder() -> ServerBuilder {
        ServerBuilder::new()
    }
}
```

### Macro ที่ Generate Builder Pattern

```rust
// macro_rules! สำหรับ builder pattern
macro_rules! builder {
    (
        $(#[$struct_meta:meta])*
        pub struct $name:ident {
            $(
                $(#[$field_meta:meta])*
                $field:ident: $type:ty $(= $default:expr)?
            ),* $(,)?
        }
    ) => {
        $(#[$struct_meta])*
        pub struct $name {
            $($field: $type),*
        }
        
        paste::paste! {
            pub struct [<$name Builder>] {
                $($field: Option<$type>),*
            }
            
            impl [<$name Builder>] {
                pub fn new() -> Self {
                    [<$name Builder>] {
                        $($field: None),*
                    }
                }
                
                $(
                    pub fn $field(mut self, value: $type) -> Self {
                        self.$field = Some(value);
                        self
                    }
                )*
                
                pub fn build(self) -> Result<$name, String> {
                    Ok($name {
                        $(
                            $field: match self.$field {
                                Some(v) => v,
                                None => builder!(@default $($default)? , $field),
                            }
                        ),*
                    })
                }
            }
            
            impl $name {
                pub fn builder() -> [<$name Builder>] {
                    [<$name Builder>]::new()
                }
            }
        }
    };
    
    (@default $default:expr, $field:ident) => { $default };
    (@default , $field:ident) => {
        return Err(format!("Field '{}' is required", stringify!($field)))
    };
}

fn main() {
    // ใช้ Builder pattern แบบ manual
    let server = Server::builder()
        .host("0.0.0.0")
        .port(8443)
        .max_connections(500)
        .timeout_seconds(60)
        .tls_enabled(true)
        .auth_required(true)
        .build()
        .expect("Failed to build server config");
    
    println!("{:#?}", server);
    
    // Missing required field
    match Server::builder().port(8080).build() {
        Ok(s) => println!("Server: {:?}", s),
        Err(e) => println!("Error: {}", e),
    }
}
```

---

## 9. Macro Best Practices

```rust
// 1. ใช้ $crate:: เพื่อ reference items จาก current crate
macro_rules! create_error {
    ($msg:expr) => {
        $crate::MyError::new($msg) // ถูกต้อง: explicit path
    };
}

// 2. ใช้ $(,)? เพื่อ optional trailing comma
macro_rules! my_vec {
    ($($elem:expr),* $(,)?) => {  // $(,)? รองรับ trailing comma
        vec![$($elem),*]
    };
}

// 3. Hygiene - macros ไม่ควรให้ตัวแปรใน macro รั่วออกมา
macro_rules! swap {
    ($a:expr, $b:expr) => {
        {
            // temp อยู่ใน block scope ไม่รั่วออกมา
            let temp = $a;
            $a = $b;
            $b = temp;
        }
    };
}

// 4. ใช้ stringify! สำหรับ debug
macro_rules! debug_expr {
    ($e:expr) => {
        println!("{} = {:?}", stringify!($e), $e)
    };
}

// 5. ใช้ concat! สำหรับ string literals
macro_rules! api_endpoint {
    ($path:expr) => {
        concat!("/api/v1/", $path)
    };
}

fn main() {
    // my_vec with trailing comma
    let v = my_vec![1, 2, 3,]; // ได้
    println!("{:?}", v);
    
    // swap
    let mut a = 5;
    let mut b = 10;
    swap!(a, b);
    println!("a={}, b={}", a, b);
    
    // debug_expr
    let x = 42;
    debug_expr!(x * 2 + 1); // prints: x * 2 + 1 = 85
    
    // api_endpoint
    const USERS_PATH: &str = api_endpoint!("users");
    println!("Path: {}", USERS_PATH);
}
```

---

## 10. Macro Debugging

```bash
# ดู macro expansion
cargo expand
cargo expand --item my_function

# RUST_LOG สำหรับ proc macro debugging
RUST_LOG=proc_macro_derive=debug cargo build

# ใช้ eprintln! ใน proc macro (แสดงใน stderr ระหว่าง compilation)
```

```rust
// ใน proc macro crate
#[proc_macro_derive(Debug2)]
pub fn derive_debug2(input: TokenStream) -> TokenStream {
    // Debug proc macro itself
    eprintln!("Deriving Debug2 for: {}", input.to_string());
    
    // ...
    TokenStream::new()
}
```

---

## สรุป

| Type | Declaration | Use Case |
|------|------------|----------|
| Declarative | `macro_rules!` | Pattern-based code generation |
| Custom Derive | `#[proc_macro_derive]` | Add behavior to types with `#[derive]` |
| Attribute | `#[proc_macro_attribute]` | Transform functions/types |
| Function-like | `#[proc_macro]` | Custom syntax |

### Common Derive Traits

| Trait | Purpose | Requires |
|-------|---------|---------|
| `Debug` | fmt with `{:?}` | - |
| `Clone` | `.clone()` deep copy | All fields: Clone |
| `Copy` | implicit copy | All fields: Copy |
| `PartialEq`/`Eq` | `==`, `!=` | All fields: PartialEq |
| `PartialOrd`/`Ord` | `<`, `>`, `<=`, `>=` | All fields: PartialOrd |
| `Hash` | use in HashMap/HashSet | All fields: Hash |
| `Default` | Default values | All fields: Default |
| `Serialize`/`Deserialize` | JSON/etc encoding | serde crate |
| `Error` | Error type | thiserror crate |

---

## Navigation

- [← Part 019: String Handling Deep Dive](../part_019/README.md)
- [→ Part 021: ](../part_021/README.md)
- [กลับหน้าหลัก](../../README.md)
