# Part 045: Rate Limiting ⏱️

## 🎯 เป้าหมายของ Part นี้

- ทำความเข้าใจ Rate Limiting concepts
- ใช้ `actix-governor` crate
- Per-IP และ Per-user rate limiting
- Rate limit headers
- Redis-backed rate limiting
- Burst handling
- Bypass สำหรับ trusted IPs

---

## 1. Rate Limiting Concepts

### 1.1 ทำไมต้องมี Rate Limiting?

- ป้องกัน DDoS attacks
- ป้องกัน brute force login
- ควบคุม API usage
- ป้องกัน scraping
- รักษา service quality

### 1.2 Algorithms

**Token Bucket:**
```
bucket_capacity = 100 tokens
refill_rate = 10 tokens/second
request consumes 1 token
```

**Sliding Window:**
```
window = 1 minute
max_requests = 60
count requests in last 60 seconds
```

**Fixed Window:**
```
window = 1 minute (00:00 - 00:59)
max_requests = 60
reset counter every minute
```

---

## 2. Setup

### 2.1 Cargo.toml

```toml
[package]
name = "rate-limiting"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
actix-governor = "0.5"
governor = "0.6"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
log = "0.4"
env_logger = "0.10"
dotenv = "0.15"
redis = { version = "0.23", features = ["tokio-comp"] }
uuid = { version = "1", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "1"
futures = "0.3"
std-nonzero-usize = "0.1"
```

---

## 3. Basic Rate Limiting ด้วย actix-governor

```rust
// src/basic_rate_limit.rs
use actix_governor::{Governor, GovernorConfigBuilder, KeyExtractor, SimpleKeyExtractionError};
use actix_web::{dev::ServiceRequest, web, HttpResponse};

/// Rate limiting ด้วย Per-IP
pub fn create_ip_rate_limiter() -> Governor<actix_governor::PeerIpKeyExtractor, actix_governor::NoOpMiddleware> {
    let config = GovernorConfigBuilder::default()
        .per_second(10)        // 10 requests ต่อวินาที
        .burst_size(30)        // burst สูงสุด 30 requests
        .finish()
        .unwrap();
    
    Governor::new(&config)
}

/// Rate limiting แบบ strict สำหรับ login endpoint
pub fn create_login_rate_limiter() -> Governor<actix_governor::PeerIpKeyExtractor, actix_governor::NoOpMiddleware> {
    let config = GovernorConfigBuilder::default()
        .per_minute(5)         // 5 attempts ต่อนาที
        .burst_size(3)         // burst สูงสุด 3
        .finish()
        .unwrap();
    
    Governor::new(&config)
}
```

---

## 4. Custom Key Extractor

```rust
// src/extractors.rs
use actix_governor::{KeyExtractor, SimpleKeyExtractionError};
use actix_web::dev::ServiceRequest;

/// Extract user ID จาก JWT token สำหรับ per-user rate limiting
#[derive(Clone)]
pub struct UserIdExtractor;

impl KeyExtractor for UserIdExtractor {
    type Key = String;
    type KeyExtractionError = SimpleKeyExtractionError<&'static str>;
    
    fn extract(&self, req: &ServiceRequest) -> Result<Self::Key, Self::KeyExtractionError> {
        // ดึง JWT token จาก header
        let token = req
            .headers()
            .get("Authorization")
            .and_then(|h| h.to_str().ok())
            .and_then(|h| h.strip_prefix("Bearer "))
            .map(|t| t.to_string());
        
        match token {
            Some(t) => {
                // Decode JWT เพื่อดึง user ID
                let jwt_secret = std::env::var("JWT_SECRET").unwrap_or_default();
                match decode_user_id(&t, &jwt_secret) {
                    Some(user_id) => Ok(format!("user:{}", user_id)),
                    None => {
                        // ถ้า token ไม่ valid ให้ใช้ IP แทน
                        Ok(format!("ip:{}", get_ip(req)))
                    }
                }
            },
            None => Ok(format!("ip:{}", get_ip(req))),
        }
    }
    
    fn exceed_rate_limit_response(
        &self,
        negative: &governor::NotUntil<governor::clock::QuantaInstant>,
        mut response: actix_web::HttpResponseBuilder,
    ) -> actix_web::HttpResponse {
        let wait_time = negative.wait_time_from(governor::clock::QuantaClock::default().now());
        
        response
            .append_header(("X-RateLimit-Retry-After", wait_time.as_secs().to_string()))
            .json(serde_json::json!({
                "error": "Too many requests",
                "retry_after_seconds": wait_time.as_secs()
            }))
    }
}

fn get_ip(req: &ServiceRequest) -> String {
    // ตรวจสอบ X-Forwarded-For header ก่อน (สำหรับ reverse proxy)
    req.headers()
        .get("X-Forwarded-For")
        .and_then(|h| h.to_str().ok())
        .and_then(|h| h.split(',').next())
        .map(|ip| ip.trim().to_string())
        .unwrap_or_else(|| {
            req.peer_addr()
                .map(|addr| addr.ip().to_string())
                .unwrap_or_else(|| "unknown".to_string())
        })
}

fn decode_user_id(token: &str, secret: &str) -> Option<String> {
    use jsonwebtoken::{decode, DecodingKey, Validation};
    use serde::Deserialize;
    
    #[derive(Deserialize)]
    struct MinimalClaims {
        sub: String,
    }
    
    decode::<MinimalClaims>(
        token,
        &DecodingKey::from_secret(secret.as_bytes()),
        &Validation::default(),
    )
    .ok()
    .map(|data| data.claims.sub)
}

/// API Key extractor - rate limit per API key
#[derive(Clone)]
pub struct ApiKeyExtractor;

impl KeyExtractor for ApiKeyExtractor {
    type Key = String;
    type KeyExtractionError = SimpleKeyExtractionError<&'static str>;
    
    fn extract(&self, req: &ServiceRequest) -> Result<Self::Key, Self::KeyExtractionError> {
        // ลอง X-API-Key header ก่อน
        let api_key = req
            .headers()
            .get("X-API-Key")
            .and_then(|h| h.to_str().ok())
            .map(|k| format!("apikey:{}", k));
        
        // ถ้าไม่มี API key ใช้ IP
        Ok(api_key.unwrap_or_else(|| {
            format!("ip:{}", get_ip(req))
        }))
    }
}
```

