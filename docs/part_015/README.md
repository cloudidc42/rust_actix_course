# Part 015: Concurrency กับ Threads 🧵

## 🎯 เป้าหมายของ Part นี้

- std::thread (spawn, join, sleep)
- Arc<Mutex<T>> สำหรับ shared state
- Arc<RwLock<T>> สำหรับ read-heavy workloads
- Channel (mpsc: sender/receiver)
- Deadlock prevention
- Thread pool pattern
- Rayon สำหรับ data parallelism
- Practical: parallel file processing

---

## 1. std::thread พื้นฐาน

```rust
use std::thread;
use std::time::Duration;

fn main() {
    // spawn - สร้าง thread ใหม่
    let handle = thread::spawn(|| {
        println!("Thread: สวัสดีจาก thread ใหม่!");
        thread::sleep(Duration::from_millis(100));
        println!("Thread: ทำงานเสร็จแล้ว");
    });
    
    println!("Main: ทำงานพร้อมกับ thread");
    
    // join - รอให้ thread ทำงานเสร็จ
    handle.join().unwrap();
    println!("Main: thread เสร็จแล้ว");

    // spawn หลาย threads พร้อมกัน
    let mut handles = Vec::new();
    
    for i in 0..5 {
        let h = thread::spawn(move || {
            println!("Thread {}: เริ่มทำงาน", i);
            thread::sleep(Duration::from_millis(50 * i));
            println!("Thread {}: เสร็จ", i);
            i * i  // return value
        });
        handles.push(h);
    }
    
    // รอทุก thread และ collect results
    let results: Vec<u64> = handles.into_iter()
        .map(|h| h.join().unwrap())
        .collect();
    
    println!("Results: {:?}", results);  // [0, 1, 4, 9, 16]

    // Thread ที่ panic จะ return Err
    let handle = thread::spawn(|| {
        panic!("Thread panicked!");
    });
    
    match handle.join() {
        Ok(_) => println!("Thread succeeded"),
        Err(e) => println!("Thread panicked: {:?}", e),
    }

    // Thread sleep
    println!("\nCounting down:");
    for i in (1..=3).rev() {
        println!("  {}", i);
        thread::sleep(Duration::from_millis(500));
    }
    println!("  Go!");

    // Named threads (ช่วย debug)
    let handle = thread::Builder::new()
        .name("worker-thread".to_string())
        .stack_size(4 * 1024 * 1024)  // 4MB stack
        .spawn(|| {
            let name = thread::current().name().unwrap_or("unnamed");
            println!("Running as: {}", name);
        })
        .unwrap();
    
    handle.join().unwrap();

    // Current thread info
    let current = thread::current();
    println!("Main thread id: {:?}", current.id());
    
    // Number of available CPUs (for parallelism decisions)
    let cpus = thread::available_parallelism()
        .map(|n| n.get())
        .unwrap_or(1);
    println!("Available parallelism: {}", cpus);
}
```

---

