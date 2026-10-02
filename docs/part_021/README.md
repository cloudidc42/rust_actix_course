# Part 021: Actix-web: Hello World Server 🌐

## 🎯 เป้าหมายของ Part นี้

- ติดตั้ง Actix-web
- สร้าง HTTP server แรก
- เข้าใจ App, HttpServer, Route
- Async handlers
- Graceful shutdown

---

## 1. ทำไม Actix-web?

```
Actix-web เป็น:
✅ Web framework ที่เร็วที่สุดใน world (TechEmpower benchmarks)
✅ Fully async (Tokio-based)
✅ Type-safe
✅ Ergonomic API
✅ Production-ready
✅ Excellent middleware support

Benchmark (requests/second):
actix-web:    ~600,000 req/s
gin (Go):     ~400,000 req/s
fasthttp:     ~300,000 req/s
express.js:    ~50,000 req/s
django:        ~10,000 req/s
```

---

## 2. Setup โปรเจกต์

### 2.1 สร้างโปรเจกต์ใหม่

```bash
cargo new actix_hello
cd actix_hello
```

### 2.2 Cargo.toml

```toml
[package]
name = "actix_hello"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
env_logger = "0.11"
log = "0.4"
```

---

## 3. Hello World Server

### 3.1 src/main.rs - ขั้นต่ำที่สุด

```rust
use actix_web::{web, App, HttpServer, HttpResponse, get};

#[get("/")]
async fn index() -> HttpResponse {
    HttpResponse::Ok().body("Hello, World!")
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(index)
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

```bash
cargo run
# รัน server บน localhost:8080
curl http://localhost:8080/
# Hello, World!
```

### 3.2 Server ที่สมบูรณ์กว่า

```rust
use actix_web::{
    get, post, put, delete,
    web, App, HttpServer, HttpResponse,
    middleware::Logger,
};
use serde::{Deserialize, Serialize};
use log::info;

// Response types
#[derive(Serialize)]
struct ApiResponse<T: Serialize> {
    success: bool,
    data: T,
    message: String,
}

impl<T: Serialize> ApiResponse<T> {
    fn ok(data: T) -> Self {
        Self {
            success: true,
            data,
            message: String::from("Success"),
        }
    }
}

#[derive(Serialize)]
struct ErrorResponse {
    success: bool,
    error: String,
    code: u16,
}

impl ErrorResponse {
    fn new(error: &str, code: u16) -> Self {
        Self {
            success: false,
            error: error.to_string(),
            code,
        }
    }
}

// ============================================
// Route Handlers
// ============================================

