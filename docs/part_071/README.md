# Part 071: Performance Optimization ใน Rust + Actix-web

## บทนำ

การเพิ่มประสิทธิภาพ (Performance Optimization) เป็นหนึ่งในทักษะสำคัญที่นักพัฒนา Rust ต้องมี บทนี้จะครอบคลุมเทคนิคต่างๆ ตั้งแต่การวัดประสิทธิภาพ (Profiling) ไปจนถึงการปรับแต่งโค้ดในระดับต่ำ

## 1. Profiling ด้วย perf และ Flamegraph

### 1.1 ติดตั้ง Tools

```bash
# ติดตั้ง perf (Linux)
sudo apt-get install linux-tools-common linux-tools-generic

# ติดตั้ง flamegraph
cargo install flamegraph

# ติดตั้ง cargo-flamegraph
cargo install cargo-flamegraph
```

### 1.2 การ Build สำหรับ Profiling

```toml
# Cargo.toml
[profile.release]
debug = true  # เปิด debug symbols สำหรับ profiling

[profile.bench]
debug = true
```

### 1.3 ใช้ cargo-flamegraph

```bash
# Run flamegraph บน binary
cargo flamegraph --bin my-server

# Run flamegraph บน test
cargo flamegraph --test my_test

# บันทึก flamegraph เป็น SVG
cargo flamegraph -o flamegraph.svg --bin my-server
```

### 1.4 ตัวอย่าง: Profiling Web Server

```rust
// src/main.rs - server สำหรับ profiling
use actix_web::{web, App, HttpServer, HttpResponse, middleware};
use std::sync::Arc;
use tokio::sync::RwLock;

#[derive(Clone)]
struct AppState {
    data: Arc<RwLock<Vec<String>>>,
}

async fn slow_endpoint(state: web::Data<AppState>) -> HttpResponse {
    let data = state.data.read().await;
    
    // การประมวลผลที่ช้า (จะเห็นใน flamegraph)
    let result: Vec<String> = data
        .iter()
        .filter(|s| s.contains("important"))
        .map(|s| s.to_uppercase()) // clone ที่ไม่จำเป็น
        .collect();
    
    HttpResponse::Ok().json(result)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let state = web::Data::new(AppState {
        data: Arc::new(RwLock::new(
            (0..100000).map(|i| format!("item_{}_important", i)).collect()
        )),
    });
    
    HttpServer::new(move || {
        App::new()
            .app_data(state.clone())
            .route("/slow", web::get().to(slow_endpoint))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

## 2. Benchmark ด้วย Criterion Crate

### 2.1 Setup Criterion

```toml
# Cargo.toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "my_benchmark"
harness = false
```

### 2.2 เขียน Benchmark

```rust
// benches/my_benchmark.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion, BenchmarkId};
use std::time::Duration;

// ฟังก์ชันที่ต้องการ benchmark
fn process_data_slow(data: &[String]) -> Vec<String> {
    data.iter()
        .filter(|s| s.contains("important"))
        .map(|s| s.to_uppercase())
        .collect()
}

fn process_data_fast(data: &[String]) -> Vec<String> {
    let mut result = Vec::new();
    for s in data {
        if s.contains("important") {
            result.push(s.to_uppercase());
        }
    }
    result
}

// ใช้ references แทน clones
fn process_data_refs<'a>(data: &'a [String]) -> Vec<&'a str> {
    data.iter()
        .filter(|s| s.contains("important"))
        .map(|s| s.as_str())
        .collect()
}

fn bench_processing(c: &mut Criterion) {
    let data: Vec<String> = (0..1000)
        .map(|i| format!("item_{}_important", i))
        .collect();
    
    let mut group = c.benchmark_group("data_processing");
    group.measurement_time(Duration::from_secs(10));
    
    group.bench_function("slow_with_clones", |b| {
        b.iter(|| process_data_slow(black_box(&data)))
    });
    
    group.bench_function("fast_with_vec", |b| {
        b.iter(|| process_data_fast(black_box(&data)))
    });
    
    group.bench_function("refs_no_alloc", |b| {
        b.iter(|| process_data_refs(black_box(&data)))
    });
    
    group.finish();
}

