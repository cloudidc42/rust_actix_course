# Part 001: การติดตั้ง Rust และ Hello World 🦀

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ติดตั้ง Rust บนระบบปฏิบัติการต่างๆ ได้
- เข้าใจโครงสร้างพื้นฐานของโปรเจกต์ Rust
- รันโปรแกรม Hello World ได้
- เข้าใจ Cargo (ตัวจัดการแพ็กเกจของ Rust)
- รู้จักคำสั่ง Cargo พื้นฐาน

---

## 1. ทำไมต้องเรียน Rust?

Rust เป็นภาษาโปรแกรมที่ถูกพัฒนาโดย Mozilla และปัจจุบันดูแลโดย Rust Foundation Rust ถูกออกแบบมาเพื่อ:

### 1.1 ความปลอดภัยของ Memory
```
ภาษาอื่น:
C/C++ → ต้องจัดการ memory เอง → เกิด Memory Leak, Buffer Overflow
Java/Python → GC จัดการให้ → ช้า, ไม่ predictable

Rust:
→ Ownership System จัดการ Memory โดยอัตโนมัติ ณ Compile Time
→ ไม่มี GC → เร็วเหมือน C/C++
→ ปลอดภัยเหมือน Java/Python
```

### 1.2 ประสิทธิภาพสูง
```
Benchmark (requests/sec):
Actix-web (Rust) : ~600,000 req/sec
Gin (Go)         : ~400,000 req/sec
Express (Node.js): ~50,000 req/sec
Django (Python)  : ~10,000 req/sec
```

### 1.3 Concurrency ที่ปลอดภัย
- ไม่มี Data Race ณ Compile Time
- Fearless Concurrency
- async/await แบบ Zero-cost

### 1.4 ใครใช้ Rust?
- **Amazon AWS** - ใช้สำหรับ cloud infrastructure
- **Microsoft** - Windows kernel components
- **Google** - Android components
- **Meta (Facebook)** - Various systems
- **Discord** - ย้ายจาก Go มา Rust ประหยัด CPU 72%
- **Cloudflare** - Edge computing

---

## 2. การติดตั้ง Rust

### 2.1 Linux / macOS

```bash
# ใช้ rustup (วิธีแนะนำ)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# เลือก option 1 (default installation)
# กด Enter

# หลังติดตั้ง ต้อง source environment
source ~/.cargo/env

# หรือเปิด terminal ใหม่
```

### 2.2 Windows

1. ดาวน์โหลด `rustup-init.exe` จาก https://rustup.rs
2. รันไฟล์และทำตามขั้นตอน
3. อาจต้องติดตั้ง Visual Studio C++ Build Tools ด้วย

### 2.3 ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบ Rust version
rustc --version
# ผลลัพธ์: rustc 1.75.0 (82e1608df 2023-12-21)

# ตรวจสอบ Cargo version
cargo --version
# ผลลัพธ์: cargo 1.75.0 (1d8b05cdd 2023-11-20)

# ตรวจสอบ rustup
rustup --version
# ผลลัพธ์: rustup 1.26.0 (5af9b9484 2023-04-05)
```

### 2.4 อัพเดต Rust

```bash
# อัพเดตเป็นเวอร์ชันล่าสุด
rustup update

# ดู toolchain ที่ติดตั้ง
rustup toolchain list
# stable-x86_64-unknown-linux-gnu (default)
# nightly-x86_64-unknown-linux-gnu

# ติดตั้ง nightly (สำหรับ features ใหม่ๆ)
rustup toolchain install nightly

