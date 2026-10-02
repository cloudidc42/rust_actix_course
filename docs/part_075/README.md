# Part 075: Deployment to Cloud

## บทนำ

บทนี้จะครอบคลุมการ deploy แอปพลิเคชัน Rust + Actix-web ไปยัง cloud platforms ต่างๆ โดยเฉพาะ Fly.io และ Railway ซึ่งเป็นแพลตฟอร์มที่ง่ายต่อการใช้งาน

## 1. Deploy to Fly.io

### 1.1 ติดตั้ง flyctl

```bash
# macOS
brew install flyctl

# Linux
curl -L https://fly.io/install.sh | sh

# Windows (PowerShell)
pwsh -Command "iwr https://fly.io/install.ps1 -useb | iex"

# Login
fly auth login
```

### 1.2 fly.toml Configuration

```toml
# fly.toml
app = "my-rust-api"
primary_region = "sin"  # Singapore

[build]
  # ใช้ Dockerfile
  dockerfile = "Dockerfile"
  [build.args]
    VERSION = "1.0.0"

[env]
  APP_ENV = "production"
  RUST_LOG = "info"
  PORT = "8080"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = true
  auto_start_machines = true
  min_machines_running = 1
  
  [http_service.concurrency]
    type = "requests"
    soft_limit = 200
    hard_limit = 250

[[vm]]
  cpu_kind = "shared"
  cpus = 1
  memory_mb = 512

[checks]
  [checks.health]
    grace_period = "10s"
    interval = "30s"
    method = "GET"
    path = "/health"
    port = 8080
    restart_limit = 3
    timeout = "5s"
    type = "http"
```

### 1.3 Initial Deployment

```bash
# สร้าง app ใหม่
fly apps create my-rust-api

# กำหนด region
fly regions set sin nrt  # Singapore, Tokyo

# สร้าง PostgreSQL database
fly postgres create \
  --name my-rust-api-db \
  --region sin \
  --vm-size shared-cpu-1x \
  --volume-size 10

# Attach database กับ app
fly postgres attach my-rust-api-db --app my-rust-api

# ดู DATABASE_URL ที่สร้างมา
fly secrets list

# Deploy
fly deploy

# ดู logs
fly logs

# SSH เข้าไปใน container
fly ssh console
```

### 1.4 Environment Variables บน Fly.io

```bash
# ตั้งค่า secrets
fly secrets set \
  JWT_SECRET="your-super-secret-key" \
  REDIS_URL="redis://..." \
  SMTP_PASSWORD="smtp-password"

# ดู secrets (แค่ names)
fly secrets list

# ลบ secret
fly secrets unset OLD_SECRET

# ตั้งค่า environment variables (non-secret)
fly config env set RUST_LOG=info
```

### 1.5 Scale บน Fly.io

```bash
# Scale จำนวน instances
fly scale count 3

# Scale machine specs
fly scale vm shared-cpu-2x

# ดู machine status
fly status

# ดู metrics
fly dashboard
```

### 1.6 Zero-downtime Deployment บน Fly.io

```toml
# fly.toml
[deploy]
  strategy = "rolling"  # rolling, canary, bluegreen, immediate
  max_unavailable = 1
  wait_timeout = "5m"
```

```bash
# Deploy ด้วย strategy
fly deploy --strategy rolling

# Monitor deployment
fly deploy --wait-timeout 5m
```

## 2. Deploy to Railway

### 2.1 Setup Railway

```bash
# ติดตั้ง Railway CLI
npm install -g @railway/cli
# หรือ
curl -fsSL https://railway.app/install.sh | sh

# Login
railway login

# Initialize project
railway init
```

### 2.2 railway.toml Configuration

