# Part 100: Career Path and Next Steps 🎓

## ยินดีด้วย! คุณทำสำเร็จแล้ว! 🎉

การเดินทาง 100 บทของเราได้สิ้นสุดลงแล้ว แต่การเดินทางสู่ความเป็นนักพัฒนา Rust ระดับโลกของคุณเพิ่งเริ่มต้นขึ้น!

---

## 1. สรุปสิ่งที่เราได้เรียนรู้ (Summary of Learning)

### Phase 1: พื้นฐาน Rust (บทที่ 001-020)

เราเริ่มต้นจากศูนย์และสร้างรากฐานที่แข็งแกร่ง:

```
✅ การติดตั้งและตั้งค่า Rust environment
✅ Ownership, Borrowing, และ Lifetimes - หัวใจของ Rust
✅ Types, Structs, Enums, และ Pattern Matching
✅ Error Handling ด้วย Result<T, E> และ Option<T>
✅ Traits และ Generics
✅ Closures และ Functional Programming
✅ Collections: Vec, HashMap, HashSet
✅ Modules, Crates, และ Package Management
```

### Phase 2: Actix-web (บทที่ 021-050)

เราสร้าง web applications ด้วย Rust:

```
✅ HTTP Server ด้วย Actix-web
✅ Routing และ Handlers
✅ Middleware
✅ JSON serialization/deserialization ด้วย Serde
✅ Database integration ด้วย SQLx
✅ Authentication และ JWT
✅ WebSockets
✅ File uploads
✅ Testing
```

### Phase 3: ขั้นสูง (บทที่ 051-080)

เราเจาะลึกสู่ features ขั้นสูง:

```
✅ Async/Await และ Tokio runtime
✅ Concurrency patterns
✅ Performance optimization
✅ Caching ด้วย Redis
✅ Message queues
✅ Microservices architecture
✅ GraphQL
✅ gRPC
```

### Phase 4: Expert Level (บทที่ 081-100)

เราถึงระดับ Expert:

```
✅ Advanced Async Patterns (join!, try_join!, select!)
✅ Memory Management (Stack/Heap, Custom Allocators)
✅ Unsafe Rust และ FFI
✅ WebAssembly
✅ Embedded Systems
✅ CLI Tools ด้วย Clap
✅ Plugin System Architecture
✅ Capstone Project: Social Platform
```

---

## 2. Rust Job Market (ตลาดงาน Rust)

### ทำไม Rust ถึงเป็นที่ต้องการสูง

```
📈 การเติบโตของตลาด Rust:
   - Stack Overflow Survey: ภาษาที่คนชอบมากที่สุด 8 ปีติดต่อกัน (2016-2024)
   - GitHub: top 10 programming languages by growth
   - Linux Kernel: ยอมรับ Rust เป็นภาษาที่ 2
   - Windows: Microsoft เริ่มเขียน Windows ใน Rust
   - Android: Google ใช้ Rust สำหรับ Android OS components
   - AWS, Meta, Cloudflare, Discord: ล้วนใช้ Rust ใน production
```

### ตำแหน่งงานที่ใช้ Rust

```rust
// ตัวอย่างทักษะที่ควรมีสำหรับแต่ละตำแหน่ง

struct RustJobRoles {
    systems_engineer: Vec<&'static str>,
    backend_engineer: Vec<&'static str>,
    blockchain_developer: Vec<&'static str>,
    embedded_engineer: Vec<&'static str>,
    security_engineer: Vec<&'static str>,
}

fn get_required_skills() -> RustJobRoles {
    RustJobRoles {
        systems_engineer: vec![
            "Rust (advanced)", "C/C++", "Linux kernel",
            "Memory management", "Concurrency",
            "Performance profiling", "LLVM"
        ],
        backend_engineer: vec![
            "Rust (intermediate+)", "Actix-web/Axum",
            "PostgreSQL", "Redis", "Docker",
            "Microservices", "API design"
        ],
        blockchain_developer: vec![
            "Rust (advanced)", "Solana/Near/Substrate",
            "Cryptography", "Distributed systems",
            "Smart contracts", "WebAssembly"
        ],
        embedded_engineer: vec![
            "Rust (intermediate)", "no_std",
            "ARM Cortex-M", "Embedded HAL",
            "RTOS", "Hardware interfaces"
        ],
        security_engineer: vec![
            "Rust (advanced)", "Cryptography",
            "Secure coding", "Fuzzing",
            "Vulnerability research", "Binary analysis"
        ],
    }
}
```

