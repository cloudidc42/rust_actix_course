# Part 068: Configuration Management in Rust

## ภาพรวม

Configuration Management ที่ดีทำให้ application deploy ได้ในสภาพแวดล้อมต่าง ๆ (dev/staging/prod) โดยไม่ต้องแก้ไขโค้ด บทนี้ครอบคลุมการใช้ config crate, dotenv, secrets management และ feature flags

## Cargo.toml

```toml
[package]
name = "config-example"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
config = "0.13"
dotenv = "0.15"
validator = { version = "0.16", features = ["derive"] }
tracing = "0.1"
tracing-subscriber = "0.3"
thiserror = "1"
```

## Configuration Structure

```rust
// src/config/mod.rs
use serde::{Deserialize, Serialize};
use std::time::Duration;
use validator::Validate;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("Configuration loading error: {0}")]
    LoadError(#[from] config::ConfigError),
    #[error("Validation error: {0}")]
    ValidationError(String),
    #[error("Missing required config: {0}")]
    MissingConfig(String),
}

/// Root config structure
#[derive(Debug, Deserialize, Validate, Clone)]
pub struct AppConfig {
    #[validate(nested)]
    pub server: ServerConfig,
    
    #[validate(nested)]
    pub database: DatabaseConfig,
    
    #[validate(nested)]
    pub redis: RedisConfig,
    
    #[validate(nested)]
    pub auth: AuthConfig,
    
    pub logging: LoggingConfig,
    
    pub features: FeatureFlags,
    
    pub environment: Environment,
}

#[derive(Debug, Deserialize, Validate, Clone)]
pub struct ServerConfig {
    #[validate(range(min = 1, max = 65535))]
    pub port: u16,
    pub host: String,
    #[serde(default = "default_workers")]
    pub workers: usize,
    #[serde(default = "default_timeout_secs")]
    pub request_timeout_secs: u64,
    #[serde(default = "default_max_connections")]
    pub max_connections: usize,
}

fn default_workers() -> usize { num_cpus::get() }
fn default_timeout_secs() -> u64 { 30 }
fn default_max_connections() -> usize { 1000 }

impl ServerConfig {
    pub fn bind_address(&self) -> String {
        format!("{}:{}", self.host, self.port)
    }
    
    pub fn request_timeout(&self) -> Duration {
        Duration::from_secs(self.request_timeout_secs)
    }
}

#[derive(Debug, Deserialize, Validate, Clone)]
pub struct DatabaseConfig {
    #[validate(url)]
    pub url: String,
    #[serde(default = "default_pool_min")]
    pub pool_min_connections: u32,
    #[serde(default = "default_pool_max")]
    pub pool_max_connections: u32,
    #[serde(default = "default_connect_timeout")]
    pub connect_timeout_secs: u64,
    #[serde(default = "default_idle_timeout")]
    pub idle_timeout_secs: u64,
    #[serde(default)]
    pub ssl_mode: SslMode,
}

fn default_pool_min() -> u32 { 2 }
fn default_pool_max() -> u32 { 10 }
fn default_connect_timeout() -> u64 { 30 }
fn default_idle_timeout() -> u64 { 600 }

#[derive(Debug, Deserialize, Clone, Default)]
#[serde(rename_all = "lowercase")]
pub enum SslMode {
    #[default]
    Prefer,
    Disable,
    Require,
    VerifyFull,
}

#[derive(Debug, Deserialize, Validate, Clone)]
pub struct RedisConfig {
    pub url: String,
    #[serde(default = "default_redis_pool")]
    pub pool_size: u32,
    #[serde(default = "default_redis_timeout")]
    pub timeout_secs: u64,
}

fn default_redis_pool() -> u32 { 5 }
fn default_redis_timeout() -> u64 { 5 }

#[derive(Debug, Deserialize, Validate, Clone)]
pub struct AuthConfig {
    #[validate(length(min = 32, message = "JWT secret must be at least 32 characters"))]
    pub jwt_secret: String,
    #[serde(default = "default_jwt_expiry")]
    pub jwt_expiry_hours: u64,
    #[serde(default = "default_refresh_expiry")]
    pub refresh_token_expiry_days: u64,
    pub allowed_origins: Vec<String>,
}

fn default_jwt_expiry() -> u64 { 24 }
fn default_refresh_expiry() -> u64 { 30 }

impl AuthConfig {
    pub fn jwt_expiry(&self) -> Duration {
        Duration::from_secs(self.jwt_expiry_hours * 3600)
    }
}

#[derive(Debug, Deserialize, Clone)]
pub struct LoggingConfig {
    #[serde(default = "default_log_level")]
    pub level: String,
    #[serde(default)]
    pub format: LogFormat,
    #[serde(default)]
    pub output: LogOutput,
}

fn default_log_level() -> String { "info".to_string() }

#[derive(Debug, Deserialize, Clone, Default)]
#[serde(rename_all = "lowercase")]
pub enum LogFormat {
    #[default]
    Pretty,
    Json,
    Compact,
}

#[derive(Debug, Deserialize, Clone, Default)]
#[serde(rename_all = "lowercase")]
pub enum LogOutput {
    #[default]
    Stdout,
    Stderr,
    File { path: String },
}

/// Feature flags สำหรับเปิด/ปิด features
#[derive(Debug, Deserialize, Clone)]
pub struct FeatureFlags {
    #[serde(default)]
    pub enable_registration: bool,
    #[serde(default)]
    pub enable_email_verification: bool,
    #[serde(default = "default_true")]
    pub enable_rate_limiting: bool,
    #[serde(default)]
    pub enable_experimental_api: bool,
    #[serde(default = "default_true")]
    pub enable_cache: bool,
    #[serde(default)]
    pub maintenance_mode: bool,
}

fn default_true() -> bool { true }

impl Default for FeatureFlags {
    fn default() -> Self {
        FeatureFlags {
            enable_registration: true,
            enable_email_verification: false,
            enable_rate_limiting: true,
            enable_experimental_api: false,
            enable_cache: true,
            maintenance_mode: false,
        }
    }
}

#[derive(Debug, Deserialize, Clone, PartialEq)]
#[serde(rename_all = "lowercase")]
pub enum Environment {
    Development,
    Staging,
    Production,
}

impl Environment {
    pub fn is_production(&self) -> bool {
        matches!(self, Environment::Production)
    }
    
    pub fn is_development(&self) -> bool {
        matches!(self, Environment::Development)
    }
}

impl std::fmt::Display for Environment {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Environment::Development => write!(f, "development"),
            Environment::Staging => write!(f, "staging"),
            Environment::Production => write!(f, "production"),
        }
    }
}
```

