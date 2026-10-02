# Part 049: Security Audit and Best Practices 🔍

## 🎯 เป้าหมายของ Part นี้

- OWASP Top 10 สำหรับ APIs
- SQL injection prevention
- XSS prevention
- CSRF protection
- Sensitive data exposure prevention
- Security headers
- cargo-audit สำหรับ dependencies
- Environment variable secrets
- Security review checklist

---

## 1. OWASP API Security Top 10

### 1.1 API1: Broken Object Level Authorization (BOLA)

```rust
// src/security/bola.rs
use actix_web::{web, HttpRequest, HttpResponse};
use sqlx::PgPool;
use uuid::Uuid;

/// ตัวอย่าง VULNERABLE code
/// ❌ ไม่ดี: ไม่ตรวจสอบว่า user เป็นเจ้าของ resource
pub async fn get_user_data_insecure(
    pool: web::Data<PgPool>,
    path: web::Path<Uuid>,
) -> actix_web::Result<HttpResponse> {
    let user_id = path.into_inner();
    
    // ❌ ใครก็ดูได้! ไม่ตรวจสอบ ownership
    let data = sqlx::query!(
        "SELECT * FROM sensitive_data WHERE user_id = $1",
        user_id
    )
    .fetch_optional(pool.get_ref())
    .await?;
    
    Ok(HttpResponse::Ok().json(data.map(|d| serde_json::json!({
        "data": d.content
    }))))
}

/// ✅ ดี: ตรวจสอบ ownership ก่อนเสมอ
pub async fn get_user_data_secure(
    req: HttpRequest,
    pool: web::Data<PgPool>,
    path: web::Path<Uuid>,
) -> actix_web::Result<HttpResponse> {
    let resource_id = path.into_inner();
    
    // ดึง current user จาก JWT claims
    let current_user_id = req
        .extensions()
        .get::<crate::auth::Claims>()
        .map(|c| c.user_id)
        .ok_or_else(|| actix_web::error::ErrorUnauthorized("Not authenticated"))?;
    
    let data = sqlx::query!(
        "SELECT * FROM sensitive_data WHERE id = $1 AND user_id = $2",
        resource_id,
        current_user_id  // ✅ ตรวจสอบ ownership เสมอ
    )
    .fetch_optional(pool.get_ref())
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    match data {
        Some(d) => Ok(HttpResponse::Ok().json(serde_json::json!({"data": "secured"}))),
        None => Ok(HttpResponse::NotFound().json(serde_json::json!({"error": "Not found"}))),
    }
}
```

### 1.2 API2: Broken Authentication

```rust
// src/security/auth_security.rs
use actix_web::HttpResponse;

/// ตัวอย่าง secure login
pub async fn secure_login_handler(
    pool: web::Data<PgPool>,
    req: web::Json<LoginRequest>,
) -> actix_web::Result<HttpResponse> {
    // 1. ✅ Generic error message - ไม่บอกว่า email หรือ password ผิด
    let user = sqlx::query!(
        "SELECT id, email, password_hash, failed_login_count, locked_until 
         FROM users WHERE email = $1",
        req.email
    )
    .fetch_optional(pool.get_ref())
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    // 2. ✅ ป้องกัน timing attack: hash ทั้งในกรณีที่ user ไม่มีอยู่
    let (user_exists, password_valid) = match &user {
        Some(u) => {
            // ตรวจสอบ account locked
            if let Some(locked_until) = u.locked_until {
                if chrono::Utc::now() < locked_until {
                    return Ok(HttpResponse::TooManyRequests().json(serde_json::json!({
                        "error": "Account temporarily locked due to too many failed attempts"
                    })));
                }
            }
            
            let valid = bcrypt::verify(&req.password, &u.password_hash)
                .unwrap_or(false);
            (true, valid)
        },
        None => {
            // ✅ ยังทำ bcrypt verify เพื่อป้องกัน timing attack
            let _ = bcrypt::verify(&req.password, "$2b$12$invalid_hash_for_timing");
            (false, false)
        }
    };
    
    if !user_exists || !password_valid {
        // ✅ อัพเดท failed_login_count
        if user_exists {
            if let Some(u) = &user {
                let new_count = u.failed_login_count + 1;
                let locked_until = if new_count >= 5 {
                    Some(chrono::Utc::now() + chrono::Duration::minutes(15))
                } else {
                    None
                };
                
                let _ = sqlx::query!(
                    "UPDATE users SET failed_login_count = $1, locked_until = $2 WHERE id = $3",
                    new_count, locked_until, u.id
                )
                .execute(pool.get_ref())
                .await;
            }
        }
        
        // ✅ Generic error message
        return Ok(HttpResponse::Unauthorized().json(serde_json::json!({
            "error": "Invalid credentials"
        })));
    }
    
    // Reset failed_login_count on successful login
    if let Some(u) = &user {
        let _ = sqlx::query!(
            "UPDATE users SET failed_login_count = 0, locked_until = NULL, last_login_at = NOW() 
             WHERE id = $1",
            u.id
        )
        .execute(pool.get_ref())
        .await;
    }
    
    Ok(HttpResponse::Ok().json(serde_json::json!({"token": "jwt_token_here"})))
}
```