// Benchmark ด้วย sizes ต่างๆ
fn bench_scaling(c: &mut Criterion) {
    let mut group = c.benchmark_group("scaling");
    
    for size in [100, 1000, 10000].iter() {
        let data: Vec<String> = (0..*size)
            .map(|i| format!("item_{}", i))
            .collect();
        
        group.bench_with_input(
            BenchmarkId::new("process", size),
            &data,
            |b, data| {
                b.iter(|| process_data_slow(black_box(data)))
            },
        );
    }
    
    group.finish();
}

criterion_group!(benches, bench_processing, bench_scaling);
criterion_main!(benches);
```

```bash
# Run benchmarks
cargo bench

# Run เฉพาะ benchmark ที่ต้องการ
cargo bench -- data_processing

# บันทึกผล baseline
cargo bench -- --save-baseline before_optimization

# เปรียบเทียบกับ baseline
cargo bench -- --baseline before_optimization
```

## 3. Memory Profiling

### 3.1 ใช้ Valgrind/Massif

```bash
# ติดตั้ง valgrind
sudo apt-get install valgrind

# Run memory profiling
valgrind --tool=massif --pages-as-heap=yes target/release/my-server

# แสดงผล massif
ms_print massif.out.* | head -100
```

### 3.2 ใช้ heaptrack

```bash
# ติดตั้ง heaptrack
sudo apt-get install heaptrack

# Profile
heaptrack target/release/my-server

# แสดงผลแบบ GUI
heaptrack_gui heaptrack.my-server.*.gz
```

### 3.3 ตัวอย่าง Memory-efficient Code

```rust
// src/memory_opt.rs

// แบบที่ 1: ใช้ String (allocates)
fn process_strings_allocating(data: Vec<String>) -> Vec<String> {
    data.into_iter()
        .map(|s| format!("processed: {}", s))  // allocation ทุก iteration
        .collect()
}

// แบบที่ 2: ใช้ String::with_capacity เพื่อ pre-allocate
fn process_strings_preallocated(data: Vec<String>) -> Vec<String> {
    let mut result = Vec::with_capacity(data.len());
    for s in data {
        let mut processed = String::with_capacity(s.len() + 12);
        processed.push_str("processed: ");
        processed.push_str(&s);
        result.push(processed);
    }
    result
}

// แบบที่ 3: ใช้ SmallVec สำหรับ arrays ขนาดเล็ก
use smallvec::SmallVec;

fn process_small_data(data: &[u32]) -> SmallVec<[u32; 8]> {
    // ถ้า data มีน้อยกว่า 8 elements จะไม่ allocate heap
    data.iter().map(|&x| x * 2).collect()
}

// แบบที่ 4: Object Pool เพื่อ reuse memory
use std::sync::Mutex;

struct BufferPool {
    pool: Mutex<Vec<Vec<u8>>>,
    buffer_size: usize,
}

impl BufferPool {
    fn new(buffer_size: usize) -> Self {
        BufferPool {
            pool: Mutex::new(Vec::new()),
            buffer_size,
        }
    }
    
    fn acquire(&self) -> Vec<u8> {
        let mut pool = self.pool.lock().unwrap();
        pool.pop().unwrap_or_else(|| Vec::with_capacity(self.buffer_size))
    }
    
    fn release(&self, mut buffer: Vec<u8>) {
        buffer.clear();  // ล้างข้อมูลแต่เก็บ capacity
        let mut pool = self.pool.lock().unwrap();
        if pool.len() < 100 {  // จำกัดจำนวน buffer ใน pool
            pool.push(buffer);
        }
    }
}
```

### 3.4 Cargo.toml สำหรับ Memory Optimization

```toml
[dependencies]
smallvec = "1.11"
bytes = "1.5"
# ใช้ jemalloc แทน system allocator
jemallocator = "0.5"
```

```rust
// src/main.rs - ใช้ jemalloc
#[cfg(not(target_env = "msvc"))]
use jemallocator::Jemalloc;

