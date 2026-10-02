# Part 050: Session Management 🍪

## 🎯 เป้าหมายของ Part นี้

- Cookie-based sessions
- ใช้ `actix-session` crate
- Redis session backend
- Session data management
- Session expiry
- Secure cookies (HttpOnly, Secure, SameSite)
- Session invalidation
- Multiple device sessions
- Practical: session-based authentication

---

## 1. Session vs JWT

| Feature | Session | JWT |
|---------|---------|-----|
| Storage | Server (Redis/DB) | Client (cookie/header) |
| Revocation | ทันที | รอ expire |
| Size | เล็ก (session ID) | ใหญ่กว่า |
| Scalability | ต้องแชร์ storage | Stateless |
| Security | ควบคุมได้มาก | Limited revocation |

**เมื่อไหร่ควรใช้ Session:**
- ต้องการ revoke ทันที (logout, security breach)
- เก็บข้อมูลมากในฝั่ง server
- Traditional web apps

---

## 2. Setup

### 2.1 Cargo.toml

```toml
[package]
name = "session-management"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
actix-session = { version = "0.9", features = ["cookie-session", "redis-session"] }
actix-web-lab = "0.20"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
bcrypt = "0.15"
redis = { version = "0.23", features = ["tokio-comp"] }
log = "0.4"
env_logger = "0.10"
dotenv = "0.15"
rand = "0.8"
```

---

## 3. Cookie Session Setup

```rust
// src/session/cookie_session.rs
use actix_session::{
    SessionMiddleware,
    storage::CookieSessionStore,
};
use actix_web::cookie::{Key, SameSite};

/// สร้าง Cookie Session middleware
pub fn create_cookie_session_middleware(
    secret: &str,
    secure: bool,
) -> SessionMiddleware<CookieSessionStore> {
    let key = Key::from(secret.as_bytes());
    
    SessionMiddleware::builder(CookieSessionStore::default(), key)
        // ✅ HttpOnly: JavaScript ไม่สามารถอ่าน cookie ได้
        .cookie_http_only(true)
        // ✅ Secure: ส่งผ่าน HTTPS เท่านั้น
        .cookie_secure(secure)
        // ✅ SameSite: ป้องกัน CSRF
        .cookie_same_site(SameSite::Lax)
        // ชื่อ cookie
        .cookie_name("session".to_string())
        // Path
        .cookie_path("/".to_string())
        // Domain (ใน production ตั้งเป็น domain จริง)
        // .cookie_domain(Some("yourdomain.com".to_string()))
        .build()
}
```

---

## 4. Redis Session Backend

```rust
// src/session/redis_session.rs
use actix_session::{
    SessionMiddleware,
    storage::RedisActorSessionStore,
};
use actix_web::cookie::{Key, SameSite, time::Duration};

/// สร้าง Redis Session middleware
pub async fn create_redis_session_middleware(
    redis_url: &str,
    secret: &str,
    secure: bool,
) -> anyhow::Result<SessionMiddleware<RedisActorSessionStore>> {
    let key = Key::from(secret.as_bytes());
    let store = RedisActorSessionStore::new(redis_url)?;
    
    Ok(SessionMiddleware::builder(store, key)
        .cookie_http_only(true)
        .cookie_secure(secure)
        .cookie_same_site(SameSite::Lax)
        .cookie_name("session".to_string())
        // Session TTL: 24 ชั่วโมง
        .session_lifecycle(
            actix_session::config::PersistentSession::default()
                .session_ttl(Duration::days(1))
        )
        .build())
}
```

---

## 5. Session Data Management

