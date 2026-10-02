# Part 072: Load Testing and Benchmarking

## บทนำ

Load Testing คือกระบวนการทดสอบว่า API ของเราจะทำงานได้อย่างไรเมื่อมีผู้ใช้จำนวนมากพร้อมกัน บทนี้จะครอบคลุมเครื่องมือต่างๆ และวิธีการวิเคราะห์ผลลัพธ์

## 1. เครื่องมือ Load Testing

### 1.1 wrk - C-based HTTP benchmarking tool

```bash
# ติดตั้ง wrk
sudo apt-get install wrk

# การใช้งานพื้นฐาน
# wrk -t<threads> -c<connections> -d<duration> <url>
wrk -t12 -c400 -d30s http://localhost:8080/api/products

# ผลลัพธ์ตัวอย่าง:
# Running 30s test @ http://localhost:8080/api/products
#   12 threads and 400 connections
#   Thread Stats   Avg      Stdev     Max   +/- Stdev
#     Latency    45.23ms   12.34ms 234.56ms   89.23%
#     Req/Sec     1.23k   234.56     2.34k    71.23%
#   441234 requests in 30.10s, 234.56MB read
# Requests/sec:  14659.53
# Transfer/sec:      7.79MB

# ใช้ Lua script กำหนด request ที่ซับซ้อนขึ้น
wrk -t4 -c100 -d30s -s test_script.lua http://localhost:8080/api/users
```

### 1.2 wrk Lua Scripts

```lua
-- test_script.lua
-- สร้าง POST request ที่มี JSON body

wrk.method = "POST"
wrk.body   = '{"name": "Test User", "email": "test@example.com"}'
wrk.headers["Content-Type"] = "application/json"
wrk.headers["Authorization"] = "Bearer test-token"

-- Callback เมื่อ request เสร็จ
function response(status, headers, body)
    if status ~= 200 then
        print("Error: " .. status .. " - " .. body)
    end
end
```

```lua
-- advanced_test.lua
-- Test หลาย endpoints สลับกัน

local paths = {
    "/api/products",
    "/api/products/1",
    "/api/products?category=electronics",
    "/api/users/profile",
}

local counter = 0

function request()
    counter = counter + 1
    local path = paths[(counter % #paths) + 1]
    return wrk.format("GET", path, {
        ["Authorization"] = "Bearer test-token",
        ["Accept"] = "application/json",
    })
end

function done(summary, latency, requests)
    io.write("------------------------------\n")
    io.write(string.format("Total requests: %d\n", summary.requests))
    io.write(string.format("Total errors:   %d\n", summary.errors.status))
    io.write(string.format("P50 latency:    %d ms\n", latency:percentile(50) / 1000))
    io.write(string.format("P90 latency:    %d ms\n", latency:percentile(90) / 1000))
    io.write(string.format("P99 latency:    %d ms\n", latency:percentile(99) / 1000))
    io.write(string.format("P99.9 latency:  %d ms\n", latency:percentile(99.9) / 1000))
end
```

### 1.3 hey - Go-based Load Testing Tool

```bash
# ติดตั้ง hey
go install github.com/rakyll/hey@latest
# หรือ
brew install hey  # macOS

# การใช้งาน
hey -n 10000 -c 100 http://localhost:8080/api/products

# กำหนด rate (requests per second)
hey -n 10000 -c 100 -q 50 http://localhost:8080/api/products

# POST request
hey -n 1000 -c 50 -m POST \
    -H "Content-Type: application/json" \
    -d '{"name":"test"}' \
    http://localhost:8080/api/products

# ผลลัพธ์ตัวอย่าง:
# Summary:
#   Total:	10.0234 secs
#   Slowest:	0.2345 secs
#   Fastest:	0.0012 secs
#   Average:	0.0234 secs
#   Requests/sec:	4988.34
#
# Response time histogram:
#   0.001 [1]	   |
#   0.025 [8234]   |■■■■■■■■■■■■■■■■■■■■■■■■■■■
#   0.048 [1234]   |■■■■
#
# Latency distribution:
#   10% in 0.0156 secs
#   25% in 0.0189 secs
#   50% in 0.0234 secs
#   75% in 0.0278 secs
#   90% in 0.0345 secs
#   95% in 0.0456 secs
#   99% in 0.0890 secs
```

