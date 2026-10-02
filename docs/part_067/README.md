# Part 067: Caching Strategies in Rust

## ภาพรวม

Caching เป็น technique สำคัญในการเพิ่มประสิทธิภาพของ web application โดยลดการเรียกข้อมูลจาก database บ่อย ๆ บทนี้จะครอบคลุม caching patterns ต่าง ๆ พร้อม Redis integration

## Caching Patterns

```
1. Cache-aside (Lazy Loading)  - โหลดใส่ cache เมื่อ miss
2. Write-through               - เขียน cache และ DB พร้อมกัน
3. Write-behind (Write-back)   - เขียน cache ก่อน แล้ว sync DB ทีหลัง
4. Read-through                - Cache อยู่หน้า DB โดยสมบูรณ์
```

## Cargo.toml

```toml
[package]
name = "caching-example"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4", "serde"] }
redis = { version = "0.23", features = ["tokio-comp", "connection-manager"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "1"
async-trait = "0.1"
tracing = "0.1"
tracing-subscriber = "0.3"
```

## Cache Interface (Trait)

```rust
// src/cache/mod.rs
use async_trait::async_trait;
use std::time::Duration;

#[derive(Debug, thiserror::Error)]
pub enum CacheError {
    #[error("Connection error: {0}")]
    ConnectionError(String),
    #[error("Serialization error: {0}")]
    SerializationError(String),
    #[error("Key not found: {0}")]
    NotFound(String),
}

#[async_trait]
pub trait Cache: Send + Sync {
    async fn get<T: serde::de::DeserializeOwned>(&self, key: &str) -> Result<Option<T>, CacheError>;
    async fn set<T: serde::Serialize + Send + Sync>(
        &self,
        key: &str,
        value: &T,
        ttl: Option<Duration>,
    ) -> Result<(), CacheError>;
    async fn delete(&self, key: &str) -> Result<(), CacheError>;
    async fn delete_pattern(&self, pattern: &str) -> Result<u64, CacheError>;
    async fn exists(&self, key: &str) -> Result<bool, CacheError>;
    async fn increment(&self, key: &str) -> Result<i64, CacheError>;
    async fn expire(&self, key: &str, ttl: Duration) -> Result<(), CacheError>;
}

// Redis implementation
pub struct RedisCache {
    client: redis::Client,
    prefix: String,
}

impl RedisCache {
    pub fn new(redis_url: &str, prefix: &str) -> Result<Self, CacheError> {
        let client = redis::Client::open(redis_url)
            .map_err(|e| CacheError::ConnectionError(e.to_string()))?;
        
        Ok(RedisCache {
            client,
            prefix: prefix.to_string(),
        })
    }
    
    fn make_key(&self, key: &str) -> String {
        format!("{}:{}", self.prefix, key)
    }
    
    async fn get_connection(&self) -> Result<redis::aio::ConnectionManager, CacheError> {
        redis::aio::ConnectionManager::new(self.client.clone())
            .await
            .map_err(|e| CacheError::ConnectionError(e.to_string()))
    }
}

#[async_trait]
impl Cache for RedisCache {
    async fn get<T: serde::de::DeserializeOwned>(&self, key: &str) -> Result<Option<T>, CacheError> {
        let full_key = self.make_key(key);
        let mut conn = self.get_connection().await?;
        
        let result: Option<String> = redis::cmd("GET")
            .arg(&full_key)
            .query_async(&mut conn)
            .await
            .map_err(|e| CacheError::ConnectionError(e.to_string()))?;
        
        match result {
            Some(json) => {
                let value: T = serde_json::from_str(&json)
                    .map_err(|e| CacheError::SerializationError(e.to_string()))?;
                Ok(Some(value))
            }
            None => Ok(None),
        }
    }
    
    async fn set<T: serde::Serialize + Send + Sync>(
        &self,
        key: &str,
        value: &T,
        ttl: Option<Duration>,
    ) -> Result<(), CacheError> {
        let full_key = self.make_key(key);
        let json = serde_json::to_string(value)
            .map_err(|e| CacheError::SerializationError(e.to_string()))?;
        
        let mut conn = self.get_connection().await?;
        
        match ttl {
            Some(duration) => {
                redis::cmd("SETEX")
                    .arg(&full_key)
                    .arg(duration.as_secs())
                    .arg(&json)
                    .query_async::<_, ()>(&mut conn)
                    .await
            }
            None => {
                redis::cmd("SET")
                    .arg(&full_key)
                    .arg(&json)
                    .query_async::<_, ()>(&mut conn)
                    .await
            }
        }
        .map_err(|e| CacheError::ConnectionError(e.to_string()))
    }
    
    async fn delete(&self, key: &str) -> Result<(), CacheError> {
        let full_key = self.make_key(key);
        let mut conn = self.get_connection().await?;
        
        redis::cmd("DEL")
            .arg(&full_key)
            .query_async::<_, ()>(&mut conn)
            .await
            .map_err(|e| CacheError::ConnectionError(e.to_string()))
    }
    
    async fn delete_pattern(&self, pattern: &str) -> Result<u64, CacheError> {
        let full_pattern = self.make_key(pattern);
        let mut conn = self.get_connection().await?;
        
        // ใช้ SCAN เพื่อหา keys (ไม่ block Redis)
        let keys: Vec<String> = redis::cmd("KEYS")
            .arg(&full_pattern)
            .query_async(&mut conn)
            .await
            .map_err(|e| CacheError::ConnectionError(e.to_string()))?;
        
        if keys.is_empty() {
            return Ok(0);
        }
        
        let count = keys.len() as u64;
        let mut cmd = redis::cmd("DEL");
        for key in &keys {
            cmd.arg(key);
        }
        
        cmd.query_async::<_, ()>(&mut conn)
            .await
            .map_err(|e| CacheError::ConnectionError(e.to_string()))?;
        
        Ok(count)
    }
    
    async fn exists(&self, key: &str) -> Result<bool, CacheError> {
        let full_key = self.make_key(key);
        let mut conn = self.get_connection().await?;
        
        let count: i32 = redis::cmd("EXISTS")
            .arg(&full_key)
            .query_async(&mut conn)
            .await
            .map_err(|e| CacheError::ConnectionError(e.to_string()))?;
        
        Ok(count > 0)
    }
    
    async fn increment(&self, key: &str) -> Result<i64, CacheError> {
        let full_key = self.make_key(key);
        let mut conn = self.get_connection().await?;
        
        redis::cmd("INCR")
            .arg(&full_key)
            .query_async(&mut conn)
            .await
            .map_err(|e| CacheError::ConnectionError(e.to_string()))
    }
    
    async fn expire(&self, key: &str, ttl: Duration) -> Result<(), CacheError> {
        let full_key = self.make_key(key);
        let mut conn = self.get_connection().await?;
        
        redis::cmd("EXPIRE")
            .arg(&full_key)
            .arg(ttl.as_secs())
            .query_async::<_, ()>(&mut conn)
            .await
            .map_err(|e| CacheError::ConnectionError(e.to_string()))
    }
}
```

