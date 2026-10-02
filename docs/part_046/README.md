# Part 046: API Keys Management 🗝️

## 🎯 เป้าหมายของ Part นี้

- สร้าง API key ที่ปลอดภัย
- เก็บ API key แบบ hashed
- API key middleware
- Key scopes และ permissions
- Key expiration
- Key rotation
- Usage tracking
- Complete CRUD API

---

## 1. API Key Concepts

**Best Practices:**
- ใช้ cryptographically random bytes
- เก็บเฉพาะ hash ใน database (เหมือน password)
- แสดง key ให้ user ครั้งเดียวเมื่อสร้าง
- Format: `prefix_base64encodedRandomBytes`
- ตัวอย่าง: `sk_live_aB3cD4eF5gH6iJ7k` หรือ `ak_7f9a2b3c4d5e6f7g`

---

## 2. Setup

### 2.1 Cargo.toml

```toml
[package]
name = "api-keys"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
rand = "0.8"
base64 = "0.21"
sha2 = "0.10"
hex = "0.4"
thiserror = "1"
log = "0.4"
env_logger = "0.10"
dotenv = "0.15"
futures = "0.3"
serde_with = "3"
```

---

## 3. API Key Generation

```rust
// src/api_key/generator.rs
use rand::{thread_rng, RngCore};
use base64::{Engine, engine::general_purpose::URL_SAFE_NO_PAD};
use sha2::{Sha256, Digest};
use hex;

/// Format: {prefix}_{random_bytes_base64}
/// ตัวอย่าง: ak_live_xK9mN3pQ7rS5tU2v
pub struct GeneratedApiKey {
    /// Plain text key (แสดงให้ user เพียงครั้งเดียว)
    pub key: String,
    /// Key prefix สำหรับ identify โดยไม่ต้อง expose full key
    pub prefix: String,
    /// Hash ที่เก็บใน database
    pub key_hash: String,
}

pub fn generate_api_key(prefix: &str) -> GeneratedApiKey {
    // สร้าง 32 random bytes
    let mut bytes = [0u8; 32];
    thread_rng().fill_bytes(&mut bytes);
    
    // Encode เป็น base64 url-safe
    let random_part = URL_SAFE_NO_PAD.encode(&bytes);
    
    // สร้าง full key
    let key = format!("{}_{}", prefix, random_part);
    
    // สร้าง prefix สำหรับแสดง (8 chars แรก)
    let key_prefix = format!("{}_{}...", prefix, &random_part[..8]);
    
    // Hash key
    let mut hasher = Sha256::new();
    hasher.update(key.as_bytes());
    let key_hash = hex::encode(hasher.finalize());
    
    GeneratedApiKey {
        key,
        prefix: key_prefix,
        key_hash,
    }
}

/// Hash key สำหรับ lookup
pub fn hash_api_key(key: &str) -> String {
    let mut hasher = Sha256::new();
    hasher.update(key.as_bytes());
    hex::encode(hasher.finalize())
}

/// ตรวจสอบ format ของ key
pub fn validate_key_format(key: &str, expected_prefix: &str) -> bool {
    key.starts_with(&format!("{}_", expected_prefix))
        && key.len() >= 40 // prefix + _ + at least 32 chars
}
```

---

## 4. API Key Model และ Scopes

