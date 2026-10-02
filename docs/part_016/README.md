# Part 016: Async/Await และ Tokio

## บทนำ

ใน Rust การเขียนโปรแกรมแบบ asynchronous (async) เป็นวิธีที่มีประสิทธิภาพสูงในการจัดการงานที่ต้องรอ เช่น การอ่านไฟล์ การเชื่อมต่อเครือข่าย หรือการรอ database ในบทนี้เราจะเรียนรู้ async/await syntax และ Tokio runtime ซึ่งเป็น async runtime ที่นิยมใช้มากที่สุดใน Rust

---

## 1. async/await Syntax พื้นฐาน

```rust
// การประกาศ async function
async fn say_hello() {
    println!("Hello from async function!");
}

// async function ที่ return ค่า
async fn add(a: i32, b: i32) -> i32 {
    a + b
}

// การ await ค่าจาก async function
async fn main_example() {
    say_hello().await;
    let result = add(5, 3).await;
    println!("Result: {}", result);
}
```

### ความแตกต่างระหว่าง sync และ async

```rust
use std::time::Duration;
use tokio::time::sleep;

// Synchronous - block thread
fn sync_sleep() {
    std::thread::sleep(Duration::from_secs(1));
    println!("sync done");
}

// Asynchronous - ไม่ block thread
async fn async_sleep() {
    sleep(Duration::from_secs(1)).await;
    println!("async done");
}

// เปรียบเทียบการทำงาน
#[tokio::main]
async fn main() {
    // sync: ใช้เวลา 3 วินาที (sequential)
    // sync_sleep();
    // sync_sleep();
    // sync_sleep();
    
    // async: ใช้เวลา ~1 วินาที (concurrent)
    tokio::join!(
        async_sleep(),
        async_sleep(),
        async_sleep()
    );
}
```

---

## 2. Future Trait

`Future` เป็น trait หลักของ async ใน Rust ทุก async function return type ที่ implement `Future`

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

// Custom Future implementation
struct DelayedValue {
    value: i32,
    ready: bool,
}

impl Future for DelayedValue {
    type Output = i32;
    
    fn poll(mut self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.ready {
            Poll::Ready(self.value)
        } else {
            self.ready = true;
            Poll::Pending
        }
    }
}

// ใช้ custom future
async fn use_custom_future() {
    let future = DelayedValue { value: 42, ready: false };
    // หมายเหตุ: ตัวอย่างนี้เพื่อแสดง concept
    // ในการใช้งานจริงต้องใช้ runtime เพื่อ poll future
    println!("Future output type: i32");
}

// Future chaining
async fn fetch_data() -> String {
    "data from network".to_string()
}

async fn process_data(data: String) -> String {
    format!("processed: {}", data)
}

async fn chain_futures() {
    let data = fetch_data().await;
    let result = process_data(data).await;
    println!("{}", result);
}
```

---

## 3. tokio::main และ tokio::spawn

### tokio::main Macro

```rust
// วิธีใช้ tokio::main
#[tokio::main]
async fn main() {
    println!("Running in Tokio runtime");
}

// กำหนด worker threads
#[tokio::main(worker_threads = 4)]
async fn main_with_threads() {
    println!("Running with 4 worker threads");
}

// Current thread runtime (single thread)
#[tokio::main(flavor = "current_thread")]
async fn main_single_thread() {
    println!("Running in single thread");
}
```

### tokio::spawn - สร้าง Task

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    // spawn task แบบง่าย
    let handle = tokio::spawn(async {
        println!("Task running in background");
        42 // return value
    });
    
    // รอผลลัพธ์
    let result = handle.await.expect("Task panicked");
    println!("Task returned: {}", result);
    
    // spawn หลาย tasks
    let mut handles = vec![];
    
    for i in 0..5 {
        let handle = tokio::spawn(async move {
            // move ต้องใส่เพราะ i เป็น captured variable
            println!("Task {} started", i);
            tokio::time::sleep(tokio::time::Duration::from_millis(100 * i)).await;
            println!("Task {} finished", i);
            i * 2
        });
        handles.push(handle);
    }
    
    // รอทุก task เสร็จ
    for handle in handles {
        let result = handle.await.expect("Task failed");
        println!("Result: {}", result);
    }
}
```