---

## 2. SQL Injection Prevention

```rust
// src/security/sql_injection.rs
use sqlx::PgPool;

/// ❌ VULNERABLE: String concatenation
pub async fn search_users_insecure(
    pool: &PgPool,
    search_term: &str,
) -> anyhow::Result<Vec<serde_json::Value>> {
    // ❌ ห้ามทำ! SQL injection vulnerability
    let query = format!(
        "SELECT * FROM users WHERE name LIKE '%{}%'",
        search_term
    );
    // ถ้า search_term = "'; DROP TABLE users; --"
    // จะกลายเป็น: SELECT * FROM users WHERE name LIKE '%'; DROP TABLE users; --%'
    
    todo!("DO NOT USE THIS")
}

/// ✅ SECURE: Parameterized queries
pub async fn search_users_secure(
    pool: &PgPool,
    search_term: &str,
) -> anyhow::Result<Vec<String>> {
    // ✅ ใช้ $1 placeholder เสมอ
    let users = sqlx::query!(
        "SELECT name FROM users WHERE name ILIKE $1",
        format!("%{}%", search_term)  // safe - value ถูก escape โดย driver
    )
    .fetch_all(pool)
    .await?;
    
    Ok(users.iter().map(|u| u.name.clone()).collect())
}

/// ✅ Dynamic ORDER BY ที่ปลอดภัย
pub async fn list_users_with_sort(
    pool: &PgPool,
    sort_field: &str,
    sort_order: &str,
) -> anyhow::Result<Vec<serde_json::Value>> {
    // ✅ Whitelist approach สำหรับ dynamic column names
    let safe_field = match sort_field {
        "name" => "name",
        "email" => "email",
        "created_at" => "created_at",
        _ => "created_at", // default
    };
    
    let safe_order = match sort_order.to_lowercase().as_str() {
        "asc" => "ASC",
        "desc" => "DESC",
        _ => "ASC",
    };
    
    // ✅ Column names ต้องไม่ใช่ user input โดยตรง
    let query = format!(
        "SELECT id, name, email FROM users ORDER BY {} {}",
        safe_field, safe_order
    );
    
    // Execute raw query (column names ผ่าน whitelist แล้ว)
    let rows = sqlx::query_as::<_, (uuid::Uuid, String, String)>(&query)
        .fetch_all(pool)
        .await?;
    
    Ok(rows.iter().map(|(id, name, email)| serde_json::json!({
        "id": id, "name": name, "email": email
    })).collect())
}

/// ✅ Batch insert ที่ปลอดภัย
pub async fn bulk_insert_users(
    pool: &PgPool,
    users: &[(String, String)],
) -> anyhow::Result<()> {
    // ✅ ใช้ transactions กับ parameterized queries
    let mut tx = pool.begin().await?;
    
    for (email, name) in users {
        sqlx::query!(
            "INSERT INTO users (email, name) VALUES ($1, $2)",
            email, name
        )
        .execute(&mut *tx)
        .await?;
    }
    
    tx.commit().await?;
    Ok(())
}
```

