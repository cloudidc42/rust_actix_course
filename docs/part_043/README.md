# Part 043: OAuth2 Integration 🔑

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ OAuth2 flow
- ใช้ `oauth2` crate ใน Rust
- Integrate Google OAuth2
- Integrate GitHub OAuth2
- Handle callback และ token exchange
- Fetch user info จาก provider
- Link OAuth account กับ local account

---

## 1. OAuth2 Flow Overview

```
User → App → Provider (Google/GitHub)
                ↓ (Authorization)
User ← App ← Provider (Code)
       ↓
       Provider (Token Exchange)
       ↓
       Provider (User Info)
```

**OAuth2 Authorization Code Flow:**
1. User คลิก "Login with Google"
2. App redirect ไป Google พร้อม `client_id`, `redirect_uri`, `scope`
3. User อนุญาตใน Google
4. Google redirect กลับมาพร้อม `code`
5. App แลก `code` เป็น `access_token`
6. App ใช้ `access_token` ดึง user info
7. App สร้าง session

---

## 2. Setup

### 2.1 Cargo.toml

```toml
[package]
name = "oauth2-integration"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
actix-session = { version = "0.9", features = ["cookie-session"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
oauth2 = "4.4"
reqwest = { version = "0.11", features = ["json"] }
dotenv = "0.15"
log = "0.4"
env_logger = "0.10"
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "1"
anyhow = "1"
base64 = "0.21"
rand = "0.8"
```

### 2.2 Environment Variables

```env
# .env
DATABASE_URL=postgresql://user:password@localhost/myapp

# Google OAuth2
GOOGLE_CLIENT_ID=your-google-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your-google-client-secret
GOOGLE_REDIRECT_URI=http://localhost:8080/auth/google/callback

# GitHub OAuth2
GITHUB_CLIENT_ID=your-github-client-id
GITHUB_CLIENT_SECRET=your-github-client-secret
GITHUB_REDIRECT_URI=http://localhost:8080/auth/github/callback

# Session
SESSION_SECRET=your-very-long-secret-key-at-least-32-chars
APP_URL=http://localhost:8080
```

---

## 3. OAuth2 Configuration

```rust
// src/oauth_config.rs
use oauth2::{
    AuthUrl, ClientId, ClientSecret, RedirectUrl, TokenUrl,
    basic::BasicClient,
};

pub struct OAuthConfig {
    pub google_client: BasicClient,
    pub github_client: BasicClient,
}

impl OAuthConfig {
    pub fn from_env() -> anyhow::Result<Self> {
        let google_client = Self::build_google_client()?;
        let github_client = Self::build_github_client()?;
        
        Ok(Self {
            google_client,
            github_client,
        })
    }
    
    fn build_google_client() -> anyhow::Result<BasicClient> {
        let client_id = ClientId::new(
            std::env::var("GOOGLE_CLIENT_ID")
                .map_err(|_| anyhow::anyhow!("GOOGLE_CLIENT_ID not set"))?
        );
        
        let client_secret = ClientSecret::new(
            std::env::var("GOOGLE_CLIENT_SECRET")
                .map_err(|_| anyhow::anyhow!("GOOGLE_CLIENT_SECRET not set"))?
        );
        
        let auth_url = AuthUrl::new(
            "https://accounts.google.com/o/oauth2/v2/auth".to_string()
        )?;
        
        let token_url = TokenUrl::new(
            "https://oauth2.googleapis.com/token".to_string()
        )?;
        
        let redirect_url = RedirectUrl::new(
            std::env::var("GOOGLE_REDIRECT_URI")
                .unwrap_or_else(|_| "http://localhost:8080/auth/google/callback".to_string())
        )?;
        
        Ok(BasicClient::new(client_id, Some(client_secret), auth_url, Some(token_url))
            .set_redirect_uri(redirect_url))
    }
    
    fn build_github_client() -> anyhow::Result<BasicClient> {
        let client_id = ClientId::new(
            std::env::var("GITHUB_CLIENT_ID")
                .map_err(|_| anyhow::anyhow!("GITHUB_CLIENT_ID not set"))?
        );
        
        let client_secret = ClientSecret::new(
            std::env::var("GITHUB_CLIENT_SECRET")
                .map_err(|_| anyhow::anyhow!("GITHUB_CLIENT_SECRET not set"))?
        );
        
        let auth_url = AuthUrl::new(
            "https://github.com/login/oauth/authorize".to_string()
        )?;
        
        let token_url = TokenUrl::new(
            "https://github.com/login/oauth/access_token".to_string()
        )?;
        
        let redirect_url = RedirectUrl::new(
            std::env::var("GITHUB_REDIRECT_URI")
                .unwrap_or_else(|_| "http://localhost:8080/auth/github/callback".to_string())
        )?;
        
        Ok(BasicClient::new(client_id, Some(client_secret), auth_url, Some(token_url))
            .set_redirect_uri(redirect_url))
    }
}
```

