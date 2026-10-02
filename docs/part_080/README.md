# Part 080: Security Hardening 🔒

## 🎯 เป้าหมายของ Part นี้

- Security hardening สำหรับ Rust API
- Dependency scanning และ secret rotation
- Network policies และ database permissions
- Penetration testing checklist
- Production security checklist

---

## 1. Dependency Security

### 1.1 cargo-audit

```bash
# ติดตั้ง cargo-audit
cargo install cargo-audit

# ตรวจสอบ vulnerabilities
cargo audit

# อัปเดต advisory database
cargo audit --update

# ใน CI/CD (fail if vulnerabilities found)
cargo audit --deny warnings
```

### 1.2 cargo-deny

```toml
# deny.toml
[advisories]
vulnerability = "deny"
unmaintained = "warn"
yanked = "deny"
notice = "warn"

[licenses]
unlicensed = "deny"
allow = ["MIT", "Apache-2.0", "BSD-2-Clause", "BSD-3-Clause"]
deny = ["GPL-3.0"]

[bans]
multiple-versions = "warn"
deny = [
    { name = "openssl" },  # ใช้ rustls แทน
]
```

```bash
cargo install cargo-deny
cargo deny check
```

---

## 2. Environment Variables และ Secrets

### 2.1 Secret Management

```rust
// src/config.rs
use std::env;

pub struct SecureConfig {
    pub database_url: String,
    pub jwt_secret: String,
    pub redis_url: String,
    pub smtp_password: String,
    pub stripe_secret_key: String,
}

impl SecureConfig {
    pub fn from_env() -> Result<Self, String> {
        Ok(Self {
            database_url: require_env("DATABASE_URL")?,
            jwt_secret: require_env_with_min_length("JWT_SECRET", 32)?,
            redis_url: require_env("REDIS_URL")?,
            smtp_password: require_env("SMTP_PASSWORD")?,
            stripe_secret_key: require_env("STRIPE_SECRET_KEY")?,
        })
    }
}

fn require_env(key: &str) -> Result<String, String> {
    env::var(key).map_err(|_| format!("Missing required env var: {}", key))
}

fn require_env_with_min_length(key: &str, min_len: usize) -> Result<String, String> {
    let value = require_env(key)?;
    if value.len() < min_len {
        return Err(format!("{} must be at least {} characters", key, min_len));
    }
    Ok(value)
}
```

### 2.2 Secret Rotation

```rust
use std::sync::Arc;
use tokio::sync::RwLock;
use chrono::{DateTime, Utc};

struct RotatingSecret {
    current: String,
    previous: Option<String>,
    rotated_at: DateTime<Utc>,
}

pub struct SecretManager {
    jwt_secret: Arc<RwLock<RotatingSecret>>,
}

impl SecretManager {
    pub fn new(initial_secret: String) -> Self {
        Self {
            jwt_secret: Arc::new(RwLock::new(RotatingSecret {
                current: initial_secret,
                previous: None,
                rotated_at: Utc::now(),
            })),
        }
    }

    pub async fn rotate_jwt_secret(&self, new_secret: String) {
        let mut secret = self.jwt_secret.write().await;
        secret.previous = Some(secret.current.clone());
        secret.current = new_secret;
        secret.rotated_at = Utc::now();
        log::info!("JWT secret rotated at {}", secret.rotated_at);
    }

    pub async fn get_current_secret(&self) -> String {
        self.jwt_secret.read().await.current.clone()
    }

    pub async fn get_previous_secret(&self) -> Option<String> {
        self.jwt_secret.read().await.previous.clone()
    }

    // Validate token against both current and previous secret
    pub async fn validate_token(&self, token: &str) -> bool {
        let secret = self.jwt_secret.read().await;
        if validate_jwt(token, &secret.current) {
            return true;
        }
        if let Some(prev) = &secret.previous {
            return validate_jwt(token, prev);
        }
        false
    }
}

fn validate_jwt(_token: &str, _secret: &str) -> bool {
    // JWT validation logic
    true
}
```

---

## 3. SQL Injection Prevention