```rust
// src/session/data.rs
use actix_session::Session;
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use chrono::{DateTime, Utc};

/// ข้อมูลใน Session
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SessionUser {
    pub user_id: Uuid,
    pub email: String,
    pub roles: Vec<String>,
    pub login_at: DateTime<Utc>,
    pub device_id: String,
    pub ip_address: Option<String>,
}

/// Session keys
pub const SESSION_USER_KEY: &str = "user";
pub const SESSION_FLASH_KEY: &str = "flash";
pub const SESSION_CSRF_KEY: &str = "csrf_token";

/// Helper สำหรับ get/set session data
pub struct SessionManager<'a> {
    session: &'a Session,
}

impl<'a> SessionManager<'a> {
    pub fn new(session: &'a Session) -> Self {
        Self { session }
    }
    
    /// Get current user จาก session
    pub fn get_user(&self) -> Option<SessionUser> {
        self.session
            .get::<SessionUser>(SESSION_USER_KEY)
            .ok()
            .flatten()
    }
    
    /// Set user ใน session (login)
    pub fn set_user(&self, user: &SessionUser) -> anyhow::Result<()> {
        self.session.insert(SESSION_USER_KEY, user)?;
        Ok(())
    }
    
    /// ลบ user จาก session (logout)
    pub fn clear_user(&self) {
        self.session.remove(SESSION_USER_KEY);
    }
    
    /// Purge session ทั้งหมด
    pub fn destroy(&self) {
        self.session.purge();
    }
    
    /// Get CSRF token
    pub fn get_csrf_token(&self) -> Option<String> {
        self.session.get::<String>(SESSION_CSRF_KEY).ok().flatten()
    }
    
    /// Set CSRF token
    pub fn set_csrf_token(&self, token: &str) -> anyhow::Result<()> {
        self.session.insert(SESSION_CSRF_KEY, token.to_string())?;
        Ok(())
    }
    
    /// Set flash message
    pub fn set_flash(&self, message: &str) -> anyhow::Result<()> {
        self.session.insert(SESSION_FLASH_KEY, message.to_string())?;
        Ok(())
    }
    
    /// Get และลบ flash message (one-time use)
    pub fn take_flash(&self) -> Option<String> {
        let msg = self.session.get::<String>(SESSION_FLASH_KEY).ok().flatten();
        self.session.remove(SESSION_FLASH_KEY);
        msg
    }
    
    /// Renew session (สร้าง session ID ใหม่ ป้องกัน session fixation)
    pub fn renew(&self) {
        self.session.renew();
    }
    
    /// ตรวจสอบว่า user authenticated หรือไม่
    pub fn is_authenticated(&self) -> bool {
        self.get_user().is_some()
    }
    
    /// Require authenticated user หรือ return error
    pub fn require_user(&self) -> anyhow::Result<SessionUser> {
        self.get_user()
            .ok_or_else(|| anyhow::anyhow!("Not authenticated"))
    }
}
```

---

## 6. Multiple Device Sessions

### 6.1 Database Schema

```sql
-- migrations/001_sessions.sql
CREATE TABLE user_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    session_id VARCHAR(255) UNIQUE NOT NULL,  -- Redis session key
    device_name VARCHAR(255),
    device_type VARCHAR(50),  -- 'mobile', 'desktop', 'tablet'
    user_agent TEXT,
    ip_address INET,
    last_activity TIMESTAMPTZ DEFAULT NOW(),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ NOT NULL,
    is_active BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_sessions_user_id ON user_sessions(user_id);
CREATE INDEX idx_sessions_session_id ON user_sessions(session_id);
CREATE INDEX idx_sessions_active ON user_sessions(user_id, is_active) WHERE is_active = TRUE;
```

### 6.2 Session Service

