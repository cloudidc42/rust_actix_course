# Part 018: Advanced Cargo and Tooling

## บทนำ

Cargo เป็น build system และ package manager ของ Rust ที่ทรงพลังมาก ในบทนี้เราจะเรียนรู้ฟีเจอร์ขั้นสูงของ Cargo รวมถึงเครื่องมือต่างๆ ที่ช่วยให้การพัฒนา Rust มีประสิทธิภาพมากขึ้น

---

## 1. Cargo Features

Features ให้คุณควบคุมได้ว่าส่วนไหนของ code จะถูก compile

### การกำหนด Features ใน Cargo.toml

```toml
[package]
name = "my-library"
version = "0.1.0"
edition = "2021"

[features]
# กำหนด default features
default = ["json", "logging"]

# Feature เดี่ยวๆ
json = ["serde/derive", "serde_json"]
logging = ["log", "env_logger"]
async = ["tokio"]
postgres = ["sqlx/postgres"]
mysql = ["sqlx/mysql"]

# Feature ที่รวม features อื่น
full = ["json", "logging", "async", "postgres"]

[dependencies]
serde = { version = "1", optional = true }
serde_json = { version = "1", optional = true }
log = { version = "0.4", optional = true }
env_logger = { version = "0.10", optional = true }
tokio = { version = "1", features = ["full"], optional = true }
sqlx = { version = "0.7", optional = true }
```

### ใช้ Features ใน Code

```rust
// ตรวจสอบ feature ด้วย #[cfg(feature = "...")]

#[cfg(feature = "json")]
use serde::{Deserialize, Serialize};

#[cfg(feature = "logging")]
use log::{info, warn};

#[cfg_attr(feature = "json", derive(Serialize, Deserialize))]
#[derive(Debug)]
pub struct Config {
    pub host: String,
    pub port: u16,
    pub debug: bool,
}

impl Config {
    pub fn new(host: &str, port: u16) -> Self {
        Config {
            host: host.to_string(),
            port,
            debug: false,
        }
    }
    
    #[cfg(feature = "logging")]
    pub fn log_config(&self) {
        info!("Config: {}:{}", self.host, self.port);
        if self.debug {
            warn!("Debug mode is enabled!");
        }
    }
    
    #[cfg(not(feature = "logging"))]
    pub fn log_config(&self) {
        println!("Config: {}:{}", self.host, self.port);
    }
}

// Feature-gated module
#[cfg(feature = "async")]
pub mod async_client {
    use tokio::net::TcpStream;
    
    pub async fn connect(addr: &str) -> Result<TcpStream, std::io::Error> {
        TcpStream::connect(addr).await
    }
}

fn main() {
    let config = Config::new("localhost", 8080);
    config.log_config();
    
    // ตรวจสอบ features ที่เปิดใช้
    #[cfg(feature = "json")]
    println!("JSON feature is enabled");
    
    #[cfg(not(feature = "json"))]
    println!("JSON feature is disabled");
    
    // ใช้งาน feature-gated API
    #[cfg(feature = "async")]
    {
        println!("Async feature is available");
    }
}
```

### รัน Cargo ด้วย Features

```bash
# รันด้วย feature เฉพาะ
cargo run --features "json,logging"

# รันด้วยทุก features
cargo run --all-features

# รันโดยไม่ใช้ default features
cargo run --no-default-features

# รันกับ feature เฉพาะโดยไม่ใช้ default
cargo run --no-default-features --features "json"

# Build release ด้วย features
cargo build --release --features "full"
```

---

## 2. Build Scripts (build.rs)

Build scripts รันก่อน compilation เพื่อทำงาน pre-build เช่น code generation, linking libraries

### build.rs พื้นฐาน

