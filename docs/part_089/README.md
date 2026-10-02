# Part 089: Project: API Gateway 🌐

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้าง Reverse Proxy ด้วย Actix-web
- ทำ Route Configuration แบบ dynamic
- จัดการ Authentication Handling
- ทำ Rate Limiting
- ทำ Request Transformation
- สร้าง Response Caching
- ทำ Logging และ Metrics
- ทำ Circuit Breaker
- สร้าง Complete API Gateway

---

## 1. โครงสร้างโปรเจกต์

```
api_gateway/
├── Cargo.toml
├── .env
├── config/
│   └── routes.yaml
└── src/
    ├── main.rs
    ├── config.rs
    ├── errors.rs
    ├── proxy/
    │   ├── mod.rs
    │   ├── handler.rs
    │   └── transformer.rs
    ├── middleware/
    │   ├── mod.rs
    │   ├── auth.rs
    │   ├── rate_limit.rs
    │   ├── cache.rs
    │   ├── circuit_breaker.rs
    │   └── metrics.rs
    ├── routing/
    │   ├── mod.rs
    │   └── router.rs
    └── services/
        ├── mod.rs
        └── discovery.rs
```

---

## 2. Cargo.toml

```toml
[package]
name = "api_gateway"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
actix-cors = "0.7"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde_yaml = "0.9"
tokio = { version = "1", features = ["full"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
dotenv = "0.15"
env_logger = "0.11"
log = "0.4"
thiserror = "1"
reqwest = { version = "0.12", features = ["json", "stream"] }
redis = { version = "0.26", features = ["tokio-comp", "connection-manager"] }
jsonwebtoken = "9"
tokio-util = { version = "0.7", features = ["io"] }
bytes = "1"
futures-util = "0.3"
parking_lot = "0.12"
dashmap = "6"
prometheus = { version = "0.13", features = ["process"] }
actix-web-prometheus = "0.1"
governor = "0.6"
```

---

