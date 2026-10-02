# Part 008: Error Handling แบบมืออาชีพ ⚠️

## 🎯 เป้าหมายของ Part นี้

- Error handling patterns ระดับ production
- anyhow และ thiserror crates
- Error propagation แบบ idiomatic
- Logging errors อย่างถูกต้อง

---

## 1. Error Handling Approaches

### 1.1 เปรียบเทียบ Approaches

```
❌ Panic (ใช้เฉพาะ bugs, ไม่ใช่ error ปกติ)
panic!("Something went wrong");
unwrap() / expect()

❌ Return bool/int (ไม่รู้ว่าเกิดอะไร)
fn process() -> bool { ... }

✅ Result<T, E> (recommended)
fn process() -> Result<Data, Error> { ... }

✅ ? operator (clean propagation)
let data = process()?;

✅ anyhow (สำหรับ applications)
fn main() -> anyhow::Result<()> { ... }

✅ thiserror (สำหรับ libraries)
#[derive(thiserror::Error, Debug)]
enum MyError { ... }
```

### 1.2 เมื่อใดควรใช้อะไร

```
panic! / unwrap:
- Programming bugs (ไม่ควรเกิดขึ้น)
- Prototype code
- Tests

Option<T>:
- ค่าอาจไม่มี แต่ไม่ใช่ error
- ค้นหาใน collection

Result<T, E>:
- Operation อาจล้มเหลวได้
- I/O, network, parsing

anyhow:
- Application code (main, handlers)
- ต้องการ context ง่ายๆ

thiserror:
- Library code
- ต้องการ typed errors
```

---

## 2. thiserror (Library Errors)

```rust
// Cargo.toml:
// thiserror = "1"
// serde = { version = "1", features = ["derive"] }

use thiserror::Error;
use std::path::PathBuf;

#[derive(Debug, Error)]
pub enum DatabaseError {
    #[error("Connection failed: {host}:{port}")]
    ConnectionFailed { host: String, port: u16 },

    #[error("Query failed: {query}")]
    QueryFailed { query: String, #[source] source: std::io::Error },

    #[error("Record not found: id={id}")]
    NotFound { id: u64 },

    #[error("Duplicate entry for key: {key}")]
    DuplicateKey { key: String },

    #[error(transparent)]
    Io(#[from] std::io::Error),
}

#[derive(Debug, Error)]
pub enum AppError {
    #[error("Database error: {0}")]
    Database(#[from] DatabaseError),

    #[error("Validation error: {field} - {message}")]
    Validation { field: String, message: String },

    #[error("Authentication failed")]
    AuthFailed,

    #[error("Not found: {resource}")]
    NotFound { resource: String },

    #[error("Internal error: {0}")]
    Internal(String),
}

impl AppError {
    pub fn is_not_found(&self) -> bool {
        matches!(self, AppError::NotFound { .. })
    }

    pub fn is_auth_error(&self) -> bool {
        matches!(self, AppError::AuthFailed)
    }

    pub fn status_code(&self) -> u16 {
        match self {
            AppError::NotFound { .. }         => 404,
            AppError::AuthFailed              => 401,
            AppError::Validation { .. }       => 400,
            AppError::Database(DatabaseError::DuplicateKey { .. }) => 409,
            _                                 => 500,
        }
    }
}

fn find_user(id: u64) -> Result<String, AppError> {
    if id == 0 {
        return Err(AppError::NotFound { resource: format!("User {}", id) });
    }
    if id == 999 {
        return Err(AppError::Database(DatabaseError::NotFound { id }));
    }
    Ok(format!("User_{}", id))
}

fn authenticate(token: &str) -> Result<u64, AppError> {
    if token == "valid_token" {
        Ok(42)
    } else {
        Err(AppError::AuthFailed)
    }
}

fn get_user_profile(token: &str) -> Result<String, AppError> {
    let user_id = authenticate(token)?;  // ? propagates AppError
    let user = find_user(user_id)?;
    Ok(format!("Profile: {}", user))
}

fn main() {
    let test_cases = [
        ("valid_token", "Success"),
        ("bad_token", "Auth fail"),
    ];

    for (token, desc) in &test_cases {
        println!("--- {} ---", desc);
        match get_user_profile(token) {
            Ok(profile) => println!("OK: {}", profile),
            Err(e) => {
                println!("Error [{}]: {}", e.status_code(), e);
                println!("Is not found: {}", e.is_not_found());
                println!("Is auth error: {}", e.is_auth_error());
            }
        }
    }
}
```

---

## 3. anyhow (Application Errors)