```toml
# railway.toml
[build]
  builder = "dockerfile"
  dockerfilePath = "./Dockerfile"

[deploy]
  startCommand = "./my-api"
  healthcheckPath = "/health"
  healthcheckTimeout = 30
  restartPolicyType = "on_failure"
  restartPolicyMaxRetries = 3
  
  [[deploy.environmentVariables]]
  name = "RUST_LOG"
  value = "info"
  
  [[deploy.environmentVariables]]
  name = "PORT"
  value = "8080"
```

### 2.3 Deploy บน Railway

```bash
# Deploy ไปยัง Railway
railway up

# กำหนด environment
railway up --environment production

# ดู logs
railway logs

# Open dashboard
railway open

# ตั้งค่า environment variables
railway variables set JWT_SECRET=your-secret

# Link กับ PostgreSQL service
railway add --plugin postgresql

# ดู DATABASE_URL
railway variables
```

### 2.4 Nixpacks (Railway's Builder)

```toml
# nixpacks.toml - ถ้าใช้ nixpacks แทน Dockerfile
[variables]
  RUST_VERSION = "1.74"

[phases.setup]
  nixPkgs = ["openssl", "pkg-config", "postgresql"]

[phases.build]
  cmds = ["cargo build --release"]

[start]
  cmd = "./target/release/my-api"
```

## 3. Environment Variables

### 3.1 Config Management ใน Rust

```rust
// src/config.rs
use serde::Deserialize;
use std::env;

#[derive(Debug, Deserialize, Clone)]
pub struct Config {
    // Database
    pub database_url: String,
    #[serde(default = "default_db_max_connections")]
    pub database_max_connections: u32,
    
    // Redis
    pub redis_url: Option<String>,
    
    // JWT
    pub jwt_secret: String,
    #[serde(default = "default_jwt_expiry")]
    pub jwt_expiry_hours: i64,
    
    // Server
    #[serde(default = "default_port")]
    pub port: u16,
    #[serde(default)]
    pub host: String,
    
    // App
    pub app_env: AppEnvironment,
    pub rust_log: Option<String>,
    
    // External services
    pub smtp_host: Option<String>,
    pub smtp_user: Option<String>,
    pub smtp_password: Option<String>,
    pub smtp_port: Option<u16>,
}

fn default_db_max_connections() -> u32 { 10 }
fn default_jwt_expiry() -> i64 { 24 }
fn default_port() -> u16 { 8080 }

#[derive(Debug, Deserialize, Clone, PartialEq)]
#[serde(rename_all = "lowercase")]
pub enum AppEnvironment {
    Development,
    Staging,
    Production,
}

impl Default for AppEnvironment {
    fn default() -> Self {
        AppEnvironment::Development
    }
}

impl Config {
    pub fn from_env() -> anyhow::Result<Self> {
        // โหลด .env file ถ้ามี (development)
        dotenvy::dotenv().ok();
        
        let config = config::Config::builder()
            .add_source(
                config::Environment::default()
                    .separator("__")
            )
            .build()?
            .try_deserialize()?;
        
        Ok(config)
    }
    
    pub fn is_production(&self) -> bool {
        self.app_env == AppEnvironment::Production
    }
    
    pub fn server_address(&self) -> String {
        format!("{}:{}", 
            if self.host.is_empty() { "0.0.0.0".to_string() } else { self.host.clone() },
            self.port
        )
    }
}
```

```toml
# Cargo.toml
[dependencies]
config = "0.13"
dotenvy = "0.15"
anyhow = "1"
```

### 3.2 Validation ของ Config

```rust
// src/config.rs
impl Config {
    pub fn validate(&self) -> anyhow::Result<()> {
        if self.jwt_secret.len() < 32 {
            anyhow::bail!("JWT_SECRET must be at least 32 characters");
        }
        
        if self.is_production() {
            // Checks ที่เข้มงวดสำหรับ production
            if !self.database_url.starts_with("postgres") {
                anyhow::bail!("DATABASE_URL must be PostgreSQL in production");
            }
            
            if self.jwt_secret.contains("secret") || self.jwt_secret.contains("test") {
                anyhow::bail!("JWT_SECRET appears to be a test value");
            }
        }
        
        Ok(())
    }
}

// ใน main.rs
#[actix_web::main]
async fn main() -> anyhow::Result<()> {
    let config = Config::from_env()?;
    config.validate()?;
    
    println!("Starting in {:?} mode on {}", config.app_env, config.server_address());
    
    Ok(())
}
```