```rust
// src/api_key/model.rs
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use std::collections::HashSet;

/// Scopes ที่ API key สามารถมีได้
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum ApiKeyScope {
    ReadAll,
    WriteAll,
    ReadUsers,
    WriteUsers,
    ReadPosts,
    WritePosts,
    DeletePosts,
    ReadAnalytics,
    ManageWebhooks,
    Admin,
}

impl ApiKeyScope {
    pub fn as_str(&self) -> &str {
        match self {
            Self::ReadAll => "read:all",
            Self::WriteAll => "write:all",
            Self::ReadUsers => "read:users",
            Self::WriteUsers => "write:users",
            Self::ReadPosts => "read:posts",
            Self::WritePosts => "write:posts",
            Self::DeletePosts => "delete:posts",
            Self::ReadAnalytics => "read:analytics",
            Self::ManageWebhooks => "manage:webhooks",
            Self::Admin => "admin",
        }
    }
    
    pub fn from_str(s: &str) -> Option<Self> {
        match s {
            "read:all" => Some(Self::ReadAll),
            "write:all" => Some(Self::WriteAll),
            "read:users" => Some(Self::ReadUsers),
            "write:users" => Some(Self::WriteUsers),
            "read:posts" => Some(Self::ReadPosts),
            "write:posts" => Some(Self::WritePosts),
            "delete:posts" => Some(Self::DeletePosts),
            "read:analytics" => Some(Self::ReadAnalytics),
            "manage:webhooks" => Some(Self::ManageWebhooks),
            "admin" => Some(Self::Admin),
            _ => None,
        }
    }
    
    /// ตรวจสอบว่า scope include action หรือไม่
    pub fn includes(&self, action: &ApiKeyScope) -> bool {
        match self {
            Self::ReadAll => matches!(action,
                Self::ReadUsers | Self::ReadPosts | Self::ReadAnalytics
            ),
            Self::WriteAll => matches!(action,
                Self::WriteUsers | Self::WritePosts | Self::DeletePosts
            ),
            Self::Admin => true,
            _ => self == action,
        }
    }
}

/// API Key stored in database
#[derive(Debug, Serialize, Deserialize, sqlx::FromRow)]
pub struct ApiKey {
    pub id: Uuid,
    pub user_id: Uuid,
    pub name: String,
    pub key_prefix: String,  // แสดงให้ user เห็นว่า key ไหน
    pub key_hash: String,    // SHA-256 hash
    pub scopes: Vec<String>,
    pub expires_at: Option<DateTime<Utc>>,
    pub last_used_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
    pub is_active: bool,
    pub request_count: i64,
}

impl ApiKey {
    pub fn has_scope(&self, required: &ApiKeyScope) -> bool {
        self.scopes.iter().any(|s| {
            ApiKeyScope::from_str(s)
                .map(|scope| scope.includes(required) || &scope == required)
                .unwrap_or(false)
        })
    }
    
    pub fn is_expired(&self) -> bool {
        match self.expires_at {
            Some(exp) => Utc::now() > exp,
            None => false,
        }
    }
    
    pub fn is_valid(&self) -> bool {
        self.is_active && !self.is_expired()
    }
}

/// Request สำหรับสร้าง API key
#[derive(Debug, Deserialize)]
pub struct CreateApiKeyRequest {
    pub name: String,
    pub scopes: Vec<String>,
    pub expires_in_days: Option<i64>,
}

/// Response เมื่อสร้าง API key ใหม่ (แสดง key ครั้งเดียว)
#[derive(Debug, Serialize)]
pub struct CreateApiKeyResponse {
    pub id: String,
    pub key: String,   // Plain text - แสดงครั้งเดียวเท่านั้น!
    pub name: String,
    pub prefix: String,
    pub scopes: Vec<String>,
    pub expires_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}
```

---

## 5. Database Schema

```sql
-- migrations/001_api_keys.sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    key_prefix VARCHAR(50) NOT NULL,
    key_hash VARCHAR(64) NOT NULL UNIQUE,
    scopes TEXT[] NOT NULL DEFAULT '{}',
    expires_at TIMESTAMPTZ,
    last_used_at TIMESTAMPTZ,
    last_used_ip INET,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    is_active BOOLEAN DEFAULT TRUE,
    request_count BIGINT DEFAULT 0
);

CREATE INDEX idx_api_keys_user_id ON api_keys(user_id);
CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
CREATE INDEX idx_api_keys_active ON api_keys(is_active) WHERE is_active = TRUE;

-- Usage log สำหรับ analytics
CREATE TABLE api_key_usage (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_key_id UUID NOT NULL REFERENCES api_keys(id) ON DELETE CASCADE,
    endpoint VARCHAR(255),
    method VARCHAR(10),
    status_code SMALLINT,
    response_time_ms INT,
    ip_address INET,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_usage_key_id ON api_key_usage(api_key_id);
CREATE INDEX idx_usage_created ON api_key_usage(created_at);
```

---

## 6. API Key Service

