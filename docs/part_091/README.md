# Part 091: Advanced Async Patterns

## บทนำ (Introduction)

ในบทนี้เราจะเรียนรู้ pattern ขั้นสูงของ async/await ใน Rust ซึ่งจะช่วยให้เราเขียนโค้ดที่มีประสิทธิภาพสูงและจัดการกับ concurrent operations ได้อย่างถูกต้อง

## 1. Futures Combinators

### join! macro - รอ futures หลายตัวพร้อมกัน

```rust
use tokio;

#[tokio::main]
async fn main() {
    // join! รอทุก future ให้เสร็จพร้อมกัน
    let (result1, result2, result3) = tokio::join!(
        fetch_user(1),
        fetch_products(),
        fetch_settings()
    );
    
    println!("User: {:?}", result1);
    println!("Products: {:?}", result2);
    println!("Settings: {:?}", result3);
}

async fn fetch_user(id: u64) -> String {
    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
    format!("User #{}", id)
}

async fn fetch_products() -> Vec<String> {
    tokio::time::sleep(tokio::time::Duration::from_millis(150)).await;
    vec!["Product A".to_string(), "Product B".to_string()]
}

async fn fetch_settings() -> std::collections::HashMap<String, String> {
    tokio::time::sleep(tokio::time::Duration::from_millis(50)).await;
    let mut map = std::collections::HashMap::new();
    map.insert("theme".to_string(), "dark".to_string());
    map
}
```

### try_join! macro - หยุดเมื่อมี error

```rust
use tokio;
use std::io;

#[tokio::main]
async fn main() {
    // try_join! จะหยุดทันทีเมื่อ future ใดก็ตาม return Err
    match tokio::try_join!(
        fetch_data_safe(true),
        fetch_data_safe(false),
    ) {
        Ok((data1, data2)) => {
            println!("Success: {} and {}", data1, data2);
        }
        Err(e) => {
            println!("Error occurred: {}", e);
        }
    }
}

async fn fetch_data_safe(should_succeed: bool) -> Result<String, io::Error> {
    tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
    if should_succeed {
        Ok("Data fetched successfully".to_string())
    } else {
        Err(io::Error::new(io::ErrorKind::Other, "Fetch failed"))
    }
}
```

### select! macro - เลือก future ที่เสร็จก่อน

```rust
use tokio;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // select! เลือก branch แรกที่พร้อม
    tokio::select! {
        result = slow_operation() => {
            println!("Slow operation completed: {}", result);
        }
        result = fast_operation() => {
            println!("Fast operation completed first: {}", result);
        }
        _ = timeout_operation() => {
            println!("Timeout occurred!");
        }
    }
}

async fn slow_operation() -> String {
    sleep(Duration::from_secs(5)).await;
    "Slow result".to_string()
}

async fn fast_operation() -> String {
    sleep(Duration::from_millis(100)).await;
    "Fast result".to_string()
}

async fn timeout_operation() {
    sleep(Duration::from_secs(3)).await;
}

// select! กับ loop สำหรับ event handling
async fn event_loop() {
    let mut interval = tokio::time::interval(Duration::from_secs(1));
    let (tx, mut rx) = tokio::sync::mpsc::channel::<String>(10);
    
    // spawn task ส่ง message
    tokio::spawn(async move {
        for i in 0..5 {
            sleep(Duration::from_millis(500)).await;
            tx.send(format!("Message {}", i)).await.unwrap();
        }
    });
    
    loop {
        tokio::select! {
            _ = interval.tick() => {
                println!("Tick!");
            }
            msg = rx.recv() => {
                match msg {
                    Some(m) => println!("Received: {}", m),
                    None => {
                        println!("Channel closed");
                        break;
                    }
                }
            }
        }
    }
}
```

## 2. Custom Future Implementation

### สร้าง Future ด้วยตัวเอง

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

// Custom Future ที่รอจนกว่าจะถึงเวลา
struct DelayFuture {
    deadline: Instant,
}

impl DelayFuture {
    fn new(duration: Duration) -> Self {
        DelayFuture {
            deadline: Instant::now() + duration,
        }
    }
}