#[get("/")]
async fn index() -> HttpResponse {
    HttpResponse::Ok()
        .content_type("text/html; charset=utf-8")
        .body(r#"
            <html>
            <head><title>Rust API</title></head>
            <body>
                <h1>🦀 Rust Actix-web API</h1>
                <p>Server is running!</p>
                <ul>
                    <li><a href="/health">Health Check</a></li>
                    <li><a href="/api/v1/hello">Hello API</a></li>
                    <li><a href="/api/v1/info">Server Info</a></li>
                </ul>
            </body>
            </html>
        "#)
}

#[get("/health")]
async fn health_check() -> HttpResponse {
    #[derive(Serialize)]
    struct Health {
        status: String,
        version: String,
        uptime_seconds: u64,
    }

    let health = Health {
        status: "healthy".to_string(),
        version: env!("CARGO_PKG_VERSION").to_string(),
        uptime_seconds: 0,  // simplified
    };

    HttpResponse::Ok().json(ApiResponse::ok(health))
}

#[get("/api/v1/hello")]
async fn hello() -> HttpResponse {
    #[derive(Serialize)]
    struct HelloData {
        message: String,
        language: String,
    }

    HttpResponse::Ok().json(ApiResponse::ok(HelloData {
        message: "สวัสดี! Hello from Rust!".to_string(),
        language: "Rust + Actix-web".to_string(),
    }))
}

#[get("/api/v1/hello/{name}")]
async fn hello_name(path: web::Path<String>) -> HttpResponse {
    let name = path.into_inner();

    #[derive(Serialize)]
    struct HelloData {
        message: String,
        name: String,
    }

    HttpResponse::Ok().json(ApiResponse::ok(HelloData {
        message: format!("สวัสดี, {}!", name),
        name,
    }))
}

#[get("/api/v1/info")]
async fn server_info() -> HttpResponse {
    #[derive(Serialize)]
    struct ServerInfo {
        name: String,
        version: String,
        framework: String,
        language: String,
        features: Vec<String>,
    }

    HttpResponse::Ok().json(ApiResponse::ok(ServerInfo {
        name: "Rust API Server".to_string(),
        version: "1.0.0".to_string(),
        framework: "Actix-web 4".to_string(),
        language: "Rust 2021".to_string(),
        features: vec![
            "Async/Await".to_string(),
            "Type-safe routing".to_string(),
            "JSON serialization".to_string(),
            "Middleware support".to_string(),
        ],
    }))
}

#[derive(Deserialize, Serialize, Debug)]
struct EchoRequest {
    message: String,
    repeat: Option<u32>,
}

#[post("/api/v1/echo")]
async fn echo(body: web::Json<EchoRequest>) -> HttpResponse {
    let repeat = body.repeat.unwrap_or(1).min(10);  // max 10 repetitions
    let messages: Vec<String> = (0..repeat)
        .map(|i| format!("[{}] {}", i + 1, body.message))
        .collect();

    #[derive(Serialize)]
    struct EchoResponse {
        original: String,
        echoed: Vec<String>,
        count: u32,
    }

    HttpResponse::Ok().json(ApiResponse::ok(EchoResponse {
        original: body.message.clone(),
        echoed: messages,
        count: repeat,
    }))
}

// 404 handler
async fn not_found() -> HttpResponse {
    HttpResponse::NotFound().json(ErrorResponse::new("Endpoint not found", 404))
}

// ============================================
// Main
// ============================================

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // Initialize logger
    env_logger::init_from_env(env_logger::Env::default().default_filter_or("info"));

    let host = "127.0.0.1";
    let port = 8080;

    info!("Starting server at http://{}:{}", host, port);
    info!("Press Ctrl+C to stop");

    HttpServer::new(|| {
        App::new()
            // Middleware
            .wrap(Logger::new("%r → %s (%Dms)"))
            // Routes
            .service(index)
            .service(health_check)
            .service(hello)
            .service(hello_name)
            .service(server_info)
            .service(echo)
            // Default (404)
            .default_service(web::route().to(not_found))
    })
    .bind((host, port))?
    .workers(4)           // number of worker threads
    .run()
    .await
}
```

---

## 4. ทดสอบ API

### 4.1 ด้วย curl

```bash
# Health check
curl http://localhost:8080/health | python3 -m json.tool

# Hello
curl http://localhost:8080/api/v1/hello

# Hello with name
curl http://localhost:8080/api/v1/hello/สมชาย

# Server info
curl http://localhost:8080/api/v1/info

# Echo (POST)
curl -X POST http://localhost:8080/api/v1/echo \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello!", "repeat": 3}'

# 404
curl http://localhost:8080/not-found
```

### 4.2 ด้วย httpie

```bash
pip install httpie

http GET localhost:8080/health
http GET localhost:8080/api/v1/hello/world
http POST localhost:8080/api/v1/echo message="สวัสดี" repeat:=3
```

---

## 5. HttpResponse Methods

### 5.1 Status Codes

```rust
use actix_web::HttpResponse;

async fn status_examples() -> HttpResponse {
    // 2xx Success
    HttpResponse::Ok()           // 200
    HttpResponse::Created()      // 201
    HttpResponse::Accepted()     // 202
    HttpResponse::NoContent()    // 204

    // 3xx Redirect
    HttpResponse::MovedPermanently()  // 301
    HttpResponse::Found()             // 302
    HttpResponse::PermanentRedirect() // 308

    // 4xx Client Error
    HttpResponse::BadRequest()     // 400
    HttpResponse::Unauthorized()   // 401
    HttpResponse::Forbidden()      // 403
    HttpResponse::NotFound()       // 404
    HttpResponse::Conflict()       // 409
    HttpResponse::UnprocessableEntity() // 422
    HttpResponse::TooManyRequests() // 429

    // 5xx Server Error
    HttpResponse::InternalServerError() // 500
    HttpResponse::BadGateway()          // 502
    HttpResponse::ServiceUnavailable()  // 503

    // Custom status
    HttpResponse::build(actix_web::http::StatusCode::from_u16(418).unwrap())
        .body("I'm a teapot")
}
```

### 5.2 Response Body Types

```rust
use actix_web::HttpResponse;
use serde::Serialize;

