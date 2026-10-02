# Part 042: Password Hashing with Bcrypt 🔒

## 🎯 เป้าหมายของ Part นี้

- ทำความเข้าใจ bcrypt และการ hash password
- ใช้ `bcrypt` crate ใน Rust
- เปรียบเทียบ bcrypt กับ argon2
- ออกแบบ password policy
- สร้าง password reset flow
- Timing-safe comparison

---

## 1. ทำไมต้อง Hash Password?

การเก็บ password แบบ plain text เป็นความเสี่ยงที่ใหญ่มาก หาก database ถูก breach ผู้โจมตีจะได้ password ทุก account ทันที

**หลักการ:**
- เก็บเฉพาะ hash ของ password ไม่ใช่ตัว password จริง
- ใช้ algorithm ที่ออกแบบมาสำหรับ password (bcrypt, argon2, scrypt)
- ไม่ใช้ MD5, SHA1, SHA256 สำหรับ password เพราะเร็วเกินไป

---

## 2. Setup

### 2.1 Cargo.toml

```toml
[package]
name = "password-hashing"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
bcrypt = "0.15"
argon2 = "0.5"
rand = "0.8"
hex = "0.4"
thiserror = "1"
log = "0.4"
env_logger = "0.10"
dotenv = "0.15"
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
validator = { version = "0.16", features = ["derive"] }
regex = "1"
```

---

## 3. Password Hashing ด้วย Bcrypt

### 3.1 โครงสร้างพื้นฐาน

```rust
// src/password.rs
use bcrypt::{hash, verify, DEFAULT_COST, BcryptError};
use thiserror::Error;

#[derive(Debug, Error)]
pub enum PasswordError {
    #[error("Failed to hash password: {0}")]
    HashError(#[from] BcryptError),
    
    #[error("Password does not meet requirements: {0}")]
    PolicyViolation(String),
    
    #[error("Invalid password")]
    InvalidPassword,
}

/// Hash password ด้วย bcrypt
/// cost_factor: 4-31 (default 12, higher = slower but more secure)
pub fn hash_password(password: &str, cost: u32) -> Result<String, PasswordError> {
    let hashed = hash(password, cost)?;
    Ok(hashed)
}

/// Verify password กับ hash ที่เก็บไว้
pub fn verify_password(password: &str, hash: &str) -> Result<bool, PasswordError> {
    let result = verify(password, hash)?;
    Ok(result)
}

/// Timing-safe password comparison
/// bcrypt::verify ทำ timing-safe comparison อยู่แล้ว
pub fn constant_time_verify(password: &str, hash: &str) -> bool {
    // bcrypt::verify ใช้ constant-time comparison ภายใน
    verify(password, hash).unwrap_or(false)
}
```

### 3.2 Cost Factor Selection

```rust
// src/cost_benchmark.rs
use bcrypt::hash;
use std::time::Instant;

/// เลือก cost factor ที่เหมาะสม
/// เป้าหมาย: ~100-300ms per hash
pub fn benchmark_cost() {
    println!("Benchmarking bcrypt cost factors:");
    println!("{:<6} {:<12}", "Cost", "Time (ms)");
    println!("{}", "-".repeat(20));
    
    for cost in 8..=14u32 {
        let start = Instant::now();
        let _ = hash("test_password_123", cost).unwrap();
        let elapsed = start.elapsed().as_millis();
        
        let recommendation = if elapsed < 50 {
            " (too fast)"
        } else if elapsed <= 300 {
            " ✓ recommended"
        } else {
            " (too slow for API)"
        };
        
        println!("{:<6} {:<12}{}", cost, elapsed, recommendation);
    }
}

/// สำหรับ production ใช้ cost ที่ทำให้ใช้เวลา ~200ms
pub const PRODUCTION_COST: u32 = 12;
/// สำหรับ test ใช้ cost ต่ำเพื่อให้ test เร็ว
pub const TEST_COST: u32 = 4;

pub fn get_hash_cost() -> u32 {
    if cfg!(test) {
        TEST_COST
    } else {
        std::env::var("BCRYPT_COST")
            .ok()
            .and_then(|v| v.parse().ok())
            .unwrap_or(PRODUCTION_COST)
    }
}
```