## 4. Database Provisioning

### 4.1 Database Migrations on Deploy

```rust
// src/main.rs
use sqlx::PgPool;

async fn run_migrations(pool: &PgPool) -> anyhow::Result<()> {
    println!("Running database migrations...");
    
    sqlx::migrate!("./migrations")
        .run(pool)
        .await
        .map_err(|e| anyhow::anyhow!("Migration failed: {}", e))?;
    
    println!("Migrations complete");
    Ok(())
}

#[actix_web::main]
async fn main() -> anyhow::Result<()> {
    let config = Config::from_env()?;
    
    let pool = sqlx::postgres::PgPoolOptions::new()
        .max_connections(config.database_max_connections)
        .connect(&config.database_url)
        .await?;
    
    // Run migrations ก่อน start server
    run_migrations(&pool).await?;
    
    // Start server...
    Ok(())
}
```

### 4.2 Fly.io Database Setup

```bash
#!/bin/bash
# scripts/setup-fly-db.sh

APP_NAME="my-rust-api"
DB_NAME="${APP_NAME}-db"
REGION="sin"

echo "Creating PostgreSQL cluster..."
fly postgres create \
    --name "$DB_NAME" \
    --region "$REGION" \
    --vm-size shared-cpu-1x \
    --volume-size 10 \
    --initial-cluster-size 1

echo "Attaching to app..."
fly postgres attach "$DB_NAME" --app "$APP_NAME"

echo "Database URL has been set as DATABASE_URL secret"
fly secrets list --app "$APP_NAME"
```

### 4.3 Database Seeding

```rust
// src/seed.rs
use sqlx::PgPool;

pub async fn seed_database(pool: &PgPool) -> anyhow::Result<()> {
    // ตรวจสอบว่ามีข้อมูลอยู่แล้วหรือไม่
    let count: i64 = sqlx::query_scalar!("SELECT COUNT(*) FROM users")
        .fetch_one(pool)
        .await
        .unwrap_or(0);
    
    if count > 0 {
        println!("Database already seeded, skipping");
        return Ok(());
    }
    
    println!("Seeding database...");
    
    // Insert seed data
    sqlx::query!(
        "INSERT INTO users (email, name, role) VALUES ($1, $2, $3)",
        "admin@example.com",
        "Admin User",
        "admin"
    )
    .execute(pool)
    .await?;
    
    println!("Database seeded successfully");
    Ok(())
}
```

## 5. Health Check Configuration

### 5.1 Comprehensive Health Check