---

## 4. Google OAuth2 Implementation

### 4.1 User Info Structure

```rust
// src/providers/google.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize, Serialize)]
pub struct GoogleUserInfo {
    pub id: String,
    pub email: String,
    pub verified_email: bool,
    pub name: String,
    pub given_name: Option<String>,
    pub family_name: Option<String>,
    pub picture: Option<String>,
    pub locale: Option<String>,
}

pub async fn fetch_google_user_info(
    access_token: &str,
) -> anyhow::Result<GoogleUserInfo> {
    let client = reqwest::Client::new();
    
    let user_info = client
        .get("https://www.googleapis.com/oauth2/v2/userinfo")
        .bearer_auth(access_token)
        .send()
        .await?
        .error_for_status()?
        .json::<GoogleUserInfo>()
        .await?;
    
    Ok(user_info)
}
```

### 4.2 Google OAuth Handlers

```rust
// src/handlers/google.rs
use actix_web::{web, HttpResponse, Result};
use actix_session::Session;
use oauth2::{
    CsrfToken, PkceCodeChallenge, PkceCodeVerifier, Scope,
    TokenResponse,
    reqwest::async_http_client,
};
use serde::Deserialize;
use sqlx::PgPool;

use crate::{
    oauth_config::OAuthConfig,
    providers::google::fetch_google_user_info,
    db::oauth_accounts,
};

/// เริ่ม Google OAuth flow
pub async fn google_login(
    session: Session,
    oauth: web::Data<OAuthConfig>,
) -> Result<HttpResponse> {
    // สร้าง PKCE challenge (เพิ่มความปลอดภัย)
    let (pkce_challenge, pkce_verifier) = PkceCodeChallenge::new_random_sha256();
    
    // สร้าง CSRF token
    let (auth_url, csrf_token) = oauth.google_client
        .authorize_url(CsrfToken::new_random)
        .add_scope(Scope::new("openid".to_string()))
        .add_scope(Scope::new("email".to_string()))
        .add_scope(Scope::new("profile".to_string()))
        .set_pkce_challenge(pkce_challenge)
        .url();
    
    // เก็บ PKCE verifier และ CSRF token ใน session
    session.insert("oauth_csrf_token", csrf_token.secret().clone())
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    session.insert("oauth_pkce_verifier", pkce_verifier.secret().clone())
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    session.insert("oauth_provider", "google")
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    Ok(HttpResponse::Found()
        .append_header(("Location", auth_url.to_string()))
        .finish())
}

#[derive(Debug, Deserialize)]
pub struct CallbackQuery {
    pub code: String,
    pub state: String,
    pub error: Option<String>,
}

/// Handle Google OAuth callback
pub async fn google_callback(
    session: Session,
    pool: web::Data<PgPool>,
    oauth: web::Data<OAuthConfig>,
    query: web::Query<CallbackQuery>,
) -> Result<HttpResponse> {
    // ตรวจสอบว่ามี error ไหม
    if let Some(error) = &query.error {
        log::warn!("OAuth error: {}", error);
        return Ok(HttpResponse::BadRequest().json(serde_json::json!({
            "error": format!("OAuth error: {}", error)
        })));
    }
    
    // ตรวจสอบ CSRF token
    let stored_csrf: Option<String> = session.get("oauth_csrf_token")
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    match stored_csrf {
        Some(csrf) if csrf == query.state => {},
        _ => {
            return Ok(HttpResponse::BadRequest().json(serde_json::json!({
                "error": "Invalid CSRF token"
            })));
        }
    }
    
    // ดึง PKCE verifier
    let pkce_verifier_secret: Option<String> = session.get("oauth_pkce_verifier")
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    let pkce_verifier = match pkce_verifier_secret {
        Some(secret) => PkceCodeVerifier::new(secret),
        None => {
            return Ok(HttpResponse::BadRequest().json(serde_json::json!({
                "error": "Missing PKCE verifier"
            })));
        }
    };
    
    // แลก code เป็น token
    let token_result = oauth.google_client
        .exchange_code(oauth2::AuthorizationCode::new(query.code.clone()))
        .set_pkce_verifier(pkce_verifier)
        .request_async(async_http_client)
        .await;
    
    let token = match token_result {
        Ok(t) => t,
        Err(e) => {
            log::error!("Token exchange error: {:?}", e);
            return Ok(HttpResponse::InternalServerError().json(serde_json::json!({
                "error": "Failed to exchange authorization code"
            })));
        }
    };
    
    // ดึง user info จาก Google
    let user_info = fetch_google_user_info(token.access_token().secret())
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    // หรือสร้าง user ถ้าไม่มี
    let user_id = oauth_accounts::find_or_create_user(
        pool.get_ref(),
        "google",
        &user_info.id,
        &user_info.email,
        user_info.name.as_str(),
        user_info.picture.as_deref(),
    )
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    // ล้าง OAuth session data
    session.remove("oauth_csrf_token");
    session.remove("oauth_pkce_verifier");
    session.remove("oauth_provider");
    
    // Set user session
    session.insert("user_id", user_id.to_string())
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    // Redirect ไป dashboard
    let app_url = std::env::var("APP_URL").unwrap_or_else(|_| "http://localhost:8080".to_string());
    Ok(HttpResponse::Found()
        .append_header(("Location", format!("{}/dashboard", app_url)))
        .finish())
}
```