## 2. Shared State กับ Arc<Mutex<T>>

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    // ปัญหา: ไม่สามารถแชร์ mutable data ได้โดยตรง
    // let mut counter = 0;
    // thread::spawn(|| { counter += 1; });  // ERROR: cannot move or borrow

    // แก้ด้วย Arc<Mutex<T>>
    // Arc = Atomic Reference Counted (ทำให้แชร์ ownership ได้)
    // Mutex = Mutual Exclusion (ทำให้ access ทีละ thread ได้)
    
    let counter = Arc::new(Mutex::new(0u64));
    let mut handles = Vec::new();
    
    for _ in 0..10 {
        let counter_clone = Arc::clone(&counter);
        
        let h = thread::spawn(move || {
            // lock() ขอ access - block ถ้า thread อื่น lock อยู่
            let mut num = counter_clone.lock().unwrap();
            *num += 1;
            // MutexGuard ถูก drop อัตโนมัติเมื่อออกจาก scope
        });
        
        handles.push(h);
    }
    
    for h in handles {
        h.join().unwrap();
    }
    
    println!("Counter: {}", *counter.lock().unwrap());  // 10

    // ตัวอย่างที่ซับซ้อนกว่า: shared Vec
    let data = Arc::new(Mutex::new(Vec::<String>::new()));
    let mut handles = Vec::new();
    
    for i in 0..5 {
        let data_clone = Arc::clone(&data);
        
        let h = thread::spawn(move || {
            let mut vec = data_clone.lock().unwrap();
            vec.push(format!("Item from thread {}", i));
        });
        
        handles.push(h);
    }
    
    for h in handles {
        h.join().unwrap();
    }
    
    let final_data = data.lock().unwrap();
    println!("Collected {} items", final_data.len());
    for item in final_data.iter() {
        println!("  {}", item);
    }
}
```

---

## 3. Mutex กับ Complex Data Structures

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::collections::HashMap;
use std::time::Instant;

// Thread-safe cache
struct ThreadSafeCache {
    data: Arc<Mutex<HashMap<String, String>>>,
    hits: Arc<Mutex<u64>>,
    misses: Arc<Mutex<u64>>,
}

impl ThreadSafeCache {
    fn new() -> Self {
        ThreadSafeCache {
            data: Arc::new(Mutex::new(HashMap::new())),
            hits: Arc::new(Mutex::new(0)),
            misses: Arc::new(Mutex::new(0)),
        }
    }
    
    fn get(&self, key: &str) -> Option<String> {
        let data = self.data.lock().unwrap();
        if let Some(value) = data.get(key) {
            *self.hits.lock().unwrap() += 1;
            Some(value.clone())
        } else {
            *self.misses.lock().unwrap() += 1;
            None
        }
    }
    
    fn set(&self, key: String, value: String) {
        let mut data = self.data.lock().unwrap();
        data.insert(key, value);
    }
    
    fn stats(&self) -> (u64, u64) {
        let hits = *self.hits.lock().unwrap();
        let misses = *self.misses.lock().unwrap();
        (hits, misses)
    }
    
    fn size(&self) -> usize {
        self.data.lock().unwrap().len()
    }
}

fn main() {
    let cache = Arc::new(ThreadSafeCache::new());
    
    // Populate cache
    {
        let c = Arc::clone(&cache);
        for i in 0..100 {
            c.set(format!("key_{}", i), format!("value_{}", i));
        }
    }
    
    let start = Instant::now();
    let mut handles = Vec::new();
    
    // Multiple readers and writers
    for t in 0..8 {
        let cache_clone = Arc::clone(&cache);
        
        let h = thread::spawn(move || {
            for i in 0..50 {
                let key = format!("key_{}", (t * 50 + i) % 100);
                
                // Mix of reads and writes
                if i % 5 == 0 {
                    cache_clone.set(
                        format!("new_key_{}_{}", t, i),
                        format!("value")
                    );
                } else {
                    let _val = cache_clone.get(&key);
                }
            }
        });
        
        handles.push(h);
    }
    
    for h in handles {
        h.join().unwrap();
    }
    
    let elapsed = start.elapsed();
    let (hits, misses) = cache.stats();
    
    println!("Cache size: {}", cache.size());
    println!("Hits: {}, Misses: {}", hits, misses);
    println!("Time: {:?}", elapsed);
    
    // try_lock - ไม่ block ถ้า Mutex ถูก lock อยู่
    let mutex = Arc::new(Mutex::new(0));
    let m2 = Arc::clone(&mutex);
    
    let _guard = mutex.lock().unwrap();  // lock main thread
    
    let h = thread::spawn(move || {
        match m2.try_lock() {
            Ok(mut val) => {
                *val += 1;
                println!("Got lock");
            }
            Err(e) => {
                println!("Could not get lock: {}", e);  // WouldBlock
            }
        }
    });
    
    h.join().unwrap();
}
```

---

## 4. Arc<RwLock<T>> สำหรับ Read-Heavy Workloads

```rust
use std::sync::{Arc, RwLock};
use std::thread;
use std::time::{Duration, Instant};
use std::collections::HashMap;

// RwLock อนุญาต:
// - หลาย readers พร้อมกันได้ (read lock)
// - writer เดียว (write lock) - block readers ทั้งหมด

#[derive(Default)]
struct Database {
    users: HashMap<u32, User>,
    user_count: u32,
}

#[derive(Debug, Clone)]
struct User {
    id: u32,
    name: String,
    email: String,
}

impl Database {
    fn new() -> Self {
        let mut db = Database::default();
        for i in 1..=100 {
            db.users.insert(i, User {
                id: i,
                name: format!("User {}", i),
                email: format!("user{}@example.com", i),
            });
        }
        db.user_count = 100;
        db
    }
    
    fn get_user(&self, id: u32) -> Option<User> {
        self.users.get(&id).cloned()
    }
    
    fn add_user(&mut self, user: User) {
        self.user_count += 1;
        self.users.insert(user.id, user);
    }
    
    fn count(&self) -> u32 {
        self.user_count
    }
}

fn main() {
    let db = Arc::new(RwLock::new(Database::new()));
    let start = Instant::now();
    let mut handles = Vec::new();
    
    // Spawn 20 reader threads
    for t in 0..20 {
        let db_clone = Arc::clone(&db);
        
        let h = thread::spawn(move || {
            let mut read_count = 0;
            for _ in 0..50 {
                let user_id = (t * 50 + read_count) % 100 + 1;
                
                // Multiple readers can read simultaneously
                let db = db_clone.read().unwrap();
                let _user = db.get_user(user_id);
                read_count += 1;
                // read lock released here
            }
            read_count
        });
        
        handles.push(h);
    }
    
    // Spawn 2 writer threads
    for w in 0..2 {
        let db_clone = Arc::clone(&db);
        
        let h = thread::spawn(move || {
            for i in 0..5 {
                let user_id = 1000 + w * 5 + i;
                
                // Only one writer at a time, blocks all readers
                let mut db = db_clone.write().unwrap();
                db.add_user(User {
                    id: user_id,
                    name: format!("New User {}", user_id),
                    email: format!("new{}@example.com", user_id),
                });
                // write lock released here
                
                thread::sleep(Duration::from_millis(10));
            }
        });
        
        handles.push(h);
    }
    
    let total_reads: u32 = handles.into_iter()
        .filter_map(|h| h.join().ok())
        .sum();
    
    let elapsed = start.elapsed();
    
    let db = db.read().unwrap();
    println!("Total reads: {}", total_reads);
    println!("Final user count: {}", db.count());
    println!("Time: {:?}", elapsed);

    // try_read / try_write
    let lock = Arc::new(RwLock::new(vec![1, 2, 3]));
    
    // Non-blocking read attempt
    match lock.try_read() {
        Ok(data) => println!("Read: {:?}", *data),
        Err(_) => println!("Lock busy"),
    }
    
    // Non-blocking write attempt
    match lock.try_write() {
        Ok(mut data) => {
            data.push(4);
            println!("After write: {:?}", *data);
        }
        Err(_) => println!("Write lock busy"),
    }
}
```

