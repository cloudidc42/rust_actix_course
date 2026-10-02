# Part 081: Project: Blog API 🦀

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ออกแบบ Blog API ที่สมบูรณ์ด้วย Rust และ Actix-web
- สร้าง Models สำหรับ Users, Posts, Comments, Tags
- ใช้งาน JWT Authentication
- ทำ Role-based Access Control (author/admin/reader)
- สร้าง Post Publishing Workflow (draft/published)
- ทำ Search Functionality และ Pagination
- เขียน Full CRUD สำหรับทุก Resources

---

## 1. โครงสร้างโปรเจกต์

```
blog_api/
├── Cargo.toml
├── .env
├── migrations/
│   ├── 001_create_users.sql
│   ├── 002_create_posts.sql
│   ├── 003_create_comments.sql
│   └── 004_create_tags.sql
└── src/
    ├── main.rs
    ├── config.rs
    ├── db.rs
    ├── errors.rs
    ├── middleware/
    │   ├── mod.rs
    │   ├── auth.rs
    │   └── rate_limit.rs
    ├── models/
    │   ├── mod.rs
    │   ├── user.rs
    │   ├── post.rs
    │   ├── comment.rs
    │   └── tag.rs
    ├── handlers/
    │   ├── mod.rs
    │   ├── auth.rs
    │   ├── users.rs
    │   ├── posts.rs
    │   ├── comments.rs
    │   └── tags.rs
    └── services/
        ├── mod.rs
        ├── auth_service.rs
        └── search_service.rs
```

---

## 2. Cargo.toml

```toml
[package]
name = "blog_api"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
actix-cors = "0.7"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }
tokio = { version = "1", features = ["full"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
jsonwebtoken = "9"
bcrypt = "0.15"
dotenv = "0.15"
env_logger = "0.11"
log = "0.4"
validator = { version = "0.18", features = ["derive"] }
thiserror = "1"
anyhow = "1"
slug = "0.1"
```

---

## 3. Models

### 3.1 User Model (`src/models/user.rs`)

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use validator::Validate;

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type, PartialEq)]
#[sqlx(type_name = "user_role", rename_all = "lowercase")]
pub enum UserRole {
    Admin,
    Author,
    Reader,
}

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct User {
    pub id: Uuid,
    pub username: String,
    pub email: String,
    #[serde(skip_serializing)]
    pub password_hash: String,
    pub display_name: Option<String>,
    pub avatar_url: Option<String>,
    pub bio: Option<String>,
    pub role: UserRole,
    pub is_active: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Validate)]
pub struct CreateUserRequest {
    #[validate(length(min = 3, max = 50))]
    pub username: String,
    #[validate(email)]
    pub email: String,
    #[validate(length(min = 8))]
    pub password: String,
    pub display_name: Option<String>,
}

#[derive(Debug, Deserialize, Validate)]
pub struct UpdateUserRequest {
    pub display_name: Option<String>,
    pub bio: Option<String>,
    pub avatar_url: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct UserProfile {
    pub id: Uuid,
    pub username: String,
    pub display_name: Option<String>,
    pub avatar_url: Option<String>,
    pub bio: Option<String>,
    pub role: UserRole,
    pub created_at: DateTime<Utc>,
}

impl From<User> for UserProfile {
    fn from(user: User) -> Self {
        UserProfile {
            id: user.id,
            username: user.username,
            display_name: user.display_name,
            avatar_url: user.avatar_url,
            bio: user.bio,
            role: user.role,
            created_at: user.created_at,
        }
    }
}
```

### 3.2 Post Model (`src/models/post.rs`)

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use validator::Validate;

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type, PartialEq)]
#[sqlx(type_name = "post_status", rename_all = "lowercase")]
pub enum PostStatus {
    Draft,
    Published,
    Archived,
}

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Post {
    pub id: Uuid,
    pub title: String,
    pub slug: String,
    pub excerpt: Option<String>,
    pub content: String,
    pub cover_image_url: Option<String>,
    pub status: PostStatus,
    pub author_id: Uuid,
    pub view_count: i64,
    pub published_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Serialize)]
pub struct PostWithAuthor {
    pub id: Uuid,
    pub title: String,
    pub slug: String,
    pub excerpt: Option<String>,
    pub content: String,
    pub cover_image_url: Option<String>,
    pub status: PostStatus,
    pub author_id: Uuid,
    pub author_username: String,
    pub author_display_name: Option<String>,
    pub view_count: i64,
    pub published_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
    pub tags: Vec<Tag>,
    pub comment_count: i64,
}

#[derive(Debug, Deserialize, Validate)]
pub struct CreatePostRequest {
    #[validate(length(min = 5, max = 200))]
    pub title: String,
    pub excerpt: Option<String>,
    #[validate(length(min = 100))]
    pub content: String,
    pub cover_image_url: Option<String>,
    pub tag_ids: Vec<Uuid>,
    pub status: Option<PostStatus>,
}

#[derive(Debug, Deserialize, Validate)]
pub struct UpdatePostRequest {
    #[validate(length(min = 5, max = 200))]
    pub title: Option<String>,
    pub excerpt: Option<String>,
    pub content: Option<String>,
    pub cover_image_url: Option<String>,
    pub tag_ids: Option<Vec<Uuid>>,
}

#[derive(Debug, Deserialize)]
pub struct PostQuery {
    pub page: Option<u32>,
    pub per_page: Option<u32>,
    pub search: Option<String>,
    pub tag: Option<String>,
    pub author: Option<String>,
    pub status: Option<PostStatus>,
}

use super::tag::Tag;
```

### 3.3 Comment Model (`src/models/comment.rs`)

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use validator::Validate;

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Comment {
    pub id: Uuid,
    pub content: String,
    pub post_id: Uuid,
    pub author_id: Uuid,
    pub parent_id: Option<Uuid>,
    pub is_approved: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Serialize)]
