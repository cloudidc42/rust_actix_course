# Part 070: Health Checks and Monitoring in Rust

## ภาพรวม

Health Checks เป็นส่วนสำคัญของ production system ช่วยให้ load balancers, Kubernetes และ monitoring systems รู้ว่า service พร้อมรับ traffic หรือไม่ บทนี้จะสร้าง complete health check system ที่พร้อม deploy ใน Kubernetes

## Health Check Endpoints

```
/health/live   → Liveness probe (is the app alive?)
/health/ready  → Readiness probe (can the app serve traffic?)
/health        → Full health check (detailed status)
/metrics       → Prometheus metrics
```

## Kubernetes Probes

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

## Cargo.toml

```toml
[package]
name = "health-checks"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio"] }
redis = { version = "0.23", features = ["tokio-comp"] }
reqwest = { version = "0.11", features = ["json"] }
async-trait = "0.1"
thiserror = "1"
prometheus = { version = "0.13", features = ["process"] }
lazy_static = "1"
tracing = "0.1"
tracing-subscriber = "0.3"
```

## Health Check Types

```rust
// src/health/types.rs
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use chrono::{DateTime, Utc};

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum HealthStatus {
    Healthy,
    Degraded,
    Unhealthy,
}

impl HealthStatus {
    pub fn is_ok(&self) -> bool {
        matches!(self, HealthStatus::Healthy | HealthStatus::Degraded)
    }
    
    /// HTTP status code สำหรับ response
    pub fn http_status(&self) -> u16 {
        match self {
            HealthStatus::Healthy => 200,
            HealthStatus::Degraded => 200, // degraded ยังให้ traffic
            HealthStatus::Unhealthy => 503,
        }
    }
}

impl std::fmt::Display for HealthStatus {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            HealthStatus::Healthy => write!(f, "healthy"),
            HealthStatus::Degraded => write!(f, "degraded"),
            HealthStatus::Unhealthy => write!(f, "unhealthy"),
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ComponentHealth {
    pub status: HealthStatus,
    pub message: Option<String>,
    pub latency_ms: Option<u64>,
    pub details: Option<serde_json::Value>,
    pub last_checked: DateTime<Utc>,
}

impl ComponentHealth {
    pub fn healthy() -> Self {
        ComponentHealth {
            status: HealthStatus::Healthy,
            message: None,
            latency_ms: None,
            details: None,
            last_checked: Utc::now(),
        }
    }
    
    pub fn healthy_with_details(details: serde_json::Value) -> Self {
        ComponentHealth {
            status: HealthStatus::Healthy,
            message: None,
            latency_ms: None,
            details: Some(details),
            last_checked: Utc::now(),
        }
    }
    
    pub fn unhealthy(message: impl Into<String>) -> Self {
        ComponentHealth {
            status: HealthStatus::Unhealthy,
            message: Some(message.into()),
            latency_ms: None,
            details: None,
            last_checked: Utc::now(),
        }
    }
    
    pub fn degraded(message: impl Into<String>) -> Self {
        ComponentHealth {
            status: HealthStatus::Degraded,
            message: Some(message.into()),
            latency_ms: None,
            details: None,
            last_checked: Utc::now(),
        }
    }
    
    pub fn with_latency(mut self, latency_ms: u64) -> Self {
        self.latency_ms = Some(latency_ms);
        self
    }
}

#[derive(Debug, Serialize, Deserialize)]
pub struct HealthResponse {
    pub status: HealthStatus,
    pub version: String,
    pub uptime_seconds: u64,
    pub checks: HashMap<String, ComponentHealth>,
    pub timestamp: DateTime<Utc>,
    pub instance_id: String,
}

impl HealthResponse {
    pub fn http_status(&self) -> actix_web::http::StatusCode {
        let code = self.status.http_status();
        actix_web::http::StatusCode::from_u16(code).unwrap_or(actix_web::http::StatusCode::OK)
    }
}
```

## Health Check Traits

```rust
// src/health/checks/mod.rs
use async_trait::async_trait;
use std::time::Instant;

use crate::health::types::ComponentHealth;

#[async_trait]
pub trait HealthCheck: Send + Sync {
    fn name(&self) -> &str;
    fn is_critical(&self) -> bool { true }
    async fn check(&self) -> ComponentHealth;
    
    /// Helper: run check กับ timing
    async fn run_with_timing(&self) -> ComponentHealth {
        let start = Instant::now();
        let mut result = self.check().await;
        result.latency_ms = Some(start.elapsed().as_millis() as u64);
        result
    }
}

pub mod database;
pub mod redis_check;
pub mod external_service;
pub mod system;
pub mod custom;
```