---

## 3. XSS Prevention

```rust
// src/security/xss.rs
use serde_json::Value;

/// Sanitize output data
pub fn sanitize_for_html_output(value: &str) -> String {
    html_escape::encode_text(value).to_string()
}

/// ตรวจสอบ content ก่อน save
pub fn validate_content_for_xss(content: &str) -> bool {
    let dangerous_patterns = [
        "<script", "</script>", "javascript:",
        "onerror=", "onload=", "onclick=",
        "eval(", "document.cookie",
        "window.location",
    ];
    
    let lower = content.to_lowercase();
    !dangerous_patterns.iter().any(|p| lower.contains(p))
}

/// Response serializer ที่ sanitize strings อัตโนมัติ
pub fn sanitize_json_strings(value: Value) -> Value {
    match value {
        Value::String(s) => Value::String(sanitize_for_html_output(&s)),
        Value::Object(map) => Value::Object(
            map.into_iter()
                .map(|(k, v)| (k, sanitize_json_strings(v)))
                .collect()
        ),
        Value::Array(arr) => Value::Array(
            arr.into_iter().map(sanitize_json_strings).collect()
        ),
        other => other,
    }
}
```

---

## 4. CSRF Protection

```rust
// src/security/csrf.rs
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpResponse,
};
use futures::future::{ready, Ready, LocalBoxFuture};
use rand::{thread_rng, RngCore};
use base64::{Engine, engine::general_purpose::URL_SAFE_NO_PAD};
use std::rc::Rc;

/// สร้าง CSRF token
pub fn generate_csrf_token() -> String {
    let mut bytes = [0u8; 32];
    thread_rng().fill_bytes(&mut bytes);
    URL_SAFE_NO_PAD.encode(&bytes)
}

/// CSRF protection middleware
pub struct CsrfProtection {
    /// Header หรือ form field name
    token_header: String,
    /// Methods ที่ต้อง check CSRF
    protected_methods: Vec<actix_web::http::Method>,
    /// Paths ที่ exempt จาก CSRF
    exempt_paths: Vec<String>,
}

impl CsrfProtection {
    pub fn new() -> Self {
        Self {
            token_header: "X-CSRF-Token".to_string(),
            protected_methods: vec![
                actix_web::http::Method::POST,
                actix_web::http::Method::PUT,
                actix_web::http::Method::PATCH,
                actix_web::http::Method::DELETE,
            ],
            exempt_paths: vec![
                "/api/auth/login".to_string(),
                "/api/auth/register".to_string(),
                "/api/webhooks".to_string(),
            ],
        }
    }
    
    pub fn exempt(mut self, path: &str) -> Self {
        self.exempt_paths.push(path.to_string());
        self
    }
}

impl<S, B> Transform<S, ServiceRequest> for CsrfProtection
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Transform = CsrfMiddleware<S>;
    type InitError = ();
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(CsrfMiddleware {
            service: Rc::new(service),
            token_header: self.token_header.clone(),
            protected_methods: self.protected_methods.clone(),
            exempt_paths: self.exempt_paths.clone(),
        }))
    }
}

pub struct CsrfMiddleware<S> {
    service: Rc<S>,
    token_header: String,
    protected_methods: Vec<actix_web::http::Method>,
    exempt_paths: Vec<String>,
}

impl<S, B> Service<ServiceRequest> for CsrfMiddleware<S>
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
        let token_header = self.token_header.clone();
        let protected_methods = self.protected_methods.clone();
        let exempt_paths = self.exempt_paths.clone();
        
        Box::pin(async move {
            let method = req.method().clone();
            let path = req.path().to_string();
            
            // Skip ถ้าเป็น method ที่ไม่ต้อง protect
            if !protected_methods.contains(&method) {
                return service.call(req).await;
            }
            
            // Skip ถ้า path อยู่ใน exempt list
            if exempt_paths.iter().any(|p| path.starts_with(p)) {
                return service.call(req).await;
            }
            
            // ดึง token จาก header
            let csrf_token = req
                .headers()
                .get(&token_header)
                .and_then(|h| h.to_str().ok())
                .map(|s| s.to_string());
            
            // ดึง token จาก session
            let session_token = req
                .extensions()
                .get::<actix_session::Session>()
                .and_then(|s| s.get::<String>("csrf_token").ok().flatten());
            
            match (csrf_token, session_token) {
                (Some(request_token), Some(stored_token)) => {
                    // Constant-time comparison
                    use subtle::ConstantTimeEq;
                    if request_token.as_bytes().ct_eq(stored_token.as_bytes()).into() {
                        service.call(req).await
                    } else {
                        Ok(req.into_response(
                            HttpResponse::Forbidden()
                                .json(serde_json::json!({"error": "Invalid CSRF token"}))
                                .map_into_boxed_body()
                        ))
                    }
                },
                _ => {
                    Ok(req.into_response(
                        HttpResponse::Forbidden()
                            .json(serde_json::json!({"error": "CSRF token required"}))
                            .map_into_boxed_body()
                    ))
                }
            }
        })
    }
}
```