## 2. k6 - Modern Load Testing Tool

### 2.1 ติดตั้งและการใช้งานพื้นฐาน

```bash
# ติดตั้ง k6
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6

# Run script
k6 run test.js

# Run ด้วย custom options
k6 run --vus 100 --duration 30s test.js
```

### 2.2 k6 Scripts ขั้นพื้นฐาน

```javascript
// basic_test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

// Configuration
export const options = {
    vus: 10,              // Virtual Users
    duration: '30s',      // ระยะเวลาทดสอบ
};

export default function () {
    const res = http.get('http://localhost:8080/api/products');
    
    check(res, {
        'status is 200': (r) => r.status === 200,
        'response time < 500ms': (r) => r.timings.duration < 500,
        'body is not empty': (r) => r.body.length > 0,
    });
    
    sleep(1);
}
```

### 2.3 k6 Scripts ขั้นสูง

```javascript
// advanced_test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const responseTime = new Trend('response_time');
const requestCount = new Counter('requests');

// Test configuration with stages
export const options = {
    stages: [
        { duration: '30s', target: 10 },   // Ramp up
        { duration: '1m', target: 50 },    // Stay at 50 VUs
        { duration: '30s', target: 100 },  // Ramp up more
        { duration: '2m', target: 100 },   // Stay at 100 VUs
        { duration: '30s', target: 0 },    // Ramp down
    ],
    thresholds: {
        http_req_duration: ['p(99)<1000'],  // 99% ต้อง < 1 วินาที
        http_req_failed: ['rate<0.01'],     // Error rate < 1%
        errors: ['rate<0.05'],
    },
};

// Base URL
const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';

// Auth token
let authToken = '';

// Setup: run ครั้งเดียวก่อน test
export function setup() {
    const loginRes = http.post(`${BASE_URL}/api/auth/login`, JSON.stringify({
        email: 'test@example.com',
        password: 'password123',
    }), {
        headers: { 'Content-Type': 'application/json' },
    });
    
    check(loginRes, { 'login successful': (r) => r.status === 200 });
    
    return { token: loginRes.json('token') };
}

// Main test function
export default function (data) {
    const token = data.token;
    const headers = {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
    };
    
    group('Product API', () => {
        // List products
        group('List Products', () => {
            const res = http.get(`${BASE_URL}/api/products`, { headers });
            requestCount.add(1);
            responseTime.add(res.timings.duration);
            
            const success = check(res, {
                'status 200': (r) => r.status === 200,
                'has products': (r) => r.json('data') !== null,
                'response time < 500ms': (r) => r.timings.duration < 500,
            });
            
            errorRate.add(!success);
        });
        
        sleep(0.5);
        
        // Get single product
        group('Get Product', () => {
            const productId = Math.floor(Math.random() * 100) + 1;
            const res = http.get(`${BASE_URL}/api/products/${productId}`, { headers });
            requestCount.add(1);
            
            check(res, {
                'status 200 or 404': (r) => r.status === 200 || r.status === 404,
            });
        });
        
        sleep(0.5);
    });
    
    group('Order API', () => {
        // Create order
        group('Create Order', () => {
            const payload = JSON.stringify({
                items: [
                    { product_id: 1, quantity: 2 },
                    { product_id: 2, quantity: 1 },
                ],
            });
            
            const res = http.post(`${BASE_URL}/api/orders`, payload, { headers });
            requestCount.add(1);
            
            const success = check(res, {
                'order created': (r) => r.status === 201,
                'has order id': (r) => r.json('id') !== null,
            });
            
            errorRate.add(!success);
            
            // ถ้า order สำเร็จ ทดสอบ get order
            if (success && res.status === 201) {
                const orderId = res.json('id');
                sleep(0.2);
                
                const getRes = http.get(`${BASE_URL}/api/orders/${orderId}`, { headers });
                check(getRes, {
                    'get order success': (r) => r.status === 200,
                });
            }
        });
        
        sleep(1);
    });
}

// Teardown: run ครั้งเดียวหลัง test
export function teardown(data) {
    console.log('Test complete. Token was:', data.token.substring(0, 10) + '...');
}
```

