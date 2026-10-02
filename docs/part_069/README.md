# Part 069: Logging and Observability in Rust

## ภาพรวม

Observability คือความสามารถในการเข้าใจสถานะภายในของ system จากสิ่งที่ observe ได้จากภายนอก ประกอบด้วย 3 เสาหลัก: Logs, Metrics และ Traces บทนี้จะสร้าง observability stack ที่สมบูรณ์ด้วย Rust

## Three Pillars of Observability

```
┌─────────────────────────────────────────┐
│           Observability                 │
├─────────────┬───────────────┬───────────┤
│    Logs     │    Metrics    │  Traces   │
│             │               │           │
│ "What       │ "How much/    │ "How did  │
│  happened"  │  How many"    │  it flow" │
│             │               │           │
│ tracing     │ prometheus    │ opentel   │
│ crate       │ metrics       │emetry     │
└─────────────┴───────────────┴───────────┘
```

## Cargo.toml

```toml
[package]
name = "observability-example"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
actix-web-prom = "0.7"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }

# Tracing
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json", "fmt"] }
tracing-actix-web = "0.7"
tracing-bunyan-formatter = "0.3"

# OpenTelemetry
opentelemetry = { version = "0.21", features = ["trace"] }
opentelemetry-otlp = { version = "0.14", features = ["grpc-tonic"] }
tracing-opentelemetry = "0.22"

# Metrics
prometheus = { version = "0.13", features = ["process"] }
lazy_static = "1"
```

## Structured Logging Setup

```rust
// src/telemetry/logging.rs
use tracing_subscriber::{
    EnvFilter, Layer, Registry,
    fmt::{self, format::FmtSpan},
    layer::SubscriberExt,
    util::SubscriberInitExt,
};
use tracing_bunyan_formatter::{BunyanFormattingLayer, JsonStorageLayer};

pub struct LogConfig {
    pub level: String,
    pub format: LogFormat,
    pub service_name: String,
    pub service_version: String,
}

pub enum LogFormat {
    Pretty,     // สำหรับ development
    Json,       // สำหรับ production
    Bunyan,     // Bunyan format (structured)
}

pub fn init_tracing(config: &LogConfig) {
    let env_filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new(&config.level));
    
    match config.format {
        LogFormat::Pretty => {
            tracing_subscriber::registry()
                .with(env_filter)
                .with(fmt::layer().pretty().with_span_events(FmtSpan::CLOSE))
                .init();
        }
        LogFormat::Json => {
            tracing_subscriber::registry()
                .with(env_filter)
                .with(fmt::layer().json())
                .init();
        }
        LogFormat::Bunyan => {
            let formatting_layer = BunyanFormattingLayer::new(
                config.service_name.clone(),
                std::io::stdout,
            );
            
            tracing_subscriber::registry()
                .with(env_filter)
                .with(JsonStorageLayer)
                .with(formatting_layer)
                .init();
        }
    }
}
```

## Structured Logging กับ tracing