---

## 4. Argon2 เป็น Alternative

```rust
// src/argon2_hash.rs
use argon2::{
    password_hash::{
        rand_core::OsRng,
        PasswordHash, PasswordHasher, PasswordVerifier, SaltString,
    },
    Argon2,
};

#[derive(Debug, thiserror::Error)]
pub enum Argon2Error {
    #[error("Hash error: {0}")]
    HashError(String),
    #[error("Verify error: {0}")]
    VerifyError(String),
}

/// Hash password ด้วย Argon2id (แนะนำสำหรับ new projects)
pub fn hash_password_argon2(password: &str) -> Result<String, Argon2Error> {
    let salt = SaltString::generate(&mut OsRng);
    let argon2 = Argon2::default();
    
    let hash = argon2
        .hash_password(password.as_bytes(), &salt)
        .map_err(|e| Argon2Error::HashError(e.to_string()))?;
    
    Ok(hash.to_string())
}

/// Verify password ที่ hash ด้วย Argon2
pub fn verify_password_argon2(password: &str, hash_str: &str) -> Result<bool, Argon2Error> {
    let parsed_hash = PasswordHash::new(hash_str)
        .map_err(|e| Argon2Error::VerifyError(e.to_string()))?;
    
    let argon2 = Argon2::default();
    
    Ok(argon2
        .verify_password(password.as_bytes(), &parsed_hash)
        .is_ok())
}

/// Argon2 parameters แบบ custom
pub fn hash_with_custom_params(password: &str) -> Result<String, Argon2Error> {
    use argon2::{Algorithm, Params, Version};
    
    let params = Params::new(
        65536, // memory cost (64 MB)
        3,     // iterations
        4,     // parallelism
        None,  // output length (default)
    ).map_err(|e| Argon2Error::HashError(e.to_string()))?;
    
    let argon2 = Argon2::new(Algorithm::Argon2id, Version::V0x13, params);
    let salt = SaltString::generate(&mut OsRng);
    
    let hash = argon2
        .hash_password(password.as_bytes(), &salt)
        .map_err(|e| Argon2Error::HashError(e.to_string()))?;
    
    Ok(hash.to_string())
}
```

---

## 5. Password Policy