# ใช้ nightly สำหรับโปรเจกต์เฉพาะ
rustup override set nightly
```

---

## 3. Editor Setup

### 3.1 VS Code (แนะนำสำหรับผู้เริ่มต้น)

```bash
# ติดตั้ง extensions ผ่าน command line
code --install-extension rust-lang.rust-analyzer
code --install-extension vadimcn.vscode-lldb
code --install-extension serayuzgur.crates
code --install-extension tamasfe.even-better-toml
```

สร้างไฟล์ `.vscode/settings.json`:
```json
{
    "editor.formatOnSave": true,
    "[rust]": {
        "editor.defaultFormatter": "rust-lang.rust-analyzer"
    },
    "rust-analyzer.checkOnSave.command": "clippy",
    "rust-analyzer.inlayHints.enable": true,
    "rust-analyzer.inlayHints.typeHints.enable": true,
    "rust-analyzer.inlayHints.parameterHints.enable": true
}
```

### 3.2 IntelliJ IDEA / CLion
- ติดตั้ง Rust plugin จาก marketplace

### 3.3 Neovim
```lua
-- ใช้ lazy.nvim
{
    "neovim/nvim-lspconfig",
    config = function()
        require('lspconfig').rust_analyzer.setup({})
    end
}
```

---

## 4. โปรเจกต์แรก: Hello World

### 4.1 สร้างโปรเจกต์ใหม่

```bash
# สร้างโปรเจกต์ใหม่ด้วย Cargo
cargo new hello_world
cd hello_world

# โครงสร้างไฟล์ที่สร้าง:
# hello_world/
# ├── Cargo.toml    ← ไฟล์ config โปรเจกต์
# └── src/
#     └── main.rs   ← ไฟล์ source code หลัก
```

### 4.2 ดูไฟล์ Cargo.toml

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at
# https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
```

**อธิบาย:**
- `[package]` - ข้อมูลของ package/crate
- `name` - ชื่อโปรเจกต์
- `version` - เวอร์ชัน (Semantic Versioning)
- `edition` - Rust edition (2015, 2018, 2021)
- `[dependencies]` - รายการ dependencies

### 4.3 ดูไฟล์ src/main.rs

```rust
fn main() {
    println!("Hello, world!");
}
```

### 4.4 รันโปรแกรม

```bash
# วิธีที่ 1: Build แล้วรัน
cargo build           # build เป็น debug mode
cargo run             # build + run

# วิธีที่ 2: Build แบบ release (optimized)
cargo build --release
./target/release/hello_world

# วิธีที่ 3: ตรวจสอบ code โดยไม่ build
cargo check           # เร็วกว่า build มาก

# ผลลัพธ์:
# Hello, world!
```

---

## 5. Hello World แบบละเอียด

### 5.1 โปรแกรมที่สมบูรณ์กว่า

```rust
// src/main.rs

// fn = function keyword
// main = ชื่อ function หลัก ที่ Rust เริ่มรันจากที่นี่
fn main() {
    // println! เป็น macro (ไม่ใช่ function)
    // ! บ่งบอกว่าเป็น macro
    println!("Hello, World!");

    // พิมพ์หลายบรรทัด
    println!("สวัสดี Rust!");
    println!("ยินดีต้อนรับสู่โลกของ Rust");

    // Format string แบบต่างๆ
    let name = "Rust";
    let version = 1.75;
    println!("ภาษา {} เวอร์ชัน {}", name, version);
    println!("ภาษา {name} เวอร์ชัน {version}");  // Rust 1.58+

    // พิมพ์ตัวเลขในรูปแบบต่างๆ
    println!("Decimal: {}", 42);
    println!("Hex: {:x}", 255);        // ff
    println!("Hex (upper): {:X}", 255);// FF
    println!("Binary: {:b}", 42);      // 101010
    println!("Octal: {:o}", 42);       // 52

    // Padding และ Alignment
    println!("{:>10}", "right");     // "     right"
    println!("{:<10}", "left");      // "left      "
    println!("{:^10}", "center");    // "  center  "
    println!("{:0>5}", 42);          // "00042"

    // Debug output
    let numbers = vec![1, 2, 3, 4, 5];
    println!("Debug: {:?}", numbers);
    println!("Pretty Debug: {:#?}", numbers);

    // eprintln! - พิมพ์ไปยัง stderr
    eprintln!("นี่คือ error message");

    // print! - ไม่มี newline
    print!("Hello ");
    print!("World");
    println!("!");  // Hello World!
}
```

### 5.2 รันและดูผลลัพธ์

```bash
cargo run
```