```rust
// src/api_key/service.rs
use sqlx::PgPool;
use uuid::Uuid;
use chrono::{Utc, Duration};

use super::{
    generator::{generate_api_key, hash_api_key},
    model::{ApiKey, ApiKeyScope, CreateApiKeyRequest, CreateApiKeyResponse},
};

/// สร้าง API key ใหม่
pub async fn create_api_key(
    pool: &PgPool,
    user_id: Uuid,
    req: &CreateApiKeyRequest,
) -> anyhow::Result<CreateApiKeyResponse> {
    // Validate scopes
    let valid_scopes: Vec<String> = req.scopes.iter()
        .filter_map(|s| ApiKeyScope::from_str(s).map(|_| s.clone()))
        .collect();
    
    if valid_scopes.len() != req.scopes.len() {
        return Err(anyhow::anyhow!("Invalid scope provided"));
    }
    
    // Generate key
    let generated = generate_api_key("sk");
    
    let expires_at = req.expires_in_days.map(|days| {
        Utc::now() + Duration::days(days)
    });
    
    // บันทึกลง database
    let key = sqlx::query_as!(
        ApiKey,
        r#"INSERT INTO api_keys (user_id, name, key_prefix, key_hash, scopes, expires_at)
           VALUES ($1, $2, $3, $4, $5, $6)
           RETURNING id, user_id, name, key_prefix, key_hash, scopes, expires_at,
                     last_used_at, created_at, is_active, request_count"#,
        user_id,
        req.name,
        generated.prefix,
        generated.key_hash,
        &valid_scopes,
        expires_at
    )
    .fetch_one(pool)
    .await?;
    
    Ok(CreateApiKeyResponse {
        id: key.id.to_string(),
        key: generated.key, // Plain text key - แสดงครั้งเดียว
        name: key.name,
        prefix: key.key_prefix,
        scopes: key.scopes,
        expires_at: key.expires_at,
        created_at: key.created_at,
    })
}

/// ค้นหา API key จาก hash
pub async fn find_api_key_by_raw(
    pool: &PgPool,
    raw_key: &str,
) -> anyhow::Result<Option<ApiKey>> {
    let key_hash = hash_api_key(raw_key);
    
    let key = sqlx::query_as!(
        ApiKey,
        r#"SELECT id, user_id, name, key_prefix, key_hash, scopes, expires_at,
                  last_used_at, created_at, is_active, request_count
           FROM api_keys
           WHERE key_hash = $1 AND is_active = TRUE"#,
        key_hash
    )
    .fetch_optional(pool)
    .await?;
    
    Ok(key)
}

/// อัพเดท last_used_at และ request_count
pub async fn record_usage(
    pool: &PgPool,
    key_id: Uuid,
    endpoint: &str,
    method: &str,
    status_code: i16,
    response_time_ms: i32,
    ip: Option<std::net::IpAddr>,
) -> anyhow::Result<()> {
    // Update last_used_at และ counter
    sqlx::query!(
        r#"UPDATE api_keys
           SET last_used_at = NOW(),
               last_used_ip = $1,
               request_count = request_count + 1
           WHERE id = $2"#,
        ip.map(|ip| ip.to_string()) as Option<String>,
        key_id
    )
    .execute(pool)
    .await?;
    
    // Log usage
    sqlx::query!(
        r#"INSERT INTO api_key_usage (api_key_id, endpoint, method, status_code, response_time_ms, ip_address)
           VALUES ($1, $2, $3, $4, $5, $6)"#,
        key_id,
        endpoint,
        method,
        status_code,
        response_time_ms,
        ip.map(|ip| ip.to_string()) as Option<String>
    )
    .execute(pool)
    .await?;
    
    Ok(())
}

/// List API keys ของ user
pub async fn list_user_api_keys(
    pool: &PgPool,
    user_id: Uuid,
) -> anyhow::Result<Vec<ApiKey>> {
    let keys = sqlx::query_as!(
        ApiKey,
        r#"SELECT id, user_id, name, key_prefix, key_hash, scopes, expires_at,
                  last_used_at, created_at, is_active, request_count
           FROM api_keys
           WHERE user_id = $1
           ORDER BY created_at DESC"#,
        user_id
    )
    .fetch_all(pool)
    .await?;
    
    Ok(keys)
}

/// Revoke (deactivate) API key
pub async fn revoke_api_key(
    pool: &PgPool,
    key_id: Uuid,
    user_id: Uuid,
) -> anyhow::Result<bool> {
    let result = sqlx::query!(
        r#"UPDATE api_keys
           SET is_active = FALSE, updated_at = NOW()
           WHERE id = $1 AND user_id = $2"#,
        key_id,
        user_id
    )
    .execute(pool)
    .await?;
    
    Ok(result.rows_affected() > 0)
}

/// Rotate API key (สร้างใหม่, revoke เก่า)
pub async fn rotate_api_key(
    pool: &PgPool,
    old_key_id: Uuid,
    user_id: Uuid,
) -> anyhow::Result<CreateApiKeyResponse> {
    // ดึง info ของ key เก่า
    let old_key = sqlx::query!(
        "SELECT name, scopes, expires_at FROM api_keys WHERE id = $1 AND user_id = $2",
        old_key_id,
        user_id
    )
    .fetch_optional(pool)
    .await?
    .ok_or_else(|| anyhow::anyhow!("API key not found"))?;
    
    // สร้าง key ใหม่ด้วย settings เดิม
    let expires_in_days = old_key.expires_at.map(|exp| {
        (exp - Utc::now()).num_days()
    });
    
    let create_req = CreateApiKeyRequest {
        name: format!("{} (rotated)", old_key.name),
        scopes: old_key.scopes,
        expires_in_days,
    };
    
    let new_key = create_api_key(pool, user_id, &create_req).await?;
    
    // Revoke key เก่า
    revoke_api_key(pool, old_key_id, user_id).await?;
    
    Ok(new_key)
}
```