## Config Loader

```rust
// src/config/loader.rs
use config::{Config, ConfigBuilder, Environment, File};
use std::env;

use super::{AppConfig, ConfigError};

pub struct ConfigLoader;

impl ConfigLoader {
    pub fn load() -> Result<AppConfig, ConfigError> {
        // Load .env file ถ้ามี
        dotenv::dotenv().ok();
        
        let environment = env::var("APP_ENV")
            .unwrap_or_else(|_| "development".to_string());
        
        tracing::info!("Loading configuration for environment: {}", environment);
        
        let config = Config::builder()
            // Base config (defaults)
            .add_source(File::with_name("config/default").required(false))
            
            // Environment-specific config
            .add_source(File::with_name(&format!("config/{}", environment)).required(false))
            
            // Local overrides (ไม่ commit)
            .add_source(File::with_name("config/local").required(false))
            
            // Environment variables (highest priority)
            // APP_SERVER_PORT=8080 maps to server.port
            .add_source(
                Environment::with_prefix("APP")
                    .prefix_separator("_")
                    .separator("__")
            )
            .build()
            .map_err(ConfigError::LoadError)?;
        
        let app_config: AppConfig = config.try_deserialize()
            .map_err(ConfigError::LoadError)?;
        
        // Validate config
        use validator::Validate;
        app_config.validate()
            .map_err(|e| ConfigError::ValidationError(e.to_string()))?;
        
        Ok(app_config)
    }
}
```

## Config Files

```yaml
# config/default.yaml
server:
  host: "0.0.0.0"
  port: 8080
  workers: 4
  request_timeout_secs: 30
  max_connections: 1000

database:
  url: "postgres://postgres:password@localhost/myapp"
  pool_min_connections: 2
  pool_max_connections: 10
  connect_timeout_secs: 30

redis:
  url: "redis://localhost:6379"
  pool_size: 5

auth:
  jwt_expiry_hours: 24
  refresh_token_expiry_days: 30
  allowed_origins:
    - "http://localhost:3000"

logging:
  level: "info"
  format: "pretty"

features:
  enable_registration: true
  enable_rate_limiting: true
  enable_cache: true

environment: development
```