---

## 5. Channels (mpsc)

mpsc = Multiple Producer, Single Consumer

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    // สร้าง channel
    let (tx, rx) = mpsc::channel::<String>();
    
    // Clone sender สำหรับหลาย producers
    let tx2 = tx.clone();
    
    // Producer 1
    thread::spawn(move || {
        let messages = vec!["msg_a1", "msg_a2", "msg_a3"];
        for msg in messages {
            tx.send(msg.to_string()).unwrap();
            thread::sleep(Duration::from_millis(50));
        }
        println!("Producer 1: done");
        // tx ถูก drop เมื่อออกจาก scope
    });
    
    // Producer 2
    thread::spawn(move || {
        let messages = vec!["msg_b1", "msg_b2", "msg_b3"];
        for msg in messages {
            tx2.send(msg.to_string()).unwrap();
            thread::sleep(Duration::from_millis(75));
        }
        println!("Producer 2: done");
        // tx2 ถูก drop เมื่อออกจาก scope
    });
    
    // Consumer - รอรับข้อความจนกว่าทุก sender จะถูก drop
    for received in rx {
        println!("Received: {}", received);
    }
    
    println!("All producers done");

    // Synchronous channel (bounded)
    let (tx, rx) = mpsc::sync_channel::<i32>(2);  // buffer size 2
    
    let handle = thread::spawn(move || {
        for i in 0..5 {
            println!("Sending {}", i);
            tx.send(i).unwrap();  // blocks when buffer full
            println!("Sent {}", i);
        }
    });
    
    thread::sleep(Duration::from_millis(200));
    
    for _ in 0..5 {
        let val = rx.recv().unwrap();
        println!("Got: {}", val);
        thread::sleep(Duration::from_millis(50));
    }
    
    handle.join().unwrap();

    // try_recv - non-blocking
    let (tx, rx) = mpsc::channel::<i32>();
    
    tx.send(42).unwrap();
    
    match rx.try_recv() {
        Ok(val) => println!("Got value: {}", val),
        Err(mpsc::TryRecvError::Empty) => println!("No message yet"),
        Err(mpsc::TryRecvError::Disconnected) => println!("Channel closed"),
    }
    
    // recv_timeout
    match rx.recv_timeout(Duration::from_millis(100)) {
        Ok(val) => println!("Got: {}", val),
        Err(mpsc::RecvTimeoutError::Timeout) => println!("Timeout!"),
        Err(mpsc::RecvTimeoutError::Disconnected) => println!("Disconnected"),
    }
}
```

---

## 6. Channel Patterns

```rust
use std::sync::mpsc;
use std::thread;

// Worker pool pattern with channels
#[derive(Debug)]
struct Task {
    id: u64,
    data: String,
}

#[derive(Debug)]
struct Result {
    task_id: u64,
    output: String,
    worker_id: usize,
}

fn process_task(task: &Task, worker_id: usize) -> Result {
    // Simulate work
    thread::sleep(std::time::Duration::from_millis(10));
    
    Result {
        task_id: task.id,
        output: format!("Processed '{}' by worker {}", task.data, worker_id),
        worker_id,
    }
}