```rust
// src/policy.rs
use regex::Regex;
use validator::Validate;

#[derive(Debug)]
pub struct PasswordPolicy {
    pub min_length: usize,
    pub max_length: usize,
    pub require_uppercase: bool,
    pub require_lowercase: bool,
    pub require_digit: bool,
    pub require_special: bool,
    pub min_unique_chars: usize,
    pub forbidden_patterns: Vec<String>,
}

impl Default for PasswordPolicy {
    fn default() -> Self {
        Self {
            min_length: 8,
            max_length: 128,
            require_uppercase: true,
            require_lowercase: true,
            require_digit: true,
            require_special: false,
            min_unique_chars: 4,
            forbidden_patterns: vec![
                "password".to_string(),
                "123456".to_string(),
                "qwerty".to_string(),
            ],
        }
    }
}

#[derive(Debug)]
pub struct PolicyViolation {
    pub field: String,
    pub message: String,
}

impl PasswordPolicy {
    pub fn validate(&self, password: &str) -> Vec<PolicyViolation> {
        let mut violations = Vec::new();
        
        // ตรวจสอบความยาว
        if password.len() < self.min_length {
            violations.push(PolicyViolation {
                field: "password".to_string(),
                message: format!("Password must be at least {} characters", self.min_length),
            });
        }
        
        if password.len() > self.max_length {
            violations.push(PolicyViolation {
                field: "password".to_string(),
                message: format!("Password must not exceed {} characters", self.max_length),
            });
        }
        
        // ตรวจสอบ uppercase
        if self.require_uppercase && !password.chars().any(|c| c.is_uppercase()) {
            violations.push(PolicyViolation {
                field: "password".to_string(),
                message: "Password must contain at least one uppercase letter".to_string(),
            });
        }
        
        // ตรวจสอบ lowercase
        if self.require_lowercase && !password.chars().any(|c| c.is_lowercase()) {
            violations.push(PolicyViolation {
                field: "password".to_string(),
                message: "Password must contain at least one lowercase letter".to_string(),
            });
        }
        
        // ตรวจสอบ digit
        if self.require_digit && !password.chars().any(|c| c.is_ascii_digit()) {
            violations.push(PolicyViolation {
                field: "password".to_string(),
                message: "Password must contain at least one digit".to_string(),
            });
        }
        
        // ตรวจสอบ special character
        if self.require_special {
            let special_chars = "!@#$%^&*()_+-=[]{}|;':\",./<>?";
            if !password.chars().any(|c| special_chars.contains(c)) {
                violations.push(PolicyViolation {
                    field: "password".to_string(),
                    message: "Password must contain at least one special character".to_string(),
                });
            }
        }
        
        // ตรวจสอบ unique characters
        let unique_count = password.chars().collect::<std::collections::HashSet<_>>().len();
        if unique_count < self.min_unique_chars {
            violations.push(PolicyViolation {
                field: "password".to_string(),
                message: format!("Password must contain at least {} unique characters", self.min_unique_chars),
            });
        }
        
        // ตรวจสอบ forbidden patterns
        let lower_password = password.to_lowercase();
        for pattern in &self.forbidden_patterns {
            if lower_password.contains(pattern.as_str()) {
                violations.push(PolicyViolation {
                    field: "password".to_string(),
                    message: format!("Password contains forbidden pattern: {}", pattern),
                });
            }
        }
        
        violations
    }
    
    pub fn is_valid(&self, password: &str) -> bool {
        self.validate(password).is_empty()
    }
    
    pub fn score(&self, password: &str) -> u8 {
        let mut score: u8 = 0;
        
        // ความยาว
        score += match password.len() {
            0..=7 => 0,
            8..=11 => 10,
            12..=15 => 20,
            16..=19 => 25,
            _ => 30,
        };
        
        // ความหลากหลาย
        if password.chars().any(|c| c.is_uppercase()) { score += 10; }
        if password.chars().any(|c| c.is_lowercase()) { score += 10; }
        if password.chars().any(|c| c.is_ascii_digit()) { score += 10; }
        
        let special_chars = "!@#$%^&*()_+-=[]{}|;':\",./<>?";
        if password.chars().any(|c| special_chars.contains(c)) { score += 20; }
        
        // Unique characters bonus
        let unique_ratio = password.chars()
            .collect::<std::collections::HashSet<_>>().len() as f32 
            / password.len() as f32;
        score += (unique_ratio * 20.0) as u8;
        
        score.min(100)
    }
}

/// Password strength enum
#[derive(Debug, PartialEq)]
pub enum PasswordStrength {
    Weak,
    Fair,
    Good,
    Strong,
    VeryStrong,
}

impl PasswordStrength {
    pub fn from_score(score: u8) -> Self {
        match score {
            0..=20 => Self::Weak,
            21..=40 => Self::Fair,
            41..=60 => Self::Good,
            61..=80 => Self::Strong,
            _ => Self::VeryStrong,
        }
    }
    
    pub fn as_str(&self) -> &str {
        match self {
            Self::Weak => "weak",
            Self::Fair => "fair",
            Self::Good => "good",
            Self::Strong => "strong",
            Self::VeryStrong => "very_strong",
        }
    }
}
```

---

## 6. Password Reset Flow