```rust
// build.rs (อยู่ที่ root ของ package)
fn main() {
    // println! ใน build.rs ส่งข้อความไปยัง Cargo
    
    // บอก Cargo ว่าต้อง re-run เมื่อไฟล์เหล่านี้เปลี่ยน
    println!("cargo:rerun-if-changed=build.rs");
    println!("cargo:rerun-if-changed=src/");
    println!("cargo:rerun-if-env-changed=MY_ENV_VAR");
    
    // ตั้งค่า environment variable ให้ main code ใช้
    println!("cargo:rustc-env=BUILD_TIME={}", get_build_time());
    println!("cargo:rustc-env=GIT_HASH={}", get_git_hash());
    
    // เพิ่ม compile flags
    // println!("cargo:rustc-cfg=feature=\"custom_feature\"");
    
    // Link native library
    // println!("cargo:rustc-link-lib=mylib");
    // println!("cargo:rustc-link-search=native=/path/to/lib");
    
    println!("Build script completed successfully");
}

fn get_build_time() -> String {
    use std::time::{SystemTime, UNIX_EPOCH};
    let duration = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap();
    duration.as_secs().to_string()
}

fn get_git_hash() -> String {
    std::process::Command::new("git")
        .args(["rev-parse", "--short", "HEAD"])
        .output()
        .map(|output| String::from_utf8_lossy(&output.stdout).trim().to_string())
        .unwrap_or_else(|_| "unknown".to_string())
}
```

### ใช้ environment variables จาก build.rs ใน main code

```rust
// src/main.rs
fn main() {
    // อ่านค่าที่ build.rs กำหนด
    let build_time = env!("BUILD_TIME");
    let git_hash = env!("GIT_HASH");
    
    println!("Build time: {}", build_time);
    println!("Git hash: {}", git_hash);
    
    // ข้อมูล package จาก Cargo.toml
    println!("Package: {} v{}", 
        env!("CARGO_PKG_NAME"),
        env!("CARGO_PKG_VERSION")
    );
    println!("Authors: {}", env!("CARGO_PKG_AUTHORS"));
    println!("Description: {}", env!("CARGO_PKG_DESCRIPTION"));
}
```

### build.rs: Code Generation

```rust
// build.rs
use std::env;
use std::fs;
use std::path::Path;

fn main() {
    println!("cargo:rerun-if-changed=assets/config.toml");
    
    // อ่านไฟล์และ generate Rust code
    let out_dir = env::var("OUT_DIR").unwrap();
    let dest_path = Path::new(&out_dir).join("generated.rs");
    
    // สร้าง code ที่ embed configuration
    let generated_code = r#"
        pub const APP_NAME: &str = "MyApp";
        pub const MAX_CONNECTIONS: u32 = 100;
        pub const DEFAULT_TIMEOUT: u64 = 30;
        
        pub fn get_config() -> &'static str {
            "embedded configuration"
        }
    "#;
    
    fs::write(&dest_path, generated_code).unwrap();
    
    println!("Generated code written to {:?}", dest_path);
}
```

```rust
// src/main.rs - include generated code
include!(concat!(env!("OUT_DIR"), "/generated.rs"));

fn main() {
    println!("App: {}", APP_NAME);
    println!("Max connections: {}", MAX_CONNECTIONS);
    println!("Config: {}", get_config());
}
```

---

## 3. Cargo Workspace

Workspace ให้คุณจัดการหลาย crates ในโปรเจกต์เดียวกัน

```
my-workspace/
├── Cargo.toml          (workspace root)
├── core/               (library crate)
│   ├── Cargo.toml
│   └── src/lib.rs
├── server/             (binary crate)
│   ├── Cargo.toml
│   └── src/main.rs
├── cli/                (binary crate)
│   ├── Cargo.toml
│   └── src/main.rs
└── shared-types/       (shared library)
    ├── Cargo.toml
    └── src/lib.rs
```

### Workspace Cargo.toml

```toml
# my-workspace/Cargo.toml
[workspace]
members = [
    "core",
    "server",
    "cli",
    "shared-types",
]

# Shared dependencies (workspace-level)
[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
log = "0.4"
thiserror = "1"

# Workspace-level settings
[workspace.package]
version = "0.1.0"
edition = "2021"
authors = ["Your Name"]
license = "MIT"
```

### Member Cargo.toml

```toml
# shared-types/Cargo.toml
[package]
name = "shared-types"
version.workspace = true
edition.workspace = true

[dependencies]
serde.workspace = true
thiserror.workspace = true
```

```toml
# core/Cargo.toml
[package]
name = "core"
version.workspace = true
edition.workspace = true

[dependencies]
shared-types = { path = "../shared-types" }
serde.workspace = true
tokio.workspace = true
```

```toml
# server/Cargo.toml
[package]
name = "server"
version.workspace = true
edition.workspace = true

[dependencies]
core = { path = "../core" }
shared-types = { path = "../shared-types" }
tokio.workspace = true
```

### Workspace Code Examples