pub struct CommentWithAuthor {
    pub id: Uuid,
    pub content: String,
    pub post_id: Uuid,
    pub author_id: Uuid,
    pub author_username: String,
    pub author_avatar: Option<String>,
    pub parent_id: Option<Uuid>,
    pub replies: Vec<CommentWithAuthor>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Validate)]
pub struct CreateCommentRequest {
    #[validate(length(min = 1, max = 2000))]
    pub content: String,
    pub parent_id: Option<Uuid>,
}
```

### 3.4 Tag Model (`src/models/tag.rs`)

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use validator::Validate;

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Tag {
    pub id: Uuid,
    pub name: String,
    pub slug: String,
    pub description: Option<String>,
    pub post_count: i64,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Validate)]
pub struct CreateTagRequest {
    #[validate(length(min = 2, max = 50))]
    pub name: String,
    pub description: Option<String>,
}
```

---

## 4. Authentication Service (`src/services/auth_service.rs`)

```rust
use chrono::{Duration, Utc};
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

use crate::errors::AppError;
use crate::models::user::UserRole;

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Claims {
    pub sub: Uuid,       // user id
    pub email: String,
    pub role: UserRole,
    pub exp: i64,
    pub iat: i64,
}

pub struct AuthService {
    secret: String,
    expiry_hours: i64,
}

impl AuthService {
    pub fn new(secret: String, expiry_hours: i64) -> Self {
        AuthService { secret, expiry_hours }
    }

    pub fn generate_token(&self, user_id: Uuid, email: &str, role: &UserRole) -> Result<String, AppError> {
        let now = Utc::now();
        let exp = (now + Duration::hours(self.expiry_hours)).timestamp();

        let claims = Claims {
            sub: user_id,
            email: email.to_string(),
            role: role.clone(),
            exp,
            iat: now.timestamp(),
        };

        encode(
            &Header::default(),
            &claims,
            &EncodingKey::from_secret(self.secret.as_bytes()),
        )
        .map_err(|e| AppError::InternalError(e.to_string()))
    }

    pub fn verify_token(&self, token: &str) -> Result<Claims, AppError> {
        let token_data = decode::<Claims>(
            token,
            &DecodingKey::from_secret(self.secret.as_bytes()),
            &Validation::default(),
        )
        .map_err(|e| AppError::Unauthorized(e.to_string()))?;

        Ok(token_data.claims)
    }

    pub fn hash_password(password: &str) -> Result<String, AppError> {
        bcrypt::hash(password, bcrypt::DEFAULT_COST)
            .map_err(|e| AppError::InternalError(e.to_string()))
    }

    pub fn verify_password(password: &str, hash: &str) -> Result<bool, AppError> {
        bcrypt::verify(password, hash)
            .map_err(|e| AppError::InternalError(e.to_string()))
    }
}
```

---

## 5. Error Handling (`src/errors.rs`)