```rust
// src/reset.rs
use chrono::{DateTime, Duration, Utc};
use rand::{thread_rng, Rng};
use hex;
use sha2::{Sha256, Digest};

/// Password reset token
#[derive(Debug, Clone)]
pub struct ResetToken {
    pub token: String,         // token ที่ส่งให้ user (plain)
    pub token_hash: String,    // hash ที่เก็บใน database
    pub expires_at: DateTime<Utc>,
    pub user_id: String,
}

impl ResetToken {
    /// สร้าง reset token ใหม่
    pub fn new(user_id: &str, expiry_minutes: i64) -> Self {
        // สร้าง random bytes
        let mut rng = thread_rng();
        let bytes: Vec<u8> = (0..32).map(|_| rng.gen()).collect();
        let token = hex::encode(&bytes);
        
        // Hash token ก่อนเก็บ database
        let mut hasher = Sha256::new();
        hasher.update(token.as_bytes());
        let token_hash = hex::encode(hasher.finalize());
        
        Self {
            token,
            token_hash,
            expires_at: Utc::now() + Duration::minutes(expiry_minutes),
            user_id: user_id.to_string(),
        }
    }
    
    /// Verify token ที่ได้รับจาก user
    pub fn verify(token: &str, stored_hash: &str) -> bool {
        let mut hasher = Sha256::new();
        hasher.update(token.as_bytes());
        let computed_hash = hex::encode(hasher.finalize());
        
        // Constant-time comparison
        use subtle::ConstantTimeEq;
        computed_hash.as_bytes().ct_eq(stored_hash.as_bytes()).into()
    }
    
    pub fn is_expired(&self) -> bool {
        Utc::now() > self.expires_at
    }
}

/// Password reset request
#[derive(Debug, serde::Deserialize)]
pub struct ResetRequest {
    pub email: String,
}

/// Password reset confirmation
#[derive(Debug, serde::Deserialize)]
pub struct ResetConfirmation {
    pub token: String,
    pub new_password: String,
    pub confirm_password: String,
}
```

---

## 7. Complete Auth Service

### 7.1 Database Schema

```sql
-- migrations/001_create_users.sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    is_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE password_reset_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    used_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_reset_tokens_user_id ON password_reset_tokens(user_id);
CREATE INDEX idx_reset_tokens_expires ON password_reset_tokens(expires_at);
```

### 7.2 Models

```rust
// src/models.rs
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use validator::Validate;

#[derive(Debug, Serialize, Deserialize, sqlx::FromRow)]
pub struct User {
    pub id: Uuid,
    pub email: String,
    pub password_hash: String,
    pub is_verified: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Validate)]
pub struct RegisterRequest {
    #[validate(email(message = "Invalid email format"))]
    pub email: String,
    
    #[validate(length(min = 8, message = "Password must be at least 8 characters"))]
    pub password: String,
    
    #[validate(must_match(other = "password", message = "Passwords do not match"))]
    pub confirm_password: String,
}

#[derive(Debug, Deserialize, Validate)]
pub struct LoginRequest {
    #[validate(email)]
    pub email: String,
    pub password: String,
}

#[derive(Debug, Serialize)]
pub struct AuthResponse {
    pub user_id: String,
    pub email: String,
    pub message: String,
}

#[derive(Debug, Serialize)]
pub struct PasswordCheckResponse {
    pub score: u8,
    pub strength: String,
    pub violations: Vec<String>,
    pub is_valid: bool,
}
```

### 7.3 Handlers

