# Part 066: API Versioning Strategies in Rust

## ภาพรวม

API Versioning เป็นสิ่งสำคัญเมื่อ API ของเราต้องพัฒนาไปเรื่อย ๆ โดยไม่ทำให้ clients เก่าพัง บทนี้ครอบคลุมกลยุทธ์ต่าง ๆ สำหรับ API versioning ใน Actix-web

## กลยุทธ์ Versioning

```
1. URL Path:     /api/v1/users     /api/v2/users
2. Header:       Accept: application/vnd.api+json;version=1
3. Query Param:  /api/users?version=2
4. Subdomain:    v1.api.example.com
```

## Cargo.toml

```toml
[package]
name = "api-versioning"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
```

## Strategy 1: URL Path Versioning

```rust
// src/versioning/url_path.rs

// V1 Models (เก่า)
pub mod v1 {
    use serde::{Deserialize, Serialize};
    use uuid::Uuid;
    
    #[derive(Debug, Serialize, Deserialize)]
    pub struct UserResponse {
        pub id: Uuid,
        pub name: String,
        pub email: String,
    }
    
    #[derive(Debug, Deserialize)]
    pub struct CreateUserRequest {
        pub name: String,
        pub email: String,
        pub password: String,
    }
}

// V2 Models (ใหม่ - เพิ่ม fields)
pub mod v2 {
    use serde::{Deserialize, Serialize};
    use uuid::Uuid;
    use chrono::{DateTime, Utc};
    
    #[derive(Debug, Serialize, Deserialize)]
    pub struct UserResponse {
        pub id: Uuid,
        pub first_name: String,  // เปลี่ยนจาก name
        pub last_name: String,   // เพิ่มใหม่
        pub email: String,
        pub phone: Option<String>, // เพิ่มใหม่
        pub created_at: DateTime<Utc>,
        pub is_verified: bool,
    }
    
    #[derive(Debug, Deserialize)]
    pub struct CreateUserRequest {
        pub first_name: String,
        pub last_name: String,
        pub email: String,
        pub password: String,
        pub phone: Option<String>,
    }
}

// Handlers V1
pub mod handlers_v1 {
    use actix_web::{web, HttpResponse};
    use uuid::Uuid;
    use super::v1::*;
    
    pub async fn get_user(path: web::Path<Uuid>) -> HttpResponse {
        // V1 response format
        HttpResponse::Ok().json(UserResponse {
            id: *path,
            name: "John Doe".to_string(),
            email: "john@example.com".to_string(),
        })
    }
    
    pub async fn create_user(body: web::Json<CreateUserRequest>) -> HttpResponse {
        HttpResponse::Created().json(UserResponse {
            id: Uuid::new_v4(),
            name: body.name.clone(),
            email: body.email.clone(),
        })
    }
    
    pub async fn list_users() -> HttpResponse {
        let users = vec![
            UserResponse {
                id: Uuid::new_v4(),
                name: "User 1".to_string(),
                email: "user1@example.com".to_string(),
            },
        ];
        HttpResponse::Ok().json(users)
    }
}

// Handlers V2
pub mod handlers_v2 {
    use actix_web::{web, HttpResponse};
    use uuid::Uuid;
    use chrono::Utc;
    use super::v2::*;
    
    pub async fn get_user(path: web::Path<Uuid>) -> HttpResponse {
        // V2 response format - ข้อมูลมากขึ้น
        HttpResponse::Ok().json(UserResponse {
            id: *path,
            first_name: "John".to_string(),
            last_name: "Doe".to_string(),
            email: "john@example.com".to_string(),
            phone: Some("+66891234567".to_string()),
            created_at: Utc::now(),
            is_verified: true,
        })
    }
    
    pub async fn create_user(body: web::Json<CreateUserRequest>) -> HttpResponse {
        HttpResponse::Created().json(UserResponse {
            id: Uuid::new_v4(),
            first_name: body.first_name.clone(),
            last_name: body.last_name.clone(),
            email: body.email.clone(),
            phone: body.phone.clone(),
            created_at: Utc::now(),
            is_verified: false,
        })
    }
    
    // V2 เพิ่ม endpoint ใหม่
    pub async fn verify_user(path: web::Path<Uuid>) -> HttpResponse {
        HttpResponse::Ok().json(serde_json::json!({
            "message": "User verified successfully"
        }))
    }
}
```