### เงินเดือน (ประมาณการ)

```
🌍 Global Market (USD/year):
   Junior Rust Developer:    $70,000 - $100,000
   Mid-level Rust Developer: $100,000 - $150,000
   Senior Rust Developer:    $150,000 - $250,000+
   Principal Engineer:       $200,000 - $400,000+

🇹🇭 Thailand Market (THB/month):
   Junior:   40,000 - 70,000
   Mid:      70,000 - 120,000
   Senior:   120,000 - 200,000+
   Remote:   สามารถสมัครงาน international ได้!
```

---

## 3. การ Contribute ให้ Open Source

### เริ่มต้น contribute

```bash
# Step 1: หา project ที่คุณสนใจ
# https://github.com/rust-lang/rust (compiler)
# https://github.com/tokio-rs/tokio (async runtime)
# https://github.com/actix/actix-web (web framework)
# https://github.com/serde-rs/serde (serialization)
# https://github.com/clap-rs/clap (CLI)

# Step 2: Fork และ Clone
git clone https://github.com/YOUR_USERNAME/PROJECT.git
cd PROJECT

# Step 3: ค้นหา "good first issue"
# ไปที่ Issues tab บน GitHub
# Filter by: good-first-issue, help-wanted

# Step 4: สร้าง branch
git checkout -b fix/issue-123-description

# Step 5: แก้ไข และ test
cargo test
cargo clippy
cargo fmt

# Step 6: Submit PR
git push origin fix/issue-123-description
# เปิด Pull Request บน GitHub
```

### Guidelines สำหรับ Good Contributions

```rust
// การเขียน PR ที่ดี

fn write_good_pr() {
    let pr = PullRequest {
        title: "Fix: descriptive one-line summary",
        description: r#"
## Problem
อธิบายปัญหาที่แก้ไข

## Solution
อธิบายวิธีการแก้ไข

## Testing
- [ ] เพิ่ม unit tests
- [ ] เพิ่ม integration tests
- [ ] ทดสอบ edge cases

## Checklist
- [ ] Code follows project style
- [ ] Tests pass
- [ ] Documentation updated
- [ ] CHANGELOG updated
"#.to_string(),
        commits: vec![
            "descriptive commit message 1",
            "descriptive commit message 2",
        ],
    };
    
    println!("Great PR: {:?}", pr);
}
```

### Projects ที่แนะนำสำหรับ beginners

```markdown
## Beginner-friendly Rust Projects

1. **rustlings** (https://github.com/rust-lang/rustlings)
   - แก้ไข exercise files
   - เพิ่ม hints
   - แปลเป็นภาษาอื่น

2. **crates.io** (https://github.com/rust-lang/crates.io)
   - Bug fixes
   - UI improvements
   - Documentation

3. **The Rust Book** (https://github.com/rust-lang/book)
   - แก้ typos
   - ปรับปรุง examples
   - แปลภาษา

4. **rust-clippy** (https://github.com/rust-lang/rust-clippy)
   - เพิ่ม lint rules
   - Bug fixes

5. **Your own crates**
   - เผยแพร่ library ที่คุณสร้าง
   - สร้าง utility tools
```

---

## 4. การสร้าง Portfolio

### สิ่งที่ควรมีใน Portfolio

```
🎯 Portfolio Checklist:

GitHub Profile:
├── README.md ที่น่าสนใจ (ใช้ GitHub profile README)
├── Pinned repositories (5-6 repos ที่ดีที่สุด)
├── Consistent contribution activity
├── Good commit messages
└── Complete documentation

Projects ที่ควรมี:
├── 1. Web API (Actix-web + PostgreSQL + Redis)
├── 2. CLI Tool (ประโยชน์จริง)
├── 3. Library/Crate บน crates.io
├── 4. Contribution ให้ open source
└── 5. Capstone Project (Social Platform)

Documentation:
├── README.md ทุก project
├── API documentation
├── Architecture diagrams
└── Performance benchmarks
```

### ตัวอย่าง Project Ideas สำหรับ Portfolio