**ผลลัพธ์:**
```
Hello, World!
สวัสดี Rust!
ยินดีต้อนรับสู่โลกของ Rust
ภาษา Rust เวอร์ชัน 1.75
ภาษา Rust เวอร์ชัน 1.75
Decimal: 42
Hex: ff
Hex (upper): FF
Binary: 101010
Octal: 52
     right
left      
  center  
00042
Debug: [1, 2, 3, 4, 5]
Pretty Debug: [
    1,
    2,
    3,
    4,
    5,
]
Hello World!
```

---

## 6. Cargo Commands ที่สำคัญ

### 6.1 คำสั่งพื้นฐาน

```bash
# สร้างโปรเจกต์ใหม่ (binary)
cargo new my_project

# สร้าง library
cargo new my_lib --lib

# สร้างใน directory ปัจจุบัน
cargo init

# Build
cargo build               # debug mode (เร็วในการ compile)
cargo build --release     # release mode (เร็วในการรัน)

# Run
cargo run                 # build + run
cargo run --release       # build release + run
cargo run -- arg1 arg2    # ส่ง arguments

# Test
cargo test                # รัน tests ทั้งหมด
cargo test test_name      # รัน test เฉพาะชื่อ

# Check (ตรวจ syntax โดยไม่ build)
cargo check

# Clean build artifacts
cargo clean

# Document
cargo doc                 # สร้าง documentation
cargo doc --open          # สร้างและเปิดใน browser

# Lint
cargo clippy              # ตรวจสอบ code quality

# Format
cargo fmt                 # จัดรูปแบบ code

# Update dependencies
cargo update
```

### 6.2 cargo watch (live reload)

```bash
# ติดตั้ง
cargo install cargo-watch

# รัน + reload อัตโนมัติเมื่อแก้ไข
cargo watch -x run
cargo watch -x "run -- --port 8080"
cargo watch -x test
cargo watch -x check
```

---

## 7. โครงสร้างโปรเจกต์ Rust

### 7.1 Binary Project

```
my_project/
├── Cargo.toml          ← Project config
├── Cargo.lock          ← Locked dependencies (auto-generated)
├── src/
│   ├── main.rs         ← Entry point
│   ├── lib.rs          ← Library root (optional)
│   ├── utils.rs        ← Module file
│   └── models/
│       ├── mod.rs      ← Module declaration
│       └── user.rs     ← Sub-module
├── tests/
│   └── integration_test.rs
├── examples/
│   └── example1.rs
├── benches/
│   └── benchmark.rs
└── target/             ← Build output (gitignore this)
    ├── debug/
    └── release/
```

### 7.2 Workspace (หลาย packages)

```
my_workspace/
├── Cargo.toml          ← Workspace config
├── api/
│   ├── Cargo.toml
│   └── src/main.rs
├── core/
│   ├── Cargo.toml
│   └── src/lib.rs
└── common/
    ├── Cargo.toml
    └── src/lib.rs
```

**Workspace Cargo.toml:**
```toml
[workspace]
members = [
    "api",
    "core",
    "common",
]

[workspace.dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
```

---

## 8. Cargo.toml ขั้นสูง

### 8.1 Cargo.toml สมบูรณ์

```toml
[package]
name = "my_app"
version = "0.1.0"
edition = "2021"
authors = ["Your Name <you@example.com>"]
description = "My awesome Rust application"
license = "MIT"
repository = "https://github.com/username/my_app"
readme = "README.md"
keywords = ["web", "api", "rust"]
categories = ["web-programming"]

# Minimum Rust version required
rust-version = "1.70"

[dependencies]
# เวอร์ชันเฉพาะ
serde = "1.0.193"

# เวอร์ชัน compatible (>= 1.0, < 2.0)
tokio = "1"

# เวอร์ชัน exact
regex = "=1.10.2"

# พร้อม features
actix-web = { version = "4", features = ["openssl"] }

# Git dependency
my_lib = { git = "https://github.com/user/my_lib", branch = "main" }

# Path dependency (local)
my_local = { path = "../my_local_lib" }

# Optional dependency
optional_crate = { version = "1", optional = true }

[dev-dependencies]
# ใช้แค่ใน test
mockall = "0.11"
tokio = { version = "1", features = ["full", "test-util"] }

[build-dependencies]
# ใช้ใน build.rs
prost-build = "0.12"

[features]
# Default features
default = ["basic"]

# Custom features
basic = []
full = ["basic", "dep:optional_crate"]
async-support = ["dep:tokio"]

[profile.dev]
opt-level = 0      # ไม่ optimize (compile เร็ว)
debug = true       # มี debug symbols
overflow-checks = true

[profile.release]
opt-level = 3      # optimize สูงสุด
debug = false
lto = true         # Link-Time Optimization
codegen-units = 1  # เพิ่ม optimization
panic = "abort"    # binary เล็กลง
strip = true       # ลบ symbols

[profile.test]
opt-level = 1

[lib]
name = "my_lib"
path = "src/lib.rs"
crate-type = ["cdylib", "rlib"]

[[bin]]
name = "main"
path = "src/main.rs"

[[bin]]
name = "worker"
path = "src/bin/worker.rs"

[[example]]
name = "basic_usage"
path = "examples/basic.rs"
```