## การ Register Routes

```rust
// src/main.rs (URL Path Versioning)
use actix_web::{web, App, HttpServer};

mod versioning;
use versioning::url_path::{handlers_v1, handlers_v2};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            // V1 routes
            .service(
                web::scope("/api/v1")
                    .service(
                        web::scope("/users")
                            .route("", web::get().to(handlers_v1::list_users))
                            .route("", web::post().to(handlers_v1::create_user))
                            .route("/{id}", web::get().to(handlers_v1::get_user))
                    )
            )
            // V2 routes - เพิ่ม endpoints ใหม่
            .service(
                web::scope("/api/v2")
                    .service(
                        web::scope("/users")
                            .route("", web::post().to(handlers_v2::create_user))
                            .route("/{id}", web::get().to(handlers_v2::get_user))
                            .route("/{id}/verify", web::post().to(handlers_v2::verify_user))
                    )
            )
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

## Strategy 2: Header Versioning

```rust
// src/versioning/header_based.rs
use actix_web::{web, HttpRequest, HttpResponse, middleware};
use std::str::FromStr;

#[derive(Debug, Clone, PartialEq)]
pub enum ApiVersion {
    V1,
    V2,
    V3,
}

impl FromStr for ApiVersion {
    type Err = String;
    
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        match s.trim() {
            "1" | "v1" | "1.0" => Ok(ApiVersion::V1),
            "2" | "v2" | "2.0" => Ok(ApiVersion::V2),
            "3" | "v3" | "3.0" => Ok(ApiVersion::V3),
            _ => Err(format!("Unknown version: {}", s)),
        }
    }
}

impl Default for ApiVersion {
    fn default() -> Self {
        ApiVersion::V1
    }
}

/// Extract version จาก headers
pub fn extract_version(req: &HttpRequest) -> ApiVersion {
    // ลอง header "API-Version" ก่อน
    if let Some(version) = req.headers().get("API-Version") {
        if let Ok(v) = version.to_str() {
            if let Ok(parsed) = ApiVersion::from_str(v) {
                return parsed;
            }
        }
    }
    
    // ลอง Accept header (Content negotiation)
    // Accept: application/vnd.myapp+json;version=2
    if let Some(accept) = req.headers().get("Accept") {
        if let Ok(v) = accept.to_str() {
            if let Some(version_part) = v.split(';').find(|p| p.trim().starts_with("version=")) {
                let version_str = version_part.trim().trim_start_matches("version=");
                if let Ok(parsed) = ApiVersion::from_str(version_str) {
                    return parsed;
                }
            }
        }
    }
    
    ApiVersion::default()
}

/// Handler ที่ dispatch ตาม version
pub async fn get_user_versioned(
    req: HttpRequest,
    path: web::Path<uuid::Uuid>,
) -> HttpResponse {
    let version = extract_version(&req);
    
    match version {
        ApiVersion::V1 => {
            HttpResponse::Ok()
                .append_header(("API-Version", "1"))
                .json(serde_json::json!({
                    "id": *path,
                    "name": "John Doe",
                    "email": "john@example.com"
                }))
        }
        ApiVersion::V2 => {
            HttpResponse::Ok()
                .append_header(("API-Version", "2"))
                .json(serde_json::json!({
                    "id": *path,
                    "first_name": "John",
                    "last_name": "Doe",
                    "email": "john@example.com",
                    "phone": "+66891234567",
                    "created_at": chrono::Utc::now()
                }))
        }
        ApiVersion::V3 => {
            HttpResponse::Ok()
                .append_header(("API-Version", "3"))
                .json(serde_json::json!({
                    "id": *path,
                    "profile": {
                        "first_name": "John",
                        "last_name": "Doe",
                        "display_name": "John Doe"
                    },
                    "contact": {
                        "email": "john@example.com",
                        "phone": "+66891234567"
                    },
                    "meta": {
                        "created_at": chrono::Utc::now(),
                        "is_verified": true,
                        "version": 3
                    }
                }))
        }
    }
}
```

## Strategy 3: Query Parameter Versioning

```rust
// src/versioning/query_param.rs
use actix_web::{web, HttpResponse};
use serde::Deserialize;
use uuid::Uuid;