---

## 7. API Key Middleware

```rust
// src/middleware/api_key.rs
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpMessage, HttpResponse,
};
use futures::future::{ready, Ready, LocalBoxFuture};
use sqlx::PgPool;
use std::rc::Rc;
use std::sync::Arc;
use std::time::Instant;

use crate::api_key::{model::{ApiKey, ApiKeyScope}, service};

/// Context ที่ inject เข้า request
#[derive(Clone, Debug)]
pub struct ApiKeyContext {
    pub key_id: uuid::Uuid,
    pub user_id: uuid::Uuid,
    pub scopes: Vec<String>,
}

pub struct ApiKeyAuth {
    pool: Arc<PgPool>,
    required_scope: Option<ApiKeyScope>,
}

impl ApiKeyAuth {
    pub fn new(pool: Arc<PgPool>) -> Self {
        Self { pool, required_scope: None }
    }
    
    pub fn with_scope(pool: Arc<PgPool>, scope: ApiKeyScope) -> Self {
        Self { pool, required_scope: Some(scope) }
    }
}

impl<S, B> Transform<S, ServiceRequest> for ApiKeyAuth
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Transform = ApiKeyAuthMiddleware<S>;
    type InitError = ();
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(ApiKeyAuthMiddleware {
            service: Rc::new(service),
            pool: self.pool.clone(),
            required_scope: self.required_scope.clone(),
        }))
    }
}

pub struct ApiKeyAuthMiddleware<S> {
    service: Rc<S>,
    pool: Arc<PgPool>,
    required_scope: Option<ApiKeyScope>,
}

impl<S, B> Service<ServiceRequest> for ApiKeyAuthMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;
    
    forward_ready!(service);
    
    fn call(&self, req: ServiceRequest) -> Self::Future {
        let service = self.service.clone();
        let pool = self.pool.clone();
        let required_scope = self.required_scope.clone();
        
        Box::pin(async move {
            let start = Instant::now();
            
            // ดึง API key จาก header
            let raw_key = req
                .headers()
                .get("X-API-Key")
                .and_then(|h| h.to_str().ok())
                .map(|s| s.to_string());
            
            // ลองดึงจาก Authorization header ด้วย
            let raw_key = raw_key.or_else(|| {
                req.headers()
                    .get("Authorization")
                    .and_then(|h| h.to_str().ok())
                    .and_then(|h| h.strip_prefix("Bearer "))
                    .map(|s| s.to_string())
            });
            
            let raw_key = match raw_key {
                Some(k) => k,
                None => {
                    return Ok(req.into_response(
                        HttpResponse::Unauthorized()
                            .json(serde_json::json!({"error": "API key required"}))
                            .map_into_boxed_body()
                    ));
                }
            };
            
            // ค้นหา key ใน database
            let api_key = service::find_api_key_by_raw(&pool, &raw_key)
                .await
                .map_err(|e| {
                    log::error!("Database error: {}", e);
                    actix_web::error::ErrorInternalServerError("Database error")
                })?;
            
            let api_key = match api_key {
                Some(k) => k,
                None => {
                    return Ok(req.into_response(
                        HttpResponse::Unauthorized()
                            .json(serde_json::json!({"error": "Invalid API key"}))
                            .map_into_boxed_body()
                    ));
                }
            };
            
            // ตรวจสอบว่า key ยัง valid
            if !api_key.is_valid() {
                return Ok(req.into_response(
                    HttpResponse::Unauthorized()
                        .json(serde_json::json!({"error": "API key is expired or revoked"}))
                        .map_into_boxed_body()
                ));
            }
            
            // ตรวจสอบ scope
            if let Some(scope) = &required_scope {
                if !api_key.has_scope(scope) {
                    return Ok(req.into_response(
                        HttpResponse::Forbidden()
                            .json(serde_json::json!({
                                "error": "Insufficient scope",
                                "required_scope": scope.as_str()
                            }))
                            .map_into_boxed_body()
                    ));
                }
            }
            
            let key_id = api_key.id;
            let user_id = api_key.user_id;
            let scopes = api_key.scopes.clone();
            
            // Inject context
            req.extensions_mut().insert(ApiKeyContext {
                key_id,
                user_id,
                scopes,
            });
            
            let endpoint = req.path().to_string();
            let method = req.method().to_string();
            let client_ip = req.peer_addr().map(|a| a.ip());
            
            let mut response = service.call(req).await?;
            
            let elapsed = start.elapsed().as_millis() as i32;
            let status = response.status().as_u16() as i16;
            
            // Record usage async
            let pool_clone = pool.clone();
            tokio::spawn(async move {
                if let Err(e) = service::record_usage(
                    &pool_clone, key_id, &endpoint, &method, status, elapsed, client_ip
                ).await {
                    log::error!("Failed to record API key usage: {}", e);
                }
            });
            
            Ok(response)
        })
    }
}
```