```yaml
# config/production.yaml
server:
  workers: 8
  request_timeout_secs: 60

database:
  pool_min_connections: 5
  pool_max_connections: 50
  ssl_mode: "require"

logging:
  level: "warn"
  format: "json"

features:
  enable_email_verification: true
  enable_rate_limiting: true

environment: production
```

```yaml
# config/staging.yaml
server:
  port: 8080

logging:
  level: "debug"
  format: "json"

features:
  enable_experimental_api: true

environment: staging
```

## Secrets Management

```rust
// src/config/secrets.rs
use std::env;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum SecretError {
    #[error("Secret not found: {0}")]
    NotFound(String),
    #[error("Invalid secret format: {0}")]
    InvalidFormat(String),
}

/// Interface สำหรับ secrets provider
pub trait SecretsProvider: Send + Sync {
    fn get_secret(&self, key: &str) -> Result<String, SecretError>;
}

/// Environment variable secrets provider
pub struct EnvSecretsProvider;

impl SecretsProvider for EnvSecretsProvider {
    fn get_secret(&self, key: &str) -> Result<String, SecretError> {
        env::var(key).map_err(|_| SecretError::NotFound(key.to_string()))
    }
}

/// File-based secrets (สำหรับ Kubernetes secrets mounted as files)
pub struct FileSecretsProvider {
    base_path: std::path::PathBuf,
}

impl FileSecretsProvider {
    pub fn new(base_path: impl Into<std::path::PathBuf>) -> Self {
        FileSecretsProvider { base_path: base_path.into() }
    }
}

impl SecretsProvider for FileSecretsProvider {
    fn get_secret(&self, key: &str) -> Result<String, SecretError> {
        let file_path = self.base_path.join(key);
        std::fs::read_to_string(&file_path)
            .map(|s| s.trim().to_string())
            .map_err(|_| SecretError::NotFound(key.to_string()))
    }
}

/// Secrets Manager wrapper ที่ลอง providers หลายตัว
pub struct SecretsManager {
    providers: Vec<Box<dyn SecretsProvider>>,
}

impl SecretsManager {
    pub fn new() -> Self {
        SecretsManager { providers: Vec::new() }
    }
    
    pub fn add_provider(mut self, provider: Box<dyn SecretsProvider>) -> Self {
        self.providers.push(provider);
        self
    }
    
    pub fn get_secret(&self, key: &str) -> Result<String, SecretError> {
        for provider in &self.providers {
            match provider.get_secret(key) {
                Ok(secret) => return Ok(secret),
                Err(SecretError::NotFound(_)) => continue,
                Err(e) => return Err(e),
            }
        }
        Err(SecretError::NotFound(key.to_string()))
    }
    
    pub fn get_secret_or_default(&self, key: &str, default: &str) -> String {
        self.get_secret(key).unwrap_or_else(|_| default.to_string())
    }
}

/// ใช้ SecretsManager เพื่อโหลด sensitive config
pub fn load_secrets(config: &mut super::AppConfig) -> Result<(), SecretError> {
    let secrets = SecretsManager::new()
        .add_provider(Box::new(EnvSecretsProvider))
        .add_provider(Box::new(FileSecretsProvider::new("/run/secrets")));
    
    // Override JWT secret จาก secrets provider
    if let Ok(jwt_secret) = secrets.get_secret("JWT_SECRET") {
        config.auth.jwt_secret = jwt_secret;
    }
    
    // Override database URL
    if let Ok(db_url) = secrets.get_secret("DATABASE_URL") {
        config.database.url = db_url;
    }
    
    Ok(())
}
```

## Feature Flags Runtime