#[cfg(not(target_env = "msvc"))]
#[global_allocator]
static GLOBAL: Jemalloc = Jemalloc;
```

## 4. Avoiding Clones (Using References)

### 4.1 ปัญหาของ Unnecessary Clones

```rust
// แบบที่ไม่ดี: clone ทุกครั้ง
fn bad_process(data: Vec<String>) -> Vec<String> {
    let filtered: Vec<String> = data
        .iter()
        .filter(|s| s.len() > 5)
        .cloned()  // clone ทั้ง String
        .collect();
    
    filtered
        .iter()
        .map(|s| s.clone().to_uppercase())  // clone อีกครั้ง!
        .collect()
}

// แบบที่ดีกว่า: ใช้ references
fn good_process(data: &[String]) -> Vec<String> {
    data.iter()
        .filter(|s| s.len() > 5)
        .map(|s| s.to_uppercase())  // ไม่ต้อง clone ก่อน
        .collect()
}

// แบบที่ดีที่สุด: return references เมื่อทำได้
fn best_process<'a>(data: &'a [String]) -> Vec<&'a str> {
    data.iter()
        .filter(|s| s.len() > 5)
        .map(|s| s.as_str())  // ไม่ clone เลย
        .collect()
}
```

### 4.2 Cow (Clone-on-Write)

```rust
use std::borrow::Cow;

// Cow ช่วยให้เราเลือกได้ว่าจะ borrow หรือ own ขึ้นอยู่กับสถานการณ์
fn process_with_cow<'a>(s: &'a str, prefix: Option<&str>) -> Cow<'a, str> {
    match prefix {
        None => Cow::Borrowed(s),  // ไม่ต้อง allocate
        Some(p) => Cow::Owned(format!("{}{}", p, s)),  // allocate เมื่อจำเป็น
    }
}

// ตัวอย่างใน HTTP handler
use actix_web::{web, HttpResponse};

async fn process_query(query: web::Query<std::collections::HashMap<String, String>>) -> HttpResponse {
    let name = query.get("name")
        .map(|s| s.as_str())
        .unwrap_or("World");
    
    // ใช้ Cow เพื่อหลีกเลี่ยง allocation ที่ไม่จำเป็น
    let greeting: Cow<str> = if name.starts_with("Dr.") {
        Cow::Borrowed(name)  // ไม่ต้องแก้ไข
    } else {
        Cow::Owned(format!("Hello, {}!", name))  // ต้องสร้างใหม่
    };
    
    HttpResponse::Ok().body(greeting.into_owned())
}
```

### 4.3 Arc vs Clone

```rust
use std::sync::Arc;

// แบบที่ไม่ดี: clone data ทั้งก้อน
#[derive(Clone)]
struct HeavyData {
    data: Vec<Vec<u8>>,  // ข้อมูลหนักมาก
}

fn bad_share(data: HeavyData) -> Vec<HeavyData> {
    vec![data.clone(), data.clone(), data]  // clone ใหญ่มาก
}

// แบบที่ดี: ใช้ Arc เพื่อ share โดยไม่ clone
fn good_share(data: Arc<HeavyData>) -> Vec<Arc<HeavyData>> {
    vec![Arc::clone(&data), Arc::clone(&data), data]  // แค่ increment reference count
}
```

## 5. Zero-copy Parsing

### 5.1 ใช้ bytes::Bytes

```rust
use bytes::Bytes;
use actix_web::{web, HttpRequest, HttpResponse};

// แทนที่จะ copy bytes ออกมาเป็น Vec<u8>
async fn handle_upload_slow(body: web::Bytes) -> HttpResponse {
    let data: Vec<u8> = body.to_vec();  // copy!
    process_data(&data);
    HttpResponse::Ok().finish()
}

// ใช้ Bytes โดยตรง (zero-copy)
async fn handle_upload_fast(body: web::Bytes) -> HttpResponse {
    process_bytes(&body);  // ไม่ copy
    HttpResponse::Ok().finish()
}

fn process_bytes(data: &[u8]) {
    // ทำงานกับ data โดยตรง
    println!("Processing {} bytes", data.len());
}

fn process_data(data: &[u8]) {
    println!("Processing {} bytes", data.len());
}
```

### 5.2 Zero-copy JSON Parsing ด้วย serde

```rust
use serde::{Deserialize, Serialize};
use actix_web::{web, HttpResponse};

