# Part 037: Advanced SQLx Queries

## สารบัญ
- [JOIN Queries](#join-queries)
- [Aggregate Functions](#aggregate-functions)
- [GROUP BY และ HAVING](#group-by-และ-having)
- [Subqueries](#subqueries)
- [CTEs (WITH clause)](#ctes-with-clause)
- [Full-Text Search](#full-text-search)
- [Dynamic Queries](#dynamic-queries)
- [JSONB Operations](#jsonb-operations)

---

## JOIN Queries

### INNER JOIN

```rust
// src/queries/joins.rs
use sqlx::PgPool;
use uuid::Uuid;
use serde::Serialize;
use chrono::{DateTime, Utc};

// Post พร้อม Author และ Category
#[derive(Debug, Serialize, sqlx::FromRow)]
pub struct PostWithDetails {
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
    pub category_slug: Option<String>,
    pub created_at: DateTime<Utc>,
}

pub async fn get_posts_with_details(
    pool: &PgPool,
    limit: i64,
    offset: i64,
) -> Result<Vec<PostWithDetails>, sqlx::Error> {
    // INNER JOIN: ต้องมี author (required)
    // LEFT JOIN: category optional
    sqlx::query_as!(
        PostWithDetails,
        r#"
        SELECT
            p.id,
            p.title,
            p.slug,
            p.excerpt,
            p.status,
            p.published_at,
            p.view_count,
            p.like_count,
            p.comment_count,
            p.author_id,
            u.display_name as author_name,
            u.avatar_url as author_avatar,
            p.category_id,
            c.name as category_name,
            c.slug as category_slug,
            p.created_at
        FROM posts p
        INNER JOIN users u ON p.author_id = u.id
        LEFT JOIN categories c ON p.category_id = c.id
        WHERE p.deleted_at IS NULL
            AND p.status = 'published'
        ORDER BY p.published_at DESC
        LIMIT $1
        OFFSET $2
        "#,
        limit,
        offset
    )
    .fetch_all(pool)
    .await
}
```

### LEFT JOIN และ RIGHT JOIN

```rust
// Posts พร้อม tags (many-to-many)
#[derive(Debug, Serialize)]
pub struct PostWithTags {
    pub id: Uuid,
    pub title: String,
    pub tags: Vec<TagInfo>,
}

#[derive(Debug, Serialize, sqlx::FromRow)]
pub struct PostWithTagRow {
    pub post_id: Uuid,
    pub post_title: String,
    pub tag_id: Option<Uuid>,
    pub tag_name: Option<String>,
    pub tag_color: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct TagInfo {
    pub id: Uuid,
    pub name: String,
    pub color: Option<String>,
}

pub async fn get_posts_with_tags(
    pool: &PgPool,
    post_ids: &[Uuid],
) -> Result<Vec<PostWithTags>, sqlx::Error> {
    // Query ด้วย LEFT JOIN
    let rows = sqlx::query_as!(
        PostWithTagRow,
        r#"
        SELECT
            p.id as post_id,
            p.title as post_title,
            t.id as tag_id,
            t.name as tag_name,
            t.color as tag_color
        FROM posts p
        LEFT JOIN post_tags pt ON p.id = pt.post_id
        LEFT JOIN tags t ON pt.tag_id = t.id
        WHERE p.id = ANY($1)
            AND p.deleted_at IS NULL
        ORDER BY p.created_at DESC, t.name ASC
        "#,
        post_ids as &[Uuid]
    )
    .fetch_all(pool)
    .await?;
    
    // Group rows ด้วย post_id
    use std::collections::HashMap;
    let mut posts_map: HashMap<Uuid, PostWithTags> = HashMap::new();
    
    for row in rows {
        let post = posts_map.entry(row.post_id).or_insert(PostWithTags {
            id: row.post_id,
            title: row.post_title,
            tags: Vec::new(),
        });
        
        if let (Some(tag_id), Some(tag_name)) = (row.tag_id, row.tag_name) {
            post.tags.push(TagInfo {
                id: tag_id,
                name: tag_name,
                color: row.tag_color,
            });
        }
    }
    
    // Preserve order
    let result: Vec<PostWithTags> = post_ids.iter()
        .filter_map(|id| posts_map.remove(id))
        .collect();
    
    Ok(result)
}

// Self JOIN: Categories พร้อม parent
#[derive(Debug, Serialize, sqlx::FromRow)]
pub struct CategoryWithParent {
    pub id: Uuid,
    pub name: String,
    pub slug: String,
    pub parent_id: Option<Uuid>,
    pub parent_name: Option<String>,
    pub parent_slug: Option<String>,
}

pub async fn get_categories_with_parent(
    pool: &PgPool,
) -> Result<Vec<CategoryWithParent>, sqlx::Error> {
    sqlx::query_as!(
        CategoryWithParent,
        r#"
        SELECT
            c.id,
            c.name,
            c.slug,
            c.parent_id,
            p.name as parent_name,
            p.slug as parent_slug
        FROM categories c
        LEFT JOIN categories p ON c.parent_id = p.id
        WHERE c.is_active = true
        ORDER BY p.sort_order NULLS FIRST, c.sort_order
        "#
    )
    .fetch_all(pool)
    .await
}
```

### CROSS JOIN และ Complex JOINs

```rust
// Complex JOIN: Users ที่ like post ของตัวเอง
#[derive(Debug, Serialize, sqlx::FromRow)]
pub struct UserPostInteraction {
    pub user_id: Uuid,
    pub username: String,
    pub post_id: Uuid,
    pub post_title: String,
    pub liked: bool,
    pub commented: bool,
}

pub async fn get_user_post_interactions(
    pool: &PgPool,
    user_id: Uuid,
) -> Result<Vec<UserPostInteraction>, sqlx::Error> {
    sqlx::query_as!(
        UserPostInteraction,
        r#"
        SELECT
            u.id as user_id,
            u.username,
            p.id as post_id,
            p.title as post_title,
            (pl.user_id IS NOT NULL) as "liked!",
            EXISTS(
                SELECT 1 FROM comments c
                WHERE c.post_id = p.id
                    AND c.author_id = u.id
                    AND c.deleted_at IS NULL
            ) as "commented!"
        FROM users u
        CROSS JOIN posts p
        LEFT JOIN post_likes pl ON pl.user_id = u.id AND pl.post_id = p.id
        WHERE u.id = $1
            AND p.status = 'published'
            AND p.deleted_at IS NULL
        ORDER BY p.published_at DESC
        LIMIT 20
        "#,
        user_id
    )
    .fetch_all(pool)
    .await
}
```

---

## Aggregate Functions

### COUNT, SUM, AVG, MAX, MIN

```rust
// src/queries/aggregates.rs

#[derive(Debug, Serialize, sqlx::FromRow)]
pub struct BlogStats {
    pub total_posts: i64,
    pub published_posts: i64,
    pub total_users: i64,
    pub total_comments: i64,
    pub avg_views: Option<f64>,
    pub max_views: Option<i32>,
    pub total_likes: Option<i64>,
}

pub async fn get_blog_stats(pool: &PgPool) -> Result<BlogStats, sqlx::Error> {
    sqlx::query_as!(
        BlogStats,
        r#"
        SELECT
            COUNT(*) as "total_posts!",
            COUNT(CASE WHEN status = 'published' THEN 1 END) as "published_posts!",
            (SELECT COUNT(*) FROM users WHERE deleted_at IS NULL) as "total_users!",
            (SELECT COUNT(*) FROM comments WHERE deleted_at IS NULL) as "total_comments!",
            AVG(view_count) as avg_views,
            MAX(view_count) as max_views,
            SUM(like_count) as total_likes
        FROM posts
        WHERE deleted_at IS NULL
        "#
    )
    .fetch_one(pool)
    .await
}

// Statistics per user
#[derive(Debug, Serialize, sqlx::FromRow)]
pub struct UserStats {
    pub user_id: Uuid,
    pub username: String,
    pub post_count: Option<i64>,
    pub total_views: Option<i64>,
    pub total_likes: Option<i64>,
    pub total_comments: Option<i64>,
    pub avg_post_views: Option<f64>,
    pub first_post_at: Option<DateTime<Utc>>,
    pub last_post_at: Option<DateTime<Utc>>,
}

pub async fn get_user_stats(
    pool: &PgPool,
    user_id: Uuid,
) -> Result<Option<UserStats>, sqlx::Error> {
    sqlx::query_as!(
        UserStats,
        r#"
        SELECT
            u.id as user_id,
            u.username,
            COUNT(p.id) as post_count,
            SUM(p.view_count) as total_views,
            SUM(p.like_count) as total_likes,
            SUM(p.comment_count) as total_comments,
            AVG(p.view_count) as avg_post_views,
            MIN(p.published_at) as first_post_at,
            MAX(p.published_at) as last_post_at
        FROM users u
        LEFT JOIN posts p ON u.id = p.author_id
            AND p.deleted_at IS NULL
            AND p.status = 'published'
        WHERE u.id = $1
            AND u.deleted_at IS NULL
        GROUP BY u.id, u.username
        "#,
        user_id
    )
    .fetch_optional(pool)
    .await
}
```

---

## GROUP BY และ HAVING

### Complex Grouping

```rust
// src/queries/groupby.rs

// Posts per category พร้อม stats
#[derive(Debug, Serialize, sqlx::FromRow)]
pub struct CategoryStats {
    pub category_id: Option<Uuid>,
    pub category_name: Option<String>,
    pub post_count: Option<i64>,
    pub total_views: Option<i64>,
    pub avg_views: Option<f64>,
    pub latest_post_at: Option<DateTime<Utc>>,
}

pub async fn get_category_stats(pool: &PgPool) -> Result<Vec<CategoryStats>, sqlx::Error> {
    sqlx::query_as!(
        CategoryStats,
        r#"
        SELECT
            c.id as category_id,
            c.name as category_name,
            COUNT(p.id) as post_count,
            SUM(p.view_count) as total_views,
            AVG(p.view_count::float) as avg_views,
            MAX(p.published_at) as latest_post_at
        FROM categories c
        LEFT JOIN posts p ON c.id = p.category_id
            AND p.status = 'published'
            AND p.deleted_at IS NULL
        WHERE c.is_active = true
        GROUP BY c.id, c.name
        HAVING COUNT(p.id) > 0
        ORDER BY total_views DESC NULLS LAST
        "#
    )
    .fetch_all(pool)
    .await
}

// Active authors (HAVING filter)
#[derive(Debug, Serialize, sqlx::FromRow)]
pub struct ActiveAuthor {
    pub author_id: Uuid,
    pub author_name: String,
    pub post_count: Option<i64>,
    pub total_views: Option<i64>,
}

pub async fn get_active_authors(
    pool: &PgPool,
    min_posts: i64,
) -> Result<Vec<ActiveAuthor>, sqlx::Error> {
    sqlx::query_as!(
        ActiveAuthor,
        r#"
        SELECT
            u.id as author_id,
            u.display_name as author_name,
            COUNT(p.id) as post_count,
            SUM(p.view_count) as total_views
        FROM users u
        INNER JOIN posts p ON u.id = p.author_id
            AND p.status = 'published'
            AND p.deleted_at IS NULL
        WHERE u.deleted_at IS NULL
        GROUP BY u.id, u.display_name
        HAVING COUNT(p.id) >= $1
        ORDER BY post_count DESC
        "#,
        min_posts
    )
    .fetch_all(pool)
    .await
}

// Posts per month (date grouping)
#[derive(Debug, Serialize, sqlx::FromRow)]
pub struct PostsPerMonth {
    pub year_month: Option<String>,
    pub post_count: Option<i64>,
    pub total_views: Option<i64>,
}

pub async fn get_posts_per_month(
    pool: &PgPool,
    months: i32,
) -> Result<Vec<PostsPerMonth>, sqlx::Error> {
    sqlx::query_as!(
        PostsPerMonth,
        r#"
        SELECT
            TO_CHAR(published_at, 'YYYY-MM') as year_month,
            COUNT(*) as post_count,
            SUM(view_count) as total_views
        FROM posts
        WHERE status = 'published'
            AND deleted_at IS NULL
            AND published_at >= NOW() - ($1 || ' months')::interval
        GROUP BY TO_CHAR(published_at, 'YYYY-MM')
        ORDER BY year_month DESC
        "#,
        months.to_string()
    )
    .fetch_all(pool)
    .await
}
```

---

## Subqueries

### Correlated และ Non-Correlated Subqueries

```rust
// src/queries/subqueries.rs

// Posts ที่ popular กว่า average
pub async fn get_popular_posts(pool: &PgPool) -> Result<Vec<Post>, sqlx::Error> {
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
        WHERE status = 'published'
            AND deleted_at IS NULL
            AND view_count > (
                SELECT AVG(view_count)
                FROM posts
                WHERE status = 'published'
                    AND deleted_at IS NULL
            )
        ORDER BY view_count DESC
        LIMIT 10
        "#
    )
    .fetch_all(pool)
    .await
}

// Users ที่มี posts มากกว่า N (Correlated subquery)
pub async fn get_prolific_authors(
    pool: &PgPool,
    min_posts: i64
) -> Result<Vec<User>, sqlx::Error> {
    sqlx::query_as!(
        User,
        r#"
        SELECT
            id, email, username, password_hash,
            display_name, bio, avatar_url, website_url,
            is_active, is_admin, is_verified,
            last_login_at, created_at, updated_at, deleted_at
        FROM users u
        WHERE deleted_at IS NULL
            AND (
                SELECT COUNT(*)
                FROM posts p
                WHERE p.author_id = u.id
                    AND p.status = 'published'
                    AND p.deleted_at IS NULL
            ) >= $1
        ORDER BY (
            SELECT COUNT(*)
            FROM posts p
            WHERE p.author_id = u.id
                AND p.status = 'published'
        ) DESC
        "#,
        min_posts
    )
    .fetch_all(pool)
    .await
}

// EXISTS subquery
pub async fn get_users_with_published_posts(
    pool: &PgPool
) -> Result<Vec<User>, sqlx::Error> {
    sqlx::query_as!(
        User,
        r#"
        SELECT
            id, email, username, password_hash,
            display_name, bio, avatar_url, website_url,
            is_active, is_admin, is_verified,
            last_login_at, created_at, updated_at, deleted_at
        FROM users u
        WHERE deleted_at IS NULL
            AND EXISTS (
                SELECT 1
                FROM posts p
                WHERE p.author_id = u.id
                    AND p.status = 'published'
                    AND p.deleted_at IS NULL
            )
        ORDER BY display_name
        "#
    )
    .fetch_all(pool)
    .await
}

// NOT EXISTS subquery
pub async fn get_users_without_posts(
    pool: &PgPool
) -> Result<Vec<User>, sqlx::Error> {
    sqlx::query_as!(
        User,
        r#"
        SELECT
            id, email, username, password_hash,
            display_name, bio, avatar_url, website_url,
            is_active, is_admin, is_verified,
            last_login_at, created_at, updated_at, deleted_at
        FROM users u
        WHERE deleted_at IS NULL
            AND NOT EXISTS (
                SELECT 1
                FROM posts p
                WHERE p.author_id = u.id
                    AND p.deleted_at IS NULL
            )
        ORDER BY created_at DESC
        "#
    )
    .fetch_all(pool)
    .await
}

// IN subquery
pub async fn get_posts_in_categories(
    pool: &PgPool,
    category_slugs: &[&str]
) -> Result<Vec<Post>, sqlx::Error> {
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
        WHERE deleted_at IS NULL
            AND status = 'published'
            AND category_id IN (
                SELECT id FROM categories
                WHERE slug = ANY($1)
            )
        ORDER BY published_at DESC
        "#,
        category_slugs as &[&str]
    )
    .fetch_all(pool)
    .await
}
```

---

## CTEs (WITH clause)

### Common Table Expressions

```rust
// src/queries/ctes.rs

// Recursive CTE สำหรับ category tree
#[derive(Debug, Serialize, sqlx::FromRow)]
pub struct CategoryTree {
    pub id: Uuid,
    pub name: String,
    pub slug: String,
    pub parent_id: Option<Uuid>,
    pub depth: Option<i32>,
    pub path: Option<String>,
}

pub async fn get_category_tree(pool: &PgPool) -> Result<Vec<CategoryTree>, sqlx::Error> {
    sqlx::query_as!(
        CategoryTree,
        r#"
        WITH RECURSIVE category_tree AS (
            -- Base case: root categories
            SELECT
                id, name, slug, parent_id,
                0 as depth,
                name::text as path
            FROM categories
            WHERE parent_id IS NULL
                AND is_active = true
            
            UNION ALL
            
            -- Recursive case: child categories
            SELECT
                c.id, c.name, c.slug, c.parent_id,
                ct.depth + 1 as depth,
                ct.path || ' > ' || c.name as path
            FROM categories c
            INNER JOIN category_tree ct ON c.parent_id = ct.id
            WHERE c.is_active = true
        )
        SELECT id, name, slug, parent_id, depth, path
        FROM category_tree
        ORDER BY path
        "#
    )
    .fetch_all(pool)
    .await
}

// CTE สำหรับ complex analytics
#[derive(Debug, Serialize, sqlx::FromRow)]
pub struct TopAuthorPost {
    pub author_id: Uuid,
    pub author_name: String,
    pub post_id: Uuid,
    pub post_title: String,
    pub view_count: i32,
    pub author_rank: Option<i64>,
}

pub async fn get_top_posts_per_author(pool: &PgPool) -> Result<Vec<TopAuthorPost>, sqlx::Error> {
    sqlx::query_as!(
        TopAuthorPost,
        r#"
        WITH author_post_ranks AS (
            SELECT
                p.id as post_id,
                p.title as post_title,
                p.view_count,
                p.author_id,
                u.display_name as author_name,
                ROW_NUMBER() OVER (
                    PARTITION BY p.author_id
                    ORDER BY p.view_count DESC
                ) as rank
            FROM posts p
            INNER JOIN users u ON p.author_id = u.id
            WHERE p.status = 'published'
                AND p.deleted_at IS NULL
                AND u.deleted_at IS NULL
        ),
        top_authors AS (
            SELECT author_id
            FROM posts
            WHERE status = 'published' AND deleted_at IS NULL
            GROUP BY author_id
            ORDER BY SUM(view_count) DESC
            LIMIT 10
        )
        SELECT
            apr.author_id,
            apr.author_name,
            apr.post_id,
            apr.post_title,
            apr.view_count,
            apr.rank as author_rank
        FROM author_post_ranks apr
        WHERE apr.author_id IN (SELECT author_id FROM top_authors)
            AND apr.rank <= 3  -- Top 3 posts per author
        ORDER BY 
            (SELECT SUM(view_count) FROM posts WHERE author_id = apr.author_id) DESC,
            apr.rank
        "#
    )
    .fetch_all(pool)
    .await
}

// CTE ช่วยให้ query readable
pub async fn get_engagement_report(
    pool: &PgPool,
    days: i32,
) -> Result<Vec<serde_json::Value>, sqlx::Error> {
    let rows = sqlx::query!(
        r#"
        WITH date_series AS (
            SELECT generate_series(
                CURRENT_DATE - ($1 || ' days')::interval,
                CURRENT_DATE,
                '1 day'::interval
            )::date as date
        ),
        daily_posts AS (
            SELECT
                DATE(published_at) as date,
                COUNT(*) as new_posts,
                SUM(view_count) as views
            FROM posts
            WHERE status = 'published'
                AND deleted_at IS NULL
                AND published_at >= CURRENT_DATE - ($1 || ' days')::interval
            GROUP BY DATE(published_at)
        ),
        daily_comments AS (
            SELECT
                DATE(created_at) as date,
                COUNT(*) as new_comments
            FROM comments
            WHERE deleted_at IS NULL
                AND created_at >= CURRENT_DATE - ($1 || ' days')::interval
            GROUP BY DATE(created_at)
        )
        SELECT
            ds.date::text,
            COALESCE(dp.new_posts, 0) as new_posts,
            COALESCE(dp.views, 0) as views,
            COALESCE(dc.new_comments, 0) as new_comments
        FROM date_series ds
        LEFT JOIN daily_posts dp ON ds.date = dp.date
        LEFT JOIN daily_comments dc ON ds.date = dc.date
        ORDER BY ds.date
        "#,
        days.to_string()
    )
    .fetch_all(pool)
    .await?;
    
    let result = rows.iter().map(|r| serde_json::json!({
        "date": r.date,
        "new_posts": r.new_posts,
        "views": r.views,
        "new_comments": r.new_comments
    })).collect();
    
    Ok(result)
}
```

---

## Full-Text Search

### PostgreSQL Full-Text Search

```rust
// src/queries/search.rs

#[derive(Debug, Serialize, sqlx::FromRow)]
pub struct SearchResult {
    pub id: Uuid,
    pub title: String,
    pub slug: String,
    pub excerpt: Option<String>,
    pub author_name: String,
    pub published_at: Option<DateTime<Utc>>,
    pub rank: Option<f32>,
    pub headline: Option<String>,
}

pub async fn search_posts(
    pool: &PgPool,
    query: &str,
    limit: i64,
    offset: i64,
) -> Result<Vec<SearchResult>, sqlx::Error> {
    sqlx::query_as!(
        SearchResult,
        r#"
        SELECT
            p.id,
            p.title,
            p.slug,
            p.excerpt,
            u.display_name as author_name,
            p.published_at,
            ts_rank(p.search_vector, plainto_tsquery('english', $1)) as rank,
            ts_headline(
                'english',
                p.content,
                plainto_tsquery('english', $1),
                'MaxWords=35, MinWords=15, ShortWord=3, MaxFragments=2'
            ) as headline
        FROM posts p
        INNER JOIN users u ON p.author_id = u.id
        WHERE p.deleted_at IS NULL
            AND p.status = 'published'
            AND p.search_vector @@ plainto_tsquery('english', $1)
        ORDER BY rank DESC, p.published_at DESC
        LIMIT $2
        OFFSET $3
        "#,
        query,
        limit,
        offset
    )
    .fetch_all(pool)
    .await
}

// ค้นหาแบบ phrase search
pub async fn search_posts_phrase(
    pool: &PgPool,
    phrase: &str,
) -> Result<Vec<SearchResult>, sqlx::Error> {
    sqlx::query_as!(
        SearchResult,
        r#"
        SELECT
            p.id,
            p.title,
            p.slug,
            p.excerpt,
            u.display_name as author_name,
            p.published_at,
            ts_rank(p.search_vector, phraseto_tsquery('english', $1)) as rank,
            ts_headline(
                'english',
                p.title || ' ' || p.content,
                phraseto_tsquery('english', $1)
            ) as headline
        FROM posts p
        INNER JOIN users u ON p.author_id = u.id
        WHERE p.deleted_at IS NULL
            AND p.status = 'published'
            AND p.search_vector @@ phraseto_tsquery('english', $1)
        ORDER BY rank DESC
        LIMIT 10
        "#,
        phrase
    )
    .fetch_all(pool)
    .await
}

// Fuzzy search ด้วย pg_trgm
pub async fn fuzzy_search_posts(
    pool: &PgPool,
    query: &str,
    similarity_threshold: f32,
) -> Result<Vec<SearchResult>, sqlx::Error> {
    sqlx::query_as!(
        SearchResult,
        r#"
        SELECT
            p.id,
            p.title,
            p.slug,
            p.excerpt,
            u.display_name as author_name,
            p.published_at,
            similarity(p.title, $1) as rank,
            NULL::text as headline
        FROM posts p
        INNER JOIN users u ON p.author_id = u.id
        WHERE p.deleted_at IS NULL
            AND p.status = 'published'
            AND similarity(p.title, $1) > $2
        ORDER BY rank DESC
        LIMIT 10
        "#,
        query,
        similarity_threshold
    )
    .fetch_all(pool)
    .await
}

// Combined search (FTS + trigram)
pub async fn smart_search(
    pool: &PgPool,
    query: &str,
) -> Result<Vec<SearchResult>, sqlx::Error> {
    sqlx::query_as!(
        SearchResult,
        r#"
        SELECT
            p.id,
            p.title,
            p.slug,
            p.excerpt,
            u.display_name as author_name,
            p.published_at,
            GREATEST(
                ts_rank(p.search_vector, plainto_tsquery('english', $1)),
                similarity(p.title, $1)::float4
            ) as rank,
            ts_headline('english', p.content, plainto_tsquery('english', $1)) as headline
        FROM posts p
        INNER JOIN users u ON p.author_id = u.id
        WHERE p.deleted_at IS NULL
            AND p.status = 'published'
            AND (
                p.search_vector @@ plainto_tsquery('english', $1)
                OR similarity(p.title, $1) > 0.3
            )
        ORDER BY rank DESC
        LIMIT 10
        "#,
        query
    )
    .fetch_all(pool)
    .await
}
```

---

## Dynamic Queries

### Query Builder Pattern

```rust
// src/queries/dynamic.rs
use sqlx::postgres::PgArguments;
use sqlx::Arguments;

// Dynamic query builder
pub struct QueryBuilder {
    conditions: Vec<String>,
    params_count: usize,
}

impl QueryBuilder {
    pub fn new() -> Self {
        Self {
            conditions: Vec::new(),
            params_count: 0,
        }
    }
    
    pub fn add_condition(&mut self, condition: String) {
        self.params_count += 1;
        self.conditions.push(condition);
    }
    
    pub fn build_where(&self) -> String {
        if self.conditions.is_empty() {
            String::new()
        } else {
            format!("WHERE {}", self.conditions.join(" AND "))
        }
    }
}

// Dynamic post search
#[derive(Debug, Default)]
pub struct AdvancedPostFilter {
    pub search: Option<String>,
    pub status: Option<String>,
    pub author_id: Option<Uuid>,
    pub category_id: Option<Uuid>,
    pub tag_ids: Option<Vec<Uuid>>,
    pub min_views: Option<i32>,
    pub published_after: Option<chrono::NaiveDate>,
    pub published_before: Option<chrono::NaiveDate>,
    pub sort_by: Option<String>,
    pub sort_order: Option<String>,
    pub page: Option<i64>,
    pub per_page: Option<i64>,
}

pub async fn advanced_post_search(
    pool: &PgPool,
    filter: AdvancedPostFilter,
) -> Result<Vec<Post>, sqlx::Error> {
    // sqlx ไม่รองรับ truly dynamic queries กับ macro
    // ต้องใช้ query() แบบ runtime แทน
    
    let mut sql = String::from(
        r#"
        SELECT DISTINCT
            p.id, p.title, p.slug, p.content, p.excerpt,
            p.cover_image_url, p.status, p.published_at,
            p.author_id, p.category_id,
            p.view_count, p.like_count, p.comment_count,
            p.meta_title, p.meta_description,
            p.created_at, p.updated_at, p.deleted_at
        FROM posts p
        "#
    );
    
    // JOIN สำหรับ tags
    if filter.tag_ids.is_some() {
        sql.push_str("LEFT JOIN post_tags pt ON p.id = pt.post_id ");
    }
    
    let mut conditions = vec!["p.deleted_at IS NULL".to_string()];
    let mut param_idx = 1usize;
    
    if filter.search.is_some() {
        conditions.push(format!(
            "p.search_vector @@ plainto_tsquery('english', ${})",
            param_idx
        ));
        param_idx += 1;
    }
    
    if filter.status.is_some() {
        conditions.push(format!("p.status = ${}", param_idx));
        param_idx += 1;
    }
    
    if filter.author_id.is_some() {
        conditions.push(format!("p.author_id = ${}", param_idx));
        param_idx += 1;
    }
    
    if filter.category_id.is_some() {
        conditions.push(format!("p.category_id = ${}", param_idx));
        param_idx += 1;
    }
    
    if filter.min_views.is_some() {
        conditions.push(format!("p.view_count >= ${}", param_idx));
        param_idx += 1;
    }
    
    sql.push_str(&format!("WHERE {}", conditions.join(" AND ")));
    
    // Sorting
    let sort_col = match filter.sort_by.as_deref() {
        Some("views") => "view_count",
        Some("likes") => "like_count",
        Some("published") => "published_at",
        _ => "created_at",
    };
    let sort_dir = match filter.sort_order.as_deref() {
        Some("asc") => "ASC",
        _ => "DESC",
    };
    sql.push_str(&format!(" ORDER BY p.{} {} NULLS LAST", sort_col, sort_dir));
    
    // Pagination
    let per_page = filter.per_page.unwrap_or(10).min(100);
    let page = filter.page.unwrap_or(1).max(1);
    let offset = (page - 1) * per_page;
    sql.push_str(&format!(" LIMIT ${} OFFSET ${}", param_idx, param_idx + 1));
    
    // Build query
    let mut query = sqlx::query(&sql);
    
    if let Some(s) = &filter.search {
        query = query.bind(s);
    }
    if let Some(s) = &filter.status {
        query = query.bind(s);
    }
    if let Some(id) = filter.author_id {
        query = query.bind(id);
    }
    if let Some(id) = filter.category_id {
        query = query.bind(id);
    }
    if let Some(v) = filter.min_views {
        query = query.bind(v);
    }
    query = query.bind(per_page).bind(offset);
    
    let rows = query.fetch_all(pool).await?;
    
    // Map rows to Post struct manually
    let posts = rows.iter().map(|row| {
        use sqlx::Row;
        Post {
            id: row.get("id"),
            title: row.get("title"),
            slug: row.get("slug"),
            content: row.get("content"),
            excerpt: row.get("excerpt"),
            cover_image_url: row.get("cover_image_url"),
            status: row.get("status"),
            published_at: row.get("published_at"),
            author_id: row.get("author_id"),
            category_id: row.get("category_id"),
            view_count: row.get("view_count"),
            like_count: row.get("like_count"),
            comment_count: row.get("comment_count"),
            meta_title: row.get("meta_title"),
            meta_description: row.get("meta_description"),
            created_at: row.get("created_at"),
            updated_at: row.get("updated_at"),
            deleted_at: row.get("deleted_at"),
        }
    }).collect();
    
    Ok(posts)
}
```

---

## JSONB Operations

### การทำงานกับ JSONB

```rust
// src/queries/jsonb.rs

// เพิ่ม JSONB column ใน migration:
// ALTER TABLE posts ADD COLUMN metadata JSONB DEFAULT '{}'::jsonb;

#[derive(Debug, Serialize, Deserialize)]
pub struct PostMetadata {
    pub reading_time_minutes: Option<i32>,
    pub word_count: Option<i32>,
    pub external_links: Option<Vec<String>>,
    pub og_image: Option<String>,
    pub custom_fields: Option<std::collections::HashMap<String, serde_json::Value>>,
}

pub async fn update_post_metadata(
    pool: &PgPool,
    post_id: Uuid,
    metadata: &PostMetadata,
) -> Result<(), sqlx::Error> {
    sqlx::query!(
        r#"
        UPDATE posts
        SET metadata = $2::jsonb
        WHERE id = $1
        "#,
        post_id,
        serde_json::to_value(metadata).unwrap()
    )
    .execute(pool)
    .await?;
    
    Ok(())
}

pub async fn get_post_metadata(
    pool: &PgPool,
    post_id: Uuid,
) -> Result<Option<serde_json::Value>, sqlx::Error> {
    let row = sqlx::query!(
        r#"
        SELECT metadata
        FROM posts
        WHERE id = $1 AND deleted_at IS NULL
        "#,
        post_id
    )
    .fetch_optional(pool)
    .await?;
    
    Ok(row.and_then(|r| r.metadata))
}

// Query ด้วย JSONB operators
pub async fn get_posts_with_og_image(pool: &PgPool) -> Result<Vec<Post>, sqlx::Error> {
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
        WHERE deleted_at IS NULL
            AND metadata ? 'og_image'  -- ตรวจสอบว่า key มีอยู่
            AND metadata->>'og_image' IS NOT NULL
        "#
    )
    .fetch_all(pool)
    .await
}

// Update specific JSONB field
pub async fn update_reading_time(
    pool: &PgPool,
    post_id: Uuid,
    minutes: i32,
) -> Result<(), sqlx::Error> {
    sqlx::query!(
        r#"
        UPDATE posts
        SET metadata = jsonb_set(
            COALESCE(metadata, '{}'::jsonb),
            '{reading_time_minutes}',
            $2::text::jsonb
        )
        WHERE id = $1
        "#,
        post_id,
        minutes.to_string()
    )
    .execute(pool)
    .await?;
    
    Ok(())
}

// Query JSONB array
pub async fn get_posts_with_tag_in_metadata(
    pool: &PgPool,
    tag: &str,
) -> Result<Vec<Post>, sqlx::Error> {
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
        WHERE deleted_at IS NULL
            AND metadata @> jsonb_build_object('tags', jsonb_build_array($1))
        "#,
        tag
    )
    .fetch_all(pool)
    .await
}

// Aggregate JSONB data
pub async fn get_metadata_statistics(pool: &PgPool) -> Result<serde_json::Value, sqlx::Error> {
    let row = sqlx::query!(
        r#"
        SELECT
            AVG((metadata->>'reading_time_minutes')::int) as avg_reading_time,
            AVG((metadata->>'word_count')::int) as avg_word_count,
            COUNT(CASE WHEN metadata ? 'og_image' THEN 1 END) as posts_with_og_image
        FROM posts
        WHERE deleted_at IS NULL
            AND status = 'published'
            AND metadata IS NOT NULL
        "#
    )
    .fetch_one(pool)
    .await?;
    
    Ok(serde_json::json!({
        "avg_reading_time_minutes": row.avg_reading_time,
        "avg_word_count": row.avg_word_count,
        "posts_with_og_image": row.posts_with_og_image
    }))
}
```

---

## Navigation

| ก่อนหน้า | หน้าหลัก | ถัดไป |
|---------|---------|------|
| [Part 036: Redis Integration](../part_036/README.md) | [README หลัก](../../README.md) | [Part 038: Repository Pattern Advanced](../part_038/README.md) |
