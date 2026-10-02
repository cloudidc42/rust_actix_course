# Part 031: SQLx กับ PostgreSQL 🗄️

## 🎯 เป้าหมายของ Part นี้

- ติดตั้งและ setup SQLx
- เชื่อมต่อ PostgreSQL
- CRUD operations ด้วย SQLx
- Prepared statements
- Connection pooling

---

## 1. Setup

### 1.1 Cargo.toml

```toml
[package]
name = "actix_db"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "chrono", "uuid"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
dotenv = "0.15"
env_logger = "0.11"
log = "0.4"
thiserror = "1"
```

### 1.2 ติดตั้ง sqlx-cli

```bash
cargo install sqlx-cli --no-default-features --features native-tls,postgres
```

### 1.3 Database Setup

```bash
# สร้าง PostgreSQL database
createdb myapp_db

# หรือด้วย psql
psql -c "CREATE DATABASE myapp_db;"

# .env file
cat > .env << EOF
DATABASE_URL=postgresql://postgres:password@localhost/myapp_db
RUST_LOG=debug
EOF
```

---

## 2. Database Connection

### 2.1 Connection Pool

```rust
// src/db.rs
use sqlx::PgPool;
use std::env;

pub async fn create_pool() -> Result<PgPool, sqlx::Error> {
    let database_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");

    PgPool::connect(&database_url).await
}

pub async fn create_pool_with_options() -> Result<PgPool, sqlx::Error> {
    let database_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");

    sqlx::postgres::PgPoolOptions::new()
        .max_connections(10)
        .min_connections(2)
        .acquire_timeout(std::time::Duration::from_secs(30))
        .idle_timeout(std::time::Duration::from_secs(600))
        .connect(&database_url)
        .await
}
```

---

## 3. Models และ Migrations

### 3.1 สร้าง Migration

```bash
sqlx migrate add create_users_table
sqlx migrate add create_posts_table
```