## Pattern 1: Cache-aside

```rust
// src/patterns/cache_aside.rs
use std::sync::Arc;
use std::time::Duration;
use uuid::Uuid;
use serde::{Deserialize, Serialize};

use crate::cache::Cache;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Product {
    pub id: Uuid,
    pub name: String,
    pub description: String,
    pub price: f64,
    pub category: String,
    pub in_stock: bool,
}

pub struct ProductRepository {
    // simulate DB
    products: std::collections::HashMap<Uuid, Product>,
}

impl ProductRepository {
    pub fn new() -> Self {
        let mut products = std::collections::HashMap::new();
        
        for i in 0..100 {
            let id = Uuid::new_v4();
            products.insert(id, Product {
                id,
                name: format!("Product {}", i),
                description: format!("Description for product {}", i),
                price: (i + 1) as f64 * 99.0,
                category: if i % 2 == 0 { "Electronics" } else { "Clothing" }.to_string(),
                in_stock: i % 3 != 0,
            });
        }
        
        ProductRepository { products }
    }
    
    pub async fn find_by_id(&self, id: Uuid) -> Option<Product> {
        // Simulate DB latency
        tokio::time::sleep(Duration::from_millis(50)).await;
        self.products.get(&id).cloned()
    }
    
    pub async fn find_by_category(&self, category: &str) -> Vec<Product> {
        tokio::time::sleep(Duration::from_millis(100)).await;
        self.products.values()
            .filter(|p| p.category == category)
            .cloned()
            .collect()
    }
}

/// Cache-aside Service
pub struct ProductService {
    repo: Arc<ProductRepository>,
    cache: Arc<dyn Cache>,
    default_ttl: Duration,
}

impl ProductService {
    pub fn new(
        repo: Arc<ProductRepository>,
        cache: Arc<dyn Cache>,
    ) -> Self {
        ProductService {
            repo,
            cache,
            default_ttl: Duration::from_secs(300), // 5 นาที
        }
    }
    
    /// Cache-aside pattern
    pub async fn get_product(&self, id: Uuid) -> Option<Product> {
        let cache_key = format!("product:{}", id);
        
        // 1. ลอง cache ก่อน
        if let Ok(Some(product)) = self.cache.get::<Product>(&cache_key).await {
            tracing::debug!("Cache HIT for product {}", id);
            return Some(product);
        }
        
        tracing::debug!("Cache MISS for product {}", id);
        
        // 2. ดึงจาก DB
        let product = self.repo.find_by_id(id).await?;
        
        // 3. เก็บใส่ cache
        if let Err(e) = self.cache.set(&cache_key, &product, Some(self.default_ttl)).await {
            tracing::warn!("Failed to cache product {}: {}", id, e);
        }
        
        Some(product)
    }
    
    /// Get products by category with cache
    pub async fn get_by_category(&self, category: &str) -> Vec<Product> {
        let cache_key = format!("products:category:{}", category);
        
        if let Ok(Some(products)) = self.cache.get::<Vec<Product>>(&cache_key).await {
            tracing::debug!("Cache HIT for category {}", category);
            return products;
        }
        
        let products = self.repo.find_by_category(category).await;
        
        if !products.is_empty() {
            let _ = self.cache.set(&cache_key, &products, Some(Duration::from_secs(60))).await;
        }
        
        products
    }
    
    /// Invalidate cache เมื่ออัพเดท product
    pub async fn update_product(&self, product: Product) -> Result<(), String> {
        // อัพเดท DB
        // self.repo.update(&product).await?;
        
        // Invalidate caches
        let cache_key = format!("product:{}", product.id);
        let _ = self.cache.delete(&cache_key).await;
        
        // Invalidate category cache ด้วย
        let category_key = format!("products:category:{}", product.category);
        let _ = self.cache.delete(&category_key).await;
        
        Ok(())
    }
    
    /// Invalidate ทั้งหมดที่เกี่ยวกับ product
    pub async fn invalidate_all(&self) -> u64 {
        self.cache.delete_pattern("product:*").await.unwrap_or(0)
    }
}
```