---

## 9. เข้าใจ Rust Compilation

### 9.1 ขั้นตอนการ Compile

```
Source Code (.rs)
    ↓
Lexical Analysis → Tokens
    ↓
Parsing → AST (Abstract Syntax Tree)
    ↓
Name Resolution
    ↓
Type Checking + Borrow Checking  ← สิ่งที่ทำให้ Rust พิเศษ
    ↓
MIR (Mid-level Intermediate Representation)
    ↓
LLVM IR
    ↓
Machine Code
```

### 9.2 Debug vs Release

```bash
# Debug mode
cargo build
# - เร็วในการ compile
# - มี debug symbols
# - ไม่ optimize
# - ใช้สำหรับ development

# Release mode
cargo build --release
# - ช้าในการ compile
# - ไม่มี debug symbols (ถ้า strip = true)
# - optimize เต็มที่
# - ใช้สำหรับ production

# เปรียบเทียบขนาด
ls -lh target/debug/hello_world
ls -lh target/release/hello_world
# Release จะเล็กกว่ามาก
```

---

## 10. ทดลองเขียนโปรแกรมแรก

### 10.1 สร้างโปรเจกต์และเขียนโค้ด

```bash
cargo new greet_program
cd greet_program
```

แก้ไข `src/main.rs`:

```rust
use std::io;
use std::io::Write;

fn main() {
    // รับ input จากผู้ใช้
    print!("กรุณาใส่ชื่อของคุณ: ");
    io::stdout().flush().unwrap();  // flush buffer

    let mut name = String::new();
    io::stdin()
        .read_line(&mut name)
        .expect("ไม่สามารถอ่าน input ได้");

    // trim() ลบ whitespace และ newline
    let name = name.trim();

    // ทักทาย
    println!("สวัสดี, {}!", name);
    println!("ยินดีต้อนรับสู่โลกของ Rust 🦀");

    // ข้อมูลเพิ่มเติม
    let name_length = name.len();
    println!("ชื่อของคุณมี {} ตัวอักษร", name_length);

    // เช็คว่าชื่อว่างไหม
    if name.is_empty() {
        println!("คุณไม่ได้ใส่ชื่อมา!");
    } else {
        println!("ชื่อของคุณเริ่มต้นด้วย: {}", &name[0..1]);
    }
}
```

```bash
cargo run
# กรุณาใส่ชื่อของคุณ: สมชาย
# สวัสดี, สมชาย!
# ยินดีต้อนรับสู่โลกของ Rust 🦀
# ชื่อของคุณมี 12 ตัวอักษร
# ชื่อของคุณเริ่มต้นด้วย: ส
```

### 10.2 เพิ่ม Dependency ครั้งแรก

แก้ไข `Cargo.toml`:
```toml
[package]
name = "greet_program"
version = "0.1.0"
edition = "2021"

[dependencies]
colored = "2"
chrono = "0.4"
```