// ใช้ borrowed strings เมื่อเป็นไปได้
#[derive(Deserialize)]
struct RequestBorrowed<'a> {
    name: &'a str,      // ยืม string จาก input buffer
    message: &'a str,
}

// เปรียบเทียบกับแบบ owned
#[derive(Deserialize)]
struct RequestOwned {
    name: String,   // copy มาเป็นของตัวเอง
    message: String,
}

// ตัวอย่างการใช้งาน
async fn handle_request(body: web::Bytes) -> HttpResponse {
    // Deserialize โดยไม่ copy string data
    match serde_json::from_slice::<RequestBorrowed>(&body) {
        Ok(req) => {
            println!("Name: {}, Message: {}", req.name, req.message);
            HttpResponse::Ok().finish()
        }
        Err(e) => HttpResponse::BadRequest().body(e.to_string()),
    }
}
```

### 5.3 Custom Zero-copy Parser

```rust
// Parser ที่ทำงานบน slice โดยตรง
struct ZeroCopyParser<'a> {
    data: &'a [u8],
    position: usize,
}

impl<'a> ZeroCopyParser<'a> {
    fn new(data: &'a [u8]) -> Self {
        ZeroCopyParser { data, position: 0 }
    }
    
    fn read_until(&mut self, delimiter: u8) -> Option<&'a [u8]> {
        let start = self.position;
        while self.position < self.data.len() {
            if self.data[self.position] == delimiter {
                let slice = &self.data[start..self.position];
                self.position += 1;  // skip delimiter
                return Some(slice);
            }
            self.position += 1;
        }
        if self.position > start {
            Some(&self.data[start..])
        } else {
            None
        }
    }
    
    fn parse_csv_row(&mut self) -> Vec<&'a [u8]> {
        let mut fields = Vec::new();
        while let Some(field) = self.read_until(b',') {
            fields.push(field);
            if self.position >= self.data.len() {
                break;
            }
        }
        fields
    }
}

fn parse_csv_zero_copy(data: &[u8]) -> Vec<Vec<&[u8]>> {
    let mut parser = ZeroCopyParser::new(data);
    let mut rows = Vec::new();
    
    // แต่ละ row คือ references ไปยัง original data - ไม่มี copy
    while parser.position < parser.data.len() {
        rows.push(parser.parse_csv_row());
    }
    
    rows
}
```

## 6. SIMD Operations (Basic)

### 6.1 ใช้ std::simd (nightly) หรือ packed_simd

```rust
// Cargo.toml
// [dependencies]
// packed_simd_2 = "0.3"

// ตัวอย่าง SIMD สำหรับการ sum array
fn sum_scalar(data: &[f32]) -> f32 {
    data.iter().sum()
}

// SIMD version (manual unrolling)
fn sum_simd_manual(data: &[f32]) -> f32 {
    let mut sum = 0.0f32;
    let chunks = data.chunks_exact(8);
    let remainder = chunks.remainder();
    
    for chunk in chunks {
        // Process 8 elements ในครั้งเดียว (CPU อาจ vectorize อัตโนมัติ)
        sum += chunk[0] + chunk[1] + chunk[2] + chunk[3]
             + chunk[4] + chunk[5] + chunk[6] + chunk[7];
    }
    
    sum + remainder.iter().sum::<f32>()
}

// ใช้ auto-vectorization hints
#[target_feature(enable = "avx2")]
unsafe fn sum_avx2(data: &[f32]) -> f32 {
    // Compiler จะ vectorize อัตโนมัติด้วย AVX2
    data.iter().sum()
}
```

### 6.2 SIMD สำหรับ String Operations

```rust
// ค้นหา byte ใน string ด้วย SIMD
fn find_byte_simd(haystack: &[u8], needle: u8) -> Option<usize> {
    // ใช้ memchr crate ที่ใช้ SIMD ภายใน
    memchr::memchr(needle, haystack)
}

// Cargo.toml: memchr = "2.6"
```

## 7. Async Performance Tips

### 7.1 Avoid Blocking in Async Context

```rust
use actix_web::{web, HttpResponse};
use tokio::task;