```rust
// ✅ SAFE: Parameterized queries
pub async fn find_user_by_email(pool: &PgPool, email: &str) -> Result<Option<User>, sqlx::Error> {
    sqlx::query_as!(
        User,
        "SELECT * FROM users WHERE email = $1",
        email  // ✅ parameterized - safe
    )
    .fetch_optional(pool)
    .await
}

// ❌ UNSAFE: String interpolation (never do this)
// let query = format!("SELECT * FROM users WHERE email = '{}'", email);

// ✅ SAFE: Dynamic column names with whitelist
pub async fn sort_users(pool: &PgPool, sort_by: &str, order: &str) -> Result<Vec<User>, sqlx::Error> {
    // Whitelist allowed columns
    let column = match sort_by {
        "name" => "name",
        "email" => "email",
        "created_at" => "created_at",
        _ => "created_at",  // default
    };

    let order_dir = if order.to_lowercase() == "asc" { "ASC" } else { "DESC" };

    // Build query with validated column names (not user input)
    let query = format!(
        "SELECT * FROM users ORDER BY {} {}",
        column, order_dir
    );

    sqlx::query_as::<_, User>(&query)
        .fetch_all(pool)
        .await
}
```

---

## 4. XSS Prevention

```rust
use actix_web::{post, web, HttpResponse};
use serde::Deserialize;

#[derive(Deserialize)]
struct CommentRequest {
    content: String,
}

fn sanitize_html(input: &str) -> String {
    // Escape HTML special characters
    input
        .replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;")
        .replace('"', "&quot;")
        .replace('\'', "&#x27;")
}

fn strip_html_tags(input: &str) -> String {
    // Simple HTML tag stripper (use ammonia crate for production)
    let mut result = String::new();
    let mut in_tag = false;
    for ch in input.chars() {
        match ch {
            '<' => in_tag = true,
            '>' => in_tag = false,
            _ if !in_tag => result.push(ch),
            _ => {}
        }
    }
    result
}

#[post("/comments")]
async fn create_comment(body: web::Json<CommentRequest>) -> HttpResponse {
    let sanitized = sanitize_html(&body.content);
    HttpResponse::Created().json(serde_json::json!({
        "content": sanitized
    }))
}
```

---

## 5. Security Headers Middleware

```rust
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error,
};
use futures::future::{ok, LocalBoxFuture, Ready};

pub struct SecurityHeaders;

impl<S, B> Transform<S, ServiceRequest> for SecurityHeaders
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = SecurityHeadersMiddleware<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ok(SecurityHeadersMiddleware { service })
    }
}

pub struct SecurityHeadersMiddleware<S> {
    service: S,
}

impl<S, B> Service<ServiceRequest> for SecurityHeadersMiddleware<S>
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
        let fut = self.service.call(req);
        Box::pin(async move {
            let mut res = fut.await?;
            let headers = res.headers_mut();

            headers.insert(
                actix_web::http::header::HeaderName::from_static("x-content-type-options"),
                actix_web::http::header::HeaderValue::from_static("nosniff"),
            );
            headers.insert(
                actix_web::http::header::HeaderName::from_static("x-frame-options"),
                actix_web::http::header::HeaderValue::from_static("DENY"),
            );
            headers.insert(
                actix_web::http::header::HeaderName::from_static("x-xss-protection"),
                actix_web::http::header::HeaderValue::from_static("1; mode=block"),
            );
            headers.insert(
                actix_web::http::header::HeaderName::from_static("strict-transport-security"),
                actix_web::http::header::HeaderValue::from_static(
                    "max-age=31536000; includeSubDomains; preload"
                ),
            );
            headers.insert(
                actix_web::http::header::HeaderName::from_static("referrer-policy"),
                actix_web::http::header::HeaderValue::from_static("strict-origin-when-cross-origin"),
            );
            headers.insert(
                actix_web::http::header::HeaderName::from_static("content-security-policy"),
                actix_web::http::header::HeaderValue::from_static(
                    "default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:; font-src 'self'"
                ),
            );
            headers.insert(
                actix_web::http::header::HeaderName::from_static("permissions-policy"),
                actix_web::http::header::HeaderValue::from_static(
                    "geolocation=(), microphone=(), camera=()"
                ),
            );

            Ok(res)
        })
    }
}
```

---

## 6. Database Security