### spawn_blocking - สำหรับ CPU-intensive tasks

```rust
#[tokio::main]
async fn main() {
    // CPU-intensive work ไม่ควรรันใน async task โดยตรง
    // ใช้ spawn_blocking แทน
    let result = task::spawn_blocking(|| {
        // งานหนักที่ block thread
        let mut sum = 0u64;
        for i in 0..1_000_000 {
            sum += i;
        }
        sum
    }).await.expect("spawn_blocking failed");
    
    println!("Sum: {}", result);
}
```

---

## 4. tokio::time

### sleep และ interval

```rust
use tokio::time::{sleep, interval, Duration, Instant};

#[tokio::main]
async fn main() {
    // sleep - หยุดรอเวลาที่กำหนด
    println!("Starting...");
    sleep(Duration::from_millis(500)).await;
    println!("After 500ms");
    
    // interval - ทำซ้ำตามช่วงเวลา
    let mut ticker = interval(Duration::from_millis(200));
    
    for i in 0..5 {
        ticker.tick().await;
        println!("Tick {}", i);
    }
    
    // วัดเวลา
    let start = Instant::now();
    sleep(Duration::from_millis(100)).await;
    println!("Elapsed: {:?}", start.elapsed());
}
```

### timeout

```rust
use tokio::time::{timeout, Duration};

async fn slow_operation() -> String {
    sleep(Duration::from_secs(5)).await;
    "done".to_string()
}

#[tokio::main]
async fn main() {
    // กำหนด timeout
    match timeout(Duration::from_secs(1), slow_operation()).await {
        Ok(result) => println!("Got result: {}", result),
        Err(_) => println!("Operation timed out!"),
    }
    
    // timeout แบบมีค่า default
    let result = timeout(Duration::from_millis(100), slow_operation())
        .await
        .unwrap_or_else(|_| "default value".to_string());
    println!("Result: {}", result);
}
```

---

## 5. tokio::sync

### Mutex - ป้องกัน data race

```rust
use tokio::sync::Mutex;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let counter = Arc::new(Mutex::new(0u32));
    let mut handles = vec![];
    
    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = tokio::spawn(async move {
            let mut lock = counter.lock().await;
            *lock += 1;
            println!("Counter: {}", *lock);
        });
        handles.push(handle);
    }
    
    for handle in handles {
        handle.await.unwrap();
    }
    
    println!("Final: {}", *counter.lock().await);
}
```

### RwLock - อ่านพร้อมกันได้ เขียนต้องรอ

```rust
use tokio::sync::RwLock;
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let data = Arc::new(RwLock::new(vec![1, 2, 3]));
    
    // อ่านพร้อมกันหลาย readers
    let data1 = Arc::clone(&data);
    let data2 = Arc::clone(&data);
    
    let reader1 = tokio::spawn(async move {
        let guard = data1.read().await;
        println!("Reader 1: {:?}", *guard);
    });
    
    let reader2 = tokio::spawn(async move {
        let guard = data2.read().await;
        println!("Reader 2: {:?}", *guard);
    });
    
    tokio::join!(reader1, reader2).0.unwrap();
    
    // เขียนต้องรอ writers ทั้งหมด
    let mut write_guard = data.write().await;
    write_guard.push(4);
    println!("After write: {:?}", *write_guard);
}
```

