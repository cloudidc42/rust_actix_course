# Part 061: Clean Architecture in Rust

## ภาพรวม

Clean Architecture เป็นแนวคิดการออกแบบซอฟต์แวร์ที่แยก concerns ออกเป็น layers ที่ชัดเจน ทำให้โค้ดทดสอบได้ง่าย บำรุงรักษาได้ดี และยืดหยุ่นต่อการเปลี่ยนแปลง ในบทนี้เราจะนำ Clean Architecture มาใช้กับ Rust และ Actix-web อย่างครบถ้วน

## โครงสร้าง Layers

```
┌─────────────────────────────────┐
│         Interface Layer         │  ← HTTP handlers, CLI, gRPC
├─────────────────────────────────┤
│       Application Layer         │  ← Use cases, Application services
├─────────────────────────────────┤
│         Domain Layer            │  ← Entities, Value objects, Domain logic
├─────────────────────────────────┤
│      Infrastructure Layer       │  ← Database, External APIs, File system
└─────────────────────────────────┘
```

กฎสำคัญ: **Dependencies ต้องชี้เข้าหา Domain เท่านั้น** (Dependency Inversion Principle)

## โครงสร้างโปรเจกต์

```
src/
├── main.rs
├── domain/
│   ├── mod.rs
│   ├── entities/
│   │   ├── mod.rs
│   │   └── user.rs
│   ├── value_objects/
│   │   ├── mod.rs
│   │   ├── email.rs
│   │   └── user_id.rs
│   ├── repositories/
│   │   ├── mod.rs
│   │   └── user_repository.rs
│   ├── services/
│   │   ├── mod.rs
│   │   └── user_domain_service.rs
│   └── errors.rs
├── application/
│   ├── mod.rs
│   ├── use_cases/
│   │   ├── mod.rs
│   │   ├── create_user.rs
│   │   ├── get_user.rs
│   │   ├── update_user.rs
│   │   └── delete_user.rs
│   ├── dtos/
│   │   ├── mod.rs
│   │   └── user_dto.rs
│   └── errors.rs
├── infrastructure/
│   ├── mod.rs
│   ├── database/
│   │   ├── mod.rs
│   │   ├── connection.rs
│   │   └── user_repository_impl.rs
│   └── errors.rs
└── interface/
    ├── mod.rs
    ├── http/
    │   ├── mod.rs
    │   ├── handlers/
    │   │   ├── mod.rs
    │   │   └── user_handler.rs
    │   └── middleware/
    │       └── mod.rs
    └── errors.rs
```

## Cargo.toml

```toml
[package]
name = "clean-arch-example"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4", "serde"] }
thiserror = "1"
async-trait = "0.1"
chrono = { version = "0.4", features = ["serde"] }
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio", "uuid", "chrono"] }
validator = { version = "0.16", features = ["derive"] }
bcrypt = "0.15"
tracing = "0.1"
tracing-subscriber = "0.3"
```

## Domain Layer - Value Objects

```rust
// src/domain/value_objects/email.rs
use std::fmt;
use thiserror::Error;

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct Email(String);

#[derive(Debug, Error)]
pub enum EmailError {
    #[error("Invalid email format: {0}")]
    InvalidFormat(String),
    #[error("Email is empty")]
    Empty,
}

impl Email {
    pub fn new(value: impl Into<String>) -> Result<Self, EmailError> {
        let value = value.into();
        
        if value.is_empty() {
            return Err(EmailError::Empty);
        }
        
        // ตรวจสอบรูปแบบ email เบื้องต้น
        if !value.contains('@') || !value.contains('.') {
            return Err(EmailError::InvalidFormat(value));
        }
        
        let parts: Vec<&str> = value.splitn(2, '@').collect();
        if parts.len() != 2 || parts[0].is_empty() || parts[1].is_empty() {
            return Err(EmailError::InvalidFormat(value));
        }
        
        Ok(Email(value.to_lowercase()))
    }
    
    pub fn value(&self) -> &str {
        &self.0
    }
}

impl fmt::Display for Email {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}

// src/domain/value_objects/user_id.rs
use uuid::Uuid;
use std::fmt;

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct UserId(Uuid);

impl UserId {
    pub fn new() -> Self {
        UserId(Uuid::new_v4())
    }
    
    pub fn from_uuid(uuid: Uuid) -> Self {
        UserId(uuid)
    }
    
    pub fn from_str(s: &str) -> Result<Self, uuid::Error> {
        Ok(UserId(Uuid::parse_str(s)?))
    }
    
    pub fn value(&self) -> Uuid {
        self.0
    }
}

impl Default for UserId {
    fn default() -> Self {
        Self::new()
    }
}

impl fmt::Display for UserId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}

// src/domain/value_objects/mod.rs
pub mod email;
pub mod user_id;

pub use email::Email;
pub use user_id::UserId;
```