```sql
-- สร้าง database user ที่มีสิทธิ์น้อยที่สุด
CREATE USER api_user WITH PASSWORD 'strong_password_here';

-- ให้สิทธิ์เฉพาะที่จำเป็น
GRANT CONNECT ON DATABASE myapp TO api_user;
GRANT USAGE ON SCHEMA public TO api_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO api_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO api_user;

-- ไม่ให้สิทธิ์ DROP, CREATE, TRUNCATE
-- ไม่ให้ SUPERUSER

-- Row Level Security (RLS)
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

CREATE POLICY user_posts ON posts
    FOR ALL TO api_user
    USING (user_id = current_setting('app.current_user_id')::uuid);

-- Set user ID in session
-- ใน Rust: sqlx::query!("SELECT set_config('app.current_user_id', $1, true)", user_id.to_string())
```

---

## 7. Rate Limiting และ DDoS Protection

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::time::{Duration, Instant};

struct RateLimiter {
    limits: Mutex<HashMap<String, Vec<Instant>>>,
    max_requests: usize,
    window: Duration,
}

impl RateLimiter {
    pub fn new(max_requests: usize, window_seconds: u64) -> Arc<Self> {
        Arc::new(Self {
            limits: Mutex::new(HashMap::new()),
            max_requests,
            window: Duration::from_secs(window_seconds),
        })
    }

    pub fn check(&self, key: &str) -> bool {
        let mut limits = self.limits.lock().unwrap();
        let now = Instant::now();
        let requests = limits.entry(key.to_string()).or_insert_with(Vec::new);

        // Remove expired entries
        requests.retain(|t| now.duration_since(*t) < self.window);

        if requests.len() >= self.max_requests {
            return false; // Rate limited
        }

        requests.push(now);
        true
    }
}
```

---

## 8. Security Audit Checklist

```
## API Security Checklist

### Authentication & Authorization
- [ ] JWT tokens have short expiry (15 min access, 7 day refresh)
- [ ] Passwords hashed with bcrypt/argon2 (cost >= 12)
- [ ] No sensitive data in JWT payload
- [ ] Role-based access control implemented
- [ ] API keys hashed in database

### Input Validation
- [ ] All user inputs validated and sanitized
- [ ] File upload types and sizes validated
- [ ] SQL parameterized queries used everywhere
- [ ] Request size limits configured
- [ ] Rate limiting on all public endpoints

### Transport Security
- [ ] HTTPS enforced (HTTP redirects to HTTPS)
- [ ] TLS 1.2+ only
- [ ] HSTS header with long max-age
- [ ] Strong cipher suites

### Headers
- [ ] X-Content-Type-Options: nosniff
- [ ] X-Frame-Options: DENY
- [ ] Content-Security-Policy configured
- [ ] Referrer-Policy set
- [ ] Permissions-Policy configured

### Database
- [ ] Principle of least privilege for DB user
- [ ] Row Level Security where appropriate
- [ ] No raw SQL string interpolation
- [ ] Backups encrypted
- [ ] Connection uses TLS

### Dependencies
- [ ] cargo audit passes (no vulnerabilities)
- [ ] Dependencies up to date
- [ ] cargo-deny configured
- [ ] No unnecessary features enabled

### Logging & Monitoring
- [ ] Sensitive data not logged (passwords, tokens, PII)
- [ ] Failed auth attempts logged
- [ ] Anomaly detection alerts configured
- [ ] Error messages don't expose internals

### Infrastructure
- [ ] Secrets in environment variables (not in code)
- [ ] .env files not committed to git
- [ ] Docker image runs as non-root
- [ ] Network policies restrict inter-service communication
```

---

## 9. สรุปสิ่งที่เรียนรู้

✅ Dependency vulnerability scanning (cargo-audit, cargo-deny)  
✅ Secret management และ rotation  
✅ SQL injection prevention (parameterized queries)  
✅ XSS prevention (sanitization)  
✅ Security headers middleware  
✅ Database least-privilege และ RLS  
✅ Rate limiting สำหรับ DDoS protection  
✅ Security audit checklist  

---

*[← Part 079: Testing Strategies](../part_079/README.md) | [Part 081: Project: Blog API →](../part_081/README.md)*