```rust
// src/telemetry/structured_logging.rs
use tracing::{info, warn, error, debug, instrument, Span};
use uuid::Uuid;
use std::time::Duration;

/// ใช้ #[instrument] สำหรับ automatic span + logging
#[instrument(
    name = "create_user",
    skip(password),  // ไม่ log password
    fields(
        user.id = tracing::field::Empty,
        user.email = %email,
    )
)]
pub async fn create_user_with_tracing(
    email: String,
    password: String,
) -> Result<Uuid, String> {
    let span = Span::current();
    
    tracing::info!("Creating new user");
    
    // Simulate work
    tokio::time::sleep(Duration::from_millis(10)).await;
    
    let user_id = Uuid::new_v4();
    
    // เพิ่ม field ลงใน span หลัง generate ID
    span.record("user.id", &user_id.to_string().as_str());
    
    tracing::info!(user_id = %user_id, "User created successfully");
    
    Ok(user_id)
}

/// Nested spans สำหรับ tracing call chain
#[instrument(name = "process_order")]
pub async fn process_order(order_id: Uuid) -> Result<(), String> {
    info!(order_id = %order_id, "Processing order");
    
    // Validate order (nested span)
    validate_order(order_id).await?;
    
    // Charge payment (nested span)
    charge_payment(order_id).await?;
    
    // Send notification (nested span)  
    send_notification(order_id).await?;
    
    info!(order_id = %order_id, "Order processed successfully");
    Ok(())
}

#[instrument(name = "validate_order", skip_all)]
async fn validate_order(order_id: Uuid) -> Result<(), String> {
    debug!("Validating order {}", order_id);
    tokio::time::sleep(Duration::from_millis(5)).await;
    Ok(())
}

#[instrument(name = "charge_payment", skip_all)]
async fn charge_payment(order_id: Uuid) -> Result<(), String> {
    debug!("Charging payment for order {}", order_id);
    tokio::time::sleep(Duration::from_millis(50)).await;
    Ok(())
}

#[instrument(name = "send_notification", skip_all)]
async fn send_notification(order_id: Uuid) -> Result<(), String> {
    debug!("Sending notification for order {}", order_id);
    tokio::time::sleep(Duration::from_millis(20)).await;
    Ok(())
}

/// Log levels ใช้งาน
pub fn demonstrate_log_levels() {
    // Error: ปัญหาที่ต้องแก้ทันที
    error!(
        error.code = "DB_CONNECTION_FAILED",
        error.message = "Cannot connect to database",
        "Critical: Database connection failed"
    );
    
    // Warn: สิ่งที่ควรสังเกต
    warn!(
        cache.key = "user:123",
        "Cache miss - falling back to database"
    );
    
    // Info: ข้อมูลทั่วไป
    info!(
        request.id = %Uuid::new_v4(),
        request.path = "/api/users",
        request.method = "GET",
        "Incoming HTTP request"
    );
    
    // Debug: ข้อมูลสำหรับ debugging
    debug!(
        query = "SELECT * FROM users WHERE id = $1",
        params = ?vec!["123"],
        "Executing database query"
    );
}
```

## Correlation IDs

```rust
// src/telemetry/correlation.rs
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpMessage,
};
use futures_util::future::LocalBoxFuture;
use std::rc::Rc;
use uuid::Uuid;

/// Correlation ID middleware
pub struct CorrelationIdMiddleware;

impl<S, B> Transform<S, ServiceRequest> for CorrelationIdMiddleware
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = CorrelationIdMiddlewareService<S>;
    type Future = std::future::Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        std::future::ready(Ok(CorrelationIdMiddlewareService {
            service: Rc::new(service),
        }))
    }
}

pub struct CorrelationIdMiddlewareService<S> {
    service: Rc<S>,
}

impl<S, B> Service<ServiceRequest> for CorrelationIdMiddlewareService<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    forward_ready!(service);

    fn call(&self, req: ServiceRequest) -> Self::Future {
        // ดึง correlation ID จาก header หรือสร้างใหม่
        let correlation_id = req
            .headers()
            .get("X-Correlation-ID")
            .and_then(|v| v.to_str().ok())
            .map(|s| s.to_string())
            .unwrap_or_else(|| Uuid::new_v4().to_string());
        
        let request_id = Uuid::new_v4().to_string();
        
        // เก็บ IDs ใน request extensions
        req.extensions_mut().insert(CorrelationIds {
            correlation_id: correlation_id.clone(),
            request_id: request_id.clone(),
        });
        
        let service = Rc::clone(&self.service);
        
        Box::pin(async move {
            // สร้าง span ที่มี correlation ID
            let span = tracing::info_span!(
                "http_request",
                correlation_id = %correlation_id,
                request_id = %request_id,
                method = %req.method(),
                path = %req.path(),
            );
            
            let _guard = span.enter();
            
            let mut res = service.call(req).await?;
            
            // เพิ่ม correlation ID ใน response header
            res.headers_mut().insert(
                actix_web::http::header::HeaderName::from_static("x-correlation-id"),
                actix_web::http::header::HeaderValue::from_str(&correlation_id).unwrap(),
            );
            res.headers_mut().insert(
                actix_web::http::header::HeaderName::from_static("x-request-id"),
                actix_web::http::header::HeaderValue::from_str(&request_id).unwrap(),
            );
            
            Ok(res)
        })
    }
}

#[derive(Clone, Debug)]
pub struct CorrelationIds {
    pub correlation_id: String,
    pub request_id: String,
}
```

## Prometheus Metrics