### 2.4 k6 สำหรับ WebSocket

```javascript
// websocket_test.js
import ws from 'k6/ws';
import { check } from 'k6';

export const options = {
    vus: 10,
    duration: '1m',
};

export default function () {
    const url = 'ws://localhost:8080/ws/chat';
    const params = {
        headers: {
            'Authorization': 'Bearer test-token',
        },
    };
    
    const res = ws.connect(url, params, function (socket) {
        socket.on('open', () => {
            socket.send(JSON.stringify({ type: 'join', room: 'general' }));
        });
        
        socket.on('message', (data) => {
            const msg = JSON.parse(data);
            if (msg.type === 'message') {
                socket.send(JSON.stringify({
                    type: 'message',
                    content: 'Hello from k6!',
                    room: 'general',
                }));
            }
        });
        
        socket.setTimeout(() => {
            socket.close();
        }, 30000);
    });
    
    check(res, { 'Connected successfully': (r) => r && r.status === 101 });
}
```

## 3. การตีความผลลัพธ์

### 3.1 Requests Per Second (RPS)

```
RPS (Throughput): จำนวน requests ที่ server จัดการได้ต่อวินาที

ดี:     > 1,000 RPS สำหรับ simple endpoints
ปานกลาง: 100-1,000 RPS
ต้องปรับปรุง: < 100 RPS

การคำนวณ:
RPS = Total Requests / Duration (seconds)
```

### 3.2 Latency Percentiles

```
P50 (Median):  50% ของ requests เร็วกว่าค่านี้
P90:          90% ของ requests เร็วกว่าค่านี้  
P95:          95% ของ requests เร็วกว่าค่านี้
P99:          99% ของ requests เร็วกว่าค่านี้
P99.9 (Max):  99.9% ของ requests เร็วกว่าค่านี้

เป้าหมายที่ดี:
P50 < 50ms
P90 < 100ms
P95 < 200ms
P99 < 500ms
```

### 3.3 Script วิเคราะห์ผลลัพธ์

```javascript
// analyze_results.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { htmlReport } from 'https://raw.githubusercontent.com/benc-uk/k6-reporter/main/dist/bundle.js';
import { textSummary } from 'https://jslib.k6.io/k6-summary/0.0.1/index.js';

export const options = {
    vus: 50,
    duration: '2m',
    thresholds: {
        http_req_duration: [
            'p(50)<100',   // P50 < 100ms
            'p(90)<200',   // P90 < 200ms
            'p(99)<500',   // P99 < 500ms
        ],
        http_req_failed: ['rate<0.01'],
        http_reqs: ['rate>100'],  // ต้องมี throughput > 100 RPS
    },
};

export default function () {
    const res = http.get('http://localhost:8080/api/products');
    check(res, { 'is 200': (r) => r.status === 200 });
    sleep(0.1);
}

// สร้าง HTML report หลัง test เสร็จ
export function handleSummary(data) {
    return {
        'report.html': htmlReport(data),
        stdout: textSummary(data, { indent: ' ', enableColors: true }),
        'summary.json': JSON.stringify(data),
    };
}
```

```bash
# Run และบันทึก output
k6 run --out json=results.json analyze_results.js

# Import ผลลัพธ์เข้า Grafana
k6 run --out influxdb=http://localhost:8086/k6 analyze_results.js
```

## 4. Finding Bottlenecks

### 4.1 Database Bottleneck Detection