#[derive(Deserialize)]
pub struct VersionQuery {
    pub version: Option<u32>,
}

pub async fn get_user(
    path: web::Path<Uuid>,
    query: web::Query<VersionQuery>,
) -> HttpResponse {
    let version = query.version.unwrap_or(1);
    
    match version {
        1 => HttpResponse::Ok().json(serde_json::json!({
            "id": *path,
            "name": "John Doe",
            "email": "john@example.com"
        })),
        2 => HttpResponse::Ok().json(serde_json::json!({
            "id": *path,
            "first_name": "John",
            "last_name": "Doe",
            "email": "john@example.com",
            "phone": "+66891234567"
        })),
        _ => HttpResponse::BadRequest().json(serde_json::json!({
            "error": format!("Unsupported version: {}", version),
            "supported_versions": [1, 2]
        })),
    }
}
```

## Backward Compatibility

```rust
// src/compatibility/mod.rs
use serde::{Deserialize, Serialize};
use uuid::Uuid;

/// V1 User (เก่า)
#[derive(Debug, Serialize, Deserialize)]
pub struct UserV1 {
    pub id: Uuid,
    pub name: String,
    pub email: String,
}

/// V2 User (ใหม่)
#[derive(Debug, Serialize, Deserialize)]
pub struct UserV2 {
    pub id: Uuid,
    pub first_name: String,
    pub last_name: String,
    pub email: String,
    pub phone: Option<String>,
}

/// Domain User (internal)
#[derive(Debug, Clone)]
pub struct UserDomain {
    pub id: Uuid,
    pub first_name: String,
    pub last_name: String,
    pub email: String,
    pub phone: Option<String>,
}

impl UserDomain {
    /// Convert เป็น V1 (backward compatible)
    pub fn to_v1(&self) -> UserV1 {
        UserV1 {
            id: self.id,
            name: format!("{} {}", self.first_name, self.last_name),
            email: self.email.clone(),
        }
    }
    
    /// Convert เป็น V2
    pub fn to_v2(&self) -> UserV2 {
        UserV2 {
            id: self.id,
            first_name: self.first_name.clone(),
            last_name: self.last_name.clone(),
            email: self.email.clone(),
            phone: self.phone.clone(),
        }
    }
    
    /// Parse จาก V1 request
    pub fn from_v1_request(req: CreateUserV1Request) -> Self {
        // แยก name เป็น first/last (best effort)
        let parts: Vec<&str> = req.name.splitn(2, ' ').collect();
        let (first, last) = if parts.len() == 2 {
            (parts[0].to_string(), parts[1].to_string())
        } else {
            (req.name.clone(), String::new())
        };
        
        UserDomain {
            id: Uuid::new_v4(),
            first_name: first,
            last_name: last,
            email: req.email,
            phone: None,
        }
    }
    
    /// Parse จาก V2 request
    pub fn from_v2_request(req: CreateUserV2Request) -> Self {
        UserDomain {
            id: Uuid::new_v4(),
            first_name: req.first_name,
            last_name: req.last_name,
            email: req.email,
            phone: req.phone,
        }
    }
}

#[derive(Deserialize)]
pub struct CreateUserV1Request {
    pub name: String,
    pub email: String,
    pub password: String,
}

#[derive(Deserialize)]
pub struct CreateUserV2Request {
    pub first_name: String,
    pub last_name: String,
    pub email: String,
    pub password: String,
    pub phone: Option<String>,
}
```

## Version Deprecation

```rust
// src/deprecation/mod.rs
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpMessage,
};
use chrono::{DateTime, Utc};
use futures_util::future::LocalBoxFuture;
use std::collections::HashMap;