```rust
// src/telemetry/metrics.rs
use actix_web::{web, HttpResponse};
use prometheus::{
    Counter, CounterVec, Gauge, GaugeVec, Histogram, HistogramVec,
    opts, register_counter_vec, register_gauge, register_histogram_vec,
    TextEncoder, Encoder,
};
use lazy_static::lazy_static;

lazy_static! {
    // HTTP Metrics
    pub static ref HTTP_REQUESTS_TOTAL: CounterVec = register_counter_vec!(
        opts!("http_requests_total", "Total HTTP requests"),
        &["method", "path", "status"]
    ).unwrap();
    
    pub static ref HTTP_REQUEST_DURATION_SECONDS: HistogramVec = register_histogram_vec!(
        "http_request_duration_seconds",
        "HTTP request duration in seconds",
        &["method", "path"],
        vec![0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
    ).unwrap();
    
    // Business Metrics
    pub static ref ORDERS_CREATED_TOTAL: Counter = prometheus::register_counter!(
        opts!("orders_created_total", "Total orders created")
    ).unwrap();
    
    pub static ref ORDER_VALUE_TOTAL: Counter = prometheus::register_counter!(
        opts!("order_value_total", "Total value of all orders")
    ).unwrap();
    
    pub static ref ACTIVE_USERS_GAUGE: Gauge = register_gauge!(
        opts!("active_users", "Currently active users")
    ).unwrap();
    
    // Database Metrics
    pub static ref DB_QUERY_DURATION_SECONDS: HistogramVec = register_histogram_vec!(
        "db_query_duration_seconds",
        "Database query duration",
        &["operation", "table"],
        vec![0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0]
    ).unwrap();
    
    pub static ref DB_POOL_SIZE: GaugeVec = register_gauge_vec!(
        opts!("db_pool_connections", "Database pool connections"),
        &["state"]  // active, idle, waiting
    ).unwrap();
    
    // Cache Metrics
    pub static ref CACHE_HITS_TOTAL: CounterVec = register_counter_vec!(
        opts!("cache_hits_total", "Total cache hits"),
        &["cache_type"]
    ).unwrap();
    
    pub static ref CACHE_MISSES_TOTAL: CounterVec = register_counter_vec!(
        opts!("cache_misses_total", "Total cache misses"),
        &["cache_type"]
    ).unwrap();
}

/// Metrics endpoint
pub async fn metrics_endpoint() -> HttpResponse {
    let encoder = TextEncoder::new();
    let metric_families = prometheus::gather();
    
    let mut buffer = Vec::new();
    encoder.encode(&metric_families, &mut buffer).unwrap();
    
    HttpResponse::Ok()
        .content_type("text/plain; version=0.0.4")
        .body(buffer)
}

/// Helper functions สำหรับ record metrics
pub fn record_http_request(method: &str, path: &str, status: u16, duration: f64) {
    HTTP_REQUESTS_TOTAL
        .with_label_values(&[method, path, &status.to_string()])
        .inc();
    
    HTTP_REQUEST_DURATION_SECONDS
        .with_label_values(&[method, path])
        .observe(duration);
}

pub fn record_db_query(operation: &str, table: &str, duration: f64) {
    DB_QUERY_DURATION_SECONDS
        .with_label_values(&[operation, table])
        .observe(duration);
}

pub fn record_cache_hit(cache_type: &str) {
    CACHE_HITS_TOTAL.with_label_values(&[cache_type]).inc();
}

pub fn record_cache_miss(cache_type: &str) {
    CACHE_MISSES_TOTAL.with_label_values(&[cache_type]).inc();
}
```

## OpenTelemetry Integration

```rust
// src/telemetry/opentelemetry.rs
use opentelemetry::{
    global,
    sdk::{
        propagation::TraceContextPropagator,
        trace::{self, Sampler},
        Resource,
    },
    KeyValue,
};
use opentelemetry_otlp::WithExportConfig;
use tracing_opentelemetry::OpenTelemetryLayer;
use tracing_subscriber::{layer::SubscriberExt, Registry};

pub struct OtelConfig {
    pub service_name: String,
    pub service_version: String,
    pub otlp_endpoint: String,
    pub sample_ratio: f64,
}

pub fn init_opentelemetry(config: &OtelConfig) -> Result<(), Box<dyn std::error::Error>> {
    // Set global propagator
    global::set_text_map_propagator(TraceContextPropagator::new());
    
    // Create OTLP exporter
    let tracer = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(
            opentelemetry_otlp::new_exporter()
                .tonic()
                .with_endpoint(&config.otlp_endpoint)
        )
        .with_trace_config(
            trace::config()
                .with_sampler(Sampler::TraceIdRatioBased(config.sample_ratio))
                .with_resource(Resource::new(vec![
                    KeyValue::new("service.name", config.service_name.clone()),
                    KeyValue::new("service.version", config.service_version.clone()),
                    KeyValue::new("deployment.environment", 
                        std::env::var("APP_ENV").unwrap_or_else(|_| "unknown".to_string())),
                ]))
        )
        .install_batch(opentelemetry::runtime::Tokio)?;
    
    // Combine tracing subscriber layers
    let otel_layer = OpenTelemetryLayer::new(tracer);
    
    let subscriber = Registry::default()
        .with(tracing_subscriber::EnvFilter::from_default_env())
        .with(tracing_subscriber::fmt::layer().json())
        .with(otel_layer);
    
    tracing::subscriber::set_global_default(subscriber)?;
    
    Ok(())
}

pub fn shutdown_opentelemetry() {
    global::shutdown_tracer_provider();
}
```