### oneshot - ส่งค่าครั้งเดียว

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel::<String>();
    
    // ส่งในอีก task
    tokio::spawn(async move {
        let result = "computation result".to_string();
        tx.send(result).expect("Receiver dropped");
    });
    
    // รับค่า
    match rx.await {
        Ok(value) => println!("Got: {}", value),
        Err(_) => println!("Sender dropped without sending"),
    }
}
```

### mpsc - Multiple Producer Single Consumer

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    // สร้าง channel ขนาด buffer 10
    let (tx, mut rx) = mpsc::channel::<i32>(10);
    
    // producer 1
    let tx1 = tx.clone();
    tokio::spawn(async move {
        for i in 0..5 {
            tx1.send(i).await.expect("channel closed");
            println!("Sent: {}", i);
        }
    });
    
    // producer 2
    let tx2 = tx.clone();
    tokio::spawn(async move {
        for i in 10..15 {
            tx2.send(i).await.expect("channel closed");
            println!("Sent: {}", i);
        }
    });
    
    // ต้อง drop tx ต้นฉบับ ไม่งั้น rx จะรอตลอด
    drop(tx);
    
    // consumer
    while let Some(value) = rx.recv().await {
        println!("Received: {}", value);
    }
    println!("Channel closed");
}
```

### broadcast - ส่งให้ subscribers ทั้งหมด

```rust
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    let (tx, mut rx1) = broadcast::channel::<String>(16);
    let mut rx2 = tx.subscribe();
    
    tokio::spawn(async move {
        let messages = vec!["Hello", "World", "Rust"];
        for msg in messages {
            tx.send(msg.to_string()).expect("send failed");
            sleep(Duration::from_millis(100)).await;
        }
    });
    
    let h1 = tokio::spawn(async move {
        while let Ok(msg) = rx1.recv().await {
            println!("Subscriber 1: {}", msg);
        }
    });
    
    let h2 = tokio::spawn(async move {
        while let Ok(msg) = rx2.recv().await {
            println!("Subscriber 2: {}", msg);
        }
    });
    
    let _ = tokio::join!(h1, h2);
}
```

---

## 6. Async Traits ด้วย async-trait

Rust ยังไม่รองรับ async fn ใน traits โดยตรง (stable) ต้องใช้ `async-trait` crate

```toml
# Cargo.toml
[dependencies]
async-trait = "0.1"
tokio = { version = "1", features = ["full"] }
```

```rust
use async_trait::async_trait;

// กำหนด async trait
#[async_trait]
trait DataFetcher {
    async fn fetch(&self, url: &str) -> Result<String, String>;
    async fn fetch_multiple(&self, urls: &[&str]) -> Vec<String>;
}

struct HttpFetcher {
    timeout_ms: u64,
}

#[async_trait]
impl DataFetcher for HttpFetcher {
    async fn fetch(&self, url: &str) -> Result<String, String> {
        // จำลองการ fetch
        tokio::time::sleep(Duration::from_millis(self.timeout_ms)).await;
        Ok(format!("data from {}", url))
    }
    
    async fn fetch_multiple(&self, urls: &[&str]) -> Vec<String> {
        let mut results = vec![];
        for url in urls {
            if let Ok(data) = self.fetch(url).await {
                results.push(data);
            }
        }
        results
    }
}

// ใช้ trait object
async fn use_fetcher(fetcher: &dyn DataFetcher) {
    let data = fetcher.fetch("https://example.com").await;
    println!("Got: {:?}", data);
}

#[tokio::main]
async fn main() {
    let fetcher = HttpFetcher { timeout_ms: 100 };
    use_fetcher(&fetcher).await;
    
    let results = fetcher.fetch_multiple(&["url1", "url2", "url3"]).await;
    for (i, r) in results.iter().enumerate() {
        println!("Result {}: {}", i, r);
    }
}
```

---

## 7. select! Macro

`select!` ให้เรารอหลาย futures พร้อมกัน และทำงานเมื่อ future แรกเสร็จ

```rust
use tokio::select;
use tokio::time::{sleep, Duration};

async fn task_a() -> &'static str {
    sleep(Duration::from_millis(300)).await;
    "Task A finished"
}

async fn task_b() -> &'static str {
    sleep(Duration::from_millis(100)).await;
    "Task B finished"
}

#[tokio::main]
async fn main() {
    // รอ task แรกที่เสร็จ
    let result = select! {
        a = task_a() => a,
        b = task_b() => b,
    };
    println!("Winner: {}", result);
    
    // select! กับ channels
    let (tx1, mut rx1) = tokio::sync::mpsc::channel::<i32>(10);
    let (tx2, mut rx2) = tokio::sync::mpsc::channel::<i32>(10);
    
    tokio::spawn(async move {
        sleep(Duration::from_millis(50)).await;
        tx1.send(1).await.unwrap();
    });
    
    tokio::spawn(async move {
        sleep(Duration::from_millis(200)).await;
        tx2.send(2).await.unwrap();
    });
    
    for _ in 0..2 {
        select! {
            Some(v) = rx1.recv() => println!("From channel 1: {}", v),
            Some(v) = rx2.recv() => println!("From channel 2: {}", v),
        }
    }
}
```

