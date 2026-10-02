# Part 065: Microservices Architecture in Rust

## ภาพรวม

Microservices Architecture คือการแบ่ง application ออกเป็น services เล็ก ๆ ที่ deploy ได้อิสระ บทนี้จะครอบคลุมการสร้าง microservices ด้วย Rust และ Actix-web พร้อม patterns สำคัญต่าง ๆ

## สถาปัตยกรรม

```
                    ┌─────────────┐
                    │ API Gateway │
                    └──────┬──────┘
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
    ┌───────────────┐ ┌───────────┐ ┌──────────────┐
    │ User Service  │ │  Product  │ │ Order Service│
    │ :8081         │ │  Service  │ │ :8083        │
    │               │ │ :8082     │ │              │
    └───────┬───────┘ └─────┬─────┘ └──────┬───────┘
            │               │              │
            └───────────────┴──────────────┘
                    Message Bus / Events
```

## Cargo.toml (User Service)

```toml
[package]
name = "user-service"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4", "serde"] }
thiserror = "1"
reqwest = { version = "0.11", features = ["json"] }
chrono = { version = "0.4", features = ["serde"] }
tracing = "0.1"
tracing-subscriber = "0.3"
tokio-retry = "0.3"
```

## Service Discovery

```rust
// src/service_discovery/mod.rs
use std::collections::HashMap;
use std::sync::{Arc, RwLock};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ServiceInstance {
    pub service_name: String,
    pub instance_id: String,
    pub host: String,
    pub port: u16,
    pub health_check_url: String,
    pub metadata: HashMap<String, String>,
}

impl ServiceInstance {
    pub fn base_url(&self) -> String {
        format!("http://{}:{}", self.host, self.port)
    }
}

/// In-memory service registry (ในทางปฏิบัติใช้ Consul/etcd)
pub struct ServiceRegistry {
    instances: Arc<RwLock<HashMap<String, Vec<ServiceInstance>>>>,
}

impl ServiceRegistry {
    pub fn new() -> Self {
        ServiceRegistry {
            instances: Arc::new(RwLock::new(HashMap::new())),
        }
    }
    
    pub fn register(&self, instance: ServiceInstance) {
        let mut registry = self.instances.write().unwrap();
        registry
            .entry(instance.service_name.clone())
            .or_insert_with(Vec::new)
            .push(instance);
    }
    
    pub fn deregister(&self, service_name: &str, instance_id: &str) {
        let mut registry = self.instances.write().unwrap();
        if let Some(instances) = registry.get_mut(service_name) {
            instances.retain(|i| i.instance_id != instance_id);
        }
    }
    
    pub fn get_instances(&self, service_name: &str) -> Vec<ServiceInstance> {
        let registry = self.instances.read().unwrap();
        registry.get(service_name).cloned().unwrap_or_default()
    }
    
    /// Round-robin load balancing
    pub fn get_instance(&self, service_name: &str) -> Option<ServiceInstance> {
        let instances = self.get_instances(service_name);
        if instances.is_empty() {
            return None;
        }
        
        // Simple round-robin (ในทางปฏิบัติใช้ atomic counter)
        use std::time::{SystemTime, UNIX_EPOCH};
        let idx = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .subsec_nanos() as usize % instances.len();
        
        Some(instances[idx].clone())
    }
}
```

## Circuit Breaker Pattern