## Pattern 2: Write-through Cache

```rust
// src/patterns/write_through.rs
use std::sync::Arc;
use std::time::Duration;
use uuid::Uuid;
use serde::{Deserialize, Serialize};

use crate::cache::Cache;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UserProfile {
    pub id: Uuid,
    pub username: String,
    pub bio: Option<String>,
    pub avatar_url: Option<String>,
    pub follower_count: u64,
}

pub struct UserProfileService {
    cache: Arc<dyn Cache>,
    ttl: Duration,
}

impl UserProfileService {
    pub fn new(cache: Arc<dyn Cache>) -> Self {
        UserProfileService {
            cache,
            ttl: Duration::from_secs(3600), // 1 ชั่วโมง
        }
    }
    
    /// Write-through: เขียน cache และ DB พร้อมกัน
    pub async fn save_profile(&self, profile: &UserProfile) -> Result<(), String> {
        // 1. เขียน DB ก่อน
        // self.db_repo.save(profile).await?;
        
        // 2. อัพเดท cache ทันที
        let cache_key = format!("user:profile:{}", profile.id);
        self.cache
            .set(&cache_key, profile, Some(self.ttl))
            .await
            .map_err(|e| e.to_string())?;
        
        tracing::info!("Profile {} written to DB and cache", profile.id);
        Ok(())
    }
    
    pub async fn get_profile(&self, user_id: Uuid) -> Option<UserProfile> {
        let cache_key = format!("user:profile:{}", user_id);
        
        // Cache-aside สำหรับ read
        if let Ok(Some(profile)) = self.cache.get::<UserProfile>(&cache_key).await {
            return Some(profile);
        }
        
        // ดึงจาก DB
        // let profile = self.db_repo.find_by_id(user_id).await?;
        let profile = UserProfile {
            id: user_id,
            username: "testuser".to_string(),
            bio: None,
            avatar_url: None,
            follower_count: 0,
        };
        
        // เขียนเข้า cache
        let _ = self.cache.set(&cache_key, &profile, Some(self.ttl)).await;
        
        Some(profile)
    }
}
```