---

## 5. GitHub OAuth2 Implementation

### 5.1 GitHub User Info

```rust
// src/providers/github.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize, Serialize)]
pub struct GitHubUserInfo {
    pub id: i64,
    pub login: String,
    pub name: Option<String>,
    pub email: Option<String>,
    pub avatar_url: Option<String>,
    pub bio: Option<String>,
    pub public_repos: i32,
    pub followers: i32,
}

#[derive(Debug, Deserialize)]
struct GitHubEmail {
    email: String,
    primary: bool,
    verified: bool,
}

pub async fn fetch_github_user_info(
    access_token: &str,
) -> anyhow::Result<GitHubUserInfo> {
    let client = reqwest::Client::new();
    
    // ดึง user profile
    let mut user_info = client
        .get("https://api.github.com/user")
        .bearer_auth(access_token)
        .header("User-Agent", "rust-actix-app/1.0")
        .send()
        .await?
        .error_for_status()?
        .json::<GitHubUserInfo>()
        .await?;
    
    // GitHub อาจไม่ return email ใน user profile ถ้า user ตั้งเป็น private
    if user_info.email.is_none() {
        let emails = client
            .get("https://api.github.com/user/emails")
            .bearer_auth(access_token)
            .header("User-Agent", "rust-actix-app/1.0")
            .send()
            .await?
            .error_for_status()?
            .json::<Vec<GitHubEmail>>()
            .await?;
        
        // ใช้ primary email
        user_info.email = emails.into_iter()
            .find(|e| e.primary && e.verified)
            .map(|e| e.email);
    }
    
    Ok(user_info)
}
```

### 5.2 GitHub Handlers