#[derive(Debug, Clone)]
pub struct VersionDeprecationInfo {
    pub deprecated_date: DateTime<Utc>,
    pub sunset_date: DateTime<Utc>,
    pub migration_guide: String,
}

pub struct DeprecationMiddleware {
    deprecated_versions: HashMap<String, VersionDeprecationInfo>,
}

impl DeprecationMiddleware {
    pub fn new() -> Self {
        let mut deprecated = HashMap::new();
        
        // V1 deprecated
        deprecated.insert("v1".to_string(), VersionDeprecationInfo {
            deprecated_date: chrono::DateTime::parse_from_rfc3339("2024-01-01T00:00:00Z")
                .unwrap().with_timezone(&Utc),
            sunset_date: chrono::DateTime::parse_from_rfc3339("2025-01-01T00:00:00Z")
                .unwrap().with_timezone(&Utc),
            migration_guide: "https://docs.example.com/migration/v1-to-v2".to_string(),
        });
        
        DeprecationMiddleware { deprecated_versions: deprecated }
    }
}

impl<S, B> Transform<S, ServiceRequest> for DeprecationMiddleware
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = DeprecationMiddlewareService<S>;
    type Future = std::future::Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        std::future::ready(Ok(DeprecationMiddlewareService {
            service,
            deprecated_versions: self.deprecated_versions.clone(),
        }))
    }
}

pub struct DeprecationMiddlewareService<S> {
    service: S,
    deprecated_versions: HashMap<String, VersionDeprecationInfo>,
}

impl<S, B> Service<ServiceRequest> for DeprecationMiddlewareService<S>
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
        let deprecated_versions = self.deprecated_versions.clone();
        
        // ตรวจสอบว่า path มี version ที่ deprecated หรือไม่
        let deprecation_info = deprecated_versions.iter().find_map(|(version, info)| {
            if path.contains(&format!("/{}/", version)) || path.ends_with(&format!("/{}", version)) {
                Some((version.clone(), info.clone()))
            } else {
                None
            }
        });
        
        let fut = self.service.call(req);
        
        Box::pin(async move {
            let mut res = fut.await?;
            
            if let Some((version, info)) = deprecation_info {
                // เพิ่ม Deprecation headers
                let headers = res.headers_mut();
                headers.insert(
                    actix_web::http::header::HeaderName::from_static("deprecation"),
                    actix_web::http::header::HeaderValue::from_str(
                        &info.deprecated_date.to_rfc3339()
                    ).unwrap(),
                );
                headers.insert(
                    actix_web::http::header::HeaderName::from_static("sunset"),
                    actix_web::http::header::HeaderValue::from_str(
                        &info.sunset_date.to_rfc3339()
                    ).unwrap(),
                );
                headers.insert(
                    actix_web::http::header::HeaderName::from_static("link"),
                    actix_web::http::header::HeaderValue::from_str(
                        &format!("<{}>; rel=\"deprecation\"", info.migration_guide)
                    ).unwrap(),
                );
            }
            
            Ok(res)
        })
    }
}
```

## Version Router (Generic)

```rust
// src/routing/version_router.rs
use actix_web::{web, HttpRequest, HttpResponse};
use std::collections::HashMap;
use std::sync::Arc;
use async_trait::async_trait;

pub type HandlerFn = Arc<dyn Fn(HttpRequest, web::Bytes) -> futures_util::future::BoxFuture<'static, HttpResponse> + Send + Sync>;

pub struct VersionRouter {
    handlers: HashMap<String, HandlerFn>,
    default_version: String,
}

impl VersionRouter {
    pub fn new(default_version: impl Into<String>) -> Self {
        VersionRouter {
            handlers: HashMap::new(),
            default_version: default_version.into(),
        }
    }
    
    pub fn register_version(
        &mut self,
        version: impl Into<String>,
        handler: HandlerFn,
    ) {
        self.handlers.insert(version.into(), handler);
    }
    