## Pattern 3: Write-behind Cache

```rust
// src/patterns/write_behind.rs
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::mpsc;
use tokio::time::interval;
use serde::{Deserialize, Serialize};
use uuid::Uuid;

use crate::cache::Cache;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PageView {
    pub page_id: Uuid,
    pub view_count: u64,
    pub last_viewed_at: chrono::DateTime<chrono::Utc>,
}

pub struct PageViewService {
    cache: Arc<dyn Cache>,
    write_sender: mpsc::Sender<(Uuid, u64)>,
}

impl PageViewService {
    pub fn new(cache: Arc<dyn Cache>) -> (Self, mpsc::Receiver<(Uuid, u64)>) {
        let (tx, rx) = mpsc::channel(1000);
        
        let service = PageViewService {
            cache,
            write_sender: tx,
        };
        
        (service, rx)
    }
    
    /// Write-behind: บันทึกใน cache ก่อน แล้ว async sync DB
    pub async fn record_view(&self, page_id: Uuid) -> Result<u64, String> {
        let cache_key = format!("pageview:count:{}", page_id);
        
        // Increment counter ใน cache
        let count = self.cache
            .increment(&cache_key)
            .await
            .map_err(|e| e.to_string())?;
        
        // Set TTL ถ้ายังไม่มี
        if count == 1 {
            let _ = self.cache.expire(&cache_key, Duration::from_secs(86400)).await;
        }
        
        // ส่ง signal ให้ background task sync DB
        let _ = self.write_sender.try_send((page_id, count as u64));
        
        Ok(count as u64)
    }
    
    pub async fn get_view_count(&self, page_id: Uuid) -> u64 {
        let cache_key = format!("pageview:count:{}", page_id);
        
        self.cache.get::<i64>(&cache_key).await
            .ok()
            .flatten()
            .unwrap_or(0) as u64
    }
}

/// Background task ที่ sync data จาก cache ไป DB
pub async fn sync_to_database(mut receiver: mpsc::Receiver<(Uuid, u64)>) {
    let mut interval = interval(Duration::from_secs(5));
    let mut pending: std::collections::HashMap<Uuid, u64> = std::collections::HashMap::new();
    
    loop {
        tokio::select! {
            // รับ events ใหม่
            Some((page_id, count)) = receiver.recv() => {
                pending.insert(page_id, count);
            }
            // Flush ทุก 5 วินาที
            _ = interval.tick() => {
                if !pending.is_empty() {
                    let items: Vec<(Uuid, u64)> = pending.drain().collect();
                    tracing::info!("Syncing {} page views to DB", items.len());
                    
                    // Batch insert to DB
                    for (page_id, count) in items {
                        tracing::debug!("Updating page {} views: {}", page_id, count);
                        // db.update_view_count(page_id, count).await;
                    }
                }
            }
        }
    }
}
```

## Redis Sorted Sets - Leaderboard