```rust
// src/config/feature_flags.rs
use std::sync::{Arc, RwLock};
use std::collections::HashMap;

/// Feature flags ที่สามารถเปลี่ยนแปลงได้ runtime
pub struct DynamicFeatureFlags {
    flags: RwLock<HashMap<String, bool>>,
}

impl DynamicFeatureFlags {
    pub fn new(initial: HashMap<String, bool>) -> Self {
        DynamicFeatureFlags {
            flags: RwLock::new(initial),
        }
    }
    
    pub fn from_config(config: &super::FeatureFlags) -> Self {
        let mut flags = HashMap::new();
        flags.insert("enable_registration".to_string(), config.enable_registration);
        flags.insert("enable_email_verification".to_string(), config.enable_email_verification);
        flags.insert("enable_rate_limiting".to_string(), config.enable_rate_limiting);
        flags.insert("enable_experimental_api".to_string(), config.enable_experimental_api);
        flags.insert("enable_cache".to_string(), config.enable_cache);
        flags.insert("maintenance_mode".to_string(), config.maintenance_mode);
        
        DynamicFeatureFlags::new(flags)
    }
    
    pub fn is_enabled(&self, flag: &str) -> bool {
        self.flags.read().unwrap()
            .get(flag)
            .copied()
            .unwrap_or(false)
    }
    
    pub fn set_flag(&self, flag: &str, enabled: bool) {
        let mut flags = self.flags.write().unwrap();
        flags.insert(flag.to_string(), enabled);
        tracing::info!("Feature flag '{}' set to {}", flag, enabled);
    }
    
    pub fn get_all(&self) -> HashMap<String, bool> {
        self.flags.read().unwrap().clone()
    }
}

// Macro สำหรับสะดวกในการใช้
#[macro_export]
macro_rules! feature_enabled {
    ($flags:expr, $feature:expr) => {
        $flags.is_enabled($feature)
    };
}

// Guard middleware สำหรับ feature flags
use actix_web::{web, HttpRequest, HttpResponse, dev::ServiceRequest};

pub async fn maintenance_mode_guard(
    req: ServiceRequest,
    flags: web::Data<Arc<DynamicFeatureFlags>>,
    srv: actix_web::dev::Service<ServiceRequest>,
) -> Result<actix_web::dev::ServiceResponse, actix_web::Error> {
    // Skip maintenance mode for health checks
    if req.path() == "/health" || req.path() == "/ready" {
        return srv.call(req).await;
    }
    
    if flags.is_enabled("maintenance_mode") {
        let response = HttpResponse::ServiceUnavailable()
            .json(serde_json::json!({
                "error": "maintenance_mode",
                "message": "Service is under maintenance. Please try again later."
            }));
        
        return Ok(req.into_response(response));
    }
    
    srv.call(req).await
}
```

## Hot Reloading Config

```rust
// src/config/hot_reload.rs
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::watch;
use tokio::time::interval;

use super::AppConfig;

pub struct HotReloadConfig {
    config: Arc<parking_lot::RwLock<AppConfig>>,
    reload_interval: Duration,
}

impl HotReloadConfig {
    pub fn new(initial: AppConfig, reload_interval: Duration) -> Self {
        HotReloadConfig {
            config: Arc::new(parking_lot::RwLock::new(initial)),
            reload_interval,
        }
    }
    
    pub fn get(&self) -> AppConfig {
        self.config.read().clone()
    }
    
    /// Background task ที่ reload config เป็นระยะ
    pub fn start_auto_reload(&self) -> tokio::task::JoinHandle<()> {
        let config = Arc::clone(&self.config);
        let interval_duration = self.reload_interval;
        
        tokio::spawn(async move {
            let mut ticker = interval(interval_duration);
            
            loop {
                ticker.tick().await;
                
                match super::loader::ConfigLoader::load() {
                    Ok(new_config) => {
                        let mut current = config.write();
                        // อัพเดทเฉพาะ feature flags (safe to hot reload)
                        current.features = new_config.features;
                        current.logging = new_config.logging;
                        tracing::info!("Configuration reloaded successfully");
                    }
                    Err(e) => {
                        tracing::error!("Failed to reload config: {}", e);
                    }
                }
            }
        })
    }
}

/// Watch-based config change notification
pub struct ConfigWatcher {
    sender: watch::Sender<AppConfig>,
    receiver: watch::Receiver<AppConfig>,
}

impl ConfigWatcher {
    pub fn new(initial: AppConfig) -> Self {
        let (sender, receiver) = watch::channel(initial);
        ConfigWatcher { sender, receiver }
    }
    
    pub fn subscribe(&self) -> watch::Receiver<AppConfig> {
        self.receiver.clone()
    }
    
    pub fn update(&self, config: AppConfig) {
        let _ = self.sender.send(config);
    }
}
```