---

## 5. Rate Limit Headers

```rust
// src/headers.rs
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpMessage,
};
use futures::future::{ready, Ready, LocalBoxFuture};
use std::rc::Rc;

/// Middleware เพิ่ม Rate Limit headers
pub struct RateLimitHeaders {
    max_requests: u64,
    window_seconds: u64,
}

impl RateLimitHeaders {
    pub fn new(max_requests: u64, window_seconds: u64) -> Self {
        Self { max_requests, window_seconds }
    }
}

impl<S, B> Transform<S, ServiceRequest> for RateLimitHeaders
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Transform = RateLimitHeadersMiddleware<S>;
    type InitError = ();
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(RateLimitHeadersMiddleware {
            service: Rc::new(service),
            max_requests: self.max_requests,
            window_seconds: self.window_seconds,
        }))
    }
}

pub struct RateLimitHeadersMiddleware<S> {
    service: Rc<S>,
    max_requests: u64,
    window_seconds: u64,
}

impl<S, B> Service<ServiceRequest> for RateLimitHeadersMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;
    
    forward_ready!(service);
    
    fn call(&self, req: ServiceRequest) -> Self::Future {
        let service = self.service.clone();
        let max_requests = self.max_requests;
        let window_seconds = self.window_seconds;
        
        Box::pin(async move {
            let mut response = service.call(req).await?;
            
            // เพิ่ม standard rate limit headers
            let headers = response.headers_mut();
            headers.insert(
                actix_web::http::header::HeaderName::from_static("x-ratelimit-limit"),
                actix_web::http::header::HeaderValue::from_str(&max_requests.to_string()).unwrap(),
            );
            headers.insert(
                actix_web::http::header::HeaderName::from_static("x-ratelimit-window"),
                actix_web::http::header::HeaderValue::from_str(&window_seconds.to_string()).unwrap(),
            );
            
            Ok(response)
        })
    }
}
```

---

## 6. Redis-backed Rate Limiting