```rust
// src/session/service.rs
use sqlx::PgPool;
use uuid::Uuid;
use chrono::{Utc, Duration};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize, sqlx::FromRow)]
pub struct UserSessionRecord {
    pub id: Uuid,
    pub user_id: Uuid,
    pub session_id: String,
    pub device_name: Option<String>,
    pub device_type: Option<String>,
    pub user_agent: Option<String>,
    pub ip_address: Option<String>,
    pub last_activity: chrono::DateTime<Utc>,
    pub created_at: chrono::DateTime<Utc>,
    pub expires_at: chrono::DateTime<Utc>,
    pub is_active: bool,
}

/// สร้าง session record ใน database
pub async fn create_session_record(
    pool: &PgPool,
    user_id: Uuid,
    session_id: &str,
    device_info: &DeviceInfo,
    ttl_days: i64,
) -> anyhow::Result<UserSessionRecord> {
    let expires_at = Utc::now() + Duration::days(ttl_days);
    
    let record = sqlx::query_as!(
        UserSessionRecord,
        r#"INSERT INTO user_sessions (user_id, session_id, device_name, device_type, user_agent, ip_address, expires_at)
           VALUES ($1, $2, $3, $4, $5, $6, $7)
           RETURNING id, user_id, session_id, device_name, device_type, user_agent,
                     ip_address::text, last_activity, created_at, expires_at, is_active"#,
        user_id,
        session_id,
        device_info.device_name,
        device_info.device_type,
        device_info.user_agent,
        device_info.ip_address,
        expires_at
    )
    .fetch_one(pool)
    .await?;
    
    Ok(record)
}

/// ดึง active sessions ของ user ทั้งหมด
pub async fn get_user_sessions(
    pool: &PgPool,
    user_id: Uuid,
) -> anyhow::Result<Vec<UserSessionRecord>> {
    let sessions = sqlx::query_as!(
        UserSessionRecord,
        r#"SELECT id, user_id, session_id, device_name, device_type, user_agent,
                  ip_address::text, last_activity, created_at, expires_at, is_active
           FROM user_sessions
           WHERE user_id = $1 AND is_active = TRUE AND expires_at > NOW()
           ORDER BY last_activity DESC"#,
        user_id
    )
    .fetch_all(pool)
    .await?;
    
    Ok(sessions)
}

/// Revoke specific session
pub async fn revoke_session(
    pool: &PgPool,
    session_id: &str,
    user_id: Uuid,
) -> anyhow::Result<bool> {
    let result = sqlx::query!(
        "UPDATE user_sessions SET is_active = FALSE WHERE session_id = $1 AND user_id = $2",
        session_id,
        user_id
    )
    .execute(pool)
    .await?;
    
    Ok(result.rows_affected() > 0)
}

/// Revoke ทุก sessions ของ user (logout everywhere)
pub async fn revoke_all_sessions(
    pool: &PgPool,
    user_id: Uuid,
    except_session: Option<&str>,
) -> anyhow::Result<u64> {
    let result = if let Some(current) = except_session {
        sqlx::query!(
            "UPDATE user_sessions SET is_active = FALSE WHERE user_id = $1 AND session_id != $2",
            user_id,
            current
        )
        .execute(pool)
        .await?
    } else {
        sqlx::query!(
            "UPDATE user_sessions SET is_active = FALSE WHERE user_id = $1",
            user_id
        )
        .execute(pool)
        .await?
    };
    
    Ok(result.rows_affected())
}

/// Update last_activity
pub async fn update_session_activity(
    pool: &PgPool,
    session_id: &str,
) -> anyhow::Result<()> {
    sqlx::query!(
        "UPDATE user_sessions SET last_activity = NOW() WHERE session_id = $1",
        session_id
    )
    .execute(pool)
    .await?;
    
    Ok(())
}

/// Cleanup expired sessions
pub async fn cleanup_expired_sessions(pool: &PgPool) -> anyhow::Result<u64> {
    let result = sqlx::query!(
        "DELETE FROM user_sessions WHERE expires_at < NOW() OR is_active = FALSE"
    )
    .execute(pool)
    .await?;
    
    Ok(result.rows_affected())
}

#[derive(Debug, Clone)]
pub struct DeviceInfo {
    pub device_name: Option<String>,
    pub device_type: Option<String>,
    pub user_agent: Option<String>,
    pub ip_address: Option<String>,
}

impl DeviceInfo {
    pub fn from_request(req: &actix_web::HttpRequest) -> Self {
        let user_agent = req.headers()
            .get("User-Agent")
            .and_then(|h| h.to_str().ok())
            .map(|s| s.to_string());
        
        let ip = req.peer_addr()
            .map(|a| a.ip().to_string());
        
        let device_type = user_agent.as_ref().map(|ua| {
            let ua_lower = ua.to_lowercase();
            if ua_lower.contains("mobile") || ua_lower.contains("android") {
                "mobile"
            } else if ua_lower.contains("tablet") || ua_lower.contains("ipad") {
                "tablet"
            } else {
                "desktop"
            }
        }).unwrap_or("unknown").to_string();
        
        Self {
            device_name: None,
            device_type: Some(device_type),
            user_agent,
            ip_address: ip,
        }
    }
}
```

---

## 7. Session Authentication Handlers