```rust
// src/leaderboard/mod.rs
use std::sync::Arc;
use serde::{Deserialize, Serialize};
use redis::AsyncCommands;

#[derive(Debug, Serialize, Deserialize)]
pub struct LeaderboardEntry {
    pub player_id: String,
    pub score: f64,
    pub rank: i64,
}

pub struct LeaderboardService {
    redis: Arc<redis::Client>,
    board_key: String,
}

impl LeaderboardService {
    pub fn new(redis: Arc<redis::Client>, board_name: &str) -> Self {
        LeaderboardService {
            redis,
            board_key: format!("leaderboard:{}", board_name),
        }
    }
    
    async fn get_conn(&self) -> Result<redis::aio::ConnectionManager, String> {
        redis::aio::ConnectionManager::new((*self.redis).clone())
            .await
            .map_err(|e| e.to_string())
    }
    
    /// เพิ่มหรืออัพเดท score
    pub async fn update_score(&self, player_id: &str, score: f64) -> Result<(), String> {
        let mut conn = self.get_conn().await?;
        
        conn.zadd(&self.board_key, score, player_id)
            .await
            .map_err(|e| e.to_string())
    }
    
    /// เพิ่ม score (increment)
    pub async fn add_score(&self, player_id: &str, delta: f64) -> Result<f64, String> {
        let mut conn = self.get_conn().await?;
        
        conn.zadd::<_, _, _, f64>(&self.board_key, delta, player_id)
            .await
            .map_err(|e| e.to_string())
    }
    
    /// ดึง top N players
    pub async fn get_top(&self, limit: isize) -> Result<Vec<LeaderboardEntry>, String> {
        let mut conn = self.get_conn().await?;
        
        // ZREVRANGE with scores
        let results: Vec<(String, f64)> = conn
            .zrevrange_withscores(&self.board_key, 0, limit - 1)
            .await
            .map_err(|e| e.to_string())?;
        
        Ok(results.into_iter().enumerate().map(|(i, (player_id, score))| {
            LeaderboardEntry {
                player_id,
                score,
                rank: (i + 1) as i64,
            }
        }).collect())
    }
    
    /// ดึง rank ของ player
    pub async fn get_rank(&self, player_id: &str) -> Result<Option<i64>, String> {
        let mut conn = self.get_conn().await?;
        
        let rank: Option<i64> = conn
            .zrevrank(&self.board_key, player_id)
            .await
            .map_err(|e| e.to_string())?;
        
        // Redis rank เริ่มที่ 0
        Ok(rank.map(|r| r + 1))
    }
    
    /// ดึง score ของ player
    pub async fn get_score(&self, player_id: &str) -> Result<Option<f64>, String> {
        let mut conn = self.get_conn().await?;
        
        conn.zscore(&self.board_key, player_id)
            .await
            .map_err(|e| e.to_string())
    }
    
    /// ดึง players ใกล้เคียง (neighbor ranking)
    pub async fn get_neighbors(
        &self,
        player_id: &str,
        range: isize,
    ) -> Result<Vec<LeaderboardEntry>, String> {
        let mut conn = self.get_conn().await?;
        
        let rank: Option<i64> = conn
            .zrevrank(&self.board_key, player_id)
            .await
            .map_err(|e| e.to_string())?;
        
        let rank = match rank {
            Some(r) => r,
            None => return Ok(vec![]),
        };
        
        let start = (rank - range).max(0);
        let end = rank + range;
        
        let results: Vec<(String, f64)> = conn
            .zrevrange_withscores(&self.board_key, start, end)
            .await
            .map_err(|e| e.to_string())?;
        
        Ok(results.into_iter().enumerate().map(|(i, (pid, score))| {
            LeaderboardEntry {
                player_id: pid,
                score,
                rank: start + i as i64 + 1,
            }
        }).collect())
    }
    
    /// Total players
    pub async fn total_players(&self) -> Result<u64, String> {
        let mut conn = self.get_conn().await?;
        
        conn.zcard(&self.board_key)
            .await
            .map_err(|e| e.to_string())
    }
}
```

## Cache Invalidation Strategies