```rust
// src/circuit_breaker/mod.rs
use std::sync::{Arc, Mutex};
use std::time::{Duration, Instant};
use tokio::time::sleep;

#[derive(Debug, Clone, PartialEq)]
pub enum CircuitState {
    Closed,   // ปกติ - requests ผ่านได้
    Open,     // Circuit เปิด - reject ทุก request
    HalfOpen, // ทดสอบว่า service กลับมาหรือยัง
}

#[derive(Debug)]
struct CircuitBreakerState {
    state: CircuitState,
    failure_count: u32,
    success_count: u32,
    last_failure_time: Option<Instant>,
    failure_threshold: u32,
    success_threshold: u32,
    timeout: Duration,
}

impl CircuitBreakerState {
    fn new(failure_threshold: u32, success_threshold: u32, timeout: Duration) -> Self {
        CircuitBreakerState {
            state: CircuitState::Closed,
            failure_count: 0,
            success_count: 0,
            last_failure_time: None,
            failure_threshold,
            success_threshold,
            timeout,
        }
    }
}

pub struct CircuitBreaker {
    name: String,
    state: Arc<Mutex<CircuitBreakerState>>,
}

#[derive(Debug, thiserror::Error)]
pub enum CircuitBreakerError {
    #[error("Circuit is open for service: {0}")]
    CircuitOpen(String),
    #[error("Request failed: {0}")]
    RequestFailed(String),
}

impl CircuitBreaker {
    pub fn new(
        name: String,
        failure_threshold: u32,
        success_threshold: u32,
        timeout: Duration,
    ) -> Self {
        CircuitBreaker {
            name: name.clone(),
            state: Arc::new(Mutex::new(CircuitBreakerState::new(
                failure_threshold,
                success_threshold,
                timeout,
            ))),
        }
    }
    
    pub async fn execute<F, T, E>(&self, operation: F) -> Result<T, CircuitBreakerError>
    where
        F: std::future::Future<Output = Result<T, E>>,
        E: std::fmt::Display,
    {
        // ตรวจสอบ state ก่อน execute
        {
            let mut state = self.state.lock().unwrap();
            
            match state.state {
                CircuitState::Open => {
                    // ตรวจว่าถึงเวลา timeout แล้วหรือยัง
                    if let Some(last_failure) = state.last_failure_time {
                        if last_failure.elapsed() >= state.timeout {
                            // เปลี่ยนเป็น HalfOpen
                            state.state = CircuitState::HalfOpen;
                            state.success_count = 0;
                            tracing::info!("Circuit breaker '{}' moving to HalfOpen", self.name);
                        } else {
                            return Err(CircuitBreakerError::CircuitOpen(self.name.clone()));
                        }
                    }
                }
                CircuitState::Closed | CircuitState::HalfOpen => {}
            }
        }
        
        // Execute operation
        match operation.await {
            Ok(result) => {
                self.record_success();
                Ok(result)
            }
            Err(e) => {
                self.record_failure();
                Err(CircuitBreakerError::RequestFailed(e.to_string()))
            }
        }
    }
    
    fn record_success(&self) {
        let mut state = self.state.lock().unwrap();
        state.success_count += 1;
        
        match state.state {
            CircuitState::HalfOpen => {
                if state.success_count >= state.success_threshold {
                    state.state = CircuitState::Closed;
                    state.failure_count = 0;
                    state.success_count = 0;
                    tracing::info!("Circuit breaker '{}' closed", self.name);
                }
            }
            CircuitState::Closed => {
                state.failure_count = 0;
            }
            _ => {}
        }
    }
    
    fn record_failure(&self) {
        let mut state = self.state.lock().unwrap();
        state.failure_count += 1;
        state.last_failure_time = Some(Instant::now());
        
        match state.state {
            CircuitState::Closed => {
                if state.failure_count >= state.failure_threshold {
                    state.state = CircuitState::Open;
                    tracing::warn!("Circuit breaker '{}' opened after {} failures", 
                        self.name, state.failure_count);
                }
            }
            CircuitState::HalfOpen => {
                state.state = CircuitState::Open;
                tracing::warn!("Circuit breaker '{}' back to Open", self.name);
            }
            _ => {}
        }
    }
    
    pub fn state(&self) -> CircuitState {
        self.state.lock().unwrap().state.clone()
    }
}
```

## HTTP Client สำหรับ Inter-service Communication

