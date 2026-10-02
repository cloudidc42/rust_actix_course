# Part 098: Capstone Project Part 1 - Social Platform

## บทนำ (Introduction)

ในบทนี้เราจะเริ่มสร้าง Capstone Project - Social Platform ที่สมบูรณ์แบบ ซึ่งรวม features ทั้งหมดที่เราเรียนมา ได้แก่ authentication, real-time, file uploads, search และ more

## 1. Project Architecture

### โครงสร้าง project

```
social-platform/
├── src/
│   ├── main.rs
│   ├── config.rs
│   ├── db.rs
│   ├── errors.rs
│   ├── auth/
│   │   ├── mod.rs
│   │   ├── middleware.rs
│   │   ├── jwt.rs
│   │   └── handlers.rs
│   ├── domain/
│   │   ├── mod.rs
│   │   ├── user.rs
│   │   ├── post.rs
│   │   ├── comment.rs
│   │   ├── like.rs
│   │   └── follow.rs
│   ├── handlers/
│   │   ├── mod.rs
│   │   ├── users.rs
│   │   ├── posts.rs
│   │   ├── comments.rs
│   │   ├── follows.rs
│   │   └── search.rs
│   ├── services/
│   │   ├── mod.rs
│   │   ├── user_service.rs
│   │   ├── post_service.rs
│   │   ├── notification_service.rs
│   │   └── upload_service.rs
│   ├── ws/
│   │   ├── mod.rs
│   │   ├── server.rs
│   │   └── session.rs
│   └── middleware/
│       ├── mod.rs
│       ├── auth.rs
│       └── rate_limit.rs
├── migrations/
├── static/
├── Cargo.toml
└── docker-compose.yml
```

```toml
# Cargo.toml
[package]
name = "social-platform"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = { version = "4", features = ["macros"] }
actix-ws = "0.2"
actix-multipart = "0.6"
actix-files = "0.6"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.7", features = [
    "postgres", "runtime-tokio-rustls", "uuid", "chrono", "json"
]}
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
jsonwebtoken = "9"
bcrypt = "0.15"
validator = { version = "0.16", features = ["derive"] }
config = "0.13"
tracing = "0.1"
tracing-actix-web = "0.7"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
anyhow = "1"
thiserror = "1"
redis = { version = "0.24", features = ["tokio-comp"] }
elasticsearch = "8"
lettre = { version = "0.11", features = ["tokio1", "tokio1-native-tls"] }
image = "0.24"
mime = "0.3"
futures = "0.3"
```

## 2. Domain Model

### User domain

```rust
// src/domain/user.rs
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use validator::Validate;

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct User {
    pub id: Uuid,
    pub username: String,
    pub email: String,
    #[serde(skip_serializing)]
    pub password_hash: String,
    pub display_name: Option<String>,
    pub bio: Option<String>,
    pub avatar_url: Option<String>,
    pub cover_url: Option<String>,
    pub is_verified: bool,
    pub is_active: bool,
    pub follower_count: i64,
    pub following_count: i64,
    pub post_count: i64,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct UserProfile {
    pub id: Uuid,
    pub username: String,
    pub display_name: Option<String>,
    pub bio: Option<String>,
    pub avatar_url: Option<String>,
    pub cover_url: Option<String>,
    pub is_verified: bool,
    pub follower_count: i64,
    pub following_count: i64,
    pub post_count: i64,
    pub is_following: bool,
    pub is_followed_by: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Validate)]
pub struct RegisterRequest {
    #[validate(length(min = 3, max = 50), regex(path = "USERNAME_REGEX"))]
    pub username: String,
    
    #[validate(email)]
    pub email: String,
    
    #[validate(length(min = 8, max = 100))]
    pub password: String,
    
    #[validate(must_match = "password")]
    pub confirm_password: String,
}

#[derive(Debug, Deserialize, Validate)]
pub struct LoginRequest {
    pub email: String,
    pub password: String,
}

#[derive(Debug, Deserialize, Validate)]
pub struct UpdateProfileRequest {
    #[validate(length(max = 100))]
    pub display_name: Option<String>,
    
    #[validate(length(max = 500))]
    pub bio: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct AuthResponse {
    pub access_token: String,
    pub refresh_token: String,
    pub token_type: String,
    pub expires_in: i64,
    pub user: UserProfile,
}

lazy_static::lazy_static! {
    static ref USERNAME_REGEX: regex::Regex = 
        regex::Regex::new(r"^[a-zA-Z0-9_]+$").unwrap();
}
```