```rust
// src/handlers/health.rs
use actix_web::{web, HttpResponse};
use sqlx::PgPool;
use serde::Serialize;
use std::time::Instant;

#[derive(Serialize)]
struct HealthResponse {
    status: HealthStatus,
    checks: HealthChecks,
    version: String,
    environment: String,
    uptime_seconds: u64,
}

#[derive(Serialize)]
#[serde(rename_all = "lowercase")]
enum HealthStatus {
    Healthy,
    Degraded,
    Unhealthy,
}

#[derive(Serialize)]
struct HealthChecks {
    database: CheckResult,
    cache: Option<CheckResult>,
}

#[derive(Serialize)]
struct CheckResult {
    status: String,
    response_time_ms: u64,
    message: Option<String>,
}

static APP_START: std::sync::OnceLock<Instant> = std::sync::OnceLock::new();

pub fn init_start_time() {
    APP_START.get_or_init(Instant::now);
}

pub async fn health(
    db: web::Data<PgPool>,
    config: web::Data<crate::config::Config>,
) -> HttpResponse {
    let start = APP_START.get_or_init(Instant::now);
    
    // Check database
    let db_start = Instant::now();
    let db_result = sqlx::query!("SELECT 1 AS x")
        .fetch_one(db.get_ref())
        .await;
    let db_time = db_start.elapsed().as_millis() as u64;
    
    let db_check = CheckResult {
        status: if db_result.is_ok() { "healthy".to_string() } else { "unhealthy".to_string() },
        response_time_ms: db_time,
        message: db_result.err().map(|e| e.to_string()),
    };
    
    let overall_status = if db_result.is_ok() {
        HealthStatus::Healthy
    } else {
        HealthStatus::Unhealthy
    };
    
    let response = HealthResponse {
        status: overall_status,
        checks: HealthChecks {
            database: db_check,
            cache: None,
        },
        version: env!("CARGO_PKG_VERSION").to_string(),
        environment: format!("{:?}", config.app_env),
        uptime_seconds: start.elapsed().as_secs(),
    };
    
    if db_result.is_ok() {
        HttpResponse::Ok().json(response)
    } else {
        HttpResponse::ServiceUnavailable().json(response)
    }
}

// Readiness check (สำหรับ Kubernetes/load balancer)
pub async fn ready(db: web::Data<PgPool>) -> HttpResponse {
    match sqlx::query!("SELECT 1 AS x").fetch_one(db.get_ref()).await {
        Ok(_) => HttpResponse::Ok().body("ready"),
        Err(_) => HttpResponse::ServiceUnavailable().body("not ready"),
    }
}

// Liveness check (app ยังทำงานอยู่ไหม)
pub async fn live() -> HttpResponse {
    HttpResponse::Ok().body("alive")
}
```

## 6. Zero-downtime Deployment

### 6.1 Graceful Shutdown

```rust
// src/main.rs
use actix_web::{web, App, HttpServer};
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;

static SHUTTING_DOWN: AtomicBool = AtomicBool::new(false);

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let db_pool = setup_database().await;
    let db_data = web::Data::new(db_pool);
    
    // Handle shutdown signals
    let shutdown = Arc::new(AtomicBool::new(false));
    let shutdown_clone = Arc::clone(&shutdown);
    
    ctrlc::set_handler(move || {
        println!("Received shutdown signal, gracefully stopping...");
        shutdown_clone.store(true, Ordering::SeqCst);
    })
    .expect("Error setting Ctrl-C handler");
    
    let server = HttpServer::new(move || {
        App::new()
            .app_data(db_data.clone())
            .route("/health", web::get().to(crate::handlers::health::health))
            .route("/ready", web::get().to(crate::handlers::health::ready))
    })
    .shutdown_timeout(30)  // รอ 30 วินาทีก่อน force kill
    .bind("0.0.0.0:8080")?
    .run();
    
    println!("Server started on port 8080");
    
    server.await
}

async fn setup_database() -> sqlx::PgPool {
    let database_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL required");
    
    sqlx::postgres::PgPoolOptions::new()
        .max_connections(10)
        .connect(&database_url)
        .await
        .expect("Failed to connect to database")
}
```

### 6.2 Rolling Deployment Script

```bash
#!/bin/bash
# scripts/deploy.sh
set -e

APP_NAME="my-rust-api"
NEW_VERSION=$1

if [ -z "$NEW_VERSION" ]; then
    echo "Usage: $0 <version>"
    exit 1
fi

echo "Deploying $APP_NAME version $NEW_VERSION..."

# Deploy with rolling strategy
fly deploy \
    --app "$APP_NAME" \
    --strategy rolling \
    --wait-timeout 300

# Verify deployment
echo "Verifying deployment..."
sleep 10

HEALTH=$(curl -sf "https://${APP_NAME}.fly.dev/health" | jq -r '.status' 2>/dev/null || echo "error")

if [ "$HEALTH" = "healthy" ]; then
    echo "✅ Deployment successful! Version $NEW_VERSION is healthy."
else
    echo "❌ Health check failed! Rolling back..."
    fly releases list --app "$APP_NAME"
    
    # Rollback to previous version
    PREV_VERSION=$(fly releases list --app "$APP_NAME" --json | jq -r '.[1].Version')
    fly deploy --app "$APP_NAME" --image "registry.fly.io/${APP_NAME}:${PREV_VERSION}"
    
    echo "Rolled back to version $PREV_VERSION"
    exit 1
fi
```