```rust
use actix_web::{HttpResponse, ResponseError};
use serde::Serialize;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("Not found: {0}")]
    NotFound(String),
    #[error("Unauthorized: {0}")]
    Unauthorized(String),
    #[error("Forbidden: {0}")]
    Forbidden(String),
    #[error("Bad request: {0}")]
    BadRequest(String),
    #[error("Conflict: {0}")]
    Conflict(String),
    #[error("Internal error: {0}")]
    InternalError(String),
    #[error("Database error: {0}")]
    DatabaseError(String),
    #[error("Validation error: {0}")]
    ValidationError(String),
}

#[derive(Serialize)]
struct ErrorResponse {
    error: String,
    message: String,
}

impl ResponseError for AppError {
    fn error_response(&self) -> HttpResponse {
        let (status, error_type) = match self {
            AppError::NotFound(_) => (actix_web::http::StatusCode::NOT_FOUND, "NOT_FOUND"),
            AppError::Unauthorized(_) => (actix_web::http::StatusCode::UNAUTHORIZED, "UNAUTHORIZED"),
            AppError::Forbidden(_) => (actix_web::http::StatusCode::FORBIDDEN, "FORBIDDEN"),
            AppError::BadRequest(_) => (actix_web::http::StatusCode::BAD_REQUEST, "BAD_REQUEST"),
            AppError::Conflict(_) => (actix_web::http::StatusCode::CONFLICT, "CONFLICT"),
            AppError::ValidationError(_) => (actix_web::http::StatusCode::UNPROCESSABLE_ENTITY, "VALIDATION_ERROR"),
            _ => (actix_web::http::StatusCode::INTERNAL_SERVER_ERROR, "INTERNAL_ERROR"),
        };

        HttpResponse::build(status).json(ErrorResponse {
            error: error_type.to_string(),
            message: self.to_string(),
        })
    }
}

impl From<sqlx::Error> for AppError {
    fn from(e: sqlx::Error) -> Self {
        match e {
            sqlx::Error::RowNotFound => AppError::NotFound("Resource not found".to_string()),
            _ => AppError::DatabaseError(e.to_string()),
        }
    }
}

impl From<validator::ValidationErrors> for AppError {
    fn from(e: validator::ValidationErrors) -> Self {
        AppError::ValidationError(e.to_string())
    }
}
```

---

## 6. Auth Middleware (`src/middleware/auth.rs`)

```rust
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpMessage,
};
use futures_util::future::LocalBoxFuture;
use std::{
    future::{ready, Ready},
    rc::Rc,
};

use crate::services::auth_service::{AuthService, Claims};
use crate::errors::AppError;

pub struct JwtAuth {
    auth_service: Rc<AuthService>,
}

impl JwtAuth {
    pub fn new(auth_service: AuthService) -> Self {
        JwtAuth {
            auth_service: Rc::new(auth_service),
        }
    }
}

impl<S, B> Transform<S, ServiceRequest> for JwtAuth
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = JwtAuthMiddleware<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(JwtAuthMiddleware {
            service: Rc::new(service),
            auth_service: self.auth_service.clone(),
        }))
    }
}

pub struct JwtAuthMiddleware<S> {
    service: Rc<S>,
    auth_service: Rc<AuthService>,
}

impl<S, B> Service<ServiceRequest> for JwtAuthMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    forward_ready!(service);

    fn call(&self, req: ServiceRequest) -> Self::Future {
        let auth_service = self.auth_service.clone();
        let service = self.service.clone();

        Box::pin(async move {
            // Extract token from Authorization header
            let token = req
                .headers()
                .get("Authorization")
                .and_then(|h| h.to_str().ok())
                .and_then(|h| h.strip_prefix("Bearer "))
                .map(|t| t.to_string());

            if let Some(token) = token {
                match auth_service.verify_token(&token) {
                    Ok(claims) => {
                        req.extensions_mut().insert(claims);
                    }
                    Err(_) => {
                        return Err(actix_web::error::ErrorUnauthorized("Invalid token"));
                    }
                }
            }

            service.call(req).await
        })
    }
}

// Helper to extract claims from request
pub fn get_claims(req: &actix_web::HttpRequest) -> Option<Claims> {
    req.extensions().get::<Claims>().cloned()
}

pub fn require_auth(req: &actix_web::HttpRequest) -> Result<Claims, AppError> {
    get_claims(req).ok_or_else(|| AppError::Unauthorized("Authentication required".to_string()))
}
```