```rust
// src/middleware/query_logger.rs
use actix_web::{dev, middleware, web, Error};
use std::time::Instant;

pub struct QueryTimer;

impl QueryTimer {
    pub fn track_query(query: &str) -> QueryGuard {
        QueryGuard {
            query: query.to_string(),
            start: Instant::now(),
        }
    }
}

pub struct QueryGuard {
    query: String,
    start: Instant,
}

impl Drop for QueryGuard {
    fn drop(&mut self) {
        let duration = self.start.elapsed();
        if duration.as_millis() > 100 {
            // Log slow queries
            eprintln!(
                "SLOW QUERY ({:?}): {}",
                duration,
                &self.query[..100.min(self.query.len())]
            );
        }
    }
}

// ใช้ใน handler
use sqlx::PgPool;

async fn get_products_with_timing(db: web::Data<PgPool>) -> actix_web::HttpResponse {
    let _timer = QueryTimer::track_query("SELECT * FROM products WITH TAGS");
    
    let result = sqlx::query!(
        r#"
        SELECT p.*, array_agg(t.tag) as tags
        FROM products p
        LEFT JOIN product_tags t ON p.id = t.product_id
        GROUP BY p.id
        "#
    )
    .fetch_all(db.get_ref())
    .await;
    
    match result {
        Ok(rows) => actix_web::HttpResponse::Ok().json(rows.len()),
        Err(_) => actix_web::HttpResponse::InternalServerError().finish(),
    }
}
```

### 4.2 PostgreSQL Query Analysis

```sql
-- เปิด slow query log
ALTER SYSTEM SET log_min_duration_statement = '100ms';
SELECT pg_reload_conf();

-- ดู queries ที่ช้า
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    rows,
    100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 20;

-- EXPLAIN ANALYZE
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT p.*, array_agg(t.tag) as tags
FROM products p
LEFT JOIN product_tags t ON p.id = t.product_id
WHERE p.category = 'electronics'
GROUP BY p.id
ORDER BY p.created_at DESC
LIMIT 20;

-- ตรวจสอบ index usage
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

### 4.3 Application-level Bottleneck Analysis

```rust
// src/telemetry.rs - OpenTelemetry integration
use tracing::{info, instrument, span, Level};
use tracing_subscriber;
use std::time::Instant;

// Instrument ฟังก์ชัน
#[instrument(skip(db))]
async fn fetch_products(
    db: &sqlx::PgPool,
    category: Option<&str>,
) -> Result<Vec<Product>, sqlx::Error> {
    let start = Instant::now();
    
    let result = sqlx::query_as!(
        Product,
        "SELECT * FROM products WHERE ($1::text IS NULL OR category = $1)",
        category
    )
    .fetch_all(db)
    .await;
    
    info!(
        duration_ms = start.elapsed().as_millis(),
        category = category,
        "Fetched products"
    );
    
    result
}

// Middleware สำหรับวัด request timing
use actix_web::{dev::ServiceRequest, dev::ServiceResponse, Error};
use actix_web::dev::Transform;
use futures::future::LocalBoxFuture;
use std::future::{ready, Ready};

pub struct TimingMiddleware;

impl<S, B> Transform<S, ServiceRequest> for TimingMiddleware
where
    S: actix_web::dev::Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = TimingMiddlewareService<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(TimingMiddlewareService { service }))
    }
}

pub struct TimingMiddlewareService<S> {
    service: S,
}

impl<S, B> actix_web::dev::Service<ServiceRequest> for TimingMiddlewareService<S>
where
    S: actix_web::dev::Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    actix_web::dev::forward_ready!(service);

    fn call(&self, req: ServiceRequest) -> Self::Future {
        let start = Instant::now();
        let path = req.path().to_owned();
        let method = req.method().to_string();
        
        let fut = self.service.call(req);
        
        Box::pin(async move {
            let res = fut.await?;
            let duration = start.elapsed();
            
            println!(
                "{} {} - {}ms - {}",
                method,
                path,
                duration.as_millis(),
                res.status()
            );
            
            Ok(res)
        })
    }
}
```

## 5. Database Query Analysis

### 5.1 Connection Pool Monitoring

```rust
// src/db_monitoring.rs
use sqlx::PgPool;
use std::time::Duration;
use tokio::time;

