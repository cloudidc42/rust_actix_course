# Part 033: CRUD Operations with SQLx

## สารบัญ
- [SELECT Queries](#select-queries)
- [INSERT Operations](#insert-operations)
- [UPDATE Operations](#update-operations)
- [DELETE และ Soft Delete](#delete-และ-soft-delete)
- [query! vs query_as! Macros](#query-vs-query_as-macros)
- [Transaction Basics](#transaction-basics)
- [Bulk Insert](#bulk-insert)
- [Complete Post CRUD Repository](#complete-post-crud-repository)

---

## SELECT Queries

### SELECT Single Record

```rust
// src/repositories/user_repository.rs
use sqlx::PgPool;
use uuid::Uuid;
use crate::models::user::User;
use crate::errors::AppError;

pub struct UserRepository {
    pool: PgPool,
}

impl UserRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
    
    // หา user ด้วย ID
    pub async fn find_by_id(&self, id: Uuid) -> Result<Option<User>, AppError> {
        let user = sqlx::query_as!(
            User,
            r#"
            SELECT 
                id, email, username, password_hash,
                display_name, bio, avatar_url, website_url,
                is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            FROM users
            WHERE id = $1 AND deleted_at IS NULL
            "#,
            id
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        Ok(user)
    }
    
    // หา user ด้วย email
    pub async fn find_by_email(&self, email: &str) -> Result<Option<User>, AppError> {
        let user = sqlx::query_as!(
            User,
            r#"
            SELECT 
                id, email, username, password_hash,
                display_name, bio, avatar_url, website_url,
                is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            FROM users
            WHERE email = $1 AND deleted_at IS NULL
            "#,
            email.to_lowercase()
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        Ok(user)
    }
    
    // หา user ด้วย username
    pub async fn find_by_username(&self, username: &str) -> Result<Option<User>, AppError> {
        let user = sqlx::query_as!(
            User,
            "SELECT * FROM users WHERE username = $1 AND deleted_at IS NULL",
            username
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        Ok(user)
    }
}
```

### SELECT List with Filtering

```rust
// src/repositories/user_repository.rs (ต่อ)
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize)]
pub struct UserFilter {
    pub search: Option<String>,
    pub is_active: Option<bool>,
    pub is_admin: Option<bool>,
    pub page: Option<i64>,
    pub per_page: Option<i64>,
}

#[derive(Debug, Serialize)]
pub struct PaginatedUsers {
    pub data: Vec<User>,
    pub total: i64,
    pub page: i64,
    pub per_page: i64,
    pub total_pages: i64,
}

impl UserRepository {
    // รายการ users พร้อม filter และ pagination
    pub async fn find_all(&self, filter: UserFilter) -> Result<PaginatedUsers, AppError> {
        let page = filter.page.unwrap_or(1).max(1);
        let per_page = filter.per_page.unwrap_or(20).min(100);
        let offset = (page - 1) * per_page;
        
        // COUNT query
        let total: i64 = sqlx::query_scalar!(
            r#"
            SELECT COUNT(*) as "count!"
            FROM users
            WHERE deleted_at IS NULL
                AND ($1::boolean IS NULL OR is_active = $1)
                AND ($2::boolean IS NULL OR is_admin = $2)
                AND ($3::text IS NULL OR (
                    email ILIKE '%' || $3 || '%' OR
                    username ILIKE '%' || $3 || '%' OR
                    display_name ILIKE '%' || $3 || '%'
                ))
            "#,
            filter.is_active,
            filter.is_admin,
            filter.search
        )
        .fetch_one(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        // DATA query
        let users = sqlx::query_as!(
            User,
            r#"
            SELECT 
                id, email, username, password_hash,
                display_name, bio, avatar_url, website_url,
                is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            FROM users
            WHERE deleted_at IS NULL
                AND ($1::boolean IS NULL OR is_active = $1)
                AND ($2::boolean IS NULL OR is_admin = $2)
                AND ($3::text IS NULL OR (
                    email ILIKE '%' || $3 || '%' OR
                    username ILIKE '%' || $3 || '%' OR
                    display_name ILIKE '%' || $3 || '%'
                ))
            ORDER BY created_at DESC
            LIMIT $4
            OFFSET $5
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
        
        let total_pages = (total + per_page - 1) / per_page;
        
        Ok(PaginatedUsers {
            data: users,
            total,
            page,
            per_page,
            total_pages,
        })
    }
    
    // หา users ที่ active ทั้งหมด
    pub async fn find_active_users(&self) -> Result<Vec<User>, AppError> {
        let users = sqlx::query_as!(
            User,
            r#"
            SELECT 
                id, email, username, password_hash,
                display_name, bio, avatar_url, website_url,
                is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            FROM users
            WHERE is_active = true AND deleted_at IS NULL
            ORDER BY created_at DESC
            "#
        )
        .fetch_all(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        Ok(users)
    }
}
```

---

## INSERT Operations

### INSERT และ Return ID

```rust
// src/repositories/user_repository.rs (ต่อ)
use crate::models::user::CreateUserDto;

#[derive(Debug, Deserialize)]
pub struct CreateUserDto {
    pub email: String,
    pub username: String,
    pub password_hash: String,
    pub display_name: String,
}

impl UserRepository {
    // สร้าง user ใหม่และ return ข้อมูลทั้งหมด
    pub async fn create(&self, dto: CreateUserDto) -> Result<User, AppError> {
        // ตรวจสอบ email ซ้ำ
        let email_exists = sqlx::query_scalar!(
            "SELECT EXISTS(SELECT 1 FROM users WHERE email = $1 AND deleted_at IS NULL)",
            dto.email.to_lowercase()
        )
        .fetch_one(&self.pool)
        .await
        .map_err(AppError::Database)?
        .unwrap_or(false);
        
        if email_exists {
            return Err(AppError::Conflict("Email already exists".to_string()));
        }
        
        // สร้าง user และ return ข้อมูลทั้งหมด
        let user = sqlx::query_as!(
            User,
            r#"
            INSERT INTO users (
                email, username, password_hash, display_name
            )
            VALUES ($1, $2, $3, $4)
            RETURNING 
                id, email, username, password_hash,
                display_name, bio, avatar_url, website_url,
                is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            "#,
            dto.email.to_lowercase(),
            dto.username.to_lowercase(),
            dto.password_hash,
            dto.display_name
        )
        .fetch_one(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        Ok(user)
    }
    
    // Insert และ return เฉพาะ ID
    pub async fn create_and_get_id(&self, dto: CreateUserDto) -> Result<Uuid, AppError> {
        let id = sqlx::query_scalar!(
            r#"
            INSERT INTO users (email, username, password_hash, display_name)
            VALUES ($1, $2, $3, $4)
            RETURNING id
            "#,
            dto.email.to_lowercase(),
            dto.username,
            dto.password_hash,
            dto.display_name
        )
        .fetch_one(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        Ok(id)
    }
    
    // Insert or Update (Upsert)
    pub async fn upsert_by_email(&self, dto: CreateUserDto) -> Result<User, AppError> {
        let user = sqlx::query_as!(
            User,
            r#"
            INSERT INTO users (email, username, password_hash, display_name)
            VALUES ($1, $2, $3, $4)
            ON CONFLICT (email) DO UPDATE SET
                username = EXCLUDED.username,
                display_name = EXCLUDED.display_name,
                updated_at = NOW()
            RETURNING 
                id, email, username, password_hash,
                display_name, bio, avatar_url, website_url,
                is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            "#,
            dto.email.to_lowercase(),
            dto.username,
            dto.password_hash,
            dto.display_name
        )
        .fetch_one(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        Ok(user)
    }
}
```

---

## UPDATE Operations

### UPDATE ทั้งหมดและ Partial Update

```rust
// src/repositories/user_repository.rs (ต่อ)

#[derive(Debug, Deserialize)]
pub struct UpdateUserDto {
    pub display_name: Option<String>,
    pub bio: Option<String>,
    pub avatar_url: Option<String>,
    pub website_url: Option<String>,
}

impl UserRepository {
    // UPDATE ด้วย COALESCE (partial update)
    pub async fn update(&self, id: Uuid, dto: UpdateUserDto) -> Result<User, AppError> {
        let user = sqlx::query_as!(
            User,
            r#"
            UPDATE users
            SET
                display_name = COALESCE($2, display_name),
                bio = COALESCE($3, bio),
                avatar_url = COALESCE($4, avatar_url),
                website_url = COALESCE($5, website_url),
                updated_at = NOW()
            WHERE id = $1 AND deleted_at IS NULL
            RETURNING 
                id, email, username, password_hash,
                display_name, bio, avatar_url, website_url,
                is_active, is_admin, is_verified,
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
        .map_err(AppError::Database)?
        .ok_or(AppError::NotFound("User not found".to_string()))?;
        
        Ok(user)
    }
    
    // UPDATE password
    pub async fn update_password(
        &self,
        id: Uuid,
        new_password_hash: &str
    ) -> Result<(), AppError> {
        let result = sqlx::query!(
            r#"
            UPDATE users
            SET password_hash = $2, updated_at = NOW()
            WHERE id = $1 AND deleted_at IS NULL
            "#,
            id,
            new_password_hash
        )
        .execute(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        if result.rows_affected() == 0 {
            return Err(AppError::NotFound("User not found".to_string()));
        }
        
        Ok(())
    }
    
    // UPDATE last login
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
    
    // UPDATE active status (activate/deactivate)
    pub async fn set_active_status(
        &self,
        id: Uuid,
        is_active: bool
    ) -> Result<User, AppError> {
        let user = sqlx::query_as!(
            User,
            r#"
            UPDATE users
            SET is_active = $2, updated_at = NOW()
            WHERE id = $1 AND deleted_at IS NULL
            RETURNING 
                id, email, username, password_hash,
                display_name, bio, avatar_url, website_url,
                is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            "#,
            id,
            is_active
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(AppError::Database)?
        .ok_or(AppError::NotFound("User not found".to_string()))?;
        
        Ok(user)
    }
}
```

---

## DELETE และ Soft Delete

### Hard Delete vs Soft Delete

```rust
// src/repositories/user_repository.rs (ต่อ)

impl UserRepository {
    // Soft delete (แนะนำ)
    pub async fn soft_delete(&self, id: Uuid) -> Result<(), AppError> {
        let result = sqlx::query!(
            r#"
            UPDATE users
            SET deleted_at = NOW(), updated_at = NOW()
            WHERE id = $1 AND deleted_at IS NULL
            "#,
            id
        )
        .execute(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        if result.rows_affected() == 0 {
            return Err(AppError::NotFound("User not found".to_string()));
        }
        
        Ok(())
    }
    
    // Restore soft deleted record
    pub async fn restore(&self, id: Uuid) -> Result<User, AppError> {
        let user = sqlx::query_as!(
            User,
            r#"
            UPDATE users
            SET deleted_at = NULL, updated_at = NOW()
            WHERE id = $1 AND deleted_at IS NOT NULL
            RETURNING 
                id, email, username, password_hash,
                display_name, bio, avatar_url, website_url,
                is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            "#,
            id
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(AppError::Database)?
        .ok_or(AppError::NotFound("User not found or not deleted".to_string()))?;
        
        Ok(user)
    }
    
    // Hard delete (permanent)
    pub async fn hard_delete(&self, id: Uuid) -> Result<(), AppError> {
        let result = sqlx::query!(
            "DELETE FROM users WHERE id = $1",
            id
        )
        .execute(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        if result.rows_affected() == 0 {
            return Err(AppError::NotFound("User not found".to_string()));
        }
        
        Ok(())
    }
    
    // Purge soft-deleted records older than N days
    pub async fn purge_old_deleted(&self, days: i64) -> Result<u64, AppError> {
        let result = sqlx::query!(
            r#"
            DELETE FROM users
            WHERE deleted_at IS NOT NULL
                AND deleted_at < NOW() - ($1 || ' days')::interval
            "#,
            days.to_string()
        )
        .execute(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        Ok(result.rows_affected())
    }
}
```

---

## query! vs query_as! Macros

### ความแตกต่างและการใช้งาน

```rust
// src/examples/query_macros.rs

use sqlx::PgPool;
use uuid::Uuid;

pub async fn examples(pool: &PgPool) -> Result<(), sqlx::Error> {
    let user_id = Uuid::new_v4();
    
    // ===== query! macro =====
    // - ตรวจสอบ SQL ใน compile time
    // - Return anonymous struct
    // - ใช้เมื่อ schema ไม่ตรงกับ struct ที่มีอยู่
    
    let row = sqlx::query!(
        "SELECT id, email, username FROM users WHERE id = $1",
        user_id
    )
    .fetch_optional(pool)
    .await?;
    
    if let Some(row) = row {
        println!("ID: {}", row.id);
        println!("Email: {}", row.email);
        println!("Username: {}", row.username);
    }
    
    // ===== query_as! macro =====
    // - Map result เข้า struct ที่กำหนด
    // - ต้องมี fields ตรงกับ SELECT columns
    
    #[derive(Debug)]
    struct UserBasic {
        id: Uuid,
        email: String,
        username: String,
    }
    
    let user = sqlx::query_as!(
        UserBasic,
        "SELECT id, email, username FROM users WHERE id = $1",
        user_id
    )
    .fetch_optional(pool)
    .await?;
    
    // ===== query_scalar! macro =====
    // - Return single scalar value
    
    let count: i64 = sqlx::query_scalar!(
        r#"SELECT COUNT(*) as "count!" FROM users WHERE is_active = true"#
    )
    .fetch_one(pool)
    .await?;
    
    println!("Active users: {}", count);
    
    // ===== query (runtime, ไม่มี compile-time check) =====
    // - ใช้เมื่อ query แบบ dynamic
    // - ช้ากว่าเพราะไม่มี optimization
    
    let search = "test".to_string();
    let rows = sqlx::query(
        "SELECT id, email FROM users WHERE email LIKE $1"
    )
    .bind(format!("%{}%", search))
    .fetch_all(pool)
    .await?;
    
    for row in rows {
        use sqlx::Row;
        let id: Uuid = row.get("id");
        let email: String = row.get("email");
        println!("{}: {}", id, email);
    }
    
    // ===== fetch methods =====
    
    // fetch_one - ต้องมี exactly 1 row (error ถ้าไม่มีหรือมากกว่า)
    let _user = sqlx::query_as!(
        UserBasic,
        "SELECT id, email, username FROM users LIMIT 1"
    )
    .fetch_one(pool)
    .await?;
    
    // fetch_optional - 0 หรือ 1 row
    let _user = sqlx::query_as!(
        UserBasic,
        "SELECT id, email, username FROM users WHERE id = $1",
        user_id
    )
    .fetch_optional(pool)
    .await?;
    
    // fetch_all - หลาย rows
    let _users = sqlx::query_as!(
        UserBasic,
        "SELECT id, email, username FROM users LIMIT 10"
    )
    .fetch_all(pool)
    .await?;
    
    // fetch (stream) - สำหรับ large result sets
    use futures::TryStreamExt;
    let mut stream = sqlx::query_as!(
        UserBasic,
        "SELECT id, email, username FROM users"
    )
    .fetch(pool);
    
    while let Some(user) = stream.try_next().await? {
        println!("Processing: {}", user.email);
    }
    
    Ok(())
}
```

---

## Transaction Basics

### การใช้ Transaction

```rust
// src/repositories/user_repository.rs (ต่อ)

impl UserRepository {
    // Transaction สำหรับการสร้าง user พร้อม profile
    pub async fn create_with_profile(
        &self,
        dto: CreateUserDto,
    ) -> Result<User, AppError> {
        // เริ่ม transaction
        let mut tx = self.pool.begin().await.map_err(AppError::Database)?;
        
        // สร้าง user
        let user = sqlx::query_as!(
            User,
            r#"
            INSERT INTO users (email, username, password_hash, display_name)
            VALUES ($1, $2, $3, $4)
            RETURNING 
                id, email, username, password_hash,
                display_name, bio, avatar_url, website_url,
                is_active, is_admin, is_verified,
                last_login_at, created_at, updated_at, deleted_at
            "#,
            dto.email.to_lowercase(),
            dto.username,
            dto.password_hash,
            dto.display_name
        )
        .fetch_one(&mut *tx)
        .await
        .map_err(|e| {
            // log error
            log::error!("Failed to create user: {}", e);
            AppError::Database(e)
        })?;
        
        // สร้าง audit log
        sqlx::query!(
            r#"
            INSERT INTO audit_logs (action, resource_type, resource_id, new_data)
            VALUES ('user.created', 'user', $1, $2)
            "#,
            user.id,
            serde_json::json!({
                "email": user.email,
                "username": user.username
            })
        )
        .execute(&mut *tx)
        .await
        .map_err(AppError::Database)?;
        
        // Commit transaction
        tx.commit().await.map_err(AppError::Database)?;
        
        Ok(user)
    }
    
    // Transaction ที่อาจ rollback
    pub async fn transfer_admin_role(
        &self,
        from_id: Uuid,
        to_id: Uuid,
    ) -> Result<(), AppError> {
        let mut tx = self.pool.begin().await.map_err(AppError::Database)?;
        
        // ตรวจสอบ from_user เป็น admin
        let from_user = sqlx::query!(
            "SELECT is_admin FROM users WHERE id = $1 AND deleted_at IS NULL",
            from_id
        )
        .fetch_optional(&mut *tx)
        .await
        .map_err(AppError::Database)?
        .ok_or(AppError::NotFound("Source user not found".to_string()))?;
        
        if !from_user.is_admin {
            return Err(AppError::BadRequest("Source user is not an admin".to_string()));
        }
        
        // ถอด admin จาก from_user
        sqlx::query!(
            "UPDATE users SET is_admin = false WHERE id = $1",
            from_id
        )
        .execute(&mut *tx)
        .await
        .map_err(AppError::Database)?;
        
        // ให้ admin กับ to_user
        let result = sqlx::query!(
            "UPDATE users SET is_admin = true WHERE id = $1 AND deleted_at IS NULL",
            to_id
        )
        .execute(&mut *tx)
        .await
        .map_err(AppError::Database)?;
        
        if result.rows_affected() == 0 {
            // Rollback เมื่อ error
            tx.rollback().await.map_err(AppError::Database)?;
            return Err(AppError::NotFound("Target user not found".to_string()));
        }
        
        // Commit
        tx.commit().await.map_err(AppError::Database)?;
        
        Ok(())
    }
}
```

---

## Bulk Insert

### การ Insert หลาย Records พร้อมกัน

```rust
// src/repositories/post_repository.rs

use crate::models::post::Post;

pub struct PostRepository {
    pool: PgPool,
}

#[derive(Debug, Deserialize)]
pub struct CreatePostDto {
    pub title: String,
    pub slug: String,
    pub content: String,
    pub excerpt: Option<String>,
    pub author_id: Uuid,
    pub category_id: Option<Uuid>,
    pub tag_ids: Vec<Uuid>,
}

impl PostRepository {
    // Bulk insert ด้วย unnest
    pub async fn bulk_create(&self, posts: Vec<CreatePostDto>) -> Result<Vec<Post>, AppError> {
        if posts.is_empty() {
            return Ok(vec![]);
        }
        
        // Prepare arrays สำหรับ unnest
        let titles: Vec<String> = posts.iter().map(|p| p.title.clone()).collect();
        let slugs: Vec<String> = posts.iter().map(|p| p.slug.clone()).collect();
        let contents: Vec<String> = posts.iter().map(|p| p.content.clone()).collect();
        let author_ids: Vec<Uuid> = posts.iter().map(|p| p.author_id).collect();
        
        let created_posts = sqlx::query_as!(
            Post,
            r#"
            INSERT INTO posts (title, slug, content, author_id)
            SELECT * FROM UNNEST($1::text[], $2::text[], $3::text[], $4::uuid[])
            RETURNING 
                id, title, slug, content, excerpt,
                cover_image_url, status, published_at,
                author_id, category_id,
                view_count, like_count, comment_count,
                meta_title, meta_description,
                created_at, updated_at, deleted_at
            "#,
            &titles,
            &slugs,
            &contents,
            &author_ids as &[Uuid]
        )
        .fetch_all(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        Ok(created_posts)
    }
    
    // Bulk insert ด้วย transaction
    pub async fn bulk_create_with_tags(
        &self,
        posts: Vec<CreatePostDto>
    ) -> Result<Vec<Post>, AppError> {
        let mut tx = self.pool.begin().await.map_err(AppError::Database)?;
        let mut created_posts = Vec::new();
        
        for post_dto in posts {
            let post = sqlx::query_as!(
                Post,
                r#"
                INSERT INTO posts (title, slug, content, excerpt, author_id, category_id)
                VALUES ($1, $2, $3, $4, $5, $6)
                RETURNING 
                    id, title, slug, content, excerpt,
                    cover_image_url, status, published_at,
                    author_id, category_id,
                    view_count, like_count, comment_count,
                    meta_title, meta_description,
                    created_at, updated_at, deleted_at
                "#,
                post_dto.title,
                post_dto.slug,
                post_dto.content,
                post_dto.excerpt,
                post_dto.author_id,
                post_dto.category_id
            )
            .fetch_one(&mut *tx)
            .await
            .map_err(AppError::Database)?;
            
            // Insert tags
            if !post_dto.tag_ids.is_empty() {
                let post_ids: Vec<Uuid> = vec![post.id; post_dto.tag_ids.len()];
                
                sqlx::query!(
                    r#"
                    INSERT INTO post_tags (post_id, tag_id)
                    SELECT * FROM UNNEST($1::uuid[], $2::uuid[])
                    ON CONFLICT DO NOTHING
                    "#,
                    &post_ids as &[Uuid],
                    &post_dto.tag_ids as &[Uuid]
                )
                .execute(&mut *tx)
                .await
                .map_err(AppError::Database)?;
            }
            
            created_posts.push(post);
        }
        
        tx.commit().await.map_err(AppError::Database)?;
        
        Ok(created_posts)
    }
}
```

---

## Complete Post CRUD Repository

### Post Repository ครบชุด

```rust
// src/repositories/post_repository.rs (ครบชุด)
use sqlx::PgPool;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use crate::errors::AppError;

#[derive(Debug, Clone, Serialize, sqlx::FromRow)]
pub struct Post {
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

#[derive(Debug, Clone, Serialize, sqlx::FromRow)]
pub struct PostWithAuthor {
    pub id: Uuid,
    pub title: String,
    pub slug: String,
    pub excerpt: Option<String>,
    pub status: String,
    pub published_at: Option<DateTime<Utc>>,
    pub view_count: i32,
    pub like_count: i32,
    pub comment_count: i32,
    pub author_id: Uuid,
    pub author_name: String,
    pub author_avatar: Option<String>,
    pub category_id: Option<Uuid>,
    pub category_name: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct CreatePostDto {
    pub title: String,
    pub content: String,
    pub excerpt: Option<String>,
    pub cover_image_url: Option<String>,
    pub category_id: Option<Uuid>,
    pub tag_ids: Option<Vec<Uuid>>,
    pub status: Option<String>,  // default: "draft"
    pub meta_title: Option<String>,
    pub meta_description: Option<String>,
}

#[derive(Debug, Deserialize)]
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

#[derive(Debug, Deserialize)]
pub struct PostFilter {
    pub search: Option<String>,
    pub status: Option<String>,
    pub author_id: Option<Uuid>,
    pub category_id: Option<Uuid>,
    pub tag_id: Option<Uuid>,
    pub page: Option<i64>,
    pub per_page: Option<i64>,
}

#[derive(Debug, Serialize)]
pub struct PaginatedPosts {
    pub data: Vec<PostWithAuthor>,
    pub total: i64,
    pub page: i64,
    pub per_page: i64,
    pub total_pages: i64,
}

pub struct PostRepository {
    pool: PgPool,
}

impl PostRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
    
    // CREATE
    pub async fn create(
        &self,
        author_id: Uuid,
        dto: CreatePostDto
    ) -> Result<Post, AppError> {
        // สร้าง slug จาก title
        let slug = slugify(&dto.title);
        let status = dto.status.unwrap_or_else(|| "draft".to_string());
        
        let published_at = if status == "published" {
            Some(Utc::now())
        } else {
            None
        };
        
        let mut tx = self.pool.begin().await.map_err(AppError::Database)?;
        
        let post = sqlx::query_as!(
            Post,
            r#"
            INSERT INTO posts (
                title, slug, content, excerpt, cover_image_url,
                status, published_at, author_id, category_id,
                meta_title, meta_description
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
            RETURNING 
                id, title, slug, content, excerpt,
                cover_image_url, status, published_at,
                author_id, category_id,
                view_count, like_count, comment_count,
                meta_title, meta_description,
                created_at, updated_at, deleted_at
            "#,
            dto.title,
            slug,
            dto.content,
            dto.excerpt,
            dto.cover_image_url,
            status,
            published_at,
            author_id,
            dto.category_id,
            dto.meta_title,
            dto.meta_description
        )
        .fetch_one(&mut *tx)
        .await
        .map_err(AppError::Database)?;
        
        // Add tags ถ้ามี
        if let Some(tag_ids) = dto.tag_ids {
            if !tag_ids.is_empty() {
                let post_ids: Vec<Uuid> = vec![post.id; tag_ids.len()];
                sqlx::query!(
                    r#"
                    INSERT INTO post_tags (post_id, tag_id)
                    SELECT * FROM UNNEST($1::uuid[], $2::uuid[])
                    ON CONFLICT DO NOTHING
                    "#,
                    &post_ids as &[Uuid],
                    &tag_ids as &[Uuid]
                )
                .execute(&mut *tx)
                .await
                .map_err(AppError::Database)?;
            }
        }
        
        tx.commit().await.map_err(AppError::Database)?;
        
        Ok(post)
    }
    
    // READ ONE
    pub async fn find_by_id(&self, id: Uuid) -> Result<Option<Post>, AppError> {
        let post = sqlx::query_as!(
            Post,
            r#"
            SELECT 
                id, title, slug, content, excerpt,
                cover_image_url, status, published_at,
                author_id, category_id,
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
        .map_err(AppError::Database)?;
        
        Ok(post)
    }
    
    pub async fn find_by_slug(&self, slug: &str) -> Result<Option<Post>, AppError> {
        let post = sqlx::query_as!(
            Post,
            r#"
            SELECT 
                id, title, slug, content, excerpt,
                cover_image_url, status, published_at,
                author_id, category_id,
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
        .map_err(AppError::Database)?;
        
        Ok(post)
    }
    
    // READ LIST
    pub async fn find_all(&self, filter: PostFilter) -> Result<PaginatedPosts, AppError> {
        let page = filter.page.unwrap_or(1).max(1);
        let per_page = filter.per_page.unwrap_or(10).min(50);
        let offset = (page - 1) * per_page;
        
        let total: i64 = sqlx::query_scalar!(
            r#"
            SELECT COUNT(DISTINCT p.id) as "count!"
            FROM posts p
            LEFT JOIN post_tags pt ON p.id = pt.post_id
            WHERE p.deleted_at IS NULL
                AND ($1::text IS NULL OR p.status = $1)
                AND ($2::uuid IS NULL OR p.author_id = $2)
                AND ($3::uuid IS NULL OR p.category_id = $3)
                AND ($4::uuid IS NULL OR pt.tag_id = $4)
                AND ($5::text IS NULL OR p.search_vector @@ plainto_tsquery($5))
            "#,
            filter.status,
            filter.author_id,
            filter.category_id,
            filter.tag_id,
            filter.search
        )
        .fetch_one(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        let posts = sqlx::query_as!(
            PostWithAuthor,
            r#"
            SELECT DISTINCT
                p.id, p.title, p.slug, p.excerpt, p.status,
                p.published_at, p.view_count, p.like_count, p.comment_count,
                p.author_id,
                u.display_name as author_name,
                u.avatar_url as author_avatar,
                p.category_id,
                c.name as category_name,
                p.created_at, p.updated_at
            FROM posts p
            INNER JOIN users u ON p.author_id = u.id
            LEFT JOIN categories c ON p.category_id = c.id
            LEFT JOIN post_tags pt ON p.id = pt.post_id
            WHERE p.deleted_at IS NULL
                AND ($1::text IS NULL OR p.status = $1)
                AND ($2::uuid IS NULL OR p.author_id = $2)
                AND ($3::uuid IS NULL OR p.category_id = $3)
                AND ($4::uuid IS NULL OR pt.tag_id = $4)
                AND ($5::text IS NULL OR p.search_vector @@ plainto_tsquery($5))
            ORDER BY p.created_at DESC
            LIMIT $6
            OFFSET $7
            "#,
            filter.status,
            filter.author_id,
            filter.category_id,
            filter.tag_id,
            filter.search,
            per_page,
            offset
        )
        .fetch_all(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        let total_pages = (total + per_page - 1) / per_page;
        
        Ok(PaginatedPosts {
            data: posts,
            total,
            page,
            per_page,
            total_pages,
        })
    }
    
    // UPDATE
    pub async fn update(
        &self,
        id: Uuid,
        author_id: Uuid,
        dto: UpdatePostDto
    ) -> Result<Post, AppError> {
        let published_at_update = dto.status.as_deref() == Some("published");
        
        let post = sqlx::query_as!(
            Post,
            r#"
            UPDATE posts
            SET
                title = COALESCE($3, title),
                content = COALESCE($4, content),
                excerpt = COALESCE($5, excerpt),
                cover_image_url = COALESCE($6, cover_image_url),
                category_id = COALESCE($7, category_id),
                status = COALESCE($8, status),
                meta_title = COALESCE($9, meta_title),
                meta_description = COALESCE($10, meta_description),
                published_at = CASE
                    WHEN $11 = true AND published_at IS NULL THEN NOW()
                    ELSE published_at
                END,
                updated_at = NOW()
            WHERE id = $1 AND author_id = $2 AND deleted_at IS NULL
            RETURNING 
                id, title, slug, content, excerpt,
                cover_image_url, status, published_at,
                author_id, category_id,
                view_count, like_count, comment_count,
                meta_title, meta_description,
                created_at, updated_at, deleted_at
            "#,
            id,
            author_id,
            dto.title,
            dto.content,
            dto.excerpt,
            dto.cover_image_url,
            dto.category_id,
            dto.status,
            dto.meta_title,
            dto.meta_description,
            published_at_update
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(AppError::Database)?
        .ok_or(AppError::NotFound("Post not found".to_string()))?;
        
        Ok(post)
    }
    
    // INCREMENT view count
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
    
    // SOFT DELETE
    pub async fn delete(&self, id: Uuid, author_id: Uuid) -> Result<(), AppError> {
        let result = sqlx::query!(
            r#"
            UPDATE posts
            SET deleted_at = NOW()
            WHERE id = $1 AND author_id = $2 AND deleted_at IS NULL
            "#,
            id,
            author_id
        )
        .execute(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        if result.rows_affected() == 0 {
            return Err(AppError::NotFound("Post not found".to_string()));
        }
        
        Ok(())
    }
    
    // COUNT
    pub async fn count_by_author(&self, author_id: Uuid) -> Result<i64, AppError> {
        let count = sqlx::query_scalar!(
            r#"
            SELECT COUNT(*) as "count!"
            FROM posts
            WHERE author_id = $1 AND deleted_at IS NULL
            "#,
            author_id
        )
        .fetch_one(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        Ok(count)
    }
}

// Helper function
fn slugify(text: &str) -> String {
    text.to_lowercase()
        .chars()
        .map(|c| if c.is_alphanumeric() { c } else { '-' })
        .collect::<String>()
        .split('-')
        .filter(|s| !s.is_empty())
        .collect::<Vec<_>>()
        .join("-")
}
```

### Error Types

```rust
// src/errors.rs
use actix_web::{HttpResponse, ResponseError};
use serde_json::json;
use std::fmt;

#[derive(Debug)]
pub enum AppError {
    Database(sqlx::Error),
    NotFound(String),
    Conflict(String),
    BadRequest(String),
    Unauthorized(String),
    Internal(String),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::Database(e) => write!(f, "Database error: {}", e),
            AppError::NotFound(msg) => write!(f, "Not found: {}", msg),
            AppError::Conflict(msg) => write!(f, "Conflict: {}", msg),
            AppError::BadRequest(msg) => write!(f, "Bad request: {}", msg),
            AppError::Unauthorized(msg) => write!(f, "Unauthorized: {}", msg),
            AppError::Internal(msg) => write!(f, "Internal error: {}", msg),
        }
    }
}

impl ResponseError for AppError {
    fn error_response(&self) -> HttpResponse {
        match self {
            AppError::Database(e) => {
                log::error!("Database error: {}", e);
                HttpResponse::InternalServerError().json(json!({
                    "error": "Internal server error"
                }))
            }
            AppError::NotFound(msg) => HttpResponse::NotFound().json(json!({
                "error": msg
            })),
            AppError::Conflict(msg) => HttpResponse::Conflict().json(json!({
                "error": msg
            })),
            AppError::BadRequest(msg) => HttpResponse::BadRequest().json(json!({
                "error": msg
            })),
            AppError::Unauthorized(msg) => HttpResponse::Unauthorized().json(json!({
                "error": msg
            })),
            AppError::Internal(msg) => {
                log::error!("Internal error: {}", msg);
                HttpResponse::InternalServerError().json(json!({
                    "error": "Internal server error"
                }))
            }
        }
    }
}
```

---

## Navigation

| ก่อนหน้า | หน้าหลัก | ถัดไป |
|---------|---------|------|
| [Part 032: Database Migrations](../part_032/README.md) | [README หลัก](../../README.md) | [Part 034: Connection Pooling](../part_034/README.md) |