## Database Health Check

```rust
// src/health/checks/database.rs
use async_trait::async_trait;
use sqlx::PgPool;
use std::time::Instant;

use crate::health::{HealthCheck, types::ComponentHealth};

pub struct DatabaseHealthCheck {
    pool: PgPool,
    name: String,
    slow_threshold_ms: u64,
}

impl DatabaseHealthCheck {
    pub fn new(pool: PgPool) -> Self {
        DatabaseHealthCheck {
            pool,
            name: "database".to_string(),
            slow_threshold_ms: 100,
        }
    }
    
    pub fn with_name(mut self, name: impl Into<String>) -> Self {
        self.name = name.into();
        self
    }
}

#[async_trait]
impl HealthCheck for DatabaseHealthCheck {
    fn name(&self) -> &str {
        &self.name
    }
    
    fn is_critical(&self) -> bool {
        true // DB down = service down
    }
    
    async fn check(&self) -> ComponentHealth {
        let start = Instant::now();
        
        match sqlx::query("SELECT 1 as health_check")
            .execute(&self.pool)
            .await
        {
            Ok(_) => {
                let latency = start.elapsed().as_millis() as u64;
                let pool_info = serde_json::json!({
                    "pool_size": self.pool.size(),
                    "idle_connections": self.pool.num_idle(),
                });
                
                if latency > self.slow_threshold_ms {
                    ComponentHealth::degraded(
                        format!("Database slow: {}ms", latency)
                    ).with_latency(latency)
                } else {
                    ComponentHealth {
                        ..ComponentHealth::healthy_with_details(pool_info)
                    }.with_latency(latency)
                }
            }
            Err(e) => {
                tracing::error!("Database health check failed: {}", e);
                ComponentHealth::unhealthy(format!("Database error: {}", e))
            }
        }
    }
}
```

## Redis Health Check

```rust
// src/health/checks/redis_check.rs
use async_trait::async_trait;
use redis::AsyncCommands;
use std::time::Instant;

use crate::health::{HealthCheck, types::ComponentHealth};

pub struct RedisHealthCheck {
    client: redis::Client,
    name: String,
}

impl RedisHealthCheck {
    pub fn new(redis_url: &str) -> Result<Self, String> {
        let client = redis::Client::open(redis_url)
            .map_err(|e| e.to_string())?;
        
        Ok(RedisHealthCheck {
            client,
            name: "redis".to_string(),
        })
    }
}

#[async_trait]
impl HealthCheck for RedisHealthCheck {
    fn name(&self) -> &str {
        &self.name
    }
    
    fn is_critical(&self) -> bool {
        false // Redis optional - degraded mode ได้
    }
    
    async fn check(&self) -> ComponentHealth {
        let start = Instant::now();
        
        match redis::aio::ConnectionManager::new(self.client.clone()).await {
            Ok(mut conn) => {
                let ping_result: Result<String, redis::RedisError> = redis::cmd("PING")
                    .query_async(&mut conn)
                    .await;
                
                match ping_result {
                    Ok(response) if response == "PONG" => {
                        let latency = start.elapsed().as_millis() as u64;
                        
                        // ดึง Redis info
                        let info: Result<String, _> = redis::cmd("INFO")
                            .arg("memory")
                            .query_async(&mut conn)
                            .await;
                        
                        let details = info.ok().map(|info_str| {
                            // Parse memory info
                            let used_memory = info_str.lines()
                                .find(|l| l.starts_with("used_memory_human:"))
                                .and_then(|l| l.split(':').nth(1))
                                .unwrap_or("unknown")
                                .trim()
                                .to_string();
                            
                            serde_json::json!({
                                "used_memory": used_memory,
                                "ping_response": response,
                            })
                        });
                        
                        ComponentHealth {
                            details,
                            ..ComponentHealth::healthy()
                        }.with_latency(latency)
                    }
                    Ok(unexpected) => {
                        ComponentHealth::degraded(format!("Unexpected PING response: {}", unexpected))
                    }
                    Err(e) => {
                        ComponentHealth::unhealthy(format!("Redis PING failed: {}", e))
                    }
                }
            }
            Err(e) => {
                ComponentHealth::unhealthy(format!("Redis connection failed: {}", e))
            }
        }
    }
}
```