pub async fn monitor_pool(pool: PgPool) {
    let mut interval = time::interval(Duration::from_secs(60));
    
    loop {
        interval.tick().await;
        
        let pool_options = pool.options();
        let size = pool.size();
        let idle = pool.num_idle();
        
        println!(
            "DB Pool - Size: {}/{}, Idle: {}",
            size,
            pool_options.get_max_connections(),
            idle
        );
        
        // Check pool health
        if idle == 0 && size >= pool_options.get_max_connections() {
            eprintln!("WARNING: Database connection pool exhausted!");
        }
    }
}

// ดู query stats ผ่าน API
use actix_web::{web, HttpResponse};

async fn db_stats(db: web::Data<PgPool>) -> HttpResponse {
    let result = sqlx::query!(
        r#"
        SELECT 
            count(*) as total_queries,
            sum(total_time) as total_time_ms,
            avg(mean_time) as avg_time_ms,
            max(mean_time) as max_time_ms
        FROM pg_stat_statements
        WHERE query NOT LIKE '%pg_stat_statements%'
        "#
    )
    .fetch_one(db.get_ref())
    .await;
    
    match result {
        Ok(stats) => HttpResponse::Ok().json(serde_json::json!({
            "total_queries": stats.total_queries,
            "total_time_ms": stats.total_time_ms,
            "avg_time_ms": stats.avg_time_ms,
            "max_time_ms": stats.max_time_ms,
        })),
        Err(_) => HttpResponse::InternalServerError().finish(),
    }
}
```

## 6. Connection Pool Sizing

### 6.1 คำนวณ Pool Size ที่เหมาะสม

```
สูตร: pool_size = (cpu_cores * 2) + disk_spindles

สำหรับ PostgreSQL:
- ไม่เกิน max_connections ของ PostgreSQL (default: 100)
- ต้องเผื่อ connections สำหรับ maintenance tasks

ตัวอย่าง:
- Server มี 4 CPU cores
- ใช้ SSD (1 spindle)
- pool_size = (4 * 2) + 1 = 9

แต่ถ้ามีหลาย app instances ต้องหาร:
- 3 instances, 9 connections each = 27 total
- ตรวจสอบว่าไม่เกิน PostgreSQL max_connections
```

```rust
// src/db_config.rs
use sqlx::postgres::PgPoolOptions;
use std::env;

pub async fn create_pool() -> sqlx::PgPool {
    let database_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    
    let max_connections = env::var("DB_MAX_CONNECTIONS")
        .unwrap_or_else(|_| "10".to_string())
        .parse::<u32>()
        .unwrap_or(10);
    
    let min_connections = env::var("DB_MIN_CONNECTIONS")
        .unwrap_or_else(|_| "2".to_string())
        .parse::<u32>()
        .unwrap_or(2);
    
    PgPoolOptions::new()
        .max_connections(max_connections)
        .min_connections(min_connections)
        .acquire_timeout(Duration::from_secs(3))
        .idle_timeout(Duration::from_secs(600))
        .max_lifetime(Duration::from_secs(1800))
        .connect(&database_url)
        .await
        .expect("Failed to create database pool")
}

