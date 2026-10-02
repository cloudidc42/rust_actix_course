# Part 054: Background Jobs with Tokio ⚙️

## 🎯 เป้าหมายของ Part นี้

- tokio::spawn สำหรับ background tasks
- Task scheduling ด้วย tokio-cron-scheduler
- Job queue patterns
- Worker pool
- Job status tracking
- Retry logic
- Dead letter queue
- สร้าง Email sending queue

---

## 1. Setup

```toml
# Cargo.toml
[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
tokio-cron-scheduler = "0.10"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }
async-trait = "0.1"
thiserror = "1"
tokio-util = "0.7"
futures-util = "0.3"
```

---

## 2. tokio::spawn Basics

```rust
// src/basic_spawn.rs
use tokio::time::{sleep, Duration};

// Simple background task
pub async fn run_background_task() {
    let handle = tokio::spawn(async {
        println!("Background task started");
        sleep(Duration::from_secs(5)).await;
        println!("Background task completed");
        42 // return value
    });

    // ทำงานอื่นต่อระหว่างรอ...
    println!("Main task continues...");

    // รอผลลัพธ์
    match handle.await {
        Ok(result) => println!("Task returned: {}", result),
        Err(e) => eprintln!("Task panicked: {:?}", e),
    }
}

// Spawn task ที่ไม่ต้องรอผล
pub fn fire_and_forget(data: String) {
    tokio::spawn(async move {
        // ทำงานใน background
        process_data(data).await;
    });
}

async fn process_data(data: String) {
    sleep(Duration::from_millis(100)).await;
    println!("Processed: {}", data);
}

// Spawn หลาย tasks พร้อมกัน
pub async fn parallel_tasks() {
    let tasks: Vec<_> = (0..10)
        .map(|i| {
            tokio::spawn(async move {
                sleep(Duration::from_millis(100)).await;
                i * i
            })
        })
        .collect();

    let results = futures_util::future::join_all(tasks).await;

    for (i, result) in results.iter().enumerate() {
        match result {
            Ok(v) => println!("Task {}: {}", i, v),
            Err(e) => eprintln!("Task {} failed: {:?}", i, e),
        }
    }
}
```

---

## 3. Job Queue System

```rust
// src/job_queue.rs
use async_trait::async_trait;
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use tokio::sync::mpsc;
use uuid::Uuid;
use chrono::{DateTime, Utc};

// Job trait
#[async_trait]
pub trait Job: Send + Sync + std::fmt::Debug {
    fn job_type(&self) -> &'static str;
    async fn execute(&self) -> Result<(), JobError>;
    fn max_retries(&self) -> u32 { 3 }
    fn retry_delay_ms(&self) -> u64 { 1000 }
}

#[derive(Debug, thiserror::Error)]
pub enum JobError {
    #[error("Job failed: {0}")]
    Failed(String),
    #[error("Job should retry: {0}")]
    Retryable(String),
    #[error("Job permanently failed: {0}")]
    Permanent(String),
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum JobStatus {
    Pending,
    Running,
    Completed,
    Failed,
    DeadLetter,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct JobRecord {
    pub id: String,
    pub job_type: String,
    pub payload: serde_json::Value,
    pub status: JobStatus,
    pub attempts: u32,
    pub max_retries: u32,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub scheduled_at: DateTime<Utc>,
    pub completed_at: Option<DateTime<Utc>>,
    pub error: Option<String>,
}

impl JobRecord {
    pub fn new(job_type: &str, payload: serde_json::Value, max_retries: u32) -> Self {
        let now = Utc::now();
        Self {
            id: Uuid::new_v4().to_string(),
            job_type: job_type.to_string(),
            payload,
            status: JobStatus::Pending,
            attempts: 0,
            max_retries,
            created_at: now,
            updated_at: now,
            scheduled_at: now,
            completed_at: None,
            error: None,
        }
    }
}
```

---

## 4. Worker Pool