## HTTP Handlers

```rust
// src/interface/http/config_handler.rs
use actix_web::{web, HttpResponse};
use std::sync::Arc;

use crate::config::feature_flags::DynamicFeatureFlags;
use crate::config::AppConfig;

/// Admin endpoint ดู config ปัจจุบัน (ซ่อน secrets)
pub async fn get_config(
    config: web::Data<Arc<AppConfig>>,
) -> HttpResponse {
    // ซ่อน sensitive data
    let safe_config = serde_json::json!({
        "environment": config.environment.to_string(),
        "server": {
            "host": config.server.host,
            "port": config.server.port,
            "workers": config.server.workers,
        },
        "database": {
            "pool_min": config.database.pool_min_connections,
            "pool_max": config.database.pool_max_connections,
            // ซ่อน URL
        },
        "features": config.features,
        "logging": {
            "level": config.logging.level,
        }
    });
    
    HttpResponse::Ok().json(safe_config)
}

/// Admin endpoint ดู feature flags
pub async fn get_feature_flags(
    flags: web::Data<Arc<DynamicFeatureFlags>>,
) -> HttpResponse {
    HttpResponse::Ok().json(flags.get_all())
}

/// Admin endpoint toggle feature flag
#[derive(serde::Deserialize)]
pub struct SetFlagRequest {
    pub enabled: bool,
}

pub async fn set_feature_flag(
    flags: web::Data<Arc<DynamicFeatureFlags>>,
    path: web::Path<String>,
    body: web::Json<SetFlagRequest>,
) -> HttpResponse {
    let flag_name = path.into_inner();
    flags.set_flag(&flag_name, body.enabled);
    
    HttpResponse::Ok().json(serde_json::json!({
        "flag": flag_name,
        "enabled": body.enabled,
        "message": format!("Feature flag '{}' updated", flag_name)
    }))
}
```

## Main with Config

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware};
use std::sync::Arc;

mod config;
use config::{loader::ConfigLoader, feature_flags::DynamicFeatureFlags};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // Load .env
    dotenv::dotenv().ok();
    
    // Load configuration
    let mut app_config = ConfigLoader::load()
        .expect("Failed to load configuration");
    
    // Load secrets
    config::secrets::load_secrets(&mut app_config).ok();
    
    // Setup logging based on config
    setup_logging(&app_config.logging);
    
    tracing::info!(
        "Starting server in {} mode on {}",
        app_config.environment,
        app_config.server.bind_address()
    );
    
    // Feature flags
    let feature_flags = Arc::new(DynamicFeatureFlags::from_config(&app_config.features));
    let config_arc = Arc::new(app_config.clone());
    
    let bind_address = app_config.server.bind_address();
    let workers = app_config.server.workers;
    
    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(Arc::clone(&config_arc)))
            .app_data(web::Data::new(Arc::clone(&feature_flags)))
            .wrap(middleware::Logger::default())
            // Admin endpoints
            .service(
                web::scope("/admin")
                    .route("/config", web::get().to(interface::http::config_handler::get_config))
                    .route("/features", web::get().to(interface::http::config_handler::get_feature_flags))
                    .route(
                        "/features/{flag}",
                        web::put().to(interface::http::config_handler::set_feature_flag)
                    )
            )
    })
    .workers(workers)
    .bind(&bind_address)?
    .run()
    .await
}