## External Service Health Check

```rust
// src/health/checks/external_service.rs
use async_trait::async_trait;
use reqwest::Client;
use std::time::{Duration, Instant};

use crate::health::{HealthCheck, types::ComponentHealth};

pub struct ExternalServiceCheck {
    name: String,
    url: String,
    client: Client,
    timeout: Duration,
    critical: bool,
}

impl ExternalServiceCheck {
    pub fn new(name: impl Into<String>, url: impl Into<String>) -> Self {
        let client = Client::builder()
            .timeout(Duration::from_secs(5))
            .build()
            .unwrap();
        
        ExternalServiceCheck {
            name: name.into(),
            url: url.into(),
            client,
            timeout: Duration::from_secs(5),
            critical: false,
        }
    }
    
    pub fn critical(mut self) -> Self {
        self.critical = true;
        self
    }
    
    pub fn with_timeout(mut self, timeout: Duration) -> Self {
        self.timeout = timeout;
        self
    }
}

#[async_trait]
impl HealthCheck for ExternalServiceCheck {
    fn name(&self) -> &str {
        &self.name
    }
    
    fn is_critical(&self) -> bool {
        self.critical
    }
    
    async fn check(&self) -> ComponentHealth {
        let start = Instant::now();
        
        match self.client.get(&self.url)
            .timeout(self.timeout)
            .send()
            .await
        {
            Ok(response) => {
                let latency = start.elapsed().as_millis() as u64;
                let status = response.status();
                
                if status.is_success() {
                    ComponentHealth::healthy_with_details(serde_json::json!({
                        "url": self.url,
                        "status_code": status.as_u16(),
                    })).with_latency(latency)
                } else if status.as_u16() >= 500 {
                    ComponentHealth::unhealthy(
                        format!("Service returned {}", status)
                    ).with_latency(latency)
                } else {
                    ComponentHealth::degraded(
                        format!("Service returned {}", status)
                    ).with_latency(latency)
                }
            }
            Err(e) if e.is_timeout() => {
                let latency = start.elapsed().as_millis() as u64;
                ComponentHealth::unhealthy("Request timeout").with_latency(latency)
            }
            Err(e) => {
                ComponentHealth::unhealthy(format!("Connection failed: {}", e))
            }
        }
    }
}
```

## System Health Check

```rust
// src/health/checks/system.rs
use async_trait::async_trait;

use crate::health::{HealthCheck, types::ComponentHealth};

pub struct SystemHealthCheck {
    max_memory_pct: f64,
    max_cpu_pct: f64,
}

impl Default for SystemHealthCheck {
    fn default() -> Self {
        SystemHealthCheck {
            max_memory_pct: 90.0,
            max_cpu_pct: 95.0,
        }
    }
}

#[async_trait]
impl HealthCheck for SystemHealthCheck {
    fn name(&self) -> &str {
        "system"
    }
    
    fn is_critical(&self) -> bool {
        false
    }
    
    async fn check(&self) -> ComponentHealth {
        // ใน production ใช้ sys-info หรือ procfs crate
        // สำหรับ example นี้ใช้ค่า mock
        
        let memory_usage = get_memory_usage_pct();
        let cpu_usage = get_cpu_usage_pct();
        
        let details = serde_json::json!({
            "memory_usage_pct": memory_usage,
            "cpu_usage_pct": cpu_usage,
            "pid": std::process::id(),
        });
        
        if memory_usage > self.max_memory_pct || cpu_usage > self.max_cpu_pct {
            ComponentHealth {
                details: Some(details),
                ..ComponentHealth::degraded(format!(
                    "High resource usage: mem={:.1}%, cpu={:.1}%",
                    memory_usage, cpu_usage
                ))
            }
        } else {
            ComponentHealth::healthy_with_details(details)
        }
    }
}

// Simulated system metrics (ใช้ sys-info crate ใน production)
fn get_memory_usage_pct() -> f64 { 45.0 }
fn get_cpu_usage_pct() -> f64 { 30.0 }
```

## Health Checker - Orchestrator