## Request Timing Middleware

```rust
// src/telemetry/timing.rs
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error,
};
use futures_util::future::LocalBoxFuture;
use std::rc::Rc;
use std::time::Instant;

use crate::telemetry::metrics::{HTTP_REQUEST_DURATION_SECONDS, HTTP_REQUESTS_TOTAL};

pub struct RequestTimingMiddleware;

impl<S, B> Transform<S, ServiceRequest> for RequestTimingMiddleware
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = RequestTimingService<S>;
    type Future = std::future::Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        std::future::ready(Ok(RequestTimingService {
            service: Rc::new(service),
        }))
    }
}

pub struct RequestTimingService<S> {
    service: Rc<S>,
}

impl<S, B> Service<ServiceRequest> for RequestTimingService<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    forward_ready!(service);

    fn call(&self, req: ServiceRequest) -> Self::Future {
        let method = req.method().to_string();
        let path = req.path().to_string();
        let start = Instant::now();
        
        let service = Rc::clone(&self.service);
        
        Box::pin(async move {
            let res = service.call(req).await?;
            
            let duration = start.elapsed().as_secs_f64();
            let status = res.status().as_u16();
            
            // Record Prometheus metrics
            HTTP_REQUESTS_TOTAL
                .with_label_values(&[&method, &path, &status.to_string()])
                .inc();
            
            HTTP_REQUEST_DURATION_SECONDS
                .with_label_values(&[&method, &path])
                .observe(duration);
            
            // Log slow requests
            if duration > 1.0 {
                tracing::warn!(
                    method = %method,
                    path = %path,
                    duration_ms = %format!("{:.2}", duration * 1000.0),
                    status = %status,
                    "Slow request detected"
                );
            }
            
            Ok(res)
        })
    }
}
```

## Database Query Tracing

```rust
// src/telemetry/db_tracing.rs
use std::time::Instant;
use tracing::instrument;

use crate::telemetry::metrics::record_db_query;

/// Wrapper สำหรับ trace database queries
pub struct TracedPool {
    pool: sqlx::PgPool,
}

impl TracedPool {
    pub fn new(pool: sqlx::PgPool) -> Self {
        TracedPool { pool }
    }
    
    #[instrument(
        name = "db_query",
        skip(self, query),
        fields(
            db.operation = %operation,
            db.table = %table,
            db.rows_affected = tracing::field::Empty,
        )
    )]
    pub async fn execute(
        &self,
        operation: &str,
        table: &str,
        query: &str,
    ) -> Result<u64, sqlx::Error> {
        let start = Instant::now();
        
        let result = sqlx::query(query)
            .execute(&self.pool)
            .await?;
        
        let duration = start.elapsed().as_secs_f64();
        let rows = result.rows_affected();
        
        // Record to span
        tracing::Span::current().record("db.rows_affected", &rows);
        
        // Record to Prometheus
        record_db_query(operation, table, duration);
        
        tracing::debug!(
            duration_ms = format!("{:.2}", duration * 1000.0),
            rows_affected = rows,
            "Query executed"
        );
        
        Ok(rows)
    }
}
```

## HTTP Handlers with Observability