```rust
// src/redis_rate_limiter.rs
use redis::AsyncCommands;
use chrono::Utc;

#[derive(Clone)]
pub struct RedisRateLimiter {
    client: redis::Client,
    max_requests: u64,
    window_seconds: u64,
}

impl RedisRateLimiter {
    pub fn new(redis_url: &str, max_requests: u64, window_seconds: u64) -> anyhow::Result<Self> {
        let client = redis::Client::open(redis_url)?;
        Ok(Self { client, max_requests, window_seconds })
    }
    
    /// Sliding window rate limit implementation
    pub async fn check_rate_limit(&self, key: &str) -> anyhow::Result<RateLimitResult> {
        let mut conn = self.client.get_async_connection().await?;
        
        let now = Utc::now().timestamp_millis();
        let window_start = now - (self.window_seconds as i64 * 1000);
        
        let redis_key = format!("rate_limit:{}", key);
        
        // ใช้ Lua script เพื่อให้ atomic
        let script = redis::Script::new(r#"
            local key = KEYS[1]
            local now = tonumber(ARGV[1])
            local window_start = tonumber(ARGV[2])
            local max_requests = tonumber(ARGV[3])
            local window_seconds = tonumber(ARGV[4])
            
            -- ลบ entries เก่าออก
            redis.call('ZREMRANGEBYSCORE', key, 0, window_start)
            
            -- นับ requests ในช่วง window
            local count = redis.call('ZCARD', key)
            
            if count >= max_requests then
                -- Rate limit exceeded
                local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
                local reset_at = 0
                if #oldest > 0 then
                    reset_at = tonumber(oldest[2]) + (window_seconds * 1000)
                end
                return {0, count, reset_at}
            else
                -- Add current request
                redis.call('ZADD', key, now, now .. '-' .. math.random())
                redis.call('EXPIRE', key, window_seconds + 1)
                return {1, count + 1, 0}
            end
        "#);
        
        let result: Vec<i64> = script
            .key(&redis_key)
            .arg(now)
            .arg(window_start)
            .arg(self.max_requests)
            .arg(self.window_seconds)
            .invoke_async(&mut conn)
            .await?;
        
        let allowed = result[0] == 1;
        let current_count = result[1] as u64;
        let reset_at = result[2];
        
        Ok(RateLimitResult {
            allowed,
            current_count,
            max_requests: self.max_requests,
            remaining: if allowed {
                self.max_requests.saturating_sub(current_count)
            } else {
                0
            },
            reset_at: if reset_at > 0 {
                Some(reset_at)
            } else {
                None
            },
        })
    }
    
    /// Fixed window rate limit (ง่ายกว่า)
    pub async fn check_fixed_window(&self, key: &str) -> anyhow::Result<RateLimitResult> {
        let mut conn = self.client.get_async_connection().await?;
        
        // Key มี window timestamp เพื่อให้ auto-reset
        let window = Utc::now().timestamp() / self.window_seconds as i64;
        let redis_key = format!("rate_limit:{}:{}", key, window);
        
        let count: u64 = conn.incr(&redis_key, 1).await?;
        
        if count == 1 {
            // Set expiry เมื่อสร้าง key ใหม่
            let _: () = conn.expire(&redis_key, self.window_seconds as usize).await?;
        }
        
        let allowed = count <= self.max_requests;
        
        Ok(RateLimitResult {
            allowed,
            current_count: count,
            max_requests: self.max_requests,
            remaining: self.max_requests.saturating_sub(count),
            reset_at: None,
        })
    }
}

#[derive(Debug, Clone)]
pub struct RateLimitResult {
    pub allowed: bool,
    pub current_count: u64,
    pub max_requests: u64,
    pub remaining: u64,
    pub reset_at: Option<i64>,
}

impl RateLimitResult {
    pub fn to_headers(&self) -> Vec<(String, String)> {
        let mut headers = vec![
            ("X-RateLimit-Limit".to_string(), self.max_requests.to_string()),
            ("X-RateLimit-Remaining".to_string(), self.remaining.to_string()),
        ];
        
        if let Some(reset) = self.reset_at {
            headers.push(("X-RateLimit-Reset".to_string(), reset.to_string()));
        }
        
        headers
    }
}
```

---

## 7. Trusted IP Bypass