```rust
// src/health/checker.rs
use std::collections::HashMap;
use std::sync::Arc;
use std::time::Instant;
use uuid::Uuid;

use crate::health::{
    checks::HealthCheck,
    types::{ComponentHealth, HealthResponse, HealthStatus},
};

pub struct HealthChecker {
    checks: Vec<Arc<dyn HealthCheck>>,
    start_time: Instant,
    instance_id: String,
}

impl HealthChecker {
    pub fn new() -> Self {
        HealthChecker {
            checks: Vec::new(),
            start_time: Instant::now(),
            instance_id: Uuid::new_v4().to_string(),
        }
    }
    
    pub fn add_check(mut self, check: Arc<dyn HealthCheck>) -> Self {
        self.checks.push(check);
        self
    }
    
    /// Full health check (ใช้ใน /health)
    pub async fn check_all(&self) -> HealthResponse {
        let mut checks = HashMap::new();
        let mut overall_status = HealthStatus::Healthy;
        
        // Run ทุก checks พร้อมกัน
        let futures: Vec<_> = self.checks.iter().map(|check| {
            let check = Arc::clone(check);
            async move {
                let result = check.run_with_timing().await;
                (check.name().to_string(), check.is_critical(), result)
            }
        }).collect();
        
        let results = futures_util::future::join_all(futures).await;
        
        for (name, is_critical, result) in results {
            match &result.status {
                HealthStatus::Unhealthy if is_critical => {
                    overall_status = HealthStatus::Unhealthy;
                }
                HealthStatus::Degraded if overall_status == HealthStatus::Healthy => {
                    overall_status = HealthStatus::Degraded;
                }
                _ => {}
            }
            
            checks.insert(name, result);
        }
        
        HealthResponse {
            status: overall_status,
            version: env!("CARGO_PKG_VERSION").to_string(),
            uptime_seconds: self.start_time.elapsed().as_secs(),
            checks,
            timestamp: chrono::Utc::now(),
            instance_id: self.instance_id.clone(),
        }
    }
    
    /// Liveness check (simple - ใช้ใน /health/live)
    /// ตรวจแค่ว่า process ยัง alive ไหม (ไม่ check dependencies)
    pub async fn liveness_check(&self) -> bool {
        true // ถ้า process ตอบได้ = alive
    }
    
    /// Readiness check (ใช้ใน /health/ready)
    /// ตรวจว่า service พร้อมรับ traffic หรือไม่
    pub async fn readiness_check(&self) -> bool {
        // Check เฉพาะ critical dependencies
        let critical_checks: Vec<_> = self.checks.iter()
            .filter(|c| c.is_critical())
            .collect();
        
        if critical_checks.is_empty() {
            return true;
        }
        
        let futures: Vec<_> = critical_checks.iter().map(|check| {
            let check: Arc<dyn HealthCheck> = Arc::clone(check);
            async move {
                let result = check.check().await;
                result.status.is_ok()
            }
        }).collect();
        
        let results = futures_util::future::join_all(futures).await;
        results.iter().all(|&ok| ok)
    }
}

impl Default for HealthChecker {
    fn default() -> Self {
        Self::new()
    }
}
```

## HTTP Handlers

```rust
// src/interface/http/health_handler.rs
use actix_web::{web, HttpResponse};
use std::sync::Arc;

use crate::health::{
    checker::HealthChecker,
    types::HealthStatus,
};

pub async fn health_live(
    checker: web::Data<Arc<HealthChecker>>,
) -> HttpResponse {
    if checker.liveness_check().await {
        HttpResponse::Ok().json(serde_json::json!({
            "status": "alive",
            "timestamp": chrono::Utc::now()
        }))
    } else {
        HttpResponse::InternalServerError().json(serde_json::json!({
            "status": "dead"
        }))
    }
}

pub async fn health_ready(
    checker: web::Data<Arc<HealthChecker>>,
) -> HttpResponse {
    if checker.readiness_check().await {
        HttpResponse::Ok().json(serde_json::json!({
            "status": "ready",
            "timestamp": chrono::Utc::now()
        }))
    } else {
        HttpResponse::ServiceUnavailable().json(serde_json::json!({
            "status": "not_ready",
            "message": "One or more critical dependencies are unavailable"
        }))
    }
}

pub async fn health_full(
    checker: web::Data<Arc<HealthChecker>>,
) -> HttpResponse {
    let health_report = checker.check_all().await;
    let http_status = health_report.http_status();
    
    HttpResponse::build(http_status).json(&health_report)
}

pub fn health_routes(cfg: &mut web::ServiceConfig) {
    cfg.service(
        web::scope("/health")
            .route("/live", web::get().to(health_live))
            .route("/ready", web::get().to(health_ready))
            .route("", web::get().to(health_full))
    );
}
```