fn main() {
    let num_workers = 4;
    let num_tasks = 20;
    
    // Task channel: main -> workers
    let (task_tx, task_rx) = mpsc::channel::<Task>();
    let task_rx = std::sync::Arc::new(std::sync::Mutex::new(task_rx));
    
    // Result channel: workers -> main
    let (result_tx, result_rx) = mpsc::channel::<Result>();
    
    // Spawn workers
    let mut worker_handles = Vec::new();
    for worker_id in 0..num_workers {
        let rx_clone = std::sync::Arc::clone(&task_rx);
        let tx_clone = result_tx.clone();
        
        let h = thread::spawn(move || {
            loop {
                // Get next task
                let task = {
                    let rx = rx_clone.lock().unwrap();
                    rx.recv()
                };
                
                match task {
                    Ok(task) => {
                        let result = process_task(&task, worker_id);
                        tx_clone.send(result).unwrap();
                    }
                    Err(_) => {
                        println!("Worker {} shutting down", worker_id);
                        break;
                    }
                }
            }
        });
        
        worker_handles.push(h);
    }
    
    // Drop the extra result_tx so receiver knows when all workers are done
    drop(result_tx);
    
    // Send tasks
    for i in 0..num_tasks {
        task_tx.send(Task {
            id: i,
            data: format!("task_data_{}", i),
        }).unwrap();
    }
    
    // Close task channel to signal workers to stop
    drop(task_tx);
    
    // Collect results
    let mut results = Vec::new();
    for result in result_rx {
        results.push(result);
    }
    
    // Wait for workers
    for h in worker_handles {
        h.join().unwrap();
    }
    
    // Summary
    println!("Processed {} tasks", results.len());
    
    let mut worker_counts = vec![0u32; num_workers];
    for r in &results {
        worker_counts[r.worker_id] += 1;
    }
    
    println!("Work distribution:");
    for (id, count) in worker_counts.iter().enumerate() {
        println!("  Worker {}: {} tasks", id, count);
    }
}
```

---

## 7. Deadlock Prevention

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    // ตัวอย่าง Deadlock (DON'T DO THIS)
    /*
    let lock_a = Arc::new(Mutex::new("A"));
    let lock_b = Arc::new(Mutex::new("B"));
    
    let a1 = Arc::clone(&lock_a);
    let b1 = Arc::clone(&lock_b);
    
    // Thread 1: lock A then B
    let t1 = thread::spawn(move || {
        let _a = a1.lock().unwrap();
        thread::sleep(Duration::from_millis(10));
        let _b = b1.lock().unwrap();  // DEADLOCK: T2 has B, waiting for A
    });
    
    let a2 = Arc::clone(&lock_a);
    let b2 = Arc::clone(&lock_b);
    
    // Thread 2: lock B then A
    let t2 = thread::spawn(move || {
        let _b = b2.lock().unwrap();
        thread::sleep(Duration::from_millis(10));
        let _a = a2.lock().unwrap();  // DEADLOCK: T1 has A, waiting for B
    });
    */

    // Prevention 1: Lock ordering - always lock in same order
    let lock_1 = Arc::new(Mutex::new("Resource 1"));
    let lock_2 = Arc::new(Mutex::new("Resource 2"));
    
    let l1 = Arc::clone(&lock_1);
    let l2 = Arc::clone(&lock_2);
    
    let t1 = thread::spawn(move || {
        // ALWAYS: lock 1 first, then 2
        let _r1 = l1.lock().unwrap();
        let _r2 = l2.lock().unwrap();
        println!("T1: got both locks (order: 1, 2)");
    });
    
    let l1 = Arc::clone(&lock_1);
    let l2 = Arc::clone(&lock_2);
    
    let t2 = thread::spawn(move || {
        // ALWAYS: lock 1 first, then 2 (same order!)
        let _r1 = l1.lock().unwrap();
        let _r2 = l2.lock().unwrap();
        println!("T2: got both locks (order: 1, 2)");
    });
    
    t1.join().unwrap();
    t2.join().unwrap();

    // Prevention 2: try_lock with timeout/retry
    let lock_a = Arc::new(Mutex::new(0));
    let lock_b = Arc::new(Mutex::new(0));
    
    let a_clone = Arc::clone(&lock_a);
    let b_clone = Arc::clone(&lock_b);
    
    let t = thread::spawn(move || {
        let max_retries = 10;
        let mut retries = 0;
        
        loop {
            let a = a_clone.try_lock();
            let b = b_clone.try_lock();
            
            match (a, b) {
                (Ok(mut a), Ok(mut b)) => {
                    *a += 1;
                    *b += 1;
                    println!("Got both locks on retry {}", retries);
                    break;
                }
                _ => {
                    retries += 1;
                    if retries >= max_retries {
                        println!("Could not acquire locks after {} retries", retries);
                        break;
                    }
                    thread::sleep(std::time::Duration::from_millis(10));
                }
            }
        }
    });
    
    t.join().unwrap();

    // Prevention 3: ใช้ scope เพื่อ release lock เร็วๆ
    let data = Arc::new(Mutex::new(vec![1, 2, 3]));
    
    let d_clone = Arc::clone(&data);
    thread::spawn(move || {
        let sum: i32 = {
            // Lock ถูก release ตอน } นี้
            let locked = d_clone.lock().unwrap();
            locked.iter().sum()
        };
        // ไม่มี lock ตอนนี้ - safe to do other things
        println!("Sum (no lock held): {}", sum);
    }).join().unwrap();

    // Prevention 4: ใช้ RwLock แทน Mutex สำหรับ read-heavy
    use std::sync::RwLock;
    let config = Arc::new(RwLock::new(std::collections::HashMap::<String, String>::new()));
    
    // Multiple concurrent readers - no deadlock risk
    let c1 = Arc::clone(&config);
    let c2 = Arc::clone(&config);
    
    let r1 = thread::spawn(move || {
        let cfg = c1.read().unwrap();
        cfg.get("key").cloned()
    });
    
    let r2 = thread::spawn(move || {
        let cfg = c2.read().unwrap();
        cfg.get("other").cloned()
    });
    
    r1.join().unwrap();
    r2.join().unwrap();
}
```