---

## 7. Handlers

### 7.1 Auth Handler (`src/handlers/auth.rs`)

```rust
use actix_web::{web, HttpResponse};
use serde::{Deserialize, Serialize};
use sqlx::PgPool;
use uuid::Uuid;
use validator::Validate;

use crate::errors::AppError;
use crate::models::user::{CreateUserRequest, UserRole};
use crate::services::auth_service::AuthService;

#[derive(Debug, Deserialize, Validate)]
pub struct LoginRequest {
    #[validate(email)]
    pub email: String,
    pub password: String,
}

#[derive(Debug, Serialize)]
pub struct AuthResponse {
    pub token: String,
    pub user_id: Uuid,
    pub username: String,
    pub role: UserRole,
}

pub async fn register(
    pool: web::Data<PgPool>,
    auth_service: web::Data<AuthService>,
    req: web::Json<CreateUserRequest>,
) -> Result<HttpResponse, AppError> {
    req.validate()?;

    // Check if email already exists
    let existing = sqlx::query_scalar::<_, i64>(
        "SELECT COUNT(*) FROM users WHERE email = $1 OR username = $2"
    )
    .bind(&req.email)
    .bind(&req.username)
    .fetch_one(pool.get_ref())
    .await?;

    if existing > 0 {
        return Err(AppError::Conflict("Email or username already exists".to_string()));
    }

    let password_hash = AuthService::hash_password(&req.password)?;
    let user_id = Uuid::new_v4();

    sqlx::query(
        r#"
        INSERT INTO users (id, username, email, password_hash, display_name, role)
        VALUES ($1, $2, $3, $4, $5, $6)
        "#
    )
    .bind(user_id)
    .bind(&req.username)
    .bind(&req.email)
    .bind(&password_hash)
    .bind(&req.display_name)
    .bind(UserRole::Reader)
    .execute(pool.get_ref())
    .await?;

    let token = auth_service.generate_token(user_id, &req.email, &UserRole::Reader)?;

    Ok(HttpResponse::Created().json(AuthResponse {
        token,
        user_id,
        username: req.username.clone(),
        role: UserRole::Reader,
    }))
}

pub async fn login(
    pool: web::Data<PgPool>,
    auth_service: web::Data<AuthService>,
    req: web::Json<LoginRequest>,
) -> Result<HttpResponse, AppError> {
    req.validate()?;

    let user = sqlx::query!(
        "SELECT id, username, email, password_hash, role as \"role: UserRole\" FROM users WHERE email = $1 AND is_active = true",
        req.email
    )
    .fetch_optional(pool.get_ref())
    .await?
    .ok_or_else(|| AppError::Unauthorized("Invalid email or password".to_string()))?;

    let is_valid = AuthService::verify_password(&req.password, &user.password_hash)?;
    if !is_valid {
        return Err(AppError::Unauthorized("Invalid email or password".to_string()));
    }

    let token = auth_service.generate_token(user.id, &user.email, &user.role)?;

    Ok(HttpResponse::Ok().json(AuthResponse {
        token,
        user_id: user.id,
        username: user.username,
        role: user.role,
    }))
}
```

### 7.2 Posts Handler (`src/handlers/posts.rs`)