## Metrics Endpoint

```rust
// src/interface/http/metrics_handler.rs
use actix_web::HttpResponse;
use prometheus::{TextEncoder, Encoder, register_counter_vec, register_histogram_vec, opts};
use lazy_static::lazy_static;

lazy_static! {
    pub static ref REQUEST_COUNTER: prometheus::CounterVec = register_counter_vec!(
        opts!("http_requests_total", "Total HTTP requests"),
        &["method", "path", "status_code"]
    ).unwrap();
    
    pub static ref REQUEST_DURATION: prometheus::HistogramVec = register_histogram_vec!(
        "http_request_duration_seconds",
        "Request duration",
        &["method", "path"],
        vec![0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
    ).unwrap();
    
    pub static ref DB_CONNECTIONS: prometheus::GaugeVec = prometheus::register_gauge_vec!(
        opts!("db_pool_connections", "Database connection pool stats"),
        &["state"]
    ).unwrap();
}

pub async fn metrics() -> HttpResponse {
    let encoder = TextEncoder::new();
    let mut buffer = Vec::new();
    
    let metric_families = prometheus::gather();
    encoder.encode(&metric_families, &mut buffer).unwrap();
    
    HttpResponse::Ok()
        .content_type("text/plain; version=0.0.4; charset=utf-8")
        .body(buffer)
}
```

## Complete Main

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware};
use sqlx::PgPool;
use std::sync::Arc;
use std::time::Duration;

mod health;
mod interface;

use health::{
    checker::HealthChecker,
    checks::{
        database::DatabaseHealthCheck,
        redis_check::RedisHealthCheck,
        external_service::ExternalServiceCheck,
        system::SystemHealthCheck,
    },
};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    tracing_subscriber::fmt::init();
    
    // Database
    let database_url = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "postgres://postgres:password@localhost/myapp".to_string());
    let pool = PgPool::connect(&database_url)
        .await
        .expect("Failed to connect to database");
    
    // Build health checker
    let health_checker = Arc::new(
        HealthChecker::new()
            // Critical checks
            .add_check(Arc::new(DatabaseHealthCheck::new(pool.clone())))
            // Non-critical checks
            .add_check(Arc::new(
                RedisHealthCheck::new("redis://localhost:6379")
                    .expect("Failed to create Redis client")
            ))
            .add_check(Arc::new(
                ExternalServiceCheck::new(
                    "payment-service",
                    "http://payment-service:8080/health"
                )
                .with_timeout(Duration::from_secs(3))
            ))
            .add_check(Arc::new(SystemHealthCheck::default()))
    );
    
    tracing::info!("Starting server on 0.0.0.0:8080");
    
    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(Arc::clone(&health_checker)))
            .wrap(middleware::Logger::default())
            // Health endpoints
            .configure(interface::http::health_handler::health_routes)
            // Metrics endpoint
            .route("/metrics", web::get().to(interface::http::metrics_handler::metrics))
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

## Kubernetes Deployment

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rust-api
  labels:
    app: rust-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rust-api
  template:
    metadata:
      labels:
        app: rust-api
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: rust-api
        image: my-registry/rust-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        - name: APP_ENV
          value: "production"
        
        # Liveness probe
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Readiness probe
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
          successThreshold: 1
        
        # Startup probe (สำหรับ slow startup)
        startupProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 2
          timeoutSeconds: 5
          failureThreshold: 30  # 30 * 2s = 60s max startup
        
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

```yaml
# kubernetes/prometheus-serviceMonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: rust-api-monitor
  labels:
    app: rust-api
spec:
  selector:
    matchLabels:
      app: rust-api
  endpoints:
  - port: http
    path: /metrics
    interval: 15s
    scrapeTimeout: 10s
```

## Alerting Rules

```yaml
# prometheus/alerts.yaml
groups:
- name: rust-api
  rules:
  
  # Service down
  - alert: ServiceDown
    expr: up{job="rust-api"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Rust API service is down"
      description: "Instance {{ $labels.instance }} is down"
  
  # High error rate
  - alert: HighErrorRate
    expr: |
      rate(http_requests_total{status_code=~"5.."}[5m])
      / rate(http_requests_total[5m]) > 0.05
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High error rate detected"
      description: "Error rate is {{ $value | humanizePercentage }}"
  
  # Slow response time
  - alert: SlowResponseTime
    expr: |
      histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1.0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Slow response time"
      description: "P95 latency is {{ $value }}s"
  
  # Health check failing
  - alert: HealthCheckFailing
    expr: |
      http_requests_total{path="/health/ready", status_code="503"} > 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Health check is failing"
```