---

## 8. Thread Pool Pattern

```rust
use std::sync::{Arc, Mutex};
use std::thread;

type Job = Box<dyn FnOnce() + Send + 'static>;

struct ThreadPool {
    workers: Vec<Worker>,
    sender: Option<std::sync::mpsc::Sender<Job>>,
}

struct Worker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>,
}

impl ThreadPool {
    fn new(size: usize) -> Self {
        assert!(size > 0, "Thread pool size must be > 0");
        
        let (sender, receiver) = std::sync::mpsc::channel::<Job>();
        let receiver = Arc::new(Mutex::new(receiver));
        
        let mut workers = Vec::with_capacity(size);
        
        for id in 0..size {
            let receiver_clone = Arc::clone(&receiver);
            
            let thread = thread::Builder::new()
                .name(format!("worker-{}", id))
                .spawn(move || {
                    loop {
                        let job = {
                            let rx = receiver_clone.lock().unwrap();
                            rx.recv()
                        };
                        
                        match job {
                            Ok(job) => {
                                // println!("Worker {} executing job", id);
                                job();
                            }
                            Err(_) => {
                                println!("Worker {} shutting down", id);
                                break;
                            }
                        }
                    }
                })
                .unwrap();
            
            workers.push(Worker {
                id,
                thread: Some(thread),
            });
        }
        
        ThreadPool {
            workers,
            sender: Some(sender),
        }
    }
    
    fn execute<F>(&self, f: F)
    where F: FnOnce() + Send + 'static
    {
        let job = Box::new(f);
        self.sender.as_ref().unwrap().send(job).unwrap();
    }
    
    fn shutdown(&mut self) {
        // Drop sender to signal workers to stop
        drop(self.sender.take());
        
        for worker in &mut self.workers {
            println!("Waiting for worker {}...", worker.id);
            if let Some(thread) = worker.thread.take() {
                thread.join().unwrap();
            }
        }
    }
}

impl Drop for ThreadPool {
    fn drop(&mut self) {
        self.shutdown();
    }
}

fn main() {
    let pool = ThreadPool::new(4);
    let results = Arc::new(Mutex::new(Vec::<String>::new()));
    
    println!("Submitting 20 jobs to pool of 4 threads...");
    
    for i in 0..20 {
        let results_clone = Arc::clone(&results);
        
        pool.execute(move || {
            // Simulate varying amounts of work
            thread::sleep(std::time::Duration::from_millis(20 + i * 5));
            
            let worker_name = thread::current()
                .name()
                .unwrap_or("unnamed")
                .to_string();
            
            let result = format!("Job {} done by {}", i, worker_name);
            
            results_clone.lock().unwrap().push(result);
        });
    }
    
    // Wait for all jobs (by dropping pool)
    drop(pool);
    
    let results = results.lock().unwrap();
    println!("Completed {} jobs", results.len());
    for r in results.iter().take(5) {
        println!("  {}", r);
    }
    if results.len() > 5 {
        println!("  ...and {} more", results.len() - 5);
    }
}
```

---

## 9. Rayon สำหรับ Data Parallelism

```toml
# Cargo.toml
[dependencies]
rayon = "1.8"
```