```rust
// src/handlers/github.rs
use actix_web::{web, HttpResponse, Result};
use actix_session::Session;
use oauth2::{CsrfToken, Scope, TokenResponse, reqwest::async_http_client};
use sqlx::PgPool;

use crate::{
    oauth_config::OAuthConfig,
    providers::github::fetch_github_user_info,
    db::oauth_accounts,
};

pub async fn github_login(
    session: Session,
    oauth: web::Data<OAuthConfig>,
) -> Result<HttpResponse> {
    let (auth_url, csrf_token) = oauth.github_client
        .authorize_url(CsrfToken::new_random)
        .add_scope(Scope::new("user:email".to_string()))
        .add_scope(Scope::new("read:user".to_string()))
        .url();
    
    session.insert("oauth_csrf_token", csrf_token.secret().clone())
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    session.insert("oauth_provider", "github")
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    Ok(HttpResponse::Found()
        .append_header(("Location", auth_url.to_string()))
        .finish())
}

pub async fn github_callback(
    session: Session,
    pool: web::Data<PgPool>,
    oauth: web::Data<OAuthConfig>,
    query: web::Query<crate::handlers::google::CallbackQuery>,
) -> Result<HttpResponse> {
    // ตรวจสอบ CSRF
    let stored_csrf: Option<String> = session.get("oauth_csrf_token")
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    match stored_csrf {
        Some(csrf) if csrf == query.state => {},
        _ => {
            return Ok(HttpResponse::BadRequest().json(serde_json::json!({
                "error": "Invalid state parameter"
            })));
        }
    }
    
    // แลก code เป็น token
    let token = oauth.github_client
        .exchange_code(oauth2::AuthorizationCode::new(query.code.clone()))
        .request_async(async_http_client)
        .await
        .map_err(|e| {
            log::error!("GitHub token exchange error: {:?}", e);
            actix_web::error::ErrorInternalServerError("Token exchange failed")
        })?;
    
    // ดึง user info
    let user_info = fetch_github_user_info(token.access_token().secret())
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    let email = user_info.email.unwrap_or_else(|| {
        format!("{}@users.noreply.github.com", user_info.login)
    });
    
    let name = user_info.name.unwrap_or_else(|| user_info.login.clone());
    
    // หรือสร้าง user
    let user_id = oauth_accounts::find_or_create_user(
        pool.get_ref(),
        "github",
        &user_info.id.to_string(),
        &email,
        &name,
        user_info.avatar_url.as_deref(),
    )
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    // ล้าง session oauth data
    session.remove("oauth_csrf_token");
    session.remove("oauth_provider");
    
    // Set user session
    session.insert("user_id", user_id.to_string())
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    let app_url = std::env::var("APP_URL").unwrap_or_else(|_| "http://localhost:8080".to_string());
    Ok(HttpResponse::Found()
        .append_header(("Location", format!("{}/dashboard", app_url)))
        .finish())
}
```

---

## 6. Database - Linking OAuth Accounts

### 6.1 Database Schema

```sql
-- migrations/001_oauth.sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    display_name VARCHAR(255),
    avatar_url TEXT,
    password_hash VARCHAR(255), -- NULL สำหรับ OAuth-only users
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE oauth_accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL,        -- 'google', 'github'
    provider_user_id VARCHAR(255) NOT NULL, -- ID จาก provider
    access_token TEXT,
    refresh_token TEXT,
    token_expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(provider, provider_user_id)
);

CREATE INDEX idx_oauth_user_id ON oauth_accounts(user_id);
CREATE INDEX idx_oauth_provider ON oauth_accounts(provider, provider_user_id);
```

### 6.2 OAuth Account Service