---

## 5. Sensitive Data Protection

```rust
// src/security/sensitive_data.rs
use serde::{Deserialize, Serialize};

/// ห้าม serialize fields ที่ sensitive
#[derive(Debug, Serialize, Deserialize)]
pub struct UserResponse {
    pub id: String,
    pub email: String,
    pub display_name: String,
    
    // ✅ ไม่ include sensitive fields
    #[serde(skip_serializing)]
    pub password_hash: String,
    
    #[serde(skip_serializing)]
    pub reset_token_hash: Option<String>,
}

/// Mask sensitive data
pub fn mask_email(email: &str) -> String {
    let parts: Vec<&str> = email.splitn(2, '@').collect();
    if parts.len() == 2 {
        let name = parts[0];
        let domain = parts[1];
        
        if name.len() <= 2 {
            format!("{}**@{}", &name[..1], domain)
        } else {
            format!("{}**{}@{}", &name[..2], &name[name.len()-1..], domain)
        }
    } else {
        "**masked**".to_string()
    }
}

pub fn mask_phone(phone: &str) -> String {
    let digits: String = phone.chars().filter(|c| c.is_ascii_digit()).collect();
    if digits.len() >= 8 {
        format!("{}****{}", &digits[..3], &digits[digits.len()-2..])
    } else {
        "****".to_string()
    }
}

pub fn mask_credit_card(card: &str) -> String {
    let digits: String = card.chars().filter(|c| c.is_ascii_digit()).collect();
    if digits.len() >= 4 {
        format!("****-****-****-{}", &digits[digits.len()-4..])
    } else {
        "****".to_string()
    }
}

/// ตรวจสอบว่า log message ไม่มี sensitive data
pub fn check_log_safety(message: &str) -> bool {
    let patterns = [
        "password", "secret", "token", "key", "credential",
        "ssn", "credit_card", "cvv",
    ];
    
    let lower = message.to_lowercase();
    !patterns.iter().any(|p| lower.contains(p))
}
```

---

## 6. cargo-audit

```bash
# ติดตั้ง cargo-audit
cargo install cargo-audit

# ตรวจสอบ vulnerabilities ใน dependencies
cargo audit

# ตรวจสอบและแสดง details
cargo audit --deny warnings

# อัพเดท advisory database
cargo audit fetch

# ตรวจสอบแบบ JSON output
cargo audit --json

# Fix vulnerabilities อัตโนมัติ (บางส่วน)
cargo audit fix
```

### 6.1 Security CI Pipeline