## Domain Layer - Entities

```rust
// src/domain/entities/user.rs
use chrono::{DateTime, Utc};
use crate::domain::value_objects::{Email, UserId};

#[derive(Debug, Clone)]
pub struct User {
    id: UserId,
    email: Email,
    name: String,
    password_hash: String,
    is_active: bool,
    created_at: DateTime<Utc>,
    updated_at: DateTime<Utc>,
}

impl User {
    pub fn new(
        id: UserId,
        email: Email,
        name: String,
        password_hash: String,
    ) -> Self {
        let now = Utc::now();
        User {
            id,
            email,
            name,
            password_hash,
            is_active: true,
            created_at: now,
            updated_at: now,
        }
    }
    
    // Business logic อยู่ใน Entity
    pub fn activate(&mut self) {
        self.is_active = true;
        self.updated_at = Utc::now();
    }
    
    pub fn deactivate(&mut self) {
        self.is_active = false;
        self.updated_at = Utc::now();
    }
    
    pub fn update_name(&mut self, name: String) -> Result<(), DomainError> {
        if name.trim().is_empty() {
            return Err(DomainError::InvalidName("Name cannot be empty".to_string()));
        }
        if name.len() > 100 {
            return Err(DomainError::InvalidName("Name too long".to_string()));
        }
        self.name = name;
        self.updated_at = Utc::now();
        Ok(())
    }
    
    pub fn change_email(&mut self, email: Email) {
        self.email = email;
        self.updated_at = Utc::now();
    }
    
    // Getters
    pub fn id(&self) -> &UserId { &self.id }
    pub fn email(&self) -> &Email { &self.email }
    pub fn name(&self) -> &str { &self.name }
    pub fn password_hash(&self) -> &str { &self.password_hash }
    pub fn is_active(&self) -> bool { self.is_active }
    pub fn created_at(&self) -> DateTime<Utc> { self.created_at }
    pub fn updated_at(&self) -> DateTime<Utc> { self.updated_at }
}

use thiserror::Error;

#[derive(Debug, Error)]
pub enum DomainError {
    #[error("Invalid name: {0}")]
    InvalidName(String),
    #[error("User is not active")]
    UserNotActive,
}

// src/domain/entities/mod.rs
pub mod user;
pub use user::User;
```

## Domain Layer - Repository Interface (Trait)

```rust
// src/domain/repositories/user_repository.rs
use async_trait::async_trait;
use crate::domain::entities::User;
use crate::domain::value_objects::{Email, UserId};
use crate::domain::errors::DomainError;

// นี่คือ interface ที่ Domain กำหนด - Infrastructure ต้อง implement
#[async_trait]
pub trait UserRepository: Send + Sync {
    async fn find_by_id(&self, id: &UserId) -> Result<Option<User>, DomainError>;
    async fn find_by_email(&self, email: &Email) -> Result<Option<User>, DomainError>;
    async fn find_all(&self, page: u32, per_page: u32) -> Result<Vec<User>, DomainError>;
    async fn save(&self, user: &User) -> Result<(), DomainError>;
    async fn update(&self, user: &User) -> Result<(), DomainError>;
    async fn delete(&self, id: &UserId) -> Result<(), DomainError>;
    async fn exists_by_email(&self, email: &Email) -> Result<bool, DomainError>;
    async fn count(&self) -> Result<u64, DomainError>;
}

// src/domain/repositories/mod.rs
pub mod user_repository;
pub use user_repository::UserRepository;
```

## Domain Layer - Errors

```rust
// src/domain/errors.rs
use thiserror::Error;
use crate::domain::value_objects::email::EmailError;

#[derive(Debug, Error)]
pub enum DomainError {
    #[error("Entity not found: {0}")]
    NotFound(String),
    
    #[error("Invalid email: {0}")]
    InvalidEmail(#[from] EmailError),
    
    #[error("Validation error: {0}")]
    ValidationError(String),
    
    #[error("Business rule violation: {0}")]
    BusinessRuleViolation(String),
    
    #[error("Repository error: {0}")]
    RepositoryError(String),
    
    #[error("Concurrency conflict")]
    ConcurrencyConflict,
}

// src/domain/mod.rs
pub mod entities;
pub mod value_objects;
pub mod repositories;
pub mod errors;
```