// แบบที่ผิด: blocking ใน async context
async fn bad_handler() -> HttpResponse {
    // std::thread::sleep blocks the entire thread!
    std::thread::sleep(std::time::Duration::from_secs(1));
    HttpResponse::Ok().finish()
}

// แบบที่ถูก: ใช้ tokio::time::sleep
async fn good_handler() -> HttpResponse {
    tokio::time::sleep(std::time::Duration::from_secs(1)).await;
    HttpResponse::Ok().finish()
}

// แบบที่ถูกสำหรับ CPU-intensive tasks
async fn cpu_intensive_handler() -> HttpResponse {
    let result = task::spawn_blocking(|| {
        // ทำงาน CPU-intensive ใน thread แยก
        heavy_computation()
    }).await.unwrap();
    
    HttpResponse::Ok().json(result)
}

fn heavy_computation() -> Vec<u64> {
    (0..1_000_000).map(|i| i * i).collect()
}
```

### 7.2 Concurrent vs Sequential

```rust
use futures::future::join_all;
use actix_web::{web, HttpResponse};

// แบบ sequential (ช้า)
async fn sequential_requests() -> Vec<String> {
    let mut results = Vec::new();
    
    for i in 0..10 {
        let result = fetch_data(i).await;
        results.push(result);
    }
    
    results
}

// แบบ concurrent (เร็ว)
async fn concurrent_requests() -> Vec<String> {
    let futures: Vec<_> = (0..10)
        .map(|i| fetch_data(i))
        .collect();
    
    join_all(futures).await
}

async fn fetch_data(id: u32) -> String {
    tokio::time::sleep(std::time::Duration::from_millis(100)).await;
    format!("data_{}", id)
}

// Handler ที่ใช้ concurrent requests
async fn optimized_handler() -> HttpResponse {
    let data = concurrent_requests().await;
    HttpResponse::Ok().json(data)
}
```

### 7.3 Stream Processing

```rust
use actix_web::{web, HttpResponse, HttpRequest};
use futures::StreamExt;

// Process large payloads เป็น stream แทนการโหลดทั้งหมดก่อน
async fn stream_handler(
    req: HttpRequest,
    mut payload: web::Payload,
) -> HttpResponse {
    let mut total_size = 0;
    let mut line_count = 0;
    
    while let Some(chunk) = payload.next().await {
        match chunk {
            Ok(bytes) => {
                total_size += bytes.len();
                line_count += bytes.iter().filter(|&&b| b == b'\n').count();
            }
            Err(e) => {
                return HttpResponse::BadRequest().body(e.to_string());
            }
        }
    }
    
    HttpResponse::Ok().json(serde_json::json!({
        "total_bytes": total_size,
        "line_count": line_count,
    }))
}
```

## 8. Buffer Pooling

### 8.1 Simple Buffer Pool

```rust
use std::sync::{Arc, Mutex};

pub struct BufferPool {
    buffers: Arc<Mutex<Vec<Vec<u8>>>>,
    buffer_capacity: usize,
    max_pool_size: usize,
}

impl BufferPool {
    pub fn new(buffer_capacity: usize, max_pool_size: usize) -> Self {
        BufferPool {
            buffers: Arc::new(Mutex::new(Vec::with_capacity(max_pool_size))),
            buffer_capacity,
            max_pool_size,
        }
    }
    
    pub fn acquire(&self) -> PooledBuffer {
        let buffer = {
            let mut pool = self.buffers.lock().unwrap();
            pool.pop().unwrap_or_else(|| Vec::with_capacity(self.buffer_capacity))
        };
        
        PooledBuffer {
            buffer,
            pool: Arc::clone(&self.buffers),
            max_pool_size: self.max_pool_size,
        }
    }
}

pub struct PooledBuffer {
    buffer: Vec<u8>,
    pool: Arc<Mutex<Vec<Vec<u8>>>>,
    max_pool_size: usize,
}

impl PooledBuffer {
    pub fn as_mut(&mut self) -> &mut Vec<u8> {
        &mut self.buffer
    }
    
    pub fn as_slice(&self) -> &[u8] {
        &self.buffer
    }
}