impl Future for DelayFuture {
    type Output = ();
    
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if Instant::now() >= self.deadline {
            Poll::Ready(())
        } else {
            // บอก runtime ให้ wake up ใน future
            let waker = cx.waker().clone();
            let deadline = self.deadline;
            
            std::thread::spawn(move || {
                let now = Instant::now();
                if deadline > now {
                    std::thread::sleep(deadline - now);
                }
                waker.wake();
            });
            
            Poll::Pending
        }
    }
}

// Custom Future สำหรับ countdown
struct CountdownFuture {
    count: u32,
    current: u32,
}

impl CountdownFuture {
    fn new(count: u32) -> Self {
        CountdownFuture { count, current: 0 }
    }
}

impl Future for CountdownFuture {
    type Output = String;
    
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.current >= self.count {
            Poll::Ready(format!("Countdown from {} complete!", self.count))
        } else {
            self.current += 1;
            println!("Count: {}", self.current);
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

#[tokio::main]
async fn main() {
    println!("Starting custom future...");
    DelayFuture::new(Duration::from_millis(100)).await;
    println!("Delay complete!");
    
    let result = CountdownFuture::new(5).await;
    println!("{}", result);
}
```

### State Machine Future

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

enum StateMachine {
    Start,
    Processing { step: u32 },
    Complete(String),
}

struct StateMachineFuture {
    state: StateMachine,
}

impl StateMachineFuture {
    fn new() -> Self {
        StateMachineFuture {
            state: StateMachine::Start,
        }
    }
}

impl Future for StateMachineFuture {
    type Output = String;
    
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        loop {
            match &self.state {
                StateMachine::Start => {
                    println!("Starting state machine...");
                    self.state = StateMachine::Processing { step: 0 };
                }
                StateMachine::Processing { step } => {
                    let current_step = *step;
                    if current_step >= 5 {
                        self.state = StateMachine::Complete(
                            "State machine completed!".to_string()
                        );
                    } else {
                        println!("Processing step {}", current_step);
                        self.state = StateMachine::Processing { step: current_step + 1 };
                        cx.waker().wake_by_ref();
                        return Poll::Pending;
                    }
                }
                StateMachine::Complete(result) => {
                    return Poll::Ready(result.clone());
                }
            }
        }
    }
}
```

## 3. Stream Processing

### สร้าง Stream และ process

```rust
use futures::stream::{self, Stream, StreamExt};
use tokio;
use std::pin::Pin;

#[tokio::main]
async fn main() {
    // สร้าง stream จาก iterator
    let basic_stream = stream::iter(vec![1, 2, 3, 4, 5]);
    
    // map และ filter
    let processed: Vec<i32> = basic_stream
        .map(|x| x * 2)
        .filter(|x| futures::future::ready(*x > 4))
        .collect()
        .await;
    
    println!("Processed: {:?}", processed);
    
    // Async stream processing
    let async_stream = stream::iter(0..10)
        .then(|i| async move {
            tokio::time::sleep(tokio::time::Duration::from_millis(10)).await;
            i * i
        });
    
    let squares: Vec<u32> = async_stream.collect().await;
    println!("Squares: {:?}", squares);
}

// Custom async stream
fn number_stream(max: u32) -> impl Stream<Item = u32> {
    stream::unfold(0u32, move |state| async move {
        if state < max {
            Some((state, state + 1))
        } else {
            None
        }
    })
}

// Stream pipeline
async fn process_pipeline() {
    let pipeline = number_stream(100)
        .chunks(10)  // grouping
        .map(|chunk| {
            let sum: u32 = chunk.iter().sum();
            sum
        })
        .take(5);
    
    let results: Vec<u32> = pipeline.collect().await;
    println!("Chunk sums: {:?}", results);
}

// Merging streams
async fn merge_streams() {
    use futures::stream::select;
    
    let stream1 = stream::iter(vec!["a1", "a2", "a3"]);
    let stream2 = stream::iter(vec!["b1", "b2", "b3"]);
    
    let merged = select(stream1, stream2);
    let all: Vec<&str> = merged.collect().await;
    println!("Merged: {:?}", all);
}
```

## 4. Backpressure Handling

### Channel-based backpressure

```rust
use tokio::sync::mpsc;
use tokio;
use std::time::Duration;

#[tokio::main]
async fn main() {
    // bounded channel สำหรับ backpressure
    let (tx, mut rx) = mpsc::channel::<u32>(10); // buffer size = 10
    
    // Producer task
    let producer = tokio::spawn(async move {
        for i in 0..100 {
            // send() จะ block เมื่อ buffer เต็ม (backpressure)
            if let Err(e) = tx.send(i).await {
                eprintln!("Producer error: {}", e);
                break;
            }
            println!("Produced: {}", i);
        }
        println!("Producer done");
    });
    
    // Consumer task (ช้ากว่า)
    let consumer = tokio::spawn(async move {
        while let Some(item) = rx.recv().await {
            tokio::time::sleep(Duration::from_millis(50)).await;
            println!("Consumed: {}", item);
        }
        println!("Consumer done");
    });
    
    let _ = tokio::join!(producer, consumer);
}

// Semaphore สำหรับจำกัด concurrent operations
use tokio::sync::Semaphore;
use std::sync::Arc;

async fn with_backpressure() {
    let semaphore = Arc::new(Semaphore::new(5)); // max 5 concurrent
    let mut handles = vec![];
    
    for i in 0..20 {
        let sem = semaphore.clone();
        let handle = tokio::spawn(async move {
            let _permit = sem.acquire().await.unwrap();
            println!("Task {} starting", i);
            tokio::time::sleep(Duration::from_millis(100)).await;
            println!("Task {} done", i);
        });
        handles.push(handle);
    }
    
    for handle in handles {
        handle.await.unwrap();
    }
}
```

## 5. Async Recursion (Box::pin)

### Recursive async functions

```rust
use std::pin::Pin;
use std::future::Future;

// ต้องใช้ Box::pin สำหรับ async recursion
async fn factorial(n: u64) -> u64 {
    if n <= 1 {
        1
    } else {
        n * factorial(n - 1).await
    }
}

// async recursion กับ custom return type
fn fibonacci(n: u32) -> Pin<Box<dyn Future<Output = u64> + Send>> {
    Box::pin(async move {
        match n {
            0 => 0,
            1 => 1,
            _ => {
                let a = fibonacci(n - 1).await;
                let b = fibonacci(n - 2).await;
                a + b
            }
        }
    })
}

// Tree traversal แบบ async
#[derive(Debug)]
struct TreeNode {
    value: i32,
    children: Vec<TreeNode>,
}

fn sum_tree(node: &TreeNode) -> Pin<Box<dyn Future<Output = i32> + '_>> {
    Box::pin(async move {
        let mut sum = node.value;
        for child in &node.children {
            sum += sum_tree(child).await;
        }
        sum
    })
}

#[tokio::main]
async fn main() {
    println!("5! = {}", factorial(5).await);
    println!("fib(10) = {}", fibonacci(10).await);
    
    let tree = TreeNode {
        value: 1,
        children: vec![
            TreeNode {
                value: 2,
                children: vec![
                    TreeNode { value: 4, children: vec![] },
                    TreeNode { value: 5, children: vec![] },
                ],
            },
            TreeNode {
                value: 3,
                children: vec![
                    TreeNode { value: 6, children: vec![] },
                ],
            },
        ],
    };
    
    println!("Tree sum: {}", sum_tree(&tree).await);
}
```

## 6. Tokio Tasks and Cancellation

### Task cancellation ด้วย CancellationToken

```rust
use tokio;
use tokio_util::sync::CancellationToken;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let token = CancellationToken::new();
    let child_token = token.child_token();
    
    let task = tokio::spawn(async move {
        tokio::select! {
            _ = child_token.cancelled() => {
                println!("Task was cancelled");
            }
            _ = do_work() => {
                println!("Task completed normally");
            }
        }
    });
    
    sleep(Duration::from_millis(500)).await;
    println!("Cancelling task...");
    token.cancel();
    
    task.await.unwrap();
    println!("Done");
}

async fn do_work() {
    println!("Starting work...");
    for i in 0..10 {
        sleep(Duration::from_millis(200)).await;
        println!("Work step {}", i);
    }
    println!("Work complete");
}

// JoinSet สำหรับจัดการหลาย tasks
use tokio::task::JoinSet;

async fn manage_tasks() {
    let mut set = JoinSet::new();
    
    for i in 0..5 {
        set.spawn(async move {
            sleep(Duration::from_millis(i * 100)).await;
            format!("Task {} result", i)
        });
    }
    
    // รอ task แรกที่เสร็จ
    while let Some(result) = set.join_next().await {
        match result {
            Ok(value) => println!("Completed: {}", value),
            Err(e) => println!("Task panicked: {}", e),
        }
    }
}
```

## 7. Semaphore for Concurrency Limiting

```rust
use tokio::sync::Semaphore;
use std::sync::Arc;
use tokio;

struct RateLimiter {
    semaphore: Arc<Semaphore>,
}

impl RateLimiter {
    fn new(max_concurrent: usize) -> Self {
        RateLimiter {
            semaphore: Arc::new(Semaphore::new(max_concurrent)),
        }
    }
    
    async fn execute<F, T>(&self, f: F) -> T
    where
        F: std::future::Future<Output = T>,
    {
        let _permit = self.semaphore.acquire().await.unwrap();
        f.await
    }
}

// Connection pool simulation
struct ConnectionPool {
    semaphore: Arc<Semaphore>,
    connections: Vec<Arc<tokio::sync::Mutex<String>>>,
}

impl ConnectionPool {
    fn new(size: usize) -> Self {
        let connections = (0..size)
            .map(|i| Arc::new(tokio::sync::Mutex::new(format!("Connection {}", i))))
            .collect();
        
        ConnectionPool {
            semaphore: Arc::new(Semaphore::new(size)),
            connections,
        }
    }
    
    async fn get_connection(&self) -> tokio::sync::MutexGuard<String> {
        let _permit = self.semaphore.acquire().await.unwrap();
        // simplified - just get first available
        self.connections[0].lock().await
    }
}

#[tokio::main]
async fn main() {
    let limiter = Arc::new(RateLimiter::new(3));
    let mut handles = vec![];
    
    for i in 0..10 {
        let lim = limiter.clone();
        let handle = tokio::spawn(async move {
            lim.execute(async move {
                println!("Executing task {}", i);
                tokio::time::sleep(tokio::time::Duration::from_millis(100)).await;
                println!("Task {} done", i);
            }).await
        });
        handles.push(handle);
    }
    
    for handle in handles {
        handle.await.unwrap();
    }
}
```

## 8. Practical: Async Data Pipeline

### ระบบประมวลผลข้อมูลแบบ async pipeline

```rust
use futures::stream::{self, StreamExt};
use tokio::sync::mpsc;
use tokio;
use std::sync::Arc;
use tokio::sync::Mutex;

#[derive(Debug, Clone)]
struct RawData {
    id: u64,
    value: f64,
    timestamp: u64,
}

#[derive(Debug, Clone)]
struct ProcessedData {
    id: u64,
    normalized_value: f64,
    category: String,
    processed_at: u64,
}

#[derive(Debug)]
struct Pipeline {
    stats: Arc<Mutex<PipelineStats>>,
}

#[derive(Debug, Default)]
struct PipelineStats {
    total_processed: u64,
    total_errors: u64,
    avg_value: f64,
}

impl Pipeline {
    fn new() -> Self {
        Pipeline {
            stats: Arc::new(Mutex::new(PipelineStats::default())),
        }
    }
    
    // Stage 1: Ingest data
    async fn ingest(&self, count: u64) -> impl futures::Stream<Item = RawData> + '_ {
        stream::iter(0..count).then(|i| async move {
            tokio::time::sleep(tokio::time::Duration::from_millis(5)).await;
            RawData {
                id: i,
                value: (i as f64).sin() * 100.0,
                timestamp: i * 1000,
            }
        })
    }
    
    // Stage 2: Validate data
    async fn validate(&self, data: RawData) -> Option<RawData> {
        if data.value.is_nan() || data.value.is_infinite() {
            let mut stats = self.stats.lock().await;
            stats.total_errors += 1;
            None
        } else {
            Some(data)
        }
    }
    
    // Stage 3: Transform data
    async fn transform(&self, data: RawData) -> ProcessedData {
        tokio::time::sleep(tokio::time::Duration::from_millis(1)).await;
        
        let normalized = (data.value + 100.0) / 200.0; // normalize to 0-1
        let category = if normalized > 0.7 {
            "high"
        } else if normalized > 0.3 {
            "medium"
        } else {
            "low"
        };
        
        ProcessedData {
            id: data.id,
            normalized_value: normalized,
            category: category.to_string(),
            processed_at: data.timestamp + 1,
        }
    }
    
    // Stage 4: Aggregate results
    async fn aggregate(&self, data: ProcessedData) {
        let mut stats = self.stats.lock().await;
        stats.total_processed += 1;
        stats.avg_value = (stats.avg_value * (stats.total_processed - 1) as f64
            + data.normalized_value) / stats.total_processed as f64;
    }
    
    async fn run(&self, item_count: u64) {
        println!("Starting pipeline with {} items...", item_count);
        
        let stream = self.ingest(item_count).await;
        
        stream
            .then(|data| async { self.validate(data).await })
            .filter_map(|opt| async { opt })
            .then(|data| async { self.transform(data).await })
            .for_each(|data| async { self.aggregate(data).await })
            .await;
        
        let stats = self.stats.lock().await;
        println!("Pipeline complete!");
        println!("  Processed: {}", stats.total_processed);
        println!("  Errors: {}", stats.total_errors);
        println!("  Avg value: {:.4}", stats.avg_value);
    }
}

// Parallel pipeline with workers
async fn parallel_pipeline() {
    let (input_tx, input_rx) = mpsc::channel::<RawData>(100);
    let (output_tx, mut output_rx) = mpsc::channel::<ProcessedData>(100);
    
    // Producer
    tokio::spawn(async move {
        for i in 0..50 {
            let data = RawData {
                id: i,
                value: (i as f64) * 2.5,
                timestamp: i * 1000,
            };
            input_tx.send(data).await.unwrap();
        }
    });
    
    // Workers (parallel processing)
    let num_workers = 4;
    let input_rx = Arc::new(Mutex::new(input_rx));
    
    for worker_id in 0..num_workers {
        let rx = input_rx.clone();
        let tx = output_tx.clone();
        
        tokio::spawn(async move {
            loop {
                let data = {
                    let mut rx = rx.lock().await;
                    rx.recv().await
                };
                
                match data {
                    Some(raw) => {
                        tokio::time::sleep(tokio::time::Duration::from_millis(10)).await;
                        let processed = ProcessedData {
                            id: raw.id,
                            normalized_value: raw.value / 100.0,
                            category: format!("worker-{}", worker_id),
                            processed_at: raw.timestamp + 100,
                        };
                        tx.send(processed).await.unwrap();
                    }
                    None => break,
                }
            }
        });
    }
    drop(output_tx); // close when all workers done
    
    // Collector
    let mut count = 0;
    while let Some(result) = output_rx.recv().await {
        count += 1;
        if count % 10 == 0 {
            println!("Collected {} items, last id: {}", count, result.id);
        }
    }
    println!("Total collected: {}", count);
}

#[tokio::main]
async fn main() {
    let pipeline = Pipeline::new();
    pipeline.run(50).await;
    
    println!("\nRunning parallel pipeline...");
    parallel_pipeline().await;
}
```

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:
- **Futures Combinators**: join!, try_join!, select! สำหรับการจัดการหลาย futures
- **Custom Future**: การ implement Future trait เอง
- **Stream Processing**: การประมวลผลข้อมูลแบบ streaming
- **Backpressure**: การควบคุม flow ด้วย bounded channels
- **Async Recursion**: ใช้ Box::pin สำหรับ recursive async functions
- **Task Management**: CancellationToken, JoinSet
- **Semaphore**: จำกัด concurrent operations
- **Data Pipeline**: ระบบประมวลผลข้อมูลแบบ async pipeline

---

[← Part 090](../part_090/README.md) | [Part 092 →](../part_092/README.md)