```rust
use rayon::prelude::*;
use std::time::Instant;

fn main() {
    let data: Vec<i64> = (0..10_000_000).collect();
    
    // Sequential sum
    let start = Instant::now();
    let seq_sum: i64 = data.iter().sum();
    let seq_time = start.elapsed();
    
    // Parallel sum with rayon
    let start = Instant::now();
    let par_sum: i64 = data.par_iter().sum();
    let par_time = start.elapsed();
    
    assert_eq!(seq_sum, par_sum);
    println!("Sum: {}", seq_sum);
    println!("Sequential: {:?}", seq_time);
    println!("Parallel:   {:?}", par_time);
    println!("Speedup: {:.2}x", seq_time.as_secs_f64() / par_time.as_secs_f64());

    // Parallel map
    let numbers: Vec<i32> = (1..=1_000_000).collect();
    
    let start = Instant::now();
    let squares: Vec<i64> = numbers.par_iter()
        .map(|&x| (x as i64) * (x as i64))
        .collect();
    println!("\nParallel map ({} items): {:?}", squares.len(), start.elapsed());

    // Parallel filter
    let start = Instant::now();
    let primes: Vec<&i32> = numbers.par_iter()
        .filter(|&&n| is_prime(n as u64))
        .collect();
    println!("Parallel filter (primes): {} found in {:?}", primes.len(), start.elapsed());

    // Parallel sort
    let mut data: Vec<i32> = (0..1_000_000).rev().collect();
    
    let start = Instant::now();
    data.par_sort();
    println!("Parallel sort: {:?}", start.elapsed());
    
    // Verify sorted
    assert!(data.windows(2).all(|w| w[0] <= w[1]));

    // Parallel for_each
    let counter = std::sync::atomic::AtomicU64::new(0);
    
    (0..1_000_000u64).into_par_iter().for_each(|_| {
        counter.fetch_add(1, std::sync::atomic::Ordering::Relaxed);
    });
    
    println!("Counter: {}", counter.load(std::sync::atomic::Ordering::Relaxed));

    // Parallel fold + reduce
    let start = Instant::now();
    let total: i64 = (1..=1_000_000i64).into_par_iter()
        .fold(|| 0i64, |acc, x| acc + x)
        .reduce(|| 0, |a, b| a + b);
    println!("Parallel fold: {} in {:?}", total, start.elapsed());

    // ParallelBridge - convert sequential to parallel
    let results: Vec<String> = (0..1000)
        .map(|i| format!("item_{}", i))
        .par_bridge()
        .filter(|s| s.contains("5"))
        .collect();
    println!("Items with '5': {}", results.len());

    // Custom thread pool
    let pool = rayon::ThreadPoolBuilder::new()
        .num_threads(2)
        .build()
        .unwrap();
    
    let result: i64 = pool.install(|| {
        (1..=1000i64).into_par_iter().sum()
    });
    println!("Custom pool result: {}", result);
}

fn is_prime(n: u64) -> bool {
    if n < 2 { return false; }
    if n == 2 { return true; }
    if n % 2 == 0 { return false; }
    let sqrt = (n as f64).sqrt() as u64;
    (3..=sqrt).step_by(2).all(|i| n % i != 0)
}
```

---

## 10. Atomic Operations

```rust
use std::sync::atomic::{AtomicBool, AtomicI32, AtomicU64, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    // AtomicU64 counter - faster than Mutex<u64> for simple counters
    let counter = Arc::new(AtomicU64::new(0));
    let mut handles = Vec::new();
    
    for _ in 0..10 {
        let c = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..1000 {
                c.fetch_add(1, Ordering::Relaxed);
            }
        }));
    }
    
    for h in handles { h.join().unwrap(); }
    println!("Counter: {}", counter.load(Ordering::SeqCst));  // 10000

    // AtomicBool - thread-safe flag
    let should_stop = Arc::new(AtomicBool::new(false));
    let flag = Arc::clone(&should_stop);
    
    let worker = thread::spawn(move || {
        let mut i = 0;
        while !flag.load(Ordering::Relaxed) {
            i += 1;
            if i % 100000 == 0 {
                println!("Worker iteration {}", i);
            }
        }
        println!("Worker stopped at iteration {}", i);
    });
    
    thread::sleep(std::time::Duration::from_millis(50));
    should_stop.store(true, Ordering::Relaxed);
    worker.join().unwrap();

    // Compare and swap (CAS)
    let value = Arc::new(AtomicI32::new(0));
    let v = Arc::clone(&value);
    
    let h = thread::spawn(move || {
        // Only increment if current value is 0
        match v.compare_exchange(0, 1, Ordering::SeqCst, Ordering::SeqCst) {
            Ok(old) => println!("CAS success: {} -> 1", old),
            Err(current) => println!("CAS failed: current is {}", current),
        }
    });
    
    h.join().unwrap();
    println!("Final value: {}", value.load(Ordering::SeqCst));

    // Ordering levels:
    // Relaxed: no ordering guarantees, only atomicity
    // Acquire/Release: synchronization between threads
    // SeqCst: sequential consistency (strongest, most expensive)
}
```