```rust
use actix_web::{web, HttpRequest, HttpResponse};
use slug::slugify;
use sqlx::PgPool;
use uuid::Uuid;
use validator::Validate;

use crate::errors::AppError;
use crate::middleware::auth::require_auth;
use crate::models::post::{CreatePostRequest, PostQuery, PostStatus, UpdatePostRequest};
use crate::models::user::UserRole;

pub async fn list_posts(
    pool: web::Data<PgPool>,
    query: web::Query<PostQuery>,
    req: HttpRequest,
) -> Result<HttpResponse, AppError> {
    let page = query.page.unwrap_or(1);
    let per_page = query.per_page.unwrap_or(10).min(100);
    let offset = ((page - 1) * per_page) as i64;

    // Check if admin/author (can see drafts)
    let claims = crate::middleware::auth::get_claims(&req);
    let show_all_statuses = claims.as_ref().map(|c| {
        c.role == UserRole::Admin || c.role == UserRole::Author
    }).unwrap_or(false);

    let posts = if show_all_statuses {
        sqlx::query_as!(
            crate::models::post::Post,
            r#"
            SELECT p.id, p.title, p.slug, p.excerpt, p.content, p.cover_image_url,
                   p.status as "status: PostStatus", p.author_id, p.view_count,
                   p.published_at, p.created_at, p.updated_at
            FROM posts p
            WHERE ($1::text IS NULL OR p.title ILIKE '%' || $1 || '%' OR p.content ILIKE '%' || $1 || '%')
            ORDER BY p.created_at DESC
            LIMIT $2 OFFSET $3
            "#,
            query.search,
            per_page as i64,
            offset
        )
        .fetch_all(pool.get_ref())
        .await?
    } else {
        sqlx::query_as!(
            crate::models::post::Post,
            r#"
            SELECT p.id, p.title, p.slug, p.excerpt, p.content, p.cover_image_url,
                   p.status as "status: PostStatus", p.author_id, p.view_count,
                   p.published_at, p.created_at, p.updated_at
            FROM posts p
            WHERE p.status = 'published'
            AND ($1::text IS NULL OR p.title ILIKE '%' || $1 || '%' OR p.content ILIKE '%' || $1 || '%')
            ORDER BY p.published_at DESC
            LIMIT $2 OFFSET $3
            "#,
            query.search,
            per_page as i64,
            offset
        )
        .fetch_all(pool.get_ref())
        .await?
    };

    let total: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM posts WHERE status = 'published'")
        .fetch_one(pool.get_ref())
        .await?;

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "posts": posts,
        "pagination": {
            "page": page,
            "per_page": per_page,
            "total": total,
            "total_pages": (total as f64 / per_page as f64).ceil() as i64,
        }
    })))
}

pub async fn get_post(
    pool: web::Data<PgPool>,
    path: web::Path<String>,
) -> Result<HttpResponse, AppError> {
    let slug = path.into_inner();

    let post = sqlx::query_as!(
        crate::models::post::Post,
        r#"
        SELECT id, title, slug, excerpt, content, cover_image_url,
               status as "status: PostStatus", author_id, view_count,
               published_at, created_at, updated_at
        FROM posts WHERE slug = $1
        "#,
        slug
    )
    .fetch_optional(pool.get_ref())
    .await?
    .ok_or_else(|| AppError::NotFound(format!("Post '{}' not found", slug)))?;

    // Increment view count
    sqlx::query!("UPDATE posts SET view_count = view_count + 1 WHERE id = $1", post.id)
        .execute(pool.get_ref())
        .await?;

    Ok(HttpResponse::Ok().json(post))
}

pub async fn create_post(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    body: web::Json<CreatePostRequest>,
) -> Result<HttpResponse, AppError> {
    body.validate()?;
    let claims = require_auth(&req)?;

    // Only authors and admins can create posts
    if claims.role == UserRole::Reader {
        return Err(AppError::Forbidden("Only authors can create posts".to_string()));
    }

    let slug = slugify(&body.title);
    let post_id = Uuid::new_v4();
    let status = body.status.clone().unwrap_or(PostStatus::Draft);

    let published_at = if status == PostStatus::Published {
        Some(chrono::Utc::now())
    } else {
        None
    };

    sqlx::query!(
        r#"
        INSERT INTO posts (id, title, slug, excerpt, content, cover_image_url, status, author_id, published_at)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        "#,
        post_id,
        body.title,
        slug,
        body.excerpt,
        body.content,
        body.cover_image_url,
        status as PostStatus,
        claims.sub,
        published_at
    )
    .execute(pool.get_ref())
    .await?;

    // Add tags
    for tag_id in &body.tag_ids {
        sqlx::query!(
            "INSERT INTO post_tags (post_id, tag_id) VALUES ($1, $2) ON CONFLICT DO NOTHING",
            post_id,
            tag_id
        )
        .execute(pool.get_ref())
        .await?;
    }

    let post = sqlx::query_as!(
        crate::models::post::Post,
        r#"
        SELECT id, title, slug, excerpt, content, cover_image_url,
               status as "status: PostStatus", author_id, view_count,
               published_at, created_at, updated_at
        FROM posts WHERE id = $1
        "#,
        post_id
    )
    .fetch_one(pool.get_ref())
    .await?;

    Ok(HttpResponse::Created().json(post))
}

pub async fn update_post(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    path: web::Path<Uuid>,
    body: web::Json<UpdatePostRequest>,
) -> Result<HttpResponse, AppError> {
    let claims = require_auth(&req)?;
    let post_id = path.into_inner();

    // Fetch existing post
    let post = sqlx::query!(
        "SELECT author_id FROM posts WHERE id = $1",
        post_id
    )
    .fetch_optional(pool.get_ref())
    .await?
    .ok_or_else(|| AppError::NotFound("Post not found".to_string()))?;

    // Only the author or admin can update
    if claims.sub != post.author_id && claims.role != UserRole::Admin {
        return Err(AppError::Forbidden("You don't have permission to update this post".to_string()));
    }

    sqlx::query!(
        r#"
        UPDATE posts
        SET title = COALESCE($1, title),
            excerpt = COALESCE($2, excerpt),
            content = COALESCE($3, content),
            cover_image_url = COALESCE($4, cover_image_url),
            updated_at = NOW()
        WHERE id = $5
        "#,
        body.title,
        body.excerpt,
        body.content,
        body.cover_image_url,
        post_id
    )
    .execute(pool.get_ref())
    .await?;

    Ok(HttpResponse::Ok().json(serde_json::json!({ "message": "Post updated successfully" })))
}

pub async fn publish_post(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    path: web::Path<Uuid>,
) -> Result<HttpResponse, AppError> {
    let claims = require_auth(&req)?;
    let post_id = path.into_inner();

    let post = sqlx::query!(
        "SELECT author_id FROM posts WHERE id = $1",
        post_id
    )
    .fetch_optional(pool.get_ref())
    .await?
    .ok_or_else(|| AppError::NotFound("Post not found".to_string()))?;

    if claims.sub != post.author_id && claims.role != UserRole::Admin {
        return Err(AppError::Forbidden("Permission denied".to_string()));
    }

    sqlx::query!(
        "UPDATE posts SET status = 'published', published_at = NOW(), updated_at = NOW() WHERE id = $1",
        post_id
    )
    .execute(pool.get_ref())
    .await?;

    Ok(HttpResponse::Ok().json(serde_json::json!({ "message": "Post published successfully" })))
}

pub async fn delete_post(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    path: web::Path<Uuid>,
) -> Result<HttpResponse, AppError> {
    let claims = require_auth(&req)?;
    let post_id = path.into_inner();

    let post = sqlx::query!(
        "SELECT author_id FROM posts WHERE id = $1",
        post_id
    )
    .fetch_optional(pool.get_ref())
    .await?
    .ok_or_else(|| AppError::NotFound("Post not found".to_string()))?;

    if claims.sub != post.author_id && claims.role != UserRole::Admin {
        return Err(AppError::Forbidden("Permission denied".to_string()));
    }

    sqlx::query!("DELETE FROM posts WHERE id = $1", post_id)
        .execute(pool.get_ref())
        .await?;

    Ok(HttpResponse::NoContent().finish())
}

use chrono;
```