## Application Layer - DTOs

```rust
// src/application/dtos/user_dto.rs
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use chrono::{DateTime, Utc};
use validator::Validate;

#[derive(Debug, Deserialize, Validate)]
pub struct CreateUserRequest {
    #[validate(email(message = "Invalid email format"))]
    pub email: String,
    
    #[validate(length(min = 1, max = 100, message = "Name must be 1-100 characters"))]
    pub name: String,
    
    #[validate(length(min = 8, message = "Password must be at least 8 characters"))]
    pub password: String,
}

#[derive(Debug, Deserialize, Validate)]
pub struct UpdateUserRequest {
    #[validate(length(min = 1, max = 100, message = "Name must be 1-100 characters"))]
    pub name: Option<String>,
    
    #[validate(email(message = "Invalid email format"))]
    pub email: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct UserResponse {
    pub id: Uuid,
    pub email: String,
    pub name: String,
    pub is_active: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Serialize)]
pub struct UserListResponse {
    pub users: Vec<UserResponse>,
    pub total: u64,
    pub page: u32,
    pub per_page: u32,
}

// src/application/dtos/mod.rs
pub mod user_dto;
pub use user_dto::*;
```

## Application Layer - Use Cases

```rust
// src/application/use_cases/create_user.rs
use std::sync::Arc;
use bcrypt::{hash, DEFAULT_COST};

use crate::domain::{
    entities::User,
    repositories::UserRepository,
    value_objects::{Email, UserId},
};
use crate::application::{
    dtos::{CreateUserRequest, UserResponse},
    errors::ApplicationError,
};

pub struct CreateUserUseCase {
    user_repository: Arc<dyn UserRepository>,
}

impl CreateUserUseCase {
    pub fn new(user_repository: Arc<dyn UserRepository>) -> Self {
        CreateUserUseCase { user_repository }
    }
    
    pub async fn execute(&self, request: CreateUserRequest) -> Result<UserResponse, ApplicationError> {
        // Validate input
        use validator::Validate;
        request.validate().map_err(|e| ApplicationError::ValidationError(e.to_string()))?;
        
        // สร้าง Email value object (domain validation)
        let email = Email::new(&request.email)
            .map_err(|e| ApplicationError::DomainError(e.to_string()))?;
        
        // ตรวจสอบว่า email ซ้ำไหม
        let exists = self.user_repository
            .exists_by_email(&email)
            .await
            .map_err(|e| ApplicationError::RepositoryError(e.to_string()))?;
            
        if exists {
            return Err(ApplicationError::Conflict(
                format!("Email {} already exists", email)
            ));
        }
        
        // Hash password
        let password_hash = hash(&request.password, DEFAULT_COST)
            .map_err(|e| ApplicationError::InternalError(e.to_string()))?;
        
        // สร้าง User entity
        let user_id = UserId::new();
        let user = User::new(
            user_id,
            email,
            request.name,
            password_hash,
        );
        
        // บันทึกลง repository
        self.user_repository
            .save(&user)
            .await
            .map_err(|e| ApplicationError::RepositoryError(e.to_string()))?;
        
        // แปลงเป็น DTO response
        Ok(UserResponse {
            id: user.id().value(),
            email: user.email().to_string(),
            name: user.name().to_string(),
            is_active: user.is_active(),
            created_at: user.created_at(),
            updated_at: user.updated_at(),
        })
    }
}

// src/application/use_cases/get_user.rs
use std::sync::Arc;
use uuid::Uuid;

use crate::domain::{
    repositories::UserRepository,
    value_objects::UserId,
};
use crate::application::{
    dtos::UserResponse,
    errors::ApplicationError,
};

pub struct GetUserUseCase {
    user_repository: Arc<dyn UserRepository>,
}

impl GetUserUseCase {
    pub fn new(user_repository: Arc<dyn UserRepository>) -> Self {
        GetUserUseCase { user_repository }
    }
    
    pub async fn execute(&self, user_id: Uuid) -> Result<UserResponse, ApplicationError> {
        let id = UserId::from_uuid(user_id);
        
        let user = self.user_repository
            .find_by_id(&id)
            .await
            .map_err(|e| ApplicationError::RepositoryError(e.to_string()))?
            .ok_or_else(|| ApplicationError::NotFound(format!("User {} not found", user_id)))?;
        
        Ok(UserResponse {
            id: user.id().value(),
            email: user.email().to_string(),
            name: user.name().to_string(),
            is_active: user.is_active(),
            created_at: user.created_at(),
            updated_at: user.updated_at(),
        })
    }
}

// src/application/use_cases/update_user.rs
use std::sync::Arc;
use uuid::Uuid;

use crate::domain::{
    repositories::UserRepository,
    value_objects::{Email, UserId},
};
use crate::application::{
    dtos::{UpdateUserRequest, UserResponse},
    errors::ApplicationError,
};

pub struct UpdateUserUseCase {
    user_repository: Arc<dyn UserRepository>,
}

impl UpdateUserUseCase {
    pub fn new(user_repository: Arc<dyn UserRepository>) -> Self {
        UpdateUserUseCase { user_repository }
    }
    
    pub async fn execute(
        &self,
        user_id: Uuid,
        request: UpdateUserRequest,
    ) -> Result<UserResponse, ApplicationError> {
        use validator::Validate;
        request.validate().map_err(|e| ApplicationError::ValidationError(e.to_string()))?;
        
        let id = UserId::from_uuid(user_id);
        
        let mut user = self.user_repository
            .find_by_id(&id)
            .await
            .map_err(|e| ApplicationError::RepositoryError(e.to_string()))?
            .ok_or_else(|| ApplicationError::NotFound(format!("User {} not found", user_id)))?;
        
        // อัพเดท name ถ้ามี
        if let Some(name) = request.name {
            user.update_name(name)
                .map_err(|e| ApplicationError::DomainError(e.to_string()))?;
        }
        
        // อัพเดท email ถ้ามี
        if let Some(email_str) = request.email {
            let new_email = Email::new(&email_str)
                .map_err(|e| ApplicationError::DomainError(e.to_string()))?;
            
            // ตรวจสอบว่า email ใหม่ไม่ซ้ำ (ถ้าต่างจากเดิม)
            if new_email != *user.email() {
                let exists = self.user_repository
                    .exists_by_email(&new_email)
                    .await
                    .map_err(|e| ApplicationError::RepositoryError(e.to_string()))?;
                    
                if exists {
                    return Err(ApplicationError::Conflict(
                        format!("Email {} already exists", new_email)
                    ));
                }
                
                user.change_email(new_email);
            }
        }
        
        self.user_repository
            .update(&user)
            .await
            .map_err(|e| ApplicationError::RepositoryError(e.to_string()))?;
        
        Ok(UserResponse {
            id: user.id().value(),
            email: user.email().to_string(),
            name: user.name().to_string(),
            is_active: user.is_active(),
            created_at: user.created_at(),
            updated_at: user.updated_at(),
        })
    }
}

// src/application/use_cases/delete_user.rs
use std::sync::Arc;
use uuid::Uuid;

use crate::domain::{
    repositories::UserRepository,
    value_objects::UserId,
};
use crate::application::errors::ApplicationError;

pub struct DeleteUserUseCase {
    user_repository: Arc<dyn UserRepository>,
}

impl DeleteUserUseCase {
    pub fn new(user_repository: Arc<dyn UserRepository>) -> Self {
        DeleteUserUseCase { user_repository }
    }
    
    pub async fn execute(&self, user_id: Uuid) -> Result<(), ApplicationError> {
        let id = UserId::from_uuid(user_id);
        
        // ตรวจสอบว่า user มีอยู่
        let user = self.user_repository
            .find_by_id(&id)
            .await
            .map_err(|e| ApplicationError::RepositoryError(e.to_string()))?;
            
        if user.is_none() {
            return Err(ApplicationError::NotFound(
                format!("User {} not found", user_id)
            ));
        }
        
        self.user_repository
            .delete(&id)
            .await
            .map_err(|e| ApplicationError::RepositoryError(e.to_string()))?;
        
        Ok(())
    }
}

// src/application/use_cases/mod.rs
pub mod create_user;
pub mod get_user;
pub mod update_user;
pub mod delete_user;

pub use create_user::CreateUserUseCase;
pub use get_user::GetUserUseCase;
pub use update_user::UpdateUserUseCase;
pub use delete_user::DeleteUserUseCase;
```