---

## 11. Practical: Parallel File Processing

```rust
use std::fs;
use std::io::{self, BufRead, BufReader};
use std::path::{Path, PathBuf};
use std::sync::{Arc, Mutex};
use std::thread;
use std::sync::mpsc;

#[derive(Debug, Default)]
struct FileStats {
    path: String,
    line_count: usize,
    word_count: usize,
    char_count: usize,
    error_lines: usize,
    warning_lines: usize,
}

fn analyze_file(path: &Path) -> io::Result<FileStats> {
    let file = fs::File::open(path)?;
    let reader = BufReader::new(file);
    
    let mut stats = FileStats {
        path: path.to_string_lossy().to_string(),
        ..Default::default()
    };
    
    for line in reader.lines() {
        let line = line?;
        stats.line_count += 1;
        stats.word_count += line.split_whitespace().count();
        stats.char_count += line.chars().count();
        
        let lower = line.to_lowercase();
        if lower.contains("error") { stats.error_lines += 1; }
        if lower.contains("warn") { stats.warning_lines += 1; }
    }
    
    Ok(stats)
}

fn process_files_parallel(paths: Vec<PathBuf>, num_threads: usize) -> Vec<FileStats> {
    let (task_tx, task_rx) = mpsc::channel::<PathBuf>();
    let task_rx = Arc::new(Mutex::new(task_rx));
    let (result_tx, result_rx) = mpsc::channel::<FileStats>();
    
    // Spawn worker threads
    let mut handles = Vec::new();
    for worker_id in 0..num_threads {
        let rx = Arc::clone(&task_rx);
        let tx = result_tx.clone();
        
        let h = thread::Builder::new()
            .name(format!("file-worker-{}", worker_id))
            .spawn(move || {
                loop {
                    let path = rx.lock().unwrap().recv();
                    match path {
                        Ok(path) => {
                            match analyze_file(&path) {
                                Ok(stats) => {
                                    tx.send(stats).ok();
                                }
                                Err(e) => {
                                    eprintln!("Error processing {:?}: {}", path, e);
                                }
                            }
                        }
                        Err(_) => break,  // Channel closed
                    }
                }
            })
            .unwrap();
        
        handles.push(h);
    }
    
    drop(result_tx);  // Drop extra sender
    
    // Send tasks
    let total = paths.len();
    for path in paths {
        task_tx.send(path).unwrap();
    }
    drop(task_tx);  // Signal workers to stop
    
    // Collect results
    let mut results = Vec::new();
    for stats in result_rx {
        results.push(stats);
        print!("\rProcessed: {}/{}", results.len(), total);
        io::Write::flush(&mut io::stdout()).ok();
    }
    println!();
    
    // Wait for workers
    for h in handles {
        h.join().ok();
    }
    
    results
}

fn generate_test_files(dir: &str, count: usize) -> io::Result<Vec<PathBuf>> {
    fs::create_dir_all(dir)?;
    let mut paths = Vec::new();
    
    for i in 0..count {
        let path = PathBuf::from(format!("{}/file_{:04}.log", dir, i));
        
        let mut content = String::new();
        for j in 0..100 {
            let prefix = match j % 10 {
                0 => "ERROR",
                1 | 2 => "WARN",
                _ => "INFO",
            };
            content.push_str(&format!("{}: Line {} in file {}\n", prefix, j, i));
        }
        
        fs::write(&path, content)?;
        paths.push(path);
    }
    
    Ok(paths)
}

fn print_summary(stats: &[FileStats]) {
    let total_lines: usize = stats.iter().map(|s| s.line_count).sum();
    let total_words: usize = stats.iter().map(|s| s.word_count).sum();
    let total_errors: usize = stats.iter().map(|s| s.error_lines).sum();
    let total_warnings: usize = stats.iter().map(|s| s.warning_lines).sum();
    
    println!("\n=== Parallel File Processing Summary ===");
    println!("Files processed: {}", stats.len());
    println!("Total lines:     {}", total_lines);
    println!("Total words:     {}", total_words);
    println!("Error lines:     {}", total_errors);
    println!("Warning lines:   {}", total_warnings);
    
    let avg_lines = if !stats.is_empty() {
        total_lines / stats.len()
    } else { 0 };
    println!("Avg lines/file:  {}", avg_lines);
    
    // Top files by error count
    let mut sorted_stats = stats.to_vec();
    sorted_stats.sort_by(|a, b| b.error_lines.cmp(&a.error_lines));
    
    println!("\nTop 3 files by errors:");
    for s in sorted_stats.iter().take(3) {
        println!("  {} - {} errors", s.path, s.error_lines);
    }
}

fn main() -> io::Result<()> {
    let test_dir = "test_logs";
    let num_files = 50;
    let num_threads = 4;
    
    // Generate test files
    println!("Generating {} test log files...", num_files);
    let paths = generate_test_files(test_dir, num_files)?;
    
    // Sequential processing (for comparison)
    let seq_start = std::time::Instant::now();
    let mut seq_results = Vec::new();
    for path in &paths {
        if let Ok(stats) = analyze_file(path) {
            seq_results.push(stats);
        }
    }
    let seq_time = seq_start.elapsed();
    println!("Sequential: {:?}", seq_time);
    
    // Parallel processing
    let par_start = std::time::Instant::now();
    let par_results = process_files_parallel(paths, num_threads);
    let par_time = par_start.elapsed();
    println!("Parallel ({} threads): {:?}", num_threads, par_time);
    println!("Speedup: {:.2}x", seq_time.as_secs_f64() / par_time.as_secs_f64());
    
    print_summary(&par_results);
    
    // Cleanup
    fs::remove_dir_all(test_dir)?;
    println!("\nCleanup complete");
    
    Ok(())
}

impl Clone for FileStats {
    fn clone(&self) -> Self {
        FileStats {
            path: self.path.clone(),
            line_count: self.line_count,
            word_count: self.word_count,
            char_count: self.char_count,
            error_lines: self.error_lines,
            warning_lines: self.warning_lines,
        }
    }
}
```