---

## 8. Handlers

```rust
// src/handlers/api_keys.rs
use actix_web::{web, HttpRequest, HttpResponse};
use sqlx::PgPool;
use uuid::Uuid;

use crate::api_key::{
    model::CreateApiKeyRequest,
    service,
};
use crate::middleware::api_key::ApiKeyContext;

/// สร้าง API key ใหม่
pub async fn create_key(
    pool: web::Data<PgPool>,
    req: web::Json<CreateApiKeyRequest>,
    // ต้องผ่าน JWT auth ก่อน
    user_id: web::ReqData<Uuid>,
) -> actix_web::Result<HttpResponse> {
    let response = service::create_api_key(
        pool.get_ref(),
        *user_id,
        &req,
    )
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    Ok(HttpResponse::Created().json(response))
}

/// List API keys
pub async fn list_keys(
    pool: web::Data<PgPool>,
    user_id: web::ReqData<Uuid>,
) -> actix_web::Result<HttpResponse> {
    let keys = service::list_user_api_keys(pool.get_ref(), *user_id)
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    // ไม่ return key_hash ให้ user
    let safe_keys: Vec<serde_json::Value> = keys.iter().map(|k| serde_json::json!({
        "id": k.id,
        "name": k.name,
        "prefix": k.key_prefix,
        "scopes": k.scopes,
        "expires_at": k.expires_at,
        "last_used_at": k.last_used_at,
        "created_at": k.created_at,
        "is_active": k.is_active,
        "request_count": k.request_count,
    })).collect();
    
    Ok(HttpResponse::Ok().json(safe_keys))
}

/// Revoke API key
pub async fn revoke_key(
    pool: web::Data<PgPool>,
    path: web::Path<Uuid>,
    user_id: web::ReqData<Uuid>,
) -> actix_web::Result<HttpResponse> {
    let key_id = path.into_inner();
    
    let revoked = service::revoke_api_key(pool.get_ref(), key_id, *user_id)
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    if revoked {
        Ok(HttpResponse::Ok().json(serde_json::json!({
            "message": "API key revoked successfully"
        })))
    } else {
        Ok(HttpResponse::NotFound().json(serde_json::json!({
            "error": "API key not found"
        })))
    }
}

/// Rotate API key
pub async fn rotate_key(
    pool: web::Data<PgPool>,
    path: web::Path<Uuid>,
    user_id: web::ReqData<Uuid>,
) -> actix_web::Result<HttpResponse> {
    let key_id = path.into_inner();
    
    let new_key = service::rotate_api_key(pool.get_ref(), key_id, *user_id)
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    Ok(HttpResponse::Created().json(new_key))
}

/// Protected endpoint ที่ใช้ API key
pub async fn api_endpoint(
    http_req: HttpRequest,
) -> actix_web::Result<HttpResponse> {
    let ctx = http_req.extensions().get::<ApiKeyContext>().cloned();
    
    match ctx {
        Some(ctx) => Ok(HttpResponse::Ok().json(serde_json::json!({
            "data": "Protected API data",
            "authenticated_as": ctx.user_id,
            "scopes": ctx.scopes,
        }))),
        None => Ok(HttpResponse::Unauthorized().finish()),
    }
}
```