## Application Layer - Errors

```rust
// src/application/errors.rs
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ApplicationError {
    #[error("Not found: {0}")]
    NotFound(String),
    
    #[error("Validation error: {0}")]
    ValidationError(String),
    
    #[error("Conflict: {0}")]
    Conflict(String),
    
    #[error("Domain error: {0}")]
    DomainError(String),
    
    #[error("Repository error: {0}")]
    RepositoryError(String),
    
    #[error("Internal error: {0}")]
    InternalError(String),
    
    #[error("Unauthorized: {0}")]
    Unauthorized(String),
    
    #[error("Forbidden: {0}")]
    Forbidden(String),
}

// src/application/mod.rs
pub mod use_cases;
pub mod dtos;
pub mod errors;
```

## Infrastructure Layer - Repository Implementation

```rust
// src/infrastructure/database/user_repository_impl.rs
use async_trait::async_trait;
use sqlx::{PgPool, Row};
use uuid::Uuid;
use chrono::{DateTime, Utc};

use crate::domain::{
    entities::User,
    repositories::UserRepository,
    value_objects::{Email, UserId},
    errors::DomainError,
};

pub struct PostgresUserRepository {
    pool: PgPool,
}

impl PostgresUserRepository {
    pub fn new(pool: PgPool) -> Self {
        PostgresUserRepository { pool }
    }
}

// Row struct สำหรับ mapping จาก database
struct UserRow {
    id: Uuid,
    email: String,
    name: String,
    password_hash: String,
    is_active: bool,
    created_at: DateTime<Utc>,
    updated_at: DateTime<Utc>,
}

impl TryFrom<UserRow> for User {
    type Error = DomainError;
    
    fn try_from(row: UserRow) -> Result<Self, Self::Error> {
        let id = UserId::from_uuid(row.id);
        let email = Email::new(row.email)
            .map_err(|e| DomainError::ValidationError(e.to_string()))?;
        
        Ok(User::new(id, email, row.name, row.password_hash))
    }
}

#[async_trait]
impl UserRepository for PostgresUserRepository {
    async fn find_by_id(&self, id: &UserId) -> Result<Option<User>, DomainError> {
        let result = sqlx::query_as!(
            UserRow,
            r#"
            SELECT id, email, name, password_hash, is_active, created_at, updated_at
            FROM users
            WHERE id = $1
            "#,
            id.value()
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(|e| DomainError::RepositoryError(e.to_string()))?;
        
        match result {
            Some(row) => Ok(Some(User::try_from(row)?)),
            None => Ok(None),
        }
    }
    
    async fn find_by_email(&self, email: &Email) -> Result<Option<User>, DomainError> {
        let result = sqlx::query!(
            r#"
            SELECT id, email, name, password_hash, is_active, created_at, updated_at
            FROM users
            WHERE email = $1
            "#,
            email.value()
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(|e| DomainError::RepositoryError(e.to_string()))?;
        
        match result {
            Some(row) => {
                let id = UserId::from_uuid(row.id);
                let email_vo = Email::new(row.email)
                    .map_err(|e| DomainError::ValidationError(e.to_string()))?;
                Ok(Some(User::new(id, email_vo, row.name, row.password_hash)))
            }
            None => Ok(None),
        }
    }
    
    async fn find_all(&self, page: u32, per_page: u32) -> Result<Vec<User>, DomainError> {
        let offset = (page - 1) * per_page;
        
        let rows = sqlx::query!(
            r#"
            SELECT id, email, name, password_hash, is_active, created_at, updated_at
            FROM users
            ORDER BY created_at DESC
            LIMIT $1 OFFSET $2
            "#,
            per_page as i64,
            offset as i64
        )
        .fetch_all(&self.pool)
        .await
        .map_err(|e| DomainError::RepositoryError(e.to_string()))?;
        
        rows.into_iter()
            .map(|row| {
                let id = UserId::from_uuid(row.id);
                let email = Email::new(row.email)
                    .map_err(|e| DomainError::ValidationError(e.to_string()))?;
                Ok(User::new(id, email, row.name, row.password_hash))
            })
            .collect()
    }
    
    async fn save(&self, user: &User) -> Result<(), DomainError> {
        sqlx::query!(
            r#"
            INSERT INTO users (id, email, name, password_hash, is_active, created_at, updated_at)
            VALUES ($1, $2, $3, $4, $5, $6, $7)
            "#,
            user.id().value(),
            user.email().value(),
            user.name(),
            user.password_hash(),
            user.is_active(),
            user.created_at(),
            user.updated_at(),
        )
        .execute(&self.pool)
        .await
        .map_err(|e| DomainError::RepositoryError(e.to_string()))?;
        
        Ok(())
    }
    
    async fn update(&self, user: &User) -> Result<(), DomainError> {
        let rows_affected = sqlx::query!(
            r#"
            UPDATE users
            SET email = $2, name = $3, is_active = $4, updated_at = $5
            WHERE id = $1
            "#,
            user.id().value(),
            user.email().value(),
            user.name(),
            user.is_active(),
            user.updated_at(),
        )
        .execute(&self.pool)
        .await
        .map_err(|e| DomainError::RepositoryError(e.to_string()))?
        .rows_affected();
        
        if rows_affected == 0 {
            return Err(DomainError::NotFound(
                format!("User {} not found", user.id())
            ));
        }
        
        Ok(())
    }
    
    async fn delete(&self, id: &UserId) -> Result<(), DomainError> {
        sqlx::query!(
            "DELETE FROM users WHERE id = $1",
            id.value()
        )
        .execute(&self.pool)
        .await
        .map_err(|e| DomainError::RepositoryError(e.to_string()))?;
        
        Ok(())
    }
    
    async fn exists_by_email(&self, email: &Email) -> Result<bool, DomainError> {
        let result = sqlx::query!(
            "SELECT COUNT(*) as count FROM users WHERE email = $1",
            email.value()
        )
        .fetch_one(&self.pool)
        .await
        .map_err(|e| DomainError::RepositoryError(e.to_string()))?;
        
        Ok(result.count.unwrap_or(0) > 0)
    }
    
    async fn count(&self) -> Result<u64, DomainError> {
        let result = sqlx::query!("SELECT COUNT(*) as count FROM users")
            .fetch_one(&self.pool)
            .await
            .map_err(|e| DomainError::RepositoryError(e.to_string()))?;
        
        Ok(result.count.unwrap_or(0) as u64)
    }
}
```