fn setup_logging(config: &config::LoggingConfig) {
    use tracing_subscriber::{EnvFilter, fmt};
    
    let filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new(&config.level));
    
    match config.format {
        config::LogFormat::Json => {
            fmt().json().with_env_filter(filter).init();
        }
        config::LogFormat::Pretty => {
            fmt().pretty().with_env_filter(filter).init();
        }
        config::LogFormat::Compact => {
            fmt().compact().with_env_filter(filter).init();
        }
    }
}
```

## Config Validation Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use validator::Validate;
    
    fn make_valid_config() -> AppConfig {
        AppConfig {
            server: ServerConfig {
                port: 8080,
                host: "0.0.0.0".to_string(),
                workers: 4,
                request_timeout_secs: 30,
                max_connections: 1000,
            },
            database: DatabaseConfig {
                url: "postgres://user:pass@localhost/db".to_string(),
                pool_min_connections: 2,
                pool_max_connections: 10,
                connect_timeout_secs: 30,
                idle_timeout_secs: 600,
                ssl_mode: SslMode::Prefer,
            },
            redis: RedisConfig {
                url: "redis://localhost:6379".to_string(),
                pool_size: 5,
                timeout_secs: 5,
            },
            auth: AuthConfig {
                jwt_secret: "a".repeat(32), // 32 chars minimum
                jwt_expiry_hours: 24,
                refresh_token_expiry_days: 30,
                allowed_origins: vec!["http://localhost:3000".to_string()],
            },
            logging: LoggingConfig {
                level: "info".to_string(),
                format: LogFormat::Pretty,
                output: LogOutput::Stdout,
            },
            features: FeatureFlags::default(),
            environment: Environment::Development,
        }
    }
    
    #[test]
    fn test_valid_config() {
        let config = make_valid_config();
        assert!(config.validate().is_ok());
    }
    
    #[test]
    fn test_invalid_port() {
        let mut config = make_valid_config();
        config.server.port = 0; // port 0 is invalid
        assert!(config.validate().is_err());
    }
    
    #[test]
    fn test_jwt_secret_too_short() {
        let mut config = make_valid_config();
        config.auth.jwt_secret = "tooshort".to_string();
        assert!(config.validate().is_err());
    }
    
    #[test]
    fn test_feature_flags_defaults() {
        let flags = FeatureFlags::default();
        assert!(flags.enable_registration);
        assert!(!flags.maintenance_mode);
        assert!(flags.enable_rate_limiting);
    }
    
    #[test]
    fn test_dynamic_feature_flags() {
        use std::collections::HashMap;
        
        let mut initial = HashMap::new();
        initial.insert("test_feature".to_string(), false);
        
        let flags = DynamicFeatureFlags::new(initial);
        assert!(!flags.is_enabled("test_feature"));
        
        flags.set_flag("test_feature", true);
        assert!(flags.is_enabled("test_feature"));
        
        // Unknown flag defaults to false
        assert!(!flags.is_enabled("unknown_feature"));
    }
    
    #[test]
    fn test_server_bind_address() {
        let config = make_valid_config();
        assert_eq!(config.server.bind_address(), "0.0.0.0:8080");
    }
    
    #[test]
    fn test_environment_production() {
        let mut config = make_valid_config();
        config.environment = Environment::Production;
        assert!(config.environment.is_production());
        assert!(!config.environment.is_development());
    }
    
    #[test]
    fn test_file_secrets_provider() {
        use std::fs;
        use tempfile::TempDir;
        
        let tmp_dir = TempDir::new().unwrap();
        let secret_path = tmp_dir.path().join("JWT_SECRET");
        fs::write(&secret_path, "my-super-secret-jwt-key-here").unwrap();
        
        let provider = FileSecretsProvider::new(tmp_dir.path());
        assert_eq!(
            provider.get_secret("JWT_SECRET").unwrap(),
            "my-super-secret-jwt-key-here"
        );
        
        assert!(provider.get_secret("NONEXISTENT").is_err());
    }
}
```

## .env Example

```bash
# .env.example
APP_ENV=development

# Server
APP_SERVER__PORT=8080
APP_SERVER__HOST=0.0.0.0

# Database
DATABASE_URL=postgres://postgres:password@localhost/myapp_dev

# Redis
APP_REDIS__URL=redis://localhost:6379

# Auth (use secrets manager in production)
JWT_SECRET=your-very-secret-jwt-key-minimum-32-characters-long

# Logging
APP_LOGGING__LEVEL=debug
APP_LOGGING__FORMAT=pretty
```

## สรุป

Config Management ที่ดีใน Rust:

1. **Type-safe** - Struct ที่ validate ได้
2. **Layered** - default → environment → local → env vars
3. **Secrets ไม่อยู่ใน code** - ใช้ SecretsManager
4. **Feature flags** - toggle features โดยไม่ deploy ใหม่
5. **Validation ที่ startup** - fail-fast ถ้า config ผิด

---

## Navigation

- [← Part 067: Caching Strategies](../part_067/README.md)
- [→ Part 069: Logging and Observability](../part_069/README.md)
- [กลับหน้าหลัก](../../README.md)