impl Drop for PooledBuffer {
    fn drop(&mut self) {
        // Return buffer to pool เมื่อ drop
        let mut pool = self.pool.lock().unwrap();
        if pool.len() < self.max_pool_size {
            let mut buffer = std::mem::take(&mut self.buffer);
            buffer.clear();
            pool.push(buffer);
        }
    }
}

// ใช้ Buffer Pool ใน handler
use actix_web::{web, HttpResponse};
use std::sync::Arc;

async fn buffered_handler(
    pool: web::Data<Arc<BufferPool>>,
    body: web::Bytes,
) -> HttpResponse {
    let mut buffer = pool.acquire();
    
    // ใช้ buffer
    buffer.as_mut().extend_from_slice(&body);
    buffer.as_mut().push(b'\n');
    
    let response_data = buffer.as_slice().to_vec();
    
    // Buffer automatically returned to pool when dropped
    HttpResponse::Ok().body(response_data)
}
```

### 8.2 Tokio Buffer Pool สำหรับ Async

```rust
use tokio::sync::Semaphore;
use std::sync::Arc;

pub struct AsyncBufferPool {
    semaphore: Arc<Semaphore>,
    buffers: Arc<Mutex<Vec<Vec<u8>>>>,
    buffer_size: usize,
}

impl AsyncBufferPool {
    pub fn new(pool_size: usize, buffer_size: usize) -> Self {
        AsyncBufferPool {
            semaphore: Arc::new(Semaphore::new(pool_size)),
            buffers: Arc::new(Mutex::new(
                (0..pool_size).map(|_| Vec::with_capacity(buffer_size)).collect()
            )),
            buffer_size,
        }
    }
    
    pub async fn acquire(&self) -> AsyncPooledBuffer {
        let permit = self.semaphore.clone().acquire_owned().await.unwrap();
        let buffer = {
            let mut pool = self.buffers.lock().unwrap();
            pool.pop().unwrap_or_else(|| Vec::with_capacity(self.buffer_size))
        };
        
        AsyncPooledBuffer {
            buffer,
            pool: Arc::clone(&self.buffers),
            _permit: permit,
        }
    }
}

use std::sync::Mutex;

pub struct AsyncPooledBuffer {
    buffer: Vec<u8>,
    pool: Arc<Mutex<Vec<Vec<u8>>>>,
    _permit: tokio::sync::OwnedSemaphorePermit,
}

impl Drop for AsyncPooledBuffer {
    fn drop(&mut self) {
        let mut pool = self.pool.lock().unwrap();
        let mut buffer = std::mem::take(&mut self.buffer);
        buffer.clear();
        pool.push(buffer);
        // permit is dropped automatically, releasing the semaphore slot
    }
}
```

## 9. Practical: Optimize a Slow Endpoint

### 9.1 Initial Slow Version

```rust
use actix_web::{web, App, HttpServer, HttpResponse};
use serde::{Deserialize, Serialize};
use sqlx::{PgPool, Row};

#[derive(Serialize, Clone)]
struct Product {
    id: i32,
    name: String,
    price: f64,
    description: String,
    category: String,
    tags: Vec<String>,
}

// แบบช้า: หลาย queries, unnecessary clones
async fn get_products_slow(
    db: web::Data<PgPool>,
    query: web::Query<std::collections::HashMap<String, String>>,
) -> HttpResponse {
    // Query 1: ดึง products ทั้งหมด
    let products: Vec<Product> = sqlx::query_as!(
        Product,
        "SELECT id, name, price, description, category FROM products"
    )
    .fetch_all(db.get_ref())
    .await
    .unwrap_or_default()
    .into_iter()
    .map(|p| Product { ..p, tags: vec![] })
    .collect();
    
    // Query 2: ดึง tags แยก (N+1 problem!)
    let mut products_with_tags = Vec::new();
    for product in products {
        let tags: Vec<String> = sqlx::query!(
            "SELECT tag FROM product_tags WHERE product_id = $1",
            product.id
        )
        .fetch_all(db.get_ref())
        .await
        .unwrap_or_default()
        .into_iter()
        .map(|r| r.tag)
        .collect();
        
        products_with_tags.push(Product {
            tags,
            ..product  // clone ข้อมูลที่มีอยู่แล้ว
        });
    }
    
    // Filter (ทำใน application แทน database)
    let filter = query.get("category").cloned();
    let filtered: Vec<Product> = products_with_tags
        .into_iter()
        .filter(|p| {
            filter.as_ref().map_or(true, |f| &p.category == f)
        })
        .collect();
    
    HttpResponse::Ok().json(filtered)
}
```

### 9.2 Optimized Version

```rust
use actix_web::{web, HttpResponse};
use serde::{Deserialize, Serialize};
use sqlx::PgPool;