    pub async fn dispatch(
        &self,
        req: HttpRequest,
        body: web::Bytes,
    ) -> HttpResponse {
        // ดึง version จาก request
        let version = self.extract_version(&req);
        
        match self.handlers.get(&version) {
            Some(handler) => handler(req, body).await,
            None => {
                // fallback ไป default version
                match self.handlers.get(&self.default_version) {
                    Some(handler) => handler(req, body).await,
                    None => HttpResponse::InternalServerError().json(serde_json::json!({
                        "error": "No handler found"
                    })),
                }
            }
        }
    }
    
    fn extract_version(&self, req: &HttpRequest) -> String {
        // 1. ลอง URL path
        for segment in req.path().split('/') {
            if segment.starts_with('v') && segment[1..].parse::<u32>().is_ok() {
                return segment.to_string();
            }
        }
        
        // 2. ลอง header
        if let Some(v) = req.headers().get("API-Version") {
            if let Ok(s) = v.to_str() {
                return format!("v{}", s.trim_start_matches('v'));
            }
        }
        
        // 3. ลอง query param
        if let Some(v) = actix_web::web::Query::<HashMap<String, String>>::from_query(req.query_string())
            .ok()
            .and_then(|q| q.get("version").cloned())
        {
            return format!("v{}", v.trim_start_matches('v'));
        }
        
        self.default_version.clone()
    }
}
```

## Documentation Per Version

```rust
// src/docs/openapi.rs
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use actix_web::{web, HttpResponse};

#[derive(Serialize)]
pub struct OpenApiSpec {
    pub openapi: String,
    pub info: ApiInfo,
    pub paths: HashMap<String, serde_json::Value>,
}

#[derive(Serialize)]
pub struct ApiInfo {
    pub title: String,
    pub version: String,
    pub description: String,
}

