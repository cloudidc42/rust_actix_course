# Part 038: Repository Pattern Advanced

## สารบัญ
- [Generic Repository Trait](#generic-repository-trait)
- [Concrete Implementations](#concrete-implementations)
- [Dependency Injection ด้วย Traits](#dependency-injection-ด้วย-traits)
- [Mock Repositories สำหรับ Testing](#mock-repositories-สำหรับ-testing)
- [Multiple Database Support](#multiple-database-support)
- [Read/Write Splitting](#readwrite-splitting)
- [Caching Layer ใน Repository](#caching-layer-ใน-repository)
- [Complete User + Post Repositories](#complete-user--post-repositories)

---

## Generic Repository Trait

### Base Repository Interface

```rust
// src/repositories/traits.rs
use async_trait::async_trait;
use serde::{de::DeserializeOwned, Serialize};
use uuid::Uuid;

// Generic CRUD operations
#[async_trait]
pub trait Repository<T, CreateDto, UpdateDto>
where
    T: Send + Sync + 'static,
    CreateDto: Send + Sync + 'static,
    UpdateDto: Send + Sync + 'static,
{
    type Error: std::error::Error + Send + Sync + 'static;
    
    async fn find_by_id(&self, id: Uuid) -> Result<Option<T>, Self::Error>;
    async fn find_all(&self, page: i64, per_page: i64) -> Result<Vec<T>, Self::Error>;
    async fn create(&self, dto: CreateDto) -> Result<T, Self::Error>;
    async fn update(&self, id: Uuid, dto: UpdateDto) -> Result<Option<T>, Self::Error>;
    async fn delete(&self, id: Uuid) -> Result<bool, Self::Error>;
    async fn count(&self) -> Result<i64, Self::Error>;
    async fn exists(&self, id: Uuid) -> Result<bool, Self::Error>;
}

// Trait สำหรับ soft delete
#[async_trait]
pub trait SoftDeletable {
    type Error: std::error::Error + Send + Sync + 'static;
    
    async fn soft_delete(&self, id: Uuid) -> Result<bool, Self::Error>;
    async fn restore(&self, id: Uuid) -> Result<bool, Self::Error>;
    async fn find_deleted(&self, page: i64, per_page: i64) -> Result<Vec<Self::Item>, Self::Error>
    where
        Self::Item: Send + Sync;
    
    type Item;
}

// Trait สำหรับ searchable entities
#[async_trait]
pub trait Searchable<T> {
    type Error: std::error::Error + Send + Sync + 'static;
    
    async fn search(&self, query: &str, limit: i64) -> Result<Vec<T>, Self::Error>;
}

// Trait สำหรับ paginated results
#[derive(Debug, serde::Serialize)]
pub struct Page<T> {
    pub data: Vec<T>,
    pub total: i64,
    pub page: i64,
    pub per_page: i64,
    pub total_pages: i64,
    pub has_next: bool,
    pub has_prev: bool,
}

impl<T> Page<T> {
    pub fn new(data: Vec<T>, total: i64, page: i64, per_page: i64) -> Self {
        let total_pages = (total + per_page - 1) / per_page;
        Self {
            has_next: page < total_pages,
            has_prev: page > 1,
            data,
            total,
            page,
            per_page,
            total_pages,
        }
    }
    
    pub fn empty(page: i64, per_page: i64) -> Self {
        Self {
            data: Vec::new(),
            total: 0,
            page,
            per_page,
            total_pages: 0,
            has_next: false,
            has_prev: false,
        }
    }
}

#[async_trait]
pub trait Paginated<T, Filter> {
    type Error: std::error::Error + Send + Sync + 'static;
    
    async fn find_paginated(
        &self,
        filter: Filter,
        page: i64,
        per_page: i64,
    ) -> Result<Page<T>, Self::Error>;
}
```

---

## Concrete Implementations

### User Repository Implementation

```rust
// src/repositories/user_repository.rs
use async_trait::async_trait;
use sqlx::PgPool;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use crate::repositories::traits::{Repository, Page, Paginated};
use crate::errors::AppError;

#[derive(Debug, Clone, Serialize, sqlx::FromRow)]
pub struct UserEntity {
    pub id: Uuid,
    pub email: String,
    pub username: String,
    pub password_hash: String,
    pub display_name: String,
    pub bio: Option<String>,
    pub avatar_url: Option<String>,
    pub website_url: Option<String>,
    pub is_active: bool,
    pub is_admin: bool,
    pub is_verified: bool,
    pub last_login_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub deleted_at: Option<DateTime<Utc>>,
}

#[derive(Debug, Deserialize)]
pub struct CreateUserDto {
    pub email: String,
    pub username: String,
    pub password_hash: String,
    pub display_name: String,
}

#[derive(Debug, Deserialize, Default)]
pub struct UpdateUserDto {
    pub display_name: Option<String>,
    pub bio: Option<String>,
    pub avatar_url: Option<String>,
    pub website_url: Option<String>,
}

#[derive(Debug, Deserialize, Default)]
pub struct UserFilter {
    pub search: Option<String>,
    pub is_active: Option<bool>,
    pub is_admin: Option<bool>,
}

// Concrete repository สำหรับ PostgreSQL
pub struct PgUserRepository {
    pool: PgPool,
}

impl PgUserRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
    
    pub async fn find_by_email(&self, email: &str) -> Result<Option<UserEntity>, AppError> {
        sqlx::query_as!(
            UserEntity,
            r#"
            SELECT id, email, username, password_hash, display_name,
                bio, avatar_url, website_url, is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            FROM users
            WHERE email = $1 AND deleted_at IS NULL
            "#,
            email.to_lowercase()
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(AppError::Database)
    }
    
    pub async fn find_by_username(&self, username: &str) -> Result<Option<UserEntity>, AppError> {
        sqlx::query_as!(
            UserEntity,
            r#"
            SELECT id, email, username, password_hash, display_name,
                bio, avatar_url, website_url, is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            FROM users
            WHERE username = $1 AND deleted_at IS NULL
            "#,
            username.to_lowercase()
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(AppError::Database)
    }
    
    pub async fn update_last_login(&self, id: Uuid) -> Result<(), AppError> {
        sqlx::query!(
            "UPDATE users SET last_login_at = NOW() WHERE id = $1",
            id
        )
        .execute(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        Ok(())
    }
}

#[async_trait]
impl Repository<UserEntity, CreateUserDto, UpdateUserDto> for PgUserRepository {
    type Error = AppError;
    
    async fn find_by_id(&self, id: Uuid) -> Result<Option<UserEntity>, AppError> {
        sqlx::query_as!(
            UserEntity,
            r#"
            SELECT id, email, username, password_hash, display_name,
                bio, avatar_url, website_url, is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            FROM users
            WHERE id = $1 AND deleted_at IS NULL
            "#,
            id
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(AppError::Database)
    }
    
    async fn find_all(&self, page: i64, per_page: i64) -> Result<Vec<UserEntity>, AppError> {
        let offset = (page - 1) * per_page;
        sqlx::query_as!(
            UserEntity,
            r#"
            SELECT id, email, username, password_hash, display_name,
                bio, avatar_url, website_url, is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            FROM users
            WHERE deleted_at IS NULL
            ORDER BY created_at DESC
            LIMIT $1 OFFSET $2
            "#,
            per_page,
            offset
        )
        .fetch_all(&self.pool)
        .await
        .map_err(AppError::Database)
    }
    
    async fn create(&self, dto: CreateUserDto) -> Result<UserEntity, AppError> {
        sqlx::query_as!(
            UserEntity,
            r#"
            INSERT INTO users (email, username, password_hash, display_name)
            VALUES ($1, $2, $3, $4)
            RETURNING id, email, username, password_hash, display_name,
                bio, avatar_url, website_url, is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            "#,
            dto.email.to_lowercase(),
            dto.username.to_lowercase(),
            dto.password_hash,
            dto.display_name
        )
        .fetch_one(&self.pool)
        .await
        .map_err(AppError::Database)
    }
    
    async fn update(&self, id: Uuid, dto: UpdateUserDto) -> Result<Option<UserEntity>, AppError> {
        sqlx::query_as!(
            UserEntity,
            r#"
            UPDATE users
            SET
                display_name = COALESCE($2, display_name),
                bio = COALESCE($3, bio),
                avatar_url = COALESCE($4, avatar_url),
                website_url = COALESCE($5, website_url),
                updated_at = NOW()
            WHERE id = $1 AND deleted_at IS NULL
            RETURNING id, email, username, password_hash, display_name,
                bio, avatar_url, website_url, is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            "#,
            id,
            dto.display_name,
            dto.bio,
            dto.avatar_url,
            dto.website_url
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(AppError::Database)
    }
    
    async fn delete(&self, id: Uuid) -> Result<bool, AppError> {
        let result = sqlx::query!(
            "UPDATE users SET deleted_at = NOW() WHERE id = $1 AND deleted_at IS NULL",
            id
        )
        .execute(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        Ok(result.rows_affected() > 0)
    }
    
    async fn count(&self) -> Result<i64, AppError> {
        sqlx::query_scalar!(
            r#"SELECT COUNT(*) as "count!" FROM users WHERE deleted_at IS NULL"#
        )
        .fetch_one(&self.pool)
        .await
        .map_err(AppError::Database)
    }
    
    async fn exists(&self, id: Uuid) -> Result<bool, AppError> {
        sqlx::query_scalar!(
            r#"
            SELECT EXISTS(
                SELECT 1 FROM users WHERE id = $1 AND deleted_at IS NULL
            ) as "exists!"
            "#,
            id
        )
        .fetch_one(&self.pool)
        .await
        .map_err(AppError::Database)
    }
}

#[async_trait]
impl Paginated<UserEntity, UserFilter> for PgUserRepository {
    type Error = AppError;
    
    async fn find_paginated(
        &self,
        filter: UserFilter,
        page: i64,
        per_page: i64,
    ) -> Result<Page<UserEntity>, AppError> {
        let offset = (page - 1) * per_page;
        
        let total = sqlx::query_scalar!(
            r#"
            SELECT COUNT(*) as "count!"
            FROM users
            WHERE deleted_at IS NULL
                AND ($1::boolean IS NULL OR is_active = $1)
                AND ($2::boolean IS NULL OR is_admin = $2)
                AND ($3::text IS NULL OR (
                    username ILIKE '%' || $3 || '%'
                    OR email ILIKE '%' || $3 || '%'
                    OR display_name ILIKE '%' || $3 || '%'
                ))
            "#,
            filter.is_active,
            filter.is_admin,
            filter.search
        )
        .fetch_one(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        let data = sqlx::query_as!(
            UserEntity,
            r#"
            SELECT id, email, username, password_hash, display_name,
                bio, avatar_url, website_url, is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            FROM users
            WHERE deleted_at IS NULL
                AND ($1::boolean IS NULL OR is_active = $1)
                AND ($2::boolean IS NULL OR is_admin = $2)
                AND ($3::text IS NULL OR (
                    username ILIKE '%' || $3 || '%'
                    OR email ILIKE '%' || $3 || '%'
                    OR display_name ILIKE '%' || $3 || '%'
                ))
            ORDER BY created_at DESC
            LIMIT $4 OFFSET $5
            "#,
            filter.is_active,
            filter.is_admin,
            filter.search,
            per_page,
            offset
        )
        .fetch_all(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        Ok(Page::new(data, total, page, per_page))
    }
}
```

---

## Dependency Injection ด้วย Traits

### Service Layer ที่ inject Repository

```rust
// src/services/user_service.rs
use async_trait::async_trait;
use std::sync::Arc;

// Trait สำหรับ User Repository (ใช้สำหรับ DI)
#[async_trait]
pub trait UserRepository: Send + Sync {
    async fn find_by_id(&self, id: Uuid) -> Result<Option<UserEntity>, AppError>;
    async fn find_by_email(&self, email: &str) -> Result<Option<UserEntity>, AppError>;
    async fn create(&self, dto: CreateUserDto) -> Result<UserEntity, AppError>;
    async fn update(&self, id: Uuid, dto: UpdateUserDto) -> Result<Option<UserEntity>, AppError>;
    async fn delete(&self, id: Uuid) -> Result<bool, AppError>;
    async fn exists(&self, id: Uuid) -> Result<bool, AppError>;
}

// Service ที่ใช้ repository ผ่าน trait (ไม่ depend on concrete impl)
pub struct UserService<R: UserRepository> {
    repo: Arc<R>,
}

impl<R: UserRepository> UserService<R> {
    pub fn new(repo: Arc<R>) -> Self {
        Self { repo }
    }
    
    pub async fn get_user(&self, id: Uuid) -> Result<UserEntity, AppError> {
        self.repo.find_by_id(id)
            .await?
            .ok_or(AppError::NotFound(format!("User {} not found", id)))
    }
    
    pub async fn register(
        &self,
        email: String,
        username: String,
        password: String,
    ) -> Result<UserEntity, AppError> {
        // ตรวจสอบ email ซ้ำ
        if let Some(_) = self.repo.find_by_email(&email).await? {
            return Err(AppError::Conflict("Email already registered".to_string()));
        }
        
        // Hash password
        let password_hash = hash_password(&password)?;
        
        // สร้าง user
        let user = self.repo.create(CreateUserDto {
            email,
            username,
            password_hash,
            display_name: String::new(),
        }).await?;
        
        Ok(user)
    }
    
    pub async fn update_profile(
        &self,
        id: Uuid,
        dto: UpdateUserDto,
    ) -> Result<UserEntity, AppError> {
        // ตรวจสอบว่า user มีอยู่
        if !self.repo.exists(id).await? {
            return Err(AppError::NotFound("User not found".to_string()));
        }
        
        self.repo.update(id, dto)
            .await?
            .ok_or(AppError::NotFound("User not found".to_string()))
    }
}

fn hash_password(password: &str) -> Result<String, AppError> {
    // ใช้ bcrypt หรือ argon2
    bcrypt::hash(password, bcrypt::DEFAULT_COST)
        .map_err(|e| AppError::Internal(format!("Password hash error: {}", e)))
}

// Registration ใน main.rs
pub fn configure_services(pool: PgPool) -> UserService<PgUserRepository> {
    let repo = Arc::new(PgUserRepository::new(pool));
    UserService::new(repo)
}
```

### Actix-web Integration ด้วย DI

```rust
// src/handlers/users.rs
use actix_web::{get, post, put, delete, web, HttpResponse};

#[get("/api/v1/users/{id}")]
pub async fn get_user(
    id: web::Path<Uuid>,
    service: web::Data<UserService<PgUserRepository>>,
) -> HttpResponse {
    match service.get_user(*id).await {
        Ok(user) => HttpResponse::Ok().json(user),
        Err(AppError::NotFound(msg)) => HttpResponse::NotFound().json(serde_json::json!({
            "error": msg
        })),
        Err(e) => {
            log::error!("Get user error: {}", e);
            HttpResponse::InternalServerError().json(serde_json::json!({
                "error": "Internal server error"
            }))
        }
    }
}
```

---

## Mock Repositories สำหรับ Testing

### Mock Implementation

```rust
// src/repositories/mocks.rs (หรือ tests/)
use async_trait::async_trait;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

// Mock Repository สำหรับ Testing
pub struct MockUserRepository {
    users: Arc<Mutex<HashMap<Uuid, UserEntity>>>,
    // เก็บ calls เพื่อ verify
    call_log: Arc<Mutex<Vec<String>>>,
}

impl MockUserRepository {
    pub fn new() -> Self {
        Self {
            users: Arc::new(Mutex::new(HashMap::new())),
            call_log: Arc::new(Mutex::new(Vec::new())),
        }
    }
    
    pub fn with_user(self, user: UserEntity) -> Self {
        self.users.lock().unwrap().insert(user.id, user);
        self
    }
    
    pub fn get_call_log(&self) -> Vec<String> {
        self.call_log.lock().unwrap().clone()
    }
    
    pub fn was_called(&self, method: &str) -> bool {
        self.call_log.lock().unwrap().iter().any(|c| c == method)
    }
}

#[async_trait]
impl UserRepository for MockUserRepository {
    async fn find_by_id(&self, id: Uuid) -> Result<Option<UserEntity>, AppError> {
        self.call_log.lock().unwrap().push("find_by_id".to_string());
        let users = self.users.lock().unwrap();
        Ok(users.get(&id).cloned())
    }
    
    async fn find_by_email(&self, email: &str) -> Result<Option<UserEntity>, AppError> {
        self.call_log.lock().unwrap().push("find_by_email".to_string());
        let users = self.users.lock().unwrap();
        Ok(users.values().find(|u| u.email == email).cloned())
    }
    
    async fn create(&self, dto: CreateUserDto) -> Result<UserEntity, AppError> {
        self.call_log.lock().unwrap().push("create".to_string());
        
        let user = UserEntity {
            id: Uuid::new_v4(),
            email: dto.email,
            username: dto.username,
            password_hash: dto.password_hash,
            display_name: dto.display_name,
            bio: None,
            avatar_url: None,
            website_url: None,
            is_active: true,
            is_admin: false,
            is_verified: false,
            last_login_at: None,
            created_at: chrono::Utc::now(),
            updated_at: chrono::Utc::now(),
            deleted_at: None,
        };
        
        self.users.lock().unwrap().insert(user.id, user.clone());
        Ok(user)
    }
    
    async fn update(&self, id: Uuid, dto: UpdateUserDto) -> Result<Option<UserEntity>, AppError> {
        self.call_log.lock().unwrap().push("update".to_string());
        
        let mut users = self.users.lock().unwrap();
        if let Some(user) = users.get_mut(&id) {
            if let Some(name) = dto.display_name {
                user.display_name = name;
            }
            if let Some(bio) = dto.bio {
                user.bio = Some(bio);
            }
            Ok(Some(user.clone()))
        } else {
            Ok(None)
        }
    }
    
    async fn delete(&self, id: Uuid) -> Result<bool, AppError> {
        self.call_log.lock().unwrap().push("delete".to_string());
        let mut users = self.users.lock().unwrap();
        Ok(users.remove(&id).is_some())
    }
    
    async fn exists(&self, id: Uuid) -> Result<bool, AppError> {
        self.call_log.lock().unwrap().push("exists".to_string());
        Ok(self.users.lock().unwrap().contains_key(&id))
    }
}

// Unit Tests
#[cfg(test)]
mod tests {
    use super::*;
    
    fn make_test_user() -> UserEntity {
        UserEntity {
            id: Uuid::new_v4(),
            email: "test@example.com".to_string(),
            username: "testuser".to_string(),
            password_hash: "hashed".to_string(),
            display_name: "Test User".to_string(),
            bio: None,
            avatar_url: None,
            website_url: None,
            is_active: true,
            is_admin: false,
            is_verified: false,
            last_login_at: None,
            created_at: chrono::Utc::now(),
            updated_at: chrono::Utc::now(),
            deleted_at: None,
        }
    }
    
    #[tokio::test]
    async fn test_register_new_user() {
        let repo = Arc::new(MockUserRepository::new());
        let service = UserService::new(repo.clone());
        
        let result = service.register(
            "new@example.com".to_string(),
            "newuser".to_string(),
            "password123".to_string(),
        ).await;
        
        assert!(result.is_ok());
        let user = result.unwrap();
        assert_eq!(user.email, "new@example.com");
        assert!(repo.was_called("create"));
    }
    
    #[tokio::test]
    async fn test_register_duplicate_email() {
        let existing_user = make_test_user();
        let repo = Arc::new(MockUserRepository::new().with_user(existing_user));
        let service = UserService::new(repo);
        
        let result = service.register(
            "test@example.com".to_string(),  // Same email
            "otheruser".to_string(),
            "password123".to_string(),
        ).await;
        
        assert!(result.is_err());
        matches!(result.unwrap_err(), AppError::Conflict(_));
    }
    
    #[tokio::test]
    async fn test_get_nonexistent_user() {
        let repo = Arc::new(MockUserRepository::new());
        let service = UserService::new(repo);
        
        let result = service.get_user(Uuid::new_v4()).await;
        
        assert!(result.is_err());
        matches!(result.unwrap_err(), AppError::NotFound(_));
    }
}
```

---

## Read/Write Splitting

### Repository ที่แยก Read/Write Pool

```rust
// src/repositories/read_write.rs
use sqlx::PgPool;

pub struct ReadWriteUserRepository {
    write_pool: PgPool,
    read_pool: PgPool,
}

impl ReadWriteUserRepository {
    pub fn new(write_pool: PgPool, read_pool: PgPool) -> Self {
        Self { write_pool, read_pool }
    }
}

#[async_trait]
impl UserRepository for ReadWriteUserRepository {
    // Reads → ใช้ read_pool (replica)
    async fn find_by_id(&self, id: Uuid) -> Result<Option<UserEntity>, AppError> {
        sqlx::query_as!(
            UserEntity,
            r#"
            SELECT id, email, username, password_hash, display_name,
                bio, avatar_url, website_url, is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            FROM users
            WHERE id = $1 AND deleted_at IS NULL
            "#,
            id
        )
        .fetch_optional(&self.read_pool)  // <-- read pool
        .await
        .map_err(AppError::Database)
    }
    
    async fn find_by_email(&self, email: &str) -> Result<Option<UserEntity>, AppError> {
        sqlx::query_as!(
            UserEntity,
            r#"
            SELECT id, email, username, password_hash, display_name,
                bio, avatar_url, website_url, is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            FROM users
            WHERE email = $1 AND deleted_at IS NULL
            "#,
            email.to_lowercase()
        )
        .fetch_optional(&self.read_pool)  // <-- read pool
        .await
        .map_err(AppError::Database)
    }
    
    // Writes → ใช้ write_pool (primary)
    async fn create(&self, dto: CreateUserDto) -> Result<UserEntity, AppError> {
        sqlx::query_as!(
            UserEntity,
            r#"
            INSERT INTO users (email, username, password_hash, display_name)
            VALUES ($1, $2, $3, $4)
            RETURNING id, email, username, password_hash, display_name,
                bio, avatar_url, website_url, is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            "#,
            dto.email.to_lowercase(),
            dto.username,
            dto.password_hash,
            dto.display_name
        )
        .fetch_one(&self.write_pool)  // <-- write pool
        .await
        .map_err(AppError::Database)
    }
    
    async fn update(&self, id: Uuid, dto: UpdateUserDto) -> Result<Option<UserEntity>, AppError> {
        sqlx::query_as!(
            UserEntity,
            r#"
            UPDATE users
            SET
                display_name = COALESCE($2, display_name),
                bio = COALESCE($3, bio),
                updated_at = NOW()
            WHERE id = $1 AND deleted_at IS NULL
            RETURNING id, email, username, password_hash, display_name,
                bio, avatar_url, website_url, is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            "#,
            id,
            dto.display_name,
            dto.bio
        )
        .fetch_optional(&self.write_pool)  // <-- write pool
        .await
        .map_err(AppError::Database)
    }
    
    async fn delete(&self, id: Uuid) -> Result<bool, AppError> {
        let result = sqlx::query!(
            "UPDATE users SET deleted_at = NOW() WHERE id = $1 AND deleted_at IS NULL",
            id
        )
        .execute(&self.write_pool)  // <-- write pool
        .await
        .map_err(AppError::Database)?;
        
        Ok(result.rows_affected() > 0)
    }
    
    async fn exists(&self, id: Uuid) -> Result<bool, AppError> {
        sqlx::query_scalar!(
            r#"
            SELECT EXISTS(
                SELECT 1 FROM users WHERE id = $1 AND deleted_at IS NULL
            ) as "exists!"
            "#,
            id
        )
        .fetch_one(&self.read_pool)  // <-- read pool
        .await
        .map_err(AppError::Database)
    }
}
```

---

## Caching Layer ใน Repository

### Cached Repository Wrapper

```rust
// src/repositories/cached.rs
use std::sync::Arc;
use deadpool_redis::Pool as RedisPool;
use crate::cache::RedisClient;

pub struct CachedUserRepository<R: UserRepository> {
    inner: Arc<R>,
    cache: RedisClient,
    ttl: u64,
}

impl<R: UserRepository + 'static> CachedUserRepository<R> {
    pub fn new(inner: Arc<R>, redis_pool: RedisPool, ttl: u64) -> Self {
        Self {
            inner,
            cache: RedisClient::new(redis_pool),
            ttl,
        }
    }
    
    fn user_cache_key(id: Uuid) -> String {
        format!("user:{}", id)
    }
    
    fn email_cache_key(email: &str) -> String {
        format!("user:email:{}", email)
    }
}

#[async_trait]
impl<R: UserRepository + Send + Sync + 'static> UserRepository for CachedUserRepository<R> {
    async fn find_by_id(&self, id: Uuid) -> Result<Option<UserEntity>, AppError> {
        let cache_key = Self::user_cache_key(id);
        
        // Check cache
        if let Ok(Some(user)) = self.cache.get_json::<UserEntity>(&cache_key).await {
            return Ok(Some(user));
        }
        
        // Fetch from DB
        let user = self.inner.find_by_id(id).await?;
        
        // Cache if found
        if let Some(ref u) = user {
            self.cache.set_json(&cache_key, u, Some(self.ttl))
                .await
                .unwrap_or_else(|e| log::error!("Cache error: {}", e));
        }
        
        Ok(user)
    }
    
    async fn find_by_email(&self, email: &str) -> Result<Option<UserEntity>, AppError> {
        let cache_key = Self::email_cache_key(email);
        
        if let Ok(Some(user)) = self.cache.get_json::<UserEntity>(&cache_key).await {
            return Ok(Some(user));
        }
        
        let user = self.inner.find_by_email(email).await?;
        
        if let Some(ref u) = user {
            self.cache.set_json(&cache_key, u, Some(self.ttl))
                .await
                .unwrap_or_else(|e| log::error!("Cache error: {}", e));
        }
        
        Ok(user)
    }
    
    async fn create(&self, dto: CreateUserDto) -> Result<UserEntity, AppError> {
        self.inner.create(dto).await
    }
    
    async fn update(&self, id: Uuid, dto: UpdateUserDto) -> Result<Option<UserEntity>, AppError> {
        let result = self.inner.update(id, dto).await?;
        
        // Invalidate cache
        if result.is_some() {
            let key = Self::user_cache_key(id);
            self.cache.delete(&key).await.ok();
        }
        
        Ok(result)
    }
    
    async fn delete(&self, id: Uuid) -> Result<bool, AppError> {
        let result = self.inner.delete(id).await?;
        
        // Invalidate cache
        if result {
            let key = Self::user_cache_key(id);
            self.cache.delete(&key).await.ok();
        }
        
        Ok(result)
    }
    
    async fn exists(&self, id: Uuid) -> Result<bool, AppError> {
        self.inner.exists(id).await
    }
}
```

---

## Complete User + Post Repositories

### Full Repository Setup

```rust
// src/repositories/mod.rs

pub mod traits;
pub mod user_repository;
pub mod post_repository;
pub mod cached;
pub mod read_write;

#[cfg(test)]
pub mod mocks;

pub use user_repository::{PgUserRepository, UserEntity, CreateUserDto, UpdateUserDto, UserFilter};
pub use post_repository::{PgPostRepository, PostEntity, CreatePostDto, UpdatePostDto, PostFilter};

// Factory functions
pub fn create_user_repo(pool: sqlx::PgPool) -> PgUserRepository {
    PgUserRepository::new(pool)
}

pub fn create_cached_user_repo(
    pool: sqlx::PgPool,
    redis_pool: deadpool_redis::Pool,
    ttl: u64,
) -> CachedUserRepository<PgUserRepository> {
    let inner = std::sync::Arc::new(PgUserRepository::new(pool));
    CachedUserRepository::new(inner, redis_pool, ttl)
}
```

### Complete Post Repository

```rust
// src/repositories/post_repository.rs (ครบชุด)
use async_trait::async_trait;
use sqlx::PgPool;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use crate::repositories::traits::{Repository, Page, Paginated};
use crate::errors::AppError;

#[derive(Debug, Clone, Serialize, sqlx::FromRow)]
pub struct PostEntity {
    pub id: Uuid,
    pub title: String,
    pub slug: String,
    pub content: String,
    pub excerpt: Option<String>,
    pub cover_image_url: Option<String>,
    pub status: String,
    pub published_at: Option<DateTime<Utc>>,
    pub author_id: Uuid,
    pub category_id: Option<Uuid>,
    pub view_count: i32,
    pub like_count: i32,
    pub comment_count: i32,
    pub meta_title: Option<String>,
    pub meta_description: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub deleted_at: Option<DateTime<Utc>>,
}

#[derive(Debug, Deserialize)]
pub struct CreatePostDto {
    pub title: String,
    pub content: String,
    pub excerpt: Option<String>,
    pub cover_image_url: Option<String>,
    pub category_id: Option<Uuid>,
    pub author_id: Uuid,
    pub status: Option<String>,
    pub meta_title: Option<String>,
    pub meta_description: Option<String>,
}

#[derive(Debug, Deserialize, Default)]
pub struct UpdatePostDto {
    pub title: Option<String>,
    pub content: Option<String>,
    pub excerpt: Option<String>,
    pub cover_image_url: Option<String>,
    pub category_id: Option<Uuid>,
    pub status: Option<String>,
    pub meta_title: Option<String>,
    pub meta_description: Option<String>,
}

#[derive(Debug, Deserialize, Default)]
pub struct PostFilter {
    pub search: Option<String>,
    pub status: Option<String>,
    pub author_id: Option<Uuid>,
    pub category_id: Option<Uuid>,
}

pub struct PgPostRepository {
    pool: PgPool,
}

impl PgPostRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
    
    pub async fn find_by_slug(&self, slug: &str) -> Result<Option<PostEntity>, AppError> {
        sqlx::query_as!(
            PostEntity,
            r#"
            SELECT id, title, slug, content, excerpt, cover_image_url,
                status, published_at, author_id, category_id,
                view_count, like_count, comment_count,
                meta_title, meta_description,
                created_at, updated_at, deleted_at
            FROM posts
            WHERE slug = $1 AND deleted_at IS NULL
            "#,
            slug
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(AppError::Database)
    }
    
    pub async fn increment_view(&self, id: Uuid) -> Result<(), AppError> {
        sqlx::query!(
            "UPDATE posts SET view_count = view_count + 1 WHERE id = $1",
            id
        )
        .execute(&self.pool)
        .await
        .map_err(AppError::Database)?;
        Ok(())
    }
    
    pub async fn toggle_like(
        &self,
        user_id: Uuid,
        post_id: Uuid
    ) -> Result<bool, AppError> {
        let mut tx = self.pool.begin().await.map_err(AppError::Database)?;
        
        let exists: bool = sqlx::query_scalar!(
            r#"
            SELECT EXISTS(
                SELECT 1 FROM post_likes WHERE user_id = $1 AND post_id = $2
            ) as "exists!"
            "#,
            user_id,
            post_id
        )
        .fetch_one(&mut *tx)
        .await
        .map_err(AppError::Database)?;
        
        if exists {
            // Unlike
            sqlx::query!(
                "DELETE FROM post_likes WHERE user_id = $1 AND post_id = $2",
                user_id, post_id
            )
            .execute(&mut *tx)
            .await
            .map_err(AppError::Database)?;
            
            sqlx::query!(
                "UPDATE posts SET like_count = GREATEST(0, like_count - 1) WHERE id = $1",
                post_id
            )
            .execute(&mut *tx)
            .await
            .map_err(AppError::Database)?;
            
            tx.commit().await.map_err(AppError::Database)?;
            Ok(false)
        } else {
            // Like
            sqlx::query!(
                "INSERT INTO post_likes (user_id, post_id) VALUES ($1, $2)",
                user_id, post_id
            )
            .execute(&mut *tx)
            .await
            .map_err(AppError::Database)?;
            
            sqlx::query!(
                "UPDATE posts SET like_count = like_count + 1 WHERE id = $1",
                post_id
            )
            .execute(&mut *tx)
            .await
            .map_err(AppError::Database)?;
            
            tx.commit().await.map_err(AppError::Database)?;
            Ok(true)
        }
    }
}

#[async_trait]
impl Repository<PostEntity, CreatePostDto, UpdatePostDto> for PgPostRepository {
    type Error = AppError;
    
    async fn find_by_id(&self, id: Uuid) -> Result<Option<PostEntity>, AppError> {
        sqlx::query_as!(
            PostEntity,
            r#"
            SELECT id, title, slug, content, excerpt, cover_image_url,
                status, published_at, author_id, category_id,
                view_count, like_count, comment_count,
                meta_title, meta_description,
                created_at, updated_at, deleted_at
            FROM posts
            WHERE id = $1 AND deleted_at IS NULL
            "#,
            id
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(AppError::Database)
    }
    
    async fn find_all(&self, page: i64, per_page: i64) -> Result<Vec<PostEntity>, AppError> {
        let offset = (page - 1) * per_page;
        sqlx::query_as!(
            PostEntity,
            r#"
            SELECT id, title, slug, content, excerpt, cover_image_url,
                status, published_at, author_id, category_id,
                view_count, like_count, comment_count,
                meta_title, meta_description,
                created_at, updated_at, deleted_at
            FROM posts
            WHERE deleted_at IS NULL
            ORDER BY created_at DESC
            LIMIT $1 OFFSET $2
            "#,
            per_page,
            offset
        )
        .fetch_all(&self.pool)
        .await
        .map_err(AppError::Database)
    }
    
    async fn create(&self, dto: CreatePostDto) -> Result<PostEntity, AppError> {
        let slug = generate_slug(&dto.title);
        let status = dto.status.unwrap_or_else(|| "draft".to_string());
        let published_at = if status == "published" { Some(Utc::now()) } else { None };
        
        sqlx::query_as!(
            PostEntity,
            r#"
            INSERT INTO posts (
                title, slug, content, excerpt, cover_image_url,
                author_id, category_id, status, published_at,
                meta_title, meta_description
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
            RETURNING id, title, slug, content, excerpt, cover_image_url,
                status, published_at, author_id, category_id,
                view_count, like_count, comment_count,
                meta_title, meta_description,
                created_at, updated_at, deleted_at
            "#,
            dto.title, slug, dto.content, dto.excerpt, dto.cover_image_url,
            dto.author_id, dto.category_id, status, published_at,
            dto.meta_title, dto.meta_description
        )
        .fetch_one(&self.pool)
        .await
        .map_err(AppError::Database)
    }
    
    async fn update(&self, id: Uuid, dto: UpdatePostDto) -> Result<Option<PostEntity>, AppError> {
        sqlx::query_as!(
            PostEntity,
            r#"
            UPDATE posts SET
                title = COALESCE($2, title),
                content = COALESCE($3, content),
                excerpt = COALESCE($4, excerpt),
                cover_image_url = COALESCE($5, cover_image_url),
                category_id = COALESCE($6, category_id),
                status = COALESCE($7, status),
                meta_title = COALESCE($8, meta_title),
                meta_description = COALESCE($9, meta_description),
                updated_at = NOW()
            WHERE id = $1 AND deleted_at IS NULL
            RETURNING id, title, slug, content, excerpt, cover_image_url,
                status, published_at, author_id, category_id,
                view_count, like_count, comment_count,
                meta_title, meta_description,
                created_at, updated_at, deleted_at
            "#,
            id, dto.title, dto.content, dto.excerpt, dto.cover_image_url,
            dto.category_id, dto.status, dto.meta_title, dto.meta_description
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(AppError::Database)
    }
    
    async fn delete(&self, id: Uuid) -> Result<bool, AppError> {
        let result = sqlx::query!(
            "UPDATE posts SET deleted_at = NOW() WHERE id = $1 AND deleted_at IS NULL",
            id
        )
        .execute(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        Ok(result.rows_affected() > 0)
    }
    
    async fn count(&self) -> Result<i64, AppError> {
        sqlx::query_scalar!(
            r#"SELECT COUNT(*) as "count!" FROM posts WHERE deleted_at IS NULL"#
        )
        .fetch_one(&self.pool)
        .await
        .map_err(AppError::Database)
    }
    
    async fn exists(&self, id: Uuid) -> Result<bool, AppError> {
        sqlx::query_scalar!(
            r#"
            SELECT EXISTS(
                SELECT 1 FROM posts WHERE id = $1 AND deleted_at IS NULL
            ) as "exists!"
            "#,
            id
        )
        .fetch_one(&self.pool)
        .await
        .map_err(AppError::Database)
    }
}

fn generate_slug(title: &str) -> String {
    title.to_lowercase()
        .chars()
        .map(|c| if c.is_alphanumeric() { c } else { '-' })
        .collect::<String>()
        .split('-')
        .filter(|s| !s.is_empty())
        .collect::<Vec<_>>()
        .join("-")
}
```

---

## Navigation

| ก่อนหน้า | หน้าหลัก | ถัดไป |
|---------|---------|------|
| [Part 037: Advanced SQLx Queries](../part_037/README.md) | [README หลัก](../../README.md) | [Part 039: Query Optimization](../part_039/README.md) |
