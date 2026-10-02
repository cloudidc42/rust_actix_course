# Part 032: Database Migrations ด้วย SQLx

## สารบัญ
- [การติดตั้ง sqlx-cli](#การติดตั้ง-sqlx-cli)
- [คำสั่ง Migration พื้นฐาน](#คำสั่ง-migration-พื้นฐาน)
- [รูปแบบการตั้งชื่อไฟล์ Migration](#รูปแบบการตั้งชื่อไฟล์-migration)
- [UP/DOWN Migrations](#updown-migrations)
- [การสร้างตารางและคอลัมน์](#การสร้างตารางและคอลัมน์)
- [การเพิ่ม Indexes](#การเพิ่ม-indexes)
- [Foreign Keys และ Constraints](#foreign-keys-และ-constraints)
- [Seeding Data](#seeding-data)
- [Environment-based Migrations](#environment-based-migrations)
- [Project: Complete Blog Schema](#project-complete-blog-schema)

---

## การติดตั้ง sqlx-cli

### ติดตั้ง sqlx-cli Tool

```bash
# ติดตั้งพร้อม PostgreSQL support
cargo install sqlx-cli --no-default-features --features native-tls,postgres

# หรือติดตั้งพร้อม all database support
cargo install sqlx-cli

# ตรวจสอบการติดตั้ง
sqlx --version
```

### ตั้งค่า Cargo.toml

```toml
[package]
name = "blog-api"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
sqlx = { version = "0.7", features = [
    "runtime-tokio-native-tls",
    "postgres",
    "macros",
    "migrate",
    "chrono",
    "uuid",
] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
dotenv = "0.15"
env_logger = "0.10"
log = "0.4"
```

### ตั้งค่า .env

```bash
# .env
DATABASE_URL=postgres://postgres:password@localhost:5432/blog_db

# สำหรับ Development
DATABASE_URL_DEV=postgres://postgres:password@localhost:5432/blog_dev

# สำหรับ Testing
DATABASE_URL_TEST=postgres://postgres:password@localhost:5432/blog_test
```

---

## คำสั่ง Migration พื้นฐาน

### การสร้าง Database

```bash
# สร้าง database ใหม่
sqlx database create

# ลบ database
sqlx database drop

# สร้าง database และ reset (ลบแล้วสร้างใหม่)
sqlx database reset
```

### คำสั่ง Migration

```bash
# สร้างไฟล์ migration ใหม่
sqlx migrate add create_users_table

# รัน migrations ทั้งหมดที่ยังไม่ได้รัน
sqlx migrate run

# ย้อน migration ล่าสุด
sqlx migrate revert

# แสดงสถานะ migration ทั้งหมด
sqlx migrate info

# รัน migration พร้อม dry-run (ไม่เปลี่ยนแปลงจริง)
sqlx migrate run --dry-run
```

### การ Embed Migrations ใน Code

```rust
// src/db/migrations.rs
use sqlx::PgPool;

pub async fn run_migrations(pool: &PgPool) -> Result<(), sqlx::migrate::MigrateError> {
    // วิธีที่ 1: ใช้ macro (compile-time)
    sqlx::migrate!("./migrations").run(pool).await?;
    
    Ok(())
}

// src/main.rs
use sqlx::postgres::PgPoolOptions;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    dotenv::dotenv().ok();
    env_logger::init();
    
    let database_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await
        .expect("Failed to connect to database");
    
    // รัน migrations อัตโนมัติเมื่อ app เริ่มต้น
    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .expect("Failed to run migrations");
    
    log::info!("Migrations completed successfully");
    
    // ... ส่วนที่เหลือของ main
    Ok(())
}
```

---

## รูปแบบการตั้งชื่อไฟล์ Migration

### รูปแบบชื่อไฟล์

```
{timestamp}_{description}.sql
```

ตัวอย่าง:
```
20240101000001_create_users_table.sql
20240101000002_create_categories_table.sql
20240101000003_create_posts_table.sql
20240101000004_create_tags_table.sql
20240101000005_create_comments_table.sql
20240101000006_add_indexes.sql
20240101000007_seed_initial_data.sql
```

### โครงสร้างโฟลเดอร์

```
project/
├── migrations/
│   ├── 20240101000001_create_users_table.sql
│   ├── 20240101000002_create_categories_table.sql
│   ├── 20240101000003_create_posts_table.sql
│   └── ...
├── src/
│   └── main.rs
└── Cargo.toml
```

---

## UP/DOWN Migrations

### รูปแบบ UP/DOWN ใน Single File

SQLx ใช้รูปแบบ `-- +migrate Up` และ `-- +migrate Down`:

```sql
-- migrations/20240101000001_create_users_table.sql

-- UP migration: สร้างตาราง
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    display_name VARCHAR(255),
    bio TEXT,
    avatar_url TEXT,
    is_active BOOLEAN NOT NULL DEFAULT true,
    is_admin BOOLEAN NOT NULL DEFAULT false,
    email_verified BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- สร้าง function สำหรับ auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

-- สร้าง trigger
CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### Reversible Migrations (ไฟล์แยก)

SQLx รองรับการแยกไฟล์ UP และ DOWN:

```bash
# สร้าง reversible migration
sqlx migrate add --reversible create_posts_table
# จะสร้าง 2 ไฟล์:
# 20240101000003_create_posts_table.up.sql
# 20240101000003_create_posts_table.down.sql
```

```sql
-- migrations/20240101000003_create_posts_table.up.sql
CREATE TABLE posts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(500) NOT NULL,
    slug VARCHAR(500) UNIQUE NOT NULL,
    content TEXT NOT NULL,
    excerpt TEXT,
    cover_image_url TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'draft',
    published_at TIMESTAMPTZ,
    author_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    category_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    view_count INTEGER NOT NULL DEFAULT 0,
    like_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    CONSTRAINT valid_status CHECK (status IN ('draft', 'published', 'archived'))
);

CREATE TRIGGER update_posts_updated_at
    BEFORE UPDATE ON posts
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

```sql
-- migrations/20240101000003_create_posts_table.down.sql
DROP TRIGGER IF EXISTS update_posts_updated_at ON posts;
DROP TABLE IF EXISTS posts;
```

---

## การสร้างตารางและคอลัมน์

### Migration: Create Users Table

```sql
-- migrations/20240101000001_create_users_table.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(100) UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    
    -- Profile information
    display_name VARCHAR(255),
    bio TEXT,
    avatar_url TEXT,
    website_url TEXT,
    location VARCHAR(255),
    
    -- Account status
    is_active BOOLEAN NOT NULL DEFAULT true,
    is_admin BOOLEAN NOT NULL DEFAULT false,
    is_verified BOOLEAN NOT NULL DEFAULT false,
    
    -- Settings
    notification_email BOOLEAN NOT NULL DEFAULT true,
    notification_push BOOLEAN NOT NULL DEFAULT false,
    
    -- Timestamps
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ
);

-- Function สำหรับ auto-update
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

COMMENT ON TABLE users IS 'ตารางผู้ใช้งานระบบ';
COMMENT ON COLUMN users.password_hash IS 'Password ที่ผ่านการ hash แล้ว (bcrypt)';
COMMENT ON COLUMN users.deleted_at IS 'Soft delete timestamp';
```

### Migration: Create Categories Table

```sql
-- migrations/20240101000002_create_categories_table.sql
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    description TEXT,
    color VARCHAR(7),  -- HEX color code
    icon VARCHAR(50),
    parent_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    sort_order INTEGER NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT true,
    post_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TRIGGER update_categories_updated_at
    BEFORE UPDATE ON categories
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### Migration: Add Columns

```sql
-- migrations/20240101000008_add_user_profile_fields.sql

-- เพิ่มคอลัมน์ใหม่
ALTER TABLE users
    ADD COLUMN IF NOT EXISTS phone VARCHAR(20),
    ADD COLUMN IF NOT EXISTS birth_date DATE,
    ADD COLUMN IF NOT EXISTS gender VARCHAR(10),
    ADD COLUMN IF NOT EXISTS timezone VARCHAR(50) DEFAULT 'UTC';

-- เพิ่ม constraint
ALTER TABLE users
    ADD CONSTRAINT valid_gender 
    CHECK (gender IN ('male', 'female', 'other', 'prefer_not_to_say'));

-- แก้ไขคอลัมน์
ALTER TABLE users
    ALTER COLUMN display_name SET NOT NULL,
    ALTER COLUMN display_name SET DEFAULT '';
```

---

## การเพิ่ม Indexes

### Migration: Add Indexes

```sql
-- migrations/20240101000006_add_indexes.sql

-- Index สำหรับ users
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
CREATE INDEX CONCURRENTLY idx_users_username ON users(username);
CREATE INDEX CONCURRENTLY idx_users_created_at ON users(created_at DESC);
CREATE INDEX CONCURRENTLY idx_users_is_active ON users(is_active) WHERE is_active = true;

-- Composite index
CREATE INDEX CONCURRENTLY idx_users_active_admin 
    ON users(is_active, is_admin) 
    WHERE is_active = true;

-- Index สำหรับ posts
CREATE INDEX CONCURRENTLY idx_posts_author_id ON posts(author_id);
CREATE INDEX CONCURRENTLY idx_posts_category_id ON posts(category_id);
CREATE INDEX CONCURRENTLY idx_posts_status ON posts(status);
CREATE INDEX CONCURRENTLY idx_posts_published_at ON posts(published_at DESC);
CREATE INDEX CONCURRENTLY idx_posts_slug ON posts(slug);

-- Composite index สำหรับ query ที่ใช้บ่อย
CREATE INDEX CONCURRENTLY idx_posts_status_published 
    ON posts(status, published_at DESC)
    WHERE status = 'published';

-- GIN Index สำหรับ Full-text search
CREATE INDEX CONCURRENTLY idx_posts_content_fts 
    ON posts USING GIN(to_tsvector('english', title || ' ' || content));

-- Partial index
CREATE INDEX CONCURRENTLY idx_posts_deleted 
    ON posts(deleted_at) 
    WHERE deleted_at IS NOT NULL;

-- Index สำหรับ tags
CREATE INDEX CONCURRENTLY idx_post_tags_post_id ON post_tags(post_id);
CREATE INDEX CONCURRENTLY idx_post_tags_tag_id ON post_tags(tag_id);
```

---

## Foreign Keys และ Constraints

### Migration: Posts with Constraints

```sql
-- migrations/20240101000003_create_posts_table.sql
CREATE TABLE posts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(500) NOT NULL,
    slug VARCHAR(500) UNIQUE NOT NULL,
    content TEXT NOT NULL,
    excerpt TEXT,
    cover_image_url TEXT,
    
    -- Status management
    status VARCHAR(20) NOT NULL DEFAULT 'draft',
    published_at TIMESTAMPTZ,
    
    -- Relations
    author_id UUID NOT NULL,
    category_id UUID,
    
    -- Stats
    view_count INTEGER NOT NULL DEFAULT 0,
    like_count INTEGER NOT NULL DEFAULT 0,
    comment_count INTEGER NOT NULL DEFAULT 0,
    
    -- Metadata
    meta_title VARCHAR(255),
    meta_description TEXT,
    
    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ,
    
    -- Constraints
    CONSTRAINT valid_status CHECK (status IN ('draft', 'published', 'archived')),
    CONSTRAINT valid_view_count CHECK (view_count >= 0),
    CONSTRAINT valid_like_count CHECK (like_count >= 0),
    CONSTRAINT published_requires_date CHECK (
        (status = 'published' AND published_at IS NOT NULL) OR
        (status != 'published')
    ),
    
    -- Foreign Keys
    CONSTRAINT fk_posts_author 
        FOREIGN KEY (author_id) 
        REFERENCES users(id) 
        ON DELETE CASCADE 
        ON UPDATE CASCADE,
        
    CONSTRAINT fk_posts_category 
        FOREIGN KEY (category_id) 
        REFERENCES categories(id) 
        ON DELETE SET NULL 
        ON UPDATE CASCADE
);

-- Tags table
CREATE TABLE tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    color VARCHAR(7),
    post_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Many-to-many: Post Tags
CREATE TABLE post_tags (
    post_id UUID NOT NULL,
    tag_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    PRIMARY KEY (post_id, tag_id),
    
    CONSTRAINT fk_post_tags_post 
        FOREIGN KEY (post_id) 
        REFERENCES posts(id) 
        ON DELETE CASCADE,
        
    CONSTRAINT fk_post_tags_tag 
        FOREIGN KEY (tag_id) 
        REFERENCES tags(id) 
        ON DELETE CASCADE
);

CREATE TRIGGER update_posts_updated_at
    BEFORE UPDATE ON posts
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### Migration: Comments with Self-Reference

```sql
-- migrations/20240101000005_create_comments_table.sql
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content TEXT NOT NULL,
    
    -- Relations
    post_id UUID NOT NULL,
    author_id UUID NOT NULL,
    parent_id UUID,  -- สำหรับ nested comments
    
    -- Status
    is_approved BOOLEAN NOT NULL DEFAULT false,
    is_spam BOOLEAN NOT NULL DEFAULT false,
    
    -- Stats
    like_count INTEGER NOT NULL DEFAULT 0,
    
    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ,
    
    CONSTRAINT fk_comments_post
        FOREIGN KEY (post_id)
        REFERENCES posts(id)
        ON DELETE CASCADE,
        
    CONSTRAINT fk_comments_author
        FOREIGN KEY (author_id)
        REFERENCES users(id)
        ON DELETE CASCADE,
        
    CONSTRAINT fk_comments_parent
        FOREIGN KEY (parent_id)
        REFERENCES comments(id)
        ON DELETE CASCADE,
        
    CONSTRAINT valid_content CHECK (length(trim(content)) > 0)
);

CREATE TRIGGER update_comments_updated_at
    BEFORE UPDATE ON comments
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

---

## Seeding Data

### Migration: Seed Initial Data

```sql
-- migrations/20240101000007_seed_initial_data.sql

-- Insert default admin user
INSERT INTO users (
    id,
    email,
    username,
    password_hash,
    display_name,
    is_admin,
    is_verified,
    is_active
) VALUES (
    'a0000000-0000-0000-0000-000000000001',
    'admin@blog.com',
    'admin',
    '$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewYpfpJkGmTk1nM.',  -- 'admin123'
    'Blog Administrator',
    true,
    true,
    true
);

-- Insert default categories
INSERT INTO categories (id, name, slug, description, color, sort_order) VALUES
    ('b0000000-0000-0000-0000-000000000001', 'Technology', 'technology', 'บทความเกี่ยวกับเทคโนโลยี', '#3B82F6', 1),
    ('b0000000-0000-0000-0000-000000000002', 'Programming', 'programming', 'บทความเกี่ยวกับการเขียนโปรแกรม', '#10B981', 2),
    ('b0000000-0000-0000-0000-000000000003', 'DevOps', 'devops', 'บทความเกี่ยวกับ DevOps', '#F59E0B', 3),
    ('b0000000-0000-0000-0000-000000000004', 'Tutorial', 'tutorial', 'บทเรียนและ Tutorial', '#EF4444', 4),
    ('b0000000-0000-0000-0000-000000000005', 'News', 'news', 'ข่าวสารและอัพเดต', '#8B5CF6', 5);

-- Insert default tags
INSERT INTO tags (id, name, slug, color) VALUES
    ('c0000000-0000-0000-0000-000000000001', 'Rust', 'rust', '#F97316'),
    ('c0000000-0000-0000-0000-000000000002', 'Python', 'python', '#3B82F6'),
    ('c0000000-0000-0000-0000-000000000003', 'Docker', 'docker', '#06B6D4'),
    ('c0000000-0000-0000-0000-000000000004', 'Kubernetes', 'kubernetes', '#6366F1'),
    ('c0000000-0000-0000-0000-000000000005', 'PostgreSQL', 'postgresql', '#0EA5E9');
```

### Conditional Seed (เฉพาะ Development)

```sql
-- migrations/20240101000009_seed_dev_data.sql
-- ใช้ DO block เพื่อ conditional insert

DO $$
DECLARE
    v_env TEXT;
BEGIN
    -- ตรวจสอบว่าเป็น dev environment
    -- ใน production จะ skip
    v_env := current_setting('app.environment', true);
    
    IF v_env IS NULL OR v_env = 'development' THEN
        -- Insert sample posts
        INSERT INTO posts (
            title, slug, content, status, published_at,
            author_id, category_id
        )
        SELECT 
            'Sample Post ' || i,
            'sample-post-' || i,
            'This is sample content for post ' || i || '. Lorem ipsum dolor sit amet.',
            'published',
            NOW() - (i * interval '1 day'),
            'a0000000-0000-0000-0000-000000000001',
            'b0000000-0000-0000-0000-000000000002'
        FROM generate_series(1, 10) i;
        
        RAISE NOTICE 'Development seed data inserted';
    ELSE
        RAISE NOTICE 'Skipping dev seed data for environment: %', v_env;
    END IF;
END $$;
```

---

## Environment-based Migrations

### การจัดการ Migration ตาม Environment

```rust
// src/config.rs
use std::env;

#[derive(Debug, Clone)]
pub struct DatabaseConfig {
    pub url: String,
    pub max_connections: u32,
    pub min_connections: u32,
    pub run_migrations: bool,
    pub seed_data: bool,
}

impl DatabaseConfig {
    pub fn from_env() -> Self {
        let environment = env::var("APP_ENV").unwrap_or_else(|_| "development".to_string());
        
        let (run_migrations, seed_data) = match environment.as_str() {
            "production" => (true, false),
            "staging" => (true, false),
            "development" => (true, true),
            "test" => (true, true),
            _ => (false, false),
        };
        
        DatabaseConfig {
            url: env::var("DATABASE_URL").expect("DATABASE_URL must be set"),
            max_connections: env::var("DB_MAX_CONNECTIONS")
                .unwrap_or_else(|_| "10".to_string())
                .parse()
                .unwrap_or(10),
            min_connections: env::var("DB_MIN_CONNECTIONS")
                .unwrap_or_else(|_| "2".to_string())
                .parse()
                .unwrap_or(2),
            run_migrations,
            seed_data,
        }
    }
}
```

```rust
// src/db/mod.rs
use sqlx::PgPool;
use sqlx::postgres::PgPoolOptions;
use crate::config::DatabaseConfig;

pub async fn create_pool(config: &DatabaseConfig) -> Result<PgPool, sqlx::Error> {
    let pool = PgPoolOptions::new()
        .max_connections(config.max_connections)
        .min_connections(config.min_connections)
        .acquire_timeout(std::time::Duration::from_secs(30))
        .connect(&config.url)
        .await?;
    
    if config.run_migrations {
        log::info!("Running database migrations...");
        sqlx::migrate!("./migrations")
            .run(&pool)
            .await
            .map_err(|e| {
                log::error!("Migration failed: {}", e);
                e
            })?;
        log::info!("Migrations completed");
    }
    
    Ok(pool)
}
```

---

## Project: Complete Blog Schema

### ไฟล์ Migration ครบชุด

```sql
-- migrations/20240101000001_initial_schema.sql

-- Enable extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- สำหรับ fuzzy search

-- Auto-update function
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

-- ===== USERS =====
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(100) UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    display_name VARCHAR(255) NOT NULL DEFAULT '',
    bio TEXT,
    avatar_url TEXT,
    website_url TEXT,
    is_active BOOLEAN NOT NULL DEFAULT true,
    is_admin BOOLEAN NOT NULL DEFAULT false,
    is_verified BOOLEAN NOT NULL DEFAULT false,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ
);

CREATE TRIGGER trg_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ===== CATEGORIES =====
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    description TEXT,
    color VARCHAR(7),
    parent_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    sort_order INTEGER NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT true,
    post_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TRIGGER trg_categories_updated_at
    BEFORE UPDATE ON categories
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ===== POSTS =====
CREATE TABLE posts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(500) NOT NULL,
    slug VARCHAR(500) UNIQUE NOT NULL,
    content TEXT NOT NULL,
    excerpt TEXT,
    cover_image_url TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'draft',
    published_at TIMESTAMPTZ,
    author_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    category_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    view_count INTEGER NOT NULL DEFAULT 0,
    like_count INTEGER NOT NULL DEFAULT 0,
    comment_count INTEGER NOT NULL DEFAULT 0,
    meta_title VARCHAR(255),
    meta_description TEXT,
    search_vector TSVECTOR,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ,
    CONSTRAINT chk_posts_status CHECK (status IN ('draft', 'published', 'archived')),
    CONSTRAINT chk_posts_counts CHECK (view_count >= 0 AND like_count >= 0)
);

CREATE TRIGGER trg_posts_updated_at
    BEFORE UPDATE ON posts
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- Auto-update search vector
CREATE OR REPLACE FUNCTION update_post_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector = 
        setweight(to_tsvector('english', coalesce(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(NEW.excerpt, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(NEW.content, '')), 'C');
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER trg_posts_search_vector
    BEFORE INSERT OR UPDATE ON posts
    FOR EACH ROW EXECUTE FUNCTION update_post_search_vector();

-- ===== TAGS =====
CREATE TABLE tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    color VARCHAR(7),
    post_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ===== POST TAGS =====
CREATE TABLE post_tags (
    post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    tag_id UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (post_id, tag_id)
);

-- ===== COMMENTS =====
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content TEXT NOT NULL,
    post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    author_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    parent_id UUID REFERENCES comments(id) ON DELETE CASCADE,
    is_approved BOOLEAN NOT NULL DEFAULT false,
    like_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ,
    CONSTRAINT chk_comments_content CHECK (length(trim(content)) > 0)
);

CREATE TRIGGER trg_comments_updated_at
    BEFORE UPDATE ON comments
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ===== POST LIKES =====
CREATE TABLE post_likes (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, post_id)
);

-- ===== MEDIA =====
CREATE TABLE media (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    filename VARCHAR(500) NOT NULL,
    original_filename VARCHAR(500) NOT NULL,
    mime_type VARCHAR(100) NOT NULL,
    file_size BIGINT NOT NULL,
    url TEXT NOT NULL,
    thumbnail_url TEXT,
    alt_text TEXT,
    uploaded_by UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ===== AUDIT LOG =====
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(100) NOT NULL,
    resource_id UUID,
    old_data JSONB,
    new_data JSONB,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ===== INDEXES =====
-- Users
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_active ON users(is_active) WHERE is_active = true;
CREATE INDEX idx_users_deleted ON users(deleted_at) WHERE deleted_at IS NOT NULL;

-- Posts
CREATE INDEX idx_posts_author_id ON posts(author_id);
CREATE INDEX idx_posts_category_id ON posts(category_id);
CREATE INDEX idx_posts_slug ON posts(slug);
CREATE INDEX idx_posts_status ON posts(status);
CREATE INDEX idx_posts_published ON posts(published_at DESC) WHERE status = 'published';
CREATE INDEX idx_posts_search ON posts USING GIN(search_vector);
CREATE INDEX idx_posts_title_trgm ON posts USING GIN(title gin_trgm_ops);

-- Comments
CREATE INDEX idx_comments_post_id ON comments(post_id);
CREATE INDEX idx_comments_author_id ON comments(author_id);
CREATE INDEX idx_comments_parent_id ON comments(parent_id);

-- Audit
CREATE INDEX idx_audit_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_created_at ON audit_logs(created_at DESC);
```

### Rust Structs สำหรับ Database Models

```rust
// src/models/user.rs
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use chrono::{DateTime, Utc};

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct User {
    pub id: Uuid,
    pub email: String,
    pub username: String,
    #[serde(skip_serializing)]
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

// src/models/post.rs
#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
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
```

---

## Navigation

| ก่อนหน้า | หน้าหลัก | ถัดไป |
|---------|---------|------|
| [Part 031: SQLx Setup](../part_031/README.md) | [README หลัก](../../README.md) | [Part 033: CRUD Operations](../part_033/README.md) |