```rust
// src/handlers/session_auth.rs
use actix_web::{web, HttpRequest, HttpResponse, Result};
use actix_session::Session;
use sqlx::PgPool;

use crate::session::{
    data::{SessionManager, SessionUser, SESSION_USER_KEY},
    service::{DeviceInfo, create_session_record, get_user_sessions, 
               revoke_session, revoke_all_sessions},
};

#[derive(Debug, serde::Deserialize)]
pub struct LoginRequest {
    pub email: String,
    pub password: String,
    pub remember_me: bool,
}

/// Login ด้วย session
pub async fn login(
    req: HttpRequest,
    session: Session,
    pool: web::Data<PgPool>,
    body: web::Json<LoginRequest>,
) -> Result<HttpResponse> {
    let session_mgr = SessionManager::new(&session);
    
    // ค้นหา user
    let user = sqlx::query!(
        "SELECT id, email, password_hash, display_name FROM users WHERE email = $1",
        body.email
    )
    .fetch_optional(pool.get_ref())
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    let user = match user {
        Some(u) => u,
        None => {
            // ยัง hash เพื่อป้องกัน timing attack
            let _ = bcrypt::verify(&body.password, "$2b$12$dummy");
            return Ok(HttpResponse::Unauthorized().json(serde_json::json!({
                "error": "Invalid credentials"
            })));
        }
    };
    
    let is_valid = bcrypt::verify(&body.password, &user.password_hash)
        .unwrap_or(false);
    
    if !is_valid {
        return Ok(HttpResponse::Unauthorized().json(serde_json::json!({
            "error": "Invalid credentials"
        })));
    }
    
    // ✅ Renew session ID ป้องกัน session fixation
    session_mgr.renew();
    
    let session_user = SessionUser {
        user_id: user.id,
        email: user.email.clone(),
        roles: vec!["user".to_string()],
        login_at: chrono::Utc::now(),
        device_id: uuid::Uuid::new_v4().to_string(),
        ip_address: req.peer_addr().map(|a| a.ip().to_string()),
    };
    
    session_mgr.set_user(&session_user)
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    // สร้าง CSRF token
    let csrf_token = crate::security::csrf::generate_csrf_token();
    session_mgr.set_csrf_token(&csrf_token)
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    // บันทึก session ลง database (สำหรับ multi-device tracking)
    let device_info = DeviceInfo::from_request(&req);
    let session_id = session.status().to_string(); // actix-session ไม่ expose ID โดยตรง
    
    Ok(HttpResponse::Ok()
        .append_header(("X-CSRF-Token", csrf_token))
        .json(serde_json::json!({
            "message": "Login successful",
            "user": {
                "id": user.id,
                "email": user.email,
            }
        })))
}

/// Logout
pub async fn logout(session: Session) -> Result<HttpResponse> {
    let session_mgr = SessionManager::new(&session);
    
    // ลบ session ทั้งหมด
    session_mgr.destroy();
    
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "message": "Logged out successfully"
    })))
}

/// ดูข้อมูล current user
pub async fn me(session: Session) -> Result<HttpResponse> {
    let session_mgr = SessionManager::new(&session);
    
    match session_mgr.get_user() {
        Some(user) => Ok(HttpResponse::Ok().json(serde_json::json!({
            "user_id": user.user_id,
            "email": user.email,
            "roles": user.roles,
            "login_at": user.login_at,
        }))),
        None => Ok(HttpResponse::Unauthorized().json(serde_json::json!({
            "error": "Not authenticated"
        }))),
    }
}

/// List active sessions ของ user
pub async fn list_sessions(
    session: Session,
    pool: web::Data<PgPool>,
) -> Result<HttpResponse> {
    let session_mgr = SessionManager::new(&session);
    let user = session_mgr.require_user()
        .map_err(|_| actix_web::error::ErrorUnauthorized("Not authenticated"))?;
    
    let sessions = get_user_sessions(pool.get_ref(), user.user_id)
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    let session_list: Vec<serde_json::Value> = sessions.iter().map(|s| serde_json::json!({
        "id": s.id,
        "device_type": s.device_type,
        "device_name": s.device_name,
        "ip_address": s.ip_address,
        "last_activity": s.last_activity,
        "created_at": s.created_at,
    })).collect();
    
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "sessions": session_list,
        "total": sessions.len()
    })))
}

/// Logout from all devices
pub async fn logout_all(
    session: Session,
    pool: web::Data<PgPool>,
) -> Result<HttpResponse> {
    let session_mgr = SessionManager::new(&session);
    let user = session_mgr.require_user()
        .map_err(|_| actix_web::error::ErrorUnauthorized("Not authenticated"))?;
    
    let revoked = revoke_all_sessions(pool.get_ref(), user.user_id, None)
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    // ลบ current session ด้วย
    session_mgr.destroy();
    
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "message": "Logged out from all devices",
        "sessions_revoked": revoked
    })))
}
```