```rust
// Portfolio project ideas

fn portfolio_ideas() -> Vec<ProjectIdea> {
    vec![
        ProjectIdea {
            title: "URL Shortener Service",
            description: "High-performance URL shortener with analytics",
            tech_stack: vec!["Actix-web", "Redis", "PostgreSQL"],
            difficulty: "Beginner",
        },
        ProjectIdea {
            title: "File Sync Tool",
            description: "CLI tool for syncing files across machines (like rsync)",
            tech_stack: vec!["Tokio", "async I/O", "compression"],
            difficulty: "Intermediate",
        },
        ProjectIdea {
            title: "Static Site Generator",
            description: "Build a fast SSG like Hugo but in Rust",
            tech_stack: vec!["Markdown parser", "Template engine", "CLI"],
            difficulty: "Intermediate",
        },
        ProjectIdea {
            title: "Distributed Key-Value Store",
            description: "Toy version of Redis with replication",
            tech_stack: vec!["Tokio", "Raft consensus", "Networking"],
            difficulty: "Advanced",
        },
        ProjectIdea {
            title: "Browser-based Game in WASM",
            description: "2D game compiled to WebAssembly",
            tech_stack: vec!["WASM", "wasm-bindgen", "Canvas API"],
            difficulty: "Intermediate",
        },
        ProjectIdea {
            title: "Blockchain Implementation",
            description: "Simple blockchain with proof-of-work",
            tech_stack: vec!["Cryptography", "Networking", "Consensus"],
            difficulty: "Advanced",
        },
    ]
}
```

---

## 5. หนังสือที่แนะนำ (Recommended Books)

### Essential Reading

```
📚 Must-Read Books:

1. "The Rust Programming Language" (official book)
   - ผู้เขียน: Steve Klabnik และ Carol Nichols
   - ออนไลน์ฟรี: https://doc.rust-lang.org/book/
   - เหมาะสำหรับ: ผู้เริ่มต้น - intermediate
   - เนื้อหา: comprehensive guide to Rust

2. "Programming Rust" (O'Reilly)
   - ผู้เขียน: Jim Blandy, Jason Orendorff, Leonora F. S. Tindall
   - ระดับ: intermediate - advanced
   - เนื้อหา: in-depth systems programming concepts
   - ★★★★★ (แนะนำมาก)

3. "Rust for Rustaceans" (No Starch Press)
   - ผู้เขียน: Jon Gjengset
   - ระดับ: advanced
   - เนื้อหา: advanced Rust patterns และ concepts
   - เหมาะหลังจากรู้ Rust พื้นฐานแล้ว

4. "Zero To Production In Rust"
   - ผู้เขียน: Luca Palmieri
   - ระดับ: intermediate
   - เนื้อหา: production-ready web service
   - แนะนำมากสำหรับ backend developers

5. "Hands-on Rust"
   - ผู้เขียน: Herbert Wolverson
   - ระดับ: beginner - intermediate
   - เนื้อหา: game development in Rust

6. "Rust Atomics and Locks"
   - ผู้เขียน: Mara Bos
   - ระดับ: advanced
   - ออนไลน์ฟรี: https://marabos.nl/atomics/
   - เนื้อหา: concurrency, atomics, memory ordering
```

---

## 6. Community Resources

### Official Resources

```
🌐 Official Channels:
├── The Rust Programming Language: https://www.rust-lang.org/
├── The Rust Reference: https://doc.rust-lang.org/reference/
├── Rust Standard Library: https://doc.rust-lang.org/std/
├── crates.io: https://crates.io/
├── Rust Edition Guide: https://doc.rust-lang.org/edition-guide/
└── Rustonomicon (unsafe): https://doc.rust-lang.org/nomicon/
```

### Community Forums

```
💬 Where to Get Help:

Reddit:
├── r/rust: https://www.reddit.com/r/rust/
└── r/learnrust: https://www.reddit.com/r/learnrust/

Discord:
└── Official Rust Discord: https://discord.gg/rust-lang
    ├── #beginners
    ├── #help
    ├── #jobs
    └── #showcase

Forums:
├── users.rust-lang.org (official forum)
└── internals.rust-lang.org (compiler internals)

Zulip:
└── rust-lang.zulipchat.com (official)

Stack Overflow:
└── Tag: [rust]
```

### Learning Resources