```rust
// src/db/oauth_accounts.rs
use sqlx::PgPool;
use uuid::Uuid;

/// ค้นหา user จาก OAuth provider หรือสร้างใหม่
pub async fn find_or_create_user(
    pool: &PgPool,
    provider: &str,
    provider_user_id: &str,
    email: &str,
    display_name: &str,
    avatar_url: Option<&str>,
) -> anyhow::Result<Uuid> {
    // ค้นหา OAuth account ที่มีอยู่
    let existing = sqlx::query!(
        r#"SELECT user_id FROM oauth_accounts
           WHERE provider = $1 AND provider_user_id = $2"#,
        provider,
        provider_user_id
    )
    .fetch_optional(pool)
    .await?;
    
    if let Some(record) = existing {
        // Update last login
        sqlx::query!(
            "UPDATE users SET updated_at = NOW() WHERE id = $1",
            record.user_id
        )
        .execute(pool)
        .await?;
        
        return Ok(record.user_id);
    }
    
    // ตรวจสอบว่า email นี้มี user อยู่แล้วไหม (link accounts)
    let user_by_email = sqlx::query!(
        "SELECT id FROM users WHERE email = $1",
        email
    )
    .fetch_optional(pool)
    .await?;
    
    let user_id = match user_by_email {
        Some(u) => {
            // Link OAuth account กับ user ที่มีอยู่
            log::info!("Linking {} OAuth to existing user {}", provider, u.id);
            u.id
        },
        None => {
            // สร้าง user ใหม่
            let new_user = sqlx::query!(
                r#"INSERT INTO users (email, display_name, avatar_url)
                   VALUES ($1, $2, $3)
                   RETURNING id"#,
                email,
                display_name,
                avatar_url
            )
            .fetch_one(pool)
            .await?;
            
            log::info!("Created new user {} via {}", new_user.id, provider);
            new_user.id
        }
    };
    
    // สร้าง OAuth account record
    sqlx::query!(
        r#"INSERT INTO oauth_accounts (user_id, provider, provider_user_id)
           VALUES ($1, $2, $3)
           ON CONFLICT (provider, provider_user_id)
           DO UPDATE SET updated_at = NOW()"#,
        user_id,
        provider,
        provider_user_id
    )
    .execute(pool)
    .await?;
    
    Ok(user_id)
}

/// ดึง OAuth providers ของ user
pub async fn get_user_providers(
    pool: &PgPool,
    user_id: Uuid,
) -> anyhow::Result<Vec<String>> {
    let providers = sqlx::query!(
        "SELECT provider FROM oauth_accounts WHERE user_id = $1",
        user_id
    )
    .fetch_all(pool)
    .await?
    .into_iter()
    .map(|r| r.provider)
    .collect();
    
    Ok(providers)
}

/// ยกเลิก link OAuth account
pub async fn unlink_provider(
    pool: &PgPool,
    user_id: Uuid,
    provider: &str,
) -> anyhow::Result<bool> {
    // ตรวจสอบว่า user ยังมีวิธี login อื่นหรือไม่
    let other_providers_count = sqlx::query!(
        r#"SELECT COUNT(*) as count FROM oauth_accounts
           WHERE user_id = $1 AND provider != $2"#,
        user_id,
        provider
    )
    .fetch_one(pool)
    .await?
    .count
    .unwrap_or(0);
    
    let has_password = sqlx::query!(
        "SELECT password_hash FROM users WHERE id = $1",
        user_id
    )
    .fetch_one(pool)
    .await?
    .password_hash
    .is_some();
    
    if other_providers_count == 0 && !has_password {
        return Err(anyhow::anyhow!(
            "Cannot unlink last authentication method"
        ));
    }
    
    let result = sqlx::query!(
        "DELETE FROM oauth_accounts WHERE user_id = $1 AND provider = $2",
        user_id,
        provider
    )
    .execute(pool)
    .await?;
    
    Ok(result.rows_affected() > 0)
}
```

---

## 7. Main Application

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware};
use actix_session::{SessionMiddleware, storage::CookieSessionStore};
use actix_web::cookie::{Key, SameSite};
use sqlx::postgres::PgPoolOptions;
use dotenv::dotenv;

mod oauth_config;
mod providers {
    pub mod google;
    pub mod github;
}
mod handlers {
    pub mod google;
    pub mod github;
}
mod db {
    pub mod oauth_accounts;
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv().ok();
    env_logger::init();
    
    let database_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await
        .expect("Failed to connect to database");
    
    sqlx::migrate!("./migrations").run(&pool).await
        .expect("Failed to run migrations");
    
    let oauth_config = oauth_config::OAuthConfig::from_env()
        .expect("Failed to initialize OAuth config");
    let oauth_data = web::Data::new(oauth_config);
    
    // Session key ต้องยาวอย่างน้อย 64 bytes
    let session_secret = std::env::var("SESSION_SECRET")
        .expect("SESSION_SECRET must be set");
    let session_key = Key::from(session_secret.as_bytes());
    
    log::info!("Starting OAuth server on 0.0.0.0:8080");
    
    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(pool.clone()))
            .app_data(oauth_data.clone())
            .wrap(middleware::Logger::default())
            .wrap(
                SessionMiddleware::builder(
                    CookieSessionStore::default(),
                    session_key.clone(),
                )
                .cookie_secure(false) // true ใน production
                .cookie_same_site(SameSite::Lax)
                .build()
            )
            .service(
                web::scope("/auth")
                    .route("/google", web::get().to(handlers::google::google_login))
                    .route("/google/callback", web::get().to(handlers::google::google_callback))
                    .route("/github", web::get().to(handlers::github::github_login))
                    .route("/github/callback", web::get().to(handlers::github::github_callback))
            )
            // Protected routes
            .route("/dashboard", web::get().to(dashboard))
            .route("/profile/providers", web::get().to(list_providers))
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}