#[derive(Serialize, Deserialize)]
struct ProductQuery {
    category: Option<String>,
    page: Option<i64>,
    limit: Option<i64>,
}

#[derive(Serialize)]
struct ProductWithTags {
    id: i32,
    name: String,
    price: f64,
    category: String,
    tags: Vec<String>,
}

// แบบเร็ว: single query ด้วย JOIN, pagination, database-side filtering
async fn get_products_fast(
    db: web::Data<PgPool>,
    query: web::Query<ProductQuery>,
) -> HttpResponse {
    let page = query.page.unwrap_or(1).max(1);
    let limit = query.limit.unwrap_or(20).min(100);
    let offset = (page - 1) * limit;
    
    // Single query ด้วย LEFT JOIN และ array aggregation
    let result = sqlx::query!(
        r#"
        SELECT 
            p.id,
            p.name,
            p.price,
            p.category,
            COALESCE(array_agg(pt.tag) FILTER (WHERE pt.tag IS NOT NULL), '{}') as tags
        FROM products p
        LEFT JOIN product_tags pt ON p.id = pt.product_id
        WHERE ($1::text IS NULL OR p.category = $1)
        GROUP BY p.id, p.name, p.price, p.category
        ORDER BY p.id
        LIMIT $2 OFFSET $3
        "#,
        query.category.as_deref(),
        limit,
        offset
    )
    .fetch_all(db.get_ref())
    .await;
    
    match result {
        Ok(rows) => {
            let products: Vec<ProductWithTags> = rows
                .into_iter()
                .map(|row| ProductWithTags {
                    id: row.id,
                    name: row.name,
                    price: row.price,
                    category: row.category,
                    tags: row.tags.unwrap_or_default(),
                })
                .collect();
            
            HttpResponse::Ok().json(products)
        }
        Err(e) => {
            eprintln!("Database error: {}", e);
            HttpResponse::InternalServerError().finish()
        }
    }
}
```

### 9.3 เพิ่ม Caching Layer

```rust
use actix_web::{web, HttpResponse};
use redis::AsyncCommands;
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use sqlx::PgPool;

#[derive(Clone)]
struct AppState {
    db: PgPool,
    redis: Arc<redis::Client>,
}

async fn get_products_cached(
    state: web::Data<AppState>,
    query: web::Query<ProductQuery>,
) -> HttpResponse {
    let cache_key = format!(
        "products:{}:{}:{}",
        query.category.as_deref().unwrap_or("all"),
        query.page.unwrap_or(1),
        query.limit.unwrap_or(20)
    );
    
    // ลอง cache ก่อน
    if let Ok(mut conn) = state.redis.get_async_connection().await {
        if let Ok(cached) = conn.get::<_, String>(&cache_key).await {
            if let Ok(data) = serde_json::from_str::<Vec<ProductWithTags>>(&cached) {
                return HttpResponse::Ok()
                    .insert_header(("X-Cache", "HIT"))
                    .json(data);
            }
        }
    }
    
    // ไม่มี cache: query database
    let products = fetch_products_from_db(&state.db, &query).await;
    
    match products {
        Ok(data) => {
            // บันทึก cache
            if let Ok(mut conn) = state.redis.get_async_connection().await {
                if let Ok(json) = serde_json::to_string(&data) {
                    let _: Result<(), _> = conn
                        .set_ex(&cache_key, json, 300)  // cache 5 นาที
                        .await;
                }
            }
            
            HttpResponse::Ok()
                .insert_header(("X-Cache", "MISS"))
                .json(data)
        }
        Err(e) => {
            eprintln!("Error: {}", e);
            HttpResponse::InternalServerError().finish()
        }
    }
}

