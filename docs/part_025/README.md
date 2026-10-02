# Part 025: State Management ใน Actix-web

## สารบัญ
- [แนะนำ State Management](#แนะนำ-state-management)
- [web::Data<T> สำหรับ shared state](#webdatat-สำหรับ-shared-state)
- [AppState Pattern](#appstate-pattern)
- [Arc<Mutex<T>> vs Arc<RwLock<T>>](#arcmutext-vs-arcrwlockt)
- [Database Pool as State](#database-pool-as-state)
- [Configuration State](#configuration-state)
- [Cache State](#cache-state)
- [Counter Examples](#counter-examples)
- [Multiple State Types](#multiple-state-types)

---

## แนะนำ State Management

ใน web application เรามักต้องการแชร์ข้อมูลระหว่าง handlers เช่น database connections, configuration, cache ฯลฯ Actix-web ใช้ `web::Data<T>` สำหรับจุดประสงค์นี้

### Dependencies

```toml
[dependencies]
actix-web = "4"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
dotenv = "0.15"
config = "0.13"
```

---

## web::Data<T> สำหรับ shared state

`web::Data<T>` คือ wrapper ที่ใช้ส่ง data ไปยัง handlers ผ่าน dependency injection

```rust
use actix_web::{web, App, HttpServer, HttpResponse};

// State ง่ายๆ
struct AppState {
    app_name: String,
    version: String,
}

async fn index(data: web::Data<AppState>) -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({
        "app": data.app_name,
        "version": data.version
    }))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let state = web::Data::new(AppState {
        app_name: "My API".to_string(),
        version: "1.0.0".to_string(),
    });
    
    HttpServer::new(move || {
        App::new()
            .app_data(state.clone())  // เพิ่ม state
            .route("/", web::get().to(index))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}

// web::Data ทำงานอย่างไร?
// - web::Data<T> = Arc<T> (reference counted pointer)
// - เมื่อ .app_data(state.clone()) - clone เป็นการ clone Arc ไม่ใช่ data
// - ทุก handler thread ได้รับ pointer ที่ชี้ไปหา data เดียวกัน
// - ถ้า T ต้องการ mutation ต้องใช้ Mutex หรือ RwLock ภายใน

// การรับ state ใน handler
async fn handler_with_state(
    state: web::Data<AppState>,
    body: web::Json<serde_json::Value>,
) -> HttpResponse {
    // ใช้ state.field ได้ตรงๆ เพราะ web::Data implement Deref
    let app_name = &state.app_name;
    
    // หรือ deref explicitly
    let app = state.as_ref();
    
    HttpResponse::Ok().json(serde_json::json!({
        "app": app_name,
        "received": body.into_inner()
    }))
}
```

---

## AppState Pattern

Pattern ที่นิยมในการรวม state ทั้งหมดไว้ใน struct เดียว

```rust
use actix_web::{web, App, HttpServer, HttpResponse, Result};
use std::sync::{Arc, RwLock, Mutex};
use std::collections::HashMap;
use serde::{Serialize, Deserialize};
use chrono::{DateTime, Utc};

// Models
#[derive(Debug, Clone, Serialize, Deserialize)]
struct User {
    id: u64,
    username: String,
    email: String,
    created_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Post {
    id: u64,
    title: String,
    content: String,
    author_id: u64,
    published: bool,
    created_at: DateTime<Utc>,
}

// Configuration
#[derive(Debug, Clone)]
struct AppConfig {
    database_url: String,
    redis_url: String,
    jwt_secret: String,
    max_upload_size: usize,
    allow_registration: bool,
}

impl AppConfig {
    fn from_env() -> Self {
        AppConfig {
            database_url: std::env::var("DATABASE_URL")
                .unwrap_or_else(|_| "postgres://localhost/myapp".to_string()),
            redis_url: std::env::var("REDIS_URL")
                .unwrap_or_else(|_| "redis://localhost:6379".to_string()),
            jwt_secret: std::env::var("JWT_SECRET")
                .unwrap_or_else(|_| "secret-key-change-in-production".to_string()),
            max_upload_size: std::env::var("MAX_UPLOAD_SIZE")
                .ok()
                .and_then(|s| s.parse().ok())
                .unwrap_or(10 * 1024 * 1024),  // 10MB default
            allow_registration: std::env::var("ALLOW_REGISTRATION")
                .map(|s| s == "true")
                .unwrap_or(true),
        }
    }
}

// In-memory stores (ในการใช้งานจริงจะเป็น database)
type UserStore = RwLock<HashMap<u64, User>>;
type PostStore = RwLock<HashMap<u64, Post>>;
type Counter = Mutex<u64>;

// AppState รวม state ทั้งหมด
struct AppState {
    config: AppConfig,
    users: UserStore,
    posts: PostStore,
    request_counter: Counter,
    next_user_id: Mutex<u64>,
    next_post_id: Mutex<u64>,
}

impl AppState {
    fn new(config: AppConfig) -> Self {
        AppState {
            config,
            users: RwLock::new(HashMap::new()),
            posts: RwLock::new(HashMap::new()),
            request_counter: Mutex::new(0),
            next_user_id: Mutex::new(1),
            next_post_id: Mutex::new(1),
        }
    }
    
    fn next_user_id(&self) -> u64 {
        let mut id = self.next_user_id.lock().unwrap();
        let current = *id;
        *id += 1;
        current
    }
    
    fn next_post_id(&self) -> u64 {
        let mut id = self.next_post_id.lock().unwrap();
        let current = *id;
        *id += 1;
        current
    }
    
    fn increment_request_counter(&self) {
        let mut counter = self.request_counter.lock().unwrap();
        *counter += 1;
    }
    
    fn get_request_count(&self) -> u64 {
        *self.request_counter.lock().unwrap()
    }
}

// Handlers
async fn create_user(
    state: web::Data<AppState>,
    body: web::Json<CreateUserRequest>,
) -> Result<HttpResponse> {
    state.increment_request_counter();
    let req = body.into_inner();
    
    // ตรวจสอบว่า email ซ้ำไหม
    {
        let users = state.users.read().unwrap();
        if users.values().any(|u| u.email == req.email) {
            return Ok(HttpResponse::Conflict().json(serde_json::json!({
                "error": "email_taken",
                "message": "This email is already registered"
            })));
        }
    }
    
    let user = User {
        id: state.next_user_id(),
        username: req.username,
        email: req.email,
        created_at: Utc::now(),
    };
    
    {
        let mut users = state.users.write().unwrap();
        users.insert(user.id, user.clone());
    }
    
    Ok(HttpResponse::Created().json(user))
}

#[derive(Deserialize)]
struct CreateUserRequest {
    username: String,
    email: String,
}

async fn list_users(state: web::Data<AppState>) -> Result<HttpResponse> {
    state.increment_request_counter();
    
    let users = state.users.read().unwrap();
    let user_list: Vec<&User> = users.values().collect();
    
    Ok(HttpResponse::Ok().json(user_list))
}

async fn get_stats(state: web::Data<AppState>) -> Result<HttpResponse> {
    let user_count = state.users.read().unwrap().len();
    let post_count = state.posts.read().unwrap().len();
    let request_count = state.get_request_count();
    
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "users": user_count,
        "posts": post_count,
        "total_requests": request_count,
        "config": {
            "allow_registration": state.config.allow_registration,
            "max_upload_size": state.config.max_upload_size
        }
    })))
}

#[actix_web::main]
async fn main_app() -> std::io::Result<()> {
    let config = AppConfig::from_env();
    let state = web::Data::new(AppState::new(config));
    
    HttpServer::new(move || {
        App::new()
            .app_data(state.clone())
            .service(
                web::scope("/api/v1")
                    .route("/users", web::post().to(create_user))
                    .route("/users", web::get().to(list_users))
                    .route("/stats", web::get().to(get_stats))
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## Arc<Mutex<T>> vs Arc<RwLock<T>>

ความแตกต่างระหว่าง Mutex และ RwLock ในการจัดการ concurrent access

```rust
use std::sync::{Arc, Mutex, RwLock};
use std::collections::HashMap;
use tokio::sync::{Mutex as TokioMutex, RwLock as TokioRwLock};

// Mutex: อนุญาต access เดียวในเวลาเดียว (ทั้ง read และ write)
// ใช้เมื่อ: write บ่อย หรือ read น้อย
struct HighWriteState {
    data: Mutex<HashMap<String, String>>,
}

impl HighWriteState {
    fn read(&self, key: &str) -> Option<String> {
        let data = self.data.lock().unwrap();
        data.get(key).cloned()
    }
    
    fn write(&self, key: String, value: String) {
        let mut data = self.data.lock().unwrap();
        data.insert(key, value);
    }
}

// RwLock: อนุญาต multiple readers หรือ single writer
// ใช้เมื่อ: read บ่อย write น้อย (เหมาะกับ cache, config)
struct HighReadState {
    data: RwLock<HashMap<String, String>>,
}

impl HighReadState {
    fn read(&self, key: &str) -> Option<String> {
        // Multiple concurrent readers allowed
        let data = self.data.read().unwrap();
        data.get(key).cloned()
    }
    
    fn write(&self, key: String, value: String) {
        // Exclusive write access
        let mut data = self.data.write().unwrap();
        data.insert(key, value);
    }
}

// Tokio Mutex: ใช้ใน async context
// ไม่ block thread ขณะรอ lock
struct AsyncState {
    data: TokioMutex<HashMap<String, String>>,
}

impl AsyncState {
    async fn read(&self, key: &str) -> Option<String> {
        let data = self.data.lock().await;
        data.get(key).cloned()
    }
    
    async fn write(&self, key: String, value: String) {
        let mut data = self.data.lock().await;
        data.insert(key, value);
    }
}

// Tokio RwLock: async RwLock
struct AsyncHighReadState {
    data: TokioRwLock<Vec<String>>,
}

impl AsyncHighReadState {
    async fn get_all(&self) -> Vec<String> {
        let data = self.data.read().await;
        data.clone()
    }
    
    async fn add(&self, item: String) {
        let mut data = self.data.write().await;
        data.push(item);
    }
}

// เปรียบเทียบ performance
// Mutex สำหรับ write-heavy workload
async fn benchmark_mutex() {
    let state = Arc::new(HighWriteState {
        data: Mutex::new(HashMap::new()),
    });
    
    // Write operations
    for i in 0..1000 {
        let state = state.clone();
        tokio::spawn(async move {
            state.write(format!("key_{}", i), format!("value_{}", i));
        });
    }
    
    // Read operations  
    for i in 0..1000 {
        let state = state.clone();
        tokio::spawn(async move {
            let _ = state.read(&format!("key_{}", i));
        });
    }
}

// แนวทางการเลือก
fn choosing_guide() {
    // ใช้ Mutex เมื่อ:
    // - Write operations บ่อยหรือเท่ากับ read
    // - Critical sections สั้น
    // - ไม่ต้องการ concurrent reads
    
    // ใช้ RwLock เมื่อ:
    // - Read operations บ่อยกว่า write มาก (เช่น config)
    // - Read operations ใช้เวลานาน (เช่น JSON serialization ขณะ holding lock)
    // - Cache patterns
    
    // ใช้ Tokio Mutex/RwLock เมื่อ:
    // - ต้องการ async/await ภายใน critical section
    // - ไม่ต้องการ block thread pool
    
    // ระวัง Deadlock!
    // - อย่า lock หลาย locks พร้อมกันโดยไม่มีลำดับที่แน่นอน
    // - อย่าถือ lock ขณะเรียก async function ที่อาจ acquire lock เดิม
}
```

---

## Database Pool as State

การใช้ sqlx connection pool เป็น state

```rust
use actix_web::{web, App, HttpServer, HttpResponse, Result};
use sqlx::{PgPool, postgres::PgPoolOptions, Row};
use serde::{Serialize, Deserialize};
use uuid::Uuid;
use chrono::{DateTime, Utc};

// Models
#[derive(Debug, Serialize, sqlx::FromRow)]
struct UserRecord {
    id: Uuid,
    username: String,
    email: String,
    is_active: bool,
    created_at: DateTime<Utc>,
    updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
struct CreateUserDb {
    username: String,
    email: String,
    password: String,
}

#[derive(Debug, Deserialize)]
struct UserQueryParams {
    page: Option<i64>,
    per_page: Option<i64>,
    active_only: Option<bool>,
}

// Database operations
async fn db_create_user(
    pool: &PgPool,
    req: &CreateUserDb,
    password_hash: &str,
) -> Result<UserRecord, sqlx::Error> {
    let user = sqlx::query_as!(
        UserRecord,
        r#"
        INSERT INTO users (id, username, email, password_hash, is_active, created_at, updated_at)
        VALUES ($1, $2, $3, $4, true, NOW(), NOW())
        RETURNING id, username, email, is_active, created_at, updated_at
        "#,
        Uuid::new_v4(),
        req.username,
        req.email,
        password_hash
    )
    .fetch_one(pool)
    .await?;
    
    Ok(user)
}

async fn db_get_users(
    pool: &PgPool,
    page: i64,
    per_page: i64,
    active_only: bool,
) -> Result<(Vec<UserRecord>, i64), sqlx::Error> {
    let offset = (page - 1) * per_page;
    
    let users = if active_only {
        sqlx::query_as!(
            UserRecord,
            r#"
            SELECT id, username, email, is_active, created_at, updated_at
            FROM users
            WHERE is_active = true
            ORDER BY created_at DESC
            LIMIT $1 OFFSET $2
            "#,
            per_page,
            offset
        )
        .fetch_all(pool)
        .await?
    } else {
        sqlx::query_as!(
            UserRecord,
            r#"
            SELECT id, username, email, is_active, created_at, updated_at
            FROM users
            ORDER BY created_at DESC
            LIMIT $1 OFFSET $2
            "#,
            per_page,
            offset
        )
        .fetch_all(pool)
        .await?
    };
    
    let total: i64 = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM users WHERE is_active = true OR $1 = false",
        active_only
    )
    .fetch_one(pool)
    .await?
    .unwrap_or(0);
    
    Ok((users, total))
}

async fn db_get_user(pool: &PgPool, id: Uuid) -> Result<Option<UserRecord>, sqlx::Error> {
    let user = sqlx::query_as!(
        UserRecord,
        r#"
        SELECT id, username, email, is_active, created_at, updated_at
        FROM users WHERE id = $1
        "#,
        id
    )
    .fetch_optional(pool)
    .await?;
    
    Ok(user)
}

// Handlers ที่ใช้ database pool
async fn api_create_user(
    pool: web::Data<PgPool>,
    body: web::Json<CreateUserDb>,
) -> Result<HttpResponse> {
    let req = body.into_inner();
    
    // ตรวจสอบ input
    if req.username.is_empty() || req.email.is_empty() {
        return Ok(HttpResponse::BadRequest().json(serde_json::json!({
            "error": "Invalid input"
        })));
    }
    
    // Hash password (ใช้ argon2 หรือ bcrypt ใน production)
    let password_hash = format!("hashed:{}", req.password);
    
    match db_create_user(&pool, &req, &password_hash).await {
        Ok(user) => Ok(HttpResponse::Created().json(user)),
        Err(sqlx::Error::Database(db_err)) if db_err.is_unique_violation() => {
            Ok(HttpResponse::Conflict().json(serde_json::json!({
                "error": "duplicate_entry",
                "message": "Username or email already exists"
            })))
        },
        Err(e) => {
            log::error!("Database error: {}", e);
            Ok(HttpResponse::InternalServerError().json(serde_json::json!({
                "error": "database_error"
            })))
        }
    }
}

async fn api_list_users(
    pool: web::Data<PgPool>,
    query: web::Query<UserQueryParams>,
) -> Result<HttpResponse> {
    let page = query.page.unwrap_or(1).max(1);
    let per_page = query.per_page.unwrap_or(20).min(100);
    let active_only = query.active_only.unwrap_or(false);
    
    match db_get_users(&pool, page, per_page, active_only).await {
        Ok((users, total)) => {
            Ok(HttpResponse::Ok().json(serde_json::json!({
                "data": users,
                "pagination": {
                    "total": total,
                    "page": page,
                    "per_page": per_page,
                    "total_pages": (total + per_page - 1) / per_page
                }
            })))
        },
        Err(e) => {
            log::error!("Database error: {}", e);
            Ok(HttpResponse::InternalServerError().json(serde_json::json!({
                "error": "database_error"
            })))
        }
    }
}

// Setup database
async fn setup_database() -> PgPool {
    let database_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    
    PgPoolOptions::new()
        .max_connections(20)
        .min_connections(5)
        .acquire_timeout(std::time::Duration::from_secs(30))
        .idle_timeout(std::time::Duration::from_secs(600))
        .max_lifetime(std::time::Duration::from_secs(1800))
        .connect(&database_url)
        .await
        .expect("Failed to connect to database")
}

// Main function
#[actix_web::main]
async fn main_db() -> std::io::Result<()> {
    dotenv::dotenv().ok();
    
    let pool = setup_database().await;
    let pool_data = web::Data::new(pool);
    
    HttpServer::new(move || {
        App::new()
            .app_data(pool_data.clone())
            .service(
                web::scope("/api/v1")
                    .route("/users", web::post().to(api_create_user))
                    .route("/users", web::get().to(api_list_users))
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## Configuration State

การจัดการ configuration ผ่าน state

```rust
use serde::{Deserialize, Serialize};
use std::sync::Arc;

// Configuration struct
#[derive(Debug, Clone, Deserialize, Serialize)]
struct DatabaseConfig {
    url: String,
    max_connections: u32,
    min_connections: u32,
    connect_timeout_secs: u64,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
struct ServerConfig {
    host: String,
    port: u16,
    workers: Option<usize>,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
struct JwtConfig {
    secret: String,
    expires_in_secs: u64,
    refresh_expires_in_secs: u64,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
struct EmailConfig {
    smtp_host: String,
    smtp_port: u16,
    from_address: String,
    from_name: String,
    enabled: bool,
}

#[derive(Debug, Clone, Deserialize, Serialize)]
struct AppSettings {
    environment: String,
    debug: bool,
    database: DatabaseConfig,
    server: ServerConfig,
    jwt: JwtConfig,
    email: EmailConfig,
}

impl AppSettings {
    fn load() -> Result<Self, config::ConfigError> {
        let env = std::env::var("APP_ENV").unwrap_or_else(|_| "development".to_string());
        
        let settings = config::Config::builder()
            // Default config
            .add_source(config::File::with_name("config/default"))
            // Environment-specific config
            .add_source(config::File::with_name(&format!("config/{}", env)).required(false))
            // Environment variables (APP_ prefix)
            .add_source(config::Environment::with_prefix("APP").separator("__"))
            .build()?;
        
        settings.try_deserialize()
    }
    
    fn is_production(&self) -> bool {
        self.environment == "production"
    }
    
    fn is_development(&self) -> bool {
        self.environment == "development"
    }
}

// Config state ที่ immutable (ไม่ต้องใช้ lock)
type Config = Arc<AppSettings>;

async fn get_config_info(config: web::Data<Config>) -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({
        "environment": config.environment,
        "debug": config.debug,
        "server": {
            "host": config.server.host,
            "port": config.server.port
        }
    }))
}

// Dynamic configuration ที่เปลี่ยนได้ตอน runtime
use std::sync::RwLock;

struct DynamicConfig {
    feature_flags: RwLock<HashMap<String, bool>>,
    rate_limits: RwLock<HashMap<String, u32>>,
}

impl DynamicConfig {
    fn new() -> Self {
        let mut flags = HashMap::new();
        flags.insert("new_checkout".to_string(), false);
        flags.insert("ai_recommendations".to_string(), true);
        
        let mut limits = HashMap::new();
        limits.insert("api".to_string(), 100);
        limits.insert("upload".to_string(), 10);
        
        DynamicConfig {
            feature_flags: RwLock::new(flags),
            rate_limits: RwLock::new(limits),
        }
    }
    
    fn is_feature_enabled(&self, feature: &str) -> bool {
        self.feature_flags
            .read()
            .unwrap()
            .get(feature)
            .copied()
            .unwrap_or(false)
    }
    
    fn set_feature(&self, feature: String, enabled: bool) {
        self.feature_flags.write().unwrap().insert(feature, enabled);
    }
    
    fn get_rate_limit(&self, endpoint: &str) -> u32 {
        self.rate_limits
            .read()
            .unwrap()
            .get(endpoint)
            .copied()
            .unwrap_or(60)
    }
}

async fn toggle_feature(
    config: web::Data<DynamicConfig>,
    path: web::Path<String>,
    body: web::Json<serde_json::Value>,
) -> HttpResponse {
    let feature = path.into_inner();
    let enabled = body.get("enabled")
        .and_then(|v| v.as_bool())
        .unwrap_or(false);
    
    config.set_feature(feature.clone(), enabled);
    
    HttpResponse::Ok().json(serde_json::json!({
        "feature": feature,
        "enabled": enabled
    }))
}
```

---

## Cache State

การใช้ in-memory cache เป็น state

```rust
use std::sync::RwLock;
use std::collections::HashMap;
use std::time::{Duration, Instant};

// Cache entry
struct CacheEntry<V> {
    value: V,
    inserted_at: Instant,
    ttl: Duration,
}

impl<V> CacheEntry<V> {
    fn new(value: V, ttl: Duration) -> Self {
        CacheEntry {
            value,
            inserted_at: Instant::now(),
            ttl,
        }
    }
    
    fn is_expired(&self) -> bool {
        self.inserted_at.elapsed() > self.ttl
    }
}

// Simple in-memory cache
struct Cache<K, V> {
    data: RwLock<HashMap<K, CacheEntry<V>>>,
    default_ttl: Duration,
}

impl<K: std::hash::Hash + Eq + Clone, V: Clone> Cache<K, V> {
    fn new(default_ttl_secs: u64) -> Self {
        Cache {
            data: RwLock::new(HashMap::new()),
            default_ttl: Duration::from_secs(default_ttl_secs),
        }
    }
    
    fn get(&self, key: &K) -> Option<V> {
        let data = self.data.read().unwrap();
        data.get(key)
            .filter(|entry| !entry.is_expired())
            .map(|entry| entry.value.clone())
    }
    
    fn set(&self, key: K, value: V) {
        self.set_with_ttl(key, value, self.default_ttl);
    }
    
    fn set_with_ttl(&self, key: K, value: V, ttl: Duration) {
        let mut data = self.data.write().unwrap();
        data.insert(key, CacheEntry::new(value, ttl));
    }
    
    fn remove(&self, key: &K) -> Option<V> {
        let mut data = self.data.write().unwrap();
        data.remove(key).map(|e| e.value)
    }
    
    fn clear_expired(&self) {
        let mut data = self.data.write().unwrap();
        data.retain(|_, v| !v.is_expired());
    }
    
    fn size(&self) -> usize {
        self.data.read().unwrap().len()
    }
}

// ใช้กับ actix-web
type UserCache = Cache<u64, serde_json::Value>;

async fn get_user_cached(
    cache: web::Data<UserCache>,
    path: web::Path<u64>,
) -> HttpResponse {
    let user_id = path.into_inner();
    
    // ตรวจสอบ cache ก่อน
    if let Some(cached_user) = cache.get(&user_id) {
        return HttpResponse::Ok()
            .append_header(("X-Cache", "HIT"))
            .json(cached_user);
    }
    
    // ไม่มีใน cache - ดึงจาก database (simulated)
    let user = serde_json::json!({
        "id": user_id,
        "username": format!("user_{}", user_id),
        "email": format!("user{}@example.com", user_id),
        "created_at": "2024-01-01T00:00:00Z"
    });
    
    // บันทึกลง cache
    cache.set(user_id, user.clone());
    
    HttpResponse::Ok()
        .append_header(("X-Cache", "MISS"))
        .json(user)
}

async fn invalidate_user_cache(
    cache: web::Data<UserCache>,
    path: web::Path<u64>,
) -> HttpResponse {
    let user_id = path.into_inner();
    cache.remove(&user_id);
    
    HttpResponse::Ok().json(serde_json::json!({
        "message": format!("Cache invalidated for user {}", user_id)
    }))
}

async fn cache_stats(cache: web::Data<UserCache>) -> HttpResponse {
    cache.clear_expired();  // Clean up expired entries
    
    HttpResponse::Ok().json(serde_json::json!({
        "size": cache.size(),
        "default_ttl_secs": cache.default_ttl.as_secs()
    }))
}
```

---

## Counter Examples

ตัวอย่างการใช้ atomic counter และ shared counter

```rust
use std::sync::atomic::{AtomicU64, AtomicI64, Ordering};
use std::sync::Arc;

// Atomic counter - ไม่ต้องการ lock (สำหรับ simple counters)
struct AtomicCounters {
    total_requests: AtomicU64,
    active_connections: AtomicI64,
    errors_500: AtomicU64,
    errors_4xx: AtomicU64,
}

impl AtomicCounters {
    fn new() -> Self {
        AtomicCounters {
            total_requests: AtomicU64::new(0),
            active_connections: AtomicI64::new(0),
            errors_500: AtomicU64::new(0),
            errors_4xx: AtomicU64::new(0),
        }
    }
    
    fn increment_requests(&self) {
        self.total_requests.fetch_add(1, Ordering::Relaxed);
    }
    
    fn increment_active_connections(&self) {
        self.active_connections.fetch_add(1, Ordering::Relaxed);
    }
    
    fn decrement_active_connections(&self) {
        self.active_connections.fetch_sub(1, Ordering::Relaxed);
    }
    
    fn increment_errors_500(&self) {
        self.errors_500.fetch_add(1, Ordering::Relaxed);
    }
    
    fn increment_errors_4xx(&self) {
        self.errors_4xx.fetch_add(1, Ordering::Relaxed);
    }
    
    fn get_stats(&self) -> serde_json::Value {
        serde_json::json!({
            "total_requests": self.total_requests.load(Ordering::Relaxed),
            "active_connections": self.active_connections.load(Ordering::Relaxed),
            "errors_500": self.errors_500.load(Ordering::Relaxed),
            "errors_4xx": self.errors_4xx.load(Ordering::Relaxed)
        })
    }
}

async fn get_counters(counters: web::Data<AtomicCounters>) -> HttpResponse {
    counters.increment_requests();
    HttpResponse::Ok().json(counters.get_stats())
}

// Per-endpoint counters ด้วย HashMap
struct EndpointCounters {
    counts: RwLock<HashMap<String, u64>>,
}

impl EndpointCounters {
    fn new() -> Self {
        EndpointCounters {
            counts: RwLock::new(HashMap::new()),
        }
    }
    
    fn increment(&self, endpoint: &str) {
        let mut counts = self.counts.write().unwrap();
        *counts.entry(endpoint.to_string()).or_insert(0) += 1;
    }
    
    fn get_all(&self) -> HashMap<String, u64> {
        self.counts.read().unwrap().clone()
    }
    
    fn get(&self, endpoint: &str) -> u64 {
        self.counts.read().unwrap()
            .get(endpoint)
            .copied()
            .unwrap_or(0)
    }
}
```

---

## Multiple State Types

การใช้ state หลายประเภทพร้อมกัน

```rust
use actix_web::{web, App, HttpServer};

// แต่ละ state type แยกกัน
struct DatabaseState {
    pool: PgPool,
}

struct CacheState {
    users: Cache<u64, serde_json::Value>,
    posts: Cache<u64, serde_json::Value>,
}

struct MetricsState {
    counters: AtomicCounters,
    endpoint_hits: EndpointCounters,
}

#[derive(Clone)]
struct SharedConfig {
    jwt_secret: String,
    debug: bool,
}

// Handler ที่ใช้หลาย state types
async fn complex_handler(
    db: web::Data<DatabaseState>,
    cache: web::Data<CacheState>,
    metrics: web::Data<MetricsState>,
    config: web::Data<SharedConfig>,
    path: web::Path<u64>,
) -> HttpResponse {
    let user_id = path.into_inner();
    metrics.counters.increment_requests();
    metrics.endpoint_hits.increment("/users/{id}");
    
    // ตรวจ cache ก่อน
    if let Some(cached) = cache.users.get(&user_id) {
        return HttpResponse::Ok()
            .append_header(("X-Cache", "HIT"))
            .json(cached);
    }
    
    // ดึงจาก database
    let user = sqlx::query!(
        "SELECT id, username, email FROM users WHERE id = $1",
        user_id as i64
    )
    .fetch_optional(&db.pool)
    .await;
    
    match user {
        Ok(Some(row)) => {
            let user_json = serde_json::json!({
                "id": row.id,
                "username": row.username,
                "email": row.email
            });
            
            // บันทึก cache
            cache.users.set(user_id, user_json.clone());
            
            HttpResponse::Ok()
                .append_header(("X-Cache", "MISS"))
                .json(user_json)
        },
        Ok(None) => HttpResponse::NotFound().json(serde_json::json!({
            "error": "User not found"
        })),
        Err(e) => {
            log::error!("Database error: {}", e);
            metrics.counters.increment_errors_500();
            HttpResponse::InternalServerError().finish()
        }
    }
}

// App setup ที่มีหลาย state
async fn setup_multi_state_app() -> std::io::Result<()> {
    // สร้าง states
    let db_state = web::Data::new(DatabaseState {
        pool: setup_database().await,
    });
    
    let cache_state = web::Data::new(CacheState {
        users: Cache::new(300),    // 5 min TTL
        posts: Cache::new(60),     // 1 min TTL
    });
    
    let metrics_state = web::Data::new(MetricsState {
        counters: AtomicCounters::new(),
        endpoint_hits: EndpointCounters::new(),
    });
    
    let config = web::Data::new(SharedConfig {
        jwt_secret: std::env::var("JWT_SECRET").unwrap_or_default(),
        debug: std::env::var("DEBUG").map(|v| v == "true").unwrap_or(false),
    });
    
    HttpServer::new(move || {
        App::new()
            // ลงทะเบียน states ทั้งหมด
            .app_data(db_state.clone())
            .app_data(cache_state.clone())
            .app_data(metrics_state.clone())
            .app_data(config.clone())
            // Routes
            .route("/users/{id}", web::get().to(complex_handler))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **web::Data<T>** - วิธีการแชร์ state ผ่าน Arc
2. **AppState Pattern** - การรวม state ทั้งหมดไว้ใน struct เดียว
3. **Mutex vs RwLock** - เมื่อไหรที่ควรใช้อะไร
4. **Database Pool** - การใช้ sqlx PgPool เป็น state
5. **Configuration State** - การจัดการ config แบบ static และ dynamic
6. **Cache State** - In-memory caching pattern
7. **Atomic Counters** - ประสิทธิภาพสูงสำหรับ simple counters
8. **Multiple States** - การใช้ state หลายประเภทพร้อมกัน

---

## การนำทาง

- [← Part 024: Middleware in Actix-web](../part_024/README.md)
- [→ Part 026: Error Handling in Actix-web](../part_026/README.md)
- [กลับหน้าหลัก](../../README.md)