## 7. Rollback Strategy

### 7.1 Fly.io Rollback

```bash
# ดู history ของ releases
fly releases list --app my-rust-api

# Rollback ไปยัง version ก่อนหน้า
fly deploy --image "registry.fly.io/my-rust-api:v1.2.3"

# หรือ rollback ทันที
fly releases rollback 3  # ย้อนไป release #3
```

### 7.2 Database Migration Rollback

```sql
-- migrations/20231201000001_add_users.sql
-- สร้าง migration ที่ reversible

-- UP
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

```sql
-- migrations/20231201000001_add_users_down.sql  
-- DOWN migration
DROP INDEX IF EXISTS idx_users_email;
DROP TABLE IF EXISTS users;
```

```rust
// การ handle migration failures
async fn run_migrations_safe(pool: &PgPool) -> anyhow::Result<()> {
    // ตรวจสอบ migration status ก่อน
    let migrator = sqlx::migrate!("./migrations");
    
    // ดู pending migrations
    let pending = migrator.migrations.iter()
        .filter(|m| !m.migration_type.is_down_migration())
        .count();
    
    println!("Pending migrations: {}", pending);
    
    // Run migrations
    match migrator.run(pool).await {
        Ok(_) => {
            println!("Migrations completed successfully");
            Ok(())
        }
        Err(e) => {
            eprintln!("Migration failed: {}", e);
            // ส่ง alert ไปยัง monitoring system
            anyhow::bail!("Database migration failed: {}", e)
        }
    }
}
```

## 8. Practical: Deploy Complete App

### 8.1 Complete Application สำหรับ Deploy

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware};
use std::time::Duration;

mod config;
mod handlers;
mod models;
mod db;
mod errors;

#[actix_web::main]
async fn main() -> anyhow::Result<()> {
    // Initialize logging
    let config = config::Config::from_env()?;
    
    let log_level = config.rust_log
        .as_deref()
        .unwrap_or("info");
    
    env_logger::Builder::new()
        .parse_env(log_level)
        .init();
    
    log::info!("Starting {} in {:?} mode", 
        env!("CARGO_PKG_NAME"),
        config.app_env
    );
    
    // Validate configuration
    config.validate()?;
    
    // Initialize start time for health checks
    handlers::health::init_start_time();
    
    // Connect to database
    log::info!("Connecting to database...");
    let pool = db::create_pool(&config).await?;
    
    // Run migrations
    log::info!("Running migrations...");
    db::run_migrations(&pool).await?;
    
    // Seed database in development
    if !config.is_production() {
        db::seed(&pool).await?;
    }
    
    let config_data = web::Data::new(config.clone());
    let pool_data = web::Data::new(pool);
    
    log::info!("Server starting on {}", config.server_address());
    
    HttpServer::new(move || {
        App::new()
            .app_data(config_data.clone())
            .app_data(pool_data.clone())
            .wrap(middleware::Logger::new(
                "%a \"%r\" %s %b %T"
            ))
            .wrap(middleware::Compress::default())
            .wrap(
                actix_web::middleware::DefaultHeaders::new()
                    .add(("X-Version", env!("CARGO_PKG_VERSION")))
            )
            // Health endpoints
            .route("/health", web::get().to(handlers::health::health))
            .route("/ready", web::get().to(handlers::health::ready))
            .route("/live", web::get().to(handlers::health::live))
            // API routes
            .service(
                web::scope("/api/v1")
                    .configure(handlers::products::configure)
                    .configure(handlers::users::configure)
                    .configure(handlers::orders::configure)
            )
    })
    .shutdown_timeout(30)
    .workers(num_cpus::get())
    .bind(config.server_address())?
    .run()
    .await?;
    
    Ok(())
}
```