**migrations/20240101000001_create_users_table.sql:**
```sql
CREATE TABLE IF NOT EXISTS users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username    VARCHAR(50) UNIQUE NOT NULL,
    email       VARCHAR(255) UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    full_name   VARCHAR(100),
    avatar_url  TEXT,
    role        VARCHAR(20) NOT NULL DEFAULT 'user',
    is_active   BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

**migrations/20240101000002_create_posts_table.sql:**
```sql
CREATE TABLE IF NOT EXISTS posts (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title       VARCHAR(200) NOT NULL,
    slug        VARCHAR(200) UNIQUE NOT NULL,
    content     TEXT NOT NULL,
    summary     TEXT,
    status      VARCHAR(20) NOT NULL DEFAULT 'draft',
    view_count  INTEGER NOT NULL DEFAULT 0,
    tags        TEXT[] DEFAULT '{}',
    published_at TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_slug ON posts(slug);
CREATE INDEX idx_posts_status ON posts(status);
CREATE INDEX idx_posts_tags ON posts USING GIN(tags);
```

```bash
# รัน migrations
sqlx migrate run

# Revert migration
sqlx migrate revert
```

---

## 4. Models

### 4.1 User Model

```rust
// src/models/user.rs
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use chrono::{DateTime, Utc};

#[derive(Debug, Serialize, Deserialize, FromRow, Clone)]
pub struct User {
    pub id: Uuid,
    pub username: String,
    pub email: String,
    #[serde(skip_serializing)]  // อย่าส่ง password hash ออกไป
    pub password_hash: String,
    pub full_name: Option<String>,
    pub avatar_url: Option<String>,
    pub role: String,
    pub is_active: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct UserResponse {
    pub id: Uuid,
    pub username: String,
    pub email: String,
    pub full_name: Option<String>,
    pub avatar_url: Option<String>,
    pub role: String,
    pub is_active: bool,
    pub created_at: DateTime<Utc>,
}

impl From<User> for UserResponse {
    fn from(user: User) -> Self {
        Self {
            id: user.id,
            username: user.username,
            email: user.email,
            full_name: user.full_name,
            avatar_url: user.avatar_url,
            role: user.role,
            is_active: user.is_active,
            created_at: user.created_at,
        }
    }
}

#[derive(Debug, Deserialize)]
pub struct CreateUserRequest {
    pub username: String,
    pub email: String,
    pub password: String,
    pub full_name: Option<String>,
}

#[derive(Debug, Deserialize)]
pub struct UpdateUserRequest {
    pub full_name: Option<String>,
    pub avatar_url: Option<String>,
}
```

---

## 5. Repository Pattern

### 5.1 User Repository

```rust
// src/repositories/user_repository.rs
use sqlx::PgPool;
use uuid::Uuid;
use crate::models::user::{User, CreateUserRequest, UpdateUserRequest};

#[derive(Clone)]
pub struct UserRepository {
    pool: PgPool,
}

impl UserRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }

    // CREATE
    pub async fn create(
        &self,
        req: &CreateUserRequest,
        password_hash: &str,
    ) -> Result<User, sqlx::Error> {
        sqlx::query_as!(
            User,
            r#"
            INSERT INTO users (username, email, password_hash, full_name)
            VALUES ($1, $2, $3, $4)
            RETURNING *
            "#,
            req.username,
            req.email,
            password_hash,
            req.full_name
        )
        .fetch_one(&self.pool)
        .await
    }

    // READ - find by id
    pub async fn find_by_id(&self, id: Uuid) -> Result<Option<User>, sqlx::Error> {
        sqlx::query_as!(
            User,
            "SELECT * FROM users WHERE id = $1",
            id
        )
        .fetch_optional(&self.pool)
        .await
    }

    // READ - find by email
    pub async fn find_by_email(&self, email: &str) -> Result<Option<User>, sqlx::Error> {
        sqlx::query_as!(
            User,
            "SELECT * FROM users WHERE email = $1 AND is_active = true",
            email
        )
        .fetch_optional(&self.pool)
        .await
    }

    // READ - find by username
    pub async fn find_by_username(&self, username: &str) -> Result<Option<User>, sqlx::Error> {
        sqlx::query_as!(
            User,
            "SELECT * FROM users WHERE username = $1",
            username
        )
        .fetch_optional(&self.pool)
        .await
    }

    // READ - list all with pagination
    pub async fn list(
        &self,
        page: u32,
        per_page: u32,
        search: Option<&str>,
    ) -> Result<Vec<User>, sqlx::Error> {
        let offset = ((page - 1) * per_page) as i64;
        let limit = per_page as i64;

        if let Some(search) = search {
            let pattern = format!("%{}%", search);
            sqlx::query_as!(
                User,
                r#"
                SELECT * FROM users
                WHERE is_active = true
                  AND (username ILIKE $1 OR email ILIKE $1 OR full_name ILIKE $1)
                ORDER BY created_at DESC
                LIMIT $2 OFFSET $3
                "#,
                pattern,
                limit,
                offset
            )
            .fetch_all(&self.pool)
            .await
        } else {
            sqlx::query_as!(
                User,
                r#"
                SELECT * FROM users
                WHERE is_active = true
                ORDER BY created_at DESC
                LIMIT $1 OFFSET $2
                "#,
                limit,
                offset
            )
            .fetch_all(&self.pool)
            .await
        }
    }

    // READ - count
    pub async fn count(&self, search: Option<&str>) -> Result<i64, sqlx::Error> {
        if let Some(search) = search {
            let pattern = format!("%{}%", search);
            sqlx::query_scalar!(
                r#"
                SELECT COUNT(*) FROM users
                WHERE is_active = true
                  AND (username ILIKE $1 OR email ILIKE $1 OR full_name ILIKE $1)
                "#,
                pattern
            )
            .fetch_one(&self.pool)
            .await
            .map(|v| v.unwrap_or(0))
        } else {
            sqlx::query_scalar!(
                "SELECT COUNT(*) FROM users WHERE is_active = true"
            )
            .fetch_one(&self.pool)
            .await
            .map(|v| v.unwrap_or(0))
        }
    }

    // UPDATE
    pub async fn update(
        &self,
        id: Uuid,
        req: &UpdateUserRequest,
    ) -> Result<Option<User>, sqlx::Error> {
        sqlx::query_as!(
            User,
            r#"
            UPDATE users
            SET
                full_name = COALESCE($1, full_name),
                avatar_url = COALESCE($2, avatar_url)
            WHERE id = $3 AND is_active = true
            RETURNING *
            "#,
            req.full_name,
            req.avatar_url,
            id
        )
        .fetch_optional(&self.pool)
        .await
    }

    // DELETE (soft delete)
    pub async fn delete(&self, id: Uuid) -> Result<bool, sqlx::Error> {
        let result = sqlx::query!(
            "UPDATE users SET is_active = false WHERE id = $1",
            id
        )
        .execute(&self.pool)
        .await?;

        Ok(result.rows_affected() > 0)
    }

    // Hard delete (ใช้ระวัง!)
    pub async fn hard_delete(&self, id: Uuid) -> Result<bool, sqlx::Error> {
        let result = sqlx::query!(
            "DELETE FROM users WHERE id = $1",
            id
        )
        .execute(&self.pool)
        .await?;

        Ok(result.rows_affected() > 0)
    }
}
```

---

## 6. Handlers

```rust
// src/handlers/users.rs
use actix_web::{get, post, put, delete, web, HttpResponse};
use uuid::Uuid;
use crate::repositories::user_repository::UserRepository;
use crate::models::user::{CreateUserRequest, UpdateUserRequest, UserResponse};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct ListQuery {
    page: Option<u32>,
    per_page: Option<u32>,
    search: Option<String>,
}