```rust
// src/invalidation/mod.rs
use std::sync::Arc;
use std::time::Duration;
use chrono::{DateTime, Utc};

use crate::cache::Cache;

/// Time-based invalidation (TTL)
pub struct TtlCache {
    cache: Arc<dyn Cache>,
}

impl TtlCache {
    /// Short TTL สำหรับข้อมูลที่เปลี่ยนบ่อย
    pub async fn cache_volatile<T: serde::Serialize + serde::de::DeserializeOwned + Send + Sync>(
        &self,
        key: &str,
        value: &T,
    ) -> Result<(), crate::cache::CacheError> {
        self.cache.set(key, value, Some(Duration::from_secs(30))).await
    }
    
    /// Long TTL สำหรับข้อมูลที่เปลี่ยนนานๆ
    pub async fn cache_stable<T: serde::Serialize + serde::de::DeserializeOwned + Send + Sync>(
        &self,
        key: &str,
        value: &T,
    ) -> Result<(), crate::cache::CacheError> {
        self.cache.set(key, value, Some(Duration::from_secs(3600))).await
    }
}

/// Event-driven invalidation
pub struct EventDrivenCache {
    cache: Arc<dyn Cache>,
}

impl EventDrivenCache {
    /// Invalidate เมื่อ product อัพเดท
    pub async fn on_product_updated(&self, product_id: uuid::Uuid, category: &str) {
        let keys = vec![
            format!("product:{}", product_id),
            format!("products:category:{}", category),
            format!("products:list:*"),
        ];
        
        for key in keys {
            let _ = self.cache.delete(&key).await;
        }
    }
    
    /// Invalidate เมื่อ user อัพเดท
    pub async fn on_user_updated(&self, user_id: uuid::Uuid) {
        let _ = self.cache.delete_pattern(&format!("user:{}:*", user_id)).await;
    }
}

/// Cache với Stale-While-Revalidate
#[derive(serde::Serialize, serde::Deserialize)]
pub struct CachedValue<T> {
    pub value: T,
    pub cached_at: DateTime<Utc>,
    pub stale_after_secs: u64,
}

impl<T: Clone> CachedValue<T> {
    pub fn new(value: T, stale_after_secs: u64) -> Self {
        CachedValue {
            value,
            cached_at: Utc::now(),
            stale_after_secs,
        }
    }
    
    pub fn is_stale(&self) -> bool {
        let elapsed = Utc::now().signed_duration_since(self.cached_at)
            .num_seconds() as u64;
        elapsed > self.stale_after_secs
    }
}
```

## HTTP Handlers with Caching

```rust
// src/interface/http/product_handler.rs
use actix_web::{web, HttpResponse, HttpRequest};
use std::sync::Arc;
use uuid::Uuid;
use std::time::Duration;

use crate::{
    patterns::cache_aside::ProductService,
    leaderboard::LeaderboardService,
    cache::RedisCache,
};

pub async fn get_product(
    service: web::Data<Arc<ProductService>>,
    path: web::Path<Uuid>,
) -> HttpResponse {
    match service.get_product(*path).await {
        Some(product) => HttpResponse::Ok().json(product),
        None => HttpResponse::NotFound().finish(),
    }
}

pub async fn get_products_by_category(
    service: web::Data<Arc<ProductService>>,
    path: web::Path<String>,
) -> HttpResponse {
    let products = service.get_by_category(&path).await;
    
    HttpResponse::Ok()
        .append_header(("Cache-Control", "public, max-age=60"))
        .json(products)
}

// Leaderboard endpoints
pub async fn get_top_players(
    leaderboard: web::Data<Arc<LeaderboardService>>,
) -> HttpResponse {
    match leaderboard.get_top(10).await {
        Ok(entries) => HttpResponse::Ok().json(entries),
        Err(e) => HttpResponse::InternalServerError().json(serde_json::json!({"error": e})),
    }
}

pub async fn submit_score(
    leaderboard: web::Data<Arc<LeaderboardService>>,
    path: web::Path<String>,
    body: web::Json<SubmitScoreRequest>,
) -> HttpResponse {
    match leaderboard.update_score(&path, body.score).await {
        Ok(_) => HttpResponse::Ok().json(serde_json::json!({"message": "Score updated"})),
        Err(e) => HttpResponse::InternalServerError().json(serde_json::json!({"error": e})),
    }
}

#[derive(serde::Deserialize)]
pub struct SubmitScoreRequest {
    pub score: f64,
}

/// HTTP Cache headers middleware
pub fn add_cache_headers(
    req: &HttpRequest,
    response: &mut actix_web::HttpResponseBuilder,
    max_age: u64,
    private: bool,
) {
    let cache_control = if private {
        format!("private, max-age={}", max_age)
    } else {
        format!("public, max-age={}", max_age)
    };
    
    response.append_header(("Cache-Control", cache_control));
    response.append_header(("Vary", "Accept, Accept-Encoding"));
}
```