```rust
// src/worker_pool.rs
use std::sync::Arc;
use tokio::sync::{mpsc, Semaphore};
use crate::job_queue::{Job, JobRecord, JobStatus, JobError};

pub struct WorkerPool {
    sender: mpsc::Sender<Box<dyn Job>>,
    semaphore: Arc<Semaphore>,
}

impl WorkerPool {
    pub fn new(workers: usize, queue_size: usize) -> Self {
        let (tx, mut rx) = mpsc::channel::<Box<dyn Job>>(queue_size);
        let semaphore = Arc::new(Semaphore::new(workers));

        // Start worker loop
        tokio::spawn(async move {
            while let Some(job) = rx.recv().await {
                let sem = semaphore.clone();

                tokio::spawn(async move {
                    // Acquire permit (rate limiting)
                    let _permit = sem.acquire().await.unwrap();

                    println!("[Worker] Executing job: {}", job.job_type());

                    match job.execute().await {
                        Ok(_) => println!("[Worker] Job completed: {}", job.job_type()),
                        Err(e) => eprintln!("[Worker] Job failed: {} - {:?}", job.job_type(), e),
                    }
                });
            }
        });

        Self {
            sender: tx,
            semaphore: Arc::new(Semaphore::new(workers)),
        }
    }

    pub async fn submit(&self, job: Box<dyn Job>) -> Result<(), &'static str> {
        self.sender
            .send(job)
            .await
            .map_err(|_| "Queue is full or closed")
    }
}

// Advanced worker pool with retry
pub struct RetryWorkerPool {
    inner_tx: mpsc::Sender<WorkerMessage>,
}

enum WorkerMessage {
    Job(Box<dyn Job>, JobRecord),
    Shutdown,
}

impl RetryWorkerPool {
    pub fn new(workers: usize) -> Self {
        let (tx, mut rx) = mpsc::channel::<WorkerMessage>(1000);

        for worker_id in 0..workers {
            let mut worker_rx = rx.clone(); // Note: would need to use separate channels
            tokio::spawn(async move {
                println!("[Worker {}] Started", worker_id);
                // worker loop would go here
            });
        }

        Self { inner_tx: tx }
    }
}
```

---

## 5. Job with Retry Logic

```rust
// src/retry.rs
use tokio::time::{sleep, Duration};
use crate::job_queue::{Job, JobError, JobRecord, JobStatus};

pub async fn execute_with_retry<J: Job>(
    job: &J,
    record: &mut JobRecord,
) -> Result<(), JobError> {
    let max_retries = job.max_retries();

    loop {
        record.attempts += 1;
        record.status = JobStatus::Running;
        record.updated_at = chrono::Utc::now();

        match job.execute().await {
            Ok(_) => {
                record.status = JobStatus::Completed;
                record.completed_at = Some(chrono::Utc::now());
                return Ok(());
            }

            Err(JobError::Permanent(e)) => {
                record.status = JobStatus::DeadLetter;
                record.error = Some(e.clone());
                return Err(JobError::Permanent(e));
            }

            Err(JobError::Retryable(e)) if record.attempts <= max_retries => {
                // Exponential backoff
                let delay = exponential_backoff(record.attempts, job.retry_delay_ms());
                eprintln!(
                    "[Retry] Job {} failed (attempt {}/{}), retrying in {}ms: {}",
                    record.id, record.attempts, max_retries, delay, e
                );

                record.error = Some(e);
                record.status = JobStatus::Pending;

                sleep(Duration::from_millis(delay)).await;
            }

            Err(e) => {
                // Max retries exceeded
                record.status = JobStatus::DeadLetter;
                record.error = Some(e.to_string());
                return Err(e);
            }
        }
    }
}

fn exponential_backoff(attempt: u32, base_delay_ms: u64) -> u64 {
    let multiplier = 2u64.pow(attempt - 1);
    let delay = base_delay_ms * multiplier;

    // Jitter: ±10%
    let jitter = (delay as f64 * 0.1 * rand_f64()) as u64;
    delay + jitter
}

fn rand_f64() -> f64 {
    // Simple pseudo-random
    use std::time::{SystemTime, UNIX_EPOCH};
    let nanos = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .subsec_nanos();
    (nanos % 1000) as f64 / 1000.0
}
```

---

## 6. Task Scheduling

```rust
// src/scheduler.rs
use tokio_cron_scheduler::{JobScheduler, Job};
use std::sync::Arc;

pub async fn setup_scheduler() -> anyhow::Result<JobScheduler> {
    let sched = JobScheduler::new().await?;

    // รัน ทุก 1 นาที
    sched
        .add(Job::new("0 * * * * *", |uuid, _lock| {
            Box::pin(async move {
                println!("[Scheduler] Running every minute job: {:?}", uuid);
                // TODO: run cleanup tasks, etc.
            })
        })?)
        .await?;

    // รัน ทุก 5 นาที
    sched
        .add(Job::new("0 */5 * * * *", |_uuid, _lock| {
            Box::pin(async move {
                println!("[Scheduler] Running every 5 minutes job");
                // process_pending_emails().await;
            })
        })?)
        .await?;

    // รัน ทุกวัน เที่ยงคืน
    sched
        .add(Job::new("0 0 0 * * *", |_uuid, _lock| {
            Box::pin(async move {
                println!("[Scheduler] Running daily midnight job");
                // cleanup_old_data().await;
            })
        })?)
        .await?;

    // One-time job (รันครั้งเดียว)
    sched
        .add(Job::new_one_shot(
            std::time::Duration::from_secs(10),
            |_uuid, _lock| {
                Box::pin(async move {
                    println!("[Scheduler] One-shot job executed");
                })
            },
        )?)
        .await?;

    sched.start().await?;

    Ok(sched)
}
```