### Post domain

```rust
// src/domain/post.rs
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use validator::Validate;

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Post {
    pub id: Uuid,
    pub user_id: Uuid,
    pub content: String,
    pub media_urls: Vec<String>,
    pub hashtags: Vec<String>,
    pub mentions: Vec<Uuid>,
    pub like_count: i64,
    pub comment_count: i64,
    pub share_count: i64,
    pub is_deleted: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Serialize)]
pub struct PostWithAuthor {
    pub id: Uuid,
    pub content: String,
    pub media_urls: Vec<String>,
    pub hashtags: Vec<String>,
    pub like_count: i64,
    pub comment_count: i64,
    pub is_liked: bool,
    pub author: PostAuthor,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Serialize)]
pub struct PostAuthor {
    pub id: Uuid,
    pub username: String,
    pub display_name: Option<String>,
    pub avatar_url: Option<String>,
    pub is_verified: bool,
}

#[derive(Debug, Deserialize, Validate)]
pub struct CreatePostRequest {
    #[validate(length(min = 1, max = 2000))]
    pub content: String,
    
    #[validate(length(max = 10))]
    pub media_urls: Vec<String>,
}

#[derive(Debug, Deserialize)]
pub struct PostQuery {
    pub page: Option<u32>,
    pub limit: Option<u32>,
    pub user_id: Option<Uuid>,
    pub hashtag: Option<String>,
}
```

### Comment, Like, Follow domains

```rust
// src/domain/comment.rs
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use chrono::{DateTime, Utc};

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Comment {
    pub id: Uuid,
    pub post_id: Uuid,
    pub user_id: Uuid,
    pub parent_id: Option<Uuid>,
    pub content: String,
    pub like_count: i64,
    pub reply_count: i64,
    pub is_deleted: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

// src/domain/like.rs
#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Like {
    pub id: Uuid,
    pub user_id: Uuid,
    pub post_id: Option<Uuid>,
    pub comment_id: Option<Uuid>,
    pub created_at: DateTime<Utc>,
}

// src/domain/follow.rs
#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Follow {
    pub id: Uuid,
    pub follower_id: Uuid,
    pub following_id: Uuid,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Serialize)]
pub struct FollowStats {
    pub follower_count: i64,
    pub following_count: i64,
    pub mutual_count: i64,
}
```

## 3. Database Setup

### Migrations

```sql
-- migrations/001_create_users.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm"; -- for full text search

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    display_name VARCHAR(100),
    bio TEXT,
    avatar_url VARCHAR(500),
    cover_url VARCHAR(500),
    is_verified BOOLEAN NOT NULL DEFAULT FALSE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    follower_count BIGINT NOT NULL DEFAULT 0,
    following_count BIGINT NOT NULL DEFAULT 0,
    post_count BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username_trgm ON users USING GIN(username gin_trgm_ops);

-- migrations/002_create_posts.sql
CREATE TABLE posts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    media_urls TEXT[] NOT NULL DEFAULT '{}',
    hashtags TEXT[] NOT NULL DEFAULT '{}',
    mentions UUID[] NOT NULL DEFAULT '{}',
    like_count BIGINT NOT NULL DEFAULT 0,
    comment_count BIGINT NOT NULL DEFAULT 0,
    share_count BIGINT NOT NULL DEFAULT 0,
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_created_at ON posts(created_at DESC);
CREATE INDEX idx_posts_hashtags ON posts USING GIN(hashtags);
CREATE INDEX idx_posts_content_trgm ON posts USING GIN(content gin_trgm_ops);

-- migrations/003_create_follows.sql
CREATE TABLE follows (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    follower_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    following_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(follower_id, following_id),
    CHECK(follower_id != following_id)
);

CREATE INDEX idx_follows_follower ON follows(follower_id);
CREATE INDEX idx_follows_following ON follows(following_id);

-- migrations/004_create_likes.sql
CREATE TABLE likes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    post_id UUID REFERENCES posts(id) ON DELETE CASCADE,
    comment_id UUID REFERENCES comments(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(user_id, post_id),
    CHECK(
        (post_id IS NOT NULL AND comment_id IS NULL) OR
        (post_id IS NULL AND comment_id IS NOT NULL)
    )
);

-- migrations/005_create_comments.sql
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    parent_id UUID REFERENCES comments(id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    like_count BIGINT NOT NULL DEFAULT 0,
    reply_count BIGINT NOT NULL DEFAULT 0,
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## 4. Authentication & Authorization

### JWT implementation

```rust
// src/auth/jwt.rs
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use chrono::{Duration, Utc};
use anyhow::Result;

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Claims {
    pub sub: String,    // user id
    pub email: String,
    pub username: String,
    pub exp: i64,
    pub iat: i64,
    pub token_type: String, // "access" or "refresh"
}