/// สร้าง OpenAPI spec สำหรับแต่ละ version
pub fn get_v1_spec() -> OpenApiSpec {
    let mut paths = HashMap::new();
    
    paths.insert("/api/v1/users".to_string(), serde_json::json!({
        "get": {
            "summary": "List users",
            "responses": {
                "200": {
                    "description": "Success",
                    "content": {
                        "application/json": {
                            "schema": {
                                "type": "array",
                                "items": {
                                    "type": "object",
                                    "properties": {
                                        "id": { "type": "string", "format": "uuid" },
                                        "name": { "type": "string" },
                                        "email": { "type": "string" }
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }));
    
    OpenApiSpec {
        openapi: "3.0.0".to_string(),
        info: ApiInfo {
            title: "My API".to_string(),
            version: "1.0.0".to_string(),
            description: "V1 API - DEPRECATED. Use V2.".to_string(),
        },
        paths,
    }
}

pub fn get_v2_spec() -> OpenApiSpec {
    let mut paths = HashMap::new();
    
    paths.insert("/api/v2/users".to_string(), serde_json::json!({
        "post": {
            "summary": "Create user",
            "requestBody": {
                "required": true,
                "content": {
                    "application/json": {
                        "schema": {
                            "type": "object",
                            "required": ["first_name", "last_name", "email", "password"],
                            "properties": {
                                "first_name": { "type": "string" },
                                "last_name": { "type": "string" },
                                "email": { "type": "string", "format": "email" },
                                "password": { "type": "string" },
                                "phone": { "type": "string" }
                            }
                        }
                    }
                }
            }
        }
    }));
    
    OpenApiSpec {
        openapi: "3.0.0".to_string(),
        info: ApiInfo {
            title: "My API".to_string(),
            version: "2.0.0".to_string(),
            description: "V2 API - Current stable version.".to_string(),
        },
        paths,
    }
}

pub async fn serve_api_docs(path: web::Path<String>) -> HttpResponse {
    match path.as_str() {
        "v1" => HttpResponse::Ok().json(get_v1_spec()),
        "v2" => HttpResponse::Ok().json(get_v2_spec()),
        _ => HttpResponse::NotFound().json(serde_json::json!({
            "error": "Version not found",
            "available_versions": ["v1", "v2"]
        })),
    }
}
```

## Migration Guide Endpoint

```rust
// src/migration/mod.rs
use actix_web::{web, HttpResponse};

pub async fn get_migration_guide(
    path: web::Path<(String, String)>,
) -> HttpResponse {
    let (from_version, to_version) = path.into_inner();
    
    match (from_version.as_str(), to_version.as_str()) {
        ("v1", "v2") => HttpResponse::Ok().json(serde_json::json!({
            "from": "v1",
            "to": "v2",
            "breaking_changes": [
                {
                    "change": "User name field split into first_name and last_name",
                    "before": { "name": "John Doe" },
                    "after": { "first_name": "John", "last_name": "Doe" }
                },
                {
                    "change": "Create user endpoint now requires first_name and last_name separately",
                    "impact": "HIGH"
                }
            ],
            "new_features": [
                "Phone number field added",
                "User verification endpoint added (POST /users/{id}/verify)",
                "created_at field now included in response"
            ],
            "deprecated_endpoints": [],
            "removed_endpoints": [],
            "migration_steps": [
                "1. Update CreateUserRequest: replace 'name' with 'first_name' + 'last_name'",
                "2. Update UserResponse parsing: combine first_name and last_name where needed",
                "3. Update API base URL from /api/v1 to /api/v2"
            ]
        })),
        _ => HttpResponse::NotFound().json(serde_json::json!({
            "error": format!("No migration guide from {} to {}", from_version, to_version)
        })),
    }
}
```

## Complete App Setup

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware as actix_middleware};
use tracing_subscriber::EnvFilter;

mod versioning;
mod compatibility;
mod deprecation;
mod docs;
mod migration;

use deprecation::DeprecationMiddleware;
use versioning::url_path::{handlers_v1, handlers_v2};
use versioning::header_based::get_user_versioned;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::from_default_env())
        .init();
    
    HttpServer::new(|| {
        App::new()
            .wrap(actix_middleware::Logger::default())
            .wrap(DeprecationMiddleware::new())
            
            // ============ URL Path Versioning ============
            // V1 API (Deprecated)
            .service(
                web::scope("/api/v1")
                    .service(
                        web::scope("/users")
                            .route("", web::get().to(handlers_v1::list_users))
                            .route("", web::post().to(handlers_v1::create_user))
                            .route("/{id}", web::get().to(handlers_v1::get_user))
                    )
            )
            // V2 API (Current)
            .service(
                web::scope("/api/v2")
                    .service(
                        web::scope("/users")
                            .route("", web::post().to(handlers_v2::create_user))
                            .route("/{id}", web::get().to(handlers_v2::get_user))
                            .route("/{id}/verify", web::post().to(handlers_v2::verify_user))
                    )
            )
            
            // ============ Header-based Versioning ============
            .service(
                web::scope("/api/users")
                    .route("/{id}", web::get().to(get_user_versioned))
            )
            
            // ============ Documentation ============
            .route("/api/docs/{version}", web::get().to(docs::openapi::serve_api_docs))
            
            // ============ Migration Guide ============
            .route(
                "/api/migration/{from}/{to}",
                web::get().to(migration::get_migration_guide)
            )
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

## Tests

```rust
#[cfg(test)]
mod tests {
    use actix_web::test;
    use super::*;
    
    #[actix_web::test]
    async fn test_v1_get_user() {
        let app = test::init_service(
            App::new().service(
                web::scope("/api/v1/users")
                    .route("/{id}", web::get().to(handlers_v1::get_user))
            )
        ).await;
        
        let user_id = uuid::Uuid::new_v4();
        let req = test::TestRequest::get()
            .uri(&format!("/api/v1/users/{}", user_id))
            .to_request();
        
        let resp = test::call_service(&app, req).await;
        assert_eq!(resp.status(), actix_web::http::StatusCode::OK);
        
        let body: serde_json::Value = test::read_body_json(resp).await;
        assert!(body["name"].is_string());
        assert!(!body.as_object().unwrap().contains_key("first_name"));
    }
    
    #[actix_web::test]
    async fn test_v2_get_user() {
        let app = test::init_service(
            App::new().service(
                web::scope("/api/v2/users")
                    .route("/{id}", web::get().to(handlers_v2::get_user))
            )
        ).await;
        
        let user_id = uuid::Uuid::new_v4();
        let req = test::TestRequest::get()
            .uri(&format!("/api/v2/users/{}", user_id))
            .to_request();
        
        let resp = test::call_service(&app, req).await;
        assert_eq!(resp.status(), actix_web::http::StatusCode::OK);
        
        let body: serde_json::Value = test::read_body_json(resp).await;
        assert!(body["first_name"].is_string());
        assert!(body["last_name"].is_string());
        assert!(!body.as_object().unwrap().contains_key("name"));
    }
    
    #[actix_web::test]
    async fn test_header_versioning_v1() {
        let app = test::init_service(
            App::new().service(
                web::scope("/api/users")
                    .route("/{id}", web::get().to(get_user_versioned))
            )
        ).await;
        
        let user_id = uuid::Uuid::new_v4();
        let req = test::TestRequest::get()
            .uri(&format!("/api/users/{}", user_id))
            .insert_header(("API-Version", "1"))
            .to_request();
        
        let resp = test::call_service(&app, req).await;
        assert_eq!(resp.status(), actix_web::http::StatusCode::OK);
        
        let body: serde_json::Value = test::read_body_json(resp).await;
        assert!(body["name"].is_string());
    }
    
    #[actix_web::test]
    async fn test_header_versioning_v2() {
        let app = test::init_service(
            App::new().service(
                web::scope("/api/users")
                    .route("/{id}", web::get().to(get_user_versioned))
            )
        ).await;
        
        let user_id = uuid::Uuid::new_v4();
        let req = test::TestRequest::get()
            .uri(&format!("/api/users/{}", user_id))
            .insert_header(("API-Version", "2"))
            .to_request();
        
        let resp = test::call_service(&app, req).await;
        assert_eq!(resp.status(), actix_web::http::StatusCode::OK);
        
        let body: serde_json::Value = test::read_body_json(resp).await;
        assert!(body["first_name"].is_string());
    }
    
    #[test]
    fn test_version_parsing() {
        use std::str::FromStr;
        use versioning::header_based::ApiVersion;
        
        assert_eq!(ApiVersion::from_str("1").unwrap(), ApiVersion::V1);
        assert_eq!(ApiVersion::from_str("v1").unwrap(), ApiVersion::V1);
        assert_eq!(ApiVersion::from_str("2.0").unwrap(), ApiVersion::V2);
        assert!(ApiVersion::from_str("99").is_err());
    }
    
    #[test]
    fn test_backward_compatibility() {
        use compatibility::*;
        
        let v2_req = CreateUserV2Request {
            first_name: "John".to_string(),
            last_name: "Doe".to_string(),
            email: "john@example.com".to_string(),
            password: "secret123".to_string(),
            phone: Some("+66891234567".to_string()),
        };
        
        let domain_user = UserDomain::from_v2_request(v2_req);
        let v1_response = domain_user.to_v1();
        
        assert_eq!(v1_response.name, "John Doe");
        assert_eq!(v1_response.email, "john@example.com");
        
        let v2_response = domain_user.to_v2();
        assert_eq!(v2_response.first_name, "John");
        assert_eq!(v2_response.last_name, "Doe");
    }
}
```

## สรุป

การเลือกกลยุทธ์ Versioning:

| กลยุทธ์ | ข้อดี | ข้อเสีย |
|---------|-------|---------|
| URL Path | ชัดเจน, cacheable | URL ยาว |
| Header | URL สะอาด | ซับซ้อน, ไม่ cacheable ง่าย |
| Query Param | ง่าย | ไม่ standard |

แนะนำ: **URL Path** สำหรับ public API เพราะชัดเจนที่สุด

---

## Navigation

- [← Part 065: Microservices Architecture](../part_065/README.md)
- [→ Part 067: Caching Strategies](../part_067/README.md)
- [กลับหน้าหลัก](../../README.md)