## Scheduled Health Check

```rust
// src/health/scheduler.rs
use std::sync::Arc;
use std::time::Duration;
use tokio::time::interval;

use crate::health::checker::HealthChecker;

/// Background task ที่ run health checks เป็นระยะ
/// และ publish metrics
pub struct HealthScheduler {
    checker: Arc<HealthChecker>,
    interval: Duration,
}

impl HealthScheduler {
    pub fn new(checker: Arc<HealthChecker>, interval: Duration) -> Self {
        HealthScheduler { checker, interval }
    }
    
    pub async fn run(self) {
        let mut ticker = interval(self.interval);
        
        loop {
            ticker.tick().await;
            
            let report = self.checker.check_all().await;
            
            // Update Prometheus gauges
            for (name, check) in &report.checks {
                let value = match check.status {
                    crate::health::types::HealthStatus::Healthy => 1.0,
                    crate::health::types::HealthStatus::Degraded => 0.5,
                    crate::health::types::HealthStatus::Unhealthy => 0.0,
                };
                
                // health_check_status{check="database"} 1.0
                // (ใน production ใช้ Prometheus gauge)
                tracing::info!(
                    check = %name,
                    status = %check.status,
                    latency_ms = ?check.latency_ms,
                    "Health check result"
                );
            }
            
            match report.status {
                crate::health::types::HealthStatus::Unhealthy => {
                    tracing::error!("Overall health: UNHEALTHY");
                }
                crate::health::types::HealthStatus::Degraded => {
                    tracing::warn!("Overall health: DEGRADED");
                }
                crate::health::types::HealthStatus::Healthy => {
                    tracing::debug!("Overall health: HEALTHY");
                }
            }
        }
    }
}
```

## Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use actix_web::test;
    use std::sync::Arc;
    
    struct AlwaysHealthy;
    
    #[async_trait::async_trait]
    impl HealthCheck for AlwaysHealthy {
        fn name(&self) -> &str { "always_healthy" }
        async fn check(&self) -> ComponentHealth {
            ComponentHealth::healthy()
        }
    }
    
    struct AlwaysUnhealthy;
    
    #[async_trait::async_trait]
    impl HealthCheck for AlwaysUnhealthy {
        fn name(&self) -> &str { "always_unhealthy" }
        fn is_critical(&self) -> bool { true }
        async fn check(&self) -> ComponentHealth {
            ComponentHealth::unhealthy("Always fails")
        }
    }
    
    struct AlwaysDegraded;
    
    #[async_trait::async_trait]
    impl HealthCheck for AlwaysDegraded {
        fn name(&self) -> &str { "always_degraded" }
        fn is_critical(&self) -> bool { false }
        async fn check(&self) -> ComponentHealth {
            ComponentHealth::degraded("Performance degraded")
        }
    }
    
    #[tokio::test]
    async fn test_all_healthy() {
        let checker = HealthChecker::new()
            .add_check(Arc::new(AlwaysHealthy));
        
        let report = checker.check_all().await;
        assert_eq!(report.status, HealthStatus::Healthy);
        assert!(report.checks.contains_key("always_healthy"));
    }
    
    #[tokio::test]
    async fn test_critical_unhealthy() {
        let checker = HealthChecker::new()
            .add_check(Arc::new(AlwaysHealthy))
            .add_check(Arc::new(AlwaysUnhealthy));
        
        let report = checker.check_all().await;
        assert_eq!(report.status, HealthStatus::Unhealthy);
    }
    
    #[tokio::test]
    async fn test_non_critical_unhealthy_is_degraded() {
        struct NonCriticalUnhealthy;
        
        #[async_trait::async_trait]
        impl HealthCheck for NonCriticalUnhealthy {
            fn name(&self) -> &str { "non_critical" }
            fn is_critical(&self) -> bool { false }
            async fn check(&self) -> ComponentHealth {
                ComponentHealth::unhealthy("Not critical")
            }
        }
        
        let checker = HealthChecker::new()
            .add_check(Arc::new(AlwaysHealthy))
            .add_check(Arc::new(NonCriticalUnhealthy));
        
        let report = checker.check_all().await;
        // Non-critical unhealthy => degraded, not unhealthy
        assert_ne!(report.status, HealthStatus::Unhealthy);
    }
    
    #[tokio::test]
    async fn test_degraded_status() {
        let checker = HealthChecker::new()
            .add_check(Arc::new(AlwaysHealthy))
            .add_check(Arc::new(AlwaysDegraded));
        
        let report = checker.check_all().await;
        assert_eq!(report.status, HealthStatus::Degraded);
    }
    
    #[tokio::test]
    async fn test_readiness_with_critical_unhealthy() {
        let checker = HealthChecker::new()
            .add_check(Arc::new(AlwaysUnhealthy));
        
        assert!(!checker.readiness_check().await);
    }
    
    #[tokio::test]
    async fn test_liveness_always_true() {
        let checker = HealthChecker::new()
            .add_check(Arc::new(AlwaysUnhealthy));
        
        assert!(checker.liveness_check().await);
    }
    
    #[actix_web::test]
    async fn test_health_live_endpoint() {
        let checker = Arc::new(HealthChecker::new());
        
        let app = test::init_service(
            App::new()
                .app_data(web::Data::new(Arc::clone(&checker)))
                .configure(health_routes)
        ).await;
        
        let req = test::TestRequest::get().uri("/health/live").to_request();
        let resp = test::call_service(&app, req).await;
        
        assert_eq!(resp.status(), actix_web::http::StatusCode::OK);
    }
    
    #[actix_web::test]
    async fn test_health_ready_endpoint_healthy() {
        let checker = Arc::new(
            HealthChecker::new().add_check(Arc::new(AlwaysHealthy))
        );
        
        let app = test::init_service(
            App::new()
                .app_data(web::Data::new(Arc::clone(&checker)))
                .configure(health_routes)
        ).await;
        
        let req = test::TestRequest::get().uri("/health/ready").to_request();
        let resp = test::call_service(&app, req).await;
        
        assert_eq!(resp.status(), actix_web::http::StatusCode::OK);
    }
    
    #[actix_web::test]
    async fn test_health_ready_endpoint_unhealthy() {
        let checker = Arc::new(
            HealthChecker::new().add_check(Arc::new(AlwaysUnhealthy))
        );
        
        let app = test::init_service(
            App::new()
                .app_data(web::Data::new(Arc::clone(&checker)))
                .configure(health_routes)
        ).await;
        
        let req = test::TestRequest::get().uri("/health/ready").to_request();
        let resp = test::call_service(&app, req).await;
        
        assert_eq!(resp.status(), actix_web::http::StatusCode::SERVICE_UNAVAILABLE);
    }
    
    #[test]
    fn test_health_status_http_codes() {
        assert_eq!(HealthStatus::Healthy.http_status(), 200);
        assert_eq!(HealthStatus::Degraded.http_status(), 200);
        assert_eq!(HealthStatus::Unhealthy.http_status(), 503);
    }
    
    #[test]
    fn test_component_health_builder() {
        let h = ComponentHealth::healthy();
        assert_eq!(h.status, HealthStatus::Healthy);
        assert!(h.message.is_none());
        
        let u = ComponentHealth::unhealthy("DB down").with_latency(100);
        assert_eq!(u.status, HealthStatus::Unhealthy);
        assert_eq!(u.message.as_deref(), Some("DB down"));
        assert_eq!(u.latency_ms, Some(100));
    }
}
```

## สรุป

Health Check System ที่สมบูรณ์ประกอบด้วย:

```
Kubernetes probes:
├── /health/live  → Liveness (ทำงานอยู่ไหม)
├── /health/ready → Readiness (พร้อมรับ traffic ไหม)
└── /health       → Full status (รายละเอียดทุก component)

Monitoring:
├── /metrics      → Prometheus format
├── Grafana       → Dashboards
└── AlertManager  → Notifications

Health Check Components:
├── Database      → Critical
├── Redis         → Non-critical
├── External APIs → Non-critical
└── System        → Non-critical (CPU/Memory)
```

---

## Navigation

- [← Part 069: Logging and Observability](../part_069/README.md)
- [→ Part 071: Testing Strategies](../part_071/README.md)
- [กลับหน้าหลัก](../../README.md)