```rust
// src/trusted_ips.rs
use std::collections::HashSet;
use std::net::IpAddr;

#[derive(Clone)]
pub struct TrustedIps {
    ips: HashSet<IpAddr>,
    cidrs: Vec<(IpAddr, u8)>,
}

impl TrustedIps {
    pub fn new() -> Self {
        Self {
            ips: HashSet::new(),
            cidrs: Vec::new(),
        }
    }
    
    pub fn from_env() -> Self {
        let mut trusted = Self::new();
        
        if let Ok(ips) = std::env::var("TRUSTED_IPS") {
            for ip_str in ips.split(',') {
                let ip_str = ip_str.trim();
                if let Ok(ip) = ip_str.parse::<IpAddr>() {
                    trusted.add_ip(ip);
                } else if ip_str.contains('/') {
                    // CIDR notation
                    let parts: Vec<&str> = ip_str.splitn(2, '/').collect();
                    if parts.len() == 2 {
                        if let (Ok(ip), Ok(prefix)) = (
                            parts[0].parse::<IpAddr>(),
                            parts[1].parse::<u8>(),
                        ) {
                            trusted.add_cidr(ip, prefix);
                        }
                    }
                }
            }
        }
        
        // เพิ่ม localhost เป็น trusted เสมอ
        trusted.add_ip("127.0.0.1".parse().unwrap());
        trusted.add_ip("::1".parse().unwrap());
        
        trusted
    }
    
    pub fn add_ip(&mut self, ip: IpAddr) {
        self.ips.insert(ip);
    }
    
    pub fn add_cidr(&mut self, network: IpAddr, prefix: u8) {
        self.cidrs.push((network, prefix));
    }
    
    pub fn is_trusted(&self, ip: &IpAddr) -> bool {
        // ตรวจสอบ exact match
        if self.ips.contains(ip) {
            return true;
        }
        
        // ตรวจสอบ CIDR
        for (network, prefix) in &self.cidrs {
            if self.ip_in_cidr(ip, network, *prefix) {
                return true;
            }
        }
        
        false
    }
    
    fn ip_in_cidr(&self, ip: &IpAddr, network: &IpAddr, prefix: u8) -> bool {
        match (ip, network) {
            (IpAddr::V4(ip), IpAddr::V4(net)) => {
                let mask = !((1u32 << (32 - prefix)) - 1);
                let ip_u32 = u32::from(*ip);
                let net_u32 = u32::from(*net);
                (ip_u32 & mask) == (net_u32 & mask)
            },
            (IpAddr::V6(ip), IpAddr::V6(net)) => {
                let mask = !((1u128 << (128 - prefix)) - 1);
                let ip_u128 = u128::from(*ip);
                let net_u128 = u128::from(*net);
                (ip_u128 & mask) == (net_u128 & mask)
            },
            _ => false,
        }
    }
}
```

---

## 8. Rate Limit Middleware (Custom)

```rust
// src/middleware/rate_limit.rs
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpResponse,
};
use futures::future::{ready, Ready, LocalBoxFuture};
use std::rc::Rc;
use std::sync::Arc;
use crate::{
    redis_rate_limiter::RedisRateLimiter,
    trusted_ips::TrustedIps,
};

pub struct RateLimitMiddlewareFactory {
    limiter: Arc<RedisRateLimiter>,
    trusted_ips: Arc<TrustedIps>,
}

impl RateLimitMiddlewareFactory {
    pub fn new(limiter: RedisRateLimiter, trusted_ips: TrustedIps) -> Self {
        Self {
            limiter: Arc::new(limiter),
            trusted_ips: Arc::new(trusted_ips),
        }
    }
}

impl<S, B> Transform<S, ServiceRequest> for RateLimitMiddlewareFactory
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Transform = RateLimitMiddleware<S>;
    type InitError = ();
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(RateLimitMiddleware {
            service: Rc::new(service),
            limiter: self.limiter.clone(),
            trusted_ips: self.trusted_ips.clone(),
        }))
    }
}

pub struct RateLimitMiddleware<S> {
    service: Rc<S>,
    limiter: Arc<RedisRateLimiter>,
    trusted_ips: Arc<TrustedIps>,
}

impl<S, B> Service<ServiceRequest> for RateLimitMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;
    
    forward_ready!(service);
    
    fn call(&self, req: ServiceRequest) -> Self::Future {
        let service = self.service.clone();
        let limiter = self.limiter.clone();
        let trusted_ips = self.trusted_ips.clone();
        
        Box::pin(async move {
            // ดึง client IP
            let client_ip = req
                .peer_addr()
                .map(|a| a.ip())
                .unwrap_or(std::net::IpAddr::V4(std::net::Ipv4Addr::LOCALHOST));
            
            // Skip rate limit สำหรับ trusted IPs
            if trusted_ips.is_trusted(&client_ip) {
                return service.call(req).await;
            }
            
            // ดึง identifier (user ID หรือ IP)
            let identifier = req
                .headers()
                .get("Authorization")
                .and_then(|h| h.to_str().ok())
                .and_then(|h| h.strip_prefix("Bearer "))
                .map(|t| format!("user:{}", t))
                .unwrap_or_else(|| format!("ip:{}", client_ip));
            
            // ตรวจสอบ rate limit
            let result = match limiter.check_rate_limit(&identifier).await {
                Ok(r) => r,
                Err(e) => {
                    log::error!("Rate limiter error: {}", e);
                    // ถ้า Redis down ให้ผ่านไปก่อน (fail open)
                    return service.call(req).await;
                }
            };
            
            if !result.allowed {
                let headers = result.to_headers();
                let mut response = HttpResponse::TooManyRequests();
                
                for (name, value) in &headers {
                    response.append_header((name.as_str(), value.as_str()));
                }
                
                let response = req.into_response(
                    response
                        .json(serde_json::json!({
                            "error": "Too many requests",
                            "limit": result.max_requests,
                            "remaining": 0,
                        }))
                        .map_into_boxed_body()
                );
                return Ok(response);
            }
            
            let mut response = service.call(req).await?;
            
            // เพิ่ม rate limit headers
            for (name, value) in result.to_headers() {
                if let Ok(header_value) = actix_web::http::header::HeaderValue::from_str(&value) {
                    response.headers_mut().insert(
                        actix_web::http::header::HeaderName::from_bytes(name.to_lowercase().as_bytes()).unwrap(),
                        header_value,
                    );
                }
            }
            
            Ok(response)
        })
    }
}
```