---

## 12. Send และ Sync Traits

```rust
// Send: type ที่สามารถ transfer ownership ข้าม threads ได้
// Sync: type ที่สามารถแชร์ reference ข้าม threads ได้

use std::sync::{Arc, Mutex};
use std::thread;

// Custom type ที่ implement Send
struct MyData {
    value: i32,
}
// MyData เป็น Send โดยอัตโนมัติถ้า fields ทั้งหมดเป็น Send

// ไม่เป็น Send: Rc<T> เพราะ reference count ไม่ thread-safe
// ใช้ Arc<T> แทน

fn requires_send<T: Send>(value: T) -> thread::JoinHandle<T> {
    thread::spawn(move || value)
}

fn requires_sync<T: Sync + Send>(value: Arc<T>) -> Vec<thread::JoinHandle<()>> {
    let mut handles = Vec::new();
    for _ in 0..3 {
        let v = Arc::clone(&value);
        handles.push(thread::spawn(move || {
            // Read value from multiple threads
            let _ = &*v;
        }));
    }
    handles
}

fn main() {
    // Send example
    let data = MyData { value: 42 };
    let handle = requires_send(data);
    let returned = handle.join().unwrap();
    println!("Returned: {}", returned.value);
    
    // Sync example
    let shared = Arc::new(Mutex::new(vec![1, 2, 3]));
    let handles = requires_sync(Arc::clone(&shared));
    for h in handles { h.join().unwrap(); }
    
    // Rc is NOT Send - compile error:
    // let rc = std::rc::Rc::new(42);
    // thread::spawn(move || println!("{}", rc));  // ERROR
    
    // Arc IS Send:
    let arc = Arc::new(42);
    thread::spawn(move || println!("Arc value: {}", arc)).join().unwrap();
    
    // Cell/RefCell are NOT Sync (but are Send)
    // Mutex<T> IS Sync (when T: Send)
}
```

---

## 13. สรุป

| Concept | Type/Function | ใช้เมื่อ |
|---------|---------------|---------|
| Spawn thread | `thread::spawn` | ต้องการทำงานพร้อมกัน |
| Join thread | `handle.join()` | รอ thread เสร็จ |
| Shared counter | `Arc<Mutex<T>>` | แชร์ mutable state |
| Read-heavy | `Arc<RwLock<T>>` | อ่านบ่อย เขียนน้อย |
| Channel | `mpsc::channel` | ส่ง data ระหว่าง threads |
| Sync channel | `mpsc::sync_channel` | bounded buffer |
| Atomic ops | `AtomicU64`, `AtomicBool` | counter/flag แบบ lock-free |
| Data parallel | `rayon::par_iter()` | process collection ขนาดใหญ่ |
| Thread pool | manual หรือ rayon | จำกัดจำนวน threads |

### Concurrency Golden Rules

1. **ส่งข้อมูลผ่าน channels** แทนการแชร์ memory
2. **ใช้ Arc เสมอ** เมื่อต้องแชร์ข้อมูลระหว่าง threads
3. **Lock ให้สั้นที่สุด** - ปล่อย lock เร็ว
4. **Lock ตามลำดับคงที่** - ป้องกัน deadlock
5. **ใช้ Rayon** สำหรับ data parallelism ที่ง่าย
6. **Atomic** สำหรับ counters/flags แทน Mutex

---

*[← Part 014: File I/O และ Filesystem](../part_014/README.md) | [Part 016: Async/Await และ Tokio →](../part_016/README.md)*