```rust
// src/interface/http/handlers.rs
use actix_web::{web, HttpResponse, HttpRequest};
use tracing::{info, instrument, Span};
use uuid::Uuid;
use std::time::Instant;
use serde::{Deserialize, Serialize};

use crate::telemetry::{
    correlation::CorrelationIds,
    metrics::{ORDERS_CREATED_TOTAL, ORDER_VALUE_TOTAL, ACTIVE_USERS_GAUGE},
};

#[derive(Deserialize)]
pub struct CreateOrderRequest {
    pub customer_id: Uuid,
    pub total_amount: f64,
}

#[derive(Serialize)]
pub struct OrderResponse {
    pub id: Uuid,
    pub status: String,
}

#[instrument(
    name = "api.create_order",
    skip(req, body),
    fields(
        customer_id = %body.customer_id,
        order_id = tracing::field::Empty,
    )
)]
pub async fn create_order(
    req: HttpRequest,
    body: web::Json<CreateOrderRequest>,
) -> HttpResponse {
    // ดึง correlation IDs
    let correlation_ids = req.extensions().get::<CorrelationIds>().cloned();
    
    if let Some(ids) = &correlation_ids {
        tracing::info!(
            correlation_id = %ids.correlation_id,
            "Processing order creation"
        );
    }
    
    let order_id = Uuid::new_v4();
    Span::current().record("order_id", &order_id.to_string().as_str());
    
    // Business logic...
    tokio::time::sleep(std::time::Duration::from_millis(10)).await;
    
    // Record business metrics
    ORDERS_CREATED_TOTAL.inc();
    ORDER_VALUE_TOTAL.inc_by(body.total_amount);
    
    info!(
        order_id = %order_id,
        customer_id = %body.customer_id,
        amount = body.total_amount,
        "Order created successfully"
    );
    
    HttpResponse::Created().json(OrderResponse {
        id: order_id,
        status: "created".to_string(),
    })
}

/// Login endpoint ที่ track active users
pub async fn login(body: web::Json<LoginRequest>) -> HttpResponse {
    // ... authenticate
    
    // Increment active users counter
    ACTIVE_USERS_GAUGE.inc();
    
    info!(user_email = %body.email, "User logged in");
    
    HttpResponse::Ok().json(serde_json::json!({"token": "xxx"}))
}

pub async fn logout(req: HttpRequest) -> HttpResponse {
    ACTIVE_USERS_GAUGE.dec();
    HttpResponse::Ok().finish()
}

#[derive(Deserialize)]
pub struct LoginRequest {
    pub email: String,
    pub password: String,
}
```

## Complete Main with Full Observability

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware};
use std::sync::Arc;