#[derive(Serialize)]
struct PaginatedResponse<T: Serialize> {
    data: Vec<T>,
    total: i64,
    page: u32,
    per_page: u32,
    total_pages: i64,
}

#[get("/users")]
async fn list_users(
    repo: web::Data<UserRepository>,
    query: web::Query<ListQuery>,
) -> HttpResponse {
    let page = query.page.unwrap_or(1).max(1);
    let per_page = query.per_page.unwrap_or(20).min(100);
    let search = query.search.as_deref();

    let (users, total) = tokio::join!(
        repo.list(page, per_page, search),
        repo.count(search)
    );

    match (users, total) {
        (Ok(users), Ok(total)) => {
            let total_pages = (total as f64 / per_page as f64).ceil() as i64;
            let data: Vec<UserResponse> = users.into_iter().map(|u| u.into()).collect();

            HttpResponse::Ok().json(PaginatedResponse {
                data,
                total,
                page,
                per_page,
                total_pages,
            })
        },
        _ => HttpResponse::InternalServerError().json(serde_json::json!({
            "error": "Database error"
        }))
    }
}

#[get("/users/{id}")]
async fn get_user(
    repo: web::Data<UserRepository>,
    path: web::Path<Uuid>,
) -> HttpResponse {
    let id = path.into_inner();

    match repo.find_by_id(id).await {
        Ok(Some(user)) => HttpResponse::Ok().json(UserResponse::from(user)),
        Ok(None) => HttpResponse::NotFound().json(serde_json::json!({
            "error": "User not found"
        })),
        Err(e) => {
            log::error!("Database error: {}", e);
            HttpResponse::InternalServerError().json(serde_json::json!({
                "error": "Internal server error"
            }))
        }
    }
}

#[post("/users")]
async fn create_user(
    repo: web::Data<UserRepository>,
    body: web::Json<CreateUserRequest>,
) -> HttpResponse {
    // Check if email exists
    match repo.find_by_email(&body.email).await {
        Ok(Some(_)) => {
            return HttpResponse::Conflict().json(serde_json::json!({
                "error": "Email already exists"
            }));
        },
        Err(e) => {
            log::error!("DB error: {}", e);
            return HttpResponse::InternalServerError().json(serde_json::json!({
                "error": "Internal error"
            }));
        },
        Ok(None) => {}
    }

    // Hash password (ใช้ argon2 ใน production)
    let password_hash = format!("hashed_{}", body.password);

    match repo.create(&body, &password_hash).await {
        Ok(user) => HttpResponse::Created().json(UserResponse::from(user)),
        Err(e) => {
            log::error!("Create user error: {}", e);
            HttpResponse::InternalServerError().json(serde_json::json!({
                "error": "Failed to create user"
            }))
        }
    }
}