pub struct JwtService {
    access_secret: String,
    refresh_secret: String,
    access_exp_hours: i64,
    refresh_exp_days: i64,
}

impl JwtService {
    pub fn new(
        access_secret: String,
        refresh_secret: String,
    ) -> Self {
        JwtService {
            access_secret,
            refresh_secret,
            access_exp_hours: 1,
            refresh_exp_days: 30,
        }
    }
    
    pub fn create_access_token(
        &self,
        user_id: Uuid,
        email: &str,
        username: &str,
    ) -> Result<String> {
        let now = Utc::now();
        let exp = now + Duration::hours(self.access_exp_hours);
        
        let claims = Claims {
            sub: user_id.to_string(),
            email: email.to_string(),
            username: username.to_string(),
            exp: exp.timestamp(),
            iat: now.timestamp(),
            token_type: "access".to_string(),
        };
        
        Ok(encode(
            &Header::default(),
            &claims,
            &EncodingKey::from_secret(self.access_secret.as_bytes()),
        )?)
    }
    
    pub fn create_refresh_token(
        &self,
        user_id: Uuid,
        email: &str,
        username: &str,
    ) -> Result<String> {
        let now = Utc::now();
        let exp = now + Duration::days(self.refresh_exp_days);
        
        let claims = Claims {
            sub: user_id.to_string(),
            email: email.to_string(),
            username: username.to_string(),
            exp: exp.timestamp(),
            iat: now.timestamp(),
            token_type: "refresh".to_string(),
        };
        
        Ok(encode(
            &Header::default(),
            &claims,
            &EncodingKey::from_secret(self.refresh_secret.as_bytes()),
        )?)
    }
    
    pub fn verify_access_token(&self, token: &str) -> Result<Claims> {
        let data = decode::<Claims>(
            token,
            &DecodingKey::from_secret(self.access_secret.as_bytes()),
            &Validation::default(),
        )?;
        
        if data.claims.token_type != "access" {
            anyhow::bail!("Invalid token type");
        }
        
        Ok(data.claims)
    }
    
    pub fn verify_refresh_token(&self, token: &str) -> Result<Claims> {
        let data = decode::<Claims>(
            token,
            &DecodingKey::from_secret(self.refresh_secret.as_bytes()),
            &Validation::default(),
        )?;
        
        if data.claims.token_type != "refresh" {
            anyhow::bail!("Invalid token type");
        }
        
        Ok(data.claims)
    }
}

// Auth middleware
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpMessage,
};
use std::future::{ready, Ready};

pub struct AuthMiddleware;

impl<S, B> Transform<S, ServiceRequest> for AuthMiddleware
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = AuthMiddlewareService<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(AuthMiddlewareService { service }))
    }
}

pub struct AuthMiddlewareService<S> {
    service: S,
}

impl<S, B> Service<ServiceRequest> for AuthMiddlewareService<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error>,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = std::pin::Pin<Box<dyn std::future::Future<Output = Result<Self::Response, Self::Error>>>>;
    
    forward_ready!(service);
    
    fn call(&self, req: ServiceRequest) -> Self::Future {
        let auth_header = req
            .headers()
            .get("Authorization")
            .and_then(|h| h.to_str().ok())
            .and_then(|h| h.strip_prefix("Bearer "))
            .map(|t| t.to_string());
        
        if let Some(token) = auth_header {
            // Verify token and extract claims
            // In real code: use JwtService from app data
            if let Ok(claims) = verify_token_mock(&token) {
                req.extensions_mut().insert(claims);
            }
        }
        
        let fut = self.service.call(req);
        Box::pin(async move { fut.await })
    }
}