---

## 9. Main Application

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware, HttpResponse};
use actix_governor::{Governor, GovernorConfigBuilder};
use dotenv::dotenv;

mod basic_rate_limit;
mod extractors;
mod headers;
mod redis_rate_limiter;
mod trusted_ips;
mod middleware {
    pub mod rate_limit;
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv().ok();
    env_logger::init();
    
    let redis_url = std::env::var("REDIS_URL")
        .unwrap_or_else(|_| "redis://127.0.0.1:6379".to_string());
    
    let trusted_ips = trusted_ips::TrustedIps::from_env();
    
    // สร้าง Redis-backed rate limiter
    let api_limiter = redis_rate_limiter::RedisRateLimiter::new(
        &redis_url,
        100, // max 100 requests
        60,  // per 60 seconds
    ).expect("Failed to create Redis rate limiter");
    
    // strict limiter สำหรับ login
    let login_limiter = redis_rate_limiter::RedisRateLimiter::new(
        &redis_url,
        5,   // max 5 requests
        60,  // per 60 seconds
    ).expect("Failed to create login rate limiter");
    
    // Governor-based rate limiter (ใช้ in-memory)
    let general_config = GovernorConfigBuilder::default()
        .per_second(20)
        .burst_size(50)
        .finish()
        .unwrap();
    
    HttpServer::new(move || {
        App::new()
            .wrap(middleware::Logger::default())
            .wrap(Governor::new(&general_config))
            .service(
                web::scope("/api")
                    .route("/public", web::get().to(public_endpoint))
                    .route("/protected", web::get().to(protected_endpoint))
            )
            .service(
                web::scope("/auth")
                    .route("/login", web::post().to(login_endpoint))
            )
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}

async fn public_endpoint() -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({"data": "Public data"}))
}

async fn protected_endpoint() -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({"data": "Protected data"}))
}

async fn login_endpoint(
    body: web::Json<serde_json::Value>,
) -> HttpResponse {
    // Login logic
    HttpResponse::Ok().json(serde_json::json!({"token": "example_token"}))
}
```

---

## 10. Testing

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::trusted_ips::TrustedIps;

    #[test]
    fn test_trusted_ips() {
        let mut trusted = TrustedIps::new();
        trusted.add_ip("192.168.1.1".parse().unwrap());
        trusted.add_cidr("10.0.0.0".parse().unwrap(), 8);
        
        assert!(trusted.is_trusted(&"192.168.1.1".parse().unwrap()));
        assert!(trusted.is_trusted(&"10.5.5.5".parse().unwrap()));
        assert!(trusted.is_trusted(&"127.0.0.1".parse().unwrap())); // localhost always trusted
        assert!(!trusted.is_trusted(&"8.8.8.8".parse().unwrap()));
    }
    
    #[test]
    fn test_cidr_check() {
        let mut trusted = TrustedIps::new();
        trusted.add_cidr("192.168.0.0".parse().unwrap(), 16);
        
        assert!(trusted.is_trusted(&"192.168.1.100".parse().unwrap()));
        assert!(trusted.is_trusted(&"192.168.255.255".parse().unwrap()));
        assert!(!trusted.is_trusted(&"192.169.0.1".parse().unwrap()));
    }
}
```

---

## 11. สรุปสิ่งที่เรียนรู้

✅ Token bucket และ sliding window concepts  
✅ actix-governor สำหรับ in-memory rate limiting  
✅ Custom KeyExtractor สำหรับ per-user limiting  
✅ X-RateLimit-* headers  
✅ Redis-backed sliding window rate limiting  
✅ Lua script สำหรับ atomic operations  
✅ Trusted IP bypass  
✅ CIDR notation support  
✅ Fail-open strategy เมื่อ Redis down  

---

*[← Part 044: RBAC](../part_044/README.md) | [Part 046: API Keys Management →](../part_046/README.md)*