---

## 8. Main Application (`src/main.rs`)

```rust
use actix_cors::Cors;
use actix_web::{middleware::Logger, web, App, HttpServer};
use dotenv::dotenv;
use sqlx::postgres::PgPoolOptions;
use std::env;

mod config;
mod db;
mod errors;
mod handlers;
mod middleware;
mod models;
mod services;

use handlers::{auth as auth_handler, comments, posts, tags, users};
use middleware::auth::JwtAuth;
use services::auth_service::AuthService;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv().ok();
    env_logger::init_from_env(env_logger::Env::new().default_filter_or("info"));

    let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    let jwt_secret = env::var("JWT_SECRET").expect("JWT_SECRET must be set");
    let host = env::var("HOST").unwrap_or_else(|_| "127.0.0.1".to_string());
    let port = env::var("PORT").unwrap_or_else(|_| "8080".to_string());

    let pool = PgPoolOptions::new()
        .max_connections(10)
        .connect(&database_url)
        .await
        .expect("Failed to create database pool");

    let auth_service = web::Data::new(AuthService::new(jwt_secret, 24));

    log::info!("Starting Blog API at http://{}:{}", host, port);

    HttpServer::new(move || {
        let cors = Cors::default()
            .allow_any_origin()
            .allow_any_method()
            .allow_any_header()
            .max_age(3600);

        let jwt_auth = JwtAuth::new(AuthService::new(
            env::var("JWT_SECRET").unwrap(),
            24,
        ));

        App::new()
            .wrap(Logger::default())
            .wrap(cors)
            .app_data(web::Data::new(pool.clone()))
            .app_data(auth_service.clone())
            // Auth routes
            .service(
                web::scope("/api/auth")
                    .route("/register", web::post().to(auth_handler::register))
                    .route("/login", web::post().to(auth_handler::login))
            )
            // Post routes
            .service(
                web::scope("/api/posts")
                    .wrap(jwt_auth)
                    .route("", web::get().to(posts::list_posts))
                    .route("", web::post().to(posts::create_post))
                    .route("/{slug}", web::get().to(posts::get_post))
                    .route("/{id}", web::put().to(posts::update_post))
                    .route("/{id}", web::delete().to(posts::delete_post))
                    .route("/{id}/publish", web::post().to(posts::publish_post))
                    .route("/{post_id}/comments", web::get().to(comments::list_comments))
                    .route("/{post_id}/comments", web::post().to(comments::create_comment))
            )
            // Tag routes
            .service(
                web::scope("/api/tags")
                    .route("", web::get().to(tags::list_tags))
                    .route("", web::post().to(tags::create_tag))
                    .route("/{id}", web::delete().to(tags::delete_tag))
            )
            // User routes
            .service(
                web::scope("/api/users")
                    .route("/{id}", web::get().to(users::get_user))
                    .route("/me", web::put().to(users::update_me))
            )
    })
    .bind(format!("{}:{}", host, port))?
    .run()
    .await
}
```