fn verify_token_mock(token: &str) -> Result<Claims, ()> {
    // Mock - in real code use JwtService
    Err(())
}
```

## 5. API Handlers

### User handlers

```rust
// src/handlers/users.rs
use actix_web::{web, HttpResponse, HttpRequest};
use uuid::Uuid;
use crate::{
    domain::user::*,
    services::user_service::UserService,
    auth::jwt::Claims,
    errors::AppError,
};

pub async fn register(
    db: web::Data<sqlx::PgPool>,
    body: web::Json<RegisterRequest>,
) -> Result<HttpResponse, AppError> {
    body.validate()?;
    
    let service = UserService::new(db.get_ref());
    let response = service.register(body.into_inner()).await?;
    
    Ok(HttpResponse::Created().json(response))
}

pub async fn login(
    db: web::Data<sqlx::PgPool>,
    body: web::Json<LoginRequest>,
) -> Result<HttpResponse, AppError> {
    let service = UserService::new(db.get_ref());
    let response = service.login(body.into_inner()).await?;
    
    Ok(HttpResponse::Ok().json(response))
}

pub async fn get_profile(
    db: web::Data<sqlx::PgPool>,
    path: web::Path<String>,
    claims: Option<web::ReqData<Claims>>,
) -> Result<HttpResponse, AppError> {
    let username = path.into_inner();
    let current_user_id = claims.map(|c| c.sub.parse::<Uuid>().ok()).flatten();
    
    let service = UserService::new(db.get_ref());
    let profile = service.get_profile(&username, current_user_id).await?;
    
    Ok(HttpResponse::Ok().json(profile))
}

pub async fn update_profile(
    db: web::Data<sqlx::PgPool>,
    claims: web::ReqData<Claims>,
    body: web::Json<UpdateProfileRequest>,
) -> Result<HttpResponse, AppError> {
    body.validate()?;
    
    let user_id = claims.sub.parse::<Uuid>()?;
    let service = UserService::new(db.get_ref());
    let profile = service.update_profile(user_id, body.into_inner()).await?;
    
    Ok(HttpResponse::Ok().json(profile))
}

pub async fn follow_user(
    db: web::Data<sqlx::PgPool>,
    path: web::Path<Uuid>,
    claims: web::ReqData<Claims>,
) -> Result<HttpResponse, AppError> {
    let follower_id = claims.sub.parse::<Uuid>()?;
    let following_id = path.into_inner();
    
    if follower_id == following_id {
        return Err(AppError::BadRequest("Cannot follow yourself".to_string()));
    }
    
    let service = UserService::new(db.get_ref());
    service.follow(follower_id, following_id).await?;
    
    Ok(HttpResponse::Ok().json(serde_json::json!({"followed": true})))
}

pub async fn unfollow_user(
    db: web::Data<sqlx::PgPool>,
    path: web::Path<Uuid>,
    claims: web::ReqData<Claims>,
) -> Result<HttpResponse, AppError> {
    let follower_id = claims.sub.parse::<Uuid>()?;
    let following_id = path.into_inner();
    
    let service = UserService::new(db.get_ref());
    service.unfollow(follower_id, following_id).await?;
    
    Ok(HttpResponse::Ok().json(serde_json::json!({"followed": false})))
}
```

### Post handlers

```rust
// src/handlers/posts.rs
use actix_web::{web, HttpResponse};
use uuid::Uuid;
use crate::{
    domain::post::*,
    services::post_service::PostService,
    auth::jwt::Claims,
    errors::AppError,
};

pub async fn create_post(
    db: web::Data<sqlx::PgPool>,
    claims: web::ReqData<Claims>,
    body: web::Json<CreatePostRequest>,
) -> Result<HttpResponse, AppError> {
    body.validate()?;
    
    let user_id = claims.sub.parse::<Uuid>()?;
    let service = PostService::new(db.get_ref());
    let post = service.create(user_id, body.into_inner()).await?;
    
    Ok(HttpResponse::Created().json(post))
}

pub async fn get_feed(
    db: web::Data<sqlx::PgPool>,
    claims: web::ReqData<Claims>,
    query: web::Query<PostQuery>,
) -> Result<HttpResponse, AppError> {
    let user_id = claims.sub.parse::<Uuid>()?;
    let service = PostService::new(db.get_ref());
    
    let page = query.page.unwrap_or(1);
    let limit = query.limit.unwrap_or(20).min(100);
    
    let posts = service.get_feed(user_id, page, limit).await?;
    
    Ok(HttpResponse::Ok().json(posts))
}