---

## 8. Session Auth Middleware

```rust
// src/middleware/session_auth.rs
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpMessage, HttpResponse,
};
use actix_session::SessionExt;
use futures::future::{ready, Ready, LocalBoxFuture};
use std::rc::Rc;

use crate::session::data::{SessionManager, SessionUser};

pub struct RequireSession;

impl<S, B> Transform<S, ServiceRequest> for RequireSession
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Transform = RequireSessionMiddleware<S>;
    type InitError = ();
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(RequireSessionMiddleware {
            service: Rc::new(service),
        }))
    }
}

pub struct RequireSessionMiddleware<S> {
    service: Rc<S>,
}

impl<S, B> Service<ServiceRequest> for RequireSessionMiddleware<S>
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
        
        Box::pin(async move {
            let session = req.get_session();
            let session_mgr = SessionManager::new(&session);
            
            match session_mgr.get_user() {
                Some(user) => {
                    // Inject user ใน request extensions
                    req.extensions_mut().insert(user);
                    service.call(req).await
                },
                None => {
                    Ok(req.into_response(
                        HttpResponse::Unauthorized()
                            .json(serde_json::json!({"error": "Authentication required"}))
                            .map_into_boxed_body()
                    ))
                }
            }
        })
    }
}
```

---

## 9. Session Cleanup Job

```rust
// src/tasks/session_cleanup.rs
use sqlx::PgPool;
use std::time::Duration;

/// Background task ลบ expired sessions
pub async fn start_session_cleanup_task(pool: PgPool) {
    tokio::spawn(async move {
        let mut interval = tokio::time::interval(Duration::from_secs(3600)); // ทุกชั่วโมง
        
        loop {
            interval.tick().await;
            
            match crate::session::service::cleanup_expired_sessions(&pool).await {
                Ok(count) => {
                    if count > 0 {
                        log::info!("Cleaned up {} expired sessions", count);
                    }
                },
                Err(e) => log::error!("Session cleanup error: {}", e),
            }
        }
    });
}
```

---

## 10. Main Application

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware};
use sqlx::postgres::PgPoolOptions;
use dotenv::dotenv;

mod session {
    pub mod cookie_session;
    pub mod redis_session;
    pub mod data;
    pub mod service;
}
mod middleware {
    pub mod session_auth;
}
mod handlers {
    pub mod session_auth;
}
mod security {
    pub mod csrf;
}
mod tasks {
    pub mod session_cleanup;
}

use session::redis_session::create_redis_session_middleware;
use middleware::session_auth::RequireSession;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv().ok();
    env_logger::init();
    
    let database_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    let redis_url = std::env::var("REDIS_URL")
        .unwrap_or_else(|_| "redis://127.0.0.1:6379".to_string());
    let session_secret = std::env::var("SESSION_SECRET")
        .expect("SESSION_SECRET must be set");
    let is_production = std::env::var("APP_ENV")
        .map(|e| e == "production")
        .unwrap_or(false);
    
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await
        .expect("Failed to connect to database");
    
    sqlx::migrate!("./migrations").run(&pool).await
        .expect("Failed to run migrations");
    
    // Start cleanup task
    tasks::session_cleanup::start_session_cleanup_task(pool.clone()).await;
    
    let session_middleware = create_redis_session_middleware(
        &redis_url,
        &session_secret,
        is_production,
    )
    .await
    .expect("Failed to create session middleware");
    
    log::info!("Starting server on 0.0.0.0:8080");
    
    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(pool.clone()))
            .wrap(middleware::Logger::default())
            .wrap(session_middleware.clone())
            // Public routes
            .service(
                web::scope("/auth")
                    .route("/login", web::post().to(handlers::session_auth::login))
                    .route("/logout", web::post().to(handlers::session_auth::logout))
            )
            // Protected routes
            .service(
                web::scope("/api")
                    .wrap(RequireSession)
                    .route("/me", web::get().to(handlers::session_auth::me))
                    .route("/sessions", web::get().to(handlers::session_auth::list_sessions))
                    .route("/sessions/revoke-all", web::post().to(handlers::session_auth::logout_all))
            )
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