### select! พร้อม cancellation

```rust
use tokio::sync::CancellationToken;

#[tokio::main]
async fn main() {
    let token = CancellationToken::new();
    let child_token = token.clone();
    
    let task = tokio::spawn(async move {
        loop {
            select! {
                _ = child_token.cancelled() => {
                    println!("Task was cancelled!");
                    break;
                }
                _ = sleep(Duration::from_millis(100)) => {
                    println!("Working...");
                }
            }
        }
    });
    
    sleep(Duration::from_millis(350)).await;
    token.cancel(); // ยกเลิก task
    task.await.unwrap();
}
```

---

## 8. Practical: Async HTTP Client ด้วย reqwest

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.11", features = ["json"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
use reqwest::Client;
use serde::{Deserialize, Serialize};
use std::time::Duration;
use tokio::time::timeout;

#[derive(Debug, Deserialize)]
struct Post {
    id: u32,
    title: String,
    body: String,
    #[serde(rename = "userId")]
    user_id: u32,
}

#[derive(Debug, Deserialize)]
struct User {
    id: u32,
    name: String,
    email: String,
}

#[derive(Debug, Serialize)]
struct CreatePost {
    title: String,
    body: String,
    #[serde(rename = "userId")]
    user_id: u32,
}

struct ApiClient {
    client: Client,
    base_url: String,
}

impl ApiClient {
    fn new(base_url: &str) -> Self {
        let client = Client::builder()
            .timeout(Duration::from_secs(10))
            .build()
            .expect("Failed to build HTTP client");
        
        ApiClient {
            client,
            base_url: base_url.to_string(),
        }
    }
    
    async fn get_post(&self, id: u32) -> Result<Post, reqwest::Error> {
        let url = format!("{}/posts/{}", self.base_url, id);
        let post = self.client
            .get(&url)
            .send()
            .await?
            .json::<Post>()
            .await?;
        Ok(post)
    }
    
    async fn get_posts(&self, limit: usize) -> Result<Vec<Post>, reqwest::Error> {
        let url = format!("{}/posts", self.base_url);
        let posts = self.client
            .get(&url)
            .send()
            .await?
            .json::<Vec<Post>>()
            .await?;
        Ok(posts.into_iter().take(limit).collect())
    }
    
    async fn create_post(&self, post: CreatePost) -> Result<Post, reqwest::Error> {
        let url = format!("{}/posts", self.base_url);
        let created = self.client
            .post(&url)
            .json(&post)
            .send()
            .await?
            .json::<Post>()
            .await?;
        Ok(created)
    }
    