```rust
// Cargo.toml:
// anyhow = "1"

use anyhow::{anyhow, bail, ensure, Context, Result};

fn read_config(path: &str) -> Result<String> {
    std::fs::read_to_string(path)
        .with_context(|| format!("Failed to read config file: {}", path))
}

fn parse_port(s: &str) -> Result<u16> {
    let port: u16 = s.parse()
        .with_context(|| format!("'{}' is not a valid port number", s))?;

    ensure!(port >= 1024, "Port {} is reserved (must be >= 1024)", port);

    Ok(port)
}

fn validate_email(email: &str) -> Result<()> {
    if !email.contains('@') {
        bail!("Invalid email: '{}' (missing @)", email);
    }
    if !email.contains('.') {
        return Err(anyhow!("Invalid email: '{}' (missing domain)", email));
    }
    Ok(())
}

fn setup_server(port_str: &str, email: &str) -> Result<String> {
    let port = parse_port(port_str)
        .context("Failed to parse server port")?;

    validate_email(email)
        .context("Failed to validate admin email")?;

    Ok(format!("Server ready on port {} (admin: {})", port, email))
}

fn main() -> Result<()> {
    let tests = [
        ("8080", "admin@example.com", true),
        ("80", "admin@example.com", false),   // port too low
        ("8080", "invalid-email", false),      // bad email
        ("abc", "admin@example.com", false),   // bad port
    ];

    for (port, email, should_work) in &tests {
        print!("Port={}, Email={}: ", port, email);
        match setup_server(port, email) {
            Ok(msg) => println!("✅ {}", msg),
            Err(e) => {
                println!("❌ {}", e);
                // anyhow shows full error chain
                for cause in e.chain().skip(1) {
                    println!("   Caused by: {}", cause);
                }
            }
        }
    }

    Ok(())
}
```

---

## 4. Error Patterns ระดับ Production

### 4.1 Error Context และ Logging

```rust
use std::fmt;

#[derive(Debug)]
pub struct ErrorContext {
    pub message: String,
    pub file: &'static str,
    pub line: u32,
    pub context: Vec<String>,
}

macro_rules! error_here {
    ($msg:expr) => {
        ErrorContext {
            message: $msg.to_string(),
            file: file!(),
            line: line!(),
            context: Vec::new(),
        }
    };
}

impl ErrorContext {
    pub fn with_context(mut self, ctx: impl Into<String>) -> Self {
        self.context.push(ctx.into());
        self
    }
}

impl fmt::Display for ErrorContext {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{} (at {}:{})", self.message, self.file, self.line)?;
        for ctx in &self.context {
            write!(f, "\n  → {}", ctx)?;
        }
        Ok(())
    }
}

fn risky_operation(value: i32) -> Result<i32, ErrorContext> {
    if value < 0 {
        return Err(error_here!("Value must be positive")
            .with_context(format!("Got: {}", value))
            .with_context("Called from business logic"));
    }
    Ok(value * 2)
}

fn main() {
    match risky_operation(-5) {
        Ok(v) => println!("Success: {}", v),
        Err(e) => println!("Error:\n{}", e),
    }

    match risky_operation(10) {
        Ok(v) => println!("Success: {}", v),
        Err(e) => println!("Error:\n{}", e),
    }
}
```

### 4.2 Retry Logic

```rust
use std::time::Duration;
use std::thread;

#[derive(Debug)]
enum RetryError {
    MaxRetriesExceeded { attempts: u32, last_error: String },
    NonRetryable(String),
}

impl std::fmt::Display for RetryError {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        match self {
            RetryError::MaxRetriesExceeded { attempts, last_error } =>
                write!(f, "Failed after {} attempts: {}", attempts, last_error),
            RetryError::NonRetryable(msg) =>
                write!(f, "Non-retryable error: {}", msg),
        }
    }
}

fn retry<T, E, F>(max_attempts: u32, mut operation: F) -> Result<T, RetryError>
where
    E: std::fmt::Display,
    F: FnMut() -> Result<T, E>,
{
    let mut last_error = String::new();

    for attempt in 1..=max_attempts {
        match operation() {
            Ok(result) => return Ok(result),
            Err(e) => {
                last_error = e.to_string();
                if attempt < max_attempts {
                    let delay_ms = 100 * (2u64.pow(attempt - 1));
                    println!("Attempt {} failed: {}. Retrying in {}ms...",
                        attempt, e, delay_ms);
                    thread::sleep(Duration::from_millis(delay_ms.min(1000)));
                }
            }
        }
    }

    Err(RetryError::MaxRetriesExceeded {
        attempts: max_attempts,
        last_error,
    })
}

fn unreliable_operation(counter: &mut u32) -> Result<String, String> {
    *counter += 1;
    if *counter < 3 {
        Err(format!("Temporary failure #{}", counter))
    } else {
        Ok(format!("Success on attempt #{}", counter))
    }
}

fn main() {
    let mut counter = 0;
    match retry(5, || unreliable_operation(&mut counter)) {
        Ok(result) => println!("Final result: {}", result),
        Err(e) => println!("All retries exhausted: {}", e),
    }

    // Reset and test max retries exceeded
    let mut counter2 = 0;
    match retry(2, || {
        counter2 += 1;
        Err::<String, String>(format!("Always fails #{}", counter2))
    }) {
        Ok(r) => println!("OK: {}", r),
        Err(e) => println!("Error: {}", e),
    }
}
```

---

## 5. สรุปและ Exercises

### 5.1 สิ่งที่เรียนรู้

✅ Error handling patterns ทั้งหมด  
✅ thiserror สำหรับ library errors  
✅ anyhow สำหรับ application errors  
✅ Error context และ chaining  
✅ Retry patterns  
✅ Error codes / status codes  

### 5.2 Exercise

**Exercise: File Processing Pipeline**
```rust
// สร้าง pipeline ที่:
// 1. อ่านไฟล์ CSV
// 2. Parse แต่ละ row
// 3. Validate ข้อมูล
// 4. Transform
// 5. เขียนผลลัพธ์
// ทุกขั้นตอนต้องมี proper error handling
```

---

*[← Part 007: Collections](../part_007/README.md) | [Part 009: Traits และ Generics →](../part_009/README.md)*