```rust
// shared-types/src/lib.rs
use serde::{Deserialize, Serialize};
use thiserror::Error;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct User {
    pub id: u64,
    pub name: String,
    pub email: String,
}

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
    
    pub fn err(msg: &str) -> Self {
        ApiResponse {
            success: false,
            data: None,
            error: Some(msg.to_string()),
        }
    }
}

#[derive(Debug, Error)]
pub enum AppError {
    #[error("User not found: {0}")]
    UserNotFound(u64),
    #[error("Database error: {0}")]
    DatabaseError(String),
    #[error("Invalid input: {0}")]
    InvalidInput(String),
}
```

```rust
// core/src/lib.rs
use shared_types::{User, AppError};
use std::collections::HashMap;
use std::sync::{Arc, RwLock};

pub struct UserRepository {
    data: Arc<RwLock<HashMap<u64, User>>>,
    next_id: Arc<RwLock<u64>>,
}

impl UserRepository {
    pub fn new() -> Self {
        UserRepository {
            data: Arc::new(RwLock::new(HashMap::new())),
            next_id: Arc::new(RwLock::new(1)),
        }
    }
    
    pub fn create(&self, name: String, email: String) -> Result<User, AppError> {
        if name.is_empty() {
            return Err(AppError::InvalidInput("Name cannot be empty".to_string()));
        }
        
        let id = {
            let mut next = self.next_id.write().unwrap();
            let id = *next;
            *next += 1;
            id
        };
        
        let user = User { id, name, email };
        self.data.write().unwrap().insert(id, user.clone());
        Ok(user)
    }
    
    pub fn find_by_id(&self, id: u64) -> Result<User, AppError> {
        self.data
            .read()
            .unwrap()
            .get(&id)
            .cloned()
            .ok_or(AppError::UserNotFound(id))
    }
    
    pub fn list_all(&self) -> Vec<User> {
        self.data.read().unwrap().values().cloned().collect()
    }
}
```

### รัน Workspace Commands

```bash
# Build ทุก members
cargo build --workspace

# Test ทุก members
cargo test --workspace

# Build specific member
cargo build -p server

# Run specific binary
cargo run -p cli

# Check ทุก members
cargo check --workspace
```

---

## 4. Rustfmt Configuration

```toml
# rustfmt.toml หรือ .rustfmt.toml ที่ root

edition = "2021"
max_width = 100
tab_spaces = 4
hard_tabs = false

# Import grouping
imports_granularity = "Module"
group_imports = "StdExternalCrate"

# Formatting options
newline_style = "Unix"
use_small_heuristics = "Default"
reorder_imports = true
reorder_modules = true

# Function formatting
fn_single_line = false
where_single_line = false

# Trailing commas
trailing_comma = "Vertical"
trailing_semicolon = true

# Match formatting
match_block_trailing_comma = true
match_arm_leading_pipes = "Never"

# Comment formatting
comment_width = 80
wrap_comments = true
normalize_comments = true

# Blank lines
blank_lines_upper_bound = 2
blank_lines_lower_bound = 0
```

```bash
# Format ทุกไฟล์
cargo fmt

# Check formatting โดยไม่แก้ไข
cargo fmt -- --check

# Format ไฟล์เดียว
rustfmt src/main.rs

# Format workspace
cargo fmt --all
```

---

## 5. Clippy Lints

```rust
// src/main.rs - ตัวอย่างที่ Clippy จะ warn

// Clippy จะแนะนำให้ใช้ .is_empty() แทน
fn check_empty(v: &Vec<i32>) -> bool {
    v.len() == 0  // clippy::len_zero
}

// ควรใช้ & แทน
fn take_string(s: String) {  // clippy::needless_pass_by_value
    println!("{}", s);
}

// Unused imports
use std::collections::HashMap;  // clippy::unused_imports

// Magic numbers
fn calculate(x: f64) -> f64 {
    x * 3.14159  // clippy::approx_constant
}

// Better: use std::f64::consts::PI
fn calculate_better(x: f64) -> f64 {
    x * std::f64::consts::PI
}
```

### Clippy Configuration

```toml
# .clippy.toml หรือ clippy.toml

# กำหนด cognitive complexity limit
cognitive-complexity-threshold = 25

# กำหนด function argument count limit
too-many-arguments-threshold = 7

# กำหนด max enum variants
enum-variant-name-threshold = 3
```