```rust
// src/clients/product_client.rs
use reqwest::Client;
use uuid::Uuid;
use std::sync::Arc;
use std::time::Duration;
use serde::{Deserialize, Serialize};
use tokio_retry::{Retry, strategy::ExponentialBackoff};

use crate::circuit_breaker::CircuitBreaker;

#[derive(Debug, Deserialize, Serialize)]
pub struct ProductDto {
    pub id: Uuid,
    pub name: String,
    pub price: f64,
    pub stock: u32,
}

pub struct ProductServiceClient {
    client: Client,
    base_url: String,
    circuit_breaker: Arc<CircuitBreaker>,
}

impl ProductServiceClient {
    pub fn new(base_url: String) -> Self {
        let client = Client::builder()
            .timeout(Duration::from_secs(5))
            .build()
            .expect("Failed to build HTTP client");
        
        let circuit_breaker = Arc::new(CircuitBreaker::new(
            "product-service".to_string(),
            5,  // failure threshold
            2,  // success threshold  
            Duration::from_secs(60), // timeout
        ));
        
        ProductServiceClient { client, base_url, circuit_breaker }
    }
    
    pub async fn get_product(&self, product_id: Uuid) -> Result<ProductDto, String> {
        let url = format!("{}/products/{}", self.base_url, product_id);
        let client = self.client.clone();
        
        self.circuit_breaker
            .execute(async move {
                client.get(&url)
                    .send()
                    .await
                    .map_err(|e| e.to_string())?
                    .json::<ProductDto>()
                    .await
                    .map_err(|e| e.to_string())
            })
            .await
            .map_err(|e| e.to_string())
    }
    
    pub async fn check_stock(&self, product_id: Uuid, quantity: u32) -> Result<bool, String> {
        let url = format!("{}/products/{}/stock?quantity={}", 
            self.base_url, product_id, quantity);
        let client = self.client.clone();
        
        // ใช้ retry สำหรับ transient failures
        let retry_strategy = ExponentialBackoff::from_millis(100)
            .max_delay(Duration::from_secs(5))
            .take(3);
        
        Retry::spawn(retry_strategy, || async {
            let response = client.get(&url)
                .send()
                .await
                .map_err(|e| e.to_string())?;
            
            if response.status().is_server_error() {
                return Err(format!("Server error: {}", response.status()));
            }
            
            response.json::<serde_json::Value>()
                .await
                .map(|v| v["available"].as_bool().unwrap_or(false))
                .map_err(|e| e.to_string())
        }).await
    }
    
    pub async fn reserve_stock(&self, product_id: Uuid, quantity: u32) -> Result<(), String> {
        let url = format!("{}/products/{}/reserve", self.base_url, product_id);
        let client = self.client.clone();
        
        self.circuit_breaker
            .execute(async move {
                let response = client.post(&url)
                    .json(&serde_json::json!({ "quantity": quantity }))
                    .send()
                    .await
                    .map_err(|e| e.to_string())?;
                
                if response.status().is_success() {
                    Ok(())
                } else {
                    Err(format!("Reserve failed: {}", response.status()))
                }
            })
            .await
            .map_err(|e| e.to_string())
    }
}
```

## API Gateway Pattern

```rust
// src/gateway/mod.rs
use actix_web::{web, HttpRequest, HttpResponse, middleware};
use reqwest::Client;
use std::collections::HashMap;
use std::sync::Arc;

pub struct GatewayRoute {
    pub path_prefix: String,
    pub service_url: String,
    pub strip_prefix: bool,
}

pub struct ApiGateway {
    client: Client,
    routes: Vec<GatewayRoute>,
    circuit_breakers: HashMap<String, Arc<CircuitBreaker>>,
}

impl ApiGateway {
    pub fn new(routes: Vec<GatewayRoute>) -> Self {
        let client = Client::builder()
            .timeout(std::time::Duration::from_secs(30))
            .build()
            .unwrap();
        
        let mut circuit_breakers = HashMap::new();
        for route in &routes {
            let cb = Arc::new(CircuitBreaker::new(
                route.service_url.clone(),
                5,
                2,
                std::time::Duration::from_secs(60),
            ));
            circuit_breakers.insert(route.service_url.clone(), cb);
        }
        
        ApiGateway { client, routes, circuit_breakers }
    }
    
    async fn proxy_request(
        &self,
        req: &HttpRequest,
        body: web::Bytes,
        target_url: String,
    ) -> HttpResponse {
        let method = req.method().clone();
        let headers = req.headers().clone();
        
        let mut request_builder = match method.as_str() {
            "GET" => self.client.get(&target_url),
            "POST" => self.client.post(&target_url),
            "PUT" => self.client.put(&target_url),
            "DELETE" => self.client.delete(&target_url),
            "PATCH" => self.client.patch(&target_url),
            _ => return HttpResponse::MethodNotAllowed().finish(),
        };
        
        // Forward headers
        for (name, value) in headers.iter() {
            if name != "host" {
                if let Ok(val) = value.to_str() {
                    request_builder = request_builder.header(name.as_str(), val);
                }
            }
        }
        
        // Forward body
        if !body.is_empty() {
            request_builder = request_builder.body(body);
        }
        
        match request_builder.send().await {
            Ok(response) => {
                let status = response.status();
                let body = response.bytes().await.unwrap_or_default();
                
                HttpResponse::build(actix_web::http::StatusCode::from_u16(status.as_u16()).unwrap())
                    .body(body)
            }
            Err(e) => {
                tracing::error!("Proxy error: {}", e);
                HttpResponse::ServiceUnavailable().json(serde_json::json!({
                    "error": "Service unavailable"
                }))
            }
        }
    }
    
    pub async fn handle_request(
        &self,
        req: HttpRequest,
        body: web::Bytes,
    ) -> HttpResponse {
        let path = req.path();
        
        // หา route ที่ match
        for route in &self.routes {
            if path.starts_with(&route.path_prefix) {
                let target_path = if route.strip_prefix {
                    path.strip_prefix(&route.path_prefix).unwrap_or(path)
                } else {
                    path
                };
                
                let query = req.query_string();
                let target_url = if query.is_empty() {
                    format!("{}{}", route.service_url, target_path)
                } else {
                    format!("{}{}?{}", route.service_url, target_path, query)
                };
                
                return self.proxy_request(&req, body, target_url).await;
            }
        }
        
        HttpResponse::NotFound().json(serde_json::json!({
            "error": "Route not found"
        }))
    }
}
```