```
📹 Video/Course Resources:

YouTube Channels:
├── Jon Gjengset (แนะนำมาก): https://www.youtube.com/@jonhoo
├── Let's Get Rusty: https://www.youtube.com/@letsgetrusty
└── No Boilerplate: https://www.youtube.com/@NoBoilerplate

Courses:
├── Rustlings: https://github.com/rust-lang/rustlings
├── Exercism Rust Track: https://exercism.org/tracks/rust
└── Comprehensive Rust (Google): https://google.github.io/comprehensive-rust/

Blogs:
├── This Week in Rust: https://this-week-in-rust.org/
├── Rust Blog: https://blog.rust-lang.org/
└── Without boats: https://without.boats/
```

### Thai Rust Community

```
🇹🇭 Rust ในประเทศไทย:

Facebook Groups:
└── Rust Programming Thailand

Discord Thai Dev Communities:
└── หลาย server ที่มี Rust channels

Meetups:
└── Bangkok Rust Meetup (ตามข่าวใน meetup.com)

Contributing Thai Content:
└── แปล Rust documentation เป็นภาษาไทย!
    https://github.com/rust-lang-th
```

---

## 7. Advanced Topics to Explore

### เส้นทางการเรียนรู้ต่อ

```rust
// Advanced topics roadmap

struct LearningPath {
    systems: Vec<Topic>,
    web: Vec<Topic>,
    embedded: Vec<Topic>,
    blockchain: Vec<Topic>,
}

fn advanced_topics() -> LearningPath {
    LearningPath {
        systems: vec![
            Topic { name: "Kernel Development", resources: "linux-kernel.git" },
            Topic { name: "OS Development", resources: "blog_os by Philipp Oppermann" },
            Topic { name: "Compiler Design", resources: "rustc dev guide" },
            Topic { name: "LLVM Integration", resources: "inkwell crate" },
        ],
        web: vec![
            Topic { name: "Axum Framework", resources: "tokio-rs/axum" },
            Topic { name: "Tonic (gRPC)", resources: "hyperium/tonic" },
            Topic { name: "SeaORM", resources: "SeaQL/sea-orm" },
            Topic { name: "Leptos (Frontend)", resources: "leptos-rs/leptos" },
            Topic { name: "Dioxus", resources: "DioxusLabs/dioxus" },
        ],
        embedded: vec![
            Topic { name: "Embassy (async embedded)", resources: "embassy-rs/embassy" },
            Topic { name: "RTIC Framework", resources: "rtic-rs/rtic" },
            Topic { name: "Drone OS", resources: "drone-os/drone" },
            Topic { name: "Tock OS", resources: "tock/tock" },
        ],
        blockchain: vec![
            Topic { name: "Solana Programs", resources: "solana-labs/solana" },
            Topic { name: "Substrate (Polkadot)", resources: "paritytech/substrate" },
            Topic { name: "NEAR Protocol", resources: "near/nearcore" },
            Topic { name: "Ethereum (reth)", resources: "paradigmxyz/reth" },
        ],
    }
}
```

### Deep Dive Topics

```
🔬 Topics สำหรับ Deep Dive:

1. Async Runtime Internals
   - Tokio source code
   - Pin และ Unpin
   - Waker mechanism
   - Executor design

2. Memory Model
   - Rust memory model
   - MIRI (MIR interpreter)
   - Stacked borrows
   - Memory ordering

3. Compiler Internals
   - MIR (Mid-level IR)
   - Borrow checker algorithm (NLL)
   - Type inference
   - Trait resolution

4. Performance Engineering
   - Flamegraph profiling
   - Criterion benchmarking
   - SIMD intrinsics
   - Cache-friendly data structures

5. Formal Verification
   - Prusti (automated verification)
   - Kani (model checking)
   - Creusot (deductive verification)
```

---

## 8. Professional Certifications

### Certifications ที่เกี่ยวข้อง

```
📜 Relevant Certifications:

Cloud Platforms:
├── AWS Certified Developer
├── Google Cloud Professional Developer
└── Azure Developer Associate

Systems/Infrastructure:
├── Linux Foundation: LFCS (Linux Foundation Certified SysAdmin)
├── CKA/CKAD (Kubernetes)
└── HashiCorp Certified: Terraform Associate

Security:
├── OSCP (Offensive Security)
├── CEH (Certified Ethical Hacker)
└── Security+

Note: ยังไม่มี official "Rust certification" แต่
      Portfolio และ contributions สำคัญกว่า certifications!
```