---

## 11. Testing

```rust
#[cfg(test)]
mod tests {
    use actix_web::{test, web, App};
    use actix_session::{SessionMiddleware, storage::CookieSessionStore};
    use actix_web::cookie::Key;

    #[actix_web::test]
    async fn test_login_creates_session() {
        let key = Key::from(b"a_very_long_secret_key_for_testing_purposes");
        
        let app = test::init_service(
            App::new()
                .wrap(SessionMiddleware::new(CookieSessionStore::default(), key))
                .route("/auth/login", web::post().to(|session: actix_session::Session| async move {
                    session.insert("user_id", "test-user-123").unwrap();
                    actix_web::HttpResponse::Ok().json(serde_json::json!({"ok": true}))
                }))
                .route("/api/me", web::get().to(|session: actix_session::Session| async move {
                    let user_id: Option<String> = session.get("user_id").unwrap_or(None);
                    match user_id {
                        Some(id) => actix_web::HttpResponse::Ok().json(serde_json::json!({"id": id})),
                        None => actix_web::HttpResponse::Unauthorized().finish(),
                    }
                }))
        ).await;
        
        // Login
        let login_req = test::TestRequest::post()
            .uri("/auth/login")
            .set_json(serde_json::json!({"email": "test@test.com", "password": "pass"}))
            .to_request();
        
        let resp = test::call_service(&app, login_req).await;
        assert!(resp.status().is_success());
        
        // ดึง cookie จาก response
        let cookie = resp.headers()
            .get("Set-Cookie")
            .and_then(|v| v.to_str().ok())
            .unwrap_or("");
        
        // ใช้ session cookie ใน next request
        let me_req = test::TestRequest::get()
            .uri("/api/me")
            .append_header(("Cookie", cookie))
            .to_request();
        
        let me_resp = test::call_service(&app, me_req).await;
        assert!(me_resp.status().is_success());
    }
    
    #[test]
    fn test_session_user_serialization() {
        use crate::session::data::SessionUser;
        
        let user = SessionUser {
            user_id: uuid::Uuid::new_v4(),
            email: "test@example.com".to_string(),
            roles: vec!["user".to_string()],
            login_at: chrono::Utc::now(),
            device_id: uuid::Uuid::new_v4().to_string(),
            ip_address: Some("127.0.0.1".to_string()),
        };
        
        // Serialize/deserialize
        let json = serde_json::to_string(&user).unwrap();
        let decoded: SessionUser = serde_json::from_str(&json).unwrap();
        
        assert_eq!(user.user_id, decoded.user_id);
        assert_eq!(user.email, decoded.email);
    }
}
```

---

## 12. สรุปสิ่งที่เรียนรู้

✅ Cookie-based session ด้วย actix-session  
✅ Redis session backend  
✅ Session data management  
✅ Secure cookie flags (HttpOnly, Secure, SameSite)  
✅ Session renewal ป้องกัน session fixation  
✅ Multiple device session tracking  
✅ Session revocation (single + all devices)  
✅ Session cleanup background task  
✅ Session auth middleware  
✅ Complete session-based auth flow  

---

## 13. ขั้นตอนต่อไป

Part นี้ครอบคลุม Session Management ครบถ้วน ในส่วนต่อไปเราจะพูดถึง:
- **Part 051**: WebSockets
- **Part 052**: Server-Sent Events
- **Part 053**: Background Jobs
- **Part 054**: Caching Strategies

---

*[← Part 049: Security Audit](../part_049/README.md) | [Part 051: WebSockets →](../part_051/README.md)*