## Health Checks

```rust
// src/health/mod.rs
use actix_web::{web, HttpResponse};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::Arc;
use async_trait::async_trait;

#[derive(Debug, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "lowercase")]
pub enum HealthStatus {
    Healthy,
    Degraded,
    Unhealthy,
}

#[derive(Debug, Serialize)]
pub struct HealthCheckResult {
    pub status: HealthStatus,
    pub checks: HashMap<String, ComponentHealth>,
    pub version: String,
    pub timestamp: chrono::DateTime<chrono::Utc>,
}

#[derive(Debug, Serialize)]
pub struct ComponentHealth {
    pub status: HealthStatus,
    pub message: Option<String>,
    pub latency_ms: Option<u64>,
}

#[async_trait]
pub trait HealthCheck: Send + Sync {
    fn name(&self) -> &str;
    async fn check(&self) -> ComponentHealth;
}

pub struct HealthChecker {
    checks: Vec<Arc<dyn HealthCheck>>,
}

impl HealthChecker {
    pub fn new() -> Self {
        HealthChecker { checks: Vec::new() }
    }
    
    pub fn add_check(&mut self, check: Arc<dyn HealthCheck>) {
        self.checks.push(check);
    }
    
    pub async fn run_all(&self) -> HealthCheckResult {
        let mut results = HashMap::new();
        let mut overall = HealthStatus::Healthy;
        
        for check in &self.checks {
            let start = std::time::Instant::now();
            let mut result = check.check().await;
            result.latency_ms = Some(start.elapsed().as_millis() as u64);
            
            match &result.status {
                HealthStatus::Unhealthy => overall = HealthStatus::Unhealthy,
                HealthStatus::Degraded if overall != HealthStatus::Unhealthy => {
                    overall = HealthStatus::Degraded;
                }
                _ => {}
            }
            
            results.insert(check.name().to_string(), result);
        }
        
        HealthCheckResult {
            status: overall,
            checks: results,
            version: env!("CARGO_PKG_VERSION").to_string(),
            timestamp: chrono::Utc::now(),
        }
    }
}

// Database health check
pub struct DatabaseHealthCheck {
    pool: sqlx::PgPool,
}

#[async_trait]
impl HealthCheck for DatabaseHealthCheck {
    fn name(&self) -> &str { "database" }
    
    async fn check(&self) -> ComponentHealth {
        match sqlx::query("SELECT 1").execute(&self.pool).await {
            Ok(_) => ComponentHealth {
                status: HealthStatus::Healthy,
                message: None,
                latency_ms: None,
            },
            Err(e) => ComponentHealth {
                status: HealthStatus::Unhealthy,
                message: Some(e.to_string()),
                latency_ms: None,
            },
        }
    }
}

// External service health check
pub struct ExternalServiceHealthCheck {
    name: String,
    url: String,
    client: reqwest::Client,
}

impl ExternalServiceHealthCheck {
    pub fn new(name: String, url: String) -> Self {
        ExternalServiceHealthCheck {
            name,
            url,
            client: reqwest::Client::new(),
        }
    }
}

#[async_trait]
impl HealthCheck for ExternalServiceHealthCheck {
    fn name(&self) -> &str { &self.name }
    
    async fn check(&self) -> ComponentHealth {
        match self.client.get(&self.url)
            .timeout(std::time::Duration::from_secs(5))
            .send()
            .await
        {
            Ok(resp) if resp.status().is_success() => ComponentHealth {
                status: HealthStatus::Healthy,
                message: None,
                latency_ms: None,
            },
            Ok(resp) => ComponentHealth {
                status: HealthStatus::Degraded,
                message: Some(format!("Unexpected status: {}", resp.status())),
                latency_ms: None,
            },
            Err(e) => ComponentHealth {
                status: HealthStatus::Unhealthy,
                message: Some(e.to_string()),
                latency_ms: None,
            },
        }
    }
}

pub async fn health_endpoint(
    checker: web::Data<Arc<HealthChecker>>,
) -> HttpResponse {
    let result = checker.run_all().await;
    let status_code = match result.status {
        HealthStatus::Healthy => actix_web::http::StatusCode::OK,
        HealthStatus::Degraded => actix_web::http::StatusCode::OK,
        HealthStatus::Unhealthy => actix_web::http::StatusCode::SERVICE_UNAVAILABLE,
    };
    
    HttpResponse::build(status_code).json(result)
}
```