### Custom Clippy Attributes ใน Code

```rust
// ปิด clippy สำหรับฟังก์ชัน
#[allow(clippy::too_many_arguments)]
fn complex_function(
    a: i32, b: i32, c: i32, d: i32,
    e: i32, f: i32, g: i32, h: i32,
) -> i32 {
    a + b + c + d + e + f + g + h
}

// ปิด clippy สำหรับทั้ง module
#![allow(clippy::module_name_repetitions)]

// เตือน clippy level
#[warn(clippy::pedantic)]
fn pedantic_function() {
    // ...
}

// Deny specific lint
#[deny(clippy::unwrap_used)]
fn safe_function() -> Option<i32> {
    Some(42)
    // unwrap() จะเป็น error ที่นี่
}
```

```bash
# Run clippy
cargo clippy

# Fix automatically
cargo clippy --fix

# Run with all lints
cargo clippy -- -W clippy::all -W clippy::pedantic

# Deny warnings (CI mode)
cargo clippy -- -D warnings
```

---

## 6. cargo-audit - Security Auditing

```bash
# ติดตั้ง cargo-audit
cargo install cargo-audit

# ตรวจสอบ vulnerabilities
cargo audit

# ตรวจสอบพร้อม JSON output
cargo audit --json

# Ignore specific advisory
cargo audit --ignore RUSTSEC-2021-XXXX

# Update advisory database
cargo audit --update
```

### audit.toml configuration

```toml
# audit.toml
[advisories]
# Ignore specific advisories
ignore = ["RUSTSEC-2021-0001"]

[output]
deny = ["warnings", "unmaintained", "unsound", "yanked"]
format = "terminal"
quiet = false

[target]
# เฉพาะ platforms
platforms = ["x86_64-unknown-linux-gnu"]
```

---

## 7. cargo-watch

```bash
# ติดตั้ง
cargo install cargo-watch

# Watch และ run เมื่อไฟล์เปลี่ยน
cargo watch -x run

# Watch และ test
cargo watch -x test

# Watch ไฟล์เฉพาะ
cargo watch -w src -x build

# หลาย commands
cargo watch -x "check --tests" -x test -x run

# Clear terminal ทุกครั้ง
cargo watch -c -x run

# Ignore patterns
cargo watch --ignore "*.log" -x run
```

---

## 8. cargo-expand - Macro Expansion

```bash
# ติดตั้ง
cargo install cargo-expand

# Expand macros ทั้งหมด
cargo expand

# Expand specific item
cargo expand --item main
cargo expand --item MyStruct

# Expand test module
cargo expand --test my_test

# Format output
cargo expand | rustfmt
```

```rust
// ตัวอย่างที่จะ expand
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: f64,
    y: f64,
}

// vec! macro
let v = vec![1, 2, 3];

// println! macro
println!("{:?}", v);
```

---

## 9. Cross-compilation Basics

```bash
# ดู targets ที่มีอยู่
rustup target list

# เพิ่ม target
rustup target add x86_64-unknown-linux-musl
rustup target add aarch64-apple-darwin
rustup target add wasm32-unknown-unknown
rustup target add x86_64-pc-windows-gnu

# Cross-compile
cargo build --target x86_64-unknown-linux-musl
cargo build --target wasm32-unknown-unknown

# ติดตั้ง cross (easier cross-compilation)
cargo install cross

# ใช้ cross
cross build --target aarch64-unknown-linux-gnu
```

### Cross-compilation ใน Cargo.toml

```toml
# .cargo/config.toml
[target.x86_64-unknown-linux-musl]
linker = "x86_64-linux-musl-gcc"

[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"

[target.wasm32-unknown-unknown]
runner = "wasm-pack"
```

---

## 10. Release Profile Optimization

```toml
# Cargo.toml

[profile.dev]
opt-level = 0          # ไม่ optimize (fast compile)
debug = true           # เก็บ debug info
debug-assertions = true # เปิด assertions
overflow-checks = true  # ตรวจ integer overflow
lto = false            # ไม่ใช้ LTO
panic = "unwind"       # unwind on panic
incremental = true     # incremental compilation
codegen-units = 256    # parallel compilation units

[profile.release]
opt-level = 3          # optimize สูงสุด
debug = false          # ไม่เก็บ debug info
debug-assertions = false
overflow-checks = false
lto = true             # ใช้ Link-Time Optimization
panic = "abort"        # abort on panic (ลด binary size)
incremental = false
codegen-units = 1      # ให้ LLVM optimize ดีขึ้น
strip = true           # ลบ symbols ออก

# Custom profile
[profile.bench]
inherits = "release"
debug = true           # เก็บ debug info สำหรับ profiling

[profile.perf]
inherits = "release"
opt-level = 3
debug = true
lto = "fat"            # Full LTO

# Optimize specific dependency
[profile.release.package.serde]
opt-level = 3

[profile.dev.package."*"]
opt-level = 2  # optimize dependencies แม้ใน dev
```