    // fetch หลาย posts พร้อมกัน
    async fn get_posts_concurrent(&self, ids: &[u32]) -> Vec<Result<Post, String>> {
        let futures: Vec<_> = ids.iter().map(|&id| {
            let client = self.client.clone();
            let url = format!("{}/posts/{}", self.base_url, id);
            async move {
                client
                    .get(&url)
                    .send()
                    .await
                    .map_err(|e| e.to_string())?
                    .json::<Post>()
                    .await
                    .map_err(|e| e.to_string())
            }
        }).collect();
        
        futures::future::join_all(futures).await
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let api = ApiClient::new("https://jsonplaceholder.typicode.com");
    
    // GET single post
    println!("=== GET Single Post ===");
    match timeout(Duration::from_secs(5), api.get_post(1)).await {
        Ok(Ok(post)) => println!("Post: {} - {}", post.id, post.title),
        Ok(Err(e)) => println!("Error: {}", e),
        Err(_) => println!("Timeout!"),
    }
    
    // GET multiple posts
    println!("\n=== GET Multiple Posts ===");
    match api.get_posts(3).await {
        Ok(posts) => {
            for post in posts {
                println!("- [{}] {}", post.id, post.title);
            }
        }
        Err(e) => println!("Error: {}", e),
    }
    
    // POST new post
    println!("\n=== CREATE Post ===");
    let new_post = CreatePost {
        title: "My Rust Post".to_string(),
        body: "Async Rust is awesome!".to_string(),
        user_id: 1,
    };
    
    match api.create_post(new_post).await {
        Ok(post) => println!("Created post with id: {}", post.id),
        Err(e) => println!("Error: {}", e),
    }
    
    // Concurrent fetching
    println!("\n=== Concurrent Fetch ===");
    let ids = vec![1, 2, 3, 4, 5];
    let results = api.get_posts_concurrent(&ids).await;
    for (id, result) in ids.iter().zip(results.iter()) {
        match result {
            Ok(post) => println!("Post {}: {}", id, post.title),
            Err(e) => println!("Post {} failed: {}", id, e),
        }
    }
    
    Ok(())
}
```

---

## 9. Error Handling ใน Async

```rust
use std::fmt;

#[derive(Debug)]
enum AppError {
    NetworkError(String),
    ParseError(String),
    Timeout,
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::NetworkError(msg) => write!(f, "Network error: {}", msg),
            AppError::ParseError(msg) => write!(f, "Parse error: {}", msg),
            AppError::Timeout => write!(f, "Operation timed out"),
        }
    }
}

async fn fallible_operation(should_fail: bool) -> Result<String, AppError> {
    if should_fail {
        Err(AppError::NetworkError("Connection refused".to_string()))
    } else {
        Ok("success".to_string())
    }
}

async fn with_retry(max_retries: u32) -> Result<String, AppError> {
    for attempt in 0..max_retries {
        match fallible_operation(attempt < 2).await {
            Ok(result) => return Ok(result),
            Err(e) => {
                println!("Attempt {} failed: {}", attempt + 1, e);
                if attempt < max_retries - 1 {
                    sleep(Duration::from_millis(100)).await;
                }
            }
        }
    }
    Err(AppError::NetworkError("Max retries exceeded".to_string()))
}

#[tokio::main]
async fn main() {
    match with_retry(4).await {
        Ok(result) => println!("Final result: {}", result),
        Err(e) => println!("Failed: {}", e),
    }
}
```

---

## 10. Tokio Tracing

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

```rust
use tracing::{info, warn, error, instrument, span, Level};

#[instrument]
async fn traced_function(id: u32) -> String {
    info!("Processing request {}", id);
    sleep(Duration::from_millis(50)).await;
    
    if id % 3 == 0 {
        warn!("Slow request detected for id {}", id);
    }
    
    format!("result_{}", id)
}

#[tokio::main]
async fn main() {
    // เริ่ม tracing subscriber
    tracing_subscriber::fmt()
        .with_max_level(Level::DEBUG)
        .init();
    
    let span = span!(Level::INFO, "main_span");
    let _guard = span.enter();
    
    info!("Application starting");
    
    let results: Vec<_> = (1..=5)
        .map(|i| tokio::spawn(traced_function(i)))
        .collect();
    
    for handle in results {
        match handle.await {
            Ok(result) => info!("Got: {}", result),
            Err(e) => error!("Task failed: {}", e),
        }
    }
}
```

---

## สรุป

| Feature | การใช้งาน |
|---------|-----------|
| `async fn` | ประกาศ async function |
| `.await` | รอ future เสร็จ |
| `tokio::spawn` | สร้าง concurrent task |
| `tokio::join!` | รอหลาย futures |
| `tokio::select!` | รอ future แรกที่เสร็จ |
| `mpsc::channel` | ส่งข้อมูลระหว่าง tasks |
| `Mutex`/`RwLock` | thread-safe data access |

---

## Navigation

- [← Part 015: Testing](../part_015/README.md)
- [→ Part 017: Smart Pointers](../part_017/README.md)
- [กลับหน้าหลัก](../../README.md)