## Interface Layer - HTTP Handlers

```rust
// src/interface/http/handlers/user_handler.rs
use actix_web::{web, HttpResponse, Result};
use std::sync::Arc;
use uuid::Uuid;
use serde::Deserialize;

use crate::application::{
    use_cases::{CreateUserUseCase, GetUserUseCase, UpdateUserUseCase, DeleteUserUseCase},
    dtos::{CreateUserRequest, UpdateUserRequest},
    errors::ApplicationError,
};

// Application state ที่ inject use cases
pub struct UserHandlerState {
    pub create_user: Arc<CreateUserUseCase>,
    pub get_user: Arc<GetUserUseCase>,
    pub update_user: Arc<UpdateUserUseCase>,
    pub delete_user: Arc<DeleteUserUseCase>,
}

#[derive(Deserialize)]
pub struct PaginationParams {
    pub page: Option<u32>,
    pub per_page: Option<u32>,
}

// แปลง ApplicationError เป็น HTTP response
fn map_error(error: ApplicationError) -> HttpResponse {
    match error {
        ApplicationError::NotFound(msg) => {
            HttpResponse::NotFound().json(serde_json::json!({
                "error": "not_found",
                "message": msg
            }))
        }
        ApplicationError::ValidationError(msg) => {
            HttpResponse::BadRequest().json(serde_json::json!({
                "error": "validation_error",
                "message": msg
            }))
        }
        ApplicationError::Conflict(msg) => {
            HttpResponse::Conflict().json(serde_json::json!({
                "error": "conflict",
                "message": msg
            }))
        }
        ApplicationError::Unauthorized(msg) => {
            HttpResponse::Unauthorized().json(serde_json::json!({
                "error": "unauthorized",
                "message": msg
            }))
        }
        _ => {
            HttpResponse::InternalServerError().json(serde_json::json!({
                "error": "internal_error",
                "message": "An internal error occurred"
            }))
        }
    }
}

pub async fn create_user(
    state: web::Data<UserHandlerState>,
    body: web::Json<CreateUserRequest>,
) -> HttpResponse {
    match state.create_user.execute(body.into_inner()).await {
        Ok(user) => HttpResponse::Created().json(user),
        Err(e) => map_error(e),
    }
}

pub async fn get_user(
    state: web::Data<UserHandlerState>,
    path: web::Path<Uuid>,
) -> HttpResponse {
    match state.get_user.execute(*path).await {
        Ok(user) => HttpResponse::Ok().json(user),
        Err(e) => map_error(e),
    }
}

pub async fn update_user(
    state: web::Data<UserHandlerState>,
    path: web::Path<Uuid>,
    body: web::Json<UpdateUserRequest>,
) -> HttpResponse {
    match state.update_user.execute(*path, body.into_inner()).await {
        Ok(user) => HttpResponse::Ok().json(user),
        Err(e) => map_error(e),
    }
}

pub async fn delete_user(
    state: web::Data<UserHandlerState>,
    path: web::Path<Uuid>,
) -> HttpResponse {
    match state.delete_user.execute(*path).await {
        Ok(_) => HttpResponse::NoContent().finish(),
        Err(e) => map_error(e),
    }
}

// Router configuration
pub fn user_routes(cfg: &mut web::ServiceConfig) {
    cfg.service(
        web::scope("/users")
            .route("", web::post().to(create_user))
            .route("/{id}", web::get().to(get_user))
            .route("/{id}", web::put().to(update_user))
            .route("/{id}", web::delete().to(delete_user))
    );
}
```