### การสร้าง Technical Reputation

```rust
// Building technical reputation

struct TechnicalReputation {
    actions: Vec<ReputationAction>,
}

fn build_reputation() -> Vec<&'static str> {
    vec![
        // Writing
        "เขียน blog posts เกี่ยวกับ Rust",
        "ตอบคำถามบน Stack Overflow",
        "เขียน tutorial บน dev.to หรือ Medium",
        "สร้าง YouTube channel สอน Rust",
        
        // Speaking
        "Present ที่ local meetups",
        "ส่ง talk proposal ไปงาน conferences",
        "สร้าง lightning talks",
        
        // Contributing
        "Contribute to popular crates",
        "รายงาน bugs ที่ดี",
        "Review pull requests",
        "สร้างและ maintain crates",
        
        // Community
        "ช่วยตอบคำถามใน Discord",
        "Mentor junior developers",
        "Organize local Rust meetups",
    ]
}
```

---

## 9. Final Project Ideas

### Projects ที่ท้าทายระดับสูง

```rust
// Advanced final project ideas

enum FinalProjectIdea {
    // Performance-critical
    GameEngine {
        description: "2D/3D game engine ใน Rust",
        examples: vec!["bevy", "ggez", "macroquad"],
    },
    
    // Systems
    DatabaseEngine {
        description: "Key-value store หรือ SQL engine",
        examples: vec!["TiKV", "toydb"],
    },
    
    // Web
    FullStackApp {
        description: "Full-stack app ด้วย Leptos/Dioxus + Axum",
        features: vec!["SSR", "WASM frontend", "REST/GraphQL API"],
    },
    
    // Distributed
    MessageBroker {
        description: "Message broker คล้าย Kafka แต่ขนาดเล็กกว่า",
        concepts: vec!["Partitioning", "Replication", "Consumer groups"],
    },
    
    // Security
    PasswordManager {
        description: "Local password manager ที่ปลอดภัย",
        features: vec!["AES encryption", "PBKDF2", "CLI + TUI"],
    },
    
    // Creative
    ProgrammingLanguage {
        description: "สร้างภาษาโปรแกรมมิ่งของตัวเอง",
        components: vec!["Lexer", "Parser", "AST", "Interpreter/Compiler"],
    },
}
```

---

## 10. Certificate of Completion

```
╔══════════════════════════════════════════════════════════════╗
║                                                              ║
║          🎓 ใบประกาศนียบัตรความสำเร็จ 🎓                    ║
║                                                              ║
║     CERTIFICATE OF COMPLETION                                ║
║     Rust + Actix-web Complete Course                         ║
║                                                              ║
║     ขอมอบให้แก่ผู้ที่ได้ศึกษาและฝึกฝน                       ║
║     การพัฒนาโปรแกรมด้วยภาษา Rust ครบทั้ง 100 บท            ║
║                                                              ║
║     หัวข้อที่ได้เรียนรู้:                                    ║
║     ✓ Rust Fundamentals                                      ║
║     ✓ Actix-web Web Development                              ║
║     ✓ Database Integration                                   ║
║     ✓ Authentication & Security                              ║
║     ✓ Real-time Applications                                 ║
║     ✓ Advanced Async Programming                             ║
║     ✓ Memory Management                                      ║
║     ✓ Unsafe Rust & FFI                                      ║
║     ✓ WebAssembly                                            ║
║     ✓ Embedded Systems                                       ║
║     ✓ CLI Tools                                              ║
║     ✓ Plugin Architecture                                    ║
║     ✓ Capstone Project                                       ║
║                                                              ║
║     วันที่สำเร็จ: __________________________                  ║
║                                                              ║
║     "Fearless Programming with Rust"                         ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

---

## ข้อความสุดท้าย: คุณเป็น Rustacean แล้ว! 🦀

```rust
// A message for you

use std::future::Future;

struct YourJourney {
    chapters_completed: u32,
    concepts_mastered: Vec<String>,
    code_written: u64,
    bugs_fixed: u32,
}