แก้ไข `src/main.rs`:
```rust
use std::io;
use std::io::Write;
use colored::Colorize;
use chrono::Local;

fn main() {
    // แสดงเวลาปัจจุบัน
    let now = Local::now();
    println!("{}", format!("เวลาปัจจุบัน: {}", now.format("%Y-%m-%d %H:%M:%S")).blue());

    print!("กรุณาใส่ชื่อของคุณ: ");
    io::stdout().flush().unwrap();

    let mut name = String::new();
    io::stdin()
        .read_line(&mut name)
        .expect("ไม่สามารถอ่าน input ได้");

    let name = name.trim();

    // ใช้ colored library
    println!("{}", format!("สวัสดี, {}!", name).green().bold());
    println!("{}", "ยินดีต้อนรับสู่โลกของ Rust 🦀".yellow());

    if name.to_lowercase() == "admin" {
        println!("{}", "⚠️  คุณเป็น Admin!".red().bold());
    }
}
```

```bash
cargo run
# จะ download dependencies ก่อน
# Compiling colored v2.1.0
# Compiling chrono v0.4.31
# Compiling greet_program v0.1.0
```

---

## 11. Rust Toolchain Components

### 11.1 Components สำคัญ

```bash
# ดู components ที่ติดตั้ง
rustup component list --installed

# components ที่มีประโยชน์
rustup component add clippy       # linter
rustup component add rustfmt      # code formatter
rustup component add rust-src     # source code (สำหรับ IDE)
rustup component add rust-docs    # offline documentation
rustup component add llvm-tools   # profiling tools
```

### 11.2 Cross-compilation Targets

```bash
# ดู targets ที่รองรับ
rustup target list

# เพิ่ม target
rustup target add x86_64-pc-windows-gnu     # Windows
rustup target add aarch64-unknown-linux-gnu  # ARM Linux
rustup target add wasm32-unknown-unknown     # WebAssembly

# Build สำหรับ target อื่น
cargo build --target x86_64-pc-windows-gnu
cargo build --target wasm32-unknown-unknown
```

---

## 12. ทำความเข้าใจ Rust Code

### 12.1 ส่วนประกอบพื้นฐาน

```rust
// นี่คือ comment แบบ single line

/*
  นี่คือ comment แบบ multi-line
*/

/// นี่คือ doc comment (สำหรับสร้าง documentation)
/// จะแสดงใน cargo doc

fn main() {
    // --------------------------------
    // Variables
    // --------------------------------

    // let = declare variable (immutable by default)
    let x = 5;
    // x = 6;  // ERROR! ไม่สามารถเปลี่ยนค่าได้

    // mut = mutable
    let mut y = 10;
    y = 20;  // OK

    // ระบุชนิดข้อมูล
    let z: i32 = 42;
    let pi: f64 = 3.14159;
    let is_active: bool = true;
    let letter: char = 'A';

    // --------------------------------
    // Printing
    // --------------------------------

    println!("x = {}", x);
    println!("y = {}", y);
    println!("z = {z}");  // shorthand
    println!("pi = {pi:.2}");  // 2 decimal places

    // --------------------------------
    // Expressions
    // --------------------------------

    // Rust เป็น expression-based
    let value = {
        let a = 3;
        let b = 4;
        a * a + b * b  // ไม่มี semicolon = return value
    };
    println!("value = {}", value);  // 25

    // --------------------------------
    // Functions
    // --------------------------------

    let result = add(3, 4);
    println!("3 + 4 = {}", result);

    // --------------------------------
    // Statements vs Expressions
    // --------------------------------

    // Statement: ไม่ return ค่า
    let _statement = 5;  // นี่เป็น statement

    // Expression: return ค่า
    let expression = 5 + 3;  // นี่เป็น expression

    println!("expression = {expression}");
}

// Function definition
fn add(a: i32, b: i32) -> i32 {
    a + b  // implicit return (ไม่มี semicolon)
    // เหมือนกับ: return a + b;
}
```

---

## 13. สรุปและ Exercises

### 13.1 สิ่งที่เรียนรู้ใน Part นี้