---

## 9. Database Migrations

### `migrations/001_create_users.sql`

```sql
CREATE TYPE user_role AS ENUM ('admin', 'author', 'reader');

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    display_name VARCHAR(100),
    avatar_url TEXT,
    bio TEXT,
    role user_role NOT NULL DEFAULT 'reader',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
```

### `migrations/002_create_posts.sql`

```sql
CREATE TYPE post_status AS ENUM ('draft', 'published', 'archived');

CREATE TABLE posts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(200) NOT NULL,
    slug VARCHAR(220) UNIQUE NOT NULL,
    excerpt TEXT,
    content TEXT NOT NULL,
    cover_image_url TEXT,
    status post_status NOT NULL DEFAULT 'draft',
    author_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    view_count BIGINT NOT NULL DEFAULT 0,
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_posts_slug ON posts(slug);
CREATE INDEX idx_posts_author ON posts(author_id);
CREATE INDEX idx_posts_status ON posts(status);
CREATE INDEX idx_posts_published ON posts(published_at DESC) WHERE status = 'published';
```

### `migrations/003_create_comments.sql`

```sql
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content TEXT NOT NULL,
    post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    author_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    parent_id UUID REFERENCES comments(id) ON DELETE CASCADE,
    is_approved BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_comments_post ON comments(post_id);
CREATE INDEX idx_comments_author ON comments(author_id);
```

### `migrations/004_create_tags.sql`

```sql
CREATE TABLE tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(50) UNIQUE NOT NULL,
    slug VARCHAR(55) UNIQUE NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE post_tags (
    post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    tag_id UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)
);
```

---

## 10. การทดสอบ API

```bash
# 1. Register
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"john","email":"john@example.com","password":"password123"}'

# 2. Login
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"john@example.com","password":"password123"}'

# 3. Create Post (with token)
curl -X POST http://localhost:8080/api/posts \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"title":"My First Post","content":"This is the content of my first post which is long enough","tag_ids":[]}'

# 4. List Posts with search and pagination
curl "http://localhost:8080/api/posts?page=1&per_page=10&search=rust"

# 5. Publish Post
curl -X POST http://localhost:8080/api/posts/<post_id>/publish \
  -H "Authorization: Bearer <token>"
```

---

## สรุป Part 081

ใน Part นี้เราได้สร้าง Blog API ที่สมบูรณ์ด้วย:
1. **Models** สำหรับ User, Post, Comment, Tag พร้อม validation
2. **JWT Authentication** และ role-based access control
3. **Post Publishing Workflow** แบบ draft/published/archived
4. **Search และ Pagination** สำหรับ posts
5. **Full CRUD** สำหรับทุก resources
6. **Database migrations** ด้วย PostgreSQL

ใน **Part 082** เราจะสร้าง **E-commerce API** ที่ซับซ้อนยิ่งขึ้น

---

*[← Part 080](../part_080/README.md) | [Part 082: E-commerce API →](../part_082/README.md)*