## Service A - User Service

```rust
// services/user-service/src/main.rs
use actix_web::{web, App, HttpServer, HttpResponse};
use std::sync::Arc;
use uuid::Uuid;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct User {
    pub id: Uuid,
    pub name: String,
    pub email: String,
}

pub async fn get_user(path: web::Path<Uuid>) -> HttpResponse {
    // Simulate getting user
    let user = User {
        id: *path,
        name: "John Doe".to_string(),
        email: "john@example.com".to_string(),
    };
    HttpResponse::Ok().json(user)
}

pub async fn health() -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({
        "status": "healthy",
        "service": "user-service",
        "version": "1.0.0"
    }))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    tracing_subscriber::fmt::init();
    
    let port = std::env::var("PORT")
        .unwrap_or_else(|_| "8081".to_string())
        .parse::<u16>()
        .expect("Invalid port");
    
    tracing::info!("User service starting on port {}", port);
    
    HttpServer::new(|| {
        App::new()
            .route("/health", web::get().to(health))
            .service(
                web::scope("/users")
                    .route("/{id}", web::get().to(get_user))
            )
    })
    .bind(format!("0.0.0.0:{}", port))?
    .run()
    .await
}
```

## Service B - Product Service

```rust
// services/product-service/src/main.rs
use actix_web::{web, App, HttpServer, HttpResponse};
use std::sync::Arc;
use uuid::Uuid;
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::RwLock;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Product {
    pub id: Uuid,
    pub name: String,
    pub price: f64,
    pub stock: u32,
}

#[derive(Deserialize)]
pub struct StockQuery {
    pub quantity: u32,
}

pub struct ProductStore {
    products: RwLock<HashMap<Uuid, Product>>,
}

impl ProductStore {
    pub fn new() -> Self {
        let mut products = HashMap::new();
        
        // Sample data
        let id1 = Uuid::new_v4();
        products.insert(id1, Product {
            id: id1,
            name: "Laptop".to_string(),
            price: 29999.0,
            stock: 50,
        });
        
        ProductStore {
            products: RwLock::new(products),
        }
    }
    
    pub fn get(&self, id: Uuid) -> Option<Product> {
        self.products.read().unwrap().get(&id).cloned()
    }
    
    pub fn check_stock(&self, id: Uuid, needed: u32) -> bool {
        self.products.read().unwrap()
            .get(&id)
            .map(|p| p.stock >= needed)
            .unwrap_or(false)
    }
}

pub async fn get_product(
    store: web::Data<Arc<ProductStore>>,
    path: web::Path<Uuid>,
) -> HttpResponse {
    match store.get(*path) {
        Some(product) => HttpResponse::Ok().json(product),
        None => HttpResponse::NotFound().finish(),
    }
}

pub async fn check_stock(
    store: web::Data<Arc<ProductStore>>,
    path: web::Path<Uuid>,
    query: web::Query<StockQuery>,
) -> HttpResponse {
    let available = store.check_stock(*path, query.quantity);
    HttpResponse::Ok().json(serde_json::json!({ "available": available }))
}

pub async fn health() -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({
        "status": "healthy",
        "service": "product-service"
    }))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let store = Arc::new(ProductStore::new());
    
    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(store.clone()))
            .route("/health", web::get().to(health))
            .service(
                web::scope("/products")
                    .route("/{id}", web::get().to(get_product))
                    .route("/{id}/stock", web::get().to(check_stock))
            )
    })
    .bind("0.0.0.0:8082")?
    .run()
    .await
}
```