use std::time::Duration;
```

## 7. Tuning Actix-web Workers

### 7.1 Worker Configuration

```rust
// src/main.rs
use actix_web::{web, App, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // จำนวน workers ที่เหมาะสม
    let workers = std::env::var("WORKERS")
        .ok()
        .and_then(|w| w.parse().ok())
        .unwrap_or_else(num_cpus::get);
    
    println!("Starting with {} workers", workers);
    
    HttpServer::new(|| {
        App::new()
            // ตั้งค่า connection limits
            .app_data(web::JsonConfig::default()
                .limit(1_048_576))  // 1MB JSON limit
    })
    .workers(workers)
    // Connection settings
    .keep_alive(std::time::Duration::from_secs(75))
    .client_request_timeout(std::time::Duration::from_secs(60))
    .client_disconnect_timeout(std::time::Duration::from_secs(5))
    // TLS settings (ถ้ามี)
    // .bind_rustls("0.0.0.0:443", tls_config)?
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

### 7.2 การทดสอบ Worker Count ต่างๆ

```bash
#!/bin/bash
# benchmark_workers.sh

echo "Testing different worker counts..."
for workers in 1 2 4 8 16; do
    echo "=== Workers: $workers ==="
    WORKERS=$workers cargo run --release &
    SERVER_PID=$!
    sleep 2  # รอ server เริ่มต้น
    
    # Run load test
    hey -n 10000 -c 100 http://localhost:8080/api/products 2>&1 | \
        grep -E "Requests/sec|Average|P99"
    
    kill $SERVER_PID
    sleep 1
done
```

## 8. Practical: Load Test a CRUD API

### 8.1 สร้าง API สำหรับทดสอบ

```rust
// src/main.rs - Complete CRUD API
use actix_web::{web, App, HttpServer, HttpResponse, middleware};
use sqlx::PgPool;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize, sqlx::FromRow)]
struct Product {
    id: Option<i32>,
    name: String,
    price: f64,
    stock: i32,
    category: String,
}

#[derive(Debug, Deserialize)]
struct ProductQuery {
    category: Option<String>,
    min_price: Option<f64>,
    max_price: Option<f64>,
    page: Option<i64>,
    per_page: Option<i64>,
}

// GET /products
async fn list_products(
    db: web::Data<PgPool>,
    query: web::Query<ProductQuery>,
) -> HttpResponse {
    let page = query.page.unwrap_or(1).max(1);
    let per_page = query.per_page.unwrap_or(20).min(100);
    let offset = (page - 1) * per_page;
    
    match sqlx::query_as!(
        Product,
        r#"
        SELECT id, name, price, stock, category
        FROM products
        WHERE ($1::text IS NULL OR category = $1)
          AND ($2::float8 IS NULL OR price >= $2)
          AND ($3::float8 IS NULL OR price <= $3)
        ORDER BY id
        LIMIT $4 OFFSET $5
        "#,
        query.category.as_deref(),
        query.min_price,
        query.max_price,
        per_page,
        offset
    )
    .fetch_all(db.get_ref())
    .await
    {
        Ok(products) => HttpResponse::Ok().json(products),
        Err(e) => {
            eprintln!("DB Error: {}", e);
            HttpResponse::InternalServerError().finish()
        }
    }
}

// GET /products/{id}
async fn get_product(
    db: web::Data<PgPool>,
    path: web::Path<i32>,
) -> HttpResponse {
    let id = path.into_inner();
    
    match sqlx::query_as!(
        Product,
        "SELECT id, name, price, stock, category FROM products WHERE id = $1",
        id
    )
    .fetch_optional(db.get_ref())
    .await
    {
        Ok(Some(product)) => HttpResponse::Ok().json(product),
        Ok(None) => HttpResponse::NotFound().json(serde_json::json!({
            "error": "Product not found"
        })),
        Err(e) => {
            eprintln!("DB Error: {}", e);
            HttpResponse::InternalServerError().finish()
        }
    }
}

// POST /products
async fn create_product(
    db: web::Data<PgPool>,
    product: web::Json<Product>,
) -> HttpResponse {
    match sqlx::query!(
        "INSERT INTO products (name, price, stock, category) VALUES ($1, $2, $3, $4) RETURNING id",
        product.name,
        product.price,
        product.stock,
        product.category
    )
    .fetch_one(db.get_ref())
    .await
    {
        Ok(row) => HttpResponse::Created().json(serde_json::json!({
            "id": row.id,
            "message": "Product created"
        })),
        Err(e) => {
            eprintln!("DB Error: {}", e);
            HttpResponse::InternalServerError().finish()
        }
    }
}

// PUT /products/{id}
async fn update_product(
    db: web::Data<PgPool>,
    path: web::Path<i32>,
    product: web::Json<Product>,
) -> HttpResponse {
    let id = path.into_inner();
    
    match sqlx::query!(
        r#"
        UPDATE products 
        SET name = $1, price = $2, stock = $3, category = $4
        WHERE id = $5
        "#,
        product.name,
        product.price,
        product.stock,
        product.category,
        id
    )
    .execute(db.get_ref())
    .await
    {
        Ok(result) if result.rows_affected() > 0 => {
            HttpResponse::Ok().json(serde_json::json!({ "message": "Updated" }))
        }
        Ok(_) => HttpResponse::NotFound().json(serde_json::json!({
            "error": "Product not found"
        })),
        Err(e) => {
            eprintln!("DB Error: {}", e);
            HttpResponse::InternalServerError().finish()
        }
    }
}

// DELETE /products/{id}
async fn delete_product(
    db: web::Data<PgPool>,
    path: web::Path<i32>,
) -> HttpResponse {
    let id = path.into_inner();
    
    match sqlx::query!("DELETE FROM products WHERE id = $1", id)
        .execute(db.get_ref())
        .await
    {
        Ok(result) if result.rows_affected() > 0 => {
            HttpResponse::NoContent().finish()
        }
        Ok(_) => HttpResponse::NotFound().finish(),
        Err(e) => {
            eprintln!("DB Error: {}", e);
            HttpResponse::InternalServerError().finish()
        }
    }
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let database_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL required");
    
    let pool = sqlx::postgres::PgPoolOptions::new()
        .max_connections(20)
        .connect(&database_url)
        .await
        .expect("Failed to connect to database");
    
    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(pool.clone()))
            .wrap(middleware::Compress::default())
            .wrap(middleware::Logger::default())
            .service(
                web::scope("/api")
                    .route("/products", web::get().to(list_products))
                    .route("/products", web::post().to(create_product))
                    .route("/products/{id}", web::get().to(get_product))
                    .route("/products/{id}", web::put().to(update_product))
                    .route("/products/{id}", web::delete().to(delete_product))
            )
    })
    .workers(num_cpus::get())
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

### 8.2 Complete k6 Test Script สำหรับ CRUD API

```javascript
// crud_load_test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080/api';

const createErrors = new Rate('create_errors');
const readErrors = new Rate('read_errors');
const updateErrors = new Rate('update_errors');
const deleteErrors = new Rate('delete_errors');
const readLatency = new Trend('read_latency');

export const options = {
    scenarios: {
        // Scenario 1: Read-heavy workload (70% reads)
        read_heavy: {
            executor: 'constant-vus',
            vus: 70,
            duration: '2m',
            exec: 'readScenario',
        },
        // Scenario 2: Write workload (30% writes)
        write_workload: {
            executor: 'constant-vus',
            vus: 30,
            duration: '2m',
            exec: 'writeScenario',
        },
    },
    thresholds: {
        http_req_duration: ['p(99)<1000'],
        http_req_failed: ['rate<0.01'],
        create_errors: ['rate<0.05'],
        read_errors: ['rate<0.01'],
    },
};

export function readScenario() {
    group('Read Operations', () => {
        // List products
        const listRes = http.get(`${BASE_URL}/products`);
        readLatency.add(listRes.timings.duration);
        
        const listOk = check(listRes, {
            'list products 200': (r) => r.status === 200,
            'list response time < 200ms': (r) => r.timings.duration < 200,
        });
        readErrors.add(!listOk);
        
        sleep(0.5);
        
        // Get specific product
        const productId = Math.floor(Math.random() * 1000) + 1;
        const getRes = http.get(`${BASE_URL}/products/${productId}`);
        
        check(getRes, {
            'get product valid response': (r) => r.status === 200 || r.status === 404,
        });
        
        sleep(0.5);
    });
}

export function writeScenario() {
    let createdId = null;
    
    group('Write Operations', () => {
        // Create product
        const createPayload = JSON.stringify({
            name: `Product ${Date.now()}`,
            price: Math.random() * 1000,
            stock: Math.floor(Math.random() * 100),
            category: ['electronics', 'clothing', 'books'][Math.floor(Math.random() * 3)],
        });
        
        const createRes = http.post(`${BASE_URL}/products`, createPayload, {
            headers: { 'Content-Type': 'application/json' },
        });
        
        const createOk = check(createRes, {
            'create product 201': (r) => r.status === 201,
        });
        createErrors.add(!createOk);
        
        if (createOk) {
            createdId = createRes.json('id');
            sleep(0.3);
            
            // Update the created product
            const updatePayload = JSON.stringify({
                name: `Updated Product ${Date.now()}`,
                price: Math.random() * 1000,
                stock: Math.floor(Math.random() * 100),
                category: 'updated',
            });
            
            const updateRes = http.put(
                `${BASE_URL}/products/${createdId}`,
                updatePayload,
                { headers: { 'Content-Type': 'application/json' } }
            );
            
            const updateOk = check(updateRes, {
                'update product 200': (r) => r.status === 200,
            });
            updateErrors.add(!updateOk);
            
            sleep(0.3);
            
            // Delete the product
            const deleteRes = http.del(`${BASE_URL}/products/${createdId}`);
            
            const deleteOk = check(deleteRes, {
                'delete product 204': (r) => r.status === 204,
            });
            deleteErrors.add(!deleteOk);
        }
        
        sleep(1);
    });
}
```

### 8.3 Script สร้าง Test Data

```bash
#!/bin/bash
# create_test_data.sh

DATABASE_URL=${DATABASE_URL:-"postgres://user:pass@localhost/testdb"}

echo "Creating test data..."

psql "$DATABASE_URL" << 'EOF'
-- สร้าง table
CREATE TABLE IF NOT EXISTS products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock INTEGER NOT NULL DEFAULT 0,
    category VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- สร้าง index
CREATE INDEX IF NOT EXISTS idx_products_category ON products(category);
CREATE INDEX IF NOT EXISTS idx_products_price ON products(price);

-- Insert test data
INSERT INTO products (name, price, stock, category)
SELECT 
    'Product ' || generate_series AS name,
    (random() * 1000)::decimal(10,2) AS price,
    (random() * 100)::integer AS stock,
    (ARRAY['electronics', 'clothing', 'books', 'food', 'toys'])[floor(random() * 5 + 1)] AS category
FROM generate_series(1, 10000);

ANALYZE products;
EOF

echo "Test data created: 10,000 products"
```

### 8.4 วิเคราะห์ผลลัพธ์ Load Test

```bash
# Run load test และวิเคราะห์
k6 run \
    --out json=results.json \
    --out influxdb=http://localhost:8086/k6 \
    crud_load_test.js

# วิเคราะห์ผลลัพธ์ JSON
cat results.json | python3 << 'EOF'
import json, sys
from collections import defaultdict

metrics = defaultdict(list)
for line in sys.stdin:
    try:
        data = json.loads(line)
        if data['type'] == 'Point':
            metrics[data['metric']].append(data['data']['value'])
    except:
        pass

for metric, values in sorted(metrics.items()):
    if values:
        sorted_v = sorted(values)
        n = len(sorted_v)
        p50 = sorted_v[int(n * 0.5)]
        p90 = sorted_v[int(n * 0.9)]
        p99 = sorted_v[int(n * 0.99)]
        print(f"{metric}: p50={p50:.2f} p90={p90:.2f} p99={p99:.2f}")
EOF
```

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **wrk** - เครื่องมือ load testing ที่เร็วและเบา
2. **k6** - เครื่องมือ load testing ที่ยืดหยุ่น scripted
3. **การตีความผลลัพธ์** - RPS, latency percentiles
4. **การหา bottleneck** - database queries, connection pool
5. **การ tune Actix-web** - workers, keep-alive, timeouts
6. **Complete CRUD load test** - ทดสอบทุก operation

---

[⬅️ Part 071: Performance Optimization](../part_071/README.md) | [➡️ Part 073: Docker and Containerization](../part_073/README.md)