pub async fn like_post(
    db: web::Data<sqlx::PgPool>,
    path: web::Path<Uuid>,
    claims: web::ReqData<Claims>,
) -> Result<HttpResponse, AppError> {
    let user_id = claims.sub.parse::<Uuid>()?;
    let post_id = path.into_inner();
    
    let service = PostService::new(db.get_ref());
    let liked = service.toggle_like(user_id, post_id).await?;
    
    Ok(HttpResponse::Ok().json(serde_json::json!({"liked": liked})))
}

pub async fn get_post(
    db: web::Data<sqlx::PgPool>,
    path: web::Path<Uuid>,
    claims: Option<web::ReqData<Claims>>,
) -> Result<HttpResponse, AppError> {
    let post_id = path.into_inner();
    let user_id = claims.map(|c| c.sub.parse::<Uuid>().ok()).flatten();
    
    let service = PostService::new(db.get_ref());
    let post = service.get_post(post_id, user_id).await?;
    
    Ok(HttpResponse::Ok().json(post))
}

pub async fn delete_post(
    db: web::Data<sqlx::PgPool>,
    path: web::Path<Uuid>,
    claims: web::ReqData<Claims>,
) -> Result<HttpResponse, AppError> {
    let user_id = claims.sub.parse::<Uuid>()?;
    let post_id = path.into_inner();
    
    let service = PostService::new(db.get_ref());
    service.delete_post(user_id, post_id).await?;
    
    Ok(HttpResponse::NoContent().finish())
}
```

## 6. Real-time WebSocket

### WebSocket server

```rust
// src/ws/server.rs
use actix::prelude::*;
use std::collections::{HashMap, HashSet};
use uuid::Uuid;

#[derive(Message)]
#[rtype(result = "()")]
pub struct Connect {
    pub addr: Recipient<WsMessage>,
    pub user_id: Uuid,
    pub session_id: String,
}

#[derive(Message)]
#[rtype(result = "()")]
pub struct Disconnect {
    pub session_id: String,
    pub user_id: Uuid,
}

#[derive(Message, Clone)]
#[rtype(result = "()")]
pub struct WsMessage(pub String);

#[derive(Message)]
#[rtype(result = "()")]
pub struct SendToUser {
    pub user_id: Uuid,
    pub message: String,
}

#[derive(Message)]
#[rtype(result = "()")]
pub struct Broadcast {
    pub message: String,
    pub exclude: Option<String>,
}

pub struct WsServer {
    // session_id -> (user_id, recipient)
    sessions: HashMap<String, (Uuid, Recipient<WsMessage>)>,
    // user_id -> set of session_ids
    user_sessions: HashMap<Uuid, HashSet<String>>,
}

impl WsServer {
    pub fn new() -> Self {
        WsServer {
            sessions: HashMap::new(),
            user_sessions: HashMap::new(),
        }
    }
    
    fn send_to_session(&self, session_id: &str, message: &str) {
        if let Some((_, addr)) = self.sessions.get(session_id) {
            addr.do_send(WsMessage(message.to_string()));
        }
    }
    
    pub fn send_to_user(&self, user_id: Uuid, message: &str) {
        if let Some(sessions) = self.user_sessions.get(&user_id) {
            for session_id in sessions {
                self.send_to_session(session_id, message);
            }
        }
    }
    
    pub fn online_users(&self) -> Vec<Uuid> {
        self.user_sessions.keys().cloned().collect()
    }
}

impl Actor for WsServer {
    type Context = Context<Self>;
}

impl Handler<Connect> for WsServer {
    type Result = ();
    
    fn handle(&mut self, msg: Connect, _: &mut Context<Self>) {
        println!("User {} connected (session: {})", msg.user_id, msg.session_id);
        
        self.sessions.insert(
            msg.session_id.clone(),
            (msg.user_id, msg.addr)
        );
        
        self.user_sessions
            .entry(msg.user_id)
            .or_insert_with(HashSet::new)
            .insert(msg.session_id);
    }
}

impl Handler<Disconnect> for WsServer {
    type Result = ();
    