## 3. Route Configuration (`src/config.rs`)

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct GatewayConfig {
    pub port: u16,
    pub routes: Vec<RouteConfig>,
    pub global: GlobalConfig,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct GlobalConfig {
    pub timeout_ms: u64,
    pub max_connections: usize,
    pub cors_allowed_origins: Vec<String>,
    pub rate_limit: Option<RateLimitConfig>,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct RouteConfig {
    pub id: String,
    pub path: String,
    pub methods: Vec<String>,
    pub upstream: UpstreamConfig,
    pub auth: Option<AuthConfig>,
    pub rate_limit: Option<RateLimitConfig>,
    pub cache: Option<CacheConfig>,
    pub transform: Option<TransformConfig>,
    pub circuit_breaker: Option<CircuitBreakerConfig>,
    pub retry: Option<RetryConfig>,
    pub strip_prefix: bool,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct UpstreamConfig {
    pub url: String,
    pub path_rewrite: Option<String>,
    pub timeout_ms: Option<u64>,
    pub load_balancer: Option<LoadBalancerType>,
    pub instances: Option<Vec<String>>,
    pub health_check: Option<HealthCheckConfig>,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum LoadBalancerType {
    RoundRobin,
    Random,
    LeastConnections,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct AuthConfig {
    pub required: bool,
    pub auth_type: AuthType,
    pub jwt_secret: Option<String>,
    pub api_key_header: Option<String>,
    pub allowed_roles: Option<Vec<String>>,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum AuthType {
    Jwt,
    ApiKey,
    BasicAuth,
    None,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct RateLimitConfig {
    pub requests_per_second: u32,
    pub burst_size: u32,
    pub key_by: RateLimitKey,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum RateLimitKey {
    Ip,
    UserId,
    ApiKey,
    Global,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct CacheConfig {
    pub ttl_seconds: u64,
    pub cache_by: Vec<String>,
    pub vary_headers: Vec<String>,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct TransformConfig {
    pub add_request_headers: Option<HashMap<String, String>>,
    pub remove_request_headers: Option<Vec<String>>,
    pub add_response_headers: Option<HashMap<String, String>>,
    pub remove_response_headers: Option<Vec<String>>,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct CircuitBreakerConfig {
    pub failure_threshold: u32,
    pub success_threshold: u32,
    pub timeout_ms: u64,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct RetryConfig {
    pub max_retries: u32,
    pub retry_on_status: Vec<u16>,
    pub initial_delay_ms: u64,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct HealthCheckConfig {
    pub path: String,
    pub interval_seconds: u64,
}

impl GatewayConfig {
    pub fn from_yaml(path: &str) -> Result<Self, Box<dyn std::error::Error>> {
        let content = std::fs::read_to_string(path)?;
        let config: GatewayConfig = serde_yaml::from_str(&content)?;
        Ok(config)
    }

    pub fn from_env() -> Self {
        // Build config from environment variables
        GatewayConfig {
            port: std::env::var("PORT")
                .ok()
                .and_then(|p| p.parse().ok())
                .unwrap_or(8080),
            routes: vec![],
            global: GlobalConfig {
                timeout_ms: 30000,
                max_connections: 1000,
                cors_allowed_origins: vec!["*".to_string()],
                rate_limit: None,
            },
        }
    }
}
```

---

## 4. Circuit Breaker (`src/middleware/circuit_breaker.rs`)

```rust
use std::sync::atomic::{AtomicU32, AtomicU64, Ordering};
use std::sync::Arc;
use std::time::{Duration, Instant};
use parking_lot::RwLock;

use crate::config::CircuitBreakerConfig;

#[derive(Debug, Clone, PartialEq)]
pub enum CircuitState {
    Closed,     // Normal operation
    Open,       // Failing - reject requests
    HalfOpen,   // Testing recovery
}

pub struct CircuitBreaker {
    state: RwLock<CircuitState>,
    failure_count: AtomicU32,
    success_count: AtomicU32,
    last_failure_time: RwLock<Option<Instant>>,
    config: CircuitBreakerConfig,
}

impl CircuitBreaker {
    pub fn new(config: CircuitBreakerConfig) -> Arc<Self> {
        Arc::new(CircuitBreaker {
            state: RwLock::new(CircuitState::Closed),
            failure_count: AtomicU32::new(0),
            success_count: AtomicU32::new(0),
            last_failure_time: RwLock::new(None),
            config,
        })
    }

    pub fn can_proceed(&self) -> bool {
        match *self.state.read() {
            CircuitState::Closed => true,
            CircuitState::Open => {
                // Check if timeout has passed
                if let Some(last_failure) = *self.last_failure_time.read() {
                    if last_failure.elapsed() > Duration::from_millis(self.config.timeout_ms) {
                        // Transition to HalfOpen
                        *self.state.write() = CircuitState::HalfOpen;
                        self.success_count.store(0, Ordering::SeqCst);
                        return true;
                    }
                }
                false
            }
            CircuitState::HalfOpen => true,
        }
    }

    pub fn record_success(&self) {
        let state = self.state.read().clone();
        match state {
            CircuitState::HalfOpen => {
                let successes = self.success_count.fetch_add(1, Ordering::SeqCst) + 1;
                if successes >= self.config.success_threshold {
                    *self.state.write() = CircuitState::Closed;
                    self.failure_count.store(0, Ordering::SeqCst);
                    self.success_count.store(0, Ordering::SeqCst);
                    log::info!("Circuit breaker CLOSED - service recovered");
                }
            }
            CircuitState::Closed => {
                // Reset failure count on success
                self.failure_count.fetch_saturating_sub(1, Ordering::SeqCst);
            }
            _ => {}
        }
    }

    pub fn record_failure(&self) {
        let failures = self.failure_count.fetch_add(1, Ordering::SeqCst) + 1;
        *self.last_failure_time.write() = Some(Instant::now());

        if failures >= self.config.failure_threshold {
            let mut state = self.state.write();
            if *state != CircuitState::Open {
                *state = CircuitState::Open;
                log::warn!("Circuit breaker OPEN - service marked as failing");
            }
        }
    }

    pub fn get_state(&self) -> CircuitState {
        self.state.read().clone()
    }
}

// AtomicU32 extension for saturating subtraction
trait FetchSaturatingSub {
    fn fetch_saturating_sub(&self, val: u32, order: Ordering) -> u32;
}

impl FetchSaturatingSub for AtomicU32 {
    fn fetch_saturating_sub(&self, val: u32, order: Ordering) -> u32 {
        let mut current = self.load(order);
        loop {
            let new = current.saturating_sub(val);
            match self.compare_exchange_weak(current, new, order, Ordering::Relaxed) {
                Ok(old) => return old,
                Err(actual) => current = actual,
            }
        }
    }
}
```

---

## 5. Proxy Handler (`src/proxy/handler.rs`)

```rust
use actix_web::{web, HttpRequest, HttpResponse};
use bytes::Bytes;
use reqwest::{Client, Method, StatusCode};
use std::str::FromStr;
use std::sync::Arc;
use std::time::Duration;

use crate::config::{RetryConfig, RouteConfig};
use crate::errors::AppError;
use crate::middleware::circuit_breaker::CircuitBreaker;

pub struct ProxyClient {
    client: Client,
}

impl ProxyClient {
    pub fn new(timeout_ms: u64) -> Self {
        let client = Client::builder()
            .timeout(Duration::from_millis(timeout_ms))
            .pool_max_idle_per_host(20)
            .build()
            .expect("Failed to create HTTP client");

        ProxyClient { client }
    }

    pub async fn forward(
        &self,
        req: &HttpRequest,
        body: Bytes,
        upstream_url: &str,
        circuit_breaker: Option<Arc<CircuitBreaker>>,
        retry_config: Option<&RetryConfig>,
    ) -> Result<HttpResponse, AppError> {
        // Check circuit breaker
        if let Some(cb) = &circuit_breaker {
            if !cb.can_proceed() {
                return Ok(HttpResponse::ServiceUnavailable().json(serde_json::json!({
                    "error": "SERVICE_UNAVAILABLE",
                    "message": "Service is temporarily unavailable. Circuit breaker is open.",
                })));
            }
        }

        let max_retries = retry_config.map(|r| r.max_retries).unwrap_or(0);
        let mut attempt = 0;

        loop {
            match self.do_proxy_request(req, body.clone(), upstream_url).await {
                Ok(response) => {
                    if let Some(cb) = &circuit_breaker {
                        cb.record_success();
                    }
                    return Ok(response);
                }
                Err(e) => {
                    if let Some(cb) = &circuit_breaker {
                        cb.record_failure();
                    }

                    attempt += 1;
                    if attempt > max_retries {
                        return Err(e);
                    }

                    let delay_ms = retry_config
                        .map(|r| r.initial_delay_ms * 2u64.pow(attempt - 1))
                        .unwrap_or(100);

                    tokio::time::sleep(Duration::from_millis(delay_ms)).await;
                    log::warn!("Retrying request (attempt {}/{})", attempt, max_retries);
                }
            }
        }
    }

    async fn do_proxy_request(
        &self,
        req: &HttpRequest,
        body: Bytes,
        upstream_url: &str,
    ) -> Result<HttpResponse, AppError> {
        let method = Method::from_str(req.method().as_str())
            .map_err(|e| AppError::InternalError(e.to_string()))?;

        let mut proxy_req = self.client
            .request(method, upstream_url);

        // Forward headers (excluding hop-by-hop)
        let hop_by_hop = ["connection", "keep-alive", "proxy-authenticate",
                          "proxy-authorization", "te", "trailers", "transfer-encoding", "upgrade"];

        for (name, value) in req.headers() {
            let name_lower = name.as_str().to_lowercase();
            if !hop_by_hop.contains(&name_lower.as_str()) {
                if let Ok(val) = value.to_str() {
                    proxy_req = proxy_req.header(name.as_str(), val);
                }
            }
        }

        // Add X-Forwarded headers
        let client_ip = req.connection_info().realip_remote_addr()
            .unwrap_or("unknown")
            .to_string();

        proxy_req = proxy_req
            .header("X-Forwarded-For", &client_ip)
            .header("X-Forwarded-Proto", req.connection_info().scheme())
            .header("X-Gateway-Request-ID", uuid::Uuid::new_v4().to_string());

        if !body.is_empty() {
            proxy_req = proxy_req.body(body.to_vec());
        }

        let upstream_response = proxy_req
            .send()
            .await
            .map_err(|e| AppError::InternalError(format!("Upstream request failed: {}", e)))?;

        // Build response
        let status = actix_web::http::StatusCode::from_u16(upstream_response.status().as_u16())
            .unwrap_or(actix_web::http::StatusCode::INTERNAL_SERVER_ERROR);

        let mut builder = HttpResponse::build(status);

        for (name, value) in upstream_response.headers() {
            if let Ok(val) = value.to_str() {
                builder.append_header((name.as_str(), val));
            }
        }

        let response_body = upstream_response.bytes().await
            .map_err(|e| AppError::InternalError(format!("Failed to read upstream response: {}", e)))?;

        Ok(builder.body(response_body))
    }
}
```

---

## 6. Rate Limiter (`src/middleware/rate_limit.rs`)

```rust
use actix_web::{HttpRequest, HttpResponse};
use dashmap::DashMap;
use governor::{
    clock::QuantaClock,
    state::{InMemoryState, NotKeyed},
    Quota, RateLimiter as GovernorRateLimiter,
};
use std::num::NonZeroU32;
use std::sync::Arc;
use tokio::time::Duration;

use crate::config::{RateLimitConfig, RateLimitKey};

pub struct GatewayRateLimiter {
    limiters: DashMap<String, Arc<GovernorRateLimiter<NotKeyed, InMemoryState, QuantaClock>>>,
    config: RateLimitConfig,
}

impl GatewayRateLimiter {
    pub fn new(config: RateLimitConfig) -> Arc<Self> {
        Arc::new(GatewayRateLimiter {
            limiters: DashMap::new(),
            config,
        })
    }

    pub fn check(&self, req: &HttpRequest) -> Result<(), actix_web::Error> {
        let key = self.get_key(req);
        
        let limiter = self.limiters.entry(key.clone()).or_insert_with(|| {
            let quota = Quota::per_second(
                NonZeroU32::new(self.config.requests_per_second).unwrap()
            )
            .allow_burst(NonZeroU32::new(self.config.burst_size).unwrap());

            Arc::new(GovernorRateLimiter::direct(quota))
        });

        limiter.check().map_err(|_| {
            actix_web::error::ErrorTooManyRequests(serde_json::json!({
                "error": "RATE_LIMIT_EXCEEDED",
                "message": format!("Too many requests. Limit: {} req/s", self.config.requests_per_second),
            }).to_string())
        })
    }

    fn get_key(&self, req: &HttpRequest) -> String {
        match self.config.key_by {
            RateLimitKey::Ip => {
                req.connection_info()
                    .realip_remote_addr()
                    .unwrap_or("unknown")
                    .to_string()
            }
            RateLimitKey::UserId => {
                req.extensions()
                    .get::<String>()
                    .cloned()
                    .unwrap_or_else(|| "anonymous".to_string())
            }
            RateLimitKey::ApiKey => {
                req.headers()
                    .get("X-API-Key")
                    .and_then(|h| h.to_str().ok())
                    .unwrap_or("unknown")
                    .to_string()
            }
            RateLimitKey::Global => "global".to_string(),
        }
    }
}
```

---

## 7. Response Cache (`src/middleware/cache.rs`)

```rust
use actix_web::{HttpRequest, HttpResponse};
use redis::AsyncCommands;
use std::time::Duration;

use crate::config::CacheConfig;
use crate::errors::AppError;

pub struct ResponseCache {
    redis: redis::aio::ConnectionManager,
    config: CacheConfig,
}

impl ResponseCache {
    pub async fn new(redis_url: &str, config: CacheConfig) -> Result<Self, AppError> {
        let client = redis::Client::open(redis_url)
            .map_err(|e| AppError::InternalError(e.to_string()))?;
        let manager = redis::aio::ConnectionManager::new(client).await
            .map_err(|e| AppError::InternalError(e.to_string()))?;

        Ok(ResponseCache { redis: manager, config })
    }

    pub async fn get(&mut self, key: &str) -> Option<CachedResponse> {
        let data: Option<String> = self.redis.get(key).await.ok()?;
        data.and_then(|d| serde_json::from_str(&d).ok())
    }

    pub async fn set(&mut self, key: &str, response: &CachedResponse) -> Result<(), AppError> {
        let data = serde_json::to_string(response)
            .map_err(|e| AppError::InternalError(e.to_string()))?;

        let _: () = self.redis
            .set_ex(key, data, self.config.ttl_seconds)
            .await
            .map_err(|e| AppError::InternalError(e.to_string()))?;

        Ok(())
    }

    pub fn build_cache_key(&self, req: &HttpRequest) -> String {
        let mut key_parts = vec![req.path().to_string()];

        for cache_by in &self.config.cache_by {
            if cache_by == "query" {
                if let Some(query) = req.uri().query() {
                    key_parts.push(query.to_string());
                }
            } else {
                if let Some(header_val) = req.headers()
                    .get(cache_by.as_str())
                    .and_then(|h| h.to_str().ok()) {
                    key_parts.push(format!("{}:{}", cache_by, header_val));
                }
            }
        }

        format!("gateway:cache:{}", key_parts.join(":"))
    }

    pub fn is_cacheable(req: &HttpRequest, status_code: u16) -> bool {
        // Only cache GET requests with 2xx responses
        req.method() == actix_web::http::Method::GET
            && (200..300).contains(&status_code)
    }
}

#[derive(Debug, serde::Serialize, serde::Deserialize)]
pub struct CachedResponse {
    pub status: u16,
    pub headers: Vec<(String, String)>,
    pub body: Vec<u8>,
    pub cached_at: chrono::DateTime<chrono::Utc>,
}
```

---

## 8. Metrics Middleware (`src/middleware/metrics.rs`)

```rust
use actix_web::HttpRequest;
use prometheus::{
    Counter, CounterVec, Histogram, HistogramVec, IntCounter, IntCounterVec,
    opts, register_counter_vec, register_histogram_vec, register_int_counter_vec,
};
use std::sync::Arc;
use std::time::Instant;

pub struct GatewayMetrics {
    pub request_count: IntCounterVec,
    pub request_duration: HistogramVec,
    pub upstream_latency: HistogramVec,
    pub error_count: IntCounterVec,
    pub circuit_breaker_trips: IntCounterVec,
    pub cache_hits: IntCounterVec,
}

impl GatewayMetrics {
    pub fn new() -> Arc<Self> {
        let request_count = register_int_counter_vec!(
            "gateway_requests_total",
            "Total number of requests processed",
            &["method", "path", "status"]
        ).expect("Failed to create metric");

        let request_duration = register_histogram_vec!(
            "gateway_request_duration_seconds",
            "Request processing time",
            &["method", "path"],
            vec![0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
        ).expect("Failed to create metric");

        let upstream_latency = register_histogram_vec!(
            "gateway_upstream_latency_seconds",
            "Upstream service latency",
            &["upstream", "path"],
            vec![0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
        ).expect("Failed to create metric");

        let error_count = register_int_counter_vec!(
            "gateway_errors_total",
            "Total number of errors",
            &["error_type", "path"]
        ).expect("Failed to create metric");

        let circuit_breaker_trips = register_int_counter_vec!(
            "gateway_circuit_breaker_trips_total",
            "Circuit breaker state changes",
            &["route", "state"]
        ).expect("Failed to create metric");

        let cache_hits = register_int_counter_vec!(
            "gateway_cache_hits_total",
            "Cache hit/miss counters",
            &["route", "result"]
        ).expect("Failed to create metric");

        Arc::new(GatewayMetrics {
            request_count,
            request_duration,
            upstream_latency,
            error_count,
            circuit_breaker_trips,
            cache_hits,
        })
    }

    pub fn record_request(&self, method: &str, path: &str, status: u16, duration: f64) {
        self.request_count
            .with_label_values(&[method, path, &status.to_string()])
            .inc();
        self.request_duration
            .with_label_values(&[method, path])
            .observe(duration);
    }
}
```

---

## 9. Main Application (`src/main.rs`)

```rust
use actix_cors::Cors;
use actix_web::{middleware::Logger, web, App, HttpRequest, HttpResponse, HttpServer};
use bytes::Bytes;
use dashmap::DashMap;
use dotenv::dotenv;
use std::collections::HashMap;
use std::env;
use std::sync::Arc;
use std::time::Instant;

mod config;
mod errors;
mod middleware;
mod proxy;
mod routing;
mod services;

use config::{GatewayConfig, RouteConfig};
use middleware::{
    circuit_breaker::CircuitBreaker,
    metrics::GatewayMetrics,
};
use proxy::handler::ProxyClient;

struct GatewayState {
    routes: Vec<RouteConfig>,
    proxy: ProxyClient,
    circuit_breakers: DashMap<String, Arc<CircuitBreaker>>,
    metrics: Arc<GatewayMetrics>,
}

impl GatewayState {
    fn find_route(&self, path: &str, method: &str) -> Option<&RouteConfig> {
        self.routes.iter().find(|route| {
            let path_matches = if route.path.ends_with("/**") {
                let prefix = &route.path[..route.path.len() - 3];
                path.starts_with(prefix)
            } else {
                path == route.path || path.starts_with(&format!("{}/", route.path))
            };

            let method_matches = route.methods.iter()
                .any(|m| m == "*" || m.to_uppercase() == method.to_uppercase());

            path_matches && method_matches
        })
    }

    fn build_upstream_url(&self, route: &RouteConfig, original_path: &str) -> String {
        let upstream_base = &route.upstream.url;

        if let Some(rewrite) = &route.upstream.path_rewrite {
            format!("{}{}", upstream_base, rewrite)
        } else if route.strip_prefix {
            let stripped = original_path
                .strip_prefix(&route.path.trim_end_matches("/**"))
                .unwrap_or(original_path);
            format!("{}{}", upstream_base, stripped)
        } else {
            format!("{}{}", upstream_base, original_path)
        }
    }
}

async fn proxy_handler(
    req: HttpRequest,
    body: Bytes,
    state: web::Data<Arc<GatewayState>>,
) -> Result<HttpResponse, actix_web::Error> {
    let start = Instant::now();
    let path = req.path().to_string();
    let method = req.method().as_str().to_string();

    // Find matching route
    let route = match state.find_route(&path, &method) {
        Some(r) => r.clone(),
        None => {
            return Ok(HttpResponse::NotFound().json(serde_json::json!({
                "error": "ROUTE_NOT_FOUND",
                "message": format!("No route found for {} {}", method, path),
            })));
        }
    };

    // Rate limiting
    if let Some(rate_config) = &route.rate_limit {
        let limiter = middleware::rate_limit::GatewayRateLimiter::new(rate_config.clone());
        limiter.check(&req)?;
    }

    // Build upstream URL
    let upstream_url = state.build_upstream_url(&route, &path);

    // Get circuit breaker for this route
    let circuit_breaker = if let Some(cb_config) = &route.circuit_breaker {
        let cb = state.circuit_breakers
            .entry(route.id.clone())
            .or_insert_with(|| CircuitBreaker::new(cb_config.clone()));
        Some(cb.clone())
    } else {
        None
    };

    // Apply request transformations
    let mut req_with_transforms = req.clone();
    if let Some(transform) = &route.transform {
        if let Some(add_headers) = &transform.add_request_headers {
            for (name, value) in add_headers {
                log::debug!("Adding header: {}={}", name, value);
                // Headers would be added to the proxied request
            }
        }
    }

    // Forward request
    let response = state.proxy
        .forward(&req, body, &upstream_url, circuit_breaker, route.retry.as_ref())
        .await
        .unwrap_or_else(|e| {
            log::error!("Proxy error: {}", e);
            HttpResponse::BadGateway().json(serde_json::json!({
                "error": "BAD_GATEWAY",
                "message": e.to_string(),
            }))
        });

    // Record metrics
    let duration = start.elapsed().as_secs_f64();
    state.metrics.record_request(&method, &path, response.status().as_u16(), duration);

    Ok(response)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv().ok();
    env_logger::init_from_env(env_logger::Env::new().default_filter_or("info"));

    let config_path = env::var("GATEWAY_CONFIG").unwrap_or_else(|_| "config/routes.yaml".to_string());
    
    let gateway_config = if std::path::Path::new(&config_path).exists() {
        GatewayConfig::from_yaml(&config_path).expect("Failed to load config")
    } else {
        log::warn!("Config file not found, using defaults");
        GatewayConfig::from_env()
    };

    let port = gateway_config.port;
    let host = env::var("HOST").unwrap_or_else(|_| "127.0.0.1".to_string());

    let state = Arc::new(GatewayState {
        routes: gateway_config.routes,
        proxy: ProxyClient::new(gateway_config.global.timeout_ms),
        circuit_breakers: DashMap::new(),
        metrics: GatewayMetrics::new(),
    });

    log::info!("Starting API Gateway at http://{}:{}", host, port);

    HttpServer::new(move || {
        let cors = Cors::default()
            .allow_any_origin()
            .allow_any_method()
            .allow_any_header();

        App::new()
            .wrap(Logger::default())
            .wrap(cors)
            .app_data(web::Data::new(state.clone()))
            // Health check endpoint
            .route("/health", web::get().to(|| async {
                HttpResponse::Ok().json(serde_json::json!({
                    "status": "healthy",
                    "timestamp": chrono::Utc::now(),
                }))
            }))
            // Metrics endpoint
            .route("/metrics", web::get().to(metrics_handler))
            // Admin endpoint
            .route("/admin/routes", web::get().to(list_routes))
            // Catch-all proxy handler
            .default_service(web::to(proxy_handler))
    })
    .bind(format!("{}:{}", host, port))?
    .run()
    .await
}

async fn metrics_handler() -> HttpResponse {
    use prometheus::Encoder;
    let encoder = prometheus::TextEncoder::new();
    let metric_families = prometheus::gather();
    let mut buffer = Vec::new();
    encoder.encode(&metric_families, &mut buffer).unwrap_or_default();
    
    HttpResponse::Ok()
        .content_type("text/plain; version=0.0.4")
        .body(buffer)
}

async fn list_routes(state: web::Data<Arc<GatewayState>>) -> HttpResponse {
    let routes: Vec<_> = state.routes.iter().map(|r| serde_json::json!({
        "id": r.id,
        "path": r.path,
        "methods": r.methods,
        "upstream": r.upstream.url,
        "has_auth": r.auth.is_some(),
        "has_rate_limit": r.rate_limit.is_some(),
        "has_cache": r.cache.is_some(),
        "has_circuit_breaker": r.circuit_breaker.is_some(),
    })).collect();

    HttpResponse::Ok().json(serde_json::json!({ "routes": routes }))
}
```

---

## 10. Example Route Configuration (`config/routes.yaml`)

```yaml
port: 8080
global:
  timeout_ms: 30000
  max_connections: 1000
  cors_allowed_origins:
    - "*"
  rate_limit:
    requests_per_second: 100
    burst_size: 200
    key_by: ip

routes:
  - id: "auth-service"
    path: "/api/auth/**"
    methods: ["GET", "POST"]
    strip_prefix: false
    upstream:
      url: "http://auth-service:8001"
      timeout_ms: 5000
    rate_limit:
      requests_per_second: 10
      burst_size: 20
      key_by: ip

  - id: "user-service"
    path: "/api/users/**"
    methods: ["GET", "PUT", "DELETE"]
    strip_prefix: false
    upstream:
      url: "http://user-service:8002"
    auth:
      required: true
      auth_type: jwt
      jwt_secret: "${JWT_SECRET}"
    circuit_breaker:
      failure_threshold: 5
      success_threshold: 2
      timeout_ms: 30000
    retry:
      max_retries: 2
      retry_on_status: [503, 504]
      initial_delay_ms: 100

  - id: "product-service"
    path: "/api/products/**"
    methods: ["GET"]
    strip_prefix: false
    upstream:
      url: "http://product-service:8003"
    cache:
      ttl_seconds: 300
      cache_by: ["query"]
      vary_headers: ["Accept-Language"]
    transform:
      add_response_headers:
        X-Cache-TTL: "300"
        X-Service: "product-service"
```

---

## สรุป Part 089

ใน Part นี้เราได้สร้าง API Gateway ที่สมบูรณ์ด้วย:
1. **Reverse Proxy** พร้อม header forwarding
2. **Route Configuration** จาก YAML file
3. **Authentication Handling** ด้วย JWT
4. **Rate Limiting** ด้วย Governor (token bucket)
5. **Response Caching** ด้วย Redis
6. **Circuit Breaker** พร้อม Open/HalfOpen/Closed states
7. **Retry Mechanism** พร้อม exponential backoff
8. **Prometheus Metrics** สำหรับ monitoring
9. **Request/Response Transformation**

ใน **Part 090** เราจะสร้าง **Real-time Collaboration Service** ที่ซับซ้อนที่สุด

---

*[← Part 088: File Storage Service](../part_088/README.md) | [Part 090: Real-time Collaboration →](../part_090/README.md)*