---

## 7. Dead Letter Queue

```rust
// src/dlq.rs
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use tokio::sync::RwLock;
use crate::job_queue::JobRecord;

pub struct DeadLetterQueue {
    jobs: Arc<RwLock<Vec<DeadLetterEntry>>>,
    max_size: usize,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DeadLetterEntry {
    pub job: JobRecord,
    pub dead_at: chrono::DateTime<chrono::Utc>,
    pub reason: String,
}

impl DeadLetterQueue {
    pub fn new(max_size: usize) -> Self {
        Self {
            jobs: Arc::new(RwLock::new(Vec::new())),
            max_size,
        }
    }

    pub async fn push(&self, job: JobRecord, reason: &str) {
        let mut queue = self.jobs.write().await;

        if queue.len() >= self.max_size {
            // Remove oldest entry
            queue.remove(0);
        }

        queue.push(DeadLetterEntry {
            job,
            dead_at: chrono::Utc::now(),
            reason: reason.to_string(),
        });
    }

    pub async fn list(&self) -> Vec<DeadLetterEntry> {
        self.jobs.read().await.clone()
    }

    pub async fn replay(&self, job_id: &str) -> Option<DeadLetterEntry> {
        let mut queue = self.jobs.write().await;

        if let Some(pos) = queue.iter().position(|e| e.job.id == job_id) {
            Some(queue.remove(pos))
        } else {
            None
        }
    }

    pub async fn clear(&self) {
        self.jobs.write().await.clear();
    }
}
```

---

## 8. Practical: Email Sending Queue