#[derive(Serialize)]
struct Data { value: i32 }

async fn body_types() -> HttpResponse {
    // String/bytes body
    HttpResponse::Ok().body("plain text")

    // JSON body
    HttpResponse::Ok().json(Data { value: 42 })

    // Empty body
    HttpResponse::NoContent().finish()

    // HTML
    HttpResponse::Ok()
        .content_type("text/html")
        .body("<h1>Hello</h1>")

    // Custom headers
    HttpResponse::Ok()
        .insert_header(("X-Custom-Header", "value"))
        .insert_header(("Cache-Control", "no-cache"))
        .json(Data { value: 42 })
}
```

---

## 6. App Configuration

### 6.1 การจัดระเบียบ Routes

```rust
use actix_web::{web, App, HttpServer};

mod api {
    pub mod v1 {
        use actix_web::{get, HttpResponse};

        #[get("/users")]
        pub async fn list_users() -> HttpResponse {
            HttpResponse::Ok().body("User list")
        }

        #[get("/products")]
        pub async fn list_products() -> HttpResponse {
            HttpResponse::Ok().body("Product list")
        }

        pub fn configure(cfg: &mut web::ServiceConfig) {
            cfg
                .service(list_users)
                .service(list_products);
        }
    }
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(
                web::scope("/api/v1")
                    .configure(api::v1::configure)
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

### 6.2 Configuration กับ State

```rust
use actix_web::{web, App, HttpServer, HttpResponse, get};
use std::sync::Mutex;

struct AppState {
    app_name: String,
    request_count: Mutex<u64>,
}

#[get("/")]
async fn index(data: web::Data<AppState>) -> HttpResponse {
    let mut count = data.request_count.lock().unwrap();
    *count += 1;
    HttpResponse::Ok().body(format!(
        "{} - Request #{}",
        data.app_name, count
    ))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let data = web::Data::new(AppState {
        app_name: String::from("My Actix App"),
        request_count: Mutex::new(0),
    });

    HttpServer::new(move || {
        App::new()
            .app_data(data.clone())
            .service(index)
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## 7. Graceful Shutdown

```rust
use actix_web::{HttpServer, App, HttpResponse, get};
use actix_web::middleware::Logger;

#[get("/")]
async fn index() -> HttpResponse {
    HttpResponse::Ok().body("Hello!")
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let server = HttpServer::new(|| {
        App::new()
            .wrap(Logger::default())
            .service(index)
    })
    .bind("127.0.0.1:8080")?
    .shutdown_timeout(30)  // รอ requests ที่กำลัง process ให้เสร็จ (30s)
    .run();

    // Handle Ctrl+C
    let srv = server.handle();
    tokio::spawn(async move {
        tokio::signal::ctrl_c().await.expect("Failed to listen for ctrl+c");
        println!("\nShutting down gracefully...");
        srv.stop(true).await;  // true = graceful
    });

    server.await
}
```

---

## 8. สรุปและ Exercises

### 8.1 สิ่งที่เรียนรู้

✅ Setup Actix-web project  
✅ สร้าง HTTP handlers  
✅ Routing (GET, POST, etc.)  
✅ JSON responses  
✅ Path parameters  
✅ Middleware (Logger)  
✅ App state  
✅ Graceful shutdown  

### 8.2 Exercises

**Exercise 1: Calculator API**
```bash
GET /api/add?a=5&b=3         → {"result": 8}
GET /api/subtract?a=10&b=3   → {"result": 7}
POST /api/calculate           → body: {"op": "multiply", "a": 4, "b": 5}
```

**Exercise 2: Todo API (in-memory)**
```bash
GET  /api/todos           → list todos
POST /api/todos           → create todo
GET  /api/todos/{id}      → get todo
PUT  /api/todos/{id}      → update todo
DELETE /api/todos/{id}    → delete todo
```

**Exercise 3: File Server**
```bash
GET /files/{filename}     → serve files from ./public/
```

---

## 9. ตัวอย่างโปรเจกต์สมบูรณ์

```
actix_hello/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── handlers/
│   │   ├── mod.rs
│   │   ├── health.rs
│   │   └── api.rs
│   ├── models/
│   │   └── mod.rs
│   └── errors/
│       └── mod.rs
└── tests/
    └── integration_test.rs
```

---

*[← Part 020: Macros](../part_020/README.md) | [Part 022: Routes และ Extractors →](../part_022/README.md)*
