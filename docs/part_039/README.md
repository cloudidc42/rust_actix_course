# Part 039: Query Optimization

## สารบัญ
- [EXPLAIN ANALYZE](#explain-analyze)
- [Index Types](#index-types)
- [Composite Indexes](#composite-indexes)
- [Partial Indexes](#partial-indexes)
- [Query Planning](#query-planning)
- [N+1 Problem และ Solutions](#n1-problem-และ-solutions)
- [Connection Pool Tuning](#connection-pool-tuning)
- [Vacuum และ Maintenance](#vacuum-และ-maintenance)

---

## EXPLAIN ANALYZE

### การใช้ EXPLAIN ANALYZE

```sql
-- ดู query plan
EXPLAIN SELECT * FROM posts WHERE author_id = 'some-uuid';

-- ดู actual execution time และ rows
EXPLAIN ANALYZE SELECT * FROM posts WHERE author_id = 'some-uuid';

-- ดูแบบ verbose
EXPLAIN (ANALYZE, VERBOSE, BUFFERS) 
SELECT p.*, u.display_name
FROM posts p
JOIN users u ON p.author_id = u.id
WHERE p.status = 'published'
ORDER BY p.published_at DESC
LIMIT 10;

-- อ่านผลลัพธ์:
-- Seq Scan = Full table scan (ช้า ถ้าตารางใหญ่)
-- Index Scan = ใช้ index
-- Index Only Scan = ดึงข้อมูลจาก index โดยไม่ต้องไป heap
-- Bitmap Index Scan = ใช้หลาย index รวมกัน
-- Hash Join = JOIN ด้วย hash table
-- Nested Loop = JOIN แบบ nested loop
-- Merge Join = JOIN ด้วย sorted merge
```

### EXPLAIN ใน Rust

```rust
// src/db/explain.rs
use sqlx::PgPool;

#[derive(Debug)]
pub struct QueryPlan {
    pub plan_text: String,
    pub execution_time_ms: Option<f64>,
    pub planning_time_ms: Option<f64>,
}

pub async fn explain_query(
    pool: &PgPool,
    sql: &str,
) -> Result<QueryPlan, sqlx::Error> {
    let explain_sql = format!("EXPLAIN (ANALYZE, FORMAT JSON) {}", sql);
    
    let row = sqlx::query(&explain_sql)
        .fetch_one(pool)
        .await?;
    
    use sqlx::Row;
    let plan_json: serde_json::Value = row.get(0);
    
    let execution_time = plan_json[0]["Execution Time"].as_f64();
    let planning_time = plan_json[0]["Planning Time"].as_f64();
    
    Ok(QueryPlan {
        plan_text: serde_json::to_string_pretty(&plan_json).unwrap_or_default(),
        execution_time_ms: execution_time,
        planning_time_ms: planning_time,
    })
}

// Slow query logger
pub async fn log_slow_queries(pool: &PgPool, threshold_ms: f64) {
    let slow_queries = sqlx::query!(
        r#"
        SELECT
            query,
            calls,
            total_exec_time / calls as avg_exec_time_ms,
            total_exec_time,
            rows / calls as avg_rows,
            stddev_exec_time
        FROM pg_stat_statements
        WHERE total_exec_time / calls > $1
            AND calls > 10
        ORDER BY avg_exec_time_ms DESC
        LIMIT 20
        "#,
        threshold_ms
    )
    .fetch_all(pool)
    .await
    .unwrap_or_default();
    
    for q in slow_queries {
        log::warn!(
            "Slow query ({:.2}ms avg): {}",
            q.avg_exec_time_ms.unwrap_or(0.0),
            q.query.chars().take(200).collect::<String>()
        );
    }
}

// ติดตาม query performance ใน code
pub async fn timed_query<T, F, Fut>(
    name: &str,
    threshold_ms: u128,
    f: F,
) -> Result<T, sqlx::Error>
where
    F: FnOnce() -> Fut,
    Fut: std::future::Future<Output = Result<T, sqlx::Error>>,
{
    let start = std::time::Instant::now();
    let result = f().await;
    let elapsed = start.elapsed().as_millis();
    
    if elapsed > threshold_ms {
        log::warn!("Slow query '{}': {}ms", name, elapsed);
    } else {
        log::debug!("Query '{}': {}ms", name, elapsed);
    }
    
    result
}
```

### Query Analysis Examples

```sql
-- ตัวอย่าง EXPLAIN output ที่ดี vs แย่

-- BAD: Sequential scan บน table ใหญ่
-- Seq Scan on posts (cost=0.00..15420.00 rows=500000 width=1024)
-- (actual time=0.050..245.832 rows=500000 loops=1)
SELECT * FROM posts WHERE status = 'published';

-- GOOD: Index scan
-- Index Scan using idx_posts_status on posts (cost=0.42..8.45 rows=1000 width=1024)
-- (actual time=0.050..1.234 rows=1000 loops=1)
SELECT * FROM posts WHERE status = 'published'
LIMIT 1000;

-- Index สำหรับ query นี้:
CREATE INDEX idx_posts_status_published
ON posts(status, published_at DESC)
WHERE status = 'published';
```

---

## Index Types

### B-tree Index (Default)

```sql
-- B-tree: เหมาะสำหรับ =, <, >, BETWEEN, LIKE 'prefix%'
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_posts_created_at ON posts(created_at DESC);
CREATE INDEX idx_posts_title ON posts(title);

-- ใช้ INCLUDE เพื่อ Index Only Scan
CREATE INDEX idx_posts_status_inc ON posts(status)
INCLUDE (title, published_at, author_id);
```

### Hash Index

```sql
-- Hash: เหมาะสำหรับ = เท่านั้น (เร็วกว่า B-tree สำหรับ equality)
CREATE INDEX idx_posts_slug_hash ON posts USING HASH(slug);
CREATE INDEX idx_users_email_hash ON users USING HASH(email);
```

### GIN Index (สำหรับ Full-Text Search และ Arrays)

```sql
-- GIN: เหมาะสำหรับ full-text search, array, jsonb
CREATE INDEX idx_posts_search_gin ON posts USING GIN(search_vector);
CREATE INDEX idx_posts_metadata_gin ON posts USING GIN(metadata);

-- สำหรับ trigram search
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_posts_title_trgm ON posts USING GIN(title gin_trgm_ops);
CREATE INDEX idx_users_email_trgm ON users USING GIN(email gin_trgm_ops);
```

### GiST Index

```sql
-- GiST: เหมาะสำหรับ geometric data, range types, full-text
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE INDEX idx_posts_published_range ON posts USING GIST(
    tstzrange(published_at, published_at + interval '1 day')
);

-- สำหรับ tsvector
CREATE INDEX idx_posts_fts_gist ON posts USING GIST(search_vector);
```

### BRIN Index (สำหรับ Large Sequential Tables)

```sql
-- BRIN: ใช้กับ timestamp/date ที่เพิ่มขึ้นตามลำดับ
-- ขนาดเล็กมาก แต่เหมาะกับ range queries
CREATE INDEX idx_audit_created_brin ON audit_logs USING BRIN(created_at);
CREATE INDEX idx_posts_created_brin ON posts USING BRIN(created_at);
```

### Rust: Index Management

```rust
// src/db/indexes.rs
use sqlx::PgPool;

pub async fn create_performance_indexes(pool: &PgPool) -> Result<(), sqlx::Error> {
    // สร้าง indexes แบบ CONCURRENTLY (ไม่ lock table)
    sqlx::query!(
        "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_posts_author_status 
         ON posts(author_id, status) 
         WHERE deleted_at IS NULL"
    )
    .execute(pool)
    .await?;
    
    sqlx::query!(
        "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_posts_category_published 
         ON posts(category_id, published_at DESC) 
         WHERE status = 'published' AND deleted_at IS NULL"
    )
    .execute(pool)
    .await?;
    
    Ok(())
}

// ตรวจสอบ index usage
pub async fn check_index_usage(pool: &PgPool) -> Result<Vec<IndexUsage>, sqlx::Error> {
    #[derive(Debug, sqlx::FromRow)]
    struct IndexUsage {
        table_name: Option<String>,
        index_name: Option<String>,
        index_scans: Option<i64>,
        tuples_read: Option<i64>,
        tuples_fetched: Option<i64>,
    }
    
    sqlx::query_as!(
        IndexUsage,
        r#"
        SELECT
            schemaname || '.' || tablename as table_name,
            indexrelname as index_name,
            idx_scan as index_scans,
            idx_tup_read as tuples_read,
            idx_tup_fetch as tuples_fetched
        FROM pg_stat_user_indexes
        WHERE schemaname = 'public'
        ORDER BY idx_scan DESC
        "#
    )
    .fetch_all(pool)
    .await
}

// หา unused indexes
pub async fn find_unused_indexes(pool: &PgPool) {
    let rows = sqlx::query!(
        r#"
        SELECT
            schemaname,
            tablename,
            indexname,
            idx_scan
        FROM pg_stat_user_indexes
        WHERE idx_scan = 0
            AND schemaname = 'public'
            AND indexname NOT LIKE '%pkey%'
        ORDER BY tablename, indexname
        "#
    )
    .fetch_all(pool)
    .await
    .unwrap_or_default();
    
    for row in rows {
        log::info!(
            "Unused index: {}.{} on {}",
            row.schemaname.unwrap_or_default(),
            row.indexname.unwrap_or_default(),
            row.tablename.unwrap_or_default()
        );
    }
}
```

---

## Composite Indexes

### การออกแบบ Composite Index

```sql
-- Composite Index: ลำดับสำคัญมาก!
-- Rule: สร้าง index ตามลำดับที่ query ใช้ (equality first, range last)

-- Query: WHERE author_id = ? AND status = ? ORDER BY published_at DESC
-- Best composite index:
CREATE INDEX idx_posts_author_status_date 
ON posts(author_id, status, published_at DESC);

-- Query: WHERE category_id = ? AND status = 'published' ORDER BY view_count DESC
CREATE INDEX idx_posts_cat_status_views
ON posts(category_id, status, view_count DESC)
WHERE status = 'published';

-- Index สำหรับ pagination ที่มี filter
CREATE INDEX idx_posts_public_paginate
ON posts(status, published_at DESC, id)
WHERE deleted_at IS NULL;
```

### Covering Index

```sql
-- Covering Index: เพิ่ม columns ที่ SELECT ใน index เพื่อ Index Only Scan
-- ไม่ต้อง fetch จาก heap

-- Query: SELECT title, published_at FROM posts WHERE status = 'published'
-- Normal index: ต้อง fetch title, published_at จาก heap
-- Covering index: ดึงได้จาก index เลย
CREATE INDEX idx_posts_cover ON posts(status, published_at DESC)
INCLUDE (title, slug, excerpt, author_id);
```

### Rust: Querying with Composite Index

```rust
// src/repositories/post_repository.rs

// Query ที่ใช้ composite index
pub async fn find_by_author_and_status(
    pool: &PgPool,
    author_id: Uuid,
    status: &str,
    page: i64,
    per_page: i64,
) -> Result<Vec<Post>, sqlx::Error> {
    let offset = (page - 1) * per_page;
    
    // Query นี้ใช้ idx_posts_author_status_date
    sqlx::query_as!(
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
        WHERE author_id = $1
            AND status = $2
            AND deleted_at IS NULL
        ORDER BY published_at DESC
        LIMIT $3
        OFFSET $4
        "#,
        author_id,
        status,
        per_page,
        offset
    )
    .fetch_all(pool)
    .await
}
```

---

## Partial Indexes

### Partial Index สำหรับ Subset ของข้อมูล

```sql
-- Index เฉพาะ published posts (ใช้ space น้อยกว่า full index)
CREATE INDEX idx_posts_published_only
ON posts(published_at DESC, author_id)
WHERE status = 'published' AND deleted_at IS NULL;

-- Index สำหรับ active users
CREATE INDEX idx_users_active_email
ON users(email)
WHERE is_active = true AND deleted_at IS NULL;

-- Index สำหรับ approved comments
CREATE INDEX idx_comments_approved
ON comments(post_id, created_at DESC)
WHERE is_approved = true AND deleted_at IS NULL;

-- Index สำหรับ unverified users (อาจมีน้อย)
CREATE INDEX idx_users_unverified
ON users(created_at)
WHERE is_verified = false AND deleted_at IS NULL;
```

### การใช้ Partial Index อย่างถูกต้อง

```rust
// src/repositories/comment_repository.rs

// Query นี้ใช้ idx_comments_approved เพราะมี WHERE clause ตรงกัน
pub async fn get_approved_comments_for_post(
    pool: &PgPool,
    post_id: Uuid,
    limit: i64,
) -> Result<Vec<Comment>, sqlx::Error> {
    // Partial index จะถูกใช้เมื่อ WHERE clause ตรงกัน
    sqlx::query_as!(
        Comment,
        r#"
        SELECT id, content, post_id, author_id, parent_id,
            is_approved, like_count, created_at, updated_at, deleted_at
        FROM comments
        WHERE post_id = $1
            AND is_approved = true  -- ตรงกับ partial index condition
            AND deleted_at IS NULL  -- ตรงกับ partial index condition
        ORDER BY created_at DESC
        LIMIT $2
        "#,
        post_id,
        limit
    )
    .fetch_all(pool)
    .await
}
```

---

## Query Planning

### Forcing Index Usage (ระวัง!)

```sql
-- PostgreSQL เลือก plan อัตโนมัติ ซึ่งมักดีกว่า manual
-- แต่ถ้าต้องการทดสอบ:

-- ปิด sequential scan (ทดสอบ)
SET enable_seqscan = OFF;
EXPLAIN SELECT * FROM posts WHERE status = 'published';
SET enable_seqscan = ON;

-- ปิด nested loop
SET enable_nestloop = OFF;
EXPLAIN ANALYZE SELECT p.*, u.username
FROM posts p JOIN users u ON p.author_id = u.id;
SET enable_nestloop = ON;
```

### Statistics Update

```sql
-- Update statistics เพื่อ query planner ได้ข้อมูลที่ถูกต้อง
ANALYZE posts;
ANALYZE users;

-- Update ทั้ง database
ANALYZE;

-- ดู statistics
SELECT
    tablename,
    attname,
    n_distinct,
    correlation
FROM pg_stats
WHERE tablename = 'posts';

-- ปรับ statistics target (default 100, เพิ่มสำหรับ complex queries)
ALTER TABLE posts ALTER COLUMN status SET STATISTICS 500;
ANALYZE posts;
```

### Configuration Tuning

```sql
-- ปรับค่า postgresql.conf สำหรับ performance

-- Memory settings (ปรับตาม RAM ที่มี)
-- shared_buffers = 25% of RAM
-- effective_cache_size = 75% of RAM
-- work_mem = RAM / max_connections / 4
-- maintenance_work_mem = 256MB ถึง 1GB

-- ดู current settings
SHOW shared_buffers;
SHOW work_mem;
SHOW effective_cache_size;

-- Checkpoint settings
-- checkpoint_completion_target = 0.9
-- wal_buffers = 16MB
```

---

## N+1 Problem และ Solutions

### ปัญหา N+1

```rust
// src/examples/n_plus_one.rs

// BAD: N+1 problem
pub async fn bad_get_posts_with_authors(pool: &PgPool) -> Vec<PostWithAuthor> {
    // Query 1: Get all posts
    let posts = sqlx::query!("SELECT * FROM posts WHERE status = 'published' LIMIT 10")
        .fetch_all(pool)
        .await
        .unwrap();
    
    let mut result = Vec::new();
    for post in &posts {
        // Query 2..N+1: Get author for each post
        let author = sqlx::query!(
            "SELECT * FROM users WHERE id = $1",
            post.author_id
        )
        .fetch_one(pool)
        .await
        .unwrap();
        
        // result.push(PostWithAuthor { post, author });
    }
    
    // รวม 11 queries สำหรับ 10 posts!
    result
}

// GOOD: Solution 1 - JOIN
pub async fn good_get_posts_with_authors_join(
    pool: &PgPool,
    limit: i64
) -> Result<Vec<PostWithAuthorRow>, sqlx::Error> {
    // Query เดียว!
    sqlx::query_as!(
        PostWithAuthorRow,
        r#"
        SELECT
            p.id, p.title, p.slug, p.status, p.published_at,
            p.view_count, p.author_id,
            u.display_name as author_name,
            u.avatar_url as author_avatar
        FROM posts p
        INNER JOIN users u ON p.author_id = u.id
        WHERE p.status = 'published'
            AND p.deleted_at IS NULL
        ORDER BY p.published_at DESC
        LIMIT $1
        "#,
        limit
    )
    .fetch_all(pool)
    .await
}

#[derive(Debug, sqlx::FromRow)]
struct PostWithAuthorRow {
    pub id: uuid::Uuid,
    pub title: String,
    pub slug: String,
    pub status: String,
    pub published_at: Option<chrono::DateTime<chrono::Utc>>,
    pub view_count: i32,
    pub author_id: uuid::Uuid,
    pub author_name: String,
    pub author_avatar: Option<String>,
}

// GOOD: Solution 2 - Batch loading
pub async fn good_get_posts_with_authors_batch(
    pool: &PgPool
) -> Result<(), sqlx::Error> {
    // Query 1: Get posts
    let posts = sqlx::query!(
        "SELECT id, title, author_id FROM posts WHERE status = 'published' LIMIT 10"
    )
    .fetch_all(pool)
    .await?;
    
    // Collect unique author IDs
    let author_ids: Vec<Uuid> = posts.iter()
        .map(|p| p.author_id)
        .collect::<std::collections::HashSet<_>>()
        .into_iter()
        .collect();
    
    // Query 2: Get all authors in ONE query
    let authors = sqlx::query!(
        "SELECT id, display_name, avatar_url FROM users WHERE id = ANY($1)",
        &author_ids as &[Uuid]
    )
    .fetch_all(pool)
    .await?;
    
    // Create lookup map
    let author_map: std::collections::HashMap<Uuid, _> = authors
        .into_iter()
        .map(|a| (a.id, a))
        .collect();
    
    // Combine
    for post in &posts {
        if let Some(author) = author_map.get(&post.author_id) {
            println!("{} by {}", post.title, author.display_name);
        }
    }
    
    // รวมเพียง 2 queries!
    Ok(())
}
```

### DataLoader Pattern

```rust
// src/utils/dataloader.rs
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use uuid::Uuid;

// Simple DataLoader สำหรับ batch loading
pub struct UserLoader {
    pool: sqlx::PgPool,
    cache: Arc<Mutex<HashMap<Uuid, UserEntity>>>,
}

impl UserLoader {
    pub fn new(pool: sqlx::PgPool) -> Self {
        Self {
            pool,
            cache: Arc::new(Mutex::new(HashMap::new())),
        }
    }
    
    pub async fn load_many(&self, ids: &[Uuid]) -> Result<HashMap<Uuid, UserEntity>, AppError> {
        // ตรวจสอบ cache
        let cached = {
            let cache = self.cache.lock().unwrap();
            let mut result = HashMap::new();
            let mut missing = Vec::new();
            
            for &id in ids {
                if let Some(user) = cache.get(&id) {
                    result.insert(id, user.clone());
                } else {
                    missing.push(id);
                }
            }
            
            (result, missing)
        };
        
        let (mut result, missing) = cached;
        
        if !missing.is_empty() {
            // Batch load missing
            let users = sqlx::query_as!(
                UserEntity,
                r#"
                SELECT id, email, username, password_hash, display_name,
                    bio, avatar_url, website_url, is_active, is_admin, is_verified,
                    last_login_at, created_at, updated_at, deleted_at
                FROM users
                WHERE id = ANY($1) AND deleted_at IS NULL
                "#,
                &missing as &[Uuid]
            )
            .fetch_all(&self.pool)
            .await
            .map_err(AppError::Database)?;
            
            // Update cache
            let mut cache = self.cache.lock().unwrap();
            for user in users {
                cache.insert(user.id, user.clone());
                result.insert(user.id, user);
            }
        }
        
        Ok(result)
    }
}
```

---

## Connection Pool Tuning

### การวัดและปรับ Pool Size

```rust
// src/db/pool_monitor.rs
use sqlx::PgPool;
use std::time::Duration;

pub struct PoolMonitor {
    pool: PgPool,
    metrics: Arc<PoolMetrics>,
}

pub struct PoolMetrics {
    pub peak_active: std::sync::atomic::AtomicU32,
    pub total_waits: std::sync::atomic::AtomicU64,
    pub total_timeouts: std::sync::atomic::AtomicU64,
}

impl PoolMonitor {
    pub async fn run(&self, interval: Duration) {
        let mut ticker = tokio::time::interval(interval);
        
        loop {
            ticker.tick().await;
            
            let size = self.pool.size();
            let idle = self.pool.num_idle();
            let active = size - idle;
            
            // Track peak
            let current_peak = self.metrics.peak_active.load(
                std::sync::atomic::Ordering::Relaxed
            );
            if active > current_peak {
                self.metrics.peak_active.store(
                    active, std::sync::atomic::Ordering::Relaxed
                );
            }
            
            // Log
            log::info!(
                "Pool: active={}/{} ({:.1}%), idle={}",
                active, size,
                if size > 0 { active as f32 / size as f32 * 100.0 } else { 0.0 },
                idle
            );
            
            // Recommend tuning
            let utilization = if size > 0 {
                active as f32 / size as f32
            } else {
                0.0
            };
            
            if utilization > 0.9 {
                log::warn!(
                    "Pool nearly full ({:.0}%). Consider increasing max_connections from {}",
                    utilization * 100.0,
                    size
                );
            } else if utilization < 0.1 && size > 5 {
                log::info!(
                    "Pool underutilized ({:.0}%). Consider decreasing max_connections from {}",
                    utilization * 100.0,
                    size
                );
            }
        }
    }
}

// การคำนวณ optimal pool size
pub fn calculate_optimal_pool_size(
    expected_rps: u32,
    avg_query_time_ms: u32,
    safety_factor: f32,
) -> u32 {
    // Little's Law: L = λW
    // L = concurrent connections needed
    // λ = request rate per second
    // W = avg service time per request
    
    let connections_needed = 
        (expected_rps as f32 * avg_query_time_ms as f32 / 1000.0) * safety_factor;
    
    connections_needed.ceil() as u32
}
```

### Advanced Pool Settings

```rust
// src/config/pool_config.rs

pub struct OptimalPoolConfig {
    pub max_connections: u32,
    pub min_connections: u32,
    pub acquire_timeout: Duration,
    pub idle_timeout: Duration,
    pub max_lifetime: Duration,
}

impl OptimalPoolConfig {
    // สำหรับ OLTP (many small, fast queries)
    pub fn oltp() -> Self {
        let cpus = num_cpus::get() as u32;
        Self {
            max_connections: cpus * 8,
            min_connections: cpus,
            acquire_timeout: Duration::from_secs(5),
            idle_timeout: Duration::from_secs(300),
            max_lifetime: Duration::from_secs(1800),
        }
    }
    
    // สำหรับ OLAP (few, complex, long queries)
    pub fn olap() -> Self {
        let cpus = num_cpus::get() as u32;
        Self {
            max_connections: cpus * 2,
            min_connections: 2,
            acquire_timeout: Duration::from_secs(60),
            idle_timeout: Duration::from_secs(1800),
            max_lifetime: Duration::from_secs(7200),
        }
    }
    
    // สำหรับ Background Jobs
    pub fn background() -> Self {
        Self {
            max_connections: 5,
            min_connections: 1,
            acquire_timeout: Duration::from_secs(120),
            idle_timeout: Duration::from_secs(300),
            max_lifetime: Duration::from_secs(3600),
        }
    }
}
```

---

## Vacuum และ Maintenance

### VACUUM และ ANALYZE

```sql
-- VACUUM ป้องกัน table bloat
-- PostgreSQL มี autovacuum แต่บางครั้งต้อง manual

-- Basic VACUUM (ไม่ lock table)
VACUUM posts;

-- VACUUM ANALYZE (เพิ่ม statistics update)
VACUUM ANALYZE posts;

-- VACUUM FULL (compact table, lock table!)
-- ใช้เมื่อ table ใหญ่มาก เพิ่มไม่คืน disk space
VACUUM FULL posts;

-- ดู bloat
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) as table_size,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename) -
                   pg_relation_size(schemaname || '.' || tablename)) as index_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC;
```

### Maintenance ใน Rust

```rust
// src/db/maintenance.rs
use sqlx::PgPool;

pub struct MaintenanceRunner {
    pool: PgPool,
}

impl MaintenanceRunner {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
    
    // VACUUM ANALYZE
    pub async fn vacuum_analyze_table(&self, table: &str) -> Result<(), sqlx::Error> {
        log::info!("Running VACUUM ANALYZE on {}", table);
        
        // VACUUM ไม่รองรับ parameterized query → ต้องใช้ string format
        // *** ระวัง SQL injection: ตรวจสอบ table name ***
        if !is_safe_identifier(table) {
            return Err(sqlx::Error::Protocol(
                format!("Unsafe table name: {}", table)
            ));
        }
        
        sqlx::query(&format!("VACUUM ANALYZE {}", table))
            .execute(&self.pool)
            .await?;
        
        log::info!("VACUUM ANALYZE complete for {}", table);
        Ok(())
    }
    
    // REINDEX
    pub async fn reindex_table(&self, table: &str) -> Result<(), sqlx::Error> {
        if !is_safe_identifier(table) {
            return Err(sqlx::Error::Protocol("Unsafe table name".into()));
        }
        
        log::info!("Reindexing {}", table);
        sqlx::query(&format!("REINDEX TABLE {}", table))
            .execute(&self.pool)
            .await?;
        Ok(())
    }
    
    // ดู table sizes
    pub async fn get_table_sizes(&self) -> Result<Vec<TableSize>, sqlx::Error> {
        #[derive(Debug, sqlx::FromRow)]
        pub struct TableSize {
            pub table_name: Option<String>,
            pub row_count: Option<i64>,
            pub total_size: Option<String>,
            pub index_size: Option<String>,
            pub toast_size: Option<String>,
        }
        
        sqlx::query_as!(
            TableSize,
            r#"
            SELECT
                relname as table_name,
                n_live_tup as row_count,
                pg_size_pretty(pg_total_relation_size(oid)) as total_size,
                pg_size_pretty(pg_indexes_size(oid)) as index_size,
                pg_size_pretty(pg_total_relation_size(reltoastrelid)) as toast_size
            FROM pg_stat_user_tables
            ORDER BY pg_total_relation_size(oid) DESC
            "#
        )
        .fetch_all(&self.pool)
        .await
    }
    
    // หา table bloat
    pub async fn get_table_bloat(&self) -> Result<Vec<TableBloat>, sqlx::Error> {
        #[derive(Debug, sqlx::FromRow)]
        pub struct TableBloat {
            pub table_name: Option<String>,
            pub dead_tuples: Option<i64>,
            pub live_tuples: Option<i64>,
            pub bloat_ratio: Option<f64>,
            pub last_vacuum: Option<chrono::DateTime<chrono::Utc>>,
            pub last_analyze: Option<chrono::DateTime<chrono::Utc>>,
        }
        
        sqlx::query_as!(
            TableBloat,
            r#"
            SELECT
                relname as table_name,
                n_dead_tup as dead_tuples,
                n_live_tup as live_tuples,
                CASE WHEN n_live_tup > 0 
                    THEN n_dead_tup::float / n_live_tup::float 
                    ELSE 0 
                END as bloat_ratio,
                last_vacuum,
                last_analyze
            FROM pg_stat_user_tables
            WHERE n_dead_tup > 1000
            ORDER BY n_dead_tup DESC
            LIMIT 20
            "#
        )
        .fetch_all(&self.pool)
        .await
    }
    
    // Scheduled maintenance
    pub async fn run_nightly_maintenance(&self) {
        log::info!("Starting nightly maintenance...");
        
        let tables = ["users", "posts", "comments", "post_likes", "audit_logs"];
        
        for table in &tables {
            if let Err(e) = self.vacuum_analyze_table(table).await {
                log::error!("Maintenance failed for {}: {}", table, e);
            }
        }
        
        // Purge old audit logs (older than 90 days)
        if let Err(e) = self.purge_old_data().await {
            log::error!("Data purge failed: {}", e);
        }
        
        log::info!("Nightly maintenance complete");
    }
    
    async fn purge_old_data(&self) -> Result<(), sqlx::Error> {
        let deleted = sqlx::query!(
            r#"
            DELETE FROM audit_logs
            WHERE created_at < NOW() - interval '90 days'
            "#
        )
        .execute(&self.pool)
        .await?;
        
        log::info!("Purged {} old audit log entries", deleted.rows_affected());
        
        Ok(())
    }
}

fn is_safe_identifier(name: &str) -> bool {
    name.chars().all(|c| c.is_alphanumeric() || c == '_')
        && !name.is_empty()
        && name.len() <= 63
}
```

### Autovacuum Tuning

```sql
-- ปรับ autovacuum สำหรับ table ที่มี write เยอะ

-- ปรับ per-table settings
ALTER TABLE posts SET (
    autovacuum_vacuum_scale_factor = 0.01,  -- vacuum เมื่อ 1% rows เป็น dead
    autovacuum_analyze_scale_factor = 0.005,
    autovacuum_vacuum_cost_delay = 2,
    autovacuum_vacuum_threshold = 100
);

ALTER TABLE audit_logs SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_vacuum_threshold = 1000
);

-- ดู autovacuum status
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    vacuum_count,
    autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 10;
```

---

## Navigation

| ก่อนหน้า | หน้าหลัก | ถัดไป |
|---------|---------|------|
| [Part 038: Repository Pattern Advanced](../part_038/README.md) | [README หลัก](../../README.md) | [Part 040: Full REST API with Database](../part_040/README.md) |