```rust
// src/handlers.rs
use actix_web::{web, HttpResponse, Result};
use bcrypt::DEFAULT_COST;
use sqlx::PgPool;

use crate::{
    models::{RegisterRequest, LoginRequest, AuthResponse, PasswordCheckResponse},
    password::{hash_password, constant_time_verify},
    policy::{PasswordPolicy, PasswordStrength},
    reset::ResetToken,
};

pub async fn register(
    pool: web::Data<PgPool>,
    req: web::Json<RegisterRequest>,
) -> Result<HttpResponse> {
    // Validate input
    use validator::Validate;
    if let Err(errors) = req.validate() {
        return Ok(HttpResponse::BadRequest().json(errors));
    }
    
    // ตรวจสอบ password policy
    let policy = PasswordPolicy::default();
    let violations = policy.validate(&req.password);
    
    if !violations.is_empty() {
        let messages: Vec<String> = violations.iter().map(|v| v.message.clone()).collect();
        return Ok(HttpResponse::BadRequest().json(serde_json::json!({
            "error": "Password policy violation",
            "violations": messages
        })));
    }
    
    // ตรวจสอบว่า email ซ้ำหรือไม่
    let existing = sqlx::query!("SELECT id FROM users WHERE email = $1", req.email)
        .fetch_optional(pool.get_ref())
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    if existing.is_some() {
        return Ok(HttpResponse::Conflict().json(serde_json::json!({
            "error": "Email already registered"
        })));
    }
    
    // Hash password
    let cost = crate::cost_benchmark::get_hash_cost();
    let password_hash = hash_password(&req.password, cost)
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    // บันทึกลง database
    let user = sqlx::query_as!(
        crate::models::User,
        r#"INSERT INTO users (email, password_hash) VALUES ($1, $2)
           RETURNING id, email, password_hash, is_verified, created_at, updated_at"#,
        req.email,
        password_hash
    )
    .fetch_one(pool.get_ref())
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    Ok(HttpResponse::Created().json(AuthResponse {
        user_id: user.id.to_string(),
        email: user.email,
        message: "Registration successful".to_string(),
    }))
}

pub async fn login(
    pool: web::Data<PgPool>,
    req: web::Json<LoginRequest>,
) -> Result<HttpResponse> {
    // ค้นหา user
    let user = sqlx::query_as!(
        crate::models::User,
        "SELECT * FROM users WHERE email = $1",
        req.email
    )
    .fetch_optional(pool.get_ref())
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    // ตรวจสอบ password (constant time เพื่อป้องกัน timing attack)
    let is_valid = match &user {
        Some(u) => constant_time_verify(&req.password, &u.password_hash),
        // ยังทำ fake hash เพื่อป้องกัน timing attack
        None => {
            let _ = constant_time_verify(&req.password, "$2b$12$invalidhashfortimingatk");
            false
        }
    };
    
    if !is_valid {
        return Ok(HttpResponse::Unauthorized().json(serde_json::json!({
            "error": "Invalid credentials"
        })));
    }
    
    let user = user.unwrap();
    
    Ok(HttpResponse::Ok().json(AuthResponse {
        user_id: user.id.to_string(),
        email: user.email,
        message: "Login successful".to_string(),
    }))
}

pub async fn check_password_strength(
    req: web::Json<serde_json::Value>,
) -> Result<HttpResponse> {
    let password = req.get("password")
        .and_then(|v| v.as_str())
        .unwrap_or("");
    
    let policy = PasswordPolicy::default();
    let violations = policy.validate(password);
    let score = policy.score(password);
    let strength = PasswordStrength::from_score(score);
    
    Ok(HttpResponse::Ok().json(PasswordCheckResponse {
        score,
        strength: strength.as_str().to_string(),
        violations: violations.iter().map(|v| v.message.clone()).collect(),
        is_valid: violations.is_empty(),
    }))
}

pub async fn request_password_reset(
    pool: web::Data<PgPool>,
    req: web::Json<crate::reset::ResetRequest>,
) -> Result<HttpResponse> {
    // ค้นหา user (ไม่บอก user ว่า email มีอยู่ไหม เพื่อป้องกัน enumeration)
    let user = sqlx::query!(
        "SELECT id FROM users WHERE email = $1",
        req.email
    )
    .fetch_optional(pool.get_ref())
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    if let Some(user) = user {
        // สร้าง reset token
        let reset_token = ResetToken::new(&user.id.to_string(), 60); // 60 minutes expiry
        
        // เก็บ hash ใน database
        sqlx::query!(
            r#"INSERT INTO password_reset_tokens (user_id, token_hash, expires_at)
               VALUES ($1, $2, $3)"#,
            user.id,
            reset_token.token_hash,
            reset_token.expires_at
        )
        .execute(pool.get_ref())
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
        
        // TODO: ส่ง email พร้อม reset_token.token
        log::info!("Password reset token for {}: {}", req.email, reset_token.token);
    }
    
    // Response เหมือนกันไม่ว่า email จะมีอยู่หรือไม่
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "message": "If the email exists, a reset link has been sent"
    })))
}

pub async fn confirm_password_reset(
    pool: web::Data<PgPool>,
    req: web::Json<crate::reset::ResetConfirmation>,
) -> Result<HttpResponse> {
    // ตรวจสอบ password match
    if req.new_password != req.confirm_password {
        return Ok(HttpResponse::BadRequest().json(serde_json::json!({
            "error": "Passwords do not match"
        })));
    }
    
    // ตรวจสอบ policy
    let policy = PasswordPolicy::default();
    if !policy.is_valid(&req.new_password) {
        return Ok(HttpResponse::BadRequest().json(serde_json::json!({
            "error": "Password does not meet requirements"
        })));
    }
    
    // Hash token และค้นหาใน database
    use sha2::{Sha256, Digest};
    let mut hasher = Sha256::new();
    hasher.update(req.token.as_bytes());
    let token_hash = hex::encode(hasher.finalize());
    
    let reset_record = sqlx::query!(
        r#"SELECT id, user_id, expires_at, used_at
           FROM password_reset_tokens
           WHERE token_hash = $1"#,
        token_hash
    )
    .fetch_optional(pool.get_ref())
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    let record = match reset_record {
        Some(r) => r,
        None => {
            return Ok(HttpResponse::BadRequest().json(serde_json::json!({
                "error": "Invalid or expired reset token"
            })));
        }
    };
    
    // ตรวจสอบว่าหมดอายุหรือใช้แล้ว
    if chrono::Utc::now() > record.expires_at {
        return Ok(HttpResponse::BadRequest().json(serde_json::json!({
            "error": "Reset token has expired"
        })));
    }
    
    if record.used_at.is_some() {
        return Ok(HttpResponse::BadRequest().json(serde_json::json!({
            "error": "Reset token has already been used"
        })));
    }
    
    // Hash password ใหม่
    let cost = crate::cost_benchmark::get_hash_cost();
    let new_hash = hash_password(&req.new_password, cost)
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    // Update password และ mark token as used
    let mut tx = pool.begin()
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    sqlx::query!(
        "UPDATE users SET password_hash = $1, updated_at = NOW() WHERE id = $2",
        new_hash,
        record.user_id
    )
    .execute(&mut *tx)
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    sqlx::query!(
        "UPDATE password_reset_tokens SET used_at = NOW() WHERE id = $1",
        record.id
    )
    .execute(&mut *tx)
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    tx.commit()
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "message": "Password reset successful"
    })))
}
```