    fn handle(&mut self, msg: Disconnect, _: &mut Context<Self>) {
        println!("User {} disconnected (session: {})", msg.user_id, msg.session_id);
        
        self.sessions.remove(&msg.session_id);
        
        if let Some(sessions) = self.user_sessions.get_mut(&msg.user_id) {
            sessions.remove(&msg.session_id);
            if sessions.is_empty() {
                self.user_sessions.remove(&msg.user_id);
            }
        }
    }
}

impl Handler<SendToUser> for WsServer {
    type Result = ();
    
    fn handle(&mut self, msg: SendToUser, _: &mut Context<Self>) {
        self.send_to_user(msg.user_id, &msg.message);
    }
}

impl Handler<Broadcast> for WsServer {
    type Result = ();
    
    fn handle(&mut self, msg: Broadcast, _: &mut Context<Self>) {
        for (session_id, (_, addr)) in &self.sessions {
            if msg.exclude.as_deref() != Some(session_id) {
                addr.do_send(WsMessage(msg.message.clone()));
            }
        }
    }
}
```

## 7. File Upload (Avatar, Media)

### Multipart upload handler

```rust
// src/handlers/uploads.rs
use actix_multipart::Multipart;
use actix_web::{web, HttpResponse};
use futures::{StreamExt, TryStreamExt};
use std::path::PathBuf;
use uuid::Uuid;
use image::ImageFormat;
use crate::{auth::jwt::Claims, errors::AppError};

const MAX_FILE_SIZE: usize = 10 * 1024 * 1024; // 10MB
const ALLOWED_MIME_TYPES: &[&str] = &["image/jpeg", "image/png", "image/gif", "image/webp"];

pub async fn upload_avatar(
    mut payload: Multipart,
    claims: web::ReqData<Claims>,
    upload_dir: web::Data<String>,
) -> Result<HttpResponse, AppError> {
    let user_id = claims.sub.parse::<Uuid>()?;
    
    while let Some(item) = payload.try_next().await? {
        let field = item;
        let content_type = field.content_type()
            .map(|ct| ct.to_string())
            .unwrap_or_default();
        
        if !ALLOWED_MIME_TYPES.contains(&content_type.as_str()) {
            return Err(AppError::BadRequest(
                format!("Unsupported file type: {}", content_type)
            ));
        }
        
        // Read file data
        let data: Vec<u8> = field
            .try_fold(Vec::new(), |mut acc, chunk| async move {
                acc.extend_from_slice(&chunk);
                if acc.len() > MAX_FILE_SIZE {
                    Err(actix_multipart::MultipartError::Payload(
                        actix_web::error::PayloadError::Overflow
                    ))
                } else {
                    Ok(acc)
                }
            })
            .await?;
        
        // Process image (resize for avatar)
        let processed = process_avatar(&data)?;
        
        // Save file
        let filename = format!("{}/avatars/{}.webp", upload_dir.as_str(), user_id);
        tokio::fs::write(&filename, &processed).await?;
        
        let url = format!("/uploads/avatars/{}.webp", user_id);
        
        return Ok(HttpResponse::Ok().json(serde_json::json!({
            "url": url,
            "size": processed.len(),
        })));
    }
    
    Err(AppError::BadRequest("No file provided".to_string()))
}

fn process_avatar(data: &[u8]) -> Result<Vec<u8>, AppError> {
    use image::{GenericImageView, imageops::FilterType};
    
    let img = image::load_from_memory(data)
        .map_err(|e| AppError::BadRequest(format!("Invalid image: {}", e)))?;
    
    // Resize to 200x200
    let resized = img.resize_to_fill(200, 200, FilterType::Lanczos3);
    
    // Convert to WebP
    let mut buffer = Vec::new();
    resized.write_to(&mut std::io::Cursor::new(&mut buffer), ImageFormat::WebP)
        .map_err(|e| AppError::Internal(format!("Image processing failed: {}", e)))?;
    
    Ok(buffer)
}