```yaml
# .github/workflows/security.yml
name: Security Audit

on:
  push:
    branches: [main, develop]
  pull_request:
  schedule:
    - cron: '0 0 * * 1'  # ทุกวันจันทร์ midnight

jobs:
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
        
      - name: Install cargo-audit
        run: cargo install cargo-audit
        
      - name: Run Security Audit
        run: cargo audit --deny warnings
        
      - name: Check for unused dependencies
        run: |
          cargo install cargo-udeps
          cargo +nightly udeps
```

---

## 7. Environment Variables Best Practices

```rust
// src/config/secrets.rs
use std::env;

#[derive(Debug, Clone)]
pub struct AppSecrets {
    pub database_url: String,
    pub jwt_secret: String,
    pub session_secret: String,
    pub smtp_password: Option<String>,
    pub redis_url: String,
    pub google_client_secret: Option<String>,
    pub github_client_secret: Option<String>,
}

impl AppSecrets {
    pub fn from_env() -> anyhow::Result<Self> {
        Ok(Self {
            database_url: env::var("DATABASE_URL")
                .map_err(|_| anyhow::anyhow!("DATABASE_URL not set"))?,
            
            jwt_secret: {
                let secret = env::var("JWT_SECRET")
                    .map_err(|_| anyhow::anyhow!("JWT_SECRET not set"))?;
                
                // ✅ ตรวจสอบว่า secret ยาวพอ
                if secret.len() < 32 {
                    return Err(anyhow::anyhow!("JWT_SECRET must be at least 32 characters"));
                }
                secret
            },
            
            session_secret: {
                let secret = env::var("SESSION_SECRET")
                    .map_err(|_| anyhow::anyhow!("SESSION_SECRET not set"))?;
                
                if secret.len() < 64 {
                    return Err(anyhow::anyhow!("SESSION_SECRET must be at least 64 characters"));
                }
                secret
            },
            
            smtp_password: env::var("SMTP_PASSWORD").ok(),
            
            redis_url: env::var("REDIS_URL")
                .unwrap_or_else(|_| "redis://127.0.0.1:6379".to_string()),
            
            google_client_secret: env::var("GOOGLE_CLIENT_SECRET").ok(),
            github_client_secret: env::var("GITHUB_CLIENT_SECRET").ok(),
        })
    }
}

/// ✅ ไม่ log secrets เด็ดขาด
impl std::fmt::Display for AppSecrets {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "AppSecrets {{ database_url: [REDACTED], jwt_secret: [REDACTED], ... }}")
    }
}
```

---

## 8. Security Review Checklist