mod telemetry;
use telemetry::{
    logging::{init_tracing, LogConfig, LogFormat},
    metrics::metrics_endpoint,
    correlation::CorrelationIdMiddleware,
    timing::RequestTimingMiddleware,
};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // Initialize tracing
    init_tracing(&LogConfig {
        level: std::env::var("LOG_LEVEL").unwrap_or_else(|_| "info".to_string()),
        format: if std::env::var("APP_ENV").as_deref() == Ok("production") {
            LogFormat::Json
        } else {
            LogFormat::Pretty
        },
        service_name: "my-service".to_string(),
        service_version: env!("CARGO_PKG_VERSION").to_string(),
    });
    
    // Optional: Initialize OpenTelemetry
    if let Ok(otlp_endpoint) = std::env::var("OTLP_ENDPOINT") {
        if let Err(e) = telemetry::opentelemetry::init_opentelemetry(
            &telemetry::opentelemetry::OtelConfig {
                service_name: "my-service".to_string(),
                service_version: env!("CARGO_PKG_VERSION").to_string(),
                otlp_endpoint,
                sample_ratio: 0.1, // sample 10%
            }
        ) {
            tracing::warn!("Failed to init OpenTelemetry: {}", e);
        }
    }
    
    tracing::info!("Starting server");
    
    HttpServer::new(|| {
        App::new()
            // Observability middlewares
            .wrap(CorrelationIdMiddleware)
            .wrap(RequestTimingMiddleware)
            .wrap(middleware::Logger::new(
                "%a %r %s %b %{Referer}i %{User-Agent}i %T"
            ))
            
            // Metrics endpoint
            .route("/metrics", web::get().to(metrics_endpoint))
            
            // API routes
            .service(
                web::scope("/api")
                    .route("/orders", web::post().to(interface::http::handlers::create_order))
            )
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

## Grafana Dashboard JSON

```json
{
  "dashboard": {
    "title": "Rust API Dashboard",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [{
          "expr": "rate(http_requests_total[5m])",
          "legendFormat": "{{method}} {{path}}"
        }]
      },
      {
        "title": "Request Duration P99",
        "type": "graph",
        "targets": [{
          "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))",
          "legendFormat": "P99 {{path}}"
        }]
      },
      {
        "title": "Error Rate",
        "type": "singlestat",
        "targets": [{
          "expr": "rate(http_requests_total{status=~\"5..\"}[5m]) / rate(http_requests_total[5m])"
        }]
      },
      {
        "title": "Active Users",
        "type": "singlestat",
        "targets": [{
          "expr": "active_users"
        }]
      },
      {
        "title": "Database Query Duration",
        "type": "graph",
        "targets": [{
          "expr": "histogram_quantile(0.95, rate(db_query_duration_seconds_bucket[5m]))",
          "legendFormat": "P95 {{operation}} {{table}}"
        }]
      },
      {
        "title": "Cache Hit Rate",
        "type": "graph",
        "targets": [{
          "expr": "rate(cache_hits_total[5m]) / (rate(cache_hits_total[5m]) + rate(cache_misses_total[5m]))",
          "legendFormat": "Hit Rate {{cache_type}}"
        }]
      }
    ]
  }
}
```

## Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use actix_web::test;
    use tracing_subscriber::fmt::TestWriter;
    
    #[test]
    fn test_tracing_setup() {
        // ทดสอบว่า setup ไม่ panic
        let subscriber = tracing_subscriber::fmt()
            .with_writer(TestWriter::new())
            .finish();
        
        // ทดสอบ log messages
        let _guard = tracing::subscriber::set_default(subscriber);
        
        tracing::info!(test = "value", "Test log message");
        tracing::warn!(code = 42, "Test warning");
    }
    
    #[test]
    fn test_prometheus_metrics() {
        use crate::telemetry::metrics::*;
        
        // Test counter increment
        let initial = HTTP_REQUESTS_TOTAL
            .with_label_values(&["GET", "/test", "200"])
            .get();
        
        record_http_request("GET", "/test", 200, 0.1);
        
        let after = HTTP_REQUESTS_TOTAL
            .with_label_values(&["GET", "/test", "200"])
            .get();
        
        assert_eq!(after, initial + 1.0);
    }
    
    #[test]
    fn test_cache_metrics() {
        use crate::telemetry::metrics::*;
        
        let initial_hits = CACHE_HITS_TOTAL.with_label_values(&["redis"]).get();
        let initial_misses = CACHE_MISSES_TOTAL.with_label_values(&["redis"]).get();
        
        record_cache_hit("redis");
        record_cache_hit("redis");
        record_cache_miss("redis");
        
        assert_eq!(CACHE_HITS_TOTAL.with_label_values(&["redis"]).get(), initial_hits + 2.0);
        assert_eq!(CACHE_MISSES_TOTAL.with_label_values(&["redis"]).get(), initial_misses + 1.0);
    }
    
    #[actix_web::test]
    async fn test_metrics_endpoint() {
        let app = test::init_service(
            actix_web::App::new()
                .route("/metrics", actix_web::web::get().to(metrics_endpoint))
        ).await;
        
        let req = test::TestRequest::get().uri("/metrics").to_request();
        let resp = test::call_service(&app, req).await;
        
        assert_eq!(resp.status(), actix_web::http::StatusCode::OK);
        
        let content_type = resp.headers()
            .get("content-type")
            .unwrap()
            .to_str()
            .unwrap();
        
        assert!(content_type.contains("text/plain"));
    }
    
    #[actix_web::test]
    async fn test_correlation_id_middleware() {
        let app = test::init_service(
            actix_web::App::new()
                .wrap(CorrelationIdMiddleware)
                .route("/test", actix_web::web::get().to(|| async {
                    actix_web::HttpResponse::Ok().finish()
                }))
        ).await;
        
        let correlation_id = uuid::Uuid::new_v4().to_string();
        
        let req = test::TestRequest::get()
            .uri("/test")
            .insert_header(("X-Correlation-ID", correlation_id.as_str()))
            .to_request();
        
        let resp = test::call_service(&app, req).await;
        
        // ตรวจสอบว่า correlation ID ถูก return กลับมา
        let resp_correlation_id = resp.headers()
            .get("x-correlation-id")
            .unwrap()
            .to_str()
            .unwrap();
        
        assert_eq!(resp_correlation_id, correlation_id);
    }
}
```

## สรุป

Observability Stack ที่สมบูรณ์:

```
Application → tracing crate → tracing-subscriber
                                    ├── stdout/file logs
                                    ├── Prometheus metrics (/metrics)
                                    └── OpenTelemetry → Jaeger/Tempo
                                    
Prometheus ← scrape /metrics → Grafana dashboards
Grafana → alerts → PagerDuty/Slack
```

---

## Navigation

- [← Part 068: Configuration Management](../part_068/README.md)
- [→ Part 070: Health Checks and Monitoring](../part_070/README.md)
- [กลับหน้าหลัก](../../README.md)