## Order Service (ใช้ทั้ง 2 services)

```rust
// services/order-service/src/main.rs
use actix_web::{web, App, HttpServer, HttpResponse};
use reqwest::Client;
use uuid::Uuid;
use serde::{Deserialize, Serialize};
use std::time::Duration;

#[derive(Deserialize)]
pub struct CreateOrderRequest {
    pub user_id: Uuid,
    pub product_id: Uuid,
    pub quantity: u32,
}

#[derive(Serialize)]
pub struct OrderCreatedResponse {
    pub order_id: Uuid,
    pub user_name: String,
    pub product_name: String,
    pub total_price: f64,
}

#[derive(Clone)]
pub struct ServiceUrls {
    pub user_service: String,
    pub product_service: String,
}

pub async fn create_order(
    urls: web::Data<ServiceUrls>,
    body: web::Json<CreateOrderRequest>,
) -> HttpResponse {
    let client = Client::builder()
        .timeout(Duration::from_secs(5))
        .build()
        .unwrap();
    
    // เรียก User Service
    let user_response = client
        .get(format!("{}/users/{}", urls.user_service, body.user_id))
        .send()
        .await;
    
    let user = match user_response {
        Ok(resp) if resp.status().is_success() => {
            match resp.json::<serde_json::Value>().await {
                Ok(u) => u,
                Err(e) => return HttpResponse::InternalServerError().json(
                    serde_json::json!({"error": format!("Failed to parse user: {}", e)})
                ),
            }
        }
        Ok(resp) if resp.status() == 404 => {
            return HttpResponse::NotFound().json(
                serde_json::json!({"error": "User not found"})
            );
        }
        _ => return HttpResponse::ServiceUnavailable().json(
            serde_json::json!({"error": "User service unavailable"})
        ),
    };
    
    // เรียก Product Service
    let product_response = client
        .get(format!("{}/products/{}", urls.product_service, body.product_id))
        .send()
        .await;
    
    let product = match product_response {
        Ok(resp) if resp.status().is_success() => {
            match resp.json::<serde_json::Value>().await {
                Ok(p) => p,
                Err(e) => return HttpResponse::InternalServerError().json(
                    serde_json::json!({"error": format!("Failed to parse product: {}", e)})
                ),
            }
        }
        Ok(resp) if resp.status() == 404 => {
            return HttpResponse::NotFound().json(
                serde_json::json!({"error": "Product not found"})
            );
        }
        _ => return HttpResponse::ServiceUnavailable().json(
            serde_json::json!({"error": "Product service unavailable"})
        ),
    };
    
    // ตรวจสอบ stock
    let stock_available = client
        .get(format!("{}/products/{}/stock?quantity={}", 
            urls.product_service, body.product_id, body.quantity))
        .send()
        .await
        .ok()
        .and_then(|r| r.json::<serde_json::Value>().blocking_ok())
        .and_then(|v| v["available"].as_bool())
        .unwrap_or(false);
    
    if !stock_available {
        return HttpResponse::BadRequest().json(
            serde_json::json!({"error": "Insufficient stock"})
        );
    }
    
    let unit_price = product["price"].as_f64().unwrap_or(0.0);
    let total_price = unit_price * body.quantity as f64;
    
    HttpResponse::Created().json(OrderCreatedResponse {
        order_id: Uuid::new_v4(),
        user_name: user["name"].as_str().unwrap_or("").to_string(),
        product_name: product["name"].as_str().unwrap_or("").to_string(),
        total_price,
    })
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let urls = ServiceUrls {
        user_service: std::env::var("USER_SERVICE_URL")
            .unwrap_or_else(|_| "http://localhost:8081".to_string()),
        product_service: std::env::var("PRODUCT_SERVICE_URL")
            .unwrap_or_else(|_| "http://localhost:8082".to_string()),
    };
    
    tracing::info!("Order service starting");
    
    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(urls.clone()))
            .route("/health", web::get().to(|| async {
                HttpResponse::Ok().json(serde_json::json!({"status": "healthy"}))
            }))
            .route("/orders", web::post().to(create_order))
    })
    .bind("0.0.0.0:8083")?
    .run()
    .await
}
```

## Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  user-service:
    build:
      context: .
      dockerfile: services/user-service/Dockerfile
    ports:
      - "8081:8081"
    environment:
      - PORT=8081
      - DATABASE_URL=postgres://postgres:password@postgres/users
    depends_on:
      - postgres
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  product-service:
    build:
      context: .
      dockerfile: services/product-service/Dockerfile
    ports:
      - "8082:8082"
    environment:
      - PORT=8082
      - DATABASE_URL=postgres://postgres:password@postgres/products
    depends_on:
      - postgres
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8082/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  order-service:
    build:
      context: .
      dockerfile: services/order-service/Dockerfile
    ports:
      - "8083:8083"
    environment:
      - PORT=8083
      - USER_SERVICE_URL=http://user-service:8081
      - PRODUCT_SERVICE_URL=http://product-service:8082
      - DATABASE_URL=postgres://postgres:password@postgres/orders
    depends_on:
      - user-service
      - product-service
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8083/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  api-gateway:
    build:
      context: .
      dockerfile: services/api-gateway/Dockerfile
    ports:
      - "8080:8080"
    environment:
      - USER_SERVICE_URL=http://user-service:8081
      - PRODUCT_SERVICE_URL=http://product-service:8082
      - ORDER_SERVICE_URL=http://order-service:8083
    depends_on:
      - user-service
      - product-service
      - order-service

  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

## Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::time::Duration;
    
    #[tokio::test]
    async fn test_circuit_breaker_opens_after_threshold() {
        let cb = CircuitBreaker::new(
            "test".to_string(),
            3, // open after 3 failures
            2,
            Duration::from_secs(60),
        );
        
        // ทำให้ fail 3 ครั้ง
        for _ in 0..3 {
            let _ = cb.execute(async { Err::<(), String>("error".to_string()) }).await;
        }
        
        assert_eq!(cb.state(), CircuitState::Open);
        
        // Request ถัดไปควร reject ทันที
        let result = cb.execute(async { Ok::<String, String>("ok".to_string()) }).await;
        assert!(matches!(result, Err(CircuitBreakerError::CircuitOpen(_))));
    }
    
    #[tokio::test]
    async fn test_circuit_breaker_closes_after_recovery() {
        let cb = CircuitBreaker::new(
            "test".to_string(),
            2,
            2, // close after 2 successes
            Duration::from_millis(100), // short timeout for testing
        );
        
        // Open the circuit
        for _ in 0..2 {
            let _ = cb.execute(async { Err::<(), String>("error".to_string()) }).await;
        }
        assert_eq!(cb.state(), CircuitState::Open);
        
        // Wait for timeout
        tokio::time::sleep(Duration::from_millis(150)).await;
        
        // ต้องผ่านเพราะ timeout แล้ว (HalfOpen)
        let _ = cb.execute(async { Ok::<(), String>(()) }).await;
        let _ = cb.execute(async { Ok::<(), String>(()) }).await;
        
        assert_eq!(cb.state(), CircuitState::Closed);
    }
    
    #[test]
    fn test_service_registry() {
        let registry = ServiceRegistry::new();
        
        let instance = ServiceInstance {
            service_name: "user-service".to_string(),
            instance_id: "user-1".to_string(),
            host: "localhost".to_string(),
            port: 8081,
            health_check_url: "http://localhost:8081/health".to_string(),
            metadata: std::collections::HashMap::new(),
        };
        
        registry.register(instance.clone());
        
        let instances = registry.get_instances("user-service");
        assert_eq!(instances.len(), 1);
        assert_eq!(instances[0].instance_id, "user-1");
        
        registry.deregister("user-service", "user-1");
        assert!(registry.get_instances("user-service").is_empty());
    }
}
```

## สรุป

Microservices patterns ที่ครอบคลุม:

1. **Service Decomposition** - แบ่งตาม business domain
2. **Circuit Breaker** - ป้องกัน cascade failures
3. **Service Discovery** - dynamic service location
4. **API Gateway** - single entry point
5. **Health Checks** - monitor service health
6. **Retry with Backoff** - handle transient failures

---

## Navigation

- [← Part 064: Event Sourcing](../part_064/README.md)
- [→ Part 066: API Versioning Strategies](../part_066/README.md)
- [กลับหน้าหลัก](../../README.md)