#[put("/users/{id}")]
async fn update_user(
    repo: web::Data<UserRepository>,
    path: web::Path<Uuid>,
    body: web::Json<UpdateUserRequest>,
) -> HttpResponse {
    let id = path.into_inner();

    match repo.update(id, &body).await {
        Ok(Some(user)) => HttpResponse::Ok().json(UserResponse::from(user)),
        Ok(None) => HttpResponse::NotFound().json(serde_json::json!({
            "error": "User not found"
        })),
        Err(e) => {
            log::error!("Update error: {}", e);
            HttpResponse::InternalServerError().json(serde_json::json!({
                "error": "Failed to update user"
            }))
        }
    }
}

#[delete("/users/{id}")]
async fn delete_user(
    repo: web::Data<UserRepository>,
    path: web::Path<Uuid>,
) -> HttpResponse {
    let id = path.into_inner();

    match repo.delete(id).await {
        Ok(true) => HttpResponse::NoContent().finish(),
        Ok(false) => HttpResponse::NotFound().json(serde_json::json!({
            "error": "User not found"
        })),
        Err(e) => {
            log::error!("Delete error: {}", e);
            HttpResponse::InternalServerError().json(serde_json::json!({
                "error": "Failed to delete user"
            }))
        }
    }
}

pub fn configure(cfg: &mut web::ServiceConfig) {
    cfg
        .service(list_users)
        .service(get_user)
        .service(create_user)
        .service(update_user)
        .service(delete_user);
}
```

---

## 7. Main.rs สมบูรณ์

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware::Logger};
use sqlx::PgPool;
use std::env;

mod db;
mod models;
mod repositories;
mod handlers;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv::dotenv().ok();
    env_logger::init_from_env(env_logger::Env::default().default_filter_or("info"));

    let pool = db::create_pool_with_options()
        .await
        .expect("Failed to create database pool");

    // Run migrations
    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .expect("Failed to run migrations");

    log::info!("Database connected and migrations applied");

    let pool_data = web::Data::new(pool);

    HttpServer::new(move || {
        let user_repo = web::Data::new(
            repositories::user_repository::UserRepository::new(
                pool_data.get_ref().clone()
            )
        );

        App::new()
            .wrap(Logger::new("%r → %s (%Dms)"))
            .app_data(pool_data.clone())
            .app_data(user_repo)
            .service(
                web::scope("/api/v1")
                    .configure(handlers::users::configure)
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## 8. ทดสอบ

```bash
# Create user
curl -X POST http://localhost:8080/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{
    "username": "johndoe",
    "email": "john@example.com",
    "password": "secret123",
    "full_name": "John Doe"
  }'

# List users
curl http://localhost:8080/api/v1/users?page=1&per_page=10

# Get user by ID
curl http://localhost:8080/api/v1/users/{uuid}

# Update user
curl -X PUT http://localhost:8080/api/v1/users/{uuid} \
  -H "Content-Type: application/json" \
  -d '{"full_name": "John Updated Doe"}'

# Delete user
curl -X DELETE http://localhost:8080/api/v1/users/{uuid}
```

---

## 9. สรุปและ Exercises

### 9.1 สิ่งที่เรียนรู้

✅ SQLx setup กับ PostgreSQL  
✅ query_as! macro  
✅ Connection pooling  
✅ Repository pattern  
✅ CRUD operations  
✅ Pagination  
✅ Soft delete  

### 9.2 Exercise

**Exercise: Posts Table**
```rust
// เพิ่ม PostRepository กับ:
// - find_by_user_id
// - find_by_slug  
// - list with filters (status, tags, date range)
// - increment_view_count
// - publish / unpublish
```

---

*[← Part 030: CORS](../part_030/README.md) | [Part 032: Database Migrations →](../part_032/README.md)*