async fn fetch_products_from_db(
    db: &PgPool,
    query: &ProductQuery,
) -> Result<Vec<ProductWithTags>, sqlx::Error> {
    let page = query.page.unwrap_or(1).max(1);
    let limit = query.limit.unwrap_or(20).min(100);
    let offset = (page - 1) * limit;
    
    let rows = sqlx::query!(
        r#"
        SELECT 
            p.id,
            p.name,
            p.price,
            p.category,
            COALESCE(array_agg(pt.tag) FILTER (WHERE pt.tag IS NOT NULL), '{}') as tags
        FROM products p
        LEFT JOIN product_tags pt ON p.id = pt.product_id
        WHERE ($1::text IS NULL OR p.category = $1)
        GROUP BY p.id, p.name, p.price, p.category
        ORDER BY p.id
        LIMIT $2 OFFSET $3
        "#,
        query.category.as_deref(),
        limit,
        offset
    )
    .fetch_all(db)
    .await?;
    
    Ok(rows.into_iter().map(|row| ProductWithTags {
        id: row.id,
        name: row.name,
        price: row.price,
        category: row.category,
        tags: row.tags.unwrap_or_default(),
    }).collect())
}
```

### 9.4 Complete Performance-Optimized Server

```rust
// src/main.rs - Complete example
use actix_web::{web, App, HttpServer, middleware};
use sqlx::postgres::PgPoolOptions;
use std::sync::Arc;

mod handlers;
mod models;
mod cache;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    env_logger::init();
    
    // Database pool ที่ optimize แล้ว
    let db_pool = PgPoolOptions::new()
        .max_connections(20)
        .min_connections(5)
        .acquire_timeout(std::time::Duration::from_secs(3))
        .idle_timeout(std::time::Duration::from_secs(600))
        .connect(&std::env::var("DATABASE_URL").expect("DATABASE_URL required"))
        .await
        .expect("Failed to create pool");
    
    // Redis client
    let redis_client = redis::Client::open(
        std::env::var("REDIS_URL").unwrap_or_else(|_| "redis://127.0.0.1/".to_string())
    ).expect("Failed to create Redis client");
    
    let state = web::Data::new(AppState {
        db: db_pool,
        redis: Arc::new(redis_client),
    });
    
    HttpServer::new(move || {
        App::new()
            .app_data(state.clone())
            .app_data(
                web::JsonConfig::default()
                    .limit(1_048_576)  // 1MB limit
            )
            .wrap(middleware::Compress::default())  // Gzip compression
            .wrap(middleware::Logger::default())
            .route("/products", web::get().to(get_products_cached))
    })
    .workers(num_cpus::get())  // ใช้ทุก CPU core
    .keep_alive(std::time::Duration::from_secs(75))
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

### 9.5 Cargo.toml สำหรับ Production

```toml
[package]
name = "optimized-api"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio-rustls", "macros"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
redis = { version = "0.23", features = ["tokio-comp"] }
env_logger = "0.10"
num_cpus = "1.16"
bytes = "1.5"
futures = "0.3"
memchr = "2.6"

[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[profile.release]
opt-level = 3
lto = true          # Link Time Optimization
codegen-units = 1   # Better optimization (slower compile)
panic = "abort"     # เล็กกว่า binary
strip = true        # ลบ debug symbols

[[bench]]
name = "api_bench"
harness = false
```

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Profiling** - ใช้ flamegraph เพื่อหา bottleneck
2. **Criterion** - เขียน benchmarks ที่น่าเชื่อถือ
3. **Memory** - ลด allocations ด้วย references และ object pooling
4. **Zero-copy** - ทำงานกับข้อมูลโดยตรงโดยไม่ copy
5. **SIMD** - ใช้ CPU vector instructions
6. **Async tips** - หลีกเลี่ยง blocking, ใช้ concurrent execution
7. **Buffer pooling** - reuse buffers เพื่อลด GC pressure

---

[⬅️ Part 070: Advanced Patterns](../part_070/README.md) | [➡️ Part 072: Load Testing](../part_072/README.md)