### 8.2 Deployment Checklist Script

```bash
#!/bin/bash
# scripts/pre-deploy-check.sh

set -e

echo "Running pre-deployment checks..."

# 1. Run tests
echo "Running tests..."
cargo test --release

# 2. Check for security vulnerabilities
echo "Checking security..."
cargo audit --deny warnings

# 3. Check code quality
echo "Checking code quality..."
cargo clippy --all-targets --all-features -- -D warnings

# 4. Check formatting
echo "Checking formatting..."
cargo fmt -- --check

# 5. Build release binary
echo "Building release..."
cargo build --release

# 6. Test Docker build
echo "Testing Docker build..."
docker build -t my-api:test .
docker run --rm my-api:test /app/my-api --version

echo "✅ All checks passed! Ready to deploy."
```

### 8.3 fly.toml สมบูรณ์

```toml
# fly.toml - Production ready
app = "my-rust-api"
primary_region = "sin"

[build]
  dockerfile = "Dockerfile"

[env]
  APP_ENV = "production"
  RUST_LOG = "info,actix_web=warn"
  PORT = "8080"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = false
  auto_start_machines = true
  min_machines_running = 2

  [http_service.concurrency]
    type = "requests"
    soft_limit = 200
    hard_limit = 250
  
  [[http_service.checks]]
    grace_period = "15s"
    interval = "30s"
    method = "GET"
    path = "/health"
    port = 8080
    timeout = "5s"
    type = "http"

[[vm]]
  cpu_kind = "shared"
  cpus = 2
  memory_mb = 1024

[deploy]
  strategy = "rolling"
  max_unavailable = 1

[[statics]]
  guest_path = "/app/static"
  url_prefix = "/static"
```

### 8.4 Monitoring Setup

```rust
// src/middleware/metrics.rs
use actix_web::{dev::{Service, ServiceRequest, ServiceResponse, Transform}, Error};
use futures::future::{LocalBoxFuture, Ready, ok};
use std::time::Instant;

pub struct MetricsMiddleware;

impl<S, B> Transform<S, ServiceRequest> for MetricsMiddleware
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Transform = MetricsService<S>;
    type InitError = ();
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ok(MetricsService { service })
    }
}

pub struct MetricsService<S> {
    service: S,
}

impl<S, B> Service<ServiceRequest> for MetricsService<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    actix_web::dev::forward_ready!(service);

    fn call(&self, req: ServiceRequest) -> Self::Future {
        let start = Instant::now();
        let method = req.method().to_string();
        let path = req.path().to_owned();
        
        let fut = self.service.call(req);
        
        Box::pin(async move {
            let res = fut.await?;
            let duration = start.elapsed();
            let status = res.status().as_u16();
            
            // Log metrics
            log::debug!(
                "method={} path={} status={} duration_ms={}",
                method,
                path,
                status,
                duration.as_millis()
            );
            
            Ok(res)
        })
    }
}
```

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Fly.io** - deploy ด้วย flyctl และ fly.toml
2. **Railway** - deploy แบบ minimal configuration
3. **Environment variables** - จัดการ config อย่างปลอดภัย
4. **Database provisioning** - migrations on deploy
5. **Health checks** - comprehensive health monitoring
6. **Zero-downtime** - rolling deployments
7. **Rollback** - strategy และ procedures
8. **Complete deployment** - จาก local ถึง production

---

[⬅️ Part 074: CI/CD with GitHub Actions](../part_074/README.md) | [➡️ Part 076: Kubernetes Basics](../part_076/README.md)