✅ ติดตั้ง Rust ผ่าน rustup  
✅ เข้าใจ Cargo และคำสั่งพื้นฐาน  
✅ สร้างโปรเจกต์และรัน Hello World  
✅ เข้าใจโครงสร้างไฟล์ Rust project  
✅ เข้าใจ Cargo.toml  
✅ ใช้ println! macro ในรูปแบบต่างๆ  
✅ เพิ่ม dependency เข้าโปรเจกต์  

### 13.2 Exercises

**Exercise 1: Basic Output**
สร้างโปรแกรมที่แสดงข้อมูลส่วนตัวของคุณ:
- ชื่อ
- อายุ
- งานอดิเรก (เป็น list)
- ภาษาโปรแกรมที่ชอบ

**Exercise 2: Calculator**
สร้างโปรแกรม calculator อย่างง่ายที่:
- รับตัวเลข 2 ตัวจาก user
- แสดงผลบวก ลบ คูณ หาร

**Exercise 3: Cargo Exploration**
- สร้าง workspace ที่มี 2 projects
- projects หนึ่งเป็น binary, อีกอันเป็น library
- ให้ binary ใช้ library

### 13.3 แนวทางการทำ Exercise

**Exercise 1 Solution:**
```rust
fn main() {
    let name = "Somchai Rust";
    let age = 25;
    let hobbies = ["เขียนโค้ด", "อ่านหนังสือ", "เล่นเกม"];
    let favorite_lang = "Rust";

    println!("=== ข้อมูลส่วนตัว ===");
    println!("ชื่อ: {}", name);
    println!("อายุ: {} ปี", age);
    println!("งานอดิเรก:");
    for (i, hobby) in hobbies.iter().enumerate() {
        println!("  {}. {}", i + 1, hobby);
    }
    println!("ภาษาโปรแกรมที่ชอบ: {}", favorite_lang);
}
```

**Exercise 2 Solution:**
```rust
use std::io;

fn main() {
    println!("=== Calculator ===");

    let a = read_number("ใส่ตัวเลขที่ 1: ");
    let b = read_number("ใส่ตัวเลขที่ 2: ");

    println!("{} + {} = {}", a, b, a + b);
    println!("{} - {} = {}", a, b, a - b);
    println!("{} × {} = {}", a, b, a * b);

    if b != 0.0 {
        println!("{} ÷ {} = {:.2}", a, b, a / b);
    } else {
        println!("ไม่สามารถหารด้วย 0 ได้!");
    }
}

fn read_number(prompt: &str) -> f64 {
    print!("{}", prompt);
    io::Write::flush(&mut io::stdout()).unwrap();

    let mut input = String::new();
    io::stdin().read_line(&mut input).unwrap();

    input.trim().parse().expect("กรุณาใส่ตัวเลข")
}
```

---

## 14. Resources และลิงก์ที่มีประโยชน์

### 14.1 Official Resources
- **Rust Book**: https://doc.rust-lang.org/book/ (The Rust Programming Language)
- **Rustlings**: https://github.com/rust-lang/rustlings (interactive exercises)
- **Rust By Example**: https://doc.rust-lang.org/rust-by-example/
- **Std Library Docs**: https://doc.rust-lang.org/std/

### 14.2 Community
- **Reddit**: r/rust
- **Discord**: https://discord.gg/rust-lang
- **Forum**: https://users.rust-lang.org/

### 14.3 Crates.io
- ค้นหา packages: https://crates.io
- เอกสาร packages: https://docs.rs

---

## สรุป Part 001

ในส่วนนี้เราได้เรียนรู้:
1. **ทำไมต้องเรียน Rust** - ความปลอดภัย, ประสิทธิภาพ, concurrency
2. **การติดตั้ง** ผ่าน rustup บน Linux/macOS/Windows
3. **Cargo** - ตัวจัดการ package และ build tool
4. **Hello World** และ println! macro ขั้นสูง
5. **โครงสร้างโปรเจกต์** และ Cargo.toml

ใน **Part 002** เราจะเรียนเรื่อง **ตัวแปร, ชนิดข้อมูล, และ Mutability** ซึ่งเป็นรากฐานสำคัญของ Rust

---

*[← กลับไปหน้าหลัก](../../README.md) | [Part 002: ตัวแปรและชนิดข้อมูล →](../part_002/README.md)*