```rust
// src/security/checklist.rs

/// Security checklist สำหรับ code review
///
/// ## Authentication & Authorization
/// - [ ] ทุก endpoint ที่ต้องการ auth มี middleware
/// - [ ] JWT secret ยาวอย่างน้อย 32 chars
/// - [ ] JWT expiry time เหมาะสม (access: 15min-1h, refresh: 7-30d)
/// - [ ] Password hash ด้วย bcrypt/argon2 (ไม่ใช้ MD5/SHA)
/// - [ ] ตรวจสอบ ownership ก่อน access resource เสมอ
/// - [ ] Brute force protection (rate limit + account lock)
///
/// ## Input Validation
/// - [ ] Validate ทุก user input
/// - [ ] ใช้ parameterized queries เสมอ (ห้าม string concat)
/// - [ ] Sanitize HTML output
/// - [ ] File upload validation (type, size, magic bytes)
/// - [ ] Input length limits
///
/// ## Secrets Management
/// - [ ] ไม่ hardcode secrets ใน code
/// - [ ] ใช้ environment variables
/// - [ ] ไม่ commit .env ลง git
/// - [ ] ใช้ secret manager ใน production
/// - [ ] Rotate secrets สม่ำเสมอ
///
/// ## Security Headers
/// - [ ] HSTS
/// - [ ] X-Content-Type-Options: nosniff
/// - [ ] X-Frame-Options: DENY
/// - [ ] CSP header
/// - [ ] Referrer-Policy
///
/// ## Dependencies
/// - [ ] Run cargo audit
/// - [ ] Update dependencies สม่ำเสมอ
/// - [ ] ใช้ minimum required permissions
///
/// ## Logging & Monitoring
/// - [ ] Log security events (login, logout, failed auth)
/// - [ ] ไม่ log sensitive data
/// - [ ] Structured logging
/// - [ ] Alert บน suspicious activities
///
/// ## Data Protection
/// - [ ] Encrypt sensitive data at rest
/// - [ ] HTTPS สำหรับ data in transit
/// - [ ] ไม่ expose internal error details
/// - [ ] Mask sensitive data ใน responses
///
/// ## CSRF
/// - [ ] CSRF tokens สำหรับ state-changing operations
/// - [ ] SameSite cookies
/// - [ ] Origin header validation

pub struct SecurityChecklist {
    items: Vec<ChecklistItem>,
}

#[derive(Debug)]
pub struct ChecklistItem {
    pub category: String,
    pub description: String,
    pub is_automated: bool,
    pub check_fn: Option<fn() -> bool>,
}

impl SecurityChecklist {
    pub fn run_automated_checks() -> Vec<(String, bool, String)> {
        let mut results = Vec::new();
        
        // ตรวจสอบ environment variables
        let jwt_secret = std::env::var("JWT_SECRET").unwrap_or_default();
        results.push((
            "JWT_SECRET length".to_string(),
            jwt_secret.len() >= 32,
            if jwt_secret.len() >= 32 {
                "PASS: JWT_SECRET is long enough".to_string()
            } else {
                format!("FAIL: JWT_SECRET is only {} chars, need 32+", jwt_secret.len())
            }
        ));
        
        results.push((
            "HTTPS enabled".to_string(),
            std::env::var("TLS_CERT_PATH").is_ok(),
            if std::env::var("TLS_CERT_PATH").is_ok() {
                "PASS: TLS configured".to_string()
            } else {
                "WARN: No TLS cert configured".to_string()
            }
        ));
        
        results.push((
            "DATABASE_URL not exposed".to_string(),
            std::env::var("DATABASE_URL")
                .map(|url| !url.contains("localhost") || 
                     std::env::var("APP_ENV").unwrap_or_default() != "production")
                .unwrap_or(false),
            "INFO: Check DATABASE_URL configuration".to_string()
        ));
        
        results
    }
}
```

---

## 9. Practical: Security Review

```rust
// src/security/audit.rs

/// ทำการ security scan แบบ runtime
pub async fn security_health_check() -> serde_json::Value {
    let checks = SecurityChecklist::run_automated_checks();
    
    let passed: Vec<_> = checks.iter().filter(|(_, ok, _)| *ok).collect();
    let failed: Vec<_> = checks.iter().filter(|(_, ok, _)| !ok).collect();
    
    serde_json::json!({
        "timestamp": chrono::Utc::now(),
        "total_checks": checks.len(),
        "passed": passed.len(),
        "failed": failed.len(),
        "results": checks.iter().map(|(name, ok, msg)| serde_json::json!({
            "check": name,
            "passed": ok,
            "message": msg
        })).collect::<Vec<_>>()
    })
}

/// Log security event
pub fn log_security_event(
    event_type: &str,
    user_id: Option<&str>,
    ip: Option<&str>,
    details: serde_json::Value,
) {
    log::warn!(
        "SECURITY_EVENT type={} user={} ip={} details={}",
        event_type,
        user_id.unwrap_or("anonymous"),
        ip.unwrap_or("unknown"),
        details
    );
}
```

---

## 10. สรุปสิ่งที่เรียนรู้

✅ OWASP API Top 10 awareness  
✅ SQL injection prevention ด้วย parameterized queries  
✅ XSS prevention ด้วย output encoding  
✅ CSRF protection ด้วย tokens  
✅ Sensitive data masking  
✅ Security headers implementation  
✅ cargo-audit สำหรับ dependency security  
✅ Environment variable secrets management  
✅ Automated security checklist  
✅ Security event logging  

---

*[← Part 048: Input Validation Advanced](../part_048/README.md) | [Part 050: Session Management →](../part_050/README.md)*
