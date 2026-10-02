# Part 036: Redis Integration

## สารบัญ
- [Redis Setup](#redis-setup)
- [Connection Pool ด้วย deadpool-redis](#connection-pool-ด้วย-deadpool-redis)
- [GET/SET/DEL Operations](#getsetdel-operations)
- [EXPIRE และ TTL](#expire-และ-ttl)
- [Cache-Aside Pattern](#cache-aside-pattern)
- [Redis Session Store](#redis-session-store)
- [Pub/Sub Basics](#pubsub-basics)
- [Rate Limiting ด้วย Redis](#rate-limiting-ด้วย-redis)
- [Practical: Caching API Responses](#practical-caching-api-responses)

---

## Redis Setup

### ติดตั้ง Dependencies

```toml
# Cargo.toml
[dependencies]
# Redis client
redis = { version = "0.24", features = [
    "tokio-comp",      # Async support ด้วย tokio
    "connection-manager",
    "json",
] }

# Connection Pool สำหรับ Redis
deadpool-redis = "0.14"

# สำหรับ session management
actix-session = { version = "0.9", features = ["redis-session-rustls"] }

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

### .env Configuration

```bash
# .env
REDIS_URL=redis://localhost:6379

# หรือ พร้อม password
REDIS_URL=redis://:password@localhost:6379

# หรือ TLS
REDIS_URL=rediss://localhost:6380

# Pool settings
REDIS_POOL_SIZE=10
REDIS_TIMEOUT_MS=5000
```

### เริ่มต้น Redis ด้วย Docker

```bash
# รัน Redis
docker run -d \
  --name redis \
  -p 6379:6379 \
  redis:7-alpine

# รัน Redis พร้อม password
docker run -d \
  --name redis \
  -p 6379:6379 \
  redis:7-alpine \
  redis-server --requirepass mypassword

# ทดสอบ
redis-cli ping
# PONG
```

---

## Connection Pool ด้วย deadpool-redis

### การตั้งค่า Pool

```rust
// src/cache/mod.rs
use deadpool_redis::{Config, Pool, Runtime, CreatePoolError};
use std::env;

pub type RedisPool = Pool;

pub fn create_redis_pool() -> Result<RedisPool, CreatePoolError> {
    let redis_url = env::var("REDIS_URL")
        .unwrap_or_else(|_| "redis://localhost:6379".to_string());
    
    let pool_size: usize = env::var("REDIS_POOL_SIZE")
        .unwrap_or_else(|_| "10".to_string())
        .parse()
        .unwrap_or(10);
    
    let cfg = Config::from_url(&redis_url);
    
    cfg.builder()?
        .max_size(pool_size)
        .runtime(Runtime::Tokio1)
        .build()
        .map_err(Into::into)
}

// ตรวจสอบ connection
pub async fn check_redis_connection(pool: &RedisPool) -> Result<(), anyhow::Error> {
    let mut conn = pool.get().await
        .map_err(|e| anyhow::anyhow!("Failed to get Redis connection: {}", e))?;
    
    let pong: String = redis::cmd("PING")
        .query_async(&mut conn)
        .await
        .map_err(|e| anyhow::anyhow!("Redis ping failed: {}", e))?;
    
    if pong != "PONG" {
        return Err(anyhow::anyhow!("Unexpected response from Redis: {}", pong));
    }
    
    Ok(())
}
```

### Redis Client แบบ simple

```rust
// src/cache/client.rs
use redis::AsyncCommands;
use deadpool_redis::{Pool, Connection};

pub struct RedisClient {
    pool: Pool,
}

impl RedisClient {
    pub fn new(pool: Pool) -> Self {
        Self { pool }
    }
    
    async fn get_conn(&self) -> Result<Connection, AppError> {
        self.pool.get().await.map_err(|e| {
            log::error!("Failed to get Redis connection: {}", e);
            AppError::Internal(format!("Redis connection failed: {}", e))
        })
    }
}
```

---

## GET/SET/DEL Operations

### Basic Operations

```rust
// src/cache/operations.rs
use redis::AsyncCommands;
use serde::{de::DeserializeOwned, Serialize};

impl RedisClient {
    // SET - เก็บ string value
    pub async fn set_string(&self, key: &str, value: &str) -> Result<(), AppError> {
        let mut conn = self.get_conn().await?;
        conn.set(key, value).await.map_err(|e| {
            AppError::Internal(format!("Redis SET error: {}", e))
        })
    }
    
    // GET - ดึง string value
    pub async fn get_string(&self, key: &str) -> Result<Option<String>, AppError> {
        let mut conn = self.get_conn().await?;
        conn.get(key).await.map_err(|e| {
            AppError::Internal(format!("Redis GET error: {}", e))
        })
    }
    
    // SET พร้อม TTL (seconds)
    pub async fn set_with_ttl(
        &self,
        key: &str,
        value: &str,
        ttl_seconds: u64
    ) -> Result<(), AppError> {
        let mut conn = self.get_conn().await?;
        conn.set_ex(key, value, ttl_seconds).await.map_err(|e| {
            AppError::Internal(format!("Redis SETEX error: {}", e))
        })
    }
    
    // SET ถ้ายังไม่มี key (SET NX)
    pub async fn set_nx(
        &self,
        key: &str,
        value: &str
    ) -> Result<bool, AppError> {
        let mut conn = self.get_conn().await?;
        let result: bool = conn.set_nx(key, value).await.map_err(|e| {
            AppError::Internal(format!("Redis SETNX error: {}", e))
        })?;
        Ok(result)
    }
    
    // DEL - ลบ key
    pub async fn delete(&self, key: &str) -> Result<bool, AppError> {
        let mut conn = self.get_conn().await?;
        let deleted: i64 = conn.del(key).await.map_err(|e| {
            AppError::Internal(format!("Redis DEL error: {}", e))
        })?;
        Ok(deleted > 0)
    }
    
    // DEL หลาย keys
    pub async fn delete_many(&self, keys: &[&str]) -> Result<i64, AppError> {
        let mut conn = self.get_conn().await?;
        conn.del(keys).await.map_err(|e| {
            AppError::Internal(format!("Redis DEL error: {}", e))
        })
    }
    
    // EXISTS - ตรวจสอบว่า key มีอยู่
    pub async fn exists(&self, key: &str) -> Result<bool, AppError> {
        let mut conn = self.get_conn().await?;
        conn.exists(key).await.map_err(|e| {
            AppError::Internal(format!("Redis EXISTS error: {}", e))
        })
    }
    
    // INCR - increment number
    pub async fn increment(&self, key: &str) -> Result<i64, AppError> {
        let mut conn = self.get_conn().await?;
        conn.incr(key, 1i64).await.map_err(|e| {
            AppError::Internal(format!("Redis INCR error: {}", e))
        })
    }
    
    // INCRBY - increment by amount
    pub async fn increment_by(&self, key: &str, amount: i64) -> Result<i64, AppError> {
        let mut conn = self.get_conn().await?;
        conn.incr(key, amount).await.map_err(|e| {
            AppError::Internal(format!("Redis INCRBY error: {}", e))
        })
    }
    
    // JSON operations (serialize/deserialize)
    pub async fn set_json<T: Serialize>(
        &self,
        key: &str,
        value: &T,
        ttl_seconds: Option<u64>
    ) -> Result<(), AppError> {
        let json = serde_json::to_string(value)
            .map_err(|e| AppError::Internal(format!("JSON serialize error: {}", e)))?;
        
        let mut conn = self.get_conn().await?;
        
        if let Some(ttl) = ttl_seconds {
            conn.set_ex(key, &json, ttl).await
        } else {
            conn.set(key, &json).await
        }
        .map_err(|e| AppError::Internal(format!("Redis SET error: {}", e)))
    }
    
    pub async fn get_json<T: DeserializeOwned>(
        &self,
        key: &str
    ) -> Result<Option<T>, AppError> {
        let mut conn = self.get_conn().await?;
        
        let json: Option<String> = conn.get(key).await.map_err(|e| {
            AppError::Internal(format!("Redis GET error: {}", e))
        })?;
        
        match json {
            None => Ok(None),
            Some(s) => {
                let value: T = serde_json::from_str(&s)
                    .map_err(|e| AppError::Internal(format!("JSON deserialize error: {}", e)))?;
                Ok(Some(value))
            }
        }
    }
    
    // KEYS pattern matching
    pub async fn keys(&self, pattern: &str) -> Result<Vec<String>, AppError> {
        let mut conn = self.get_conn().await?;
        conn.keys(pattern).await.map_err(|e| {
            AppError::Internal(format!("Redis KEYS error: {}", e))
        })
    }
    
    // SCAN (safer than KEYS for production)
    pub async fn scan_keys(&self, pattern: &str) -> Result<Vec<String>, AppError> {
        let mut conn = self.get_conn().await?;
        
        let mut keys = Vec::new();
        let mut cursor = 0u64;
        
        loop {
            let (new_cursor, batch): (u64, Vec<String>) = redis::cmd("SCAN")
                .arg(cursor)
                .arg("MATCH")
                .arg(pattern)
                .arg("COUNT")
                .arg(100)
                .query_async(&mut conn)
                .await
                .map_err(|e| AppError::Internal(format!("Redis SCAN error: {}", e)))?;
            
            keys.extend(batch);
            cursor = new_cursor;
            
            if cursor == 0 {
                break;
            }
        }
        
        Ok(keys)
    }
    
    // Hash operations
    pub async fn hset(&self, key: &str, field: &str, value: &str) -> Result<(), AppError> {
        let mut conn = self.get_conn().await?;
        conn.hset(key, field, value).await.map_err(|e| {
            AppError::Internal(format!("Redis HSET error: {}", e))
        })
    }
    
    pub async fn hget(&self, key: &str, field: &str) -> Result<Option<String>, AppError> {
        let mut conn = self.get_conn().await?;
        conn.hget(key, field).await.map_err(|e| {
            AppError::Internal(format!("Redis HGET error: {}", e))
        })
    }
    
    pub async fn hgetall(&self, key: &str) -> Result<std::collections::HashMap<String, String>, AppError> {
        let mut conn = self.get_conn().await?;
        conn.hgetall(key).await.map_err(|e| {
            AppError::Internal(format!("Redis HGETALL error: {}", e))
        })
    }
    
    // List operations
    pub async fn lpush(&self, key: &str, value: &str) -> Result<i64, AppError> {
        let mut conn = self.get_conn().await?;
        conn.lpush(key, value).await.map_err(|e| {
            AppError::Internal(format!("Redis LPUSH error: {}", e))
        })
    }
    
    pub async fn rpop(&self, key: &str) -> Result<Option<String>, AppError> {
        let mut conn = self.get_conn().await?;
        conn.rpop(key, None).await.map_err(|e| {
            AppError::Internal(format!("Redis RPOP error: {}", e))
        })
    }
}
```

---

## EXPIRE และ TTL

### การจัดการ Expiration

```rust
// src/cache/ttl.rs

impl RedisClient {
    // ตั้ง TTL สำหรับ key ที่มีอยู่แล้ว
    pub async fn expire(&self, key: &str, ttl_seconds: u64) -> Result<bool, AppError> {
        let mut conn = self.get_conn().await?;
        conn.expire(key, ttl_seconds as i64).await.map_err(|e| {
            AppError::Internal(format!("Redis EXPIRE error: {}", e))
        })
    }
    
    // ตั้ง TTL เป็น timestamp
    pub async fn expire_at(&self, key: &str, timestamp: i64) -> Result<bool, AppError> {
        let mut conn = self.get_conn().await?;
        conn.expire_at(key, timestamp).await.map_err(|e| {
            AppError::Internal(format!("Redis EXPIREAT error: {}", e))
        })
    }
    
    // ดู TTL ที่เหลือ (seconds)
    // -1 = no expiry, -2 = key doesn't exist
    pub async fn ttl(&self, key: &str) -> Result<i64, AppError> {
        let mut conn = self.get_conn().await?;
        conn.ttl(key).await.map_err(|e| {
            AppError::Internal(format!("Redis TTL error: {}", e))
        })
    }
    
    // ดู TTL เป็น milliseconds
    pub async fn pttl(&self, key: &str) -> Result<i64, AppError> {
        let mut conn = self.get_conn().await?;
        conn.pttl(key).await.map_err(|e| {
            AppError::Internal(format!("Redis PTTL error: {}", e))
        })
    }
    
    // ลบ TTL (ทำให้ key ไม่หมดอายุ)
    pub async fn persist(&self, key: &str) -> Result<bool, AppError> {
        let mut conn = self.get_conn().await?;
        conn.persist(key).await.map_err(|e| {
            AppError::Internal(format!("Redis PERSIST error: {}", e))
        })
    }
}

// TTL Constants
pub mod ttl {
    pub const ONE_MINUTE: u64 = 60;
    pub const FIVE_MINUTES: u64 = 300;
    pub const FIFTEEN_MINUTES: u64 = 900;
    pub const ONE_HOUR: u64 = 3600;
    pub const SIX_HOURS: u64 = 21600;
    pub const ONE_DAY: u64 = 86400;
    pub const ONE_WEEK: u64 = 604800;
    pub const ONE_MONTH: u64 = 2592000;
}
```

---

## Cache-Aside Pattern

### Cache-Aside Implementation

```rust
// src/cache/cache_aside.rs
use std::time::Instant;

pub struct CacheAside {
    redis: RedisClient,
}

impl CacheAside {
    pub fn new(redis: RedisClient) -> Self {
        Self { redis }
    }
    
    // Cache-Aside: อ่านจาก cache ก่อน ถ้าไม่มีค่อยอ่าน DB
    pub async fn get_or_fetch<T, F, Fut>(
        &self,
        cache_key: &str,
        ttl_seconds: u64,
        fetch_fn: F,
    ) -> Result<T, AppError>
    where
        T: Serialize + DeserializeOwned,
        F: FnOnce() -> Fut,
        Fut: std::future::Future<Output = Result<T, AppError>>,
    {
        // 1. ลองดึงจาก cache
        if let Some(cached) = self.redis.get_json::<T>(cache_key).await? {
            log::debug!("Cache HIT: {}", cache_key);
            return Ok(cached);
        }
        
        log::debug!("Cache MISS: {}", cache_key);
        
        // 2. ดึงจาก DB
        let start = Instant::now();
        let data = fetch_fn().await?;
        
        log::debug!(
            "DB fetch for '{}' took {}ms",
            cache_key,
            start.elapsed().as_millis()
        );
        
        // 3. เก็บใน cache
        if let Err(e) = self.redis.set_json(cache_key, &data, Some(ttl_seconds)).await {
            // Log error แต่ไม่ fail - cache error ไม่ควร block request
            log::error!("Failed to cache '{}': {}", cache_key, e);
        }
        
        Ok(data)
    }
    
    // Invalidate cache
    pub async fn invalidate(&self, cache_key: &str) -> Result<(), AppError> {
        self.redis.delete(cache_key).await?;
        log::debug!("Cache invalidated: {}", cache_key);
        Ok(())
    }
    
    // Invalidate ด้วย pattern
    pub async fn invalidate_pattern(&self, pattern: &str) -> Result<usize, AppError> {
        let keys = self.redis.scan_keys(pattern).await?;
        let count = keys.len();
        
        if !keys.is_empty() {
            let key_refs: Vec<&str> = keys.iter().map(|s| s.as_str()).collect();
            self.redis.delete_many(&key_refs).await?;
        }
        
        log::debug!("Invalidated {} keys matching '{}'", count, pattern);
        Ok(count)
    }
}

// Key Naming Convention
pub struct CacheKey;

impl CacheKey {
    pub fn user(id: uuid::Uuid) -> String {
        format!("user:{}", id)
    }
    
    pub fn user_by_email(email: &str) -> String {
        format!("user:email:{}", email)
    }
    
    pub fn post(id: uuid::Uuid) -> String {
        format!("post:{}", id)
    }
    
    pub fn post_by_slug(slug: &str) -> String {
        format!("post:slug:{}", slug)
    }
    
    pub fn posts_list(page: i64, per_page: i64) -> String {
        format!("posts:list:{}:{}", page, per_page)
    }
    
    pub fn post_count() -> String {
        "posts:count".to_string()
    }
    
    pub fn user_posts(user_id: uuid::Uuid, page: i64) -> String {
        format!("user:{}:posts:{}", user_id, page)
    }
}
```

---

## Redis Session Store

### Session Management ด้วย Redis

```rust
// src/session/redis_session.rs
use actix_session::SessionMiddleware;
use actix_session::storage::RedisSessionStore;
use actix_web::cookie::Key;

pub async fn create_session_middleware(
    redis_url: &str
) -> Result<SessionMiddleware<RedisSessionStore>, anyhow::Error> {
    let redis_store = RedisSessionStore::new(redis_url).await?;
    
    // สร้าง secret key สำหรับ cookie signing
    let secret_key = Key::generate();  // ใน production ใช้ key จาก env
    
    let middleware = SessionMiddleware::builder(redis_store, secret_key)
        .session_lifecycle(
            actix_session::config::PersistentSession::default()
                .session_ttl(actix_web::cookie::time::Duration::hours(24))
        )
        .cookie_secure(true)
        .cookie_http_only(true)
        .cookie_same_site(actix_web::cookie::SameSite::Strict)
        .build();
    
    Ok(middleware)
}

// Session operations ใน handler
use actix_session::Session;
use actix_web::{post, get, web, HttpResponse};

#[derive(serde::Serialize, serde::Deserialize)]
pub struct SessionData {
    pub user_id: uuid::Uuid,
    pub email: String,
    pub is_admin: bool,
}

#[post("/auth/login")]
pub async fn login(
    session: Session,
    pool: web::Data<sqlx::PgPool>,
    credentials: web::Json<LoginCredentials>,
) -> HttpResponse {
    // Authenticate user...
    let user_id = uuid::Uuid::new_v4();  // placeholder
    
    // เก็บข้อมูลใน session
    match session.insert("user_id", user_id.to_string()) {
        Ok(_) => {}
        Err(e) => {
            log::error!("Session insert error: {}", e);
            return HttpResponse::InternalServerError().finish();
        }
    }
    
    session.insert("is_admin", false).ok();
    
    HttpResponse::Ok().json(serde_json::json!({
        "message": "Login successful"
    }))
}

#[post("/auth/logout")]
pub async fn logout(session: Session) -> HttpResponse {
    session.purge();
    HttpResponse::Ok().json(serde_json::json!({
        "message": "Logged out"
    }))
}

#[get("/auth/me")]
pub async fn get_current_user(session: Session) -> HttpResponse {
    match session.get::<String>("user_id") {
        Ok(Some(user_id)) => HttpResponse::Ok().json(serde_json::json!({
            "user_id": user_id
        })),
        _ => HttpResponse::Unauthorized().finish()
    }
}

#[derive(serde::Deserialize)]
struct LoginCredentials {
    email: String,
    password: String,
}
```

---

## Pub/Sub Basics

### Redis Pub/Sub

```rust
// src/cache/pubsub.rs
use redis::AsyncCommands;
use std::sync::Arc;
use tokio::sync::broadcast;

pub struct PubSubManager {
    pool: deadpool_redis::Pool,
    // Channel สำหรับส่ง messages ใน app
    sender: broadcast::Sender<PubSubMessage>,
}

#[derive(Debug, Clone)]
pub struct PubSubMessage {
    pub channel: String,
    pub payload: String,
}

impl PubSubManager {
    pub fn new(pool: deadpool_redis::Pool) -> (Self, broadcast::Receiver<PubSubMessage>) {
        let (sender, receiver) = broadcast::channel(1000);
        
        (Self { pool, sender }, receiver)
    }
    
    // Publish message
    pub async fn publish(&self, channel: &str, message: &str) -> Result<(), AppError> {
        let mut conn = self.pool.get().await.map_err(|e| {
            AppError::Internal(format!("Redis connection error: {}", e))
        })?;
        
        conn.publish(channel, message).await.map_err(|e| {
            AppError::Internal(format!("Redis PUBLISH error: {}", e))
        })
    }
    
    // Subscribe และ forward messages
    pub async fn start_subscriber(
        pool: deadpool_redis::Pool,
        channels: Vec<String>,
        sender: broadcast::Sender<PubSubMessage>,
    ) {
        let client = redis::Client::open(
            std::env::var("REDIS_URL").unwrap_or_else(|_| "redis://localhost:6379".to_string())
        ).expect("Invalid Redis URL");
        
        let conn = client.get_async_pubsub().await.expect("Failed to connect");
        let mut pubsub = conn;
        
        for channel in &channels {
            pubsub.subscribe(channel).await.expect("Failed to subscribe");
        }
        
        log::info!("Subscribed to channels: {:?}", channels);
        
        use futures::StreamExt;
        let mut stream = pubsub.on_message();
        
        while let Some(msg) = stream.next().await {
            let channel: String = msg.get_channel().unwrap_or_default();
            let payload: String = msg.get_payload().unwrap_or_default();
            
            log::debug!("Received on {}: {}", channel, payload);
            
            let _ = sender.send(PubSubMessage { channel, payload });
        }
    }
}

// ตัวอย่างการใช้ Pub/Sub สำหรับ real-time notifications
pub async fn example_pubsub_usage() {
    // Publisher
    let publisher_pool = create_redis_pool().expect("Failed to create pool");
    let manager = PubSubManager::new(publisher_pool.clone());
    
    // Publish notification
    manager.publish(
        "notifications",
        &serde_json::to_string(&serde_json::json!({
            "type": "new_comment",
            "post_id": "some-uuid",
            "comment_id": "another-uuid"
        })).unwrap()
    ).await.expect("Failed to publish");
    
    // Subscribe ใน background task
    let (subscriber_manager, mut receiver) = PubSubManager::new(publisher_pool);
    
    tokio::spawn(async move {
        let (tx, _rx) = broadcast::channel::<PubSubMessage>(1000);
        PubSubManager::start_subscriber(
            subscriber_manager.pool,
            vec!["notifications".to_string()],
            tx
        ).await;
    });
    
    // รับ messages
    while let Ok(msg) = receiver.recv().await {
        println!("Received: {} = {}", msg.channel, msg.payload);
    }
}
```

---

## Rate Limiting ด้วย Redis

### Rate Limiter Implementation

```rust
// src/middleware/rate_limit.rs
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpResponse,
};
use deadpool_redis::Pool;
use futures::future::LocalBoxFuture;
use redis::AsyncCommands;
use std::future::{self, Ready};

pub struct RateLimiter {
    redis_pool: Pool,
    max_requests: u64,
    window_seconds: u64,
}

impl RateLimiter {
    pub fn new(redis_pool: Pool, max_requests: u64, window_seconds: u64) -> Self {
        Self {
            redis_pool,
            max_requests,
            window_seconds,
        }
    }
}

// Sliding window rate limiter
pub async fn check_rate_limit(
    redis_pool: &Pool,
    identifier: &str,  // IP address หรือ user ID
    max_requests: u64,
    window_seconds: u64,
) -> Result<RateLimitStatus, AppError> {
    let mut conn = redis_pool.get().await.map_err(|e| {
        AppError::Internal(format!("Redis connection error: {}", e))
    })?;
    
    let key = format!("rate_limit:{}", identifier);
    let now = chrono::Utc::now().timestamp_millis();
    let window_start = now - (window_seconds as i64 * 1000);
    
    // ใช้ Redis MULTI/EXEC สำหรับ atomic operations
    let (count, _): (i64, i64) = redis::pipe()
        .atomic()
        // ลบ requests เก่าออกจาก sorted set
        .cmd("ZREMRANGEBYSCORE").arg(&key).arg("-inf").arg(window_start)
        // นับ requests ใน window
        .cmd("ZCARD").arg(&key)
        .query_async(&mut conn)
        .await
        .map_err(|e| AppError::Internal(format!("Redis pipeline error: {}", e)))?;
    
    if count >= max_requests as i64 {
        // ดู TTL ที่เหลือ
        let oldest_score: Option<f64> = conn.zrange_withscores::<_, Vec<(String, f64)>>(&key, 0, 0)
            .await
            .ok()
            .and_then(|v| v.first().map(|(_, score)| *score));
        
        let retry_after = oldest_score
            .map(|s| ((s as i64 + window_seconds as i64 * 1000 - now) / 1000).max(0))
            .unwrap_or(window_seconds as i64);
        
        return Ok(RateLimitStatus {
            allowed: false,
            remaining: 0,
            reset_after_seconds: retry_after,
            total_requests: count as u64,
        });
    }
    
    // เพิ่ม request ปัจจุบันใน sorted set
    let _: () = redis::pipe()
        .atomic()
        .cmd("ZADD").arg(&key).arg(now).arg(now.to_string())
        .cmd("EXPIRE").arg(&key).arg(window_seconds)
        .query_async(&mut conn)
        .await
        .map_err(|e| AppError::Internal(format!("Redis pipeline error: {}", e)))?;
    
    Ok(RateLimitStatus {
        allowed: true,
        remaining: (max_requests as i64 - count - 1).max(0) as u64,
        reset_after_seconds: window_seconds as i64,
        total_requests: (count + 1) as u64,
    })
}

#[derive(Debug)]
pub struct RateLimitStatus {
    pub allowed: bool,
    pub remaining: u64,
    pub reset_after_seconds: i64,
    pub total_requests: u64,
}

// Fixed window rate limiter (ง่ายกว่า)
pub async fn check_rate_limit_fixed(
    redis_pool: &Pool,
    identifier: &str,
    max_requests: u64,
    window_seconds: u64,
) -> Result<bool, AppError> {
    let mut conn = redis_pool.get().await.map_err(|e| {
        AppError::Internal(format!("Redis error: {}", e))
    })?;
    
    let key = format!("rate_fixed:{}:{}", identifier, 
        chrono::Utc::now().timestamp() / window_seconds as i64);
    
    let count: i64 = conn.incr(&key, 1).await.map_err(|e| {
        AppError::Internal(format!("Redis INCR error: {}", e))
    })?;
    
    if count == 1 {
        // ตั้ง TTL สำหรับ key ใหม่
        let _: () = conn.expire(&key, window_seconds as i64).await.ok().unwrap_or(());
    }
    
    Ok(count <= max_requests as i64)
}

// Actix middleware ที่ใช้ rate limiter
use actix_web::web;

pub async fn rate_limit_middleware(
    req: ServiceRequest,
    redis_pool: &Pool,
) -> Result<ServiceRequest, actix_web::Error> {
    // ดึง IP address
    let ip = req.connection_info()
        .realip_remote_addr()
        .unwrap_or("unknown")
        .to_string();
    
    // Rate limit: 100 requests per minute
    match check_rate_limit(redis_pool, &ip, 100, 60).await {
        Ok(status) if status.allowed => Ok(req),
        Ok(status) => {
            Err(actix_web::error::ErrorTooManyRequests(
                format!("Rate limit exceeded. Try again in {} seconds", status.reset_after_seconds)
            ))
        }
        Err(e) => {
            log::error!("Rate limit check failed: {}", e);
            Ok(req)  // Fail open
        }
    }
}
```

---

## Practical: Caching API Responses

### Post Service ที่ใช้ Cache

```rust
// src/services/post_service.rs
use crate::cache::{RedisClient, CacheKey, ttl};
use crate::repositories::PostRepository;
use serde::{Deserialize, Serialize};

pub struct PostService {
    repo: PostRepository,
    cache: RedisClient,
}

impl PostService {
    pub fn new(repo: PostRepository, cache: RedisClient) -> Self {
        Self { repo, cache }
    }
    
    // อ่าน post ด้วย cache
    pub async fn get_post_by_slug(&self, slug: &str) -> Result<Option<Post>, AppError> {
        let cache_key = CacheKey::post_by_slug(slug);
        
        // ลอง cache ก่อน
        if let Some(cached) = self.cache.get_json::<Post>(&cache_key).await? {
            log::debug!("Cache HIT for post slug: {}", slug);
            return Ok(Some(cached));
        }
        
        // ดึงจาก DB
        match self.repo.find_by_slug(slug).await? {
            None => Ok(None),
            Some(post) => {
                // เก็บใน cache (15 นาที)
                self.cache.set_json(&cache_key, &post, Some(ttl::FIFTEEN_MINUTES))
                    .await
                    .unwrap_or_else(|e| log::error!("Cache set error: {}", e));
                
                // เก็บ ID → slug mapping ด้วย
                let id_key = CacheKey::post(post.id);
                self.cache.set_json(&id_key, &post, Some(ttl::FIFTEEN_MINUTES))
                    .await
                    .unwrap_or_else(|e| log::error!("Cache set error: {}", e));
                
                Ok(Some(post))
            }
        }
    }
    
    // อ่าน post list ด้วย cache
    pub async fn get_posts(
        &self,
        page: i64,
        per_page: i64
    ) -> Result<PaginatedPosts, AppError> {
        let cache_key = CacheKey::posts_list(page, per_page);
        
        // Cache list ไว้ 5 นาที (list เปลี่ยนบ่อยกว่า individual)
        if let Some(cached) = self.cache.get_json::<PaginatedPosts>(&cache_key).await? {
            return Ok(cached);
        }
        
        let filter = PostFilter {
            status: Some("published".to_string()),
            page: Some(page),
            per_page: Some(per_page),
            ..Default::default()
        };
        
        let posts = self.repo.find_all(filter).await?;
        
        // Cache result
        self.cache.set_json(&cache_key, &posts, Some(ttl::FIVE_MINUTES))
            .await
            .unwrap_or_else(|e| log::error!("Cache set error: {}", e));
        
        Ok(posts)
    }
    
    // Update post และ invalidate cache
    pub async fn update_post(
        &self,
        id: uuid::Uuid,
        author_id: uuid::Uuid,
        dto: UpdatePostDto
    ) -> Result<Post, AppError> {
        let post = self.repo.update(id, author_id, dto).await?;
        
        // Invalidate cache
        let id_key = CacheKey::post(id);
        let slug_key = CacheKey::post_by_slug(&post.slug);
        
        self.cache.delete(&id_key).await.ok();
        self.cache.delete(&slug_key).await.ok();
        
        // Invalidate list caches (pattern)
        self.cache.scan_keys("posts:list:*")
            .await
            .unwrap_or_default()
            .iter()
            .for_each(|key| {
                let cache = self.cache.clone();
                let k = key.clone();
                tokio::spawn(async move {
                    cache.delete(&k).await.ok();
                });
            });
        
        Ok(post)
    }
    
    // Warm up cache (เรียกตอน startup)
    pub async fn warm_up_cache(&self) -> Result<(), AppError> {
        log::info!("Warming up post cache...");
        
        // Cache 3 pages แรก
        for page in 1..=3 {
            match self.get_posts(page, 10).await {
                Ok(_) => log::debug!("Cached page {}", page),
                Err(e) => log::warn!("Failed to cache page {}: {}", page, e),
            }
        }
        
        log::info!("Cache warm-up complete");
        Ok(())
    }
    
    // View count ด้วย Redis (batch update DB)
    pub async fn increment_view_count(&self, post_id: uuid::Uuid) -> Result<(), AppError> {
        let key = format!("post:views:{}", post_id);
        let count = self.cache.increment(&key).await?;
        
        // ตั้ง TTL ถ้าเป็น key ใหม่
        if count == 1 {
            self.cache.expire(&key, ttl::ONE_HOUR).await.ok();
        }
        
        // เมื่อถึง threshold → flush ไป DB
        if count % 10 == 0 {
            let repo = self.repo.clone();
            tokio::spawn(async move {
                if let Err(e) = repo.increment_view(post_id).await {
                    log::error!("Failed to update view count in DB: {}", e);
                }
            });
        }
        
        Ok(())
    }
}

// Handler
use actix_web::{get, web, HttpResponse};

#[get("/api/v1/posts/{slug}")]
pub async fn get_post(
    slug: web::Path<String>,
    service: web::Data<PostService>,
) -> HttpResponse {
    match service.get_post_by_slug(&slug).await {
        Ok(Some(post)) => {
            // Increment view count (non-blocking)
            let svc = service.clone();
            let post_id = post.id;
            tokio::spawn(async move {
                svc.increment_view_count(post_id).await.ok();
            });
            
            HttpResponse::Ok().json(post)
        }
        Ok(None) => HttpResponse::NotFound().json(serde_json::json!({
            "error": "Post not found"
        })),
        Err(e) => {
            log::error!("Get post error: {}", e);
            HttpResponse::InternalServerError().json(serde_json::json!({
                "error": "Internal server error"
            }))
        }
    }
}
```

### Main.rs ที่รวม Redis

```rust
// src/main.rs (พร้อม Redis)
use actix_web::{web, App, HttpServer, middleware};

#[tokio::main]
async fn main() -> std::io::Result<()> {
    dotenv::dotenv().ok();
    env_logger::init();
    
    // Database pool
    let pool = create_pool_from_env().await.expect("DB pool failed");
    
    // Redis pool
    let redis_pool = create_redis_pool().expect("Redis pool failed");
    
    // Check Redis connection
    check_redis_connection(&redis_pool)
        .await
        .expect("Redis connection check failed");
    
    log::info!("Connected to Redis");
    
    // Run migrations
    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .expect("Migrations failed");
    
    let pool_data = web::Data::new(pool);
    let redis_data = web::Data::new(redis_pool);
    
    HttpServer::new(move || {
        App::new()
            .app_data(pool_data.clone())
            .app_data(redis_data.clone())
            .wrap(middleware::Logger::default())
            // Routes...
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}

async fn create_pool_from_env() -> Result<sqlx::PgPool, sqlx::Error> {
    let url = std::env::var("DATABASE_URL").expect("DATABASE_URL not set");
    sqlx::postgres::PgPoolOptions::new()
        .max_connections(10)
        .connect(&url)
        .await
}

fn create_redis_pool() -> Result<deadpool_redis::Pool, deadpool_redis::CreatePoolError> {
    let url = std::env::var("REDIS_URL")
        .unwrap_or_else(|_| "redis://localhost:6379".to_string());
    
    deadpool_redis::Config::from_url(&url)
        .builder()?
        .max_size(10)
        .runtime(deadpool_redis::Runtime::Tokio1)
        .build()
        .map_err(Into::into)
}

async fn check_redis_connection(pool: &deadpool_redis::Pool) -> Result<(), Box<dyn std::error::Error>> {
    let mut conn = pool.get().await?;
    let _: String = redis::cmd("PING").query_async(&mut conn).await?;
    Ok(())
}
```

---

## Navigation

| ก่อนหน้า | หน้าหลัก | ถัดไป |
|---------|---------|------|
| [Part 035: Database Transactions](../part_035/README.md) | [README หลัก](../../README.md) | [Part 037: Advanced SQLx Queries](../part_037/README.md) |