## Main - Dependency Injection

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware};
use sqlx::PgPool;
use std::sync::Arc;
use tracing_subscriber::{EnvFilter, fmt};

mod domain;
mod application;
mod infrastructure;
mod interface;

use application::use_cases::{
    CreateUserUseCase, GetUserUseCase, UpdateUserUseCase, DeleteUserUseCase,
};
use infrastructure::database::PostgresUserRepository;
use interface::http::handlers::user_handler::{UserHandlerState, user_routes};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // Setup logging
    tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::from_default_env())
        .init();
    
    // Database connection
    let database_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    let pool = PgPool::connect(&database_url)
        .await
        .expect("Failed to connect to database");
    
    // Infrastructure layer - Repository implementation
    let user_repository = Arc::new(PostgresUserRepository::new(pool));
    
    // Application layer - Use cases (Dependency Injection)
    let create_user = Arc::new(CreateUserUseCase::new(Arc::clone(&user_repository)));
    let get_user = Arc::new(GetUserUseCase::new(Arc::clone(&user_repository)));
    let update_user = Arc::new(UpdateUserUseCase::new(Arc::clone(&user_repository)));
    let delete_user = Arc::new(DeleteUserUseCase::new(Arc::clone(&user_repository)));
    
    // Interface layer - Handler state
    let handler_state = web::Data::new(UserHandlerState {
        create_user,
        get_user,
        update_user,
        delete_user,
    });
    
    tracing::info!("Starting server on 0.0.0.0:8080");
    
    HttpServer::new(move || {
        App::new()
            .app_data(handler_state.clone())
            .configure(user_routes)
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

## Unit Tests

```rust
// tests/domain/user_test.rs
#[cfg(test)]
mod tests {
    use crate::domain::{
        entities::User,
        value_objects::{Email, UserId},
    };
    
    #[test]
    fn test_create_user() {
        let id = UserId::new();
        let email = Email::new("test@example.com").unwrap();
        let user = User::new(id, email.clone(), "Test User".to_string(), "hash".to_string());
        
        assert_eq!(user.email(), &email);
        assert_eq!(user.name(), "Test User");
        assert!(user.is_active());
    }
    
    #[test]
    fn test_update_name() {
        let id = UserId::new();
        let email = Email::new("test@example.com").unwrap();
        let mut user = User::new(id, email, "Old Name".to_string(), "hash".to_string());
        
        user.update_name("New Name".to_string()).unwrap();
        assert_eq!(user.name(), "New Name");
    }
    
    #[test]
    fn test_update_name_too_long() {
        let id = UserId::new();
        let email = Email::new("test@example.com").unwrap();
        let mut user = User::new(id, email, "Name".to_string(), "hash".to_string());
        
        let long_name = "a".repeat(101);
        assert!(user.update_name(long_name).is_err());
    }
    
    #[test]
    fn test_email_validation() {
        assert!(Email::new("valid@email.com").is_ok());
        assert!(Email::new("invalid-email").is_err());
        assert!(Email::new("").is_err());
        assert!(Email::new("@example.com").is_err());
    }
    
    #[test]
    fn test_deactivate_user() {
        let id = UserId::new();
        let email = Email::new("test@example.com").unwrap();
        let mut user = User::new(id, email, "Test".to_string(), "hash".to_string());
        
        user.deactivate();
        assert!(!user.is_active());
        
        user.activate();
        assert!(user.is_active());
    }
}

// tests/application/create_user_test.rs
#[cfg(test)]
mod tests {
    use std::sync::Arc;
    use async_trait::async_trait;
    use std::collections::HashMap;
    use std::sync::Mutex;
    
    use crate::domain::{
        entities::User,
        repositories::UserRepository,
        value_objects::{Email, UserId},
        errors::DomainError,
    };
    use crate::application::{
        dtos::CreateUserRequest,
        use_cases::CreateUserUseCase,
    };
    
    // Mock repository สำหรับ testing
    struct MockUserRepository {
        users: Mutex<HashMap<String, User>>,
    }
    
    impl MockUserRepository {
        fn new() -> Self {
            MockUserRepository {
                users: Mutex::new(HashMap::new()),
            }
        }
    }
    
    #[async_trait]
    impl UserRepository for MockUserRepository {
        async fn find_by_id(&self, id: &UserId) -> Result<Option<User>, DomainError> {
            let users = self.users.lock().unwrap();
            Ok(users.get(&id.to_string()).cloned())
        }
        
        async fn find_by_email(&self, email: &Email) -> Result<Option<User>, DomainError> {
            let users = self.users.lock().unwrap();
            Ok(users.values().find(|u| u.email() == email).cloned())
        }
        
        async fn find_all(&self, _page: u32, _per_page: u32) -> Result<Vec<User>, DomainError> {
            let users = self.users.lock().unwrap();
            Ok(users.values().cloned().collect())
        }
        
        async fn save(&self, user: &User) -> Result<(), DomainError> {
            let mut users = self.users.lock().unwrap();
            users.insert(user.id().to_string(), user.clone());
            Ok(())
        }
        
        async fn update(&self, user: &User) -> Result<(), DomainError> {
            let mut users = self.users.lock().unwrap();
            users.insert(user.id().to_string(), user.clone());
            Ok(())
        }
        
        async fn delete(&self, id: &UserId) -> Result<(), DomainError> {
            let mut users = self.users.lock().unwrap();
            users.remove(&id.to_string());
            Ok(())
        }
        
        async fn exists_by_email(&self, email: &Email) -> Result<bool, DomainError> {
            let users = self.users.lock().unwrap();
            Ok(users.values().any(|u| u.email() == email))
        }
        
        async fn count(&self) -> Result<u64, DomainError> {
            let users = self.users.lock().unwrap();
            Ok(users.len() as u64)
        }
    }
    
    #[tokio::test]
    async fn test_create_user_success() {
        let repo = Arc::new(MockUserRepository::new());
        let use_case = CreateUserUseCase::new(repo);
        
        let request = CreateUserRequest {
            email: "test@example.com".to_string(),
            name: "Test User".to_string(),
            password: "password123".to_string(),
        };
        
        let result = use_case.execute(request).await;
        assert!(result.is_ok());
        
        let user = result.unwrap();
        assert_eq!(user.email, "test@example.com");
        assert_eq!(user.name, "Test User");
        assert!(user.is_active);
    }
    
    #[tokio::test]
    async fn test_create_user_duplicate_email() {
        let repo = Arc::new(MockUserRepository::new());
        let use_case = CreateUserUseCase::new(Arc::clone(&repo));
        
        let request1 = CreateUserRequest {
            email: "test@example.com".to_string(),
            name: "User 1".to_string(),
            password: "password123".to_string(),
        };
        
        let request2 = CreateUserRequest {
            email: "test@example.com".to_string(),
            name: "User 2".to_string(),
            password: "password456".to_string(),
        };
        
        use_case.execute(request1).await.unwrap();
        let result = use_case.execute(request2).await;
        
        assert!(result.is_err());
        // ตรวจสอบว่าเป็น Conflict error
    }
}
```

## สรุป

Clean Architecture ใน Rust มีข้อดีหลัก:

1. **Testability** - แต่ละ layer ทดสอบได้อิสระ โดยใช้ mock
2. **Maintainability** - เปลี่ยน database หรือ framework โดยไม่กระทบ domain logic
3. **Type Safety** - Value objects ป้องกัน primitive obsession
4. **Clear Boundaries** - Trait ทำหน้าที่เป็น interface ระหว่าง layers

---

## Navigation

- [← Part 060: Advanced Authentication](../part_060/README.md)
- [→ Part 062: Domain-Driven Design (DDD)](../part_062/README.md)
- [กลับหน้าหลัก](../../README.md)