async fn dashboard(session: actix_session::Session) -> actix_web::Result<actix_web::HttpResponse> {
    let user_id: Option<String> = session.get("user_id")
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    match user_id {
        Some(id) => Ok(actix_web::HttpResponse::Ok().json(serde_json::json!({
            "message": "Welcome to dashboard",
            "user_id": id
        }))),
        None => Ok(actix_web::HttpResponse::Unauthorized().json(serde_json::json!({
            "error": "Not authenticated"
        }))),
    }
}

async fn list_providers(
    session: actix_session::Session,
    pool: web::Data<sqlx::PgPool>,
) -> actix_web::Result<actix_web::HttpResponse> {
    let user_id_str: Option<String> = session.get("user_id")
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    let user_id: uuid::Uuid = match user_id_str {
        Some(id) => id.parse().map_err(|_| actix_web::error::ErrorBadRequest("Invalid user ID"))?,
        None => return Ok(actix_web::HttpResponse::Unauthorized().finish()),
    };
    
    let providers = db::oauth_accounts::get_user_providers(pool.get_ref(), user_id)
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    Ok(actix_web::HttpResponse::Ok().json(serde_json::json!({
        "connected_providers": providers
    })))
}
```

---

## 8. ตั้งค่า Google OAuth2

### 8.1 Google Cloud Console Steps

1. ไปที่ [Google Cloud Console](https://console.cloud.google.com)
2. สร้าง project ใหม่หรือเลือก project ที่มีอยู่
3. ไปที่ **APIs & Services** → **Credentials**
4. คลิก **Create Credentials** → **OAuth client ID**
5. เลือก **Web application**
6. เพิ่ม Authorized redirect URIs: `http://localhost:8080/auth/google/callback`
7. คัดลอก Client ID และ Client Secret

### 8.2 GitHub OAuth App Setup

1. ไปที่ [GitHub Settings](https://github.com/settings/developers)
2. คลิก **New OAuth App**
3. กรอก:
   - Application name: ชื่อ app
   - Homepage URL: `http://localhost:8080`
   - Authorization callback URL: `http://localhost:8080/auth/github/callback`
4. คัดลอก Client ID และ Client Secret

---

## 9. Security Considerations

```rust
// src/security.rs

/// ตรวจสอบ state parameter ป้องกัน CSRF
pub fn validate_oauth_state(session_state: &str, callback_state: &str) -> bool {
    // ใช้ constant-time comparison
    use subtle::ConstantTimeEq;
    session_state.as_bytes().ct_eq(callback_state.as_bytes()).into()
}

/// Token rotation - เปลี่ยน access token เมื่อหมดอายุ
pub async fn refresh_access_token(
    refresh_token: &str,
    client: &oauth2::basic::BasicClient,
) -> anyhow::Result<oauth2::basic::BasicTokenResponse> {
    use oauth2::{RefreshToken, TokenResponse, reqwest::async_http_client};
    
    let refresh = RefreshToken::new(refresh_token.to_string());
    
    let token = client
        .exchange_refresh_token(&refresh)
        .request_async(async_http_client)
        .await?;
    
    Ok(token)
}

/// ตรวจสอบ token expiry
pub fn is_token_expired(expires_at: Option<chrono::DateTime<chrono::Utc>>) -> bool {
    match expires_at {
        Some(exp) => chrono::Utc::now() > exp,
        None => false, // ถ้าไม่รู้ expiry ถือว่ายังใช้ได้
    }
}
```

---

## 10. สรุปสิ่งที่เรียนรู้

✅ OAuth2 Authorization Code Flow  
✅ PKCE challenge สำหรับ security  
✅ CSRF protection ด้วย state parameter  
✅ Google OAuth2 integration  
✅ GitHub OAuth2 integration  
✅ Token exchange และ user info fetching  
✅ Link OAuth accounts กับ local users  
✅ Account linking strategy  

---

*[← Part 042: Password Hashing](../part_042/README.md) | [Part 044: RBAC →](../part_044/README.md)*
