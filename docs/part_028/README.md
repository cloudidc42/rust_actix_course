# Part 028: CORS and Security Headers ใน Actix-web

## สารบัญ
- [แนะนำ CORS](#แนะนำ-cors)
- [actix-cors Crate](#actix-cors-crate)
- [CORS Configuration](#cors-configuration)
- [Wildcard vs Specific Origins](#wildcard-vs-specific-origins)
- [Security Headers](#security-headers)
- [actix-web Middleware สำหรับ Security Headers](#actix-web-middleware-สำหรับ-security-headers)
- [Rate Limiting Concepts](#rate-limiting-concepts)
- [HTTPS Redirect](#https-redirect)

---

## แนะนำ CORS

CORS (Cross-Origin Resource Sharing) คือ security mechanism ที่ browser ใช้ป้องกันการเรียก API จาก origin ที่ไม่ได้รับอนุญาต

### ทำไมต้องมี CORS?

```
Frontend: https://myapp.com
API:       https://api.myapp.com

Browser จะบล็อก request จาก myapp.com ไปหา api.myapp.com
ถ้า api.myapp.com ไม่มี CORS headers ที่อนุญาต
```

### Dependencies

```toml
[dependencies]
actix-web = "4"
actix-cors = "0.7"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
```

---

## actix-cors Crate

`actix-cors` ให้ CORS middleware สำหรับ Actix-web

```rust
use actix_cors::Cors;
use actix_web::{web, App, HttpServer, HttpResponse, http};

// CORS แบบง่ายที่สุด (สำหรับ development)
async fn simple_cors_example() -> std::io::Result<()> {
    HttpServer::new(|| {
        let cors = Cors::permissive();  // อนุญาตทุก origin (ใช้ใน development เท่านั้น!)
        
        App::new()
            .wrap(cors)
            .route("/api/data", web::get().to(|| async {
                HttpResponse::Ok().json(serde_json::json!({"data": "hello"}))
            }))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}

// CORS แบบ default (strict)
async fn default_cors_example() -> std::io::Result<()> {
    HttpServer::new(|| {
        let cors = Cors::default()
            .allowed_origin("https://myapp.com")
            .allowed_methods(vec!["GET", "POST", "PUT", "DELETE"])
            .allowed_headers(vec![
                http::header::AUTHORIZATION,
                http::header::ACCEPT,
                http::header::CONTENT_TYPE,
            ])
            .max_age(3600);
        
        App::new()
            .wrap(cors)
            .route("/api/data", web::get().to(|| async {
                HttpResponse::Ok().json(serde_json::json!({"data": "hello"}))
            }))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## CORS Configuration

การ configure CORS อย่างละเอียด

```rust
use actix_cors::Cors;
use actix_web::{web, App, HttpServer, http};

// Configuration สำหรับ production
fn production_cors() -> Cors {
    let allowed_origins = vec![
        "https://myapp.com",
        "https://www.myapp.com",
        "https://admin.myapp.com",
    ];
    
    let mut cors = Cors::default();
    
    for origin in &allowed_origins {
        cors = cors.allowed_origin(origin);
    }
    
    cors
        // HTTP methods ที่อนุญาต
        .allowed_methods(vec!["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"])
        
        // Headers ที่ client สามารถส่งได้
        .allowed_headers(vec![
            http::header::AUTHORIZATION,
            http::header::ACCEPT,
            http::header::CONTENT_TYPE,
            http::header::HeaderName::from_static("x-request-id"),
            http::header::HeaderName::from_static("x-api-key"),
        ])
        
        // Headers ที่ client สามารถอ่านจาก response ได้
        .expose_headers(vec![
            http::header::HeaderName::from_static("x-request-id"),
            http::header::HeaderName::from_static("x-ratelimit-limit"),
            http::header::HeaderName::from_static("x-ratelimit-remaining"),
            http::header::HeaderName::from_static("x-total-count"),
        ])
        
        // อนุญาต credentials (cookies, authorization headers)
        .supports_credentials()
        
        // Cache preflight request เป็นเวลา 1 ชั่วโมง
        .max_age(3600)
}

// Configuration สำหรับ development
fn development_cors() -> Cors {
    Cors::default()
        .allowed_origin("http://localhost:3000")
        .allowed_origin("http://localhost:5173")  // Vite
        .allowed_origin("http://localhost:4200")  // Angular
        .allowed_origin("http://127.0.0.1:3000")
        .allowed_methods(vec!["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"])
        .allowed_headers(vec![
            http::header::AUTHORIZATION,
            http::header::ACCEPT,
            http::header::CONTENT_TYPE,
        ])
        .supports_credentials()
        .max_age(3600)
}

// Configuration ตาม environment
fn configure_cors() -> Cors {
    let env = std::env::var("APP_ENV").unwrap_or_else(|_| "development".to_string());
    
    match env.as_str() {
        "production" => production_cors(),
        "staging" => {
            Cors::default()
                .allowed_origin("https://staging.myapp.com")
                .allowed_methods(vec!["GET", "POST", "PUT", "DELETE"])
                .allowed_headers(vec![
                    http::header::AUTHORIZATION,
                    http::header::CONTENT_TYPE,
                ])
                .max_age(3600)
        },
        _ => development_cors(),
    }
}

// CORS กับ allowed_origin_fn (dynamic origin checking)
fn dynamic_cors() -> Cors {
    Cors::default()
        .allowed_origin_fn(|origin, _req_head| {
            // origin คือ bytes
            let origin_str = std::str::from_utf8(origin.as_bytes()).unwrap_or("");
            
            // อนุญาต origins ที่ match pattern
            origin_str.ends_with(".myapp.com") ||
            origin_str == "https://partner.example.com" ||
            origin_str.starts_with("http://localhost:")
        })
        .allowed_methods(vec!["GET", "POST"])
        .allowed_headers(vec![
            http::header::CONTENT_TYPE,
            http::header::AUTHORIZATION,
        ])
        .max_age(3600)
}

// ตัวอย่าง App setup
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .wrap(configure_cors())
            .service(
                web::scope("/api/v1")
                    .route("/users", web::get().to(|| async {
                        HttpResponse::Ok().json(serde_json::json!({"users": []}))
                    }))
                    .route("/users", web::post().to(|| async {
                        HttpResponse::Created().finish()
                    }))
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## Wildcard vs Specific Origins

การเลือกระหว่าง wildcard (`*`) และ specific origins

```rust
use actix_cors::Cors;
use actix_web::http;

// ❌ Wildcard - ไม่ปลอดภัยสำหรับ production
fn wildcard_cors() -> Cors {
    // "Access-Control-Allow-Origin: *"
    // ปัญหา:
    // 1. ไม่สามารถใช้ supports_credentials() ได้
    // 2. ทุก website สามารถ call API ได้
    // 3. ไม่เหมาะกับ API ที่มี authentication
    Cors::default()
        .allow_any_origin()  // = "*"
        .allowed_methods(vec!["GET"])
}

// ✅ Specific Origins - ปลอดภัยสำหรับ production
fn specific_origins_cors() -> Cors {
    Cors::default()
        .allowed_origin("https://myapp.com")
        .allowed_origin("https://www.myapp.com")
        .supports_credentials()  // ใช้ร่วมกับ specific origins ได้
        .allowed_methods(vec!["GET", "POST", "PUT", "DELETE"])
        .allowed_headers(vec![
            http::header::AUTHORIZATION,
            http::header::CONTENT_TYPE,
        ])
}

// ✅ Public API - อนุญาต wildcard แต่ไม่มี credentials
fn public_api_cors() -> Cors {
    // เหมาะกับ public read-only API เช่น weather API, news API
    Cors::default()
        .allow_any_origin()  // "*"
        .allowed_methods(vec!["GET"])
        // ห้ามใช้ supports_credentials() กับ allow_any_origin()!
}

// Preflight request คืออะไร?
// Browser จะส่ง OPTIONS request ก่อน request จริง
// เพื่อตรวจสอบว่า server อนุญาต CORS หรือไม่
//
// Preflight:
// OPTIONS /api/users HTTP/1.1
// Origin: https://myapp.com
// Access-Control-Request-Method: POST
// Access-Control-Request-Headers: Content-Type, Authorization
//
// Response:
// Access-Control-Allow-Origin: https://myapp.com
// Access-Control-Allow-Methods: GET, POST, PUT, DELETE
// Access-Control-Allow-Headers: Content-Type, Authorization
// Access-Control-Max-Age: 3600
```

---

## Security Headers

Security headers ช่วยป้องกัน attacks ต่างๆ

```rust
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error,
    http::header,
};
use futures_util::future::LocalBoxFuture;
use std::future::{ready, Ready};
use std::rc::Rc;
use std::cell::RefCell;

// Security Headers Middleware
pub struct SecurityHeaders {
    config: SecurityConfig,
}

#[derive(Clone)]
pub struct SecurityConfig {
    // Content Security Policy
    csp: Option<String>,
    // HTTP Strict Transport Security
    hsts: Option<HstsConfig>,
    // X-Frame-Options
    frame_options: FrameOptions,
    // X-Content-Type-Options
    content_type_options: bool,
    // X-XSS-Protection  
    xss_protection: bool,
    // Referrer-Policy
    referrer_policy: Option<String>,
    // Permissions-Policy
    permissions_policy: Option<String>,
}

#[derive(Clone)]
pub struct HstsConfig {
    max_age: u64,
    include_subdomains: bool,
    preload: bool,
}

#[derive(Clone)]
pub enum FrameOptions {
    Deny,
    SameOrigin,
    Allow(String),  // Allow from specific origin
}

impl SecurityConfig {
    pub fn default() -> Self {
        SecurityConfig {
            csp: Some(
                "default-src 'self'; \
                 img-src 'self' data: https:; \
                 script-src 'self'; \
                 style-src 'self' 'unsafe-inline'; \
                 font-src 'self'; \
                 connect-src 'self'; \
                 frame-ancestors 'none'".to_string()
            ),
            hsts: Some(HstsConfig {
                max_age: 31536000,  // 1 year
                include_subdomains: true,
                preload: false,
            }),
            frame_options: FrameOptions::Deny,
            content_type_options: true,
            xss_protection: true,
            referrer_policy: Some("strict-origin-when-cross-origin".to_string()),
            permissions_policy: Some(
                "camera=(), microphone=(), geolocation=(), payment=()".to_string()
            ),
        }
    }
    
    pub fn api() -> Self {
        SecurityConfig {
            csp: None,  // API ไม่ต้องการ CSP
            hsts: Some(HstsConfig {
                max_age: 31536000,
                include_subdomains: true,
                preload: false,
            }),
            frame_options: FrameOptions::Deny,
            content_type_options: true,
            xss_protection: false,  // API ไม่ต้องการ XSS protection
            referrer_policy: Some("no-referrer".to_string()),
            permissions_policy: None,
        }
    }
}

impl SecurityHeaders {
    pub fn new(config: SecurityConfig) -> Self {
        SecurityHeaders { config }
    }
    
    pub fn default_web() -> Self {
        SecurityHeaders::new(SecurityConfig::default())
    }
    
    pub fn for_api() -> Self {
        SecurityHeaders::new(SecurityConfig::api())
    }
}

impl<S, B> Transform<S, ServiceRequest> for SecurityHeaders
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = SecurityHeadersService<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(SecurityHeadersService {
            service: Rc::new(RefCell::new(service)),
            config: self.config.clone(),
        }))
    }
}

pub struct SecurityHeadersService<S> {
    service: Rc<RefCell<S>>,
    config: SecurityConfig,
}

impl<S, B> Service<ServiceRequest> for SecurityHeadersService<S>
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
        let config = self.config.clone();
        
        Box::pin(async move {
            let mut res = service.borrow_mut().call(req).await?;
            let headers = res.headers_mut();
            
            // X-Content-Type-Options: ป้องกัน MIME sniffing
            if config.content_type_options {
                headers.insert(
                    header::HeaderName::from_static("x-content-type-options"),
                    header::HeaderValue::from_static("nosniff"),
                );
            }
            
            // X-Frame-Options: ป้องกัน clickjacking
            let frame_value = match &config.frame_options {
                FrameOptions::Deny => "DENY",
                FrameOptions::SameOrigin => "SAMEORIGIN",
                FrameOptions::Allow(origin) => origin.as_str(),
            };
            headers.insert(
                header::HeaderName::from_static("x-frame-options"),
                header::HeaderValue::from_str(frame_value).unwrap(),
            );
            
            // X-XSS-Protection: เปิด XSS filter ใน browser เก่า
            if config.xss_protection {
                headers.insert(
                    header::HeaderName::from_static("x-xss-protection"),
                    header::HeaderValue::from_static("1; mode=block"),
                );
            }
            
            // Content-Security-Policy
            if let Some(ref csp) = config.csp {
                headers.insert(
                    header::HeaderName::from_static("content-security-policy"),
                    header::HeaderValue::from_str(csp).unwrap(),
                );
            }
            
            // Strict-Transport-Security (HSTS)
            if let Some(ref hsts) = config.hsts {
                let mut hsts_value = format!("max-age={}", hsts.max_age);
                if hsts.include_subdomains {
                    hsts_value.push_str("; includeSubDomains");
                }
                if hsts.preload {
                    hsts_value.push_str("; preload");
                }
                headers.insert(
                    header::HeaderName::from_static("strict-transport-security"),
                    header::HeaderValue::from_str(&hsts_value).unwrap(),
                );
            }
            
            // Referrer-Policy
            if let Some(ref policy) = config.referrer_policy {
                headers.insert(
                    header::HeaderName::from_static("referrer-policy"),
                    header::HeaderValue::from_str(policy).unwrap(),
                );
            }
            
            // Permissions-Policy
            if let Some(ref policy) = config.permissions_policy {
                headers.insert(
                    header::HeaderName::from_static("permissions-policy"),
                    header::HeaderValue::from_str(policy).unwrap(),
                );
            }
            
            // ลบ headers ที่เปิดเผยข้อมูล server
            headers.remove("server");
            headers.remove("x-powered-by");
            
            Ok(res)
        })
    }
}
```

---

## actix-web Middleware สำหรับ Security Headers

ใช้ DefaultHeaders middleware ที่มาพร้อมกับ Actix-web

```rust
use actix_web::{middleware::DefaultHeaders, App, HttpServer, web};

async fn setup_security_with_default_headers() -> std::io::Result<()> {
    HttpServer::new(|| {
        // DefaultHeaders เป็น middleware ที่ง่ายที่สุดในการเพิ่ม headers
        let security_headers = DefaultHeaders::new()
            .add(("X-Content-Type-Options", "nosniff"))
            .add(("X-Frame-Options", "DENY"))
            .add(("X-XSS-Protection", "1; mode=block"))
            .add(("Referrer-Policy", "strict-origin-when-cross-origin"))
            .add(("Permissions-Policy", "camera=(), microphone=(), geolocation=()"))
            .add(("Strict-Transport-Security", "max-age=31536000; includeSubDomains"))
            .add(("Content-Security-Policy", 
                "default-src 'self'; img-src 'self' data:; script-src 'self'; style-src 'self' 'unsafe-inline'"))
            // ซ่อน server information
            .add(("Server", ""))
            .add(("X-Powered-By", ""));
        
        App::new()
            .wrap(security_headers)
            .wrap(configure_cors())
            .route("/", web::get().to(|| async {
                actix_web::HttpResponse::Ok().body("Hello, secure world!")
            }))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}

// หรือใช้ Security Headers middleware ที่สร้างเอง
async fn setup_with_custom_security() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .wrap(SecurityHeaders::for_api())  // ใช้ API config
            .wrap(configure_cors())
            .service(
                web::scope("/api/v1")
                    .route("/data", web::get().to(|| async {
                        actix_web::HttpResponse::Ok().json(
                            serde_json::json!({"data": "secure"})
                        )
                    }))
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}

// CORS + Security Headers ทำงานร่วมกัน
async fn full_security_setup() -> std::io::Result<()> {
    use actix_cors::Cors;
    
    HttpServer::new(|| {
        let cors = Cors::default()
            .allowed_origin("https://myapp.com")
            .allowed_methods(vec!["GET", "POST", "PUT", "DELETE"])
            .allowed_headers(vec![
                actix_web::http::header::AUTHORIZATION,
                actix_web::http::header::CONTENT_TYPE,
            ])
            .supports_credentials()
            .max_age(3600);
        
        let security = DefaultHeaders::new()
            .add(("X-Content-Type-Options", "nosniff"))
            .add(("X-Frame-Options", "DENY"))
            .add(("Strict-Transport-Security", "max-age=31536000; includeSubDomains"));
        
        App::new()
            .wrap(cors)
            .wrap(security)
            .route("/api/health", web::get().to(|| async {
                actix_web::HttpResponse::Ok().json(serde_json::json!({"status": "healthy"}))
            }))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## Rate Limiting Concepts

แนวคิด rate limiting สำหรับ API security

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::time::{Duration, Instant};
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpResponse,
};
use futures_util::future::LocalBoxFuture;
use std::future::{ready, Ready};
use std::rc::Rc;
use std::cell::RefCell;

// Token Bucket Algorithm
// แต่ละ IP มี "bucket" ที่ token จะเพิ่มขึ้นเรื่อยๆ
// แต่ละ request จะใช้ token 1 ตัว
struct TokenBucket {
    tokens: f64,
    max_tokens: f64,
    refill_rate: f64,  // tokens per second
    last_refill: Instant,
}

impl TokenBucket {
    fn new(max_tokens: f64, refill_rate: f64) -> Self {
        TokenBucket {
            tokens: max_tokens,
            max_tokens,
            refill_rate,
            last_refill: Instant::now(),
        }
    }
    
    fn try_consume(&mut self) -> bool {
        self.refill();
        if self.tokens >= 1.0 {
            self.tokens -= 1.0;
            true
        } else {
            false
        }
    }
    
    fn refill(&mut self) {
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_refill).as_secs_f64();
        self.tokens = (self.tokens + elapsed * self.refill_rate).min(self.max_tokens);
        self.last_refill = now;
    }
    
    fn time_until_next_token(&self) -> f64 {
        if self.tokens >= 1.0 {
            0.0
        } else {
            (1.0 - self.tokens) / self.refill_rate
        }
    }
}

// Sliding Window Algorithm
// นับ requests ใน window ที่เลื่อนไปตามเวลา
struct SlidingWindowCounter {
    requests: Vec<Instant>,
    window_size: Duration,
    max_requests: usize,
}

impl SlidingWindowCounter {
    fn new(max_requests: usize, window_secs: u64) -> Self {
        SlidingWindowCounter {
            requests: vec![],
            window_size: Duration::from_secs(window_secs),
            max_requests,
        }
    }
    
    fn check_and_record(&mut self) -> (bool, usize) {
        let now = Instant::now();
        let cutoff = now - self.window_size;
        
        // ลบ requests ที่เก่าเกิน window
        self.requests.retain(|&t| t > cutoff);
        
        let current_count = self.requests.len();
        
        if current_count < self.max_requests {
            self.requests.push(now);
            (true, self.max_requests - current_count - 1)  // (allowed, remaining)
        } else {
            (false, 0)
        }
    }
    
    fn reset_time(&self) -> Option<Duration> {
        self.requests.first().map(|first| {
            let age = Instant::now().duration_since(*first);
            self.window_size.checked_sub(age).unwrap_or(Duration::ZERO)
        })
    }
}

// Rate Limiter State
struct RateLimiterState {
    buckets: HashMap<String, SlidingWindowCounter>,
    config: RateLimitConfig,
}

#[derive(Clone)]
struct RateLimitConfig {
    requests_per_window: usize,
    window_secs: u64,
    // Different limits for different endpoint types
    read_limit: usize,
    write_limit: usize,
    auth_limit: usize,
}

impl RateLimitConfig {
    fn default() -> Self {
        RateLimitConfig {
            requests_per_window: 100,
            window_secs: 60,
            read_limit: 200,
            write_limit: 50,
            auth_limit: 10,
        }
    }
}

type RateLimiterStore = Arc<Mutex<RateLimiterState>>;

// Rate Limiter Middleware
pub struct RateLimitMiddleware {
    store: RateLimiterStore,
}

impl RateLimitMiddleware {
    pub fn new(config: RateLimitConfig) -> Self {
        RateLimitMiddleware {
            store: Arc::new(Mutex::new(RateLimiterState {
                buckets: HashMap::new(),
                config,
            })),
        }
    }
}

impl<S, B> Transform<S, ServiceRequest> for RateLimitMiddleware
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = RateLimitService<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(RateLimitService {
            service: Rc::new(RefCell::new(service)),
            store: self.store.clone(),
        }))
    }
}

pub struct RateLimitService<S> {
    service: Rc<RefCell<S>>,
    store: RateLimiterStore,
}

impl<S, B> Service<ServiceRequest> for RateLimitService<S>
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
        let store = self.store.clone();
        
        // Key สำหรับ rate limiting (IP address + endpoint)
        let client_ip = req.peer_addr()
            .map(|addr| addr.ip().to_string())
            .unwrap_or_else(|| "unknown".to_string());
        
        let method = req.method().to_string();
        let path_category = categorize_path(req.path());
        let key = format!("{}:{}:{}", client_ip, method, path_category);
        
        Box::pin(async move {
            let (allowed, remaining, retry_after) = {
                let mut state = store.lock().unwrap();
                let window_secs = state.config.window_secs;
                let limit = match path_category.as_str() {
                    "auth" => state.config.auth_limit,
                    "write" => state.config.write_limit,
                    _ => state.config.read_limit,
                };
                
                let counter = state.buckets
                    .entry(key.clone())
                    .or_insert_with(|| SlidingWindowCounter::new(limit, window_secs));
                
                let (allowed, remaining) = counter.check_and_record();
                let retry_after = if !allowed {
                    counter.reset_time().map(|d| d.as_secs()).unwrap_or(window_secs)
                } else {
                    0
                };
                
                (allowed, remaining, retry_after)
            };
            
            if !allowed {
                let (req_parts, _) = req.into_parts();
                let response = HttpResponse::TooManyRequests()
                    .append_header(("X-RateLimit-Remaining", "0"))
                    .append_header(("X-RateLimit-Reset", retry_after.to_string()))
                    .append_header(("Retry-After", retry_after.to_string()))
                    .json(serde_json::json!({
                        "error": "rate_limit_exceeded",
                        "message": "Too many requests",
                        "retry_after": retry_after
                    }));
                
                return Ok(ServiceResponse::new(req_parts, response).map_into_right_body());
            }
            
            let mut res = service.borrow_mut().call(req).await?;
            
            res.headers_mut().insert(
                actix_web::http::header::HeaderName::from_static("x-ratelimit-remaining"),
                actix_web::http::header::HeaderValue::from_str(&remaining.to_string()).unwrap(),
            );
            
            Ok(res.map_into_left_body())
        })
    }
}

fn categorize_path(path: &str) -> String {
    if path.starts_with("/api/auth") || path.starts_with("/api/login") {
        "auth".to_string()
    } else if path.starts_with("/api") {
        "api".to_string()
    } else {
        "read".to_string()
    }
}
```

---

## HTTPS Redirect

การ redirect HTTP ไปยัง HTTPS

```rust
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpResponse,
};
use futures_util::future::LocalBoxFuture;
use std::future::{ready, Ready};
use std::rc::Rc;
use std::cell::RefCell;

// HTTPS Redirect Middleware
pub struct HttpsRedirect {
    enabled: bool,
    https_port: u16,
}

impl HttpsRedirect {
    pub fn new() -> Self {
        HttpsRedirect {
            enabled: std::env::var("FORCE_HTTPS")
                .map(|v| v == "true")
                .unwrap_or(false),
            https_port: std::env::var("HTTPS_PORT")
                .ok()
                .and_then(|p| p.parse().ok())
                .unwrap_or(443),
        }
    }
    
    pub fn enabled(mut self, enabled: bool) -> Self {
        self.enabled = enabled;
        self
    }
}

impl<S, B> Transform<S, ServiceRequest> for HttpsRedirect
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = HttpsRedirectService<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(HttpsRedirectService {
            service: Rc::new(RefCell::new(service)),
            enabled: self.enabled,
            https_port: self.https_port,
        }))
    }
}

pub struct HttpsRedirectService<S> {
    service: Rc<RefCell<S>>,
    enabled: bool,
    https_port: u16,
}

impl<S, B> Service<ServiceRequest> for HttpsRedirectService<S>
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
        let enabled = self.enabled;
        let https_port = self.https_port;
        
        Box::pin(async move {
            if !enabled {
                return service.borrow_mut().call(req).await?.map_into_left_body();
            }
            
            // ตรวจสอบว่า request มาทาง HTTPS แล้วหรือยัง
            let is_https = req.connection_info().scheme() == "https"
                || req.headers().get("X-Forwarded-Proto")
                    .and_then(|v| v.to_str().ok())
                    .map(|v| v == "https")
                    .unwrap_or(false);
            
            if is_https {
                return service.borrow_mut().call(req).await?.map_into_left_body();
            }
            
            // สร้าง HTTPS URL
            let host = req.connection_info().host().to_string();
            let host_without_port = host.split(':').next().unwrap_or(&host);
            let path = req.path();
            let query = req.query_string();
            
            let https_url = if https_port == 443 {
                if query.is_empty() {
                    format!("https://{}{}", host_without_port, path)
                } else {
                    format!("https://{}{}?{}", host_without_port, path, query)
                }
            } else {
                if query.is_empty() {
                    format!("https://{}:{}{}", host_without_port, https_port, path)
                } else {
                    format!("https://{}:{}{}?{}", host_without_port, https_port, path, query)
                }
            };
            
            let (req_parts, _) = req.into_parts();
            let response = HttpResponse::MovedPermanently()
                .append_header(("Location", https_url))
                .finish();
            
            Ok(ServiceResponse::new(req_parts, response).map_into_right_body())
        })
    }
}

// Complete security setup
#[actix_web::main]
async fn main_secure() -> std::io::Result<()> {
    HttpServer::new(|| {
        use actix_cors::Cors;
        use actix_web::http;
        
        let cors = Cors::default()
            .allowed_origin("https://myapp.com")
            .allowed_methods(vec!["GET", "POST", "PUT", "DELETE", "OPTIONS"])
            .allowed_headers(vec![
                http::header::AUTHORIZATION,
                http::header::CONTENT_TYPE,
                http::header::ACCEPT,
            ])
            .expose_headers(vec![
                http::header::HeaderName::from_static("x-ratelimit-limit"),
                http::header::HeaderName::from_static("x-ratelimit-remaining"),
            ])
            .supports_credentials()
            .max_age(3600);
        
        App::new()
            // Security layers (จากนอกสุดเข้าใน)
            .wrap(HttpsRedirect::new())
            .wrap(cors)
            .wrap(SecurityHeaders::default_web())
            .wrap(RateLimitMiddleware::new(RateLimitConfig::default()))
            .wrap(actix_web::middleware::Logger::default())
            
            // Routes
            .service(
                web::scope("/api/v1")
                    .route("/status", web::get().to(|| async {
                        actix_web::HttpResponse::Ok().json(serde_json::json!({
                            "status": "healthy",
                            "timestamp": chrono::Utc::now().to_rfc3339()
                        }))
                    }))
            )
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **CORS** - Cross-Origin Resource Sharing concept และการตั้งค่า
2. **actix-cors** - การใช้ crate สำหรับ CORS middleware
3. **CORS Configuration** - Allowed origins, methods, headers, credentials
4. **Wildcard vs Specific** - เมื่อไหรที่ควรใช้อะไร
5. **Security Headers** - X-Frame-Options, CSP, HSTS, X-XSS-Protection
6. **Custom Middleware** - การสร้าง security header middleware เอง
7. **Rate Limiting** - Token bucket และ sliding window algorithms
8. **HTTPS Redirect** - การบังคับใช้ HTTPS

---

## การนำทาง

- [← Part 027: Static Files and Templates](../part_027/README.md)
- [→ Part 029: Form Data and Multipart](../part_029/README.md)
- [กลับหน้าหลัก](../../README.md)
