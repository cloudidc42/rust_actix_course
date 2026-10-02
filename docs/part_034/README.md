# Part 034: Connection Pooling

## สารบัญ
- [PgPool Setup และ Configuration](#pgpool-setup-และ-configuration)
- [Pool Size Tuning](#pool-size-tuning)
- [Connection Timeout](#connection-timeout)
- [Acquire/Release Lifecycle](#acquirerelease-lifecycle)
- [Health Check Queries](#health-check-queries)
- [Pool Metrics](#pool-metrics)
- [Multiple Database Pools](#multiple-database-pools)
- [Environment Configuration](#environment-configuration)

---

## ทำไมต้องใช้ Connection Pooling?

Connection Pooling คือการเก็บ database connections ที่สร้างไว้แล้วในพูล เพื่อนำมาใช้ซ้ำแทนที่จะสร้างใหม่ทุกครั้ง

**ปัญหาถ้าไม่มี Connection Pool:**
- แต่ละ request สร้าง connection ใหม่ → ช้า (50-100ms ต่อ connection)
- Database มี limit จำนวน connections
- ถ้า request มาพร้อมกันเยอะ → database overload

**ประโยชน์ของ Connection Pool:**
- Reuse connections ที่มีอยู่ → เร็วกว่ามาก
- จำกัดจำนวน connections ที่ database รับ
- Backpressure เมื่อ connections เต็ม

---

## PgPool Setup และ Configuration

### การสร้าง Pool พื้นฐาน

```rust
// src/main.rs
use sqlx::postgres::PgPoolOptions;
use sqlx::PgPool;
use std::time::Duration;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    dotenv::dotenv().ok();
    
    let database_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    
    // สร้าง pool แบบง่าย
    let pool = PgPool::connect(&database_url)
        .await
        .expect("Failed to connect to database");
    
    // หรือแบบ custom options
    let pool = PgPoolOptions::new()
        .max_connections(20)
        .min_connections(5)
        .acquire_timeout(Duration::from_secs(30))
        .idle_timeout(Duration::from_secs(600))
        .max_lifetime(Duration::from_secs(1800))
        .connect(&database_url)
        .await
        .expect("Failed to create connection pool");
    
    println!("Database pool created successfully");
    
    Ok(())
}
```

### Pool Options อธิบาย

```rust
// src/db/pool.rs
use sqlx::postgres::PgPoolOptions;
use sqlx::PgPool;
use std::time::Duration;

pub async fn create_pool(database_url: &str) -> Result<PgPool, sqlx::Error> {
    PgPoolOptions::new()
        // จำนวน connections สูงสุด
        // ควรตั้งตาม: database max_connections / จำนวน app instances
        .max_connections(20)
        
        // จำนวน connections ขั้นต่ำที่ maintain ไว้
        // ช่วยลด latency เมื่อ traffic เพิ่มขึ้น
        .min_connections(5)
        
        // timeout เมื่อรอ connection จาก pool
        // ถ้าเกินเวลา → error แทนที่จะรอนาน
        .acquire_timeout(Duration::from_secs(30))
        
        // เวลาที่ connection ถือว่า idle และสามารถปิดได้
        // None = ไม่มี timeout (default)
        .idle_timeout(Duration::from_secs(600))  // 10 นาที
        
        // อายุสูงสุดของ connection ก่อนที่จะสร้างใหม่
        // ช่วยป้องกัน stale connections
        .max_lifetime(Duration::from_secs(1800))  // 30 นาที
        
        // Test connection ก่อนให้ใช้งาน
        // ป้องกันการได้ connection ที่ dead
        .test_before_acquire(true)
        
        .connect(database_url)
        .await
}
```

### Connection String Options

```rust
// src/db/pool.rs (ต่อ)
use sqlx::postgres::PgConnectOptions;
use std::str::FromStr;

pub async fn create_pool_with_options(database_url: &str) -> Result<PgPool, sqlx::Error> {
    let connect_options = PgConnectOptions::from_str(database_url)?
        // Application name สำหรับ monitoring
        .application_name("my-blog-api")
        
        // Statement cache size (ต่อ connection)
        .statement_cache_capacity(100)
        
        // SSL mode
        // .ssl_mode(sqlx::postgres::PgSslMode::Require)
        
        // Timezone
        .timezone("UTC".to_owned())
        
        // Extra options
        .extra_float_digits(2);
    
    PgPoolOptions::new()
        .max_connections(20)
        .connect_with(connect_options)
        .await
}
```

---

## Pool Size Tuning

### การคำนวณขนาด Pool ที่เหมาะสม

```rust
// src/config/database.rs
use serde::Deserialize;
use std::env;

#[derive(Debug, Clone, Deserialize)]
pub struct PoolConfig {
    pub max_connections: u32,
    pub min_connections: u32,
    pub acquire_timeout_seconds: u64,
    pub idle_timeout_seconds: u64,
    pub max_lifetime_seconds: u64,
}

impl Default for PoolConfig {
    fn default() -> Self {
        // สูตรคำนวณ: 
        // max_connections = (cpu_cores * 2) + effective_spindle_count
        // สำหรับ web API: cpu_cores * 4 ถึง cpu_cores * 10
        
        let cpu_count = num_cpus::get() as u32;
        
        Self {
            max_connections: cpu_count * 4,  // 4 connections per CPU core
            min_connections: cpu_count,       // 1 connection per CPU core (minimum)
            acquire_timeout_seconds: 30,
            idle_timeout_seconds: 600,
            max_lifetime_seconds: 1800,
        }
    }
}

impl PoolConfig {
    pub fn from_env() -> Self {
        let default = Self::default();
        
        Self {
            max_connections: env::var("DB_POOL_MAX")
                .ok()
                .and_then(|v| v.parse().ok())
                .unwrap_or(default.max_connections),
                
            min_connections: env::var("DB_POOL_MIN")
                .ok()
                .and_then(|v| v.parse().ok())
                .unwrap_or(default.min_connections),
                
            acquire_timeout_seconds: env::var("DB_ACQUIRE_TIMEOUT")
                .ok()
                .and_then(|v| v.parse().ok())
                .unwrap_or(default.acquire_timeout_seconds),
                
            idle_timeout_seconds: env::var("DB_IDLE_TIMEOUT")
                .ok()
                .and_then(|v| v.parse().ok())
                .unwrap_or(default.idle_timeout_seconds),
                
            max_lifetime_seconds: env::var("DB_MAX_LIFETIME")
                .ok()
                .and_then(|v| v.parse().ok())
                .unwrap_or(default.max_lifetime_seconds),
        }
    }
}
```

### Pool Size สำหรับ Scenarios ต่างๆ

```rust
// ตัวอย่าง configuration ตาม load pattern

// 1. Light load (development/staging)
let light_pool = PgPoolOptions::new()
    .max_connections(5)
    .min_connections(1)
    .connect(&database_url)
    .await?;

// 2. Medium load (production web API)
let medium_pool = PgPoolOptions::new()
    .max_connections(20)
    .min_connections(5)
    .acquire_timeout(Duration::from_secs(30))
    .idle_timeout(Duration::from_secs(300))
    .connect(&database_url)
    .await?;

// 3. High load (high-traffic API)
let high_pool = PgPoolOptions::new()
    .max_connections(50)
    .min_connections(10)
    .acquire_timeout(Duration::from_secs(10))  // เร็วขึ้น fail fast
    .idle_timeout(Duration::from_secs(120))
    .connect(&database_url)
    .await?;

// 4. Background jobs (batch processing)
let batch_pool = PgPoolOptions::new()
    .max_connections(5)
    .min_connections(1)
    .acquire_timeout(Duration::from_secs(60))  // รอนานได้
    .connect(&database_url)
    .await?;
```

---

## Connection Timeout

### Handling Timeout Errors

```rust
// src/db/pool.rs (ต่อ)
use sqlx::error::Error as SqlxError;
use std::time::Duration;

pub async fn acquire_with_retry(pool: &PgPool) -> Result<sqlx::pool::PoolConnection<sqlx::Postgres>, AppError> {
    let mut attempts = 0;
    let max_attempts = 3;
    
    loop {
        match pool.acquire().await {
            Ok(conn) => return Ok(conn),
            Err(e) => {
                attempts += 1;
                
                if attempts >= max_attempts {
                    log::error!("Failed to acquire connection after {} attempts: {}", attempts, e);
                    return Err(AppError::Database(e));
                }
                
                // ตรวจสอบว่าเป็น timeout error
                match &e {
                    SqlxError::PoolTimedOut => {
                        log::warn!("Pool timeout, retrying... ({}/{})", attempts, max_attempts);
                        tokio::time::sleep(Duration::from_millis(100 * attempts as u64)).await;
                    }
                    _ => {
                        return Err(AppError::Database(e));
                    }
                }
            }
        }
    }
}

// Middleware สำหรับ handle pool exhaustion
pub async fn check_pool_health(pool: &PgPool) -> Result<(), AppError> {
    let pool_size = pool.size();
    let idle_size = pool.num_idle();
    
    log::debug!(
        "Pool status - size: {}, idle: {}, active: {}",
        pool_size,
        idle_size,
        pool_size - idle_size
    );
    
    // Warning เมื่อ pool ใกล้เต็ม
    if idle_size == 0 && pool_size > 0 {
        log::warn!("Connection pool exhausted! All {} connections are in use", pool_size);
    }
    
    Ok(())
}
```

### Timeout Configuration ต่าง Environment

```toml
# .env.development
DB_POOL_MAX=5
DB_POOL_MIN=1
DB_ACQUIRE_TIMEOUT=30
DB_IDLE_TIMEOUT=300
DB_MAX_LIFETIME=900

# .env.production
DB_POOL_MAX=25
DB_POOL_MIN=5
DB_ACQUIRE_TIMEOUT=10
DB_IDLE_TIMEOUT=600
DB_MAX_LIFETIME=1800

# .env.test
DB_POOL_MAX=3
DB_POOL_MIN=1
DB_ACQUIRE_TIMEOUT=5
DB_IDLE_TIMEOUT=60
DB_MAX_LIFETIME=300
```

---

## Acquire/Release Lifecycle

### วงจรชีวิตของ Connection

```rust
// src/db/lifecycle.rs

// Connection จะถูก release อัตโนมัติเมื่อ drop
pub async fn example_auto_release(pool: &PgPool) {
    {
        // Acquire connection
        let mut conn = pool.acquire().await.unwrap();
        
        // ใช้งาน
        sqlx::query!("SELECT 1")
            .execute(&mut *conn)
            .await
            .unwrap();
        
        // Connection released เมื่อ conn ออกจาก scope
    }  // <-- conn dropped here, connection returned to pool
    
    println!("Connection has been returned to pool");
}

// Manual release
pub async fn example_manual_release(pool: &PgPool) {
    let conn = pool.acquire().await.unwrap();
    
    // ทำงานบางอย่าง
    sqlx::query!("SELECT 1")
        .execute(&*conn)
        .await
        .unwrap();
    
    // Return connection to pool explicitly
    drop(conn);
    
    // หรือใช้ close() เพื่อ close connection (ไม่ return to pool)
    // conn.close().await; // ปิด connection แทนที่จะ return to pool
}

// Transaction lifecycle
pub async fn example_transaction_lifecycle(pool: &PgPool) {
    let mut tx = pool.begin().await.unwrap();
    
    // ทำงานใน transaction
    sqlx::query!("INSERT INTO users (email, username, password_hash, display_name) VALUES ('test@example.com', 'test', 'hash', 'Test')")
        .execute(&mut *tx)
        .await
        .unwrap();
    
    // Commit → connection returned to pool
    tx.commit().await.unwrap();
    
    // หรือ Rollback → connection returned to pool
    // tx.rollback().await.unwrap();
    
    // ถ้า drop tx โดยไม่ commit/rollback → auto rollback
}
```

### Pool Event Listener (Monitoring)

```rust
// src/db/monitoring.rs
use sqlx::postgres::PgPool;
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;

#[derive(Debug, Default)]
pub struct PoolMetrics {
    pub connections_created: AtomicU64,
    pub connections_closed: AtomicU64,
    pub queries_executed: AtomicU64,
    pub query_errors: AtomicU64,
}

impl PoolMetrics {
    pub fn new() -> Arc<Self> {
        Arc::new(Self::default())
    }
    
    pub fn record_connection_created(&self) {
        self.connections_created.fetch_add(1, Ordering::Relaxed);
    }
    
    pub fn record_connection_closed(&self) {
        self.connections_closed.fetch_add(1, Ordering::Relaxed);
    }
    
    pub fn record_query(&self) {
        self.queries_executed.fetch_add(1, Ordering::Relaxed);
    }
    
    pub fn record_error(&self) {
        self.query_errors.fetch_add(1, Ordering::Relaxed);
    }
    
    pub fn report(&self) -> PoolMetricsReport {
        PoolMetricsReport {
            connections_created: self.connections_created.load(Ordering::Relaxed),
            connections_closed: self.connections_closed.load(Ordering::Relaxed),
            queries_executed: self.queries_executed.load(Ordering::Relaxed),
            query_errors: self.query_errors.load(Ordering::Relaxed),
        }
    }
}

#[derive(Debug, serde::Serialize)]
pub struct PoolMetricsReport {
    pub connections_created: u64,
    pub connections_closed: u64,
    pub queries_executed: u64,
    pub query_errors: u64,
}
```

---

## Health Check Queries

### Database Health Check Endpoint

```rust
// src/handlers/health.rs
use actix_web::{get, web, HttpResponse};
use sqlx::PgPool;
use serde_json::json;
use std::time::Instant;

#[get("/health")]
pub async fn health_check(pool: web::Data<PgPool>) -> HttpResponse {
    let start = Instant::now();
    
    // ตรวจสอบ database connection
    let db_result = sqlx::query!("SELECT 1 as health_check")
        .fetch_one(pool.as_ref())
        .await;
    
    let db_latency = start.elapsed().as_millis();
    
    match db_result {
        Ok(_) => HttpResponse::Ok().json(json!({
            "status": "healthy",
            "database": {
                "status": "connected",
                "latency_ms": db_latency
            },
            "pool": {
                "size": pool.size(),
                "idle": pool.num_idle(),
                "active": pool.size() - pool.num_idle()
            }
        })),
        Err(e) => {
            log::error!("Database health check failed: {}", e);
            HttpResponse::ServiceUnavailable().json(json!({
                "status": "unhealthy",
                "database": {
                    "status": "disconnected",
                    "error": e.to_string()
                }
            }))
        }
    }
}

// Detailed health check
#[get("/health/detailed")]
pub async fn detailed_health_check(pool: web::Data<PgPool>) -> HttpResponse {
    let mut checks = Vec::new();
    
    // Check 1: Basic connectivity
    let connectivity = check_basic_connectivity(&pool).await;
    checks.push(("connectivity", connectivity));
    
    // Check 2: Query performance
    let performance = check_query_performance(&pool).await;
    checks.push(("performance", performance));
    
    // Check 3: Pool utilization
    let pool_check = check_pool_utilization(&pool).await;
    checks.push(("pool", pool_check));
    
    let all_healthy = checks.iter().all(|(_, ok)| *ok);
    
    let status_code = if all_healthy { 200 } else { 503 };
    
    HttpResponse::build(actix_web::http::StatusCode::from_u16(status_code).unwrap())
        .json(json!({
            "status": if all_healthy { "healthy" } else { "degraded" },
            "checks": checks.iter().map(|(name, ok)| {
                json!({
                    "name": name,
                    "status": if *ok { "pass" } else { "fail" }
                })
            }).collect::<Vec<_>>()
        }))
}

async fn check_basic_connectivity(pool: &PgPool) -> bool {
    sqlx::query!("SELECT 1").fetch_one(pool).await.is_ok()
}

async fn check_query_performance(pool: &PgPool) -> bool {
    let start = Instant::now();
    let result = sqlx::query!("SELECT COUNT(*) FROM pg_stat_activity")
        .fetch_one(pool)
        .await;
    
    // ถ้า query ใช้เวลาเกิน 1 วินาที ถือว่า degraded
    result.is_ok() && start.elapsed().as_millis() < 1000
}

async fn check_pool_utilization(pool: &PgPool) -> bool {
    let size = pool.size();
    let idle = pool.num_idle();
    
    // ถ้า pool ใช้งานเกิน 90% → warn
    if size > 0 {
        let utilization = (size - idle) as f32 / size as f32;
        utilization < 0.9
    } else {
        true
    }
}
```

---

## Pool Metrics

### การติดตาม Pool Statistics

```rust
// src/db/metrics.rs
use sqlx::PgPool;
use serde::Serialize;

#[derive(Debug, Serialize)]
pub struct PoolStats {
    pub size: u32,
    pub idle: u32,
    pub active: u32,
    pub utilization_percent: f32,
}

pub fn get_pool_stats(pool: &PgPool) -> PoolStats {
    let size = pool.size();
    let idle = pool.num_idle();
    let active = size - idle;
    let utilization = if size > 0 {
        (active as f32 / size as f32) * 100.0
    } else {
        0.0
    };
    
    PoolStats {
        size,
        idle,
        active,
        utilization_percent: utilization,
    }
}

// Periodic metrics logging
pub async fn start_metrics_reporter(pool: PgPool) {
    let mut interval = tokio::time::interval(
        std::time::Duration::from_secs(60)  // รายงานทุก 1 นาที
    );
    
    loop {
        interval.tick().await;
        
        let stats = get_pool_stats(&pool);
        
        log::info!(
            "Pool stats - size: {}, idle: {}, active: {}, utilization: {:.1}%",
            stats.size,
            stats.idle,
            stats.active,
            stats.utilization_percent
        );
        
        // ส่ง metrics ไปยัง monitoring system (เช่น Prometheus)
        if stats.utilization_percent > 80.0 {
            log::warn!(
                "High pool utilization: {:.1}% - consider increasing max_connections",
                stats.utilization_percent
            );
        }
    }
}
```

### Query Time Tracking

```rust
// src/db/tracing.rs
use sqlx::{PgPool, Executor};
use std::time::Instant;

pub struct TimedQuery<'a> {
    pool: &'a PgPool,
    query_name: &'static str,
}

impl<'a> TimedQuery<'a> {
    pub fn new(pool: &'a PgPool, name: &'static str) -> Self {
        Self { pool, query_name: name }
    }
    
    pub async fn execute(&self, sql: &str) -> Result<(), sqlx::Error> {
        let start = Instant::now();
        
        let result = sqlx::query(sql).execute(self.pool).await;
        
        let elapsed = start.elapsed().as_millis();
        
        if elapsed > 100 {
            log::warn!(
                "Slow query '{}': {}ms",
                self.query_name,
                elapsed
            );
        } else {
            log::debug!(
                "Query '{}' completed in {}ms",
                self.query_name,
                elapsed
            );
        }
        
        result.map(|_| ())
    }
}
```

---

## Multiple Database Pools

### หลาย Database สำหรับ Read/Write Splitting

```rust
// src/db/multi_pool.rs
use sqlx::PgPool;
use std::sync::Arc;

#[derive(Clone)]
pub struct DatabasePools {
    // Primary database สำหรับ writes
    pub primary: PgPool,
    // Replica databases สำหรับ reads (load balanced)
    pub replicas: Vec<PgPool>,
    // Counter สำหรับ round-robin
    counter: Arc<std::sync::atomic::AtomicUsize>,
}

impl DatabasePools {
    pub async fn new(
        primary_url: &str,
        replica_urls: &[&str],
    ) -> Result<Self, sqlx::Error> {
        let primary = create_write_pool(primary_url).await?;
        
        let mut replicas = Vec::new();
        for url in replica_urls {
            let replica = create_read_pool(url).await?;
            replicas.push(replica);
        }
        
        Ok(Self {
            primary,
            replicas,
            counter: Arc::new(std::sync::atomic::AtomicUsize::new(0)),
        })
    }
    
    // Get read pool (round-robin load balancing)
    pub fn read_pool(&self) -> &PgPool {
        if self.replicas.is_empty() {
            return &self.primary;
        }
        
        let index = self.counter.fetch_add(1, std::sync::atomic::Ordering::Relaxed)
            % self.replicas.len();
        
        &self.replicas[index]
    }
    
    // Get write pool (always primary)
    pub fn write_pool(&self) -> &PgPool {
        &self.primary
    }
}

async fn create_write_pool(url: &str) -> Result<PgPool, sqlx::Error> {
    sqlx::postgres::PgPoolOptions::new()
        .max_connections(20)
        .min_connections(5)
        .connect(url)
        .await
}

async fn create_read_pool(url: &str) -> Result<PgPool, sqlx::Error> {
    sqlx::postgres::PgPoolOptions::new()
        .max_connections(30)  // Reads มักมากกว่า writes
        .min_connections(5)
        .connect(url)
        .await
}
```

### Multiple Database สำหรับ Multi-tenancy

```rust
// src/db/tenant_pools.rs
use std::collections::HashMap;
use std::sync::{Arc, RwLock};
use sqlx::PgPool;

#[derive(Clone)]
pub struct TenantDatabasePools {
    pools: Arc<RwLock<HashMap<String, PgPool>>>,
}

impl TenantDatabasePools {
    pub fn new() -> Self {
        Self {
            pools: Arc::new(RwLock::new(HashMap::new())),
        }
    }
    
    pub async fn get_or_create(&self, tenant_id: &str, db_url: &str) -> Result<PgPool, sqlx::Error> {
        // ตรวจสอบ pool ที่มีอยู่แล้ว
        {
            let pools = self.pools.read().unwrap();
            if let Some(pool) = pools.get(tenant_id) {
                return Ok(pool.clone());
            }
        }
        
        // สร้าง pool ใหม่สำหรับ tenant
        let pool = sqlx::postgres::PgPoolOptions::new()
            .max_connections(10)
            .min_connections(2)
            .connect(db_url)
            .await?;
        
        let mut pools = self.pools.write().unwrap();
        pools.insert(tenant_id.to_string(), pool.clone());
        
        log::info!("Created new pool for tenant: {}", tenant_id);
        
        Ok(pool)
    }
    
    pub fn remove(&self, tenant_id: &str) {
        let mut pools = self.pools.write().unwrap();
        if let Some(pool) = pools.remove(tenant_id) {
            // Pool will be closed when dropped
            tokio::spawn(async move {
                pool.close().await;
            });
        }
    }
}
```

---

## Environment Configuration

### Complete Database Configuration

```rust
// src/config/mod.rs
use serde::Deserialize;
use std::env;

#[derive(Debug, Clone)]
pub struct AppConfig {
    pub database: DatabaseConfig,
    pub server: ServerConfig,
    pub environment: String,
}

#[derive(Debug, Clone)]
pub struct DatabaseConfig {
    pub primary_url: String,
    pub replica_urls: Vec<String>,
    pub pool: PoolConfig,
}

#[derive(Debug, Clone)]
pub struct PoolConfig {
    pub max_connections: u32,
    pub min_connections: u32,
    pub acquire_timeout_secs: u64,
    pub idle_timeout_secs: u64,
    pub max_lifetime_secs: u64,
    pub test_before_acquire: bool,
}

#[derive(Debug, Clone)]
pub struct ServerConfig {
    pub host: String,
    pub port: u16,
    pub workers: usize,
}

impl AppConfig {
    pub fn from_env() -> Self {
        let environment = env::var("APP_ENV")
            .unwrap_or_else(|_| "development".to_string());
        
        let pool_config = match environment.as_str() {
            "production" => PoolConfig {
                max_connections: 25,
                min_connections: 5,
                acquire_timeout_secs: 10,
                idle_timeout_secs: 600,
                max_lifetime_secs: 1800,
                test_before_acquire: true,
            },
            "staging" => PoolConfig {
                max_connections: 10,
                min_connections: 2,
                acquire_timeout_secs: 20,
                idle_timeout_secs: 300,
                max_lifetime_secs: 900,
                test_before_acquire: true,
            },
            _ => PoolConfig {  // development, test
                max_connections: 5,
                min_connections: 1,
                acquire_timeout_secs: 30,
                idle_timeout_secs: 60,
                max_lifetime_secs: 300,
                test_before_acquire: false,
            },
        };
        
        // Override จาก environment variables
        let pool_config = PoolConfig {
            max_connections: env::var("DB_POOL_MAX")
                .ok()
                .and_then(|v| v.parse().ok())
                .unwrap_or(pool_config.max_connections),
            min_connections: env::var("DB_POOL_MIN")
                .ok()
                .and_then(|v| v.parse().ok())
                .unwrap_or(pool_config.min_connections),
            ..pool_config
        };
        
        // Replica URLs (comma-separated)
        let replica_urls: Vec<String> = env::var("DATABASE_REPLICA_URLS")
            .unwrap_or_default()
            .split(',')
            .filter(|s| !s.is_empty())
            .map(|s| s.trim().to_string())
            .collect();
        
        AppConfig {
            database: DatabaseConfig {
                primary_url: env::var("DATABASE_URL")
                    .expect("DATABASE_URL must be set"),
                replica_urls,
                pool: pool_config,
            },
            server: ServerConfig {
                host: env::var("SERVER_HOST")
                    .unwrap_or_else(|_| "0.0.0.0".to_string()),
                port: env::var("SERVER_PORT")
                    .unwrap_or_else(|_| "8080".to_string())
                    .parse()
                    .unwrap_or(8080),
                workers: env::var("SERVER_WORKERS")
                    .ok()
                    .and_then(|v| v.parse().ok())
                    .unwrap_or_else(num_cpus::get),
            },
            environment,
        }
    }
}

// สร้าง pool จาก config
pub async fn create_pool_from_config(config: &DatabaseConfig) -> Result<PgPool, sqlx::Error> {
    use sqlx::postgres::PgPoolOptions;
    use std::time::Duration;
    
    PgPoolOptions::new()
        .max_connections(config.pool.max_connections)
        .min_connections(config.pool.min_connections)
        .acquire_timeout(Duration::from_secs(config.pool.acquire_timeout_secs))
        .idle_timeout(Duration::from_secs(config.pool.idle_timeout_secs))
        .max_lifetime(Duration::from_secs(config.pool.max_lifetime_secs))
        .test_before_acquire(config.pool.test_before_acquire)
        .connect(&config.primary_url)
        .await
}
```

### Main.rs ที่ใช้ Config ครบชุด

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware};
use sqlx::PgPool;

mod config;
mod db;
mod handlers;
mod models;
mod repositories;
mod errors;

use config::AppConfig;
use db::create_pool_from_config;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // Load environment
    dotenv::dotenv().ok();
    
    // Init logging
    env_logger::Builder::from_env(
        env_logger::Env::default().default_filter_or("info")
    ).init();
    
    // Load config
    let config = AppConfig::from_env();
    log::info!("Starting server in {} environment", config.environment);
    
    // Create database pool
    let pool = create_pool_from_config(&config.database)
        .await
        .expect("Failed to create database pool");
    
    log::info!(
        "Database pool created: max={}, min={}",
        config.database.pool.max_connections,
        config.database.pool.min_connections
    );
    
    // Run migrations
    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .expect("Failed to run migrations");
    
    log::info!("Migrations completed");
    
    // Start pool metrics reporter
    let pool_clone = pool.clone();
    tokio::spawn(async move {
        db::start_metrics_reporter(pool_clone).await;
    });
    
    let pool_data = web::Data::new(pool);
    let server_config = config.server.clone();
    
    log::info!(
        "Starting HTTP server on {}:{}",
        server_config.host,
        server_config.port
    );
    
    HttpServer::new(move || {
        App::new()
            .app_data(pool_data.clone())
            .wrap(middleware::Logger::default())
            .service(handlers::health::health_check)
            .service(handlers::health::detailed_health_check)
            // เพิ่ม routes อื่นๆ
    })
    .workers(server_config.workers)
    .bind(format!("{}:{}", server_config.host, server_config.port))?
    .run()
    .await
}
```

---

## Navigation

| ก่อนหน้า | หน้าหลัก | ถัดไป |
|---------|---------|------|
| [Part 033: CRUD Operations](../part_033/README.md) | [README หลัก](../../README.md) | [Part 035: Database Transactions](../part_035/README.md) |