### Measuring Binary Size

```bash
# Build release
cargo build --release

# Check binary size
ls -lh target/release/my-app

# Strip symbols (ลด size)
strip target/release/my-app

# ใช้ upx compression
upx --best target/release/my-app

# Analyze binary size with cargo-bloat
cargo install cargo-bloat
cargo bloat --release
cargo bloat --release --crates
cargo bloat --release -n 20  # top 20 functions
```

---

## 11. Useful Cargo Commands

```bash
# ดู dependency tree
cargo tree
cargo tree --no-dedupe
cargo tree -d  # duplicates only
cargo tree -i serde  # packages using serde

# ตรวจสอบ outdated packages
cargo install cargo-outdated
cargo outdated

# Update dependencies
cargo update
cargo update --precise serde@1.0.150

# Generate documentation
cargo doc
cargo doc --open
cargo doc --no-deps --document-private-items

# Benchmark
cargo bench

# Check license compatibility
cargo install cargo-license
cargo license

# Dependency sizes
cargo install cargo-deps
cargo deps | dot -Tpng > deps.png

# Clean build artifacts
cargo clean
cargo clean --release

# Publish to crates.io
cargo login
cargo publish --dry-run
cargo publish
```

---

## 12. Cargo.toml - Full Example

```toml
[package]
name = "my-app"
version = "1.0.0"
edition = "2021"
authors = ["Developer <dev@example.com>"]
description = "A comprehensive Rust application"
documentation = "https://docs.rs/my-app"
homepage = "https://github.com/user/my-app"
repository = "https://github.com/user/my-app"
readme = "README.md"
keywords = ["web", "server", "api"]
categories = ["web-programming"]
license = "MIT OR Apache-2.0"
exclude = [".github/", "docs/", "*.md"]
include = ["src/", "Cargo.toml", "LICENSE*"]
build = "build.rs"
rust-version = "1.70.0"

[lib]
name = "my_lib"
path = "src/lib.rs"
crate-type = ["cdylib", "rlib"]

[[bin]]
name = "server"
path = "src/bin/server.rs"

[[bin]]
name = "cli"
path = "src/bin/cli.rs"

[[example]]
name = "basic"
path = "examples/basic.rs"

[[bench]]
name = "performance"
path = "benches/performance.rs"
harness = false  # ใช้ criterion

[dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
reqwest = { version = "0.11", features = ["json", "rustls-tls"] }
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio"] }
anyhow = "1"
thiserror = "1"
tracing = "0.1"

[dev-dependencies]
tokio = { version = "1", features = ["test-util"] }
mockall = "0.11"
criterion = { version = "0.5", features = ["html_reports"] }
assert_cmd = "2"
predicates = "3"

[build-dependencies]
cc = "1"  # C code compilation

[features]
default = ["json"]
json = ["serde/derive"]
postgres = ["sqlx/postgres"]
redis = ["redis"]

[patch.crates-io]
# ใช้ local version แทน
# some-crate = { path = "../some-crate" }
```

---

## สรุป

| Tool | ใช้สำหรับ |
|------|-----------|
| `cargo features` | Conditional compilation |
| `build.rs` | Pre-build tasks, code generation |
| `workspace` | Multi-crate projects |
| `rustfmt` | Code formatting |
| `clippy` | Linting, best practices |
| `cargo-audit` | Security vulnerability scanning |
| `cargo-watch` | Auto-rebuild on file changes |
| `cargo-expand` | Macro expansion inspection |
| Cross-compilation | Build for different targets |
| Release profiles | Optimize for production |

---

## Navigation

- [← Part 017: Smart Pointers](../part_017/README.md)
- [→ Part 019: String Handling Deep Dive](../part_019/README.md)
- [กลับหน้าหลัก](../../README.md)