impl YourJourney {
    fn final_message(&self) -> String {
        format!(r#"
สวัสดีนักพัฒนา Rust ผู้ยอดเยี่ยม!

คุณได้เรียนจบ {} บทแล้ว นั่นหมายความว่าคุณได้ผ่านการเรียนรู้
จาก "Hello, World!" ไปสู่การสร้าง production-ready applications
ที่ใช้ async/await, WebSockets, และ microservices architecture

สิ่งที่คุณมีตอนนี้:
• ความเข้าใจ Rust ในระดับลึก
• ทักษะการสร้าง web applications
• ประสบการณ์กับ real-world patterns
• Portfolio projects ที่แสดงความสามารถ

จำไว้เสมอว่า:
"The compiler is your friend, not your enemy"
เมื่อ Rust บอกว่า error มันกำลังช่วยให้โค้ดคุณดีขึ้น

ก้าวต่อไปจากที่นี่:
1. สร้าง projects ต่อเนื่อง
2. Contribute ให้ open source
3. แบ่งปันความรู้กับชุมชน
4. ไม่หยุดเรียนรู้

ยินดีต้อนรับสู่ Rustacean Community! 🦀
        "#, self.chapters_completed)
    }
}

fn main() {
    let journey = YourJourney {
        chapters_completed: 100,
        concepts_mastered: vec![
            "Ownership".into(),
            "Borrowing".into(),
            "Async/Await".into(),
            "Actix-web".into(),
            "And so much more!".into(),
        ],
        code_written: 50_000, // lines
        bugs_fixed: 42,
    };
    
    println!("{}", journey.final_message());
    
    // Your future is bright!
    let future: std::pin::Pin<Box<dyn Future<Output = Success>>> = Box::pin(async {
        // คุณกำลังเดินหน้าสู่ความสำเร็จ!
        Success {
            career: "Rust Developer",
            impact: "Building fast, safe, reliable software",
            community: "Part of the global Rust community",
        }
    });
    
    println!("🚀 Keep coding, keep growing!");
    println!("🦀 Ferris is proud of you!");
}

#[derive(Debug)]
struct Success {
    career: &'static str,
    impact: &'static str,
    community: &'static str,
}
```

---

## Quick Reference: คำสั่งที่ใช้บ่อย

```bash
# Cargo commands
cargo new project_name          # สร้าง project ใหม่
cargo build                     # build project
cargo build --release           # build optimized
cargo run                       # run project
cargo test                      # รัน tests
cargo test -- --nocapture       # รัน tests พร้อม output
cargo doc --open                # สร้างและเปิด documentation
cargo clippy                    # lint checker
cargo fmt                       # format code
cargo update                    # update dependencies
cargo audit                     # check security vulnerabilities
cargo bench                     # run benchmarks

# Package management
cargo add serde                 # เพิ่ม dependency
cargo add serde --features derive
cargo remove serde              # ลบ dependency

# Workspace
cargo build --workspace         # build ทุก crates
cargo test --workspace          # test ทุก crates

# Cross compilation
rustup target add thumbv7em-none-eabihf  # เพิ่ม target
cargo build --target thumbv7em-none-eabihf

# Nightly features
rustup install nightly
cargo +nightly build

# Performance
cargo flamegraph               # profiling (requires cargo-flamegraph)
cargo criterion                # benchmarking

# Documentation
cargo doc                      # generate docs
cargo doc --open               # generate and open docs

# Publishing to crates.io
cargo login                    # login
cargo publish                  # publish crate
cargo publish --dry-run        # test before publishing
```

---

## สุดท้าย: ความคิดสร้างสรรค์ไม่มีขีดจำกัด

```
"Rust doesn't guarantee you won't make bugs,
 but it does guarantee a whole class of bugs
 that plague other languages simply cannot happen."

คุณไม่ได้แค่เรียน programming language
คุณได้เรียนวิธีคิดแบบ Rust:
- คิดถึง ownership และ lifetime
- คิดถึง safety ก่อน
- คิดถึง performance อย่างมีสติ
- คิดถึง correctness โดย design

ขอให้โชคดีในเส้นทาง Rust ของคุณ!
และจำไว้ว่า: ชุมชน Rust ยินดีต้อนรับเสมอ 🦀❤️

- Ferris the Crab
```

---

[← Part 099](../part_099/README.md) | [กลับไปหน้าแรก ↑](../../README.md)

---

*จบหลักสูตร Rust + Actix-web ครบ 100 บท*
*สร้างด้วยความรักต่อภาษา Rust และชุมชนนักพัฒนาไทย 🇹🇭*