pub async fn upload_media(
    mut payload: Multipart,
    claims: web::ReqData<Claims>,
    upload_dir: web::Data<String>,
) -> Result<HttpResponse, AppError> {
    let user_id = claims.sub.parse::<Uuid>()?;
    let mut uploaded_urls = Vec::new();
    
    while let Some(item) = payload.try_next().await? {
        let field = item;
        let content_type = field.content_type()
            .map(|ct| ct.to_string())
            .unwrap_or_default();
        
        if !ALLOWED_MIME_TYPES.contains(&content_type.as_str()) {
            continue;
        }
        
        let data: Vec<u8> = field
            .try_fold(Vec::new(), |mut acc, chunk| async move {
                acc.extend_from_slice(&chunk);
                Ok(acc)
            })
            .await?;
        
        let file_id = Uuid::new_v4();
        let ext = match content_type.as_str() {
            "image/jpeg" => "jpg",
            "image/png" => "png",
            "image/gif" => "gif",
            _ => "webp",
        };
        
        let filename = format!("{}/media/{}/{}.{}", 
            upload_dir.as_str(), user_id, file_id, ext);
        
        tokio::fs::create_dir_all(format!("{}/media/{}", upload_dir.as_str(), user_id)).await?;
        tokio::fs::write(&filename, &data).await?;
        
        let url = format!("/uploads/media/{}/{}.{}", user_id, file_id, ext);
        uploaded_urls.push(url);
    }
    
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "urls": uploaded_urls,
        "count": uploaded_urls.len()
    })))
}
```

## 8. Main Application Setup

### main.rs

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware};
use sqlx::postgres::PgPoolOptions;
use tracing_actix_web::TracingLogger;

mod config;
mod db;
mod errors;
mod auth;
mod domain;
mod handlers;
mod services;
mod ws;

use config::Config;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // Initialize tracing
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::from_default_env()
        )
        .init();
    
    // Load config
    let config = Config::from_env().expect("Failed to load config");
    
    // Setup database pool
    let pool = PgPoolOptions::new()
        .max_connections(config.db.pool_size)
        .connect(&config.db.url)
        .await
        .expect("Failed to connect to database");
    
    // Run migrations
    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .expect("Failed to run migrations");
    
    tracing::info!("Starting server on {}:{}", config.server.host, config.server.port);
    
    let db = web::Data::new(pool);
    let upload_dir = web::Data::new(config.uploads.dir.clone());
    
    HttpServer::new(move || {
        App::new()
            .app_data(db.clone())
            .app_data(upload_dir.clone())
            .wrap(TracingLogger::default())
            .wrap(middleware::Compress::default())
            .wrap(
                actix_cors::Cors::default()
                    .allow_any_origin()
                    .allow_any_method()
                    .allow_any_header()
            )
            // Auth routes
            .route("/api/auth/register", web::post().to(handlers::users::register))
            .route("/api/auth/login", web::post().to(handlers::users::login))
            // User routes
            .route("/api/users/{username}", web::get().to(handlers::users::get_profile))
            .route("/api/users/me", web::put().to(handlers::users::update_profile))
            .route("/api/users/{id}/follow", web::post().to(handlers::users::follow_user))
            .route("/api/users/{id}/unfollow", web::post().to(handlers::users::unfollow_user))
            // Post routes
            .route("/api/posts", web::post().to(handlers::posts::create_post))
            .route("/api/posts/feed", web::get().to(handlers::posts::get_feed))
            .route("/api/posts/{id}", web::get().to(handlers::posts::get_post))
            .route("/api/posts/{id}", web::delete().to(handlers::posts::delete_post))
            .route("/api/posts/{id}/like", web::post().to(handlers::posts::like_post))
            // Upload routes
            .route("/api/uploads/avatar", web::post().to(handlers::uploads::upload_avatar))
            .route("/api/uploads/media", web::post().to(handlers::uploads::upload_media))
            // Static files
            .service(actix_files::Files::new("/uploads", &config.uploads.dir))
    })
    .bind((config.server.host.as_str(), config.server.port))?
    .run()
    .await
}
```

## สรุป (Summary)

ใน Part 1 ของ Capstone Project เราได้สร้าง:
- **Project Architecture**: โครงสร้าง project ที่ scalable
- **Domain Models**: Users, Posts, Comments, Likes, Follows
- **Database**: PostgreSQL migrations ด้วย UUID, indexes
- **Authentication**: JWT access/refresh tokens
- **API Handlers**: RESTful endpoints
- **Real-time**: WebSocket server ด้วย Actix actors
- **File Uploads**: Avatar และ media upload ด้วย image processing

---

[← Part 097](../part_097/README.md) | [Part 099 →](../part_099/README.md)