### 7.4 Main Application

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware};
use sqlx::postgres::PgPoolOptions;
use dotenv::dotenv;

mod cost_benchmark;
mod password;
mod argon2_hash;
mod policy;
mod reset;
mod models;
mod handlers;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv().ok();
    env_logger::init();
    
    // Run benchmark ตอน startup
    cost_benchmark::benchmark_cost();
    
    let database_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await
        .expect("Failed to connect to database");
    
    // Run migrations
    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .expect("Failed to run migrations");
    
    log::info!("Starting server on 0.0.0.0:8080");
    
    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(pool.clone()))
            .wrap(middleware::Logger::default())
            .service(
                web::scope("/auth")
                    .route("/register", web::post().to(handlers::register))
                    .route("/login", web::post().to(handlers::login))
                    .route("/password/check", web::post().to(handlers::check_password_strength))
                    .route("/password/reset", web::post().to(handlers::request_password_reset))
                    .route("/password/reset/confirm", web::post().to(handlers::confirm_password_reset))
            )
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

---

## 8. การทดสอบ

```rust
// src/tests.rs
#[cfg(test)]
mod tests {
    use crate::password::*;
    use crate::policy::*;
    use crate::reset::*;
    use crate::cost_benchmark::TEST_COST;

    #[test]
    fn test_hash_and_verify() {
        let password = "MySecureP@ss123";
        let hash = hash_password(password, TEST_COST).unwrap();
        
        assert!(hash.starts_with("$2b$"));
        assert!(constant_time_verify(password, &hash));
        assert!(!constant_time_verify("wrong_password", &hash));
    }
    
    #[test]
    fn test_different_hashes_same_password() {
        let password = "TestPassword123";
        let hash1 = hash_password(password, TEST_COST).unwrap();
        let hash2 = hash_password(password, TEST_COST).unwrap();
        
        // bcrypt ใช้ random salt ทุกครั้ง ดังนั้น hash ต้องต่างกัน
        assert_ne!(hash1, hash2);
        
        // แต่ verify ต้องผ่านทั้งคู่
        assert!(constant_time_verify(password, &hash1));
        assert!(constant_time_verify(password, &hash2));
    }
    
    #[test]
    fn test_password_policy() {
        let policy = PasswordPolicy::default();
        
        // Password ที่ผ่าน
        assert!(policy.is_valid("MyPass123"));
        
        // Password ที่ไม่ผ่าน
        assert!(!policy.is_valid("short"));          // สั้นเกินไป
        assert!(!policy.is_valid("allowercase123")); // ไม่มี uppercase
        assert!(!policy.is_valid("ALLUPPERCASE1")); // ไม่มี lowercase
        assert!(!policy.is_valid("NoDigitsHere"));  // ไม่มี digit
    }
    
    #[test]
    fn test_password_score() {
        let policy = PasswordPolicy::default();
        
        let weak_score = policy.score("abc");
        let strong_score = policy.score("MyV3ryStr0ng!Pass#2024");
        
        assert!(weak_score < strong_score);
        
        let weak_strength = PasswordStrength::from_score(weak_score);
        assert_eq!(weak_strength, PasswordStrength::Weak);
    }
    
    #[test]
    fn test_reset_token() {
        let token = ResetToken::new("user-123", 60);
        
        // Token ต้องไม่ใช่ hash
        assert_ne!(token.token, token.token_hash);
        
        // Verify ต้องผ่านด้วย plain token
        assert!(ResetToken::verify(&token.token, &token.token_hash));
        
        // Verify ต้องไม่ผ่านด้วย hash
        assert!(!ResetToken::verify(&token.token_hash, &token.token_hash));
        
        // ไม่ควรหมดอายุทันที
        assert!(!token.is_expired());
    }
    
    #[tokio::test]
    async fn test_argon2() {
        use crate::argon2_hash::*;
        
        let password = "TestArgon2Pass123";
        let hash = hash_password_argon2(password).unwrap();
        
        assert!(verify_password_argon2(password, &hash).unwrap());
        assert!(!verify_password_argon2("wrong", &hash).unwrap());
    }
}
```

---

## 9. สรุปเปรียบเทียบ Bcrypt vs Argon2

| Feature | Bcrypt | Argon2 |
|---------|--------|--------|
| Year | 1999 | 2015 |
| Memory hardness | No | Yes |
| Parallelism | No | Yes |
| PHC winner | No | Yes (2015) |
| Rust support | ดี | ดีมาก |
| Recommendation | OK for existing | New projects |

### เมื่อไหร่ควรใช้อะไร:
- **Bcrypt**: ระบบเก่าที่ใช้อยู่, ทีมคุ้นเคย
- **Argon2**: ระบบใหม่, ต้องการ security สูงสุด
- **Scrypt**: ทางเลือกที่ดีเช่นกัน

---

## 10. สรุปสิ่งที่เรียนรู้

✅ Hash password ด้วย bcrypt  
✅ Cost factor selection และ benchmark  
✅ Argon2 เป็น modern alternative  
✅ Password policy validation  
✅ Password strength scoring  
✅ Timing-safe comparison  
✅ Password reset flow ที่ปลอดภัย  
✅ เก็บเฉพาะ hash ของ reset token  

---

*[← Part 041: JWT Authentication](../part_041/README.md) | [Part 043: OAuth2 Integration →](../part_043/README.md)*