```rust
// src/email_queue.rs
use async_trait::async_trait;
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use tokio::sync::{mpsc, RwLock};
use uuid::Uuid;
use chrono::{DateTime, Utc};
use crate::job_queue::{Job, JobError, JobRecord, JobStatus};

// Email job
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SendEmailJob {
    pub to: String,
    pub subject: String,
    pub html_body: String,
    pub text_body: Option<String>,
}

#[async_trait]
impl Job for SendEmailJob {
    fn job_type(&self) -> &'static str { "send_email" }

    async fn execute(&self) -> Result<(), JobError> {
        // ในที่นี้จะ simulate การส่ง email
        println!(
            "[EmailJob] Sending email to: {} subject: {}",
            self.to, self.subject
        );

        // Simulate network call
        tokio::time::sleep(std::time::Duration::from_millis(50)).await;

        // Simulate random failure (10% chance)
        if rand_bool(0.1) {
            return Err(JobError::Retryable(
                "SMTP connection timeout".to_string()
            ));
        }

        println!("[EmailJob] Email sent successfully to: {}", self.to);
        Ok(())
    }

    fn max_retries(&self) -> u32 { 3 }
    fn retry_delay_ms(&self) -> u64 { 2000 }
}

fn rand_bool(probability: f64) -> bool {
    use std::time::{SystemTime, UNIX_EPOCH};
    let nanos = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .subsec_nanos();
    (nanos % 100) as f64 / 100.0 < probability
}

// Email queue manager
pub struct EmailQueueManager {
    queue: Arc<RwLock<Vec<JobRecord>>>,
    sender: mpsc::Sender<JobRecord>,
}

impl EmailQueueManager {
    pub fn new() -> Self {
        let queue = Arc::new(RwLock::new(Vec::new()));
        let (tx, mut rx) = mpsc::channel::<JobRecord>(1000);

        let queue_clone = queue.clone();

        // Start worker
        tokio::spawn(async move {
            println!("[EmailQueue] Worker started");

            while let Some(mut record) = rx.recv().await {
                // Update status to running
                {
                    let mut q = queue_clone.write().await;
                    if let Some(r) = q.iter_mut().find(|r| r.id == record.id) {
                        r.status = JobStatus::Running;
                    }
                }

                // Deserialize job
                let job: SendEmailJob = serde_json::from_value(record.payload.clone())
                    .unwrap_or_else(|e| {
                        eprintln!("[EmailQueue] Failed to deserialize job: {}", e);
                        SendEmailJob {
                            to: String::new(),
                            subject: String::new(),
                            html_body: String::new(),
                            text_body: None,
                        }
                    });

                // Execute with retry
                let max_retries = job.max_retries();
                let base_delay = job.retry_delay_ms();
                let mut attempts = 0u32;
                let mut success = false;

                while attempts <= max_retries {
                    attempts += 1;
                    match job.execute().await {
                        Ok(_) => {
                            success = true;
                            break;
                        }
                        Err(JobError::Permanent(e)) => {
                            record.error = Some(e);
                            break;
                        }
                        Err(e) if attempts > max_retries => {
                            record.error = Some(e.to_string());
                            break;
                        }
                        Err(e) => {
                            let delay = base_delay * 2u64.pow(attempts - 1);
                            eprintln!(
                                "[EmailQueue] Retry {}/{} after {}ms: {}",
                                attempts, max_retries, delay, e
                            );
                            tokio::time::sleep(
                                std::time::Duration::from_millis(delay)
                            ).await;
                        }
                    }
                }

                // Update final status
                let final_status = if success {
                    JobStatus::Completed
                } else {
                    JobStatus::DeadLetter
                };

                let mut q = queue_clone.write().await;
                if let Some(r) = q.iter_mut().find(|r| r.id == record.id) {
                    r.status = final_status;
                    r.attempts = attempts;
                    r.updated_at = Utc::now();
                    if success {
                        r.completed_at = Some(Utc::now());
                    } else {
                        r.error = record.error.clone();
                    }
                }
            }
        });

        Self {
            queue,
            sender: tx,
        }
    }

    pub async fn enqueue(
        &self,
        to: &str,
        subject: &str,
        html_body: &str,
    ) -> anyhow::Result<String> {
        let job = SendEmailJob {
            to: to.to_string(),
            subject: subject.to_string(),
            html_body: html_body.to_string(),
            text_body: None,
        };

        let record = JobRecord::new(
            "send_email",
            serde_json::to_value(&job)?,
            job.max_retries(),
        );
        let job_id = record.id.clone();

        self.queue.write().await.push(record.clone());
        self.sender.send(record).await?;

        Ok(job_id)
    }

    pub async fn get_status(&self, job_id: &str) -> Option<JobRecord> {
        self.queue
            .read()
            .await
            .iter()
            .find(|r| r.id == job_id)
            .cloned()
    }

    pub async fn list_jobs(&self) -> Vec<JobRecord> {
        self.queue.read().await.clone()
    }
}

// Actix-web handlers
use actix_web::{web, HttpResponse};

pub async fn enqueue_email(
    queue: web::Data<EmailQueueManager>,
    body: web::Json<SendEmailRequest>,
) -> HttpResponse {
    match queue
        .enqueue(&body.to, &body.subject, &body.html_body)
        .await
    {
        Ok(job_id) => HttpResponse::Accepted().json(serde_json::json!({
            "job_id": job_id,
            "status": "queued"
        })),
        Err(e) => HttpResponse::InternalServerError().json(serde_json::json!({
            "error": e.to_string()
        })),
    }
}

pub async fn get_job_status(
    queue: web::Data<EmailQueueManager>,
    path: web::Path<String>,
) -> HttpResponse {
    let job_id = path.into_inner();

    match queue.get_status(&job_id).await {
        Some(record) => HttpResponse::Ok().json(record),
        None => HttpResponse::NotFound().json(serde_json::json!({
            "error": "Job not found"
        })),
    }
}

pub async fn list_jobs(
    queue: web::Data<EmailQueueManager>,
) -> HttpResponse {
    let jobs = queue.list_jobs().await;
    HttpResponse::Ok().json(jobs)
}

#[derive(Deserialize)]
pub struct SendEmailRequest {
    pub to: String,
    pub subject: String,
    pub html_body: String,
}

use serde::Deserialize;

// Main
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    use actix_web::{App, HttpServer};

    let email_queue = web::Data::new(EmailQueueManager::new());

    HttpServer::new(move || {
        App::new()
            .app_data(email_queue.clone())
            .route("/api/emails", web::post().to(enqueue_email))
            .route("/api/emails", web::get().to(list_jobs))
            .route("/api/emails/{id}", web::get().to(get_job_status))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## สรุป

✅ tokio::spawn สำหรับ background tasks  
✅ Task scheduling ด้วย tokio-cron-scheduler  
✅ Job queue pattern  
✅ Worker pool พร้อม semaphore  
✅ Retry logic ด้วย exponential backoff  
✅ Dead letter queue  
✅ Email queue ตัวอย่าง  

### Exercise

1. เพิ่ม priority queue (high/medium/low priority)
2. สร้าง job dashboard ด้วย SSE
3. Implement distributed lock สำหรับ cron jobs
4. เพิ่ม job chaining (รัน job ต่อจาก job อื่น)

---

*[← Part 053: File Streaming and Upload](../part_053/README.md) | [Part 055: Email Service →](../part_055/README.md)*
