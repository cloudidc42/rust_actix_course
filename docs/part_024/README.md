# Part 024: Middleware ใน Actix-web

## สารบัญ
- [แนะนำ Middleware](#แนะนำ-middleware)
- [Logger Middleware](#logger-middleware)
- [Compression Middleware](#compression-middleware)
- [Custom Middleware (Transform + Service)](#custom-middleware)
- [Request ID Middleware](#request-id-middleware)
- [Timing Middleware](#timing-middleware)
- [Error Handling Middleware](#error-handling-middleware)
- [Ordering of Middleware](#ordering-of-middleware)

---

## แนะนำ Middleware

Middleware คือ code ที่ทำงานระหว่าง request เข้ามาและ response ออกไป ใน Actix-web เราใช้ `.wrap()` เพื่อเพิ่ม middleware

### แนวคิดพื้นฐาน

```
Request → Middleware 1 → Middleware 2 → Handler → Middleware 2 → Middleware 1 → Response
```

```toml
# Cargo.toml
[dependencies]
actix-web = "4"
actix-cors = "0.7"
actix-rt = "2"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
uuid = { version = "1", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }
log = "0.4"
env_logger = "0.11"
futures-util = "0.3"
pin-project-lite = "0.2"
```

---

## Logger Middleware

Actix-web มี Logger middleware built-in ที่ใช้ได้ทันที

```rust
use actix_web::{middleware, web, App, HttpServer, HttpResponse};
use env_logger::Env;

async fn index() -> HttpResponse {
    HttpResponse::Ok().body("Hello World!")
}

// Basic Logger
async fn basic_logger_example() {
    // initialize logger
    env_logger::init_from_env(Env::default().default_filter_or("info"));
    
    HttpServer::new(|| {
        App::new()
            // default format: %a %t "%r" %s %b "%{Referer}i" "%{User-Agent}i" %T
            .wrap(middleware::Logger::default())
            .route("/", web::get().to(index))
    })
    .bind("127.0.0.1:8080").unwrap()
    .run()
    .await
    .unwrap();
}

// Custom Logger format
async fn custom_logger_example() {
    env_logger::init_from_env(Env::default().default_filter_or("info"));
    
    HttpServer::new(|| {
        App::new()
            .wrap(
                middleware::Logger::new(
                    // Custom format
                    r#"%{r}a "%r" %s %b "%{Referer}i" "%{User-Agent}i" %T"#
                )
                // หรือใช้ format string ที่ละเอียดกว่า
                // %a = remote IP address
                // %t = time request received
                // %r = first line of request (method, path, version)
                // %s = response status code
                // %b = size of response in bytes
                // %T = time to serve request in seconds
                // %D = time to serve request in milliseconds
                // %{name}i = request header value
                // %{name}o = response header value
                // %{name}e = environment variable value
            )
            .route("/", web::get().to(index))
    })
    .bind("127.0.0.1:8080").unwrap()
    .run()
    .await
    .unwrap();
}

// More detailed custom format
async fn detailed_logger_example() {
    env_logger::init_from_env(Env::default().default_filter_or("debug"));
    
    HttpServer::new(|| {
        let logger = middleware::Logger::new(
            // รูปแบบ JSON-like สำหรับ log analysis
            r#"{"ip": "%a", "method": "%{method}xi", "path": "%U", "status": %s, "bytes": %b, "duration_ms": %D, "user_agent": "%{User-Agent}i", "referer": "%{Referer}i"}"#
        )
        .exclude("/health")  // ไม่ log health check endpoint
        .exclude_regex("^/static")  // ไม่ log static files
        .log_target("actix_web::middleware::logger");
        
        App::new()
            .wrap(logger)
            .route("/", web::get().to(index))
            .route("/health", web::get().to(|| async { HttpResponse::Ok().body("OK") }))
    })
    .bind("127.0.0.1:8080").unwrap()
    .run()
    .await
    .unwrap();
}
```

---

## Compression Middleware

Compress middleware ช่วยลดขนาด response โดยอัตโนมัติ

```rust
use actix_web::{middleware::Compress, web, App, HttpServer, HttpResponse};
use actix_web::http::header;

async fn large_response() -> HttpResponse {
    // สร้าง large JSON response
    let data: Vec<serde_json::Value> = (0..1000)
        .map(|i| serde_json::json!({
            "id": i,
            "name": format!("Item {}", i),
            "description": "This is a longer description that adds more bytes to compress",
            "data": vec!["value1", "value2", "value3", "value4", "value5"]
        }))
        .collect();
    
    HttpResponse::Ok()
        .content_type("application/json")
        .json(data)
}

// Setup compression
async fn compression_example() {
    HttpServer::new(|| {
        App::new()
            // Compress: auto-negotiates based on Accept-Encoding header
            // Supports: gzip, br (brotli), zstd, deflate
            .wrap(Compress::default())
            .route("/data", web::get().to(large_response))
    })
    .bind("127.0.0.1:8080").unwrap()
    .run()
    .await
    .unwrap();
}

// Handler ที่ควบคุม compression เอง
async fn manual_compression() -> HttpResponse {
    HttpResponse::Ok()
        .content_type("application/json")
        // บอก client ว่า content เป็น gzip
        .append_header((header::CONTENT_ENCODING, "gzip"))
        .body("compressed content here")
}

// ปิด compression สำหรับ specific routes
use actix_web::dev::{ServiceRequest, ServiceResponse};
use actix_web::middleware::DefaultHeaders;

async fn compression_with_exceptions() {
    HttpServer::new(|| {
        App::new()
            .wrap(Compress::default())
            // เพิ่ม headers
            .wrap(
                DefaultHeaders::new()
                    .add(("X-Version", "1.0"))
            )
            .route("/large-data", web::get().to(large_response))
            // route นี้ไม่ compress
            .service(
                web::resource("/streaming")
                    .wrap(Compress::default())
                    .route(web::get().to(|| async { 
                        HttpResponse::Ok().body("no compression") 
                    }))
            )
    })
    .bind("127.0.0.1:8080").unwrap()
    .run()
    .await
    .unwrap();
}
```

---

## Custom Middleware

การสร้าง middleware เองต้อง implement `Transform` trait และ `Service` trait

```rust
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpResponse,
};
use futures_util::future::LocalBoxFuture;
use std::future::{ready, Ready};
use std::rc::Rc;
use std::cell::RefCell;

// Step 1: สร้าง Middleware struct (เก็บ config)
pub struct Authentication {
    exclude_paths: Vec<String>,
}

impl Authentication {
    pub fn new() -> Self {
        Authentication {
            exclude_paths: vec!["/health".to_string(), "/login".to_string()],
        }
    }
    
    pub fn exclude(mut self, path: &str) -> Self {
        self.exclude_paths.push(path.to_string());
        self
    }
}

// Step 2: Implement Transform trait (factory สร้าง Service)
impl<S, B> Transform<S, ServiceRequest> for Authentication
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = AuthenticationMiddleware<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(AuthenticationMiddleware {
            service: Rc::new(RefCell::new(service)),
            exclude_paths: self.exclude_paths.clone(),
        }))
    }
}

// Step 3: สร้าง Middleware Service struct
pub struct AuthenticationMiddleware<S> {
    service: Rc<RefCell<S>>,
    exclude_paths: Vec<String>,
}

// Step 4: Implement Service trait
impl<S, B> Service<ServiceRequest> for AuthenticationMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;
    
    forward_ready!(service);
    
    fn call(&self, req: ServiceRequest) -> Self::Future {
        let path = req.path().to_string();
        let exclude_paths = self.exclude_paths.clone();
        let service = self.service.clone();
        
        Box::pin(async move {
            // ตรวจสอบว่า path นี้ต้อง authenticate หรือไม่
            if exclude_paths.iter().any(|p| path.starts_with(p.as_str())) {
                return service.borrow_mut().call(req).await;
            }
            
            // ตรวจสอบ Authorization header
            let auth_header = req.headers()
                .get("Authorization")
                .and_then(|v| v.to_str().ok());
            
            match auth_header {
                Some(token) if token.starts_with("Bearer ") => {
                    let token = &token[7..];  // Remove "Bearer " prefix
                    
                    // ตรวจสอบ token (ในการใช้งานจริงจะ verify JWT)
                    if token == "valid-token-123" {
                        // Token valid - ดำเนินการต่อ
                        service.borrow_mut().call(req).await
                    } else {
                        // Token invalid
                        let (req, _) = req.into_parts();
                        let response = HttpResponse::Unauthorized()
                            .json(serde_json::json!({
                                "error": "invalid_token",
                                "message": "The provided token is invalid"
                            }));
                        Ok(ServiceResponse::new(req, response).map_into_right_body())
                    }
                },
                _ => {
                    // No token
                    let (req, _) = req.into_parts();
                    let response = HttpResponse::Unauthorized()
                        .json(serde_json::json!({
                            "error": "missing_token",
                            "message": "Authorization header is required"
                        }));
                    Ok(ServiceResponse::new(req, response).map_into_right_body())
                }
            }
        })
    }
}

// การใช้งาน Authentication middleware
async fn protected_handler() -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({"message": "Protected resource"}))
}

async fn public_handler() -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({"message": "Public resource"}))
}
```

---

## Request ID Middleware

Middleware สำหรับเพิ่ม unique ID ให้แต่ละ request

```rust
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpMessage,
};
use futures_util::future::LocalBoxFuture;
use std::future::{ready, Ready};
use std::rc::Rc;
use std::cell::RefCell;
use uuid::Uuid;

// Request ID extension สำหรับเก็บ request ID
#[derive(Clone)]
pub struct RequestId(pub String);

// Middleware struct
pub struct RequestIdMiddleware;

impl<S, B> Transform<S, ServiceRequest> for RequestIdMiddleware
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = RequestIdMiddlewareService<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(RequestIdMiddlewareService {
            service: Rc::new(RefCell::new(service)),
        }))
    }
}

pub struct RequestIdMiddlewareService<S> {
    service: Rc<RefCell<S>>,
}

impl<S, B> Service<ServiceRequest> for RequestIdMiddlewareService<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;
    
    forward_ready!(service);
    
    fn call(&self, req: ServiceRequest) -> Self::Future {
        let service = self.service.clone();
        
        Box::pin(async move {
            // ดึง request ID จาก header หรือสร้างใหม่
            let request_id = req.headers()
                .get("X-Request-ID")
                .and_then(|v| v.to_str().ok())
                .map(String::from)
                .unwrap_or_else(|| Uuid::new_v4().to_string());
            
            // เก็บ request ID ใน extensions
            req.extensions_mut().insert(RequestId(request_id.clone()));
            
            // ดำเนินการต่อ
            let mut res = service.borrow_mut().call(req).await?;
            
            // เพิ่ม request ID ใน response header
            res.headers_mut().insert(
                actix_web::http::header::HeaderName::from_static("x-request-id"),
                actix_web::http::header::HeaderValue::from_str(&request_id).unwrap(),
            );
            
            Ok(res)
        })
    }
}

// Handler ที่ใช้ request ID
use actix_web::HttpRequest;

async fn handler_with_request_id(req: HttpRequest) -> HttpResponse {
    // ดึง request ID จาก extensions
    let request_id = req.extensions()
        .get::<RequestId>()
        .map(|id| id.0.clone())
        .unwrap_or_else(|| "unknown".to_string());
    
    log::info!("[{}] Processing request", request_id);
    
    HttpResponse::Ok().json(serde_json::json!({
        "request_id": request_id,
        "message": "Request processed successfully"
    }))
}
```

---

## Timing Middleware

Middleware สำหรับวัดเวลาที่ใช้ในการ process request

```rust
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error,
};
use futures_util::future::LocalBoxFuture;
use std::future::{ready, Ready};
use std::rc::Rc;
use std::cell::RefCell;
use std::time::Instant;

// Timing middleware
pub struct TimingMiddleware {
    slow_request_threshold_ms: u128,
}

impl TimingMiddleware {
    pub fn new() -> Self {
        TimingMiddleware {
            slow_request_threshold_ms: 1000,  // 1 second
        }
    }
    
    pub fn slow_threshold(mut self, threshold_ms: u128) -> Self {
        self.slow_request_threshold_ms = threshold_ms;
        self
    }
}

impl<S, B> Transform<S, ServiceRequest> for TimingMiddleware
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = TimingMiddlewareService<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(TimingMiddlewareService {
            service: Rc::new(RefCell::new(service)),
            slow_threshold_ms: self.slow_request_threshold_ms,
        }))
    }
}

pub struct TimingMiddlewareService<S> {
    service: Rc<RefCell<S>>,
    slow_threshold_ms: u128,
}

impl<S, B> Service<ServiceRequest> for TimingMiddlewareService<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;
    
    forward_ready!(service);
    
    fn call(&self, req: ServiceRequest) -> Self::Future {
        let service = self.service.clone();
        let slow_threshold = self.slow_threshold_ms;
        let method = req.method().to_string();
        let path = req.path().to_string();
        let start_time = Instant::now();
        
        Box::pin(async move {
            let res = service.borrow_mut().call(req).await?;
            
            let duration = start_time.elapsed();
            let duration_ms = duration.as_millis();
            
            // Log timing information
            if duration_ms >= slow_threshold {
                log::warn!(
                    "SLOW REQUEST: {} {} took {}ms (threshold: {}ms)",
                    method, path, duration_ms, slow_threshold
                );
            } else {
                log::debug!("{} {} completed in {}ms", method, path, duration_ms);
            }
            
            // เพิ่ม timing header ใน response
            let mut res = res;
            res.headers_mut().insert(
                actix_web::http::header::HeaderName::from_static("x-response-time"),
                actix_web::http::header::HeaderValue::from_str(
                    &format!("{}ms", duration_ms)
                ).unwrap(),
            );
            
            Ok(res)
        })
    }
}

// Metrics collector สำหรับเก็บ statistics
use std::sync::{Arc, Mutex};
use std::collections::HashMap;

#[derive(Default, Clone)]
pub struct RequestMetrics {
    pub total_requests: u64,
    pub total_duration_ms: u128,
    pub path_counts: HashMap<String, u64>,
    pub status_counts: HashMap<u16, u64>,
}

pub type SharedMetrics = Arc<Mutex<RequestMetrics>>;

// Metrics middleware
pub struct MetricsMiddleware {
    metrics: SharedMetrics,
}

impl MetricsMiddleware {
    pub fn new(metrics: SharedMetrics) -> Self {
        MetricsMiddleware { metrics }
    }
}

impl<S, B> Transform<S, ServiceRequest> for MetricsMiddleware
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = MetricsMiddlewareService<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(MetricsMiddlewareService {
            service: Rc::new(RefCell::new(service)),
            metrics: self.metrics.clone(),
        }))
    }
}

pub struct MetricsMiddlewareService<S> {
    service: Rc<RefCell<S>>,
    metrics: SharedMetrics,
}

impl<S, B> Service<ServiceRequest> for MetricsMiddlewareService<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;
    
    forward_ready!(service);
    
    fn call(&self, req: ServiceRequest) -> Self::Future {
        let service = self.service.clone();
        let metrics = self.metrics.clone();
        let path = req.path().to_string();
        let start = Instant::now();
        
        Box::pin(async move {
            let res = service.borrow_mut().call(req).await?;
            let duration_ms = start.elapsed().as_millis();
            let status = res.status().as_u16();
            
            // Update metrics
            let mut m = metrics.lock().unwrap();
            m.total_requests += 1;
            m.total_duration_ms += duration_ms;
            *m.path_counts.entry(path).or_insert(0) += 1;
            *m.status_counts.entry(status).or_insert(0) += 1;
            
            Ok(res)
        })
    }
}

// Handler สำหรับดู metrics
use actix_web::web;

async fn get_metrics(metrics: web::Data<SharedMetrics>) -> HttpResponse {
    let m = metrics.lock().unwrap();
    let avg_duration = if m.total_requests > 0 {
        m.total_duration_ms as f64 / m.total_requests as f64
    } else {
        0.0
    };
    
    HttpResponse::Ok().json(serde_json::json!({
        "total_requests": m.total_requests,
        "average_duration_ms": avg_duration,
        "paths": m.path_counts,
        "status_codes": m.status_counts
    }))
}
```

---

## Error Handling Middleware

Middleware สำหรับจัดการ errors ทั้งหมดอย่าง consistent

```rust
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    body::EitherBody,
    Error, HttpResponse,
};
use futures_util::future::LocalBoxFuture;
use std::future::{ready, Ready};
use std::rc::Rc;
use std::cell::RefCell;

// Error response structure
#[derive(serde::Serialize)]
struct ErrorResponse {
    success: bool,
    error: String,
    message: String,
    status_code: u16,
    #[serde(skip_serializing_if = "Option::is_none")]
    request_id: Option<String>,
    timestamp: String,
}

// Error handling middleware
pub struct ErrorHandlingMiddleware;

impl<S, B> Transform<S, ServiceRequest> for ErrorHandlingMiddleware
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<EitherBody<B>>;
    type Error = Error;
    type InitError = ();
    type Transform = ErrorHandlingMiddlewareService<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(ErrorHandlingMiddlewareService {
            service: Rc::new(service),
        }))
    }
}

pub struct ErrorHandlingMiddlewareService<S> {
    service: Rc<S>,
}

impl<S, B> Service<ServiceRequest> for ErrorHandlingMiddlewareService<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<EitherBody<B>>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;
    
    forward_ready!(service);
    
    fn call(&self, req: ServiceRequest) -> Self::Future {
        let service = self.service.clone();
        let request_id = req.headers()
            .get("X-Request-ID")
            .and_then(|v| v.to_str().ok())
            .map(String::from);
        
        Box::pin(async move {
            let result = service.call(req).await;
            
            match result {
                Ok(res) => {
                    let status = res.status();
                    
                    // ตรวจสอบว่าเป็น error status code หรือไม่
                    if status.is_client_error() || status.is_server_error() {
                        let status_code = status.as_u16();
                        let error_type = if status.is_client_error() { 
                            "client_error" 
                        } else { 
                            "server_error" 
                        };
                        
                        let message = match status_code {
                            400 => "Bad Request",
                            401 => "Unauthorized",
                            403 => "Forbidden",
                            404 => "Not Found",
                            405 => "Method Not Allowed",
                            409 => "Conflict",
                            422 => "Unprocessable Entity",
                            429 => "Too Many Requests",
                            500 => "Internal Server Error",
                            502 => "Bad Gateway",
                            503 => "Service Unavailable",
                            _ => "Error",
                        };
                        
                        // Log errors
                        if status.is_server_error() {
                            log::error!("Server error {} for request {:?}", status_code, request_id);
                        } else {
                            log::warn!("Client error {} for request {:?}", status_code, request_id);
                        }
                        
                        // สร้าง structured error response
                        let (req_parts, _original_body) = res.into_parts();
                        let error_response = HttpResponse::build(status)
                            .json(ErrorResponse {
                                success: false,
                                error: error_type.to_string(),
                                message: message.to_string(),
                                status_code,
                                request_id,
                                timestamp: chrono::Utc::now().to_rfc3339(),
                            });
                        
                        Ok(ServiceResponse::new(req_parts, error_response).map_into_right_body())
                    } else {
                        Ok(res.map_into_left_body())
                    }
                },
                Err(err) => {
                    log::error!("Unhandled error: {}", err);
                    Err(err)
                }
            }
        })
    }
}
```

---

## Ordering of Middleware

ลำดับของ middleware สำคัญมาก - middleware ที่ใส่ก่อนจะ wrap อยู่รอบนอก

```rust
use actix_web::{App, HttpServer, web, HttpResponse, middleware};

async fn demo_handler() -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({"status": "ok"}))
}

// ตัวอย่าง middleware ง่ายๆ เพื่อแสดงลำดับ
pub struct OrderDemo(pub &'static str);

impl<S, B> Transform<S, ServiceRequest> for OrderDemo
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = OrderDemoService<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(OrderDemoService {
            service: Rc::new(RefCell::new(service)),
            name: self.0,
        }))
    }
}

pub struct OrderDemoService<S> {
    service: Rc<RefCell<S>>,
    name: &'static str,
}

impl<S, B> Service<ServiceRequest> for OrderDemoService<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;
    
    forward_ready!(service);
    
    fn call(&self, req: ServiceRequest) -> Self::Future {
        let service = self.service.clone();
        let name = self.name;
        
        Box::pin(async move {
            println!("[{}] Before handler", name);
            let res = service.borrow_mut().call(req).await?;
            println!("[{}] After handler", name);
            Ok(res)
        })
    }
}

// การใช้งาน - สังเกตลำดับการทำงาน
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    env_logger::init_from_env(Env::default().default_filter_or("info"));
    
    let metrics: SharedMetrics = Arc::new(Mutex::new(RequestMetrics::default()));
    let metrics_clone = metrics.clone();
    
    HttpServer::new(move || {
        // ลำดับ middleware เมื่อ request เข้ามา (จากนอกสุดเข้าใน):
        // 1. ErrorHandling (ห่อทุกอย่าง)
        // 2. Logger (log request/response)
        // 3. Compress (compress response)
        // 4. TimingMiddleware (วัดเวลา)
        // 5. RequestIdMiddleware (เพิ่ม request ID)
        // 6. Handler
        //
        // เมื่อ response ออกมา ลำดับจะกลับกัน:
        // Handler → RequestIdMiddleware → TimingMiddleware → Compress → Logger → ErrorHandling
        
        App::new()
            .app_data(web::Data::new(metrics_clone.clone()))
            // Middleware ลำดับ: ใส่หลังสุด = ทำงานก่อนสุด (รอบในสุด)
            // ใส่ก่อน = ทำงานหลังสุด (รอบนอกสุด)
            
            // รอบนอกสุด - จัดการ error ทั้งหมด
            .wrap(ErrorHandlingMiddleware)
            
            // Logger
            .wrap(middleware::Logger::default())
            
            // Compression
            .wrap(Compress::default())
            
            // Timing
            .wrap(TimingMiddleware::new().slow_threshold(500))
            
            // Request ID - ทำงานรอบในสุด (ใกล้ handler ที่สุด)
            .wrap(RequestIdMiddleware)
            
            // Routes
            .route("/", web::get().to(demo_handler))
            .route("/metrics", web::get().to(get_metrics))
            .service(
                web::scope("/api")
                    // Middleware เฉพาะ scope นี้
                    .wrap(Authentication::new().exclude("/api/public"))
                    .route("/protected", web::get().to(handler_with_request_id))
                    .route("/public", web::get().to(public_handler))
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## ตัวอย่างสมบูรณ์: Rate Limiting Middleware

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::time::{Duration, Instant};

#[derive(Clone)]
struct RateLimitEntry {
    count: u32,
    window_start: Instant,
}

pub struct RateLimiter {
    max_requests: u32,
    window_duration: Duration,
    store: Arc<Mutex<HashMap<String, RateLimitEntry>>>,
}

impl RateLimiter {
    pub fn new(max_requests: u32, window_seconds: u64) -> Self {
        RateLimiter {
            max_requests,
            window_duration: Duration::from_secs(window_seconds),
            store: Arc::new(Mutex::new(HashMap::new())),
        }
    }
}

impl<S, B> Transform<S, ServiceRequest> for RateLimiter
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = RateLimiterService<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(RateLimiterService {
            service: Rc::new(RefCell::new(service)),
            max_requests: self.max_requests,
            window_duration: self.window_duration,
            store: self.store.clone(),
        }))
    }
}

pub struct RateLimiterService<S> {
    service: Rc<RefCell<S>>,
    max_requests: u32,
    window_duration: Duration,
    store: Arc<Mutex<HashMap<String, RateLimitEntry>>>,
}

impl<S, B> Service<ServiceRequest> for RateLimiterService<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;
    
    forward_ready!(service);
    
    fn call(&self, req: ServiceRequest) -> Self::Future {
        let service = self.service.clone();
        let max_requests = self.max_requests;
        let window_duration = self.window_duration;
        let store = self.store.clone();
        
        // ดึง IP address จาก request
        let client_ip = req.peer_addr()
            .map(|addr| addr.ip().to_string())
            .unwrap_or_else(|| "unknown".to_string());
        
        Box::pin(async move {
            let now = Instant::now();
            let allowed;
            let remaining;
            let reset_after;
            
            {
                let mut store = store.lock().unwrap();
                
                let entry = store.entry(client_ip.clone()).or_insert(RateLimitEntry {
                    count: 0,
                    window_start: now,
                });
                
                // ตรวจสอบว่า window หมดอายุหรือยัง
                if now.duration_since(entry.window_start) >= window_duration {
                    entry.count = 0;
                    entry.window_start = now;
                }
                
                entry.count += 1;
                allowed = entry.count <= max_requests;
                remaining = if allowed { max_requests - entry.count } else { 0 };
                reset_after = window_duration.checked_sub(now.duration_since(entry.window_start))
                    .unwrap_or(Duration::ZERO)
                    .as_secs();
            }
            
            if !allowed {
                let (req_parts, _) = req.into_parts();
                let response = HttpResponse::TooManyRequests()
                    .append_header(("X-RateLimit-Limit", max_requests.to_string()))
                    .append_header(("X-RateLimit-Remaining", "0"))
                    .append_header(("X-RateLimit-Reset", reset_after.to_string()))
                    .append_header(("Retry-After", reset_after.to_string()))
                    .json(serde_json::json!({
                        "error": "rate_limit_exceeded",
                        "message": format!("Too many requests. Limit is {} per {} seconds", max_requests, window_duration.as_secs()),
                        "retry_after": reset_after
                    }));
                
                return Ok(ServiceResponse::new(req_parts, response).map_into_right_body());
            }
            
            let mut res = service.borrow_mut().call(req).await?;
            
            // เพิ่ม rate limit headers
            res.headers_mut().insert(
                actix_web::http::header::HeaderName::from_static("x-ratelimit-limit"),
                actix_web::http::header::HeaderValue::from_str(&max_requests.to_string()).unwrap(),
            );
            res.headers_mut().insert(
                actix_web::http::header::HeaderName::from_static("x-ratelimit-remaining"),
                actix_web::http::header::HeaderValue::from_str(&remaining.to_string()).unwrap(),
            );
            
            Ok(res.map_into_left_body())
        })
    }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Middleware concept** - วิธีการทำงานของ middleware ใน Actix-web
2. **Logger middleware** - การ log requests/responses
3. **Compression** - การบีบอัด response
4. **Custom middleware** - การสร้าง middleware ด้วย `Transform` + `Service` traits
5. **Request ID** - การเพิ่ม unique ID ให้แต่ละ request
6. **Timing** - การวัดเวลาและเก็บ metrics
7. **Error handling** - การจัดการ errors อย่าง consistent
8. **Ordering** - ลำดับการทำงานของ middleware
9. **Rate limiting** - ตัวอย่าง middleware ที่ใช้งานได้จริง

---

## การนำทาง

- [← Part 023: JSON Request/Response](../part_023/README.md)
- [→ Part 025: State Management](../part_025/README.md)
- [กลับหน้าหลัก](../../README.md)