## Rate Limiting กับ Redis

```rust
// src/rate_limit/mod.rs
use std::sync::Arc;
use std::time::Duration;
use redis::AsyncCommands;

pub struct RateLimiter {
    redis: Arc<redis::Client>,
    max_requests: u32,
    window_secs: u64,
}

impl RateLimiter {
    pub fn new(redis: Arc<redis::Client>, max_requests: u32, window_secs: u64) -> Self {
        RateLimiter { redis, max_requests, window_secs }
    }
    
    /// Sliding window rate limiting
    pub async fn check_rate_limit(&self, identifier: &str) -> Result<RateLimitResult, String> {
        let key = format!("ratelimit:{}", identifier);
        let mut conn = redis::aio::ConnectionManager::new((*self.redis).clone())
            .await
            .map_err(|e| e.to_string())?;
        
        let now = std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .unwrap()
            .as_millis() as i64;
        
        let window_start = now - (self.window_secs as i64 * 1000);
        
        // Remove old entries
        let _: () = redis::cmd("ZREMRANGEBYSCORE")
            .arg(&key)
            .arg("-inf")
            .arg(window_start)
            .query_async(&mut conn)
            .await
            .map_err(|e| e.to_string())?;
        
        // Count current requests
        let count: u32 = redis::cmd("ZCARD")
            .arg(&key)
            .query_async(&mut conn)
            .await
            .map_err(|e| e.to_string())?;
        
        if count >= self.max_requests {
            return Ok(RateLimitResult {
                allowed: false,
                current: count,
                limit: self.max_requests,
                reset_after_ms: (self.window_secs * 1000) as u32,
            });
        }
        
        // Add current request
        let _: () = redis::cmd("ZADD")
            .arg(&key)
            .arg(now)
            .arg(now.to_string())
            .query_async(&mut conn)
            .await
            .map_err(|e| e.to_string())?;
        
        // Set expiry
        let _: () = redis::cmd("EXPIRE")
            .arg(&key)
            .arg(self.window_secs)
            .query_async(&mut conn)
            .await
            .map_err(|e| e.to_string())?;
        
        Ok(RateLimitResult {
            allowed: true,
            current: count + 1,
            limit: self.max_requests,
            reset_after_ms: (self.window_secs * 1000) as u32,
        })
    }
}

pub struct RateLimitResult {
    pub allowed: bool,
    pub current: u32,
    pub limit: u32,
    pub reset_after_ms: u32,
}
```

## Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::Arc;
    use std::collections::HashMap;
    use std::sync::Mutex;
    use async_trait::async_trait;
    use std::time::Duration;
    
    // Mock Cache
    struct MockCache {
        data: Mutex<HashMap<String, String>>,
    }
    
    impl MockCache {
        fn new() -> Self {
            MockCache { data: Mutex::new(HashMap::new()) }
        }
    }
    
    #[async_trait]
    impl Cache for MockCache {
        async fn get<T: serde::de::DeserializeOwned>(&self, key: &str) -> Result<Option<T>, CacheError> {
            let data = self.data.lock().unwrap();
            match data.get(key) {
                Some(json) => {
                    let value = serde_json::from_str(json)
                        .map_err(|e| CacheError::SerializationError(e.to_string()))?;
                    Ok(Some(value))
                }
                None => Ok(None),
            }
        }
        
        async fn set<T: serde::Serialize + Send + Sync>(
            &self,
            key: &str,
            value: &T,
            _ttl: Option<Duration>,
        ) -> Result<(), CacheError> {
            let json = serde_json::to_string(value)
                .map_err(|e| CacheError::SerializationError(e.to_string()))?;
            self.data.lock().unwrap().insert(key.to_string(), json);
            Ok(())
        }
        
        async fn delete(&self, key: &str) -> Result<(), CacheError> {
            self.data.lock().unwrap().remove(key);
            Ok(())
        }
        
        async fn delete_pattern(&self, pattern: &str) -> Result<u64, CacheError> {
            let prefix = pattern.trim_end_matches('*');
            let mut data = self.data.lock().unwrap();
            let keys_to_delete: Vec<String> = data.keys()
                .filter(|k| k.starts_with(prefix))
                .cloned()
                .collect();
            let count = keys_to_delete.len() as u64;
            for k in keys_to_delete {
                data.remove(&k);
            }
            Ok(count)
        }
        
        async fn exists(&self, key: &str) -> Result<bool, CacheError> {
            Ok(self.data.lock().unwrap().contains_key(key))
        }
        
        async fn increment(&self, key: &str) -> Result<i64, CacheError> {
            let mut data = self.data.lock().unwrap();
            let current: i64 = data.get(key)
                .and_then(|v| v.parse().ok())
                .unwrap_or(0);
            let new_val = current + 1;
            data.insert(key.to_string(), new_val.to_string());
            Ok(new_val)
        }
        
        async fn expire(&self, _key: &str, _ttl: Duration) -> Result<(), CacheError> {
            Ok(())
        }
    }
    
    #[tokio::test]
    async fn test_cache_aside_miss_then_hit() {
        let cache = Arc::new(MockCache::new());
        let repo = Arc::new(ProductRepository::new());
        let service = ProductService::new(repo.clone(), cache.clone());
        
        // Get first product ID
        let product_id = {
            let products: Vec<_> = repo.products.values().take(1).collect();
            products[0].id
        };
        
        // First call - cache miss
        let product1 = service.get_product(product_id).await;
        assert!(product1.is_some());
        
        // Verify it's in cache
        assert!(cache.exists(&format!("product:{}", product_id)).await.unwrap());
        
        // Second call - should be cache hit (same result)
        let product2 = service.get_product(product_id).await;
        assert_eq!(product1.unwrap().id, product2.unwrap().id);
    }
    
    #[tokio::test]
    async fn test_cache_invalidation() {
        let cache = Arc::new(MockCache::new());
        let repo = Arc::new(ProductRepository::new());
        let service = ProductService::new(repo.clone(), cache.clone());
        
        let product_id = {
            let products: Vec<_> = repo.products.values().take(1).collect();
            products[0].id
        };
        
        // Cache product
        service.get_product(product_id).await;
        assert!(cache.exists(&format!("product:{}", product_id)).await.unwrap());
        
        // Get product and update
        let mut product = service.get_product(product_id).await.unwrap();
        product.price = 999.0;
        
        // Update should invalidate cache
        service.update_product(product).await.unwrap();
        assert!(!cache.exists(&format!("product:{}", product_id)).await.unwrap());
    }
    
    #[tokio::test]
    async fn test_page_view_write_behind() {
        let cache = Arc::new(MockCache::new());
        let (service, _rx) = PageViewService::new(cache);
        
        let page_id = uuid::Uuid::new_v4();
        
        service.record_view(page_id).await.unwrap();
        service.record_view(page_id).await.unwrap();
        service.record_view(page_id).await.unwrap();
        
        let count = service.get_view_count(page_id).await;
        assert_eq!(count, 3);
    }
}
```

## สรุป

Caching Patterns เปรียบเทียบ:

| Pattern | เหมาะกับ | ข้อดี | ข้อเสีย |
|---------|---------|-------|---------|
| Cache-aside | Read-heavy | Simple | Cache miss cost |
| Write-through | Read+Write balance | Consistent | Write latency |
| Write-behind | Write-heavy | Fast writes | Data loss risk |
| Read-through | Read-heavy | Transparent | Complex setup |

---

## Navigation

- [← Part 066: API Versioning](../part_066/README.md)
- [→ Part 068: Configuration Management](../part_068/README.md)
- [กลับหน้าหลัก](../../README.md)