---

## 9. Main Application

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware};
use sqlx::postgres::PgPoolOptions;
use std::sync::Arc;
use dotenv::dotenv;

mod api_key {
    pub mod generator;
    pub mod model;
    pub mod service;
}
mod middleware {
    pub mod api_key;
}
mod handlers {
    pub mod api_keys;
}

use middleware::api_key::{ApiKeyAuth, ApiKeyScope};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv().ok();
    env_logger::init();
    
    let database_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    
    let pool = Arc::new(
        PgPoolOptions::new()
            .max_connections(5)
            .connect(&database_url)
            .await
            .expect("Failed to connect to database")
    );
    
    sqlx::migrate!("./migrations")
        .run(pool.as_ref())
        .await
        .expect("Failed to run migrations");
    
    let pool_clone = pool.clone();
    
    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::from(pool.clone()))
            .wrap(middleware::Logger::default())
            // API key management routes (JWT auth required)
            .service(
                web::scope("/api/keys")
                    .route("", web::get().to(handlers::api_keys::list_keys))
                    .route("", web::post().to(handlers::api_keys::create_key))
                    .route("/{id}", web::delete().to(handlers::api_keys::revoke_key))
                    .route("/{id}/rotate", web::post().to(handlers::api_keys::rotate_key))
            )
            // Protected API routes (API key auth)
            .service(
                web::scope("/api/v1")
                    .wrap(ApiKeyAuth::new(pool.clone()))
                    .route("/data", web::get().to(handlers::api_keys::api_endpoint))
            )
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

---

## 10. Testing

```rust
#[cfg(test)]
mod tests {
    use super::api_key::{generator::*, model::*};

    #[test]
    fn test_generate_api_key() {
        let key1 = generate_api_key("sk");
        let key2 = generate_api_key("sk");
        
        // Keys ต้องไม่ซ้ำกัน
        assert_ne!(key1.key, key2.key);
        
        // Key ต้องขึ้นต้นด้วย prefix
        assert!(key1.key.starts_with("sk_"));
        
        // Hash ต้องไม่เท่ากับ key
        assert_ne!(key1.key, key1.key_hash);
    }
    
    #[test]
    fn test_hash_consistency() {
        let key = generate_api_key("ak");
        
        // Hash key เดิมสองครั้ง ต้องได้ผลเหมือนกัน
        let hash1 = hash_api_key(&key.key);
        let hash2 = hash_api_key(&key.key);
        
        assert_eq!(hash1, hash2);
        assert_eq!(hash1, key.key_hash);
    }
    
    #[test]
    fn test_scope_includes() {
        assert!(ApiKeyScope::ReadAll.includes(&ApiKeyScope::ReadUsers));
        assert!(ApiKeyScope::ReadAll.includes(&ApiKeyScope::ReadPosts));
        assert!(!ApiKeyScope::ReadAll.includes(&ApiKeyScope::WriteUsers));
        assert!(ApiKeyScope::Admin.includes(&ApiKeyScope::ReadAll));
    }
    
    #[test]
    fn test_key_format_validation() {
        assert!(validate_key_format("sk_abcdefghijklmnopqrstuvwxyz12345678", "sk"));
        assert!(!validate_key_format("invalid_key", "sk"));
        assert!(!validate_key_format("sk_short", "sk"));
    }
}
```

---

## 11. สรุปสิ่งที่เรียนรู้

✅ สร้าง cryptographically secure API keys  
✅ เก็บเฉพาะ SHA-256 hash ใน database  
✅ แสดง plain key ให้ user ครั้งเดียวเท่านั้น  
✅ Scopes และ permissions system  
✅ Key expiration  
✅ Key rotation (สร้างใหม่, revoke เก่า)  
✅ Usage tracking และ analytics  
✅ API key middleware  
✅ Async usage logging  

---

*[← Part 045: Rate Limiting](../part_045/README.md) | [Part 047: TLS and HTTPS →](../part_047/README.md)*
